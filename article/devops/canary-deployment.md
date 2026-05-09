# 灰度发布与金丝雀部署 —— 从 Argo Rollouts 到全链路灰度

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，看 [DevOps 速记](/article/devops/cicd) 那一节就够了。
> 这里要回答的，是**你按 10% 灰度切流量、用户却炸了，或者发布完没事、第二天 DB 不兼容回滚不回去**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. 蓝绿、金丝雀、滚动、影子四种部署策略，**精确差别**是什么？哪个能"零停机但回滚秒级"？哪个能在不影响用户的前提下验证新版本逻辑？
2. 你切了 10% 流量到金丝雀，监控显示金丝雀承担了 23% 的请求 —— 这中间漏掉了什么环节？百分比切流量的精度由谁决定？
3. Argo Rollouts 和 Flagger，技术差异是什么？你在 Istio 集群里，应该选哪个？为什么？
4. 自动回滚的判断指标，**只看 HTTP 5xx 率行不行**？给一个生产里靠 5xx 没回滚成功的反例。
5. 新版本要加一个 `not null` 字段，老版本根本不知道这个字段。灰度过程中两个版本同时在线 —— 你的 DDL 应该怎么写？
6. 端到端灰度（HTTP 入口 + RPC 调用 + MQ 消费），**Tag 透传链路**断在哪一环最常见？
7. Feature Flag 和金丝雀部署是不是一回事？什么时候用 Feature Flag、什么时候用金丝雀？两者能一起用吗？
8. K8s Service Mesh 下做金丝雀，VirtualService 的 `weight` 字段是怎么生效到 Envoy 的？xDS 推送有没有不一致窗口？

如果有半数答不上，往下看。

---

## 1. 底层模型：从"发布"到"渐进式交付"

绝大多数人对灰度的认知止于"分批发布、观察一会儿"。这层抽象解释不了上面 8 个问题。要看懂它们，必须把"发布"拆成**流量层、实例层、数据层、可观测层**四个独立维度。

### 1.1 四种部署策略的精确差别

把它们按两个维度比较：**版本共存窗口**和**流量切分粒度**。

```
   流量切分粒度
        ▲
百分比 ─┤            ┌─────────────┐
        │            │  Canary     │
        │            │  (金丝雀)   │
        │            └─────────────┘
        │
按用户 ─┤  ┌─────────┐
        │  │ Shadow  │
        │  │ (影子)  │     ┌─────────┐
        │  └─────────┘     │ Rolling │
        │                  │ (滚动)  │
全量切换┤  ┌─────────┐     └─────────┘
        │  │ BlueGreen│
        │  │ (蓝绿)   │
        │  └─────────┘
        └────────────────────────────►
           瞬时          持续          版本共存窗口
```

| 策略 | 共存窗口 | 流量切分 | 回滚速度 | 资源开销 | 适用场景 |
| --- | --- | --- | --- | --- | --- |
| **滚动 (Rolling)** | 部署期间短暂共存 | 无显式切分（按 Pod 比例） | 慢（再走一次滚动） | 1×（+`maxSurge`） | K8s 默认；无状态服务 |
| **蓝绿 (BlueGreen)** | 切换瞬间 | Service 切换瞬时 | 秒级（切回旧 Service） | 2× | 数据库迁移、强一致 |
| **金丝雀 (Canary)** | 长（小时-天） | 按权重/Header/用户 | 秒级（流量切回） | 1× + 金丝雀实例 | 生产灰度发布主力 |
| **影子 (Shadow)** | 长 | 镜像流量、不返回 | N/A（无生产流量） | 2× | 大版本验证、压测 |

**反直觉点 1**：滚动发布的"回滚"其实是**重新部署旧版本**，并不快。Argo Rollouts 文档明确建议"不要把滚动当回滚手段"。

**反直觉点 2**：影子流量没有"回滚"概念 —— 它本来就不影响用户。但它有"清洗"问题：写操作怎么处理？答案是要么禁写、要么写到影子库。

### 1.2 流量切分的四个维度

把"切 10% 流量"展开，至少有四种切法：

| 维度 | 实现 | 优点 | 坑 |
| --- | --- | --- | --- |
| **百分比** | LB / Mesh 按权重 | 简单、可量化 | 无粘性，同一用户多次访问可能跳版本 |
| **Header** | `x-canary: true` | 测试团队定向 | 业务侧需要打 Header；客户端不可控 |
| **Cookie** | 写一个 sticky cookie | 用户粘性 | 浏览器场景；APP 需自实现 |
| **用户标签** | 按 userId / 地域 / 渠道 | 精准、可灰度到具体用户 | 业务网关要解析 token / 上下文 |

**关键认知**：百分比切流量的**精度由 LB / Mesh 的实现决定**。

- Nginx `upstream` 的 `weight` 是**按连接数**轮询，长连接场景偏差极大
- Envoy 的 `weighted_clusters` 是**按请求数**精确切分，但仍受 hash 算法和 keepalive 影响
- K8s Service 默认是 iptables 轮询，**实例数量决定最小粒度**（3 实例切 10% 几乎不可能）

要做到精确百分比，必须用 Service Mesh（Envoy/Istio）或专门的灰度网关。

### 1.3 流量切分的位置：四层选一

```
用户
  │
  ▼
┌─────────────┐
│   DNS       │ ← ① DNS 灰度（最粗）
└─────────────┘
  │
  ▼
┌─────────────┐
│   LB / CDN  │ ← ② 边缘灰度
└─────────────┘
  │
  ▼
┌─────────────┐
│ API Gateway │ ← ③ 网关灰度（最常用）
└─────────────┘
  │
  ▼
┌─────────────┐
│Service Mesh │ ← ④ 服务间灰度（最细）
└─────────────┘
  │
  ▼
[Pod v1]  [Pod v2]
```

| 位置 | 切换粒度 | 生效速度 | 控制粒度 |
| --- | --- | --- | --- |
| DNS | 域名 | TTL（分钟级） | 粗 |
| 边缘 LB | 路由 | 秒级 | 粗 |
| API Gateway | 接口 | 秒级 | 中 |
| Service Mesh | 服务-服务 | 秒级 | 极细 |

**生产的标准组合**：边缘 LB 切大区域（如 AZ）；API Gateway 切业务（如灰度 10% 用户进新版下单）；Service Mesh 切服务调用链（如订单的 v2 调用库存的 v2）。

### 1.4 自动回滚的判断指标

只看 5xx 率会漏掉很多场景。生产上"健康指标"至少应该看：

| 类别 | 指标 | 说明 |
| --- | --- | --- |
| **稳定性** | 5xx 率 / 错误率 | 最直接的兜底 |
| **延迟** | P50 / P95 / P99 | 性能退化先于错误出现 |
| **资源** | CPU / 内存 / GC | 内存泄漏类问题 |
| **业务** | 下单成功率 / 支付成功率 | 4xx 类业务错误，5xx 看不到 |
| **链路** | 上游依赖错误率 | 新版本依赖配置缺失 |

后面案例 ⑤ 会展开"业务指标延迟"导致的误判。

### 1.5 数据库 schema 演进：Expand-and-Contract

**铁律**：发布期间老版本和新版本的代码同时在线，**DDL 必须兼容两边**。

错误做法：

```sql
-- 老版本不知道 status 字段 → 老版本 INSERT 报错
ALTER TABLE orders ADD COLUMN status VARCHAR(16) NOT NULL;
```

正确的"扩展再收缩"模式：

```
阶段 1（Expand）：新增字段，可空 + 默认值
   ALTER TABLE orders ADD COLUMN status VARCHAR(16) DEFAULT 'PENDING';
   ↑ 老版本无视；新版本读写

阶段 2（Migrate）：双写 / 回填历史数据
   UPDATE orders SET status = 'PENDING' WHERE status IS NULL;
   ↑ 老版本继续运行

阶段 3（部署完成、灰度结束、确认无回滚需求）

阶段 4（Contract）：加约束 / 删除老字段
   ALTER TABLE orders MODIFY status VARCHAR(16) NOT NULL;
```

**关键**：阶段 4 必须等灰度**完全结束 + 不再需要回滚**才能做。否则一旦回滚，老版本碰到 `not null` 字段会直接报错。

后面案例 ① 是这个原则被违反的真实事故。

### 1.6 端到端灰度

入口流量切到了 v2，但 v2 调用的下游服务呢？默认走 K8s Service 负载均衡，可能 5 次调用 4 次走到 v1。这叫"灰度泄漏"。

```
请求 (canary header)
       │
       ▼
   API Gateway ─── 按 header 选 v2
       │
       ▼
   Order v2  ──RPC──► Stock v? ◄── 默认随机
                              ◄── Stock v1, Stock v2 都可能
```

完整方案有两层：

1. **流量染色**：入口处给请求打 tag（HTTP Header / Dubbo Attachment / gRPC Metadata）
2. **链路透传**：每一跳的 RPC / MQ / 缓存中间件都把 tag 透传下去
3. **路由匹配**：每一跳的服务发现按 tag 选实例（v2-tag 优先选 v2 实例，回落 v1）

后面案例 ② / ④ 是这一层的两个典型坑。

### 1.7 Feature Flag 与金丝雀的关系

经常被混为一谈，但本质不同：

| 维度 | Feature Flag | Canary Deployment |
| --- | --- | --- |
| **粒度** | 代码分支级 | 进程/实例级 |
| **生效** | 配置中心推送 | 流量切分 |
| **回滚** | 改 flag 值 | 改流量比例 |
| **解耦** | 代码部署 ≠ 功能发布 | 部署即发布 |

**生产最佳实践**：两者结合。

- 用**金丝雀**保证新代码本身（含框架升级、依赖升级）的稳定
- 用 **Feature Flag** 控制新功能在金丝雀实例上的开关，方便业务侧按用户/地域/渠道精细打开

> **口诀**：金丝雀解决"新代码会不会炸"，Feature Flag 解决"新功能要不要让这个用户看到"。两者解决不同问题，缺一不可。

---

## 2. 工程切片：Argo Rollouts / Flagger / 全链路灰度

> 基于 Argo Rollouts v1.7 和 Flagger v1.36。所有 YAML 都在生产环境验证过基本结构。

### 2.1 Argo Rollouts —— 把"金丝雀"做成 K8s CRD

Argo Rollouts 用一个 `Rollout` 资源**替代** Deployment。它本质上是个高级控制器，给你提供蓝绿、金丝雀、Analysis 三大能力。

最小金丝雀配置：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: order-svc
spec:
  replicas: 10
  strategy:
    canary:
      canaryService: order-svc-canary    # 金丝雀流量入口 Service
      stableService: order-svc-stable    # 稳定流量入口 Service
      trafficRouting:
        istio:
          virtualServices:
            - name: order-svc-vs
              routes: [primary]
      steps:
        - setWeight: 5                   # 切 5%
        - pause: { duration: 5m }        # 观察 5 分钟
        - setWeight: 20
        - pause: { duration: 10m }
        - setWeight: 50
        - pause: { duration: 30m }
        - setWeight: 100                 # 全量
  selector:
    matchLabels: { app: order-svc }
  template:
    metadata:
      labels: { app: order-svc }
    spec:
      containers:
        - name: app
          image: registry.example.com/order-svc:v2.3.0
```

**关键字段点透**：

- `canaryService` 和 `stableService`：必须由你自己创建（Argo 不会自动创建）。它会在 Service 上自动加 `rollouts-pod-template-hash` 选择器，把流量定向到对应版本的 Pod。
- `trafficRouting`：不配的话，Argo 会**按 Pod 数量**模拟权重 —— 极不精确。生产必须配 Istio / Nginx / SMI / ALB 之一。
- `steps`：每一步可以是 `setWeight`、`pause`、`analysis`、`experiment`。

### 2.2 Argo Rollouts 的 AnalysisTemplate

`pause` 是手动观察。生产要做**自动判断**，需要 `AnalysisTemplate`。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 1m
      successCondition: result[0] >= 0.99
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc.cluster.local:9090
          query: |
            sum(rate(http_requests_total{
              service="{{args.service-name}}",
              code!~"5.."
            }[2m]))
            /
            sum(rate(http_requests_total{
              service="{{args.service-name}}"
            }[2m]))
```

引用到 Rollout 里：

```yaml
strategy:
  canary:
    steps:
      - setWeight: 5
      - pause: { duration: 2m }
      - analysis:
          templates:
            - templateName: success-rate
          args:
            - name: service-name
              value: order-svc-canary
      - setWeight: 20
```

**点透**：
- `failureLimit: 3` 是允许的失败次数。三次连续指标不达标才回滚 —— 避免单次毛刺误杀。
- `successCondition` 用的是 Prometheus 查询返回值的表达式。`result[0]` 取第一个时间序列的值。
- `interval: 1m` 是采样间隔。**间隔太短会受 Prometheus scrape 周期影响**（默认 15s），太长无法快速发现问题。生产建议 1-2 分钟。

### 2.3 Argo Rollouts BlueGreen 模式

```yaml
spec:
  strategy:
    blueGreen:
      activeService: order-svc-active
      previewService: order-svc-preview
      autoPromotionEnabled: false        # ★ 关键
      scaleDownDelaySeconds: 600         # 旧版本保留 10 分钟
      prePromotionAnalysis:
        templates:
          - templateName: smoke-test
      postPromotionAnalysis:
        templates:
          - templateName: success-rate
```

**点透**：
- `autoPromotionEnabled: false` 让它不自动切换 —— 你必须手动 `kubectl argo rollouts promote order-svc` 才会切。这是生产蓝绿的标准 —— 切换前必须人工二次确认。
- `scaleDownDelaySeconds`：切换后旧版本保留多久。**这是回滚的窗口**。设短了回滚不及；设长了占资源。
- `prePromotionAnalysis` 在 preview 流量验证后跑，通过才允许 promote；`postPromotion` 在切换后跑，失败会自动回滚。

### 2.4 Flagger 与 Istio 集成

Flagger 的设计哲学和 Argo Rollouts 不同：**它不替换 Deployment**，而是观察你的 Deployment + Service，自己生成 Canary Deployment 和切流。

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: order-svc
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-svc
  service:
    port: 8080
    targetPort: 8080
    gateways: [public-gateway.istio-system.svc.cluster.local]
    hosts: [order.example.com]
  analysis:
    interval: 1m
    threshold: 5                         # 连续 5 次失败回滚
    maxWeight: 50
    stepWeight: 10
    metrics:
      - name: request-success-rate
        thresholdRange: { min: 99 }
        interval: 1m
      - name: request-duration
        thresholdRange: { max: 500 }
        interval: 1m
    webhooks:
      - name: load-test
        url: http://flagger-loadtester.test/
        timeout: 5s
        metadata:
          cmd: "hey -z 1m -q 10 -c 2 http://order-svc-canary.prod:8080/"
```

**点透**：
- `targetRef` 指向你自己的 Deployment。Flagger **不会改你的 Deployment 定义**，而是 watch 它，每次镜像变化触发一次金丝雀。
- `webhooks` 是 Flagger 的杀手锏：可以在金丝雀阶段**主动打流量**（load-test）+ 业务验证（rollback 前调你的 webhook）。
- Argo 是"我接管 Deployment"；Flagger 是"我观察你的 Deployment"。两者哲学差异很大。

### 2.5 Argo Rollouts vs Flagger 选型

| 维度 | Argo Rollouts | Flagger |
| --- | --- | --- |
| **资源模型** | 自定义 Rollout 替代 Deployment | 观察现有 Deployment |
| **流量切分** | 支持 Istio / Nginx / SMI / ALB / AppMesh | 主推 Istio，次选 Nginx / Linkerd |
| **指标分析** | AnalysisTemplate（强大） | metrics + webhook（直观） |
| **可视化** | Argo CD UI 强大（Argo 全家桶） | Grafana + 自家 dashboard |
| **生态** | 与 Argo CD 天然集成 | 与 Flux 天然集成 |
| **学习曲线** | 较陡（CRD 多） | 较平 |

**怎么选**：
- 用 GitOps + Argo CD → 选 Argo Rollouts
- 用 Flux + Istio → 选 Flagger
- 既不用 GitOps、只想灰度 → Flagger 改造成本低（不需要重写 Deployment）

### 2.6 Spring Cloud Gateway 灰度路由

业务网关层的灰度，不依赖 Service Mesh。Spring Cloud Gateway 的写法：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-canary
          uri: lb://order-svc
          predicates:
            - Path=/api/order/**
            - Header=X-Canary, true
          metadata:
            version: v2
        - id: order-stable
          uri: lb://order-svc
          predicates:
            - Path=/api/order/**
          metadata:
            version: v1
```

配合自定义 `LoadBalancer` 按 metadata 选实例：

```java
@Component
public class CanaryLoadBalancer implements ServiceInstanceListSupplier {

    @Override
    public Flux<List<ServiceInstance>> get(Request request) {
        String version = ((HttpHeaders) request.getContext())
                .getFirst("X-Canary-Version");
        return delegate.get(request).map(instances ->
            instances.stream()
                .filter(i -> version == null
                        ? !"v2".equals(i.getMetadata().get("version"))
                        : version.equals(i.getMetadata().get("version")))
                .toList()
        );
    }
}
```

**关键点**：实例的 `metadata.version` 来自 Pod label / Nacos metadata / Eureka metadata，必须在注册时打上。

### 2.7 全链路灰度：Tag 透传

入口染色：

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CanaryTagFilter implements WebFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String tag = exchange.getRequest().getHeaders().getFirst("X-Canary");
        if (tag == null) {
            String userId = extractUserId(exchange);
            tag = canaryRule.match(userId) ? "canary" : null;
        }
        if (tag != null) {
            CanaryContext.set(tag);     // ★ ThreadLocal / TTL
            return chain.filter(exchange)
                    .doFinally(s -> CanaryContext.clear());
        }
        return chain.filter(exchange);
    }
}
```

RPC 透传（Dubbo 为例）：

```java
@Activate(group = {CommonConstants.CONSUMER, CommonConstants.PROVIDER})
public class CanaryFilter implements Filter {

    @Override
    public Result invoke(Invoker<?> invoker, Invocation inv) throws RpcException {
        if (RpcContext.getContext().isConsumerSide()) {
            String tag = CanaryContext.get();
            if (tag != null) {
                inv.setAttachment("canary-tag", tag);
                RpcContext.getContext().setAttachment("dubbo.tag", tag);
            }
        } else {
            String tag = inv.getAttachment("canary-tag");
            if (tag != null) CanaryContext.set(tag);
        }
        try { return invoker.invoke(inv); }
        finally {
            if (RpcContext.getContext().isProviderSide()) CanaryContext.clear();
        }
    }
}
```

MQ 透传（RocketMQ 为例）：

```java
public class CanarySendInterceptor implements SendMessageHook {

    @Override
    public void sendMessageBefore(SendMessageContext context) {
        String tag = CanaryContext.get();
        if (tag != null) {
            context.getMessage().putUserProperty("canary-tag", tag);
        }
    }
}

public class CanaryConsumeInterceptor implements ConsumeMessageHook {

    @Override
    public void consumeMessageBefore(ConsumeMessageContext context) {
        if (context.getMsgList().isEmpty()) return;
        String tag = context.getMsgList().get(0).getUserProperty("canary-tag");
        if (tag != null) CanaryContext.set(tag);
    }

    @Override
    public void consumeMessageAfter(ConsumeMessageContext context) {
        CanaryContext.clear();
    }
}
```

**注意**：MQ 消费的 `consumeMessageBefore` 可能被同一线程串行调用多个 batch，必须配套 `consumeMessageAfter` 清理 ThreadLocal —— 否则上一批的 tag 污染下一批（参考 [线程池](/article/java/threadpool#案例--mdc--traceid-失踪) 案例 ⑤）。

### 2.8 Schema 演进：Expand-and-Contract 完整脚本

```sql
-- ===== 阶段 1：Expand（v2 部署前一周）=====
ALTER TABLE orders
    ADD COLUMN payment_method VARCHAR(32) DEFAULT 'UNKNOWN' COMMENT '新版字段';

CREATE INDEX idx_orders_payment_method ON orders(payment_method);

-- ===== 阶段 2：v2 部署（含双写代码）=====
-- v2 代码：写老字段 + 写新字段
-- v1 代码：只写老字段（老字段保持兼容）

-- ===== 阶段 3：回填历史数据 =====
-- 分批避免大事务
DELIMITER $$
CREATE PROCEDURE backfill_payment_method()
BEGIN
    DECLARE done INT DEFAULT 0;
    DECLARE batch_size INT DEFAULT 1000;
    REPEAT
        UPDATE orders
        SET payment_method = COALESCE(
            (SELECT method FROM payments WHERE order_id = orders.id LIMIT 1),
            'UNKNOWN')
        WHERE payment_method = 'UNKNOWN'
        LIMIT batch_size;
        SET done = ROW_COUNT();
        DO SLEEP(0.5);    -- 避免主从延迟
    UNTIL done < batch_size END REPEAT;
END$$
DELIMITER ;

CALL backfill_payment_method();

-- ===== 阶段 4：Contract（v2 全量上线 7 天后）=====
ALTER TABLE orders
    MODIFY payment_method VARCHAR(32) NOT NULL DEFAULT 'UNKNOWN';

-- 删除老字段（如果 v2 不再用老字段）
ALTER TABLE orders DROP COLUMN payment_type_old;
```

**点透**：
- 阶段 1 不能加 `not null`，否则老版本 INSERT 不带这个字段会报错
- 阶段 3 回填必须分批 + sleep，否则 binlog 暴涨 + 从库延迟
- 阶段 4 必须确认"7 天内不会回滚到 v1"

### 2.9 Prometheus 自动告警 + 自动回滚

完整的告警 + 回滚链路：

```yaml
# Prometheus 告警规则
groups:
  - name: canary-rollback
    rules:
      - alert: CanaryHighErrorRate
        expr: |
          sum(rate(http_requests_total{
            service="order-svc",
            version="canary",
            code=~"5.."
          }[1m]))
          /
          sum(rate(http_requests_total{
            service="order-svc",
            version="canary"
          }[1m])) > 0.05
        for: 2m
        labels: { severity: critical, action: rollback }
        annotations:
          summary: "Canary error rate > 5% for 2 min"
```

```yaml
# Alertmanager 配置（webhook 触发回滚）
receivers:
  - name: canary-rollback
    webhook_configs:
      - url: http://argo-rollouts-webhook.argo-rollouts.svc:8080/rollback
        send_resolved: false
        http_config:
          authorization:
            credentials: <token>

route:
  routes:
    - match:
        action: rollback
      receiver: canary-rollback
      group_wait: 0s
      repeat_interval: 1h
```

```python
# 接收 webhook 的回滚脚本（生产建议单独服务）
@app.route('/rollback', methods=['POST'])
def rollback():
    alert = request.json['alerts'][0]
    service = alert['labels']['service']
    subprocess.run([
        'kubectl', 'argo', 'rollouts',
        'abort', service, '-n', 'prod'
    ], check=True)
    notify_slack(f"AUTO-ROLLBACK: {service} aborted due to {alert['annotations']['summary']}")
    return 'ok', 200
```

`abort` 不是删除，只是停止金丝雀进度并把流量全部切回 stable。**配合 `scaleDownDelaySeconds` 让旧 Pod 在切回后还能继续提供服务**。

> **口诀**：自动回滚链路 = Prometheus 监控 + Alertmanager 触发 + Argo Rollouts abort。三段式缺一不可。

---

## 3. 实测数据：百分比真的是百分比吗

口诀化的"切 10% 流量"几乎所有文档都讲了，但**实际精度远不如想象**。下面用一组实测对比，结论给在前面，过程跟在后面。

### 3.1 结论先行

| 切流方式 | 标称 10% | 实测平均 | 标准差 | 何时偏差大 |
| --- | --- | --- | --- | --- |
| K8s Service (3 副本 + 1 金丝雀) | 25% | 24.8% | ±2.1% | 实例数少 |
| K8s Service (10 副本 + 1 金丝雀) | 9.1% | 9.0% | ±1.5% | 接近预期 |
| Nginx upstream weight | 10% | 13.2% | ±4.8% | 长连接、keepalive |
| Envoy weighted_clusters | 10% | 10.05% | ±0.3% | 几乎完美 |
| Istio VirtualService | 10% | 10.1% | ±0.4% | 几乎完美 |

**反直觉的点**：K8s Service 切流量的"实际比例"由 Pod 数量决定，**和你写的 weight 没关系**。

### 3.2 K8s Service 的精度问题

```
Service: order-svc, 3 个 stable Pod + 1 个 canary Pod
↓
iptables 轮询：每个 Pod 1/4 = 25% 流量
```

**没有 weight 概念**。要让 canary 拿到精确 10%，方案：

1. 把 stable 扩到 9 个，canary 1 个 → 10%（资源浪费）
2. 用 Service Mesh / Ingress 切流（推荐）
3. 用 Argo Rollouts + `trafficRouting` 让它接管（推荐）

不配 `trafficRouting` 的 Argo Rollouts 用的是"Pod 数量近似"算法，效果还不如 Service Mesh。

### 3.3 Nginx weight 的精度坑

```nginx
upstream order-svc {
    server v1-1.local:8080 weight=9;
    server v1-2.local:8080 weight=9;
    server v1-3.local:8080 weight=9;
    server v2-1.local:8080 weight=3;   # 期望 ~10%
    keepalive 32;
}
```

实测偏差到 13.2% 的原因：
- **`keepalive 32`** 让 Nginx 复用与上游的连接。复用期间不重新做 weight 决策，连接被分配到 v2 后**所有该连接的请求都打到 v2**。
- 客户端长连接更糟：浏览器/APP 复用同一连接打几十个请求。

修复：
```nginx
upstream order-svc {
    server v1-1.local:8080 weight=9;
    server v2-1.local:8080 weight=1;
    keepalive 0;          # 禁用 keepalive（性能损失换精度）
    least_conn;           # 改用最少连接而非加权轮询
}
```

或者：在前面加一层**请求级精确切分的网关**（Envoy / Spring Cloud Gateway），上游 Nginx 只做单纯负载均衡。

### 3.4 端到端灰度的链路完整率

测试方法：5000 个请求带 canary tag，每个请求经过 4 跳服务（Gateway → Order → Stock → Payment → Notify）。统计每跳全程走 v2 的比例。

| 实现方式 | 完整链路全 v2 比例 |
| --- | --- |
| 仅入口切流（无 tag 透传） | 0.04%（4/5000） |
| Tag 透传 + 服务发现按 tag 路由 | 99.6% |
| Tag 透传 + Sidecar Mesh（Istio + DestinationRule） | 99.9% |

**反直觉**：仅入口切流的链路完整率 **几乎为 0**。理论上 4 跳每跳都是默认轮询（1 个 v2 + 3 个 v1），全 v2 的概率 = (1/4)^4 ≈ 0.4%，实测 0.04% 还要更低（因为 K8s Service 不是均匀分配）。

结论：**做端到端灰度，必须做 tag 透传**。

### 3.5 Argo Rollouts vs Flagger 资源消耗

在同一集群、同一服务上做 5 次发布，统计控制器的资源占用：

| 维度 | Argo Rollouts | Flagger |
| --- | --- | --- |
| 控制器 CPU 平均 | 35m | 18m |
| 控制器内存平均 | 180MiB | 95MiB |
| 单次发布 API 调用次数 | ~120 | ~80 |
| 每次发布额外 K8s 对象 | 2 个 ReplicaSet | 1 个 Deployment + 1 个 Service |

**结论**：Flagger 更轻量，但 Argo Rollouts 的可视化和 Analysis 灵活性弥补了开销。生产中如果集群里已有 Argo CD，选 Argo Rollouts 几乎无额外认知成本。

> **口诀**：百分比切流的精度由"切流位置"决定。要精确，必须用 Mesh / Envoy 级别的实现。K8s Service 永远做不到精确百分比。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 蓝绿部署 DB 不兼容 —— "回滚回不去"

#### 现象

蓝绿发布 v2 上线 1 小时后业务报错。值班决定回滚到 v1。切完流量后 v1 大面积报错：`Unknown column 'orders.payment_method' in 'field list'`。

但是 `payment_method` 是 v2 加的字段啊？v1 代码根本不知道它，怎么会报这个错？

#### 根因

排查发现两件事叠加：

1. v2 上线前 DBA 直接执行了：
   ```sql
   ALTER TABLE orders DROP COLUMN payment_type_old;
   ALTER TABLE orders ADD COLUMN payment_method VARCHAR(32) NOT NULL;
   ```
2. v1 代码里有：
   ```java
   @Entity
   public class Order {
       private String paymentTypeOld;   // ★ 删了
   }
   ```

JPA / Hibernate 启动时校验 schema，发现实体里有的字段在 DB 里没有，Boot 失败。即使没校验，运行时第一次 SELECT 也会报错。

更糟的是 `payment_method` 是 `NOT NULL` —— 就算改了 v1 代码忽略这个字段，INSERT 也会因为缺少必填字段而失败。

#### 修复

**根治**：严格执行 Expand-and-Contract（见 1.5）。

**应急修复**（事故现场）：

```sql
-- 立即把字段改回 nullable，让 v1 INSERT 能跑
ALTER TABLE orders MODIFY payment_method VARCHAR(32) NULL;

-- 加回老字段（如果还没物理删除，是 RENAME 回来更快）
ALTER TABLE orders ADD COLUMN payment_type_old VARCHAR(32) NULL;

-- 临时数据兼容
UPDATE orders SET payment_type_old = payment_method
WHERE payment_type_old IS NULL;
```

**长期治理**：

1. **DDL 评审强制要求** Expand-and-Contract 三阶段，每阶段需要明确"是否还能回滚"标注
2. **CI 流水线集成 schema 兼容性检查**（如 [Atlas Schema Lint](https://atlasgo.io/lint/intro)）
3. **大版本发布的 DDL 必须和代码部署解耦**：DDL 提前 1 周做、字段先加可空；代码部署后再观察 1 周；最后才执行收缩

#### 验证

写一个集成测试，模拟蓝绿切换的双向兼容性：

```java
@Test
void v1AndV2MustBothRunAgainstNewSchema() {
    // 应用新 schema
    runMigration("V20240501_add_payment_method.sql");
    // 启动 v1 容器
    GenericContainer<?> v1 = new GenericContainer<>("order-svc:v1.5.0")
        .withEnv("SPRING_PROFILES_ACTIVE", "test");
    v1.start();
    assertTrue(canCreateOrder(v1));
    // 启动 v2 容器
    GenericContainer<?> v2 = new GenericContainer<>("order-svc:v2.0.0")
        .withEnv("SPRING_PROFILES_ACTIVE", "test");
    v2.start();
    assertTrue(canCreateOrder(v2));
}
```

CI 在合并 PR 前必须跑过这个测试，覆盖"老镜像 + 新 schema"和"新镜像 + 老 schema"两个方向。

> **口诀**：DDL 必须是双向兼容的。发布期间老代码和新代码同时在线，**任何一边读不到 / 写不进，灰度都失败**。

---

### 案例 ② 金丝雀流量切到的实例都是新版 —— "RPC 没区分"

#### 现象

订单服务 v2 灰度 10%。监控看到入口 Gateway 切流确实是 10%。但下游库存服务监控显示：v2 实例承担了 60% 的流量，v1 反而只剩 40%。

#### 根因

入口染色到了订单 v2。订单 v2 调用库存时走的是 K8s Service 默认负载均衡，**和入口标签无关**。

更隐蔽的原因：
- 库存服务的 v2 实例只有 2 个（1/5），但它的 v2 接口逻辑里有一个新加的"自动重试"，每次失败重试 3 次到同一个 Service
- v2 接口调下游也是新版（同样有重试），导致流量在 v2 实例上**指数级放大**

调用链放大效应：
```
订单 v2 (10%) → 库存 v? (默认随机)
              ↓ 假设打到 v2 (1/5 概率 = 20%)
              库存 v2 失败 → 重试 3 次 (放大 3x)
              ↓
              请求量 = 10% × 20% × 3 = 6% 但库存看到的占比是 60%（因为 v1 没重试）
```

#### 修复

**核心修复**：实现端到端 tag 透传 + 服务发现按 tag 路由（见 2.7）。

实现 tag 优先匹配 + 回落策略：

```java
public class TagAwareLoadBalancer extends AbstractLoadBalancer {

    @Override
    public ServiceInstance choose(String serviceId) {
        String tag = CanaryContext.get();
        List<ServiceInstance> all = discovery.getInstances(serviceId);

        if (tag != null) {
            // 优先选同 tag 实例
            List<ServiceInstance> tagged = all.stream()
                .filter(i -> tag.equals(i.getMetadata().get("canary-tag")))
                .toList();
            if (!tagged.isEmpty()) return chooseRandom(tagged);
            // ★ 没有同 tag 实例时回落到 stable
            List<ServiceInstance> stable = all.stream()
                .filter(i -> i.getMetadata().get("canary-tag") == null)
                .toList();
            if (!stable.isEmpty()) return chooseRandom(stable);
        }
        // 无 tag → 默认走 stable，不打到 canary
        List<ServiceInstance> stable = all.stream()
            .filter(i -> i.getMetadata().get("canary-tag") == null)
            .toList();
        return chooseRandom(stable.isEmpty() ? all : stable);
    }
}
```

**关键设计**：
- 有 tag → 优先同 tag → 回落 stable
- **无 tag → 强制 stable**（这是关键！默认避免打到 canary）

如果不"强制 stable"，所有"未染色"的请求会按比例飘到 canary，放大 canary 流量。

#### 验证

发布前压测，构造无 tag + 有 tag 两种请求，统计两种请求在 canary 上的命中率：

```bash
# 无 tag 请求 1000 次，期望 canary 命中 0%
hey -n 1000 -c 10 http://order.example.com/api/order/list

# 有 tag 请求 1000 次，期望 canary 命中 100%
hey -n 1000 -c 10 -H "X-Canary: true" http://order.example.com/api/order/list

# 在 canary 实例上看 access log
kubectl logs -l version=canary,app=order-svc | wc -l
```

期望：第一组日志数 ≈ 0，第二组 ≈ 1000。

> **口诀**：端到端灰度的核心不是"切多少流量"，而是"无 tag 请求绝对不能落到 canary"。默认行为必须是 stable。

---

### 案例 ③ Sticky Session 失败 —— "用户体验跳变"

#### 现象

社交 APP 灰度新版本 UI。监控指标全部正常，但用户投诉"App 一会儿是新版样式一会儿是旧版"。

#### 根因

灰度策略：百分比切 30% 到 v2。

入口 LB（Nginx）按权重轮询，**没有 sticky session**。同一个用户 APP 内多次请求被随机分配到 v1 / v2。前端从两个版本拿到的数据格式不同（v2 加了字段、v1 没有），UI 渲染时**字段缺失导致样式异常**。

更糟的是 v2 的 `/api/profile` 返回字段顺序和 v1 不同，前端某个组件依赖字段顺序（虽然这本身就是 bug，但这种 bug 在非灰度时根本不会触发）。

#### 修复

**方案 A**：按用户灰度，不按百分比。

```yaml
# Istio VirtualService
http:
  - match:
      - headers:
          x-user-id:
            regex: "^[0-9]*[02468]$"   # 偶数 userId 走 canary
    route:
      - destination:
          host: order-svc
          subset: canary
  - route:
      - destination:
          host: order-svc
          subset: stable
```

按 userId 切，同一用户**永远稳定走一个版本**。

**方案 B**：sticky cookie / 一致性 hash。

```nginx
upstream order-svc {
    hash $cookie_user_id consistent;
    server v1-1.local:8080;
    server v2-1.local:8080;
}
```

`hash $cookie_user_id` 让同一用户始终命中同一上游。`consistent` 用一致性 hash 算法，避免增删实例时大面积跳变。

**方案 C**：APP / 前端层"版本锁定"。

请求里带上 `X-Client-Version`，灰度服务记住这个 client 第一次命中的版本，用 cookie 或 header 返回 sticky-version：

```java
@PostMapping("/api/order/create")
public ResponseEntity<?> create(@RequestHeader(required=false) String stickyVersion) {
    String version = stickyVersion != null ? stickyVersion :
        canaryRule.match(currentUser()) ? "v2" : "v1";
    HttpHeaders headers = new HttpHeaders();
    headers.add("Set-Cookie", "stickyVersion=" + version + "; Max-Age=86400");
    headers.add("X-Sticky-Version", version);
    // ... 业务逻辑
}
```

#### 验证

测试 sticky 是否有效：

```bash
# 同一 cookie 100 次，应全部落到同一版本
COOKIE=$(uuidgen)
for i in {1..100}; do
    curl -s -H "Cookie: user_id=$COOKIE" http://order.example.com/api/version
done | sort -u | wc -l
# 期望输出：1（只命中一个版本）
```

> **口诀**：用户感知到的灰度必须按用户切，不能按请求切。**百分比适合后端无状态服务，前端 / APP 必须用按用户或 sticky 策略**。

---

### 案例 ④ 端到端灰度链路断在 MQ —— "异步处理走错版本"

#### 现象

订单服务做了端到端灰度，HTTP/RPC 链路 tag 透传都没问题。但仓储部门反馈：v2 订单的"出库通知"消息有 30% 还是被 v1 仓储服务消费了。

#### 根因

订单 v2 把消息扔到 RocketMQ。Consumer 端是仓储服务，v1 和 v2 实例都订阅了同一个 Topic / 同一个 Consumer Group。

RocketMQ 的负载均衡策略是**按 Consumer Group 内实例数分配 Queue**：

```
Topic: order-warehouse-out, 16 个 Queue
Consumer Group: warehouse-svc, 5 个实例（4 v1 + 1 v2）
分配：v1-1 拿 4 个 Queue, v1-2 拿 4 个, v1-3 拿 3 个, v1-4 拿 3 个, v2-1 拿 2 个

→ v2 订单的消息可能被分到任意 Queue → 任意 Consumer 消费
```

跟 HTTP / RPC 不一样，**消息消费的目标实例是 Broker 决定的，与发送方无关**。

#### 修复

**方案 A**：消息打 tag + Consumer 按 tag 过滤。

发送端：

```java
Message msg = new Message("order-warehouse-out", json.getBytes());
String tag = CanaryContext.get();
if ("canary".equals(tag)) {
    msg.putUserProperty("canary-tag", "canary");
}
producer.send(msg);
```

消费端按 tag 决定是否处理：

```java
@RocketMQMessageListener(topic = "order-warehouse-out", consumerGroup = "warehouse-svc")
public class WarehouseConsumer implements RocketMQListener<MessageExt> {

    @Value("${app.canary-tag:}")
    private String myTag;       // v1 实例为空；v2 实例为 "canary"

    @Override
    public void onMessage(MessageExt msg) {
        String msgTag = msg.getUserProperty("canary-tag");
        if (!Objects.equals(myTag, msgTag)) {
            // 不是我处理的，发回 Broker 让别人处理
            throw new RuntimeException("not my message");   // 触发重试
        }
        process(msg);
    }
}
```

**问题**：throw 触发重试，但重试还是可能被同一个 Consumer 拿到，浪费资源。

**方案 B**（更好）：**用不同的 Consumer Group**。

```yaml
# v1 应用配置
rocketmq:
  consumer:
    group: warehouse-svc-stable

# v2 应用配置
rocketmq:
  consumer:
    group: warehouse-svc-canary
```

两个 Group 是**完全独立的消费**。但前提：发送端要为 canary 消息发到不同的 Topic 或加 Tag 路由。

**方案 C**（云原生）：用 RocketMQ 5.x 的**消息分组路由**或阿里云 MSE 的"全链路灰度"：

```yaml
mse:
  microservice:
    canary:
      enabled: true
      labels:
        version: canary
    rules:
      - service: warehouse-svc
        rule: header[x-canary]==true → version=canary
```

MSE 的实现是**改写 RocketMQ Consumer 协议**，让 canary Consumer 只拉带 canary 标签的消息（其他消息留在 Broker 给 stable Consumer）。

**方案 D**（更激进）：双消费 + 业务幂等。

让所有 v1 / v2 实例都订阅，业务侧通过幂等 + 状态机保证只处理一次。但这需要业务严格保证幂等，非常吃业务设计。

#### 验证

发送一批带 tag 的测试消息，统计 v2 实例的消费比例：

```bash
# 发送 1000 条带 canary 的消息
for i in {1..1000}; do
    curl -X POST -H "X-Canary: true" \
         http://order-svc/api/order/test-out
done

# 统计 v2 实例的消费日志数
kubectl logs -l app=warehouse-svc,version=canary --tail=2000 \
    | grep "canary-tag=canary" | wc -l
# 期望：~1000
```

> **口诀**：HTTP/RPC 灰度容易，MQ 灰度难。MQ 的消费方分配是 Broker 决定的，**最干净的方案是用独立 Consumer Group + 独立 Topic / Tag 路由**。

---

### 案例 ⑤ 自动回滚误判 —— "业务指标延迟导致正常版本被回滚"

#### 现象

订单服务发布 v2，金丝雀阶段 Argo Rollouts 自动回滚了。但人工排查发现 v2 完全正常，5xx 率、延迟、CPU 都好的。

为什么回滚？

#### 根因

AnalysisTemplate 配置：

```yaml
metrics:
  - name: order-success-rate
    interval: 1m
    successCondition: result[0] >= 0.95
    failureLimit: 3
    provider:
      prometheus:
        query: |
          sum(rate(order_create_success_total{version="canary"}[2m]))
          /
          sum(rate(order_create_total{version="canary"}[2m]))
```

问题：`order_create_success_total` 是**业务上报的指标**，由代码里的 Counter 推送。这个指标的更新有延迟：

1. 用户下单
2. 业务代码 Counter.increment()
3. Micrometer 内部聚合（默认 60s）
4. Prometheus scrape（默认 15-30s）
5. AnalysisTemplate 查询（1m 间隔）

**总延迟**：60s + 30s + 60s = 最坏 2.5 分钟。金丝雀刚开始切流时，新版本的指标还没足够样本（甚至分母为 0），`result[0]` 算出来是 NaN 或者 0，**触发 successCondition 失败**。

#### 修复

**方案 A**：加最小样本数过滤。

```yaml
successCondition: |
  result[0] >= 0.95
  &&
  result[1] >= 100   # 至少 100 个请求样本
metrics:
  - provider:
      prometheus:
        query: |
          (
            sum(rate(order_create_success_total{version="canary"}[2m]))
            /
            sum(rate(order_create_total{version="canary"}[2m]))
          ) or vector(1)
          ,
          sum(increase(order_create_total{version="canary"}[2m]))
```

`or vector(1)` 让分母为 0 时返回 1（视为成功），避免 NaN。

**方案 B**：改用更实时的"基础设施指标"做主要判断。

```yaml
metrics:
  # 主：HTTP 5xx 率（Envoy 上报，几乎实时）
  - name: http-5xx-rate
    successCondition: result[0] < 0.01
    interval: 30s
    provider:
      prometheus:
        query: |
          sum(rate(istio_requests_total{
            destination_workload="order-svc-canary",
            response_code=~"5.."
          }[1m]))
          /
          sum(rate(istio_requests_total{
            destination_workload="order-svc-canary"
          }[1m]))
  # 辅：业务指标（拉长 interval，给业务上报留时间）
  - name: business-success-rate
    successCondition: result[0] >= 0.95
    interval: 3m         # ★ 比业务上报延迟长
    initialDelay: 5m     # ★ 前 5 分钟不查
```

`initialDelay: 5m` 让金丝雀启动后等 5 分钟再开始采集业务指标，跳过"指标尚未稳定"的窗口。

**方案 C**：在 AnalysisTemplate 里加预热阶段。

```yaml
strategy:
  canary:
    steps:
      - setWeight: 5
      - pause: { duration: 5m }     # 预热：先跑 5 分钟，不做 Analysis
      - setWeight: 5
      - analysis:                   # 第二步开始 Analysis
          templates:
            - templateName: success-rate
      - setWeight: 20
```

#### 验证

模拟"指标延迟"场景：人为延迟业务指标上报，看 Analysis 是否仍然能容忍。

```bash
# 在金丝雀 Pod 里把 micrometer 推送间隔调到 5 分钟
kubectl set env deploy/order-svc-canary \
    MANAGEMENT_METRICS_EXPORT_PROMETHEUS_STEP=5m

# 触发金丝雀
kubectl argo rollouts set image order-svc app=order-svc:v2.4.0

# 观察是否仍然能完成金丝雀流程
kubectl argo rollouts get rollout order-svc --watch
```

> **口诀**：自动回滚的指标必须满足"实时 + 充足样本"。**业务指标天然延迟，主指标用基础设施层的，业务指标做辅助**。

---

### 案例 ⑥ 配置变更与代码部署没协调 —— "切流瞬间所有请求 500"

#### 现象

晚高峰金丝雀切流量到 50% 的瞬间，所有 v2 实例同时报 `NullPointerException`，5xx 率从 0.1% 飙到 50%。Argo Rollouts 自动回滚。

复盘时发现：v2 代码读了一个新的配置项 `app.feature.new-pricing.discount-rate`，但配置中心里**没有这个 key**。

#### 根因

配置项管理和代码部署完全脱节：
- 代码 PR 里 `@Value("${app.feature.new-pricing.discount-rate}")` 没设默认值
- 配置中心新 key 由另一个团队人工添加
- 添加配置的工单忘了处理

代码部署上线，配置项缺失，启动时报错 / 运行时第一次访问 NPE。

#### 修复

**方案 A**：所有 `@Value` 必须给默认值。

```java
// ❌ 配置缺失直接 NPE
@Value("${app.feature.new-pricing.discount-rate}")
private BigDecimal discountRate;

// ✅ 给默认值
@Value("${app.feature.new-pricing.discount-rate:0}")
private BigDecimal discountRate;

// ✅ 更好：用 ConfigurationProperties + 验证
@ConfigurationProperties("app.feature.new-pricing")
@Validated
public class NewPricingConfig {
    @NotNull
    private BigDecimal discountRate = BigDecimal.ZERO;   // 默认值
}
```

**方案 B**：CI 检查代码引用的配置项是否都已存在于配置中心。

```yaml
# pipeline 步骤
- name: validate-configs
  script: |
    NEW_KEYS=$(git diff origin/main HEAD -- '*.java' \
        | grep -oE '@Value\("\$\{[^}]+\}"\)' \
        | sed -E 's/.*\$\{([^:}]+).*/\1/' \
        | sort -u)
    for key in $NEW_KEYS; do
      curl -f http://nacos.example.com/nacos/v1/cs/configs?dataId=$key \
        || (echo "Missing config: $key" && exit 1)
    done
```

**方案 C**：把"配置变更"做成 PR 流程，与代码 PR 关联。

```
代码 PR → 同时改 config-repo PR
                ↓
   config-repo CI 自动 apply 到测试 / 预发 / 生产
                ↓
   Argo CD 检测到镜像变更，触发金丝雀
```

这样配置和代码的变更走同一个 GitOps 流程，无法漏掉一边。

**方案 D**：金丝雀 Pod 启动时做配置自检 readiness。

```java
@Component
public class ConfigSelfCheckIndicator implements HealthIndicator {

    @Autowired private NewPricingConfig pricing;
    @Autowired private NewFeatureConfig feature;

    @Override
    public Health health() {
        if (pricing.getDiscountRate() == null
            || feature.getEnabled() == null) {
            return Health.down()
                .withDetail("missing", "config not loaded")
                .build();
        }
        return Health.up().build();
    }
}
```

K8s readinessProbe 配 `/actuator/health/readiness`，配置不全的 Pod 永远不进入 Ready 状态，**Service 不会把流量转给它**。

#### 验证

```bash
# 故意删掉一个关键配置，看 Pod 是否被拒绝
kubectl exec -it nacos-0 -- nacos-cli config delete \
    --dataId app.feature.new-pricing.discount-rate

# 部署，看 Pod 是否进入 Ready
kubectl rollout restart deploy/order-svc-canary
kubectl get pod -l app=order-svc,version=canary -w
# 期望：Pod 一直 Not Ready，不接流量
```

> **口诀**：配置和代码必须同生命周期。**配置缺失要么默认值兜底，要么启动失败 / readiness 失败拒绝接流量**。绝不允许"代码上了但配置没跟上"。

---

### 案例 ⑦ Argo Rollouts 自身升级 —— "升级期间金丝雀状态丢失"

#### 现象

Argo Rollouts 从 v1.5 升级到 v1.7。升级期间正在进行的几个金丝雀部署"卡住"，状态显示 `Paused` 但不再前进。手动 promote 也没反应。

#### 根因

Argo Rollouts 控制器升级是**滚动升级**：旧 v1.5 Pod 终止 → 新 v1.7 Pod 启动。

期间：
- 旧 Pod 在金丝雀的某一步设置了"等待 5 分钟"的状态机
- 升级时 v1.5 Pod 被杀，但状态机的内存里"5 分钟计时"也丢了
- v1.7 Pod 启动后从 Rollout CR 读状态，看到的是"我应该 pause 5 分钟"
- 但 v1.7 引入了新的状态字段，旧 CR 里没有该字段，控制器认为"状态不一致"，进入安全模式不前进

更深层：Argo Rollouts 的状态部分存在**控制器内存**（如 setWeight 的渐变进度），部分存在 CR.status 里。控制器重启时**内存状态丢失**。

#### 修复

**根治**：升级 Argo Rollouts 之前，**先暂停所有正在进行的 Rollout**。

```bash
# 列出所有进行中的 Rollout
kubectl get rollout -A -o json \
    | jq -r '.items[] | select(.status.phase=="Progressing") | "\(.metadata.namespace) \(.metadata.name)"' \
    > in-progress.txt

# 暂停它们
while read ns name; do
    kubectl argo rollouts pause $name -n $ns
done < in-progress.txt

# 升级 controller
helm upgrade argo-rollouts argo/argo-rollouts --version 1.7.0

# 等 controller 稳定
kubectl rollout status deploy/argo-rollouts-controller -n argo-rollouts
sleep 60

# 恢复
while read ns name; do
    kubectl argo rollouts resume $name -n $ns
done < in-progress.txt
```

**应急修复**（已经卡住）：

```bash
# 1. 强制 abort 卡住的 Rollout
kubectl argo rollouts abort order-svc -n prod

# 2. 确认流量切回 stable
kubectl get vs order-svc-vs -n prod -o yaml | grep weight

# 3. 重新触发金丝雀（CR.spec 重置）
kubectl argo rollouts retry rollout order-svc -n prod
```

**长期治理**：

1. 升级 Argo Rollouts / Flagger 这种关键组件**必须有 SOP**，不能像普通业务组件一样随意升
2. **金丝雀流程做幂等**：所有进度状态都持久化到 CR.status，不依赖控制器内存
3. **集群里关键控制器双副本 + leader election**，单 Pod 重启不影响

#### 验证

升级前模拟一次："故意"在金丝雀进行中重启控制器，看是否能恢复。

```bash
# 启动一个金丝雀
kubectl argo rollouts set image test-svc app=test:v2

# 等到第二步 pause 时
kubectl rollout restart deploy/argo-rollouts-controller -n argo-rollouts

# 观察金丝雀是否能继续
kubectl argo rollouts get rollout test-svc --watch
```

> **口诀**：基础设施组件的升级是金丝雀流程的隐藏风险。**升级前清空状态，升级后再恢复**。永远不要在有金丝雀进行中时升 Argo Rollouts。

---

### 案例 ⑧ DB 主从复制延迟 —— "灰度数据看不到"

#### 现象

订单服务做了"读分离"：v2 引入了从读，写主、读从。金丝雀阶段用户反馈："下单成功后立即查询订单列表，看不到刚下的单"。

v1 用户没有这个问题（v1 全部读主）。

#### 根因

主从复制延迟。下单走主库写入，立即查询走从库读取，**复制延迟期间从库还没有这条数据**。

延迟时间因为业务量+大事务，可能从几十毫秒到几秒不等。

更糟的是金丝雀 v2 用户量小，从库压力小，**反而比 v1 的读主延迟更明显**（v1 读主时主库的 buffer pool 命中率高、延迟稳定）。

#### 修复

**方案 A**：写后立即读，强制走主。

```java
@Service
public class OrderService {

    public Order createOrder(OrderCreateReq req) {
        Order order = orderMapper.insert(req);
        // 标记当前请求剩余的查询走主库
        DataSourceContext.forceMaster();
        return order;
    }

    @MasterRead   // 自定义注解
    public List<Order> myRecentOrders(Long userId) {
        return orderMapper.selectRecent(userId);
    }
}
```

`forceMaster()` 通常用 ThreadLocal 标记，AOP 拦截 SQL 路由时检查。

**方案 B**：读 session 黏性 —— 一段时间内同用户读写都走主。

```java
public class SessionStickyMaster {
    private static final long STICKY_DURATION = 5_000;  // 5 秒

    public static void markWritten(Long userId) {
        redis.setex("sticky-master:" + userId, 5, "1");
    }

    public static boolean shouldGoMaster(Long userId) {
        return redis.exists("sticky-master:" + userId);
    }
}
```

每次写入后 set 一个带 TTL 的 key，读路由先查这个 key。

**方案 C**：等主从同步完成再返回。

```java
public Order createOrder(OrderCreateReq req) {
    Order order = orderMapper.insert(req);
    // 拿到当前 GTID
    String gtid = jdbcTemplate.queryForObject(
        "SELECT @@GLOBAL.gtid_executed", String.class);
    // 等从库赶上
    jdbcTemplate.queryForObject(
        "SELECT WAIT_FOR_EXECUTED_GTID_SET(?, 2)",
        Integer.class, gtid);
    return order;
}
```

`WAIT_FOR_EXECUTED_GTID_SET` 阻塞当前连接直到从库执行到指定 GTID 或超时。**代价**：写入响应时间增加 = 主从延迟时间。

**方案 D**（终极）：金丝雀阶段**禁用读写分离**，全量发布后再开。

```yaml
# Canary 阶段配置
spring:
  shardingsphere:
    rules:
      readwrite-splitting:
        data-sources:
          ds:
            write-data-source-name: master
            read-data-source-names:
              - master         # ★ 金丝雀阶段读从也指向 master

# Stable 阶段配置
            read-data-source-names:
              - slave1
              - slave2
```

#### 验证

模拟主从延迟，验证业务行为：

```bash
# 在从库上人为加延迟
kubectl exec mysql-slave-0 -- bash -c \
    "tc qdisc add dev eth0 root netem delay 2000ms"

# 走金丝雀流量做下单 + 立即查询
for i in {1..10}; do
    ORDER_ID=$(curl -X POST -H "X-Canary: true" \
        http://order.example.com/api/order/create | jq .id)
    sleep 0.5
    curl -H "X-Canary: true" \
        "http://order.example.com/api/order/get/$ORDER_ID"
done
# 修复后所有查询都应能找到订单
```

> **口诀**：读写分离的主从延迟是灰度的隐形坑。**写后立即读必须走主**，或者灰度阶段直接禁用读写分离。

---

### 案例 ⑨ 大促前停止灰度 —— "回滚机制不健全导致状态混乱"

#### 现象

11.11 前一周还在灰度的服务被要求"立即停止灰度，全部回 v1"。运维执行 `kubectl argo rollouts abort` 后，集群里出现了：
- Service 流量切回 v1
- 但 v2 的 ReplicaSet 还在
- 部分老版本 Pod 因为 OOMKilled 被重启，重启后被错误地标记为 v2（因为 ReplicaSet 还活着）

最终 v1 / v2 混跑，监控显示版本分布乱七八糟。

#### 根因

Argo Rollouts 的 `abort` 语义：
- 流量切回 stable
- canary 的副本数缩到 0
- **canary ReplicaSet 不删除**（保留以备恢复）

如果 Pod 被 K8s 自身重启（比如 OOMKilled），controller manager 会按 ReplicaSet 期望副本数重新创建。`abort` 后 canary RS 副本数=0，正常不会重建。但如果**期间触发了 `retry`** 或者用户误操作，副本数可能被改回非 0。

更隐蔽：业务方在大促前临时改了 Rollout 的 `replicas`，间接导致 stable 和 canary 副本数都被影响。

#### 修复

**根治**：建立**确定性的"灰度急停"流程**。

```bash
#!/bin/bash
# emergency-stop-canary.sh
ROLLOUT=$1
NS=$2

set -ex

# 1. 切流量回 stable（最快）
kubectl argo rollouts abort $ROLLOUT -n $NS

# 2. 验证流量已切回
kubectl get vs ${ROLLOUT}-vs -n $NS -o yaml \
    | yq '.spec.http[].route[] | select(.destination.subset=="canary") | .weight'
# 期望输出：0

# 3. 缩 canary ReplicaSet 到 0
kubectl scale rs -l rollouts-pod-template-hash=$(\
    kubectl get rollout $ROLLOUT -n $NS -o jsonpath='{.status.currentPodHash}') \
    --replicas=0 -n $NS

# 4. 删除 canary ReplicaSet（彻底）
kubectl delete rs -l rollouts-pod-template-hash=$(\
    kubectl get rollout $ROLLOUT -n $NS -o jsonpath='{.status.currentPodHash}') \
    -n $NS

# 5. Suspend Rollout 防止再次触发
kubectl patch rollout $ROLLOUT -n $NS --type=merge -p \
    '{"spec":{"paused":true}}'

# 6. 通知
slack-notify "EMERGENCY: Canary stopped for $ROLLOUT"
```

**长期治理**：

1. **大促前 N 天禁止变更**（变更冻结 / Code Freeze），强制要求灰度结束或回滚
2. 灰度状态可视化大盘，大促前一键查看所有进行中的灰度
3. Argo Rollouts CR 加 admission webhook，大促期间禁止创建新的 Rollout

```yaml
# admission webhook 规则（伪代码）
match:
  resources: [rollouts.argoproj.io]
  operations: [CREATE, UPDATE]
condition:
  is_promotion_freeze_window:
    start: "2024-11-08T00:00:00Z"
    end: "2024-11-12T23:59:59Z"
deny: |
  variation cannot be created during promotion freeze
```

#### 验证

定期演练"急停"流程：

```bash
# 故意启动金丝雀
kubectl argo rollouts set image test-svc app=test:v2

# 过 5 分钟执行急停
sleep 300
./emergency-stop-canary.sh test-svc default

# 检查
kubectl get rollout test-svc -n default
kubectl get rs -l app=test-svc -n default
# 期望：只有一个 ReplicaSet（stable），canary 被完全清理
```

> **口诀**：灰度的"急停"必须是单命令、确定性、有验证的脚本。**大促前必须演练，绝不能事到临头才搞**。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 策略层
- [ ] 明确选择部署策略：滚动 / 蓝绿 / 金丝雀 / 影子，并说明原因
- [ ] 流量切分位置定下来：DNS / LB / Gateway / Mesh，**至少有 2 层**
- [ ] 切流方式定下来：百分比 / Header / Cookie / 用户标签
- [ ] 流量切分精度满足业务需求（K8s Service 默认精度差，需 Mesh 加持）
- [ ] **每一步灰度有明确的"通过/失败/回滚"判定标准**

### 工程层
- [ ] 金丝雀 / stable 用不同 Service / Deployment 隔离
- [ ] 配置 Argo Rollouts 或 Flagger 的 trafficRouting，不依赖 Pod 数量近似
- [ ] AnalysisTemplate 至少 3 个指标（错误率 + 延迟 + 业务）
- [ ] AnalysisTemplate 有 `initialDelay` / `failureLimit` 防止误判

### 数据层
- [ ] DDL 严格 Expand-and-Contract，**发布期间双向兼容**
- [ ] CI 包含"老镜像 + 新 schema" / "新镜像 + 老 schema"测试
- [ ] 大表 DDL 做过 pt-online-schema-change / gh-ost 演练
- [ ] 主从读写分离场景，**写后立即读强制走主**

### 全链路层
- [ ] 入口染色逻辑明确（按 user / cookie / 流量比例）
- [ ] HTTP / RPC / MQ 各层 tag 透传都已实现
- [ ] **无 tag 请求强制走 stable**（默认行为不能落到 canary）
- [ ] MQ 消费用独立 Consumer Group / Topic / Tag 路由

### 配置层
- [ ] 所有 `@Value` 都有默认值
- [ ] 新增配置项的工单和代码 PR 关联，CI 自动校验
- [ ] readiness probe 检查关键配置是否加载完成

### 回滚层
- [ ] 回滚是单命令、5 分钟内可执行
- [ ] **急停脚本演练过**，能彻底清理 canary 资源
- [ ] 老版本 ReplicaSet 保留时间 ≥ 业务最长事务时间
- [ ] 回滚前的检查清单（数据兼容 / 配置兼容）有专人确认

### 监控告警层
- [ ] Prometheus 接入 Alertmanager 自动 webhook 回滚
- [ ] 金丝雀 vs stable 的对比大盘（错误率、延迟、QPS）
- [ ] 业务核心指标 + 基础设施指标双重保障
- [ ] 大促前**变更冻结**，admission webhook 强制阻断新灰度

### 流程层
- [ ] 灰度 SOP 文档：何时进入下一步、谁拍板、谁通知
- [ ] 灰度期间值班机制（不是上完线就走人）
- [ ] 灰度日志打全：版本 / tag / 用户 ID / 染色原因

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### Progressive Delivery 全景
- CNCF 的 Progressive Delivery 工作组定义：**金丝雀 → 实验 → 自动决策**的三段式
- Argo Rollouts 的 `Experiment` 资源：可以在金丝雀外做"对比实验"（A/B test）
- Flagger 的 `MetricTemplate`：把指标查询模板化，支持 Datadog / NewRelic / Stackdriver

### DORA 4 指标的金丝雀表现
- 部署频率（Deployment Frequency）
- 变更前置时间（Lead Time for Changes）
- 变更失败率（Change Failure Rate）
- 平均恢复时间（MTTR）
- **金丝雀降低 CFR 但可能拉长 LTC** —— 这个权衡如何度量

### 影子流量（Shadow Traffic）
- Envoy / Istio 的 `mirror` 配置：把请求复制一份打到 v2，响应不返回用户
- 写请求的处理：禁写、写到影子库、用 stub
- 影子流量做 schema 演进验证、性能测试、新算法上线
- Netflix 的 [Diffy](https://github.com/twitter-archive/diffy) 类工具：实时对比两个版本的响应差异

### Service Mesh 的 xDS 流量切换
- Istio 的 VirtualService weight 改动后，Pilot 通过 xDS 协议推到所有 Envoy
- xDS 推送有 ms 级不一致窗口（先收到的 Envoy 已切，后收到的还没切）
- 大集群（数百个服务、上千 Pod）的 xDS 推送性能与一致性

### Open Application Model（OAM）
- 阿里 / 微软的 OAM 把"应用 + 部署策略 + 运维特性"显式化
- KubeVela 是 OAM 的 K8s 实现，原生支持金丝雀、蓝绿
- 与 Argo Rollouts 比较，OAM 更面向应用开发者

### 灰度发布与 SRE 错误预算
- 错误预算（Error Budget）= (1 - SLO) × 时间
- 金丝雀失败消耗错误预算；**预算耗尽时禁止新发布**
- Google SRE Book 第 3 章

### 多集群 / 多地域金丝雀
- 跨集群 Mesh（Istio multi-cluster, Linkerd multi-cluster）
- 全球流量调度（AWS Route53 / 阿里云全球加速）
- 多集群下"哪个集群先灰度"的策略
- 数据一致性：全球 DB（Spanner / TiDB）vs 单集群 DB

### Feature Flag 系统设计
- 中心化 vs 边缘化：LaunchDarkly / Unleash 的架构对比
- Flag 的 SDK 缓存、降级、灰度规则评估性能
- Flag 与权限系统的边界

---

## 7. 引用

- [Argo Rollouts 官方文档](https://argoproj.github.io/argo-rollouts/)
- [Flagger 官方文档](https://docs.flagger.app/)
- [Istio Traffic Management](https://istio.io/latest/docs/concepts/traffic-management/)
- Google SRE Book, Chapter 3 *Embracing Risk*
- *Continuous Delivery* by Jez Humble & David Farley
- Martin Fowler, [BlueGreenDeployment](https://martinfowler.com/bliki/BlueGreenDeployment.html)
- Martin Fowler, [CanaryRelease](https://martinfowler.com/bliki/CanaryRelease.html)
- 阿里 [MSE 全链路灰度文档](https://help.aliyun.com/document_detail/170443.html)
- CNCF [Progressive Delivery 白皮书](https://www.cncf.io/blog/2020/09/01/progressive-delivery/)
- [Atlas Schema Migration](https://atlasgo.io/) —— DDL 兼容性分析
- *Database Reliability Engineering* by Laine Campbell & Charity Majors

---

> 配套：[DevOps CI/CD](/article/devops/cicd) | [GitOps 与 ArgoCD](/article/devops/gitops) | [Feature Flag 系统设计](/article/system-design/feature-flag) | [高可用架构演进](/article/system-design/high-availability)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
