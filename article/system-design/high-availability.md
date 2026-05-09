# 高可用架构演进 —— 从单机到异地多活的每一步

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，看 [系统设计速记](/article/system-design/) 那一节就够了。
> 这里要回答的，是**做架构评审被问"我们能不能扛住一个机房挂掉"、半夜机房光纤被挖断时**你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. "可用性 4 个 9" 一年最多宕多久？5 个 9 呢？6 个 9 呢？把后两个 9 加上去的**真实成本**是什么？
2. 同城双活和异地多活的**本质差别**是什么？为什么"双活"在很多公司只是双数据中心，而不是双活？
3. 单元化（Cell-based）架构是什么？它和"分库分表"、"微服务拆分"在哪一层做的取舍是不同的？
4. 大厂都在 PPT 里讲"异地多活"，但真正做到双向写、可切流的不超过 5 家。**为什么吹得多做得少**？
5. 写流量分发到两个机房，DB 双向同步，用户改密码后立即登录、刚好被路由到另一个机房 —— 你要怎么解决这个一致性问题？
6. RTO 和 RPO 是什么？给一个定义不严谨就会让你赔钱的例子。
7. 混沌工程（Chaos Engineering）解决的是高可用建设的**哪一类**问题？为什么定期演练比"代码 Review"更重要？
8. "故障注入"具体注入什么？写一段你会真的在生产灰度里跑的代码。

如果有半数答不上，往下看。

---

## 1. 底层模型：可用性、故障域与单元

绝大多数文章把高可用讲成"加机器、做主备、上 ZooKeeper"，然后画一张三机房图就结束了。这层认知不够指导真正的架构决策。要看懂上面 8 个问题，必须先把可用性这件事的**经济模型**和**物理边界**讲清楚。

### 1.1 可用性等级与代价（每加一个 9 成本翻倍）

业界把可用性按 9 的个数分级。一年 365 天 × 24 小时 = 525,600 分钟。可用性 X 意味着每年最多宕机 `525600 × (1 - X)` 分钟：

| 可用性 | 一年最大宕机时间 | 典型实现 | 大致年成本（量级） |
| --- | --- | --- | --- |
| **99%**（2 个 9） | 87.6 小时 | 单机 + 重启 | 1× |
| **99.9%**（3 个 9） | 8.76 小时 | 主备 + 监控 | 2~3× |
| **99.99%**（4 个 9） | 52.6 分钟 | 同城双机房 + 自动切换 | 5~10× |
| **99.999%**（5 个 9） | 5.26 分钟 | 同城多活 + 异地灾备 | 20~50× |
| **99.9999%**（6 个 9） | **31.5 秒** | 异地多活 + 单元化 + 混沌演练 | 100×+ |

每加一个 9 成本不是"翻倍"那么轻 —— 是**指数级**的。从 4 个 9 到 5 个 9，你要新建一个机房、招一支 SRE 团队、做单元化改造、跨机房专线（一根专线 100Gbps 一年百万级）；从 5 个 9 到 6 个 9，你要把所有依赖（DNS、CA、第三方支付、短信网关）也做到 6 个 9，这往往不在你控制范围内。

**关键认知**：99.99% 是绝大多数 To C 业务的合理目标。99.999% 只有金融核心、铁路购票、大促首页这种"宕一秒上头条"的业务才值得做。**99.9999% 在我国除了运营商核心网、银联核心交换，没有几家真正达成**。

> **口诀**：定可用性目标先看 SLA 罚金 / 业务 GMV 损失，再看实现成本，**别拍脑袋追 9**。

### 1.2 故障域（Failure Domain）与爆炸半径

"高可用"不是抽象概念，而是"任何一个故障域挂掉，业务能不能继续"。故障域是**物理上同时挂的最小集合**：

```
                故障域（由小到大）
单进程 < 单机 < 单机柜 < 单机房 < 单 AZ < 单 Region < 单运营商 < 单云厂商
   ↑       ↑       ↑       ↑       ↑       ↑          ↑          ↑
 OOM    硬件故障  ToR     断电     地震     断网       骨干网     云厂商整挂
```

每一层故障的**真实概率**（按行业经验估算）：

| 故障域 | 年发生概率 | 典型时长 | 实例 |
| --- | --- | --- | --- |
| 单进程 | ~50%（按月看几乎必发） | 秒级 | OOM、coredump、deadlock |
| 单机 | ~5% | 小时级 | 内存条故障、电源烧 |
| 单机柜 | ~1% | 小时级 | ToR 交换机挂、机柜断电 |
| 单机房 | ~0.1% | 数小时 | 制冷故障、电力切换失败 |
| 单 AZ | ~0.05% | 数小时 | 整片光纤被挖断 |
| 单 Region | ~0.01% | 不可预期 | 地震、洪水、政策性切断 |
| 单云厂商 | 数年一次 | 数小时～数天 | AWS us-east-1 大故障史 |

**爆炸半径**指一次故障会影响多少用户。高可用架构的核心目标是：**让爆炸半径最小化**，并且**让恢复时间可预期**。

如果你部署在单云单 Region，那这个云厂商的故障 = 你的故障。2017 年 AWS S3 us-east-1 挂了 4 小时，全球半个互联网瘫痪 —— 你以为你在用 AWS S3 的"四个九"，实际你用的是 us-east-1 这一个 Region 的可用性。

> **口诀**：算可用性必须按"最大故障域"算。**单 Region 部署的服务，可用性上限就是这个 Region 的可用性**。

### 1.3 4 种部署形态

```
① 单机                    ② 主备                    ③ 同城双活                ④ 异地多活
┌─────┐                   ┌─────┐ ┌─────┐           ┌─────┐ ─◀─▶─ ┌─────┐    ┌─────┐ ─◀─▶─ ┌─────┐
│  A  │                   │  A  │ │  B  │           │ AZ1 │       │ AZ2 │    │ R1  │       │ R2  │
└─────┘                   └─────┘ └─────┘           └─────┘       └─────┘    └─────┘       └─────┘
                            主    备(冷/热)         同城 < 50km             跨城 > 1000km
单点 SPOF                 切换需人工介入           写主一边/双写            双写 + 单元路由
RTO=∞                     RTO 分钟级              RTO 秒级                  RTO 秒级
                                                                            RPO 接近 0
```

| 维度 | 单机 | 主备 | 同城双活 | 异地多活 |
| --- | --- | --- | --- | --- |
| **物理距离** | 0 | 同机房 | 同城 < 50km | 跨城 > 1000km |
| **延迟** | 0 | < 1ms | 1~3ms | 20~100ms |
| **RPO** | 不可控 | 秒～分钟 | 接近 0 | 接近 0（设计妥当时） |
| **RTO** | 重启时间 | 分钟～小时 | 秒级 | 秒级 |
| **故障域覆盖** | 单进程 | 单机 | 单机房 | 单 Region / 单城市 |
| **数据一致性** | 强 | 异步弱 | 强（同步复制） | 最终一致 |
| **改造代价** | 0 | 低 | 中 | 极高 |

**坑**：很多公司说自己"同城双活"，实际是"同城双机房，但写流量只在一边"。真正的双活要求**两边都接写**且**任意一边挂掉对端能立即扛住全量**。后者意味着你的容量永远要按 2× 准备 —— 这是大厂"吹得多做得少"的根本原因。

### 1.4 单元化（Cell）架构的本质：完整服务栈复制

异地多活的核心难点是**写冲突**。如果两个机房同时改一行数据，谁说了算？传统 DB 双向同步会死在这个问题上。

阿里、蚂蚁、字节的解决方案是**单元化**：把全量用户按某个维度（通常是 UID）切片，每片用户的**写流量永远去同一个单元**。这样就不存在跨单元的并发写。

```
                                                  (按 UID 路由)
                          ┌──────────────────────────────────────────┐
                          │                                          │
                          ▼                                          ▼
                ┌─────────────────────┐                    ┌─────────────────────┐
                │   Cell-A (杭州)     │                    │   Cell-B (上海)     │
                │ ─────────────────── │                    │ ─────────────────── │
   UID%2==0 ──▶ │ 接入层 → 应用层    │                    │ 接入层 → 应用层    │ ◀── UID%2==1
                │           ↓        │                    │           ↓        │
                │       业务 DB-A    │ ◀───单向同步───▶  │       业务 DB-B    │
                │       (UID%2==0    │      (异步 / 双写)  │       (UID%2==1    │
                │       的全量数据)  │                    │       的全量数据)   │
                └─────────────────────┘                    └─────────────────────┘
                          │                                          │
                          └────── 全局数据 ──────────────────────────┘
                                  (商品、配置：单写多读)
```

单元的关键属性：

1. **完整服务栈**：每个单元都有自己的接入层、应用层、缓存、DB。**单元内自治**，不依赖另一个单元也能完成业务。
2. **UID 维度切片**：路由规则在最外层（GSLB / 接入层），用户的所有请求只去一个单元。
3. **数据按维度独立**：每个单元的 DB 只放属于自己的用户数据。
4. **全局数据例外处理**：商品、配置这种全用户共享的数据走"单写多读"模式（一个单元写，所有单元读）。

为什么这套设计能成？**因为消除了跨单元的写冲突**。冲突只可能发生在单元切换的瞬间（用户从 Cell-A 切到 Cell-B），这是个可控的窄窗口，可以用版本号 + 写禁/读禁配合解决。

### 1.5 数据多活的三个模式

不是所有数据都能/需要走单元化。生产中按数据特性走三种模式：

| 模式 | 适用数据 | 实现方式 | 一致性 |
| --- | --- | --- | --- |
| **主备（Active-Standby）** | 强一致性要求高、写少 | 主写 + 异步/同步复制 | 强（同步）/ 最终（异步） |
| **多写（Multi-Master）** | 用户维度可切分 | 单元化路由 + 单元内主写 | 单元内强、跨单元最终 |
| **多读单写（Single-Writer）** | 读多写少、全局数据 | 一个机房写、所有机房读 | 读到的是某瞬间的快照 |

```
主备                    多写（单元化）              多读单写（全局）
                                                    ┌──────┐
┌──A(主)──┐             ┌─Cell-A─┐ ┌─Cell-B─┐     │  A   │ (写)
│         │ → 复制 →    │ 写A的  │ │ 写B的  │     │  写  │
│  B(备)  │             │ 数据   │ │ 数据   │     └──┬───┘
└─────────┘             └────────┘ └────────┘        │
                                                     ▼ 复制
                                              ┌──────┴──────┐
                                              ▼             ▼
                                          ┌──────┐      ┌──────┐
                                          │ A 读 │      │ B 读 │
                                          └──────┘      └──────┘
```

生产中三种往往同时存在：用户数据走多写、商品库走多读单写、配置中心走主备。**架构师的工作不是"选一种"，而是把每张表归类到正确的模式**。

### 1.6 流量分发：DNS / Anycast / GSLB / 单元化路由

异地多活离不开"把对的请求送到对的单元"。流量分发分四层：

```
用户  ─▶  ① DNS                  域名 → IP（按地理 / 按权重）
       │  ② Anycast               同一 IP 在多地宣告，BGP 选最近
       │  ③ GSLB（全局负载均衡）  根据健康度 / 容量动态返回 IP
       │  ④ 单元化路由            HTTP 入口按 UID 转发到目标单元
       ▼
     单元 (Cell)
```

**每一层的特性和坑**：

- **DNS**：最古老、最便宜，但有 **TTL 缓存陷阱**。运营商 LocalDNS 会忽略你设的 TTL（设 60 秒可能缓存 1 小时），故障切换后用户还在访问宕机 IP。
- **Anycast**：用 BGP 协议让同一 IP 在多地宣告，路由器自动选最近的。优点是切换无需 DNS 介入；缺点是需要自有 AS（自治系统）号，不是所有公司能玩。
- **GSLB**：基于 HTTP/302 或 DNS 的智能调度，能根据后端健康度、容量、延迟做决策。商业产品（F5 GTM、阿里云 GTM、CloudFlare LB）成熟，自建复杂度极高。
- **单元化路由**：HTTP 入口层（Nginx / Tengine / Envoy）解析 UID，按规则转发到目标单元。这是"用户永远在同一个单元"的执行者。

> **口诀**：高可用的流量层 = **DNS 切大方向 + Anycast/GSLB 切机房 + 单元路由切单元**。三层各管一段，**TTL 必须短**。

---

## 2. 源码切片：核心机制

> 基于阿里 Tengine、蚂蚁 OceanBase、Karmada、Istio 等真实开源代码片段。注释为本文添加。

### 2.1 阿里单元化路由：UID hash 到 Cell 的算法

蚂蚁的"逻辑机房（LDC）"路由规则简化版：

```java
// 把 UID 末位映射到一个 0-99 的"分片号"，再把分片号映射到机房
public class UnitRouter {

    // 分片号 → 单元（机房）的映射表，运维下发，可热更新
    private volatile Map<Integer, String> shardToUnit;

    public String route(String uid) {
        // ① 取 UID 后两位，做模运算得到分片号
        int shard = Math.abs(uid.hashCode()) % 100;

        // ② 查映射表
        String unit = shardToUnit.get(shard);
        if (unit == null) {
            throw new IllegalStateException("no unit for shard " + shard);
        }
        return unit;
    }

    // 切流操作 = 修改映射表
    public void migrateShard(int shard, String fromUnit, String toUnit) {
        // 实际生产里这里有"禁写 + 等待复制完成 + 切映射 + 解禁"四步
        shardToUnit.put(shard, toUnit);
    }
}
```

**逐段点透**：

- **为什么是 100 而不是 10000？** 100 个分片足够支持 100 个单元的细粒度切流（最多）。分片粒度太细管理复杂；太粗不能精确灰度。100 是行业惯用值。
- **`hashCode()` 不是好选择**：同样 UID 在不同 JDK 版本可能 hash 不同。生产里要用稳定 hash（如 MurmurHash、CityHash），且 hash 算法**永远不能变**，否则历史数据全错位。
- **切流不是原子的**：把 shard 5 从 Cell-A 迁到 Cell-B，必须先**禁写 Cell-A 上 shard 5 的数据**，等 Cell-A 的 binlog 同步到 Cell-B（确保 Cell-B 拿到最新版本），再修改路由表，最后**禁读 Cell-A 的 shard 5**。整个过程秒级，但每一步都不能漏。

### 2.2 蚂蚁 OceanBase 多机房部署的 Paxos 配置

OceanBase 用 Paxos 做强一致性多副本。三机房 5 副本配置（`zone` = 机房）：

```sql
-- 创建一个 5 副本资源池，分布在 3 个 zone
CREATE RESOURCE POOL pool_5
    UNIT='unit_2c8g',
    UNIT_NUM=5,
    ZONE_LIST=('zone_hz_1', 'zone_hz_2', 'zone_sh_1');

-- 创建租户，3 个 zone 副本数 2/2/1
CREATE TENANT tenant_main
    RESOURCE_POOL_LIST=('pool_5'),
    LOCALITY='F@zone_hz_1, F@zone_hz_1, F@zone_hz_2, F@zone_hz_2, F@zone_sh_1';
    -- F 表示 Full 副本（参与 Paxos 选举）
```

**逐段点透**：

- **5 副本而不是 3 副本**：Paxos 需要"多数派"才能写。5 副本可以容忍 2 副本同时挂；3 副本只能容忍 1 副本挂。**机房整挂 + 单机故障**这种组合下，5 副本能扛、3 副本不能。
- **2/2/1 分布**：杭州两个机房各 2 副本（同城多数派），上海 1 副本（异地灾备）。任何一个杭州机房挂掉，剩下 2+1=3 副本仍能形成多数派写入；杭州整体挂掉，上海 1 副本不够多数派 → 数据可读不可写（这是异地灾备的合理代价）。
- **为什么不 1/1/3 上海多数派？** 那么写延迟取决于上海的 Paxos 日志同步延迟（杭州→上海 30ms+），所有写都要跨城 ack，吞吐降一个数量级。

更激进的"三地五中心"：

```sql
LOCALITY='F@zone_hz_1, F@zone_hz_2, F@zone_sh_1, F@zone_sh_2, F@zone_sz_1'
```

杭州 2、上海 2、深圳 1。任何一个城市整挂，其他两个城市的 4 副本仍能多数派。代价：写延迟取决于"第二近的城市"的 ack（通常是杭州→上海 30ms）。

### 2.3 K8s 多集群部署（Karmada）

Karmada 是 CNCF 多集群管理项目，把"在哪个集群跑多少副本"声明化：

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: nginx-multicluster
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx
  placement:
    clusterAffinity:
      clusterNames:
        - cluster-hangzhou
        - cluster-shanghai
        - cluster-shenzhen
    replicaScheduling:
      replicaSchedulingType: Divided
      replicaDivisionPreference: Weighted
      weightPreference:
        staticWeightList:
          - targetCluster:
              clusterNames: [cluster-hangzhou]
            weight: 5
          - targetCluster:
              clusterNames: [cluster-shanghai]
            weight: 3
          - targetCluster:
              clusterNames: [cluster-shenzhen]
            weight: 2
```

**逐段点透**：

- **Divided + Weighted**：总副本数（假设 100）按权重 5:3:2 分到三个集群 = 50/30/20。这是异地多活的"流量比例"在 K8s 层的表达。
- **故障域和权重的关系**：单个集群挂了，Karmada 会把这个集群的 Pod 数转嫁到其他集群（如果配了 `failoverBehavior`）。但你必须**预留容量**，否则故障切换 = 把另两个集群打挂（案例 ⑦）。
- **真正的坑不在 Karmada 本身**：是底层集群的 etcd、CoreDNS、CNI 是不是真的隔离。如果三个集群共享一个 ZooKeeper 注册中心，注册中心挂 = 三集群同时挂。

### 2.4 Service Mesh（Istio）的多机房流量管理

跨机房流量切换在 Istio 里用 `DestinationRule` + `VirtualService`：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service.svc.cluster.local
  subsets:
    - name: hangzhou
      labels: { region: hz }
    - name: shanghai
      labels: { region: sh }
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-routing
spec:
  hosts: [payment-service.svc.cluster.local]
  http:
    - match:
        - headers:
            x-uid-shard:    # 单元化路由头，由入口层注入
              regex: "^[0-4][0-9]$"   # shard 0-49
      route:
        - destination: { host: payment-service.svc.cluster.local, subset: hangzhou }
    - route:    # 默认路由：去上海
        - destination: { host: payment-service.svc.cluster.local, subset: shanghai }
```

**逐段点透**：

- **`x-uid-shard` 头由谁注入**：入口层（Tengine / Envoy）在第一跳就计算好 UID 的 shard 写到 header 里，下游所有服务直接读 header，不用每跳都重算。
- **regex 匹配 shard**：用正则匹配前缀范围，方便批量切流。把 `^[0-4][0-9]$` 改成 `^[0-9][0-9]$` = 全切到杭州（紧急逃生用）。
- **默认路由是"兜底"**：万一 UID 计算失败、header 丢失，请求至少能进入一个机房，而不是 5xx。

### 2.5 故障切换的 health check 与流量切换协调

故障切换的关键是"health check 准确 + 切流原子"。一个常见的协调器实现：

```go
type FailoverCoordinator struct {
    healthChecker  HealthChecker      // 探活
    dnsManager     DNSManager         // 改 DNS
    lbManager      LBManager          // 改 LB 后端
    notifier       Notifier           // 通知
    stateLock      sync.Mutex
}

func (c *FailoverCoordinator) OnRegionDown(region string) {
    c.stateLock.Lock()
    defer c.stateLock.Unlock()

    // ① 二次确认：避免误判触发切换
    if !c.healthChecker.ConfirmDownAfter(region, 30*time.Second, 3) {
        log.Warn("region %s recovered before failover triggered", region)
        return
    }

    // ② 通知人工：自动切换前必须 SRE 确认（很多公司这一步是必需的）
    if !c.notifier.GetHumanApproval(region, 60*time.Second) {
        log.Error("failover for %s aborted: no human approval", region)
        return
    }

    // ③ 摘除 LB 后端（最快）
    c.lbManager.RemoveBackend(region)

    // ④ DNS 切换（慢，但能影响新连接）
    c.dnsManager.UpdateRecord(region, []string{"backup-region"})

    // ⑤ 等待 TTL 过期后验证流量真的切走了
    time.Sleep(c.dnsManager.TTL() + 30*time.Second)

    if !c.healthChecker.NoTrafficTo(region) {
        c.notifier.Alert("traffic NOT cleared from %s after DNS TTL", region)
    }
}
```

**逐段点透**：

- **二次确认 `ConfirmDownAfter(30s, 3)`**：30 秒内连续 3 次确认才算挂。避免一次网络抖动触发切流。生产中很多事故就是误判（监控自己挂了 → 以为业务挂了 → 切流 → 把好机房压垮）。
- **GetHumanApproval**：是自动还是人工？大厂的实践是**RTO < 1 分钟必须自动**（人来不及），**RTO 5 分钟以上必须人工**（避免误切）。1~5 分钟之间各家不同。
- **LB 摘除在前，DNS 在后**：LB 摘除是秒级生效（只影响 LB 后面的实例），DNS 是分钟级生效（要等 TTL）。先摘 LB 让用户立即停止访问宕机机房；DNS 改了再让新连接绕过。
- **最后的 NoTrafficTo 验证**：很多公司切完不验证就以为完事，结果 LocalDNS 还在缓存老 IP，用户还在访问 —— 案例 ④ 就是这个。

### 2.6 数据冲突检测与解决（CRDT / 最后写入获胜 / 业务自定义）

异地多活无法避免短暂的"双写"窗口（切流瞬间、网络分区恢复时）。冲突解决三种主流方案：

**方案 A：最后写入获胜（LWW, Last-Write-Wins）**

```sql
-- 每行加一个 update_time，冲突时取大的
UPDATE user SET name='Alice', update_time=NOW(6)
WHERE uid=123 AND update_time < NOW(6);
```

简单粗暴，但有**时钟漂移问题**：两个机房时钟差 100ms，"晚提交的写"可能时间戳更小，被丢弃。

**方案 B：CRDT（Conflict-free Replicated Data Types）**

适合特定数据类型（计数器、集合）。比如计数器：

```java
// G-Counter：增加型计数器（不能减）
class GCounter {
    Map<String, Long> counts = new HashMap<>();   // nodeId → count

    void increment(String nodeId) {
        counts.merge(nodeId, 1L, Long::sum);
    }

    long value() {
        return counts.values().stream().mapToLong(Long::longValue).sum();
    }

    // 合并两个副本：每个 nodeId 取最大值
    GCounter merge(GCounter other) {
        GCounter result = new GCounter();
        Set<String> all = new HashSet<>(counts.keySet());
        all.addAll(other.counts.keySet());
        for (String node : all) {
            result.counts.put(node,
                Math.max(counts.getOrDefault(node, 0L),
                         other.counts.getOrDefault(node, 0L)));
        }
        return result;
    }
}
```

CRDT 的数学保证：**任意顺序合并、任意次数合并，结果一致**。代价是只支持特定数据结构。

**方案 C：业务自定义合并**

复杂业务（订单、用户资料）通常需要业务侧定义冲突解决逻辑：

```java
public Order resolve(Order localVersion, Order remoteVersion) {
    // 状态机优先：已支付 > 待支付 > 已取消
    if (localVersion.status.priority() > remoteVersion.status.priority()) {
        return localVersion;
    }
    if (remoteVersion.status.priority() > localVersion.status.priority()) {
        return remoteVersion;
    }
    // 同状态：按 update_time
    return localVersion.updateTime > remoteVersion.updateTime
        ? localVersion : remoteVersion;
}
```

**关键认知**：单元化做到位的话，**生产中真实冲突极少**（只发生在切流瞬间）。所以业务自定义合并 80% 的代码是"防御性的"，平时跑不到。但你必须有 —— 没有，那 0.01% 的窗口就是事故现场。

---

## 3. 实测数据：高可用的物理上限

### 3.1 单机 / 主备 / 双活 / 多活 的 RTO / RPO 对比

**RTO（Recovery Time Objective）**：故障到恢复的最大可接受时间。
**RPO（Recovery Point Objective）**：故障可丢失的最大数据时间窗口。

| 部署形态 | 典型 RTO | 典型 RPO | 数据丢失场景 |
| --- | --- | --- | --- |
| **单机** | 重启时间（数分钟到数小时） | 上次备份的间隔（数小时到一天） | 磁盘损坏 = 数小时数据丢失 |
| **主备 + 异步复制** | 1~10 分钟（人工切换）/ 30 秒（自动） | 1~10 秒（复制延迟） | 主挂瞬间未复制的数据丢失 |
| **主备 + 半同步** | 同上 | 0（如果半同步生效） | 半同步退化为异步时丢 |
| **同城双活 + Paxos** | 30 秒 | 0 | 多数派写入，多数派挂才会丢 |
| **异地多活 + 单元化** | 30~120 秒（要等切流生效） | 0（按单元划分，无跨单元写） | 切流瞬间写错单元的数据 |

**坑**：RPO=0 是个**理论值**，前提是"复制机制完全工作"。生产中半同步可能因为超时退化为异步、Paxos 可能因为脑裂暂停写入、单元化可能因为路由抖动写错单元。**真实 RPO 必须按"最坏情况"算**。

### 3.2 单元化前 vs 后跨机房请求比例

单元化的最关键指标不是 RTO，而是**跨机房请求比例**。一次跨机房 RPC = 30~100ms 延迟 + 跨机房专线带宽消耗。

| 场景 | 跨机房请求比例 | 用户感知 |
| --- | --- | --- |
| **单元化前**（普通双机房） | 30~50%（按服务依赖随机） | P99 200ms+ |
| **单元化后**（按 UID） | < 5%（仅全局数据访问） | P99 50ms |
| **单元化后**（业务异常路由） | 突增到 30%+（告警信号） | P99 飙升 |

**关键认知**：单元化的 KPI 应该是"跨机房比例 < 5%"，而不是"机房数量"。如果你做了双机房但跨机房比例还是 30%，那是双机房不是双活。

### 3.3 故障切换的实际秒数

```
                    总切换时间 = ① 探活 + ② 决策 + ③ LB 摘除 + ④ DNS 生效 + ⑤ 数据一致性确认
   时间分布          30秒        10秒    5秒        TTL+缓冲      变量
                    ▼          ▼      ▼          ▼              ▼
   故障发生 ─────────●──────────●──────●──────────●──────────────●──── 全量恢复
                    ↑                                            ↑
                  T=0                                        T = 60~180秒（典型）
```

各阶段实测耗时（自有 IDC 经验值）：

| 阶段 | 自动切换 | 人工切换 |
| --- | --- | --- |
| 探活确认（去抖） | 15~30 秒 | 同 |
| 决策（脚本/审批） | 1 秒 / 5~30 分钟 | 5~30 分钟 |
| LB 后端摘除 | 1~5 秒 | 同 |
| DNS TTL 过期 | TTL（设 60s 则 60s） | 同 |
| 数据一致性等待 | 0（双活）/ 复制 lag（主备） | 同 |
| **总计** | **45~120 秒** | 10~40 分钟 |

**LocalDNS 的真实 TTL**：你设 60 秒，实际生效要看运营商：

| 运营商 LocalDNS | 真实生效时间 | 备注 |
| --- | --- | --- |
| 公司内网 DNS（自控） | 60 秒（按设置） | 唯一可信的 |
| 联通 / 电信 / 移动城域 | 5~30 分钟 | 各家不同 |
| 公共 DNS（114 / 8.8.8.8） | 60 秒 ~ 5 分钟 | 较好 |
| 部分手机基站 DNS | 1~24 小时 | 最差 |

**这就是为什么"DNS 切换 + 老连接保护"必须配套**：DNS 改了不代表所有用户立刻去新机房，老 IP 上的实例必须**保持运行**（即使不再接新请求）直到老连接全部超时。

### 3.4 异地多活下数据库写入延迟的物理下限

光速 30 万公里/秒。光纤传输实际约 20 万公里/秒（有折射率 1.5 的损耗）。

| 距离 | 单程延迟（理论） | 单程延迟（实际） | 一次 RTT |
| --- | --- | --- | --- |
| 同机房 | < 0.01ms | < 0.5ms | < 1ms |
| 同城 50km | 0.25ms | 1~3ms | 2~6ms |
| 杭州-上海 200km | 1ms | 5~10ms | 10~20ms |
| 北京-上海 1300km | 6.5ms | 15~25ms | 30~50ms |
| 北京-广州 2200km | 11ms | 25~40ms | 50~80ms |
| 跨大洲（如中美） | 60~80ms | 100~200ms | 200~400ms |

**关键认知**：跨城同步复制**至少 30ms**。如果你的事务里有 10 次跨城写，事务延迟至少 300ms。这是**物理学决定的下限**，再优化也降不下去。

这就是为什么真正的异地多活只能用**单元化避免跨城写**，而不是依赖"跨城同步复制"。后者的延迟成本任何业务都接受不了。

### 3.5 故障注入演练前后的"事故响应时间"

**MTTR（Mean Time To Recovery）**：从故障发生到恢复的平均时间。是高可用最有用的单一指标。

某团队（自有数据，量级供参考）半年混沌演练前后对比：

| 故障类型 | 演练前 MTTR | 演练后 MTTR | 改善原因 |
| --- | --- | --- | --- |
| 单实例 OOM | 8 分钟 | 45 秒 | k8s liveness probe 配对了 |
| 单 AZ 网络抖动 | 25 分钟 | 3 分钟 | 健康检查权重配置正确 |
| 主库挂掉 | 40 分钟（人工切换） | 90 秒（自动切换） | DBA 流程脚本化 |
| 单机房整挂 | 从未演练过 = 不可知 | 4 分钟 | 单元化切流脚本上线 |
| 第三方依赖（短信网关）挂 | 全站挂 | 局部降级 | 熔断 + 备用通道 |

**关键认知**：MTTR 的改善 90% 来自"演练发现的盲区"，而不是"代码 review 的优化"。**没演练过的故障 MTTR 总是不可知**，而不可知 = 客观上无穷大。

> **口诀**：高可用的真实指标是 MTTR，不是 SLA。**没演练过 = 不可知 = 等同于不可用**。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 双活回切失败 —— "主机房恢复后切回去发现数据不一致"

#### 现象

杭州主机房挂了 30 分钟，自动切流到上海。一个小时后杭州恢复，运维想"切回去"，结果发现：杭州的 DB 里数据停留在故障前那一刻，上海这一小时的写入还没同步过来。直接切流回去会让用户"看到自己一小时前的数据"。

#### 根因

故障切流时通常配置了"切流后停止反向同步"（避免数据来回乱跑）。具体场景：

```
正常态：杭州 ⇄ 上海 双向同步
        ↓
故障时：杭州挂，上海接全量，关闭杭州 → 上海 同步（防止杭州起来后老数据覆盖新数据）
                           只保留上海 → 杭州 同步
        ↓
杭州恢复：上海 → 杭州 的同步通道**没有自动开**（运维忘了/脚本没配）
        ↓
直接回切：杭州数据 = 故障前的旧数据 → 用户看到旧数据
```

主备模式下也常见这个坑：故障切到备库后，备库就成了"主"，原主恢复后是"备"。但很多人想当然地把它当回主库用。

#### 修复

**回切的正确流程（4 步法）**：

```
1. 反向同步：开启 上海 → 杭州 的复制通道
2. 等同步追平：监控复制 lag，必须 < 1 秒持续 5 分钟
3. 杭州预热：把流量按 1% / 5% / 20% / 50% 灰度切回，观察错误率
4. 全量切回：错误率正常后切到 100%，但保留 上海 → 杭州 的同步（恢复双活）
```

代码层的"安全切回"模板：

```go
func (c *Coordinator) FailbackTo(primary, backup string) error {
    // 第一步：开反向同步，且必须确认开启成功
    if err := c.replicator.Resume(backup, primary); err != nil {
        return fmt.Errorf("resume replication failed: %w", err)
    }

    // 第二步：等同步追平（这一步可能要小时级）
    deadline := time.Now().Add(2 * time.Hour)
    for time.Now().Before(deadline) {
        lag, err := c.replicator.Lag(backup, primary)
        if err != nil { return err }
        if lag < 1*time.Second {
            if c.checkSustained(backup, primary, 5*time.Minute, 1*time.Second) {
                break
            }
        }
        time.Sleep(10 * time.Second)
    }
    if time.Now().After(deadline) {
        return fmt.Errorf("replication never caught up")
    }

    // 第三步：灰度切流
    for _, pct := range []int{1, 5, 20, 50, 100} {
        if err := c.lb.SetTraffic(primary, pct); err != nil { return err }
        time.Sleep(5 * time.Minute)
        if c.errorRate(primary) > 0.01 {
            c.lb.SetTraffic(primary, 0)
            return fmt.Errorf("error rate too high at %d%%", pct)
        }
    }
    return nil
}
```

#### 验证

故障演练时把"回切"也演练一遍。**只演练"切走"不演练"切回"是最常见的演练遗漏**。

> **口诀**：切走容易切回难。**回切前必须等反向同步追平 + 灰度回流**，否则切回 = 数据回退。

---

### 案例 ② 单元化路由错误 —— "用户在新单元找不到自己的数据"

#### 现象

某天凌晨切流后，部分用户登录后发现"账户余额变成 0、订单全没了"。客服爆炸，紧急回滚。

#### 根因

切流脚本的 bug：把 shard 33 的路由从 Cell-A 改到了 Cell-B，但 **Cell-B 的 DB 里根本没有 shard 33 的用户数据**（数据应该提前同步过去）。用户请求被路由到 Cell-B 后，Cell-B 查不到，返回"账户初始状态"（=0）。

更深一层：单元化迁移的标准流程是 **"先复制数据，再切路由"**，但这次脚本的两步顺序反了。

```
错误流程：
   步骤1：路由切换 (shard 33 → Cell-B)   ←── ★ 此时 Cell-B 没数据
   步骤2：数据复制 (Cell-A → Cell-B)
   步骤3：等同步追平

正确流程：
   步骤1：开启 Cell-A → Cell-B 单向复制 (shard 33)
   步骤2：等待复制追平
   步骤3：禁写 Cell-A 上 shard 33（双写状态结束）
   步骤4：等待最后一批写入复制完成
   步骤5：切换路由 (shard 33 → Cell-B)
   步骤6：禁读 Cell-A 上 shard 33（确保用户都去 Cell-B）
```

#### 修复

**操作必须脚本化 + 强制顺序检查**：

```python
class ShardMigration:
    def __init__(self, shard, src, dst):
        self.shard = shard
        self.src = src
        self.dst = dst
        self.state = "INIT"

    def migrate(self):
        try:
            self._step("INIT -> REPLICATING", self._enable_replication)
            self._step("REPLICATING -> CAUGHT_UP", self._wait_caught_up)
            self._step("CAUGHT_UP -> WRITE_FENCED", self._fence_write_on_src)
            self._step("WRITE_FENCED -> FINAL_SYNCED", self._wait_final_sync)
            self._step("FINAL_SYNCED -> ROUTED", self._switch_route)
            self._step("ROUTED -> READ_FENCED", self._fence_read_on_src)
        except Exception as e:
            self._rollback()
            raise

    def _step(self, transition, fn):
        log.info(f"shard {self.shard}: {transition}")
        fn()
        self.state = transition.split(" -> ")[1]
```

每一步**必须**在前一步成功后才能执行。任何一步失败 → 自动回滚。

**业务侧兜底**：用户查不到自己数据时，**不要返回"初始值"，要返回 5xx 让上游重试**。返回 0 余额会让用户以为钱丢了 + 操作员以为系统正常 = 灾难放大。

```java
// 错的
public Account getAccount(long uid) {
    Account a = repo.findByUid(uid);
    return a != null ? a : new Account(uid, BigDecimal.ZERO);   // ❌ 找不到给 0
}

// 对的
public Account getAccount(long uid) {
    Account a = repo.findByUid(uid);
    if (a == null) {
        // 检查这个 uid 是不是真的应该在本单元
        String expectedUnit = router.route(String.valueOf(uid));
        if (!expectedUnit.equals(currentUnit)) {
            throw new WrongUnitException(uid, expectedUnit);   // ★ 让上游路由重试
        }
        // 真的不存在 → 返回 404
        throw new AccountNotFoundException(uid);
    }
    return a;
}
```

#### 验证

灰度迁移：**先迁 1 个测试 UID**（自己的账号），人肉验证一遍所有读写操作。再迁 10 个、100 个、1%、10%、全量。

> **口诀**：单元化迁移的铁律是**"先数据，后路由"**。任何一步顺序错 = 数据"消失"。业务侧必须有"该用户是否在本单元"的校验。

---

### 案例 ③ 数据库双写冲突 —— "UID 冲突 / 库存重复扣减"

#### 现象

两个机房都开通了用户注册功能（设计上不应该，但运营要求"两边都能注册"），结果几小时内出现了几十个用户 UID 重复 —— 同一个 UID 对应两个不同的注册手机号，登录时随机看到一份数据。

#### 根因

**多机房用本地自增 ID = 必然冲突**。即使两边都用 `auto_increment`，且配置了不同的 `auto_increment_increment` / `auto_increment_offset`（杭州生成奇数 ID，上海生成偶数 ID），看起来不冲突，但实际上：

- 配置漂移：某个 DB 实例没改对配置
- 主备切换后配置没生效
- 历史 ID 段就有重叠

库存扣减更经典：

```sql
-- 杭州机房                          -- 上海机房
SELECT stock FROM goods WHERE id=1;  SELECT stock FROM goods WHERE id=1;
-- 读到 stock=1                      -- 读到 stock=1
UPDATE goods SET stock=stock-1       UPDATE goods SET stock=stock-1
WHERE id=1 AND stock>0;              WHERE id=1 AND stock>0;
-- 成功                              -- 成功
                                    -- 同步到杭州，stock 变 -1（超卖）
```

#### 修复

**ID 生成的正确姿势**（按推荐度排序）：

| 方案 | 优点 | 缺点 |
| --- | --- | --- |
| **Snowflake**（机房 + 机器 + 时间 + 序列） | 全局唯一、有序 | 时钟回拨问题 |
| **UUIDv7**（时间有序的 UUID） | 全局唯一、无中心 | 比自增长一倍 |
| **号段模式**（类美团 Leaf） | 性能高、可恢复 | 重启丢号段 |
| **数据库自增 + 多机房 offset** | 简单 | 配置易漂移、扩缩容难 |
| **业务主键**（手机号、订单号编码） | 业务可读 | 跨机房需协调 |

Snowflake 模板（含机房号）：

```java
public class Snowflake {
    private static final long EPOCH = 1577836800000L;  // 2020-01-01
    private static final int DC_BITS = 5;          // 32 个机房
    private static final int WORKER_BITS = 5;      // 每机房 32 个 worker
    private static final int SEQ_BITS = 12;        // 每毫秒 4096

    private final long dcId;       // 机房 ID
    private final long workerId;
    private long lastTimestamp = -1L;
    private long sequence = 0L;

    public synchronized long next() {
        long now = System.currentTimeMillis();
        if (now < lastTimestamp) {
            // ★ 时钟回拨：不能继续生成，否则 ID 重复
            throw new IllegalStateException("clock moved backwards");
        }
        if (now == lastTimestamp) {
            sequence = (sequence + 1) & ((1 << SEQ_BITS) - 1);
            if (sequence == 0) {
                now = waitNextMillis(lastTimestamp);
            }
        } else {
            sequence = 0;
        }
        lastTimestamp = now;
        return ((now - EPOCH) << (DC_BITS + WORKER_BITS + SEQ_BITS))
             | (dcId << (WORKER_BITS + SEQ_BITS))
             | (workerId << SEQ_BITS)
             | sequence;
    }
}
```

**库存扣减的正确做法**：单元化路由 —— 商品按"商品 ID hash"路由到固定单元，**写流量永远只在一个单元**。

```
全局商品 ID 100 → 永远在 Cell-A 写
全局商品 ID 200 → 永远在 Cell-B 写
任意机房读：本地异步同步过来的副本（最终一致）
```

或者更简单：**库存扣减的请求统一路由到一个"库存中心"**，不要异地多活。库存这种强一致性 + 写密集的场景，单点写比双活靠谱得多。

#### 验证

线下用 sysbench 跑 1 万次并发注册（模拟两个机房），统计 UID 是否唯一。

```bash
# 跑一段时间后
SELECT uid, COUNT(*) FROM user GROUP BY uid HAVING COUNT(*) > 1;
-- 必须返回空
```

> **口诀**：异地多活的"双写"必须是**按维度切分的"伪双写"**（每条数据归属一个单元）。**真正的双向写一行数据 = 数据不一致 + 库存超卖**。

---

### 案例 ④ DNS TTL 太长 —— "故障切换后用户还在访问宕机机房"

#### 现象

机房故障，DNS 切到备机房，但运维监控发现 30 分钟后还有 5% 的用户在访问宕机的机房（直接看 LB 日志）。

#### 根因

DNS 记录的 TTL 设了 600 秒（10 分钟），但运营商 LocalDNS 经常**忽略 TTL，按自己策略缓存**（联通某些地区可达 24 小时）。即使你设 60 秒，LocalDNS 可能缓存几小时。

更糟：**Java 的 InetAddress 默认 JVM 级缓存 DNS 结果**：

```java
// 默认配置（OpenJDK）：
// networkaddress.cache.ttl = -1（永久缓存）
// networkaddress.cache.negative.ttl = 10
```

意思是 JVM 启动时解析了一次 DNS，**之后再也不解析**。除非你显式改：

```java
// 启动参数
-Dsun.net.inetaddr.ttl=60
-Dsun.net.inetaddr.negative.ttl=10

// 或者代码里
java.security.Security.setProperty("networkaddress.cache.ttl", "60");
```

#### 修复

**多层防御**：

1. **DNS TTL 设短**：60 秒。注意不是越短越好（TTL 太短会增加 DNS 服务器负载）。
2. **客户端解析配置**：明确设置 JVM 的 DNS 缓存 TTL。
3. **不依赖 DNS 切换故障流量**：DNS 是慢路径。快路径用 Anycast 或 GSLB。
4. **故障机房保留"逃生路径"**：故障机房的 LB 仍然运行（即使后端挂了），但 LB 配置成"自动 302 跳转到备机房入口"。这样老 DNS 缓存的用户能立即被引导走。

```nginx
# 故障机房的 LB 配置（应急切换）
server {
    listen 443 ssl;
    server_name api.example.com;

    location / {
        # 所有请求 302 到备机房
        return 302 https://backup.example.com$request_uri;
    }
}
```

5. **App 客户端短轮询**：移动端 App 可以定期主动重连 / 重新解析 DNS，不等系统级缓存。

#### 验证

混沌演练：手动把一个机房的所有后端关掉，但 LB 保持运行。观察"302 跳转"的请求量曲线 —— 应该在 1 分钟内升到 80%+，5 分钟内升到 99%+。

```bash
# 在故障机房 LB 上
tail -f /var/log/nginx/access.log | awk '$9 == 302 {count++} END {print count}'

# 同时观察备机房接到的"302 跳转过来"的请求量
```

> **口诀**：DNS 是慢路径，故障切换不能只靠 DNS。**故障机房 LB 必须保持运行 + 自动跳转**，配合短 TTL 才有效。

---

### 案例 ⑤ 健康检查失效 —— "实例已挂但 LB 仍转发请求"

#### 现象

某服务实例的进程崩了（OOM），但 LB 一分钟内还在向它转发请求（用户看到 5xx）。监控系统也以为它是健康的。

#### 根因

健康检查的两种典型错配：

**错配 1：健康检查端口和服务端口不一样**

```yaml
# K8s 配置
livenessProbe:
  httpGet:
    path: /health
    port: 9090       # ★ 这是 metrics 端口（Prometheus 抓的），不是业务端口
  periodSeconds: 30
```

业务进程崩了，但 metrics 端口（在另一个进程里）还活着 → liveness probe 始终成功 → K8s 不重启 Pod。LB 看 K8s 的 endpoint，也以为它健康。

**错配 2：健康检查太肤浅（只看进程是不是在）**

```java
// 错：永远返回 200
@GetMapping("/health")
public String health() { return "OK"; }
```

进程在跑、但 DB 连接全断了、Redis 连不上 —— 业务实际不可用。健康检查永远 200，LB 一直转发。

**错配 3：检查频率 / 阈值不合理**

```yaml
livenessProbe:
  periodSeconds: 60          # 60 秒一次
  failureThreshold: 5        # 连续 5 次失败才算挂
  # 实际感知故障时间 = 60s × 5 = 5 分钟
```

5 分钟才感知到故障，期间所有用户都看到 5xx。

#### 修复

**深度健康检查（Deep Health Check）**：

```java
@GetMapping("/health/deep")
public ResponseEntity<Map<String, Object>> deepHealth() {
    Map<String, Object> details = new HashMap<>();
    boolean healthy = true;

    // ① DB 连接
    try (Connection c = dataSource.getConnection()) {
        c.createStatement().execute("SELECT 1");
        details.put("db", "ok");
    } catch (Exception e) {
        details.put("db", "fail: " + e.getMessage());
        healthy = false;
    }

    // ② Redis
    try {
        redis.ping();
        details.put("redis", "ok");
    } catch (Exception e) {
        details.put("redis", "fail");
        healthy = false;
    }

    // ③ 关键下游
    try {
        downstream.healthCheck();
        details.put("downstream", "ok");
    } catch (Exception e) {
        details.put("downstream", "degraded");   // 下游挂不一定让自己也挂
    }

    return healthy
        ? ResponseEntity.ok(details)
        : ResponseEntity.status(503).body(details);
}
```

**两层探针配合（K8s）**：

```yaml
livenessProbe:        # 进程级：进程死了重启
  httpGet:
    path: /health/live    # 浅：只看进程在不在
    port: 8080
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:       # 业务级：业务能不能服务
  httpGet:
    path: /health/deep    # 深：DB / Redis / 下游
    port: 8080
  periodSeconds: 5
  failureThreshold: 2     # ★ 更敏感：10 秒就摘流量
```

`readiness` 失败只摘流量、不重启；`liveness` 失败才重启。这避免了"瞬时抖动 → 重启 → 又抖动"的雪崩。

**LB 层独立健康检查**：

```
LB → /health/lb-check    （独立端点，专给 LB 看）
     ↓
     业务自己控制：维护期可以主动返回 503，让 LB 自动摘流量
```

```java
@GetMapping("/health/lb-check")
public ResponseEntity<String> lbCheck() {
    if (maintenanceMode.get()) {
        return ResponseEntity.status(503).body("maintenance");
    }
    return ResponseEntity.ok("ok");
}
```

#### 验证

混沌演练：用 `tc` / `iptables` 故意切断业务实例的 DB 连接，观察 readiness probe 是否在 10 秒内失败、LB 是否在 30 秒内摘除流量。

```bash
# 故意阻断 DB 连接
sudo iptables -A OUTPUT -p tcp --dport 3306 -j DROP

# 观察 LB 后端列表
curl http://lb-admin/backends/myservice
# 实例应在 30 秒内消失

# 恢复
sudo iptables -D OUTPUT -p tcp --dport 3306 -j DROP
```

> **口诀**：健康检查必须**深度检查依赖** + **liveness 浅 / readiness 深**两层配合。**返回常量 200 的 /health = 没做健康检查**。

---

### 案例 ⑥ 网络分区下"双主" —— "两个机房都以为对方挂了，并发写入"

#### 现象

杭州和上海机房之间的专线断了 30 秒。两边的 ZooKeeper 集群（5 节点 = 杭州 3 + 上海 2）各自选举出新 Leader（杭州的 3 节点形成多数派；上海 2 节点不能选主，但应用以为可以）。结果杭州正常写入；上海的某些应用因为缓存了旧的 Leader 信息，**也在写**。30 秒后专线恢复，两边数据冲突。

#### 根因

这是经典的**脑裂（Split-Brain）**。任何依赖"多数派选主"的系统在网络分区时都可能裸奔。

具体到 ZooKeeper：5 节点 ZK 集群，3 节点这一边能选主、能写；2 节点这一边不能选主、不能写（这是 ZK 的设计保证）。

但**应用层不一定知道 ZK 已经"不能写"**。常见错误：

```java
// 应用伪代码
String leader = zkClient.getData("/myservice/leader");
boolean amILeader = leader.equals(myAddr);
if (amILeader) {
    // 我以为我是 Leader，开始写主库
    db.write(...);
}
```

应用启动时读了一次 `/myservice/leader`，缓存了。网络分区后即使 ZK session 失效，应用本地缓存还在 → 还以为自己是 Leader → 还在写。

#### 修复

**铁律**：**不要在应用层缓存"我是不是 Leader"的判断**。每次写之前都要确认。

```java
// 正确做法：每次写都检查 ZK Session 状态
public void write(Data d) {
    if (!zkClient.getZk().getState().isConnected()) {
        throw new NotLeaderException("zk session lost");
    }
    if (!zkClient.exists("/myservice/leader/" + myAddr, true)) {
        throw new NotLeaderException("leadership ephemeral node lost");
    }
    db.write(d);
}
```

更彻底：用**强一致性的协调服务做 fencing**。每次拿到锁时拿一个**单调递增的 fencing token**：

```java
// 1. 抢锁，拿 token
long token = lockService.acquire("/myservice/leader");

// 2. 写底层存储时带上 token
storage.write(data, token);

// 3. 存储侧拒绝小于"已见过的最大 token"的写
//    这样即使老 Leader 复活，它的 token 是旧的，写会被拒
```

这是 Google Chubby / etcd 论文里"防止陈旧 Leader 写入"的标准模式。

**网络分区检测自动降级**：

```java
public class PartitionDetector {
    public PartitionState detect() {
        // 检查到对端机房的连通性
        boolean peerReachable = ping(peerDcGateway);
        boolean externalReachable = ping(externalCheckPoint);

        if (!peerReachable && externalReachable) {
            return PartitionState.PARTITIONED;   // 只是和对端断了
        }
        if (!peerReachable && !externalReachable) {
            return PartitionState.LOCAL_ISOLATED; // 自己被孤立了
        }
        return PartitionState.NORMAL;
    }
}

// 业务层：检测到分区时，少数派机房自动降级为只读
if (detector.detect() == PartitionState.LOCAL_ISOLATED && isMinority()) {
    serviceMode = ReadOnly;
}
```

#### 验证

用 [chaos-mesh](https://chaos-mesh.org/) 模拟网络分区：

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: partition-hz-sh
spec:
  action: partition
  mode: all
  selector:
    namespaces: [hangzhou]
  direction: to
  target:
    selector:
      namespaces: [shanghai]
    mode: all
  duration: 60s
```

跑完观察：两边是否都能写？数据是否冲突？分区恢复后是否能正确合并？

> **口诀**：脑裂的根因是"应用层没用 fencing token"。**只信 ZK 的"我是 Leader"会出事，必须每次写前重新确认 + 带 token**。

---

### 案例 ⑦ 容量预留不足 —— "故障切换后单机房承担全流量被打挂"

#### 现象

机房 A 挂了，流量自动切到机房 B，5 分钟后机房 B 也挂了 —— B 单机房扛不住全量流量。整个服务雪崩。

#### 根因

双活架构通常默认"两个机房各 50%"。但**真正"高可用"的双活要求每个机房能扛 100% 全量**。这意味着：

- CPU / 内存预留：每机房日常使用率不能超过 50%（留 50% 故障余量）
- DB 容量：每机房 DB 副本要能扛全量读写
- 缓存容量：单机房缓存大小不变（缓存穿透流量翻倍）
- 下游依赖容量：每机房依赖的下游服务也要能扛全量

但生产里常见的反模式：

```
日常态：A 50% 流量 + B 50% 流量
        A CPU 用 60%，B CPU 用 60%（成本优化嘛）
        ↓
        A 挂了 → B 接 100%
        B CPU 飙到 120% → 部分请求超时
        超时引发雪崩 → B 也挂
```

#### 修复

**容量公式**：

```
每机房应预留容量 = 全量流量 × 1.2（峰值容差） × 1.2（突发流量容差）
```

意味着每机房日常**最多用 35%**（如果是双活）。**这是异地多活的真实成本** —— 你要按平时 3 倍的容量买机器。

成本控制方案：

**方案 A：3 机房，每机房日常 50% 利用率**

```
正常态：A、B、C 各 33.3% 流量，CPU 50%
故障态：任一机房挂 → 其他两个各接 50%，CPU 75%（仍可承受）
```

3 机房成本上比 2 机房高 50%，但单机房利用率可以高一倍 —— 总成本只多 25% 左右，且容错能力更强。

**方案 B：2 机房 + 弹性扩容**

```
正常态：A、B 各 50% 流量，CPU 60%
故障态：A 挂 → B 立即弹性扩容（云上 Auto Scaling）→ 5 分钟内扩出双倍容量
```

要点：**Auto Scaling 必须能在故障窗口内完成**。如果你的扩容要 30 分钟（要装系统、要拉镜像、要预热缓存），等不及。

**方案 C：核心 + 非核心降级**

```
故障态：B 接 100% 流量，CPU 飙升
       → 自动降级非核心功能（评论、推荐、个性化）
       → 保住核心功能（下单、支付、登录）
```

这是阿里"全链路压测 + 分级降级"的核心思路。需要业务侧从一开始就把功能分级。

#### 容量验证（必做）

**单机房压测**：把流量全打到一个机房，看它能不能扛。

```bash
# 阿里"全链路压测"的简化版：导入流量到单机房
# 用真实生产流量录制 + 回放
sw-record --duration 1h --output prod-traffic.pb
sw-replay --target hangzhou-only --rate 100% --input prod-traffic.pb

# 观察：是否雪崩、降级是否生效、扩容是否触发
```

阿里每年大促前做 N 轮全链路压测，每个机房单独承担全量。**没做过就上线 = 故障切换 = 雪崩**。

> **口诀**：双活的成本是 **每机房按全量预留容量**。预留不够 = 故障切换 = 雪崩。**唯一的验证方式是单机房全量压测**。

---

### 案例 ⑧ 缓存与 DB 跨机房不一致 —— "用户在 A 机房写、B 机房读"

#### 现象

切流过程中，部分用户的请求在 A 机房写了 DB（且 A 机房本地 Redis 缓存也更新了），但接下来的请求被路由到 B 机房，B 机房的 Redis 还是老数据。用户看到"我刚改的没生效"。

#### 根因

缓存 + 多活的经典问题：

```
A 机房：
  1. UPDATE db SET name='Alice' WHERE id=1
  2. DEL local_redis user:1                ← A 的 Redis

DB 同步：A → B（异步，假设 lag 1s）

B 机房：
  Redis 还是 user:1='Bob'（没人删它）
  GET 请求来了：先查 B 的 Redis → 命中老数据 → 返回 'Bob'
```

`Cache-Aside` 模式（先写 DB 再删 cache）在单机房是对的，**多机房就漏了**：B 机房的缓存没被通知。

#### 修复

**方案 A：跨机房 cache 失效广播**

```
A 机房 UPDATE 后：
  1. DEL A 机房 local Redis
  2. 发 MQ 消息（带上 cache key）
B 机房消费 MQ：
  3. DEL B 机房 local Redis
```

```java
public void update(User u) {
    db.update(u);
    localRedis.del("user:" + u.id);
    // 跨机房广播
    mq.publish("cache-invalidation", new InvalidationMsg("user", u.id));
}

// B 机房消费者
@MqConsumer("cache-invalidation")
public void onInvalidation(InvalidationMsg msg) {
    localRedis.del(msg.type + ":" + msg.id);
}
```

要点：MQ 必须**跨机房可靠传递**（有重试、不丢）。RocketMQ / Kafka 的跨机房同步通常用 Connector 或 Replicator。

**方案 B：缓存查询时带"最少新鲜度"**

```java
public User get(long uid) {
    User u = localRedis.get("user:" + uid);
    if (u != null && u.cachedAt > recentWriteTime(uid)) {
        return u;     // 缓存比已知最近一次写新
    }
    u = db.get(uid);
    localRedis.setex("user:" + uid, u, 60);
    return u;
}

// 通过 Bloom Filter 快速判断"这个 uid 最近有没有写过"
private long recentWriteTime(long uid) {
    if (recentWriteBloom.mightContain(uid)) {
        return Long.MAX_VALUE;   // 可能写过，强制走 DB
    }
    return 0;
}
```

**方案 C：跨机房读 DB 主库**

切流瞬间的查询直接读 DB 主库（绕过缓存），代价是 DB 压力大。仅适合短窗口。

**方案 D：缓存归属和 DB 一样按 UID 分 Cell**

每个 UID 的缓存只在它"应该所属的 Cell"。请求路由错的话，先按 UID 路由到对的 Cell，再查那个 Cell 的缓存。这其实就是单元化的延伸。

#### 验证

```bash
# 1. 在 A 机房 UPDATE 一条数据
curl -X POST -d '{"id":1, "name":"Alice"}' a.example.com/user

# 2. 立即在 B 机房查这条数据（带强制路由 header）
curl -H "X-Force-Region: B" b.example.com/user/1

# 3. 必须读到新值 'Alice'
```

跑这个测试 1000 次，统计读到老值的比例。健康的多活系统应该 < 0.01%。

> **口诀**：多活下缓存 = 跨机房失效广播 + 写少的强制读 DB。**单机房 cache-aside 套到多活 = 数据看起来不一致**。

---

### 案例 ⑨ 异步任务跨机房执行 —— "定时任务被多个机房同时触发"

#### 现象

每天 00:00 跑的"日终账单生成"任务被两个机房同时触发，结果生成了两份账单 + 给用户发了两条短信。客服电话被打爆。

#### 根因

定时任务（Quartz / xxl-job / Spring `@Scheduled`）部署在每个机房都跑。开发以为"调度器有自己的分布式锁"，实际上：

- `@Scheduled` 没有任何分布式协调
- Quartz 的"集群模式"只能在共享 DB 的实例间协调，**跨机房如果 DB 是双向同步的，两边都能拿锁**
- xxl-job 有调度中心，但调度中心本身要是没做高可用，故障切换时也容易"双触发"

```
A 机房调度中心 ──触发──▶ A 机房 worker ──执行
B 机房调度中心 ──触发──▶ B 机房 worker ──执行
                  ↑
                  两个调度中心都"健康"，都触发了同一个任务
```

#### 修复

**方案 A：定时任务永远只在主单元跑**

```java
@Scheduled(cron = "0 0 0 * * *")
public void dailyBilling() {
    if (!unitInfo.isPrimaryUnit()) {
        return;   // 非主单元直接跳过
    }
    doBilling();
}
```

`isPrimaryUnit()` 由配置中心动态控制。故障切换时把"主单元"标记切换。

**方案 B：基于幂等 ID 的执行去重**

每个任务实例都尝试执行，但执行前先抢一个**全局唯一锁**（用 Redis、ZK、或 DB 的 unique key）：

```java
public void dailyBilling() {
    String lockKey = "task:dailyBilling:" + LocalDate.now();
    boolean acquired = redis.setnx(lockKey, instanceId(), 24, TimeUnit.HOURS);
    if (!acquired) {
        log.info("task already running on another instance");
        return;
    }
    try {
        doBilling();
    } finally {
        redis.del(lockKey);
    }
}
```

注意：Redis 必须是**全局共享**（跨机房的 Redis 集群），不能是机房本地 Redis。否则两边都能 setnx 成功。

**方案 C：业务层幂等保证**

最稳的方式：任务本身做成幂等的。同一个账单日期、同一个用户，无论跑多少次，结果一致。

```java
public void generateBill(long uid, LocalDate date) {
    // 用 (uid, date) 做唯一键
    Bill existing = billRepo.findByUidAndDate(uid, date);
    if (existing != null) {
        return;   // 已经生成过
    }
    Bill bill = compute(uid, date);
    try {
        billRepo.save(bill);          // unique constraint on (uid, date)
    } catch (DuplicateKeyException e) {
        // 别的实例同时写了，没事
    }
    sendSms(uid, bill);   // ★ 这里也要幂等：用 bill.id 作为 SMS 的 dedup key
}
```

短信网关的 dedup：

```java
public void sendSms(String dedupKey, String content) {
    // 查 dedup_key 是否发过（24 小时内）
    if (redis.exists("sms-sent:" + dedupKey)) return;
    smsGateway.send(content);
    redis.setex("sms-sent:" + dedupKey, "1", 24, TimeUnit.HOURS);
}
```

#### 验证

模拟两个机房同时触发：

```bash
# 同时在两个机房触发
curl http://a.example.com/admin/job/dailyBilling/trigger &
curl http://b.example.com/admin/job/dailyBilling/trigger &
wait

# 检查账单生成数 = 用户数 × 1（不是 × 2）
SELECT COUNT(*) FROM bill WHERE bill_date = CURDATE();

# 检查短信发送数 = 用户数 × 1
SELECT COUNT(*) FROM sms_log WHERE sms_date = CURDATE();
```

> **口诀**：多机房定时任务 = **三道闸门**（主单元判断 + 全局锁 + 业务幂等）。**任意一道漏了 = 双触发**。短信、扣款、退款这种有副作用的任务必须三道全用。

---

## 5. 上线检查清单

打印贴墙 / 加进架构评审 checklist。

### 部署层

- [ ] 部署形态明确：单机 / 主备 / 同城双活 / 异地多活，且和业务 SLA 目标匹配
- [ ] 故障域清晰：能讲出"任何一个故障域挂掉，业务表现是什么"
- [ ] 容量按"全量流量 × 1.2 × 1.2"预留，每机房日常利用率 < 50%（双活）/ < 33%（三活）
- [ ] 所有核心依赖（DB / 缓存 / MQ / 注册中心 / 配置中心）都有跨机房高可用
- [ ] 跨机房专线带宽 ≥ 峰值流量 × 1.5

### 流量切换层

- [ ] DNS TTL ≤ 60 秒，且所有客户端（含 JVM）配了短的 DNS 缓存
- [ ] 故障机房 LB 保留运行 + 自动跳转到备机房
- [ ] 健康检查分两层：liveness 浅（看进程） + readiness 深（看 DB / Redis / 下游）
- [ ] LB 摘除后端的反应时间 ≤ 30 秒
- [ ] 自动切换前有"二次确认"避免误判（30 秒内连续 N 次失败）
- [ ] 切流脚本有完整状态机（INIT → REPLICATING → CAUGHT_UP → ROUTED → DONE），任意一步失败自动回滚

### 数据一致性层

- [ ] 所有 ID 生成跨机房唯一（Snowflake / UUIDv7 / 号段）
- [ ] 异地多活的写流量按维度切分（UID / 商品 ID），**没有真正的"双写一行"**
- [ ] 切流时按"先复制数据 → 等追平 → 禁写源 → 等最后同步 → 切路由 → 禁读源" 6 步执行
- [ ] 业务侧有"该用户/数据是否在本单元"的校验，不存在时返回 5xx 而不是 0/null
- [ ] 多活下缓存有跨机房失效广播，或缓存按单元切分
- [ ] 强一致性数据用 fencing token，不依赖应用层缓存的 Leader 状态

### 任务与异步层

- [ ] 定时任务三道闸门：主单元判断 + 全局锁 + 业务幂等
- [ ] 短信 / 邮件 / 扣款等有副作用的操作有 dedup key
- [ ] MQ 跨机房同步（双活）有可靠性保证（不丢、可重试）
- [ ] 异步任务的执行单元和数据归属单元一致

### 演练层

- [ ] 单机房全量压测每季度至少 1 次
- [ ] 故障演练每月至少 1 次（按故障类型轮换：单实例、单机房、网络分区、依赖挂）
- [ ] 切流演练（含切走 + 切回）每季度 1 次
- [ ] 演练记录有 RTO / RPO 实测数据，且和目标 SLA 对得上

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### 全球分布式数据库

- **Google Spanner**：用 TrueTime（GPS + 原子钟）实现全球外部一致性，论文是高可用领域必读
- **TiDB**：开源 Spanner-style，国内最大的"准 NewSQL"实践，可以读它的 Placement Rules 文档
- **CockroachDB**：另一个 Spanner-style 实现，比 TiDB 更强调多 Region 部署
- **Aurora Global Database**：物理复制（不走逻辑日志），跨 Region 复制 lag < 1 秒，但只能"一主多读"

### 网络层

- **Anycast 的 BGP 部署**：自有 AS、跨多个 IDC 宣告同一 IP，路由器 BGP 选最近。CloudFlare、Google DNS 都这么做
- **Geo-DNS 的真实表现**：用户 IP 到机房的映射不一定准（VPN、运营商 NAT），需要 GSLB 兜底
- **QUIC 协议**：基于 UDP，能解决 TCP 在网络切换时连接中断的问题，对移动端高可用关键

### 灾难恢复（DR）

- **DR 等级（Tier 0~6）**：从"无备份"到"零数据丢失 + 零停机"。金融业有强制等级要求
- **冷备 / 温备 / 热备的成本曲线**：温备性价比通常最高
- **数据备份的 3-2-1 原则**：3 份副本、2 种介质、1 份异地

### 混沌工程

- **Netflix Chaos Monkey**：祖师爷，每天在生产随机杀实例
- **chaos-mesh**：CNCF 项目，K8s 上的故障注入瑞士军刀
- **Litmus**：另一个 K8s 混沌工具，社区更活跃
- **Chaos Engineering Principles**：必读的方法论文档（principlesofchaos.org）
- **GameDay**：一种"全员参与的故障演练"模式，AWS 和 Google 都在内部跑

### SRE 工作簿

- **错误预算（Error Budget）**：把可用性当资源用，剩多少决定是否能上新功能
- **SLO / SLA / SLI 的区分**：SLO 是内部目标，SLA 是对外承诺，SLI 是测量指标
- **演练手册（Runbook）**：每个高可用场景都应该有一份可一键执行的脚本
- **事后复盘（Postmortem）模板**：Google 的 Blameless Postmortem 是行业标杆

### 业务连续性管理（BCM）

- **BCP / DRP 的关系**：BCP（业务连续性计划）覆盖更广，DRP（灾难恢复计划）只管 IT
- **ISO 22301**：业务连续性的国际标准，金融和能源行业常考
- **跨地域合规**：欧盟 GDPR / 中国数据安全法对"数据出境"的限制，影响异地多活的物理选址

### 高可用的成本曲线

- **钱**：每加一个 9 翻 5~10 倍。算清楚 SLA 罚金 vs 改造成本的临界点
- **人**：SRE / DBA / 网络团队的总人头。一支 50 人的 SRE 团队是大厂标配
- **研发效率**：高可用约束（必须经过演练才能上线、必须 dual-write、必须 fencing）会让特性开发变慢 2~3 倍
- **替代方案**：很多业务没必要做异地多活，做好"单机房高可用 + 异地冷备"性价比更高

---

## 7. 引用

- Werner Vogels, *Eventually Consistent* —— 异步复制和最终一致性的奠基论文（CACM 2009）
- Pat Helland, *Life beyond Distributed Transactions* —— 分布式事务的"不可能三角"
- Leslie Lamport, *Paxos Made Simple* —— Paxos 论文
- Diego Ongaro, *In Search of an Understandable Consensus Algorithm* —— Raft 论文
- Google, *Spanner: Google's Globally Distributed Database* —— TrueTime + 全球外部一致性
- Netflix Tech Blog, *The Netflix Simian Army* —— Chaos Monkey 起源
- 阿里云, *单元化架构实践* —— 国内单元化最完整的公开材料
- 蚂蚁金服, *OceanBase 三地五中心架构* —— 国内异地多活金融级实践
- Google SRE Book —— [sre.google/sre-book](https://sre.google/sre-book)
- *Chaos Engineering: System Resiliency in Practice* —— O'Reilly 的混沌工程权威书

---

> 配套：[系统设计主题导览](/article/system-design/) ｜ [分布式系统主题导览](/article/distributed/) ｜ [可观测性](/article/observability/)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
