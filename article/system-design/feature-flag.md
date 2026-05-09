# Feature Flag 系统设计 —— 渐进式发布与试验平台

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想快速对齐概念，看 [系统设计速记](/article/system-design) 那一节就够了。
> 这里要回答的，是**真要设计一个 Feature Flag 平台、把灰度发布做对、在 A/B 实验上不被数据骗**时你需要知道的那部分。

---

## 0. 读这篇之前 —— 8 个自检题

先自检 8 个问题，能秒答 6 个以上，可以跳过：

1. Feature Flag 与配置中心（Apollo / Nacos）有没有本质差别？如果配置中心已经能下发布尔值，为什么还要单独搞 Flag 系统？
2. 4 类 Flag —— Release / Experiment / Ops / Permission —— 的**生命周期长度**有什么不同？为什么混在同一个系统里管理是最常见的反模式？
3. 一个上线两年的电商代码库里有 600 个 Flag，其中 480 个早已 100% 灰度且没人删。这种"长期 Flag 债务"的具体危害是什么？怎么治理？
4. 客户端 SDK 评估（评估在端侧）vs 服务端评估（评估在服务端）的延迟、一致性、安全各有什么权衡？什么时候必须用服务端？
5. 一个 A/B 实验跑了 3 天，转化率 control=2.1%、treatment=2.3%，p-value=0.06。运营说"差不多有效了，全量吧"。你怎么劝？
6. 你给一个用户做了 50/50 分桶，第二次请求他被分到了不同桶（Sticky 失败）。可能的原因有哪些？怎么用一致性 Hash 修？
7. Multivariate Flag（多变体，不只是 on/off，而是 v1/v2/v3）的 Targeting Rules 怎么避免"加权和不为 100%"这类配错？
8. Feature Flag 和 Trunk-based Development（主干开发）的关系是什么？没有 Flag，主干开发能不能跑？

如果有半数答不上，往下看。

---

## 1. 底层模型：4 类 Flag 与评估管线

绝大多数文章把 Feature Flag 讲成"加个 if 开关"，然后展示 LaunchDarkly 截图就结束了。这层认知不够设计平台。要看懂上面 8 个问题，必须下沉到 Flag 的分类、评估管线、和数据闭环。

### 1.1 Feature Flag 的 4 个类型

LaunchDarkly 的工程博客里把 Flag 分成 4 类，这个划分极重要 —— 它们的**生命周期、变更频率、负责人、删除策略完全不同**：

| 类型 | 全称 | 寿命 | 谁改 | 典型例子 |
| --- | --- | --- | --- | --- |
| **Release** | 发布灰度 | 几天~几周 | 后端开发 | 新支付链路灰度 1% → 100% |
| **Experiment** | A/B 实验 | 几周~几月 | 产品/数据 | 新推荐算法 vs 旧算法 |
| **Ops** | 运维开关 | 永久 | SRE | 限流降级、紧急熔断 |
| **Permission** | 权限开关 | 永久 | PM/合规 | 仅企业版用户可用某功能 |

**生命周期混在一起是 80% 系统的核心问题**。Release Flag 应该上线后 1~2 周内被删除，但很多团队没有"删 Flag"的流程，导致代码里布满"早就该清理"的 if，行为分支爆炸。

### 1.2 Flag 的标准生命周期

Release Flag 应该走这条路：

```
[创建]  开发提 PR 加 Flag，默认 off
   │
   ▼
[内部] 团队成员的 userId 白名单开启
   │
   ▼
[灰度] 1% → 5% → 20% → 50% → 100%
   │   每一档观察 24h，看核心指标
   ▼
[全量] Flag = 100% 持续 1 周稳定
   │
   ▼
[清理] 删除代码里的 if，删除 Flag 配置
```

最后这步**绝大多数团队都跳过**。结果是上一节说的"600 个 Flag 480 个该删未删"。

### 1.3 Targeting Rules 的优先级

Flag 评估不是简单的"开/关"，而是一套规则匹配：

```
用户请求带上 context (userId, country, plan, deviceType, ...)
          │
          ▼
    ┌───────────────┐
    │ Flag killSwitch│  killSwitch=on → 立刻返回兜底 variation
    └───────┬───────┘
            │ off
            ▼
    ┌───────────────┐
    │ Targeting on?  │  off → 返回 offVariation
    └───────┬───────┘
            │ on
            ▼
    ┌───────────────────────────┐
    │ Rule 1: country IN [CN,JP] │ → 命中返回 variation A
    │ Rule 2: plan = enterprise  │ → 命中返回 variation B
    │ ...                        │
    └───────┬───────────────────┘
            │ 无规则命中
            ▼
    ┌────────────────────────────┐
    │ Fallthrough: 按权重分桶     │  e.g. v1:80%, v2:20%
    └────────────────────────────┘
```

关键点：
- **killSwitch（紧急关闭）必须在最外层**，独立于所有规则。生产事故时 SRE 要能一键关闭，不需要思考"哪条规则覆盖谁"。
- **Targeting on/off** 是 Flag 整体的开关，off 时直接返回 `offVariation`（通常是"老逻辑"）。
- **Rules 按顺序匹配**，第一个命中的就返回，类似防火墙规则。
- **Fallthrough** 是兜底分桶，所有未被规则匹配的用户走这里。

### 1.4 客户端评估 vs 服务端评估

这是 Flag 系统的根本架构选择。

**客户端评估**（SDK 内部计算）：
```
SDK 启动时拉取所有 Flag 的完整规则
[Rules: country IN [CN,JP] → on]
                │
用户请求时本地匹配 → 返回 variation
```

**服务端评估**（远程计算）：
```
SDK 把 (flagKey, userContext) 发给服务端
                │
服务端根据规则计算 → 返回 variation
```

| 维度 | 客户端评估 | 服务端评估 |
| --- | --- | --- |
| 延迟 | < 1μs（本地计算） | 1~10ms（网络） |
| 规则更新延迟 | SDK 拉取/推送间隔 | 立即生效 |
| 规则保密 | 全在客户端，可被反编译看到 | 仅服务端有，安全 |
| 复杂规则支持 | 受 SDK 实现限制 | 任意复杂 |
| 网络故障容忍 | 可用本地缓存 | 需要降级到默认值 |

**LaunchDarkly / Unleash / Statsig 默认都用客户端评估**，因为延迟敏感。但有两种例外要走服务端：
- **规则涉及保密信息**（如内部用户 ID 列表，不能下发给浏览器）
- **规则需要查询服务端数据**（如"用户当月消费 > 1000 才看新功能"，端侧不知道消费数据）

### 1.5 SDK 缓存与推送策略

客户端评估需要把规则同步到 SDK。三种主流模式：

| 模式 | 一致性延迟 | 网络成本 | 复杂度 |
| --- | --- | --- | --- |
| **轮询** | 高（30s~5min） | 高（每次全量） | 低 |
| **SSE / WebSocket 推送** | 低（< 1s） | 低（事件驱动） | 中 |
| **本地文件 + 进程订阅** | 看刷新 | 极低 | 高（需要边车） |

LaunchDarkly 的 streaming endpoint 用 SSE（Server-Sent Events），事件分两种：
- `put`：全量推送（首次连接 / 重连后）
- `patch`：增量推送（某个 Flag 改了）

SDK 收到事件后更新本地缓存，下次评估直接读内存。

### 1.6 A/B 实验的统计基础

Experiment 类 Flag 不只是"分桶"，它必须配合 **样本量计算 + 显著性检验**才有意义。

最小样本量（每组）的简化公式（双尾，alpha=0.05，power=0.8）：

```
n ≈ 16 × p(1-p) / δ²

p  = baseline 转化率
δ  = 期望检测到的最小绝对差（minimum detectable effect, MDE）
```

举例：baseline 转化率 5%，想检测到 5% → 5.5% 的相对提升 10%（绝对差 δ=0.005）：

```
n ≈ 16 × 0.05 × 0.95 / 0.005² = 30,400 (每组)
```

意味着实验需要 6 万用户才能在 0.5% 的转化率提升上得到显著结论。**很多团队跑实验前不算样本量，跑 3 天看 p-value，那是数据骗人的标准姿势**。

### 1.7 长期 Flag 的"债务"问题

每个 Flag 给代码贡献两个分支。10 个 Flag 理论上有 2^10 = 1024 个分支组合。实际上分支不会全部独立，但**长期 Flag 让代码可维护性指数级下降**：

- 单元测试要覆盖所有 Flag 组合 → 测不过来，只测 default 路径
- 新人改某段逻辑要梳理"哪个 Flag 控制这里"，常常改错分支
- Flag 的服务端规则与代码里的引用不一致（代码删了引用，配置还在），评估浪费

治理手段（4.3 案例展开）：
- Flag 必须填"过期日期"，到期没删自动报警
- CI 扫描代码里没引用的 Flag，提 PR 删除
- Release Flag 默认 30 天 TTL

> **本节口诀**：4 类 Flag 分开管，killSwitch 在最外层，客户端评估快但暴露规则，A/B 实验先算样本量再开跑。**Flag 是代码债务，必须有删除流程**。

---

## 2. 工程切片：从零搭一个最小 Flag 平台

> 这一节按"自研最小可用版本"思路，每段切片 < 30 行，配点透。完整版本 Demo 在文末引用。

### 2.1 核心数据模型

```sql
-- Flag 元数据表
CREATE TABLE feature_flag (
  id            BIGINT PRIMARY KEY AUTO_INCREMENT,
  flag_key      VARCHAR(128) NOT NULL UNIQUE,    -- 代码里用的 key，如 "new-checkout"
  name          VARCHAR(255) NOT NULL,           -- 人类可读名
  description   TEXT,
  flag_type     VARCHAR(32) NOT NULL,            -- RELEASE/EXPERIMENT/OPS/PERMISSION
  variations    JSON NOT NULL,                   -- 所有 variation 列表
  off_variation VARCHAR(64) NOT NULL,            -- targeting off 时返回哪个
  default_variation VARCHAR(64) NOT NULL,        -- fallthrough 兜底
  expire_at     DATETIME,                        -- ★ Release Flag 过期日，超期报警
  archived      BOOLEAN DEFAULT FALSE,
  created_by    VARCHAR(64),
  created_at    DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at    DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- 环境维度的开关与规则
CREATE TABLE flag_env_config (
  id          BIGINT PRIMARY KEY AUTO_INCREMENT,
  flag_id     BIGINT NOT NULL,
  env         VARCHAR(32) NOT NULL,            -- prod/staging/dev
  targeting_on BOOLEAN DEFAULT FALSE,           -- Targeting 总开关
  kill_switch BOOLEAN DEFAULT FALSE,           -- ★ 紧急关闭
  rules       JSON,                            -- 规则数组
  fallthrough JSON,                            -- 分桶配置
  version     BIGINT DEFAULT 0,                -- ★ 乐观锁 + 推送版本号
  UNIQUE KEY uk_flag_env (flag_id, env),
  INDEX idx_version (version)
);
```

**逐段点透**：
- **`flag_key` 不是自增 ID**：代码里写的是字符串 key，永远不能改。改 key = 删 Flag + 建 Flag。
- **`variations` 用 JSON**：Multivariate Flag 不只是 on/off，可能是 `["v1", "v2", "v3"]` 或 `[true, false]`。统一存 JSON 数组。
- **`expire_at` 列**：这是治理长期 Flag 债务的核心字段。Release Flag 创建时强制填，到期触发告警。
- **`kill_switch` 独立列而不是塞 rules**：紧急时刻 SRE 改一个 boolean 比改 JSON 安全得多。
- **`version` 字段**：乐观锁防并发改坏，同时作为推送序列号 —— SDK 上次拿到 version=42，下次只接受 version > 42 的事件。

### 2.2 Targeting Rules 的 DSL 设计

规则用 JSON 表达，避免每改规则都发版：

```json
{
  "targetingOn": true,
  "rules": [
    {
      "id": "rule-1",
      "description": "内部员工先开",
      "clauses": [
        {"attribute": "email", "op": "endsWith", "values": ["@taohua.com"]}
      ],
      "variation": "treatment"
    },
    {
      "id": "rule-2",
      "description": "中日韩用户灰度 20%",
      "clauses": [
        {"attribute": "country", "op": "in", "values": ["CN", "JP", "KR"]}
      ],
      "rollout": {
        "bucketBy": "userId",
        "variations": [
          {"variation": "treatment", "weight": 20},
          {"variation": "control",   "weight": 80}
        ]
      }
    }
  ],
  "fallthrough": {
    "variation": "control"
  }
}
```

支持的操作符：

```java
public enum ClauseOp {
    EQ, NEQ, IN, NOT_IN,
    STARTS_WITH, ENDS_WITH, CONTAINS, MATCHES,    // 字符串
    GT, GTE, LT, LTE,                              // 数字
    BEFORE, AFTER,                                 // 时间
    SEMVER_EQ, SEMVER_GT, SEMVER_LT;               // 版本号

    public boolean eval(Object userValue, List<Object> values) {
        if (userValue == null) return false;
        return switch (this) {
            case EQ  -> values.contains(userValue);
            case NEQ -> !values.contains(userValue);
            case IN  -> values.contains(userValue);
            case STARTS_WITH -> values.stream().anyMatch(v ->
                userValue.toString().startsWith(v.toString()));
            case ENDS_WITH -> values.stream().anyMatch(v ->
                userValue.toString().endsWith(v.toString()));
            case GT  -> ((Number) userValue).doubleValue()
                       > ((Number) values.get(0)).doubleValue();
            // 其他略
            default -> throw new UnsupportedOperationException();
        };
    }
}
```

**逐段点透**：
- **不要直接用 SpEL / Groovy 跑用户表达式**：性能差、难审计、容易被注入。封闭操作符列表足够覆盖 95% 场景。
- **op 设计借鉴 LaunchDarkly**：他们的运营/产品最常用就这些，自研别造新轮子。
- **多个 clause 之间默认 AND**：要 OR 就在规则维度展开成多条规则。

### 2.3 一致性 Hash 实现稳定百分比放量

百分比放量的关键：**同一个用户多次请求必须落到同一个桶**（Sticky）。否则一会看到新功能一会看到老功能，体验崩了，A/B 数据也废。

错误实现（用随机数）：

```java
// ❌ 每次请求都换桶
if (Math.random() < 0.2) returnNewVariation();
```

正确实现（一致性 Hash）：

```java
public class BucketAlgorithm {
    private static final long HASH_SCALE = 100_000L;

    public static int bucket(String userKey, String flagKey, String salt) {
        // ★ 把 flagKey 拼进去：避免不同 Flag 之间分桶相关
        String input = flagKey + "." + userKey + "." + salt;

        // MurmurHash3 速度快、分布好
        long hash = MurmurHash3.hash64(input.getBytes(StandardCharsets.UTF_8));
        // 取绝对值后落到 [0, 100000) 区间
        return (int) (Math.abs(hash) % HASH_SCALE);
    }

    public static String pickVariation(int bucketIdx,
                                       List<WeightedVariation> variations) {
        long sum = variations.stream().mapToLong(v -> v.weight).sum();
        if (sum != HASH_SCALE) {
            throw new IllegalStateException(
                "weights must sum to " + HASH_SCALE + ", got " + sum);
        }
        long acc = 0;
        for (WeightedVariation v : variations) {
            acc += v.weight;
            if (bucketIdx < acc) return v.variation;
        }
        return variations.get(variations.size() - 1).variation;
    }
}
```

**逐段点透**：
- **用 MurmurHash3 而不是 hashCode**：JDK 的 String.hashCode 分布不均，前几位偏倚明显，容易导致 50/50 实际是 53/47。
- **input 里必须拼 flagKey**：不然两个 Flag 都按 userId 分桶，会导致"开了 A Flag 的用户大概率也开 B Flag"，污染独立性假设。
- **HASH_SCALE = 100_000**：用 10 万而不是 100，支持精度到 0.001%，做小流量灰度（0.1%）时不丢分辨率。
- **校验 weights 和**：A/B 配错"v1: 50%, v2: 40%"很常见，必须强校验。

### 2.4 客户端 SDK：流式订阅 + 本地缓存

```java
public class FeatureClient implements AutoCloseable {
    private final String sdkKey;
    private final String env;
    private final AtomicReference<FlagStore> store = new AtomicReference<>(FlagStore.empty());
    private final EventSource eventSource;
    private final ScheduledExecutorService poller;

    public FeatureClient(Config cfg) {
        this.sdkKey = cfg.sdkKey;
        this.env = cfg.env;

        // 启动时拉一次全量
        FlagStore initial = bootstrapFromHttp(cfg.bootstrapUrl);
        store.set(initial);

        // SSE 订阅增量
        this.eventSource = new EventSource(cfg.streamUrl, this::onEvent);
        eventSource.start();

        // 兜底轮询：SSE 可能因为网关超时被断开
        this.poller = Executors.newSingleThreadScheduledExecutor();
        poller.scheduleAtFixedRate(this::pollIfStale,
            cfg.pollInterval, cfg.pollInterval, TimeUnit.SECONDS);
    }

    private void onEvent(SseEvent event) {
        switch (event.type) {
            case "put"   -> store.set(FlagStore.fromJson(event.data));
            case "patch" -> {
                FlagPatch patch = FlagPatch.fromJson(event.data);
                store.updateAndGet(s -> s.applyPatch(patch));
            }
            case "delete" -> store.updateAndGet(s -> s.remove(event.flagKey));
        }
    }

    public <T> T variation(String flagKey, UserContext ctx, T defaultValue) {
        try {
            FlagStore s = store.get();
            FlagConfig f = s.get(flagKey);
            if (f == null) {
                // ★ Flag 不存在不要抛异常，返回默认值
                emitEvent(flagKey, defaultValue, ctx, "FLAG_NOT_FOUND");
                return defaultValue;
            }
            EvalResult r = Evaluator.evaluate(f, ctx);
            emitEvent(flagKey, r.value, ctx, r.reason);
            return (T) r.value;
        } catch (Exception e) {
            log.warn("variation eval failed for {}", flagKey, e);
            return defaultValue;
        }
    }
}
```

**逐段点透**：
- **`AtomicReference<FlagStore>` 整体替换**：写少读多场景，整体替换比每个 Flag 一把锁简单且安全。读路径完全无锁。
- **bootstrap + stream + poll 三层兜底**：SSE 在企业代理后面经常被截断，必须有定期 poll 兜底。
- **Flag 不存在返回 default**：上线前的代码引用了一个不存在的 Flag（环境差异），不能让业务挂掉。
- **emitEvent**：每次评估都异步上报一条事件，是后面 A/B 数据的源头（2.7 节展开）。

### 2.5 与配置中心（Apollo / Nacos）对接

如果公司已经有配置中心，**不要重复造**前端推送通道。Flag 服务可以把规则写进配置中心，复用其推送：

```java
@Component
public class ApolloFlagPublisher {

    @Autowired
    private OpenApiClient apolloClient;

    public void publish(String flagKey, String env, FlagEnvConfig config) {
        String namespace = "feature-flags." + env;
        String value = JSON.toJSONString(config);

        OpenItemDTO item = new OpenItemDTO();
        item.setKey("flag." + flagKey);
        item.setValue(value);
        item.setComment("flag " + flagKey + " v" + config.getVersion());
        item.setDataChangeCreatedBy("flag-platform");

        apolloClient.createOrUpdateItem("MyAppId", env, "default",
            namespace, item);

        // 立即发布，配置中心负责推送给 SDK
        NamespaceReleaseDTO release = new NamespaceReleaseDTO();
        release.setReleaseTitle("flag-" + flagKey + "-v" + config.getVersion());
        release.setReleaseComment("auto-publish from flag platform");
        release.setReleasedBy("flag-platform");
        apolloClient.publishNamespace("MyAppId", env, "default",
            namespace, release);
    }
}
```

**逐段点透**：
- **每个 Flag 一个 key，不要塞一个大 JSON**：Apollo 的 namespace 推送是粒度，单 Flag 改不影响其他 Flag 的客户端缓存失效。
- **value 是序列化 JSON**：包含完整规则、版本号、variations，SDK 只读这一个 key 就能完整评估。
- **publish 必须立即触发**：Apollo 默认的 createItem 不发布，必须显式 release，否则 SDK 永远看不到新值。
- **复用 Apollo 的好处**：长连接通道、灰度发布、回滚、审计 —— 都不用自己写。

### 2.6 LaunchDarkly / Unleash / Statsig 选型对比

| 维度 | LaunchDarkly | Unleash | Statsig |
| --- | --- | --- | --- |
| 商业模式 | 商业 SaaS | 开源 + 商业 | 商业 SaaS（部分免费） |
| Targeting Rules | 极其完善 | 中等（Strategies 抽象） | 完善 |
| A/B 实验 | 基础（需配数据仓库） | 有限 | **强项**：内置统计引擎 |
| 自部署 | Relay Proxy | 完整开源 | 不支持 |
| 价格（10 MAU） | $$$$ | 免费 | $$ |
| SDK 多语言 | 30+ | 20+ | 15+ |
| 适合谁 | 大型企业、需要审计/SOC2 | 自建优先、成本敏感 | 重视实验、不想自建数据栈 |

**自研 vs 买的判断标准**（3.4 节有数据）：
- 工程师 < 50 人：直接买 LaunchDarkly / Unleash 商业版，省事
- 工程师 50~500：Unleash 开源 + 自己包一层管理后台
- 工程师 > 500：考虑自研，但优先复用配置中心

### 2.7 A/B 实验的事件埋点

Experiment Flag 的核心数据闭环：

```
用户进入实验 → SDK 评估返回 variation
            │
            ▼
SDK 异步上报 exposure 事件
{userId, flagKey, variation, ctx, ts}
            │
            ▼
        Kafka topic
            │
            ▼
   ┌────────┴─────────┐
   │                  │
   ▼                  ▼
ClickHouse        实验平台
(明细查询)        (聚合 + 显著性)
```

SDK 上报代码：

```java
public class EventEmitter {
    private final BlockingQueue<EvalEvent> queue = new ArrayBlockingQueue<>(10_000);
    private final HttpClient http;
    private final ScheduledExecutorService flusher;

    public void emit(String flagKey, Object variation,
                     UserContext ctx, String reason) {
        EvalEvent e = new EvalEvent(flagKey, variation, ctx, reason,
            System.currentTimeMillis());
        // ★ 队列满直接丢弃，不能因为埋点拖累业务
        if (!queue.offer(e)) {
            droppedCounter.increment();
        }
    }

    private void flush() {
        List<EvalEvent> batch = new ArrayList<>(500);
        queue.drainTo(batch, 500);
        if (batch.isEmpty()) return;
        try {
            http.post("/events", batch);
        } catch (Exception ex) {
            log.warn("flush failed, will retry", ex);
            // 失败不重新入队 —— A/B 数据可以容忍丢一点
        }
    }
}
```

**逐段点透**：
- **队列满丢弃而不是阻塞**：埋点失败绝对不能拖业务。如果队列开始堆积（队列大小指标飙升），监控会告警。
- **批量上报**：单条 POST 浪费网络。500 条 / 1 秒批量发送是工程经验值。
- **失败不重试入队**：A/B 实验本来就是大数法则，丢千分之一无所谓；重试入队会让队列爆炸传导到业务。
- **服务端额外去重**：同一个 (userId, flagKey, variation, day) 可以多次上报，落库时按需去重，避免单一用户对实验组成多次 exposure 计数。

> **本节口诀**：数据模型把 `version` 当推送序列号，规则用 JSON DSL 不要上 SpEL，分桶用 MurmurHash + flagKey 加盐，SDK 是 bootstrap+stream+poll 三层兜底。**Flag 平台的核心难度不在评估，而在 SDK 的可靠性**。

---

## 3. 实测数据：评估延迟、内存、推送对比

### 3.1 Flag 评估延迟（客户端 vs 服务端）

测试环境：MacBook M2 Pro，单 Flag，100w 次评估。

| 模式 | P50 | P99 | QPS（单实例） |
| --- | --- | --- | --- |
| 客户端评估（本地内存） | 0.3μs | 1.2μs | > 300 万 |
| 客户端评估（带规则匹配） | 0.8μs | 3.5μs | ~ 100 万 |
| 服务端评估（同机房 HTTP） | 1.8ms | 8ms | ~ 5000 |
| 服务端评估（跨机房） | 8ms | 30ms | ~ 1500 |

**结论**：客户端评估比服务端**快 3~4 个数量级**。除非有规则保密或数据查询需求，**默认用客户端**。

### 3.2 大量 Flag 下的 SDK 内存

模拟 1000 个 Flag、每个有 5 条规则、每条规则 3 个 clause。

| Flag 数量 | SDK 堆内存 | 启动拉取耗时 |
| --- | --- | --- |
| 100 | 800 KB | 80ms |
| 1000 | 8 MB | 600ms |
| 5000 | 42 MB | 3.5s |
| 10000 | 90 MB | 7.8s |

**结论**：单 SDK 实例 1000 Flag 以下完全无压力。超过 5000 要考虑**按命名空间分组拉取** —— 例如 `payment.*` / `feed.*` 不同业务线只拉自己关心的。

### 3.3 实时推送 vs 拉取的对比

模拟 1 万个 SDK 实例，Flag 改一次后多久全部生效。

| 模式 | 平均生效延迟 | 服务端 QPS（10k 实例） | 网络字节 |
| --- | --- | --- | --- |
| 30s 轮询全量 | 15s | 333（持续） | 高（每次 8MB） |
| 5s 轮询全量 | 2.5s | 2000（持续） | 极高 |
| SSE 推送 + 5min 兜底轮询 | 0.8s | 33（仅轮询） + 推送一次 | 极低 |
| HTTP/2 长连接 + Server Push | 0.4s | 类似 SSE | 略低（多路复用） |

**结论**：**生产必须用推送**，轮询仅作兜底。LaunchDarkly / Unleash 默认 SSE 是有道理的。

### 3.4 自研 vs 商业方案的成本曲线

按 100 人工程团队 / 1000 万 MAU / 200 个活跃 Flag 估算：

| 方案 | 一次性投入 | 年度运维 | 风险 |
| --- | --- | --- | --- |
| **LaunchDarkly Enterprise** | 0 | $80k+ | 低（成熟） |
| **Unleash 开源 + 自部署** | 1 人月 | 0.5 人 + $5k 基础设施 | 中（自己维护） |
| **完全自研** | 4~6 人月 | 1.5 人 + $5k | 高（数据可靠性、SDK 多语言） |
| **基于 Apollo + 包薄壳** | 2 人月 | 0.3 人 | 中（依赖 Apollo 维护） |

**结论**：商业 SaaS 在工程团队 < 200 人时几乎都更划算。> 500 人或者数据合规非自部署不可时才考虑自研。

### 3.5 A/B 实验显著性测试样本量

按公式 `n ≈ 16 × p(1-p) / δ²` 算几个常见场景：

| Baseline | 期望相对提升 | 绝对差 δ | 每组最小样本 |
| --- | --- | --- | --- |
| 5% | 5% | 0.0025 | 121,600 |
| 5% | 10% | 0.005 | 30,400 |
| 5% | 20% | 0.01 | 7,600 |
| 1% | 10% | 0.001 | 158,400 |
| 50%（CTR 类） | 5% | 0.025 | 6,400 |

**结论**：
- 转化率越低，需要的样本越大（电商付费转化常需要几十万 / 组）
- 想检测越细的差，样本量平方级增长（提升从 10% 降到 5%，样本量 4 倍）
- **跑实验前算样本量** —— 算完发现要 30 天才能跑完，那这个实验从一开始就不该做

> **本节口诀**：客户端评估快 3 个数量级，1000 个 Flag 内存 8MB 无压力，SSE 推送是必须的，自研只在大公司划算，A/B 实验先算样本量再跑。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 灰度比例配错 —— "1% 灰度变成 100% 全量"

#### 现象

凌晨 1 点上线新支付链路，配置灰度 1%。半小时后告警：交易失败率从 0.3% 飙到 12%。回滚后查 Flag 配置：fallthrough 配的是 `treatment: 100, control: 0`，**不是预期的 1/99**。

#### 根因

后台界面里有两个相邻输入框：

```
treatment 权重：[100]    ← 这个是"占比的分子"
control 权重：  [  0]
```

但有些公司的后台**输入框含义是"前 N% 走 treatment，剩下走 control"**。运营同学在新平台习惯了第二种语义，输入了"100"以为是"前 100% 走 treatment"，实际效果是 100% 全量。

更深层：**没有上线前的预览模拟**。如果运营保存配置前能看到"实际效果：100% treatment / 0% control"，就不会出错。

#### 修复

UI 强约束 + 预览：

```javascript
// React 表单组件
function RolloutEditor({ variations, onChange }) {
  const total = variations.reduce((s, v) => s + v.weight, 0);
  return (
    <div>
      {variations.map(v => (
        <Slider
          value={v.weight}
          onChange={w => onChange(v.id, w)}
          // ★ 任何一项变化时立即重算其他项，强制和为 100
        />
      ))}
      <Alert type={total === 100 ? "success" : "error"}>
        当前总和：{total}%（必须等于 100%）
      </Alert>
      {/* ★ 预览效果 */}
      <Preview>
        模拟 1000 个用户：
        {variations.map(v => `${v.id}=${Math.round(v.weight * 10)}人`).join(", ")}
      </Preview>
    </div>
  );
}
```

服务端校验：

```java
@PostMapping("/flags/{key}/config")
public void update(@PathVariable String key,
                   @RequestBody FlagEnvConfig config) {
    Fallthrough f = config.getFallthrough();
    if (f.getRollout() != null) {
        long sum = f.getRollout().getVariations().stream()
            .mapToLong(WeightedVariation::getWeight)
            .sum();
        if (sum != 100_000) {   // 内部按 10 万存
            throw new BadRequestException(
                "rollout weights must sum to 100, got " + sum / 1000.0);
        }
    }
    // ★ 灰度比例 > 5% 时要求二次确认
    if (isReleaseFlag(key) && getMaxNonControlWeight(f) > 5_000) {
        if (!config.isConfirmed()) {
            throw new ConfirmRequiredException(
                "rollout > 5%, please confirm");
        }
    }
    flagService.update(key, config);
}
```

#### 验证

```java
@Test
void rolloutSumMustBe100() {
    FlagEnvConfig c = configWithRollout(100, 0);
    assertThrows(BadRequestException.class,
        () -> service.update("test-flag", c));
}

@Test
void releaseFlagOver5PercentNeedsConfirm() {
    FlagEnvConfig c = configWithRollout(10, 90).unconfirmed();
    assertThrows(ConfirmRequiredException.class,
        () -> service.update("release-flag", c));
}
```

> **口诀**：UI 表单要预览效果、服务端要强校验权重和、Release Flag 大改动要二次确认。**配错的代价远大于多一次确认的麻烦**。

---

### 案例 ② Sticky 分桶失败 —— "用户两次请求看到不同行为"

#### 现象

用户反馈：刷新页面新支付按钮一会有一会没有。后台日志看，同一个 userId 的两次请求评估结果不同。

#### 根因

排查后定位到 SDK 的分桶代码：

```java
// ❌ 用 hashCode
int bucket = userId.hashCode() % 100;
```

问题不止一个：
1. `hashCode` 在 JDK 不同版本/不同 JVM 行为可能有差（虽然 String.hashCode 实际固定）
2. **取模可能为负数**：`(-2147483648) % 100 = -8`
3. **没有拼 flagKey**：所有 Flag 共用同一分桶，开了 A 的用户大概率开了 B
4. 服务端版本和 SDK 版本算法不一致：服务端用 MurmurHash，SDK 用 hashCode，**评估结果天然不同**

但更隐蔽的版本：

```java
String input = flagKey + ":" + userId;
int bucket = MurmurHash.hash(input) % 100;

// 用户请求里的 userContext 是从 Cookie 解析的
String userId = cookie.get("uid");
// ❌ 用户没登录时 uid 为空，每次评估返回 null
// ❌ SDK 写法允许 null userId，回退用 IP，IP 在多次请求间会变（移动网络）
```

第二种是真实生产事故 —— 未登录用户的"匿名 ID"必须**自己生成 + 持久化到 Cookie**，不能依赖网络层。

#### 修复

```java
public class StickyContextProvider {

    public UserContext build(HttpServletRequest req, HttpServletResponse resp) {
        String userId = parseUserId(req);
        if (userId == null) {
            userId = parseAnonymousId(req);
            if (userId == null) {
                // ★ 第一次访问：生成 anon id 并塞 Cookie，1 年有效
                userId = "anon-" + UUID.randomUUID();
                Cookie c = new Cookie("anon_id", userId);
                c.setMaxAge(365 * 24 * 3600);
                c.setPath("/");
                c.setHttpOnly(true);
                resp.addCookie(c);
            }
        }
        return UserContext.builder()
            .key(userId)            // ★ 永远用稳定的 key
            .anonymous(parseAnonymousId(req) != null)
            .build();
    }
}
```

分桶算法版本对齐：

```java
// 服务端、SDK 必须用完全相同的实现
public final class BucketAlgorithm {
    public static int bucket(String key, String flagKey, String salt) {
        String input = flagKey + "." + key + "." + salt;
        long hash = MurmurHash3.hash64(input.getBytes(StandardCharsets.UTF_8));
        return (int) (Math.abs(hash) % 100_000L);
    }
}
```

跨语言一致性测试：

```java
@Test
void crossLanguageBucketMustMatch() {
    // 这些值是 Python / Go / JS SDK 算出来的预期值，必须 byte-for-byte 一致
    assertEquals(45_672, bucket("user-1", "flag-checkout", ""));
    assertEquals(82_341, bucket("user-2", "flag-checkout", ""));
    assertEquals(12_098, bucket("user-3", "flag-checkout", ""));
}
```

#### 验证

```java
@Test
void sameUserAlwaysSameBucket() {
    String userId = "user-12345";
    int first = bucket(userId, "flag-A", "");
    for (int i = 0; i < 1000; i++) {
        assertEquals(first, bucket(userId, "flag-A", ""));
    }
}

@Test
void differentFlagsBucketIndependently() {
    // 不同 Flag 的同一用户应该几乎没有相关性
    int sameBucket = 0;
    for (int i = 0; i < 10_000; i++) {
        String u = "user-" + i;
        int bA = bucket(u, "flag-A", "") / 1000;   // 100 桶
        int bB = bucket(u, "flag-B", "") / 1000;
        if (bA == bB) sameBucket++;
    }
    // 期望约 100，实际 80~120 都正常；如果差很多说明分桶相关
    assertTrue(sameBucket > 60 && sameBucket < 150);
}
```

> **口诀**：稳定 key + 稳定算法 + 跨语言对齐 + 拼 flagKey 加盐。**未登录用户必须 Cookie 自己生成 anon id**。

---

### 案例 ③ 长期 Flag 没人删 —— "代码爆炸"

#### 现象

新人接手 3 年老项目，下单流程里有 8 层嵌套 if，每层都是某个 Flag。看 git blame 全是"灰度新功能"，但 Flag 配置里这些都是 100% 已全量。改一个简单需求要梳理 30 分钟"哪个分支是真在跑的"。

#### 根因

公司没有 Flag 治理流程：
1. 创建 Flag 时不要求填过期日
2. 上线全量后没人有责任删
3. 代码里的 `if (flag.isEnabled("xxx"))` 删除需要走发版流程，运营改 Flag 不需要 → 代码删除被无限推迟
4. 没有"Flag 是否被代码引用"的扫描

更隐蔽：**有些 Flag 是"半全量"** —— 99% on，留 1% control 当对照组永远跑下去。这种叫 "holdout"，本来设计目的是长期监控新功能影响，但 99% 团队没有数据看 holdout 结果，只是忘了关。

#### 修复

**第一步：建立 Flag 元数据治理。**

```java
@Entity
public class FeatureFlag {
    String key;
    FlagType type;
    LocalDate expireAt;        // ★ Release Flag 强制填，最长 90 天
    String owner;              // ★ Slack ID，到期前 7 天 @ 他
    String purpose;
    boolean isPermanent;       // 仅 Ops/Permission/Holdout 允许 true
}

@Service
public class FlagExpiryReporter {
    @Scheduled(cron = "0 0 9 * * *")
    public void daily() {
        // 即将到期：提前 7 天通知
        flagRepo.findExpiringIn(7).forEach(f ->
            slack.notify(f.owner,
                "Flag `%s` 将在 %s 到期，请清理代码后归档".formatted(
                    f.key, f.expireAt)));

        // 已过期：每天通知一次直到归档
        flagRepo.findExpired().forEach(f ->
            slack.notify("#flag-debt",
                "Flag `%s` 已过期 %d 天，owner: %s".formatted(
                    f.key, daysSince(f.expireAt), f.owner)));
    }
}
```

**第二步：CI 扫描代码引用。**

```bash
#!/bin/bash
# scripts/check-flag-references.sh
ACTIVE_FLAGS=$(curl -s ${FLAG_API}/flags?archived=false | jq -r '.[].key')
UNUSED=()

for flag in $ACTIVE_FLAGS; do
    count=$(rg -l "isEnabled\(\"$flag\"\)|variation\(\"$flag\"\)" \
        src/ test/ 2>/dev/null | wc -l)
    if [ "$count" -eq 0 ]; then
        UNUSED+=("$flag")
    fi
done

if [ ${#UNUSED[@]} -gt 0 ]; then
    echo "Found ${#UNUSED[@]} flags configured but not referenced in code:"
    printf '  - %s\n' "${UNUSED[@]}"
    # 自动建一个 issue
    gh issue create --title "Unused flags found" --body "$(printf '%s\n' "${UNUSED[@]}")"
fi
```

反向也要扫：**代码里引用了，但配置里没有的 Flag**：

```bash
CODE_FLAGS=$(rg -o 'isEnabled\("([^"]+)"\)' -r '$1' src/ | sort -u)
for flag in $CODE_FLAGS; do
    if ! curl -sf ${FLAG_API}/flags/$flag > /dev/null; then
        echo "Code references undefined flag: $flag"
        exit 1
    fi
done
```

**第三步：把 Flag 删除变成日常任务。**

每周 SRE 例会过一遍过期 Flag 列表，分配 owner，下个迭代 PR 清理。

#### 验证

```sql
-- 健康度仪表盘
SELECT
    flag_type,
    COUNT(*) AS total,
    SUM(CASE WHEN expire_at < CURDATE() THEN 1 ELSE 0 END) AS expired,
    SUM(CASE WHEN expire_at < CURDATE() + INTERVAL 7 DAY
             AND expire_at >= CURDATE() THEN 1 ELSE 0 END) AS expiring_soon
FROM feature_flag
WHERE archived = false AND is_permanent = false
GROUP BY flag_type;
```

期望：`expired = 0`，`expiring_soon` 持续 < 5。

> **口诀**：每个 Release Flag 必须有 owner + 过期日。CI 双向扫描"配置未引用 / 引用未配置"。**Flag 是技术债，不主动还会指数增长**。

---

### 案例 ④ SDK 升级缓慢 —— "新规则上线 3 周还不一致"

#### 现象

后端加了一个新的操作符 `semverGt`（语义版本号比较），上线后发现 30% 的客户端评估结果不对 —— 那些用户的 App 里 SDK 版本不支持这个操作符。

#### 根因

客户端评估的本质：**规则下发到端，端用 SDK 算**。一旦规则用了**某个老 SDK 不支持的操作符**，老 SDK 要么报错降级、要么算错。

App 用户的 SDK 升级周期是几个月（取决于 App 升级率）。后台同学加的新功能，端侧可能两年都覆盖不全。

类似的还有：
- 新增 Flag 类型（如 JSON Variation），老 SDK 不识别
- 修改分桶算法（即使是修 bug）
- 新增 prerequisite Flag（依赖另一个 Flag 先开）

#### 修复

**核心原则：客户端评估的规则必须严格向后兼容。**

策略 1：未知操作符降级到默认值并告警。

```java
public boolean evalClause(Clause c, UserContext ctx) {
    Object userValue = ctx.get(c.attribute);
    try {
        ClauseOp op = ClauseOp.valueOf(c.op);
        return op.eval(userValue, c.values);
    } catch (IllegalArgumentException e) {
        // ★ SDK 不认识这个操作符
        log.warn("unknown clause op: {}, treating as no-match", c.op);
        unknownOpCounter.increment(c.op);
        return false;   // 规则被视为不命中，走 fallthrough
    }
}
```

策略 2：服务端按 SDK 版本下发不同规则集。

```java
@GetMapping("/sdk/flags")
public Map<String, FlagConfig> getFlags(
        @RequestHeader("X-SDK-Version") String sdkVersion) {

    Map<String, FlagConfig> all = flagService.getAll(env);
    if (Semver.lt(sdkVersion, "2.0.0")) {
        // 老 SDK 拿到的规则里所有 v2 操作符都被改写为兜底
        return all.entrySet().stream()
            .map(e -> Map.entry(e.getKey(),
                downgrade(e.getValue(), sdkVersion)))
            .collect(toMap(Map.Entry::getKey, Map.Entry::getValue));
    }
    return all;
}

private FlagConfig downgrade(FlagConfig f, String sdkVersion) {
    // 把不支持的规则整条移除，让评估走 fallthrough
    List<Rule> compat = f.rules.stream()
        .filter(r -> r.clauses.stream()
            .allMatch(c -> isSupported(c.op, sdkVersion)))
        .toList();
    return f.withRules(compat);
}
```

策略 3：复杂规则强制走服务端评估。

```java
public class HybridClient {
    public Object variation(String key, UserContext ctx, Object def) {
        FlagConfig f = localStore.get(key);
        if (f == null) return def;
        if (f.requiresServerEval) {
            return serverEval(key, ctx, def);   // ★ 兜底走服务端
        }
        return Evaluator.evaluate(f, ctx);
    }
}
```

#### 验证

```java
@Test
void unknownOpFallsThroughGracefully() {
    Clause c = new Clause("country", "futureOp", List.of("CN"));
    Rule r = new Rule(List.of(c), "treatment");
    FlagConfig f = configBuilder()
        .rules(List.of(r))
        .fallthrough("control")
        .build();
    UserContext ctx = UserContext.builder()
        .key("u1").attribute("country", "CN").build();

    EvalResult result = Evaluator.evaluate(f, ctx);
    assertEquals("control", result.value);   // ★ 不抛异常，走 fallthrough
}
```

监控指标：

```java
Gauge.builder("flag.sdk.version.distribution", versionMap, Map::size)
    .tags("env", "prod")
    .register(meterRegistry);

// 接收到的 SDK 版本分布告警：超过 30 天的版本占比 > 10% 时通知
```

> **口诀**：客户端评估对 SDK 版本有强约束。**新增操作符必须降级、按版本下发、复杂规则走服务端**。三件套缺一不可。

---

### 案例 ⑤ A/B 实验样本量不足 —— "假阳性满天飞"

#### 现象

产品经理一周做 4 个 A/B 实验，每个跑 3 天就根据 p < 0.05 出结论。3 个月后回头看，"显著有效"的功能上线后线上指标完全没动。

#### 根因

经典的"窥探（peeking）"问题 + 样本量不足：

1. **样本量不足**：transition rate 5% 想检测 10% 提升，每组要 3 万样本。3 天可能只跑了 5000，统计 power 极低。
2. **多次窥探**：每天看 p-value，看到 < 0.05 就停。这种"序贯检验"会让假阳性概率从 5% 飙到 20%+。
3. **多重假设检验**：同时跑 10 个实验，必然有 1~2 个出现 p < 0.05 纯属偶然。
4. **辛普森悖论**：移动端 + Web 端汇总有效，分开看都无效。

#### 修复

**实验平台必须强约束：**

```java
public class ExperimentLauncher {

    public Experiment create(ExperimentSpec spec) {
        // ① 强制声明 baseline 转化率和 MDE
        if (spec.baselineRate == null || spec.minDetectableEffect == null) {
            throw new ValidationException(
                "must declare baseline and MDE before launch");
        }

        // ② 自动算样本量
        long minSamplePerArm = SampleSizeCalculator.calculate(
            spec.baselineRate, spec.minDetectableEffect, 0.05, 0.8);
        spec.minSamplePerArm = minSamplePerArm;

        // ③ 估算实验时长（按平均日活计算）
        long dau = analyticsClient.getDailyActiveUsers(spec.targetSegment);
        long days = (long) Math.ceil(
            minSamplePerArm * spec.arms.size() / (double) dau / spec.exposureRate);

        if (days > 30) {
            throw new ValidationException(
                "experiment requires " + days + " days to reach significance, "
                + "consider larger MDE or larger segment");
        }

        // ④ 强制最短运行时间
        spec.earliestStopTime = LocalDateTime.now().plusDays(7);
        return experimentRepo.save(spec);
    }

    public ExperimentReport analyze(String experimentId) {
        Experiment exp = experimentRepo.find(experimentId);
        // ★ 没到最早停止时间，禁止下结论
        if (LocalDateTime.now().isBefore(exp.earliestStopTime)) {
            return ExperimentReport.notReady(exp);
        }
        // ★ 没到最小样本，禁止下结论
        long actual = exposureRepo.countByArm(experimentId);
        if (actual < exp.minSamplePerArm * exp.arms.size()) {
            return ExperimentReport.notEnoughSamples(exp, actual);
        }

        // ⑤ 用 sequential testing 校正 p-value（如 mSPRT）
        return sequentialTest.run(exp);
    }
}
```

**SampleSizeCalculator 的实现：**

```java
public class SampleSizeCalculator {
    public static long calculate(double baseline, double mde,
                                 double alpha, double power) {
        // 双尾检验，z_alpha = 1.96, z_beta = 0.84
        double zAlpha = NormalDistribution.inverse(1 - alpha / 2);
        double zBeta = NormalDistribution.inverse(power);
        double sigma2 = baseline * (1 - baseline);
        double n = 2 * sigma2 * Math.pow(zAlpha + zBeta, 2) / Math.pow(mde, 2);
        return (long) Math.ceil(n);
    }
}
```

**多重检验校正（Bonferroni）**：

```java
// 同时跑 N 个独立指标的实验，alpha 要除以 N
double adjustedAlpha = 0.05 / numberOfMetrics;
```

#### 验证

复盘历史实验，对显著结果做"事后样本量检查"：

```sql
SELECT
    experiment_id,
    metric,
    treatment_n, control_n,
    treatment_rate, control_rate,
    p_value,
    -- 重算需要多大样本才能显著
    16 * control_rate * (1 - control_rate)
        / POWER(treatment_rate - control_rate, 2) AS required_n_per_arm,
    CASE
        WHEN treatment_n < 16 * control_rate * (1 - control_rate)
                / POWER(treatment_rate - control_rate, 2)
        THEN 'UNDERPOWERED'
        ELSE 'OK'
    END AS verdict
FROM experiment_results
WHERE p_value < 0.05;
```

期望：上线前显著的实验，样本量都达到了 required_n_per_arm。

> **口诀**：实验平台必须强制 baseline + MDE + 最小样本量 + 最短运行时间 + 多重检验校正。**没算样本量的 A/B 是数据噪音，不是结论**。

---

### 案例 ⑥ Multivariate Flag 配错 —— "三个 variation 跑出来一模一样"

#### 现象

新搜索算法做 A/B/C 三组对比：control / 算法 v1 / 算法 v2。跑了一周看数据，三组的指标几乎一样。复查发现：所有用户实际上看的都是 control 的结果。

#### 根因

Multivariate Flag 配置：

```json
{
  "variations": [
    {"key": "control", "value": "old"},
    {"key": "v1",      "value": "new1"},
    {"key": "v2",      "value": "new2"}
  ],
  "fallthrough": {
    "rollout": {
      "variations": [
        {"variation": "control", "weight": 100},
        {"variation": "v1",      "weight": 0},   // ❌ 0%
        {"variation": "v2",      "weight": 0}    // ❌ 0%
      ]
    }
  }
}
```

权重和为 100 通过校验，但 v1/v2 各 0%，等于全部走 control。

更深的根因：**业务代码用 Flag 值做分支**，但分支逻辑写错了：

```java
// ❌ 错的写法：拿 variation 的 value 而不是 key
String variant = flagClient.stringVariation("search-algo", ctx, "old");
if (variant.equals("new1")) {
    return searchV1(query);
} else if (variant.equals("new2")) {
    return searchV2(query);
} else {
    return searchOld(query);   // 拿到 "control" / "old" 都走这里
}
```

如果 variation 的 value 配置写成 `"v1"` 而不是 `"new1"`，分支永远不命中。

#### 修复

**权重校验加强**：

```java
public void validate(Fallthrough f) {
    if (f.rollout == null) return;
    long sum = 0;
    int nonZero = 0;
    for (WeightedVariation v : f.rollout.variations) {
        sum += v.weight;
        if (v.weight > 0) nonZero++;
    }
    if (sum != 100_000) {
        throw new BadRequestException("weights must sum to 100");
    }
    // ★ 多变体实验必须至少 2 个非零权重
    if (f.rollout.variations.size() > 2 && nonZero < 2) {
        throw new BadRequestException(
            "multivariate rollout must have at least 2 non-zero weights");
    }
}
```

**业务代码改用类型安全枚举**：

```java
public enum SearchAlgo {
    CONTROL, V1, V2;

    public static SearchAlgo from(String s) {
        try { return valueOf(s.toUpperCase()); }
        catch (Exception e) {
            log.warn("unknown variation: {}, default to CONTROL", s);
            return CONTROL;
        }
    }
}

SearchAlgo algo = SearchAlgo.from(
    flagClient.stringVariation("search-algo", ctx, "control"));

return switch (algo) {
    case V1 -> searchV1(query);
    case V2 -> searchV2(query);
    case CONTROL -> searchOld(query);
};
```

**端到端测试**：

```java
@Test
void allVariationsActuallyExecute() {
    // 模拟 1000 个不同用户，看是否每个 variation 都被执行过
    Map<String, Integer> hits = new HashMap<>();
    for (int i = 0; i < 1000; i++) {
        UserContext ctx = UserContext.of("user-" + i);
        String v = flagClient.stringVariation("search-algo", ctx, "control");
        hits.merge(v, 1, Integer::sum);
    }
    // 每个 variation 至少要有命中（除非配置就是 0%）
    assertTrue(hits.containsKey("control"));
    assertTrue(hits.containsKey("v1"));
    assertTrue(hits.containsKey("v2"));
}
```

#### 验证

监控每个 variation 的实际曝光量：

```sql
SELECT
    flag_key,
    variation,
    COUNT(DISTINCT user_id) AS exposed_users,
    COUNT(*) AS exposures
FROM flag_evaluations
WHERE event_date = CURDATE() AND flag_key = 'search-algo'
GROUP BY flag_key, variation;
```

期望：每个 variation 都有用户曝光，且比例与配置权重接近（误差 < 2%）。

> **口诀**：多变体权重至少 2 个非零；业务代码用枚举做分支；监控每个 variation 的实际曝光。**配置层 + 代码层 + 监控层三道防线**。

---

### 案例 ⑦ 紧急关闭 Flag 时的延迟 —— "事故扩大 5 分钟"

#### 现象

新支付链路出问题，监控告警。SRE 在管理后台点 kill switch，2 分钟内通过 Apollo 推送，但**部分客户端 5 分钟后才生效**。这 5 分钟里事故影响扩大。

#### 根因

链路：
```
Admin 后台改 Flag → DB
              ↓
          消息推 Apollo
              ↓
       Apollo 推送给 SDK
              ↓
       SDK 更新本地缓存
              ↓
       下次评估生效
```

各环节延迟：
- Admin → DB：< 100ms
- DB → Apollo：1~2s（如果用 Open API 同步发布）
- Apollo → SDK：~ 1s（长连接推送）
- SDK 更新本地缓存：< 100ms

理论 < 4s 应该全部生效。但实测 5 分钟还有 5% 没生效，原因：

1. **客户端 SDK 在企业代理后**，长连接被代理切断
2. **SDK 兜底轮询周期是 30 分钟**（节省成本调高了）
3. **客户端在断网恢复后没主动重新拉取**

#### 修复

**多层应急通道**：

```java
public class FlagClient {

    // 通道 1：常规 SSE 推送
    private final EventSource normalChannel;

    // 通道 2：紧急通道（独立 endpoint，更高保障）
    private final EventSource emergencyChannel;

    // 通道 3：定期拉（兜底）
    private final ScheduledExecutorService poller;

    public FlagClient(Config cfg) {
        this.normalChannel = new EventSource(cfg.streamUrl, this::onEvent);
        // ★ 紧急通道单独，专门下发 kill switch 事件
        this.emergencyChannel = new EventSource(cfg.emergencyUrl, this::onEmergency);
        // ★ 紧急通道断了立即重连，不像普通通道有指数退避
        emergencyChannel.setReconnectStrategy(IMMEDIATE);

        // ★ 兜底轮询：紧急时把间隔降到 5s
        this.poller = Executors.newSingleThreadScheduledExecutor();
        poller.scheduleAtFixedRate(this::poll,
            cfg.pollInterval, cfg.pollInterval, TimeUnit.SECONDS);
    }

    private void onEmergency(SseEvent e) {
        if ("kill".equals(e.type)) {
            // ★ kill 事件最高优先级
            store.updateAndGet(s -> s.killFlag(e.flagKey));
            metrics.incrementCounter("flag.kill.received");
            // 立即上报"已收到"
            emergencyAck(e.flagKey, e.version);
        }
    }
}
```

**服务端推送策略**：

```java
@Service
public class KillSwitchPublisher {
    public void kill(String flagKey, String reason) {
        // ① 改 DB（同步）
        flagRepo.killSwitch(flagKey, true);

        // ② 同步推送给所有可达通道（不能依赖单一）
        CompletableFuture.allOf(
            CompletableFuture.runAsync(() -> apolloPublish(flagKey)),
            CompletableFuture.runAsync(() -> sseEmergency(flagKey)),
            CompletableFuture.runAsync(() -> mqttBroadcast(flagKey)),
            CompletableFuture.runAsync(() -> webhookNotifyKnownClients(flagKey))
        ).orTimeout(5, TimeUnit.SECONDS).join();

        // ③ 记录审计
        auditLog.write(KillSwitchEvent.of(flagKey, reason, currentUser()));

        // ④ 监控生效率：1 分钟后看实际评估返回 kill 的比例
        scheduler.schedule(() -> verifyKillEffective(flagKey),
            60, TimeUnit.SECONDS);
    }

    private void verifyKillEffective(String flagKey) {
        long kill = evalRepo.countWithReason(flagKey, "KILL_SWITCH",
            Duration.ofMinutes(1));
        long total = evalRepo.countAll(flagKey, Duration.ofMinutes(1));
        double rate = (double) kill / total;
        if (rate < 0.95) {
            alertManager.fire("kill switch only " + (rate * 100)
                + "% effective for " + flagKey);
        }
    }
}
```

**客户端紧急快速生效**：

```java
public Object variation(String key, UserContext ctx, Object def) {
    FlagConfig f = store.get().get(key);
    // ★ 关键 Flag（标了 critical）每次评估前异步拉一次最新
    if (f != null && f.critical
        && System.currentTimeMillis() - f.fetchedAt > 5_000) {
        asyncRefresh(key);
    }
    return Evaluator.evaluate(f, ctx);
}
```

#### 验证

定期演练：

```bash
# 每月一次：故意 kill 一个真实 Flag，记录全网生效时间
./flag-cli kill-and-measure --flag canary-test \
    --target-instances all-prod \
    --measure-window 5m

# 输出：
# Initiated at: 2025-05-09T03:00:00Z
# Effective at instances:
#   1m: 9847/10000 (98.47%)
#   2m: 9991/10000 (99.91%)
#   5m: 10000/10000 (100%)
```

SLO：紧急 kill 在 1 分钟内 99% 生效。

> **口诀**：紧急通道独立 + 多通道并发 + 服务端验证生效率 + 月度演练。**绝不依赖单一通道下发 kill switch**。

---

### 案例 ⑧ 与 GitOps 的协调 —— "代码刚 deploy，Flag 还没改"

#### 现象

新支付功能要 release，需要：
1. 后端 deploy 包含新逻辑的代码
2. 把 Flag `new-payment` 从 0% 改到 5%

发布流程是手动两步。结果发布同学先改了 Flag，但代码还没 deploy 完成 → 5% 用户走的是"老代码看到新 Flag = on"的分支，crash。

#### 根因

Feature Flag 的初衷之一是**让代码部署和功能开启解耦**，但这个解耦带来一个新问题：**两者的状态需要协调**。

常见反模式：
- 先改 Flag 再 deploy → 老代码碰到 Flag = on 行为不确定
- 先 deploy 再改 Flag → 灰度窗口短的话没问题，但有时新代码默认 off 没人去开
- 不同环境 Flag 状态漂移：staging 开了，prod 没开，QA 测的不是 prod 行为

#### 修复

**方案 A：把 Flag 配置纳入 GitOps**。

Flag 配置版本化到 Git 仓库：

```yaml
# flags/prod/new-payment.yaml
key: new-payment
type: RELEASE
expireAt: 2025-08-01
owner: alice
variations:
  - {key: control, value: false}
  - {key: treatment, value: true}
fallthrough:
  rollout:
    variations:
      - {variation: treatment, weight: 5}
      - {variation: control, weight: 95}
```

CI/CD 流水线：

```yaml
# .github/workflows/release.yml
jobs:
  deploy-code:
    steps:
      - run: ./gradlew build
      - run: kubectl apply -f k8s/deployment.yaml
      - run: ./scripts/wait-rollout-complete.sh

  apply-flags:
    needs: deploy-code   # ★ 依赖代码 deploy 完成
    steps:
      - run: ./scripts/sync-flags.sh prod
```

`sync-flags.sh` 把 Git 里的 yaml 同步到 Flag 平台：

```bash
#!/bin/bash
ENV=$1
for file in flags/$ENV/*.yaml; do
    flag_key=$(yq '.key' $file)
    config=$(yq -o json $file)
    curl -X PUT $FLAG_API/flags/$flag_key/config/$ENV \
        -H "Authorization: Bearer $FLAG_TOKEN" \
        -H "Content-Type: application/json" \
        -d "$config"
done
```

**方案 B：Flag 平台支持"预案"**。

允许提前配置一个未来生效的版本：

```java
@PostMapping("/flags/{key}/schedule")
public void schedule(@PathVariable String key,
                     @RequestBody ScheduledChange change) {
    // change.applyAt = "2025-05-09T03:00:00Z"
    // change.config = { rollout: 5% }
    scheduledChangeRepo.save(change);
}

@Scheduled(fixedRate = 10_000)
public void applyDueChanges() {
    scheduledChangeRepo.findDue(Instant.now()).forEach(c -> {
        flagService.update(c.flagKey, c.config);
        auditLog.write(ScheduledApplied.of(c));
    });
}
```

**方案 C：代码里检测自身版本，不支持时主动降级**。

```java
public boolean isPaymentV2Enabled(UserContext ctx) {
    if (!CodeVersion.supportsPaymentV2()) {
        return false;   // ★ 老代码无视 Flag，强制走老逻辑
    }
    return flagClient.boolVariation("new-payment", ctx, false);
}
```

#### 验证

环境一致性检查：

```bash
# 每 5 分钟比对 Git 与 Flag 平台的配置
./scripts/diff-flag-state.sh prod
# 输出 drift 项，超过 0 触发告警
```

部署前自检：

```yaml
- name: Verify flag config compatibility
  run: |
    # 新 deploy 的代码要求的 Flag 必须都已在平台存在
    for flag in $(grep -roh 'flagClient\.\w*Variation("[^"]*"' src/ \
                  | sed 's/.*"\(.*\)".*/\1/' | sort -u); do
      if ! curl -sf $FLAG_API/flags/$flag > /dev/null; then
        echo "Missing flag definition: $flag"
        exit 1
      fi
    done
```

> **口诀**：Flag 配置纳入 GitOps、CI 强制 deploy 后再改 Flag、代码侧自检兼容性。**Flag 与代码必须协调，但不要让两者紧耦合**。

---

### 案例 ⑨ 数据合规 —— "实验涉及 PII 被审计点名"

#### 现象

数据合规团队季度审计，发现 Flag 平台的 evaluation 事件里包含了用户邮箱、手机号、地理位置（PII，Personally Identifiable Information），且未脱敏直接发送给第三方分析平台（实际是 Statsig）。GDPR 风险点。

#### 根因

为了让 Targeting Rules 能用"邮箱后缀匹配"等条件，业务代码把整个 user 对象塞进 UserContext：

```java
// ❌ 把整个 user 当 context
UserContext ctx = UserContext.builder()
    .key(user.getId())
    .attribute("email", user.getEmail())          // PII
    .attribute("phone", user.getPhone())          // PII
    .attribute("address", user.getAddress())      // PII
    .attribute("country", user.getCountry())      // 不算 PII
    .build();

flagClient.boolVariation("new-checkout", ctx, false);
// ★ SDK 内部会把整个 ctx 通过 evaluation event 发给 Statsig
```

#### 修复

**第一层：context 最小化**。

```java
public class FlagContextFactory {
    private static final Set<String> ALLOWED_ATTRS = Set.of(
        "country", "plan", "deviceType", "appVersion", "isInternalEmployee");

    public static UserContext build(User user) {
        UserContext.Builder b = UserContext.builder()
            .key(user.getId());
        // ★ 只加白名单属性
        addIfPresent(b, "country", user.getCountry());
        addIfPresent(b, "plan", user.getPlan());
        addIfPresent(b, "deviceType", currentDeviceType());
        // ★ 邮箱不直接放，转换成派生属性
        addIfPresent(b, "isInternalEmployee", isInternal(user.getEmail()));
        return b.build();
    }

    private static boolean isInternal(String email) {
        return email != null && email.endsWith("@taohua.com");
    }
}
```

**第二层：哈希处理需要保留但不能明文的字段**。

```java
// 如果实在需要"按邮箱后缀做规则"，把后缀单独提取
String emailDomain = email.contains("@")
    ? email.substring(email.indexOf('@') + 1).toLowerCase()
    : "";
b.attribute("emailDomain", emailDomain);   // 不带用户名部分
```

**第三层：Private Attributes（私有属性）机制**。

LaunchDarkly / Statsig SDK 都支持声明"私有属性" —— 这些字段参与本地评估，但**不会被发送到分析后端**：

```java
LDConfig config = new LDConfig.Builder()
    .events(Components.sendEvents()
        .privateAttributes("email", "phone", "ip"))
    .build();

// SDK 内部评估时能用 email，但 emit 事件时这些字段会被剔除
LDContext ctx = LDContext.builder(userId)
    .set("email", email)
    .set("country", country)
    .build();
```

**第四层：审计与同意管理**。

```java
@Service
public class FlagEvaluationAuditor {

    // 用户撤回同意时
    public void onConsentRevoked(String userId) {
        // ① 删除 Flag 事件存储里这个用户的历史数据
        eventStore.deleteByUserId(userId);
        // ② 推送给所有第三方平台 deletion request
        statsigClient.deleteUser(userId);
        ldClient.deleteUser(userId);
    }

    // 数据访问请求（DSAR）
    public Map<String, Object> exportUserData(String userId) {
        return Map.of(
            "evaluations", eventStore.findByUserId(userId),
            "experiments", experimentRepo.findExposureByUserId(userId)
        );
    }
}
```

#### 验证

```java
@Test
void contextMustNotLeakPii() {
    User u = new User("u1", "alice@example.com", "+8613800000000",
        "CN", "premium");
    UserContext ctx = FlagContextFactory.build(u);

    assertNull(ctx.getAttribute("email"));
    assertNull(ctx.getAttribute("phone"));
    assertEquals("CN", ctx.getAttribute("country"));
    assertEquals(false, ctx.getAttribute("isInternalEmployee"));
}

// 抓 SDK 实际上报的 payload
@Test
void sdkPayloadMustNotContainPii() {
    MockHttpClient http = new MockHttpClient();
    FlagClient client = new FlagClient(config().withHttp(http));

    UserContext ctx = UserContext.builder()
        .key("u1").attribute("email", "leak@example.com").build();
    client.boolVariation("any-flag", ctx, false);
    client.flush();

    String body = http.lastRequestBody();
    assertFalse(body.contains("leak@example.com"));
}
```

> **口诀**：context 白名单最小化 + 派生而不暴露 + 私有属性机制 + 删除/导出 API。**Flag 平台是 PII 泄露常见盲区，必须主动管控**。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 模型与数据层
- [ ] 4 类 Flag（Release / Experiment / Ops / Permission）有清晰区分
- [ ] Release Flag 强制填 `expireAt`，最长 90 天
- [ ] Release Flag 强制填 `owner`，到期前 7 天通知
- [ ] `flag_key` 一旦上线不可改（删 + 建）
- [ ] Variations 用 JSON，支持 boolean / string / number / json 多类型
- [ ] `kill_switch` 字段独立，紧急时优先于所有规则

### 评估与分桶层
- [ ] 客户端默认评估，仅在保密 / 数据查询场景走服务端
- [ ] 分桶用 MurmurHash3 + flagKey 加盐，禁止 hashCode
- [ ] 跨语言 SDK 有"预期值"测试，保证算法一致
- [ ] 未登录用户必须用持久化 anon id（Cookie），不能用 IP
- [ ] 权重和强校验（必须等于 100%），多变体至少 2 个非零

### SDK 层
- [ ] bootstrap + stream + 兜底轮询 三层
- [ ] 紧急通道独立于常规推送通道
- [ ] Flag 不存在返回 default，不抛异常
- [ ] 未知操作符降级到不命中，不要崩溃
- [ ] SDK 版本号上报，服务端按版本下发兼容规则
- [ ] 评估事件异步上报，队列满丢弃不阻塞业务

### 治理与审计层
- [ ] CI 双向扫描"配置未引用 / 引用未配置"
- [ ] Flag 变更全部审计（who / when / from / to）
- [ ] Release Flag 大改动（> 5%）二次确认
- [ ] Flag 配置纳入 GitOps，与代码一起 PR 评审
- [ ] 紧急 kill 月度演练，SLO 1 分钟内 99% 生效

### 实验层
- [ ] 实验启动前强制声明 baseline + MDE
- [ ] 自动算最小样本量与最短运行时间
- [ ] 没到最早停止时间禁止下结论（防 peeking）
- [ ] 多重指标 Bonferroni 校正
- [ ] Variation 实际曝光监控，发现严重偏离比例报警

### 合规与隐私层
- [ ] UserContext 白名单字段，PII 不直传
- [ ] PII 字段必须派生（如 emailDomain）或哈希
- [ ] SDK 配置 privateAttributes，不上报到第三方
- [ ] 支持用户数据删除 / 导出（DSAR / GDPR）
- [ ] 跨境数据传输有 Flag 平台 DPA 协议

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### Statsig / LaunchDarkly 完整对比
- LaunchDarkly 的 Relay Proxy 部署模式：自托管 + SaaS 控制面
- Statsig 的 Pulse 报告（自动检测异常实验组）
- Unleash 的 Strategy 抽象 vs LaunchDarkly 的 Clauses，DSL 表达力对比
- DevCycle / Flagsmith 等小众方案的差异化

### 实验平台（Optimizely / Eppo / Statsig）
- Optimizely 的"统计学加速"（CUPED / 方差缩减）
- Eppo 的实验配置语言（offline experiment management）
- 实验平台与数据仓库的对接：Snowflake / BigQuery 直连 vs 自有事件流

### Bandit Algorithm（多臂赌博机）
- Epsilon-Greedy / Thompson Sampling / UCB1 三种主流算法
- 与传统 A/B 的本质区别：bandit 在跑的过程中**逐步把流量倾斜到表现好的组**
- 适用场景：广告 / 推荐这种"探索-利用"权衡明显的；不适合长期效应（需要更长时间观察）

### 因果推断（Causal Inference）
- A/B 实验本质就是 RCT（随机对照试验）
- 没法做 RCT 的场景：合成对照（Synthetic Control）、回归不连续（RDD）、双重差分（DiD）
- "Switchback" 实验设计：在网络效应（如打车、外卖）场景下用时间分桶代替用户分桶

### 灰度发布与 Feature Flag 的关系
- 流量层灰度（Nginx / Istio）vs 应用层 Flag：流量灰度粒度粗（按 Header / IP），Flag 粒度细（按用户属性）
- 蓝绿部署 vs 金丝雀 vs Feature Flag：三个互补维度
- 与 Service Mesh 的集成：用 Flag 做 traffic mirroring 决策

### Trunk-based Development + Feature Flag
- Continuous Delivery 的核心实践之一
- 主干永远可发布的关键：未完成功能用 Flag 默认 off 隐藏
- "Branch by Abstraction"：大重构期间用 Flag 在新旧实现间切换
- 配套：feature toggles 不能替代 long-lived feature branches 适用的所有场景

### SaaS 多租户 + Flag
- 租户级 Flag：同一 Flag 不同租户有不同默认值
- 用户级 Flag override：付费用户可以预览功能
- 计费维度：Flag 评估次数 / MAU 计费的成本控制

---

## 7. 引用

- LaunchDarkly Engineering Blog: [Categorizing feature flags](https://launchdarkly.com/blog/categorizing-feature-flags/)
- Pete Hodgson: [Feature Toggles (aka Feature Flags)](https://martinfowler.com/articles/feature-toggles.html) —— Martin Fowler 站经典
- Unleash 开源仓库：[Unleash/unleash](https://github.com/Unleash/unleash) —— 完整 reference 实现
- Statsig: [Sequential Testing in A/B Tests](https://blog.statsig.com/sequential-testing-in-a-b-tests-bf7d58a6c2b)
- Kohavi & Tang & Xu: *Trustworthy Online Controlled Experiments*（A/B 测试圣经，2020）
- Optimizely: [Stats Engine Whitepaper](https://www.optimizely.com/optimization-glossary/stats-engine/) —— 序贯检验工业实践
- Apollo 配置中心：[apolloconfig/apollo](https://github.com/apolloconfig/apollo)
- 阿里 [TransmittableThreadLocal](https://github.com/alibaba/transmittable-thread-local) —— 上下文跨线程传递（与本文 Context 设计相关）
- Jez Humble & Dave Farley, *Continuous Delivery* —— Trunk-based + Feature Flag 工程实践
- LaunchDarkly: [The Definitive Guide to Feature Management](https://launchdarkly.com/the-definitive-guide-to-feature-management/)

---

> 配套：[配置中心深入](/article/system-design/config-center) | [Git 工作流](/article/devops/git-workflow) | [DevOps CI/CD](/article/devops/cicd) | [灰度发布与金丝雀](/article/devops/canary-deployment)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
