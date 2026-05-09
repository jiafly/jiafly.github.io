# Serverless 与 Knative / KEDA —— K8s 原生函数计算

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，去 [K8s 实战](/article/k8s/k8s-essentials) 那一节够了。
> 这里要回答的，是**当你打算把"按量付费、扩缩到零、事件驱动"这一套塞进自家 K8s 集群**时，你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. FaaS、BaaS、Serverless 三个词经常混用 —— 它们的精确边界在哪？AWS Lambda 属于哪一类？Firebase Auth 呢？Knative Service 呢？
2. Knative 分 Serving 和 Eventing 两块，前者解决什么问题、后者解决什么问题？两者**能不能各自单独装**？
3. 一个 Java Spring Boot 容器的"冷启动"在裸 K8s 上是几秒？同样的代码用 GraalVM Native Image 编译后又是几毫秒？两者差几个数量级？
4. KEDA 和 K8s 自带的 HPA 都能"按指标扩缩"，**它们的关键差别**是什么？为什么 KEDA 能扩到 0 而 HPA 不能？
5. Knative 的 Activator 在请求路径里到底干什么？为什么"扩到 0 之后第一个请求"必须经过它？
6. "函数即服务"和"容器即服务"的边界是什么？把一个 Spring Boot 应用包成 Knative Service，它算函数还是容器？
7. AWS Lambda、Cloudflare Workers、Knative，**冷启动一个数量级差异在哪**？为什么 Cloudflare Workers 能做到 < 5ms？
8. 同样跑 1000 QPS 的事件处理，KEDA 触发的 Pod 扩缩 vs HPA 基于 CPU 的扩缩，**谁会更准**？谁更稳？

如果有半数答不上，往下看。

---

## 1. 底层模型

绝大多数人对 Serverless 的认知止于"AWS Lambda 那种东西"。但当你真的要在自家 K8s 上跑 Knative + KEDA，这层抽象解释不了上面 8 个问题。要看懂它们，必须把"Serverless"拆成几个独立轴。

### 1.1 Serverless 的核心承诺：三件事，缺一不可

Serverless（无服务器）这个词被滥用到失去意义。它的最小可识别承诺只有三条：

1. **按需启动**：流量来才有进程，没流量就没进程
2. **自动扩缩**：扩缩动作由平台自动完成，**用户写 0 行扩缩代码**
3. **按量付费**：只为"实际占用的算力 × 时间"付钱，不为"预留"付钱

第三条对用户最有吸引力，但**对自建 K8s 集群没意义** —— 物理机已经买了，不存在"按量付费"。所以自建 Serverless 平台（Knative）的真实价值是前两条，加一个隐藏价值：**资源利用率**。

| 部署模式 | 闲时资源 | 流量来时延 | 计费 |
| --- | --- | --- | --- |
| 永久 Pod（Deployment） | 100% 占用 | 0ms | 不变 |
| HPA（最少 1 副本） | 1 个 Pod 的占用 | 扩容耗时 | 不变 |
| **Knative Scale-to-Zero** | **0 占用** | 冷启动耗时 | 节省闲时 |

第三行就是 Knative 的核心卖点。

### 1.2 FaaS / CaaS / BaaS：一张图分清

```
                  Serverless（按需 + 自动扩缩 + 按量付费）
                 ┌─────────────────┬──────────────────┐
                 │                 │                  │
              FaaS              CaaS              BaaS
        Function as a    Container as a     Backend as a
            Service          Service           Service
        ─────────────    ─────────────     ─────────────
        AWS Lambda       AWS Fargate        Firebase Auth
        Cloudflare       Google Cloud Run   AWS Cognito
          Workers        Knative Service    Supabase DB
        Vercel Func      Azure Container    PlanetScale
                          Apps              MongoDB Atlas
        ─────────────    ─────────────     ─────────────
        交付：函数代码    交付：容器镜像     交付：API
        运行：平台抽象    运行：你定 image  运行：托管服务
        冷启动：< 100ms   冷启动：1~10s     无冷启动概念
```

三类的边界：
- **FaaS** 你写函数（一段代码 + 入口签名），平台帮你打包、运行、扩缩
- **CaaS** 你交容器镜像，平台帮你 schedule、扩缩、扩到 0
- **BaaS** 你不写后端代码，调它的 API（数据库、认证、推送、文件）

**Knative Service 严格说是 CaaS**，不是 FaaS。它运行的是你打好的容器镜像，只是带了"扩到 0 + 按需启动"。要变成"函数"，得在上面叠一层（OpenFunction、Fission 等）。

> **口诀**：Serverless 是承诺，FaaS/CaaS/BaaS 是交付形态。Lambda 是 FaaS，Knative 是 CaaS，Firebase 是 BaaS。

### 1.3 Knative Serving：四个 CRD 的协作

Knative Serving 是 Knative 的"流量入口 + 扩缩"那一半。它定义了 4 个核心 CRD：

```
Service
  │
  ├── 维护 Configuration（容器镜像、env、资源等"期望")
  │
  └── 维护 Route（流量怎么分给各 Revision）

每次更新 Service 的 spec.template，Configuration 会生成一个新的：
Revision
  │
  ├── 不可变（image hash、env、并发配置都凝固了）
  │
  └── 由 Deployment + PodAutoscaler (PA) 实现

Route 决定 100% 流量进 v1，还是 90/10 灰度
```

四个 CRD 的关系：

| CRD | 作用 | 类比 |
| --- | --- | --- |
| **Service** | 用户定义的入口对象，最高层 | Deployment + Service + HPA + Ingress 的合体 |
| **Configuration** | 维护"当前期望的容器版本" | 像 ReplicaSet 之于 Deployment |
| **Revision** | 一个具体的、不可变的 Pod 模板 + 配置 | 一次部署的快照 |
| **Route** | 把外部流量分配给一个或多个 Revision | Ingress + 流量切分 |

**最反直觉的设计**：Revision 是不可变的。改 image 不是 update，是**生成新 Revision**。这让"金丝雀（Canary）发布"和"零停机回滚"成为天然能力 —— 旧 Revision 还在，路由切一下就行。

### 1.4 Knative Eventing：CloudEvents + Broker + Trigger

Knative Eventing 是 Knative 的"事件驱动"那一半。它的核心是把"事件源 → 处理函数"的连接抽象出来：

```
Event Source ──► Broker ──► Trigger（带过滤器）──► Service / Subscriber
   Kafka       （像 SQS）      ↓                       ↓
   GitHub                   过滤 type=order.created   你的 Knative Service
   PingSource                                          自动扩缩 / 扩到 0
   ApiServerSource
```

为什么不直接 Kafka → 函数？因为：
- **解耦**：函数不需要知道事件源是 Kafka 还是 RabbitMQ，统一从 Broker 收
- **重试 / 死信**：Broker 负责重试和死信队列
- **CloudEvents 标准化**：所有事件统一成 CloudEvents 格式，跨厂商可移植

CloudEvents（CNCF 标准）就是一段 JSON：

```json
{
  "specversion": "1.0",
  "type": "com.example.order.created",
  "source": "/orders",
  "id": "A234-1234-1234",
  "time": "2026-05-09T12:00:00Z",
  "datacontenttype": "application/json",
  "data": { "orderId": 42, "amount": 99.99 }
}
```

收到的 Knative Service 只看 HTTP body 是 CloudEvents 格式，不关心源。

### 1.5 KEDA：把"事件指标"翻译成 K8s HPA

KEDA（Kubernetes Event-Driven Autoscaler）解决的是 HPA 解决不了的问题：**按外部事件源的"队列深度"扩缩**。

HPA 只能按 CPU、内存、Custom Metrics（自己接 Prometheus）扩缩。如果你想"按 Kafka topic 的 lag 扩缩"，HPA 原生做不到。

KEDA 的架构：

```
Kafka topic 'orders'   <── KEDA Operator 周期 poll lag
    │
    │ lag = 5000
    ▼
KEDA 把 lag 翻译成：「该 Deployment 当前需要 N 个副本」
    │
    ▼
KEDA 写 ExternalMetrics API ──► HPA 读 ──► 调整 replicas
                                  │
                                  └── 扩到 0 时由 KEDA 直接操作 Deployment 的 replicas
```

KEDA 与 HPA 的关键差别：

| 维度 | HPA | KEDA |
| --- | --- | --- |
| 指标源 | CPU/Mem/Custom Metric | 50+ 种 Scaler（Kafka/RabbitMQ/Redis/SQS/...） |
| 最小副本 | 必须 ≥ 1 | **可以 = 0** |
| 实现机制 | Controller 直读 Metrics API | Controller poll 外部源 + 翻译成 ExternalMetrics |
| 与 HPA 关系 | 它就是 HPA | 在 HPA 之上，用 HPA 跑 [1,N]，自己处理 0↔1 |

**KEDA 能扩到 0 的关键**：当 lag = 0 持续 `cooldownPeriod` 后，KEDA 直接把 Deployment 的 replicas 设成 0，绕过 HPA（HPA 不允许 minReplicas=0）。来了第一条事件，KEDA 又把 replicas 设成 1，HPA 接管后续。

### 1.6 Scale-to-Zero 的代价：冷启动

扩到 0 听起来美好，但代价是**冷启动（cold start）**：第一个请求来时，还没有 Pod，需要：

```
请求 ──► Activator ──► (1) 触发扩容到 1
                       (2) Activator 缓存请求
                       (3) 等 Pod ready (containerProbe)
                       (4) Activator 把请求转给 Pod
                       (5) Activator 退出请求路径
```

冷启动总耗时 = 拉镜像（如果首次）+ 容器启动 + 应用初始化 + readinessProbe 通过。

不同应用形态的典型冷启动量级：

| 形态 | 镜像拉取 | 容器启动 | 应用初始化 | 总冷启动 |
| --- | --- | --- | --- | --- |
| Cloudflare Workers（V8 Isolate） | 0 | 0 | 0 | **< 5ms** |
| AWS Lambda（Node.js）| 已缓存 | 0 | ~100ms | **~150ms** |
| AWS Lambda（Java JVM）| 已缓存 | ~500ms | ~3s | **~3.5s** |
| Knative + Go | 已缓存 | ~50ms | ~50ms | **~200ms** |
| Knative + Spring Boot JVM | 已缓存 | ~500ms | **~5s** | **~5.5s** |
| Knative + Spring Boot GraalVM Native | 已缓存 | ~50ms | **~50ms** | **~150ms** |

冷启动是 Serverless 的**头号工程难题**。后面案例 ① 详细讲。

### 1.7 Activator + Autoscaler：Knative 的"扩到 0" 怎么实现

Knative Serving 有两个核心组件：

```
                        ┌──────────────┐
请求  ──► Ingress ─────► │   Activator  │ (在 0 副本时位于请求路径)
                        └──────┬───────┘
                               │ 转发
                               ▼
                        ┌──────────────┐
                        │  User Pod    │
                        └──────┬───────┘
                               │ metric: 当前并发数
                               ▼
                        ┌──────────────┐
                        │  Autoscaler  │ (KPA: Knative Pod Autoscaler)
                        └──────┬───────┘
                               │ 决策：要 N 个副本
                               ▼
                        ┌──────────────┐
                        │  Deployment  │
                        └──────────────┘
```

**Activator** 的两个角色：
- **Buffer**：0 副本时把请求 hold 住，等 Pod ready 后转发
- **Throttle**：高并发时 Activator 也可以位于请求路径，做请求级别的限流（防止把刚启动的 Pod 压垮）

**Autoscaler**（默认是 KPA，Knative Pod Autoscaler）的决策模型：
- **基于并发数**（不是 CPU）：`desiredPods = totalConcurrency / containerConcurrency`
- 6 秒 stable window + 60 秒 panic window 双窗口算法
- 突发流量进 panic mode（用 6s 窗口决策），平稳流量退回 stable

**KPA vs HPA**：Knative 也支持用 HPA 替代 KPA（`autoscaling.knative.dev/class: hpa.autoscaling.knative.dev`），但 HPA 的最小副本必须 ≥ 1，没法扩到 0。要扩到 0 必须用 KPA。

### 1.8 Knative vs AWS Lambda vs Cloudflare Workers

三种主流 Serverless 形态对比：

| 维度 | Knative (CaaS) | AWS Lambda (FaaS) | Cloudflare Workers |
| --- | --- | --- | --- |
| 隔离单位 | 容器 (cgroup + namespace) | MicroVM (Firecracker) | V8 Isolate |
| 冷启动 | 100ms ~ 5s | 100ms ~ 几秒 | < 5ms |
| 单实例最大内存 | 任意（你定） | 10 GB | 128 MB |
| 单次执行时长 | 任意（你定） | 15 min | 50ms CPU |
| 镜像格式 | OCI (任意语言) | Lambda 自定义 ZIP / 镜像 | JS / WASM |
| 跨地域分布 | 你自己搞 | Region 级 | **天然全球分布**（300+ PoP）|
| 自建 | ✅（K8s 上跑） | ❌ | ❌ |

Cloudflare Workers 的"快"靠 V8 Isolate：**同一个 V8 进程里跑成千上万个用户的代码**，每个 Isolate 是 V8 内的内存隔离区，启动一个 Isolate 只要分配几 MB 内存。代价是**只能跑 JS / WASM**，不能跑任意原生代码。

Knative 的"慢"是因为它走完整的 K8s Pod 生命周期：调度 → 拉镜像 → 启动容器 → readinessProbe。这是它能跑任意语言/任意镜像的代价。

> **口诀**：极致冷启动 → V8 Isolate；任意语言通用性 → 容器；K8s 自建 → Knative；公有云免运维 → Lambda。**没有银弹，按需选择**。

---

## 2. 工程切片：从最小可运行到生产配置

> 所有 YAML 基于 Knative 1.13、KEDA 2.13、K8s 1.28。

### 2.1 最小可运行的 Knative Service

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello
  namespace: default
spec:
  template:
    metadata:
      annotations:
        # 扩到 0 的最低 Pod 数（默认 0）
        autoscaling.knative.dev/min-scale: "0"
        # 最大 Pod 数
        autoscaling.knative.dev/max-scale: "10"
        # 单 Pod 目标并发数（默认 100）
        autoscaling.knative.dev/target: "10"
    spec:
      # 单 Pod 硬上限并发：超过会被 Activator 排队
      containerConcurrency: 100
      containers:
      - image: gcr.io/knative-samples/helloworld-go
        env:
        - name: TARGET
          value: "World"
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
```

部署：

```bash
kubectl apply -f hello.yaml

# 查看 Service 状态（Knative 自动创建 Configuration / Revision / Route）
kubectl get ksvc hello
# NAME    URL                                     LATESTCREATED   LATESTREADY   READY
# hello   http://hello.default.example.com       hello-00001     hello-00001   True

kubectl get revision
# NAME           CONFIG NAME   GENERATION   READY   REASON
# hello-00001    hello         1            True
```

访问测试：

```bash
curl -H "Host: hello.default.example.com" http://<ingress-ip>/

# 等 60 秒后再看（默认 stable-window=60s 后 idle）
kubectl get pods -l serving.knative.dev/service=hello
# 应该看不到 Pod，已经扩到 0

# 再 curl 一次，会看到 Pod 启动（冷启动）
```

**逐段点透**：
- `target: "10"` 是 Autoscaler 的"目标并发"，不是硬限制。**实际并发达到 10 就开始扩**
- `containerConcurrency: 100` 是 Pod 的硬上限，Activator 看到 Pod 已达 100 就把请求 hold 住等扩容
- 不写 `min-scale` 默认是 0，扩到 0；写 `min-scale: 1` 就**禁用扩到 0**，永远保留 1 副本（避免冷启动）

### 2.2 多 Revision + 流量切分（Canary）

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello
spec:
  template:
    metadata:
      name: hello-v2     # ★ 显式命名 Revision
    spec:
      containers:
      - image: gcr.io/knative-samples/helloworld-go:v2
  traffic:
  - revisionName: hello-v1
    percent: 90          # 旧版本 90%
  - revisionName: hello-v2
    percent: 10          # 新版本 10%
    tag: canary          # 给 v2 一个"专门访问"的 URL
```

应用后会有两个 URL：
- `http://hello.default.example.com` —— 90/10 分配
- `http://canary-hello.default.example.com` —— 100% 进 v2，**专门用来回归测试**

灰度通过后：

```yaml
  traffic:
  - revisionName: hello-v2
    percent: 100
```

如果出问题，路由切回 v1，**完全不重启容器**：

```yaml
  traffic:
  - revisionName: hello-v1
    percent: 100
```

旧 Revision 保留多久？由 GC 配置决定（默认保留最近 N 个）：

```yaml
# in ConfigMap config-gc
spec:
  data:
    "max-non-active-revisions": "20"
    "retain-since-create-time": "48h"
    "retain-since-last-active-time": "15h"
```

> **口诀**：Knative 的回滚是路由切换，不是重启 Pod。这是它比 Deployment + RollingUpdate 强的地方。

### 2.3 Knative 的 Activator + Autoscaler 实战配置

`config-autoscaler` ConfigMap 里的关键参数（生产必调）：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-autoscaler
  namespace: knative-serving
data:
  # 稳定窗口（默认 60s）：根据这窗口的平均并发决策
  stable-window: "60s"

  # panic 窗口比例（默认 10%，即 6s）
  panic-window-percentage: "10.0"

  # panic 触发阈值（默认 200%）：6s 内的并发 > 稳定的 2 倍 → 进 panic
  panic-threshold-percentage: "200.0"

  # 扩到 0 之前的 grace period（默认 30s）
  scale-to-zero-grace-period: "30s"

  # Pod 已经 idle 多久后才允许被销毁（默认 0s = 立即）
  scale-to-zero-pod-retention-period: "0s"

  # 单 Pod 目标并发（默认 100）
  container-concurrency-target-default: "100"

  # 突发流量保护：Activator 会持续位于请求路径，单 Pod 平均并发超过此值时
  target-burst-capacity: "200"
```

**target-burst-capacity 是最容易踩坑的参数**：
- = 0：Activator 只在 0 副本时位于请求路径
- > 0：Activator 长期位于请求路径，能吸收突发流量但增加一跳延迟
- = -1：Activator 永远位于请求路径

如果你的服务突发流量高，建议 `target-burst-capacity > 0`。如果追求极致 P99 latency，设 0 但要承担"突发流量打挂 Pod"的风险。

### 2.4 KEDA ScaledObject 三种典型触发器

#### Kafka 触发器

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer
  namespace: default
spec:
  scaleTargetRef:
    name: order-consumer       # 你的 Deployment 名字
  pollingInterval: 30          # KEDA poll 触发器的间隔（默认 30s）
  cooldownPeriod: 300          # 没事件后保持 N 秒再扩到 0
  minReplicaCount: 0           # ★ 扩到 0
  maxReplicaCount: 50
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka.kafka.svc:9092
      consumerGroup: order-group
      topic: orders
      lagThreshold: "100"      # 单 Pod 目标 lag = 100
      offsetResetPolicy: latest
```

机理：KEDA 周期 poll Kafka，得到当前消费组的 lag。如果 lag = 5000、单 Pod lagThreshold = 100，KEDA 算 desiredReplicas = 50。

#### RabbitMQ 触发器

```yaml
spec:
  triggers:
  - type: rabbitmq
    metadata:
      protocol: amqp
      queueName: tasks
      mode: QueueLength
      value: "20"              # 单 Pod 目标队列长度 = 20
    authenticationRef:
      name: rabbitmq-auth      # TriggerAuthentication 引用
```

#### Cron 触发器（"工作时间常驻、夜间扩到 0"）

```yaml
spec:
  triggers:
  - type: cron
    metadata:
      timezone: Asia/Shanghai
      start: 0 9 * * *        # 早 9 点开始
      end: 0 18 * * *         # 晚 6 点结束
      desiredReplicas: "5"    # 工作时间保持 5 副本
```

非工作时间走其他触发器（或没有触发器）的逻辑 → 扩到 0。

#### 多触发器组合

```yaml
spec:
  triggers:
  - type: kafka
    ...
  - type: cron
    ...
```

**KEDA 会取所有触发器算出的 desiredReplicas 的最大值**。这是一个常被忽略的特性：组合 Cron + Kafka 可以做到"工作时间常驻 N 副本，夜间按 lag 扩到 0"。

> **口诀**：KEDA 的 `lagThreshold` 不是"总 lag 上限"，是"单 Pod 应该处理的 lag 量"。算副本数 = 总 lag / lagThreshold。

### 2.5 Knative Eventing 完整链路

最小化的事件处理 demo（Kafka → Broker → Trigger → Knative Service）：

#### 1. 安装 Broker（基于 Kafka）

```yaml
apiVersion: eventing.knative.dev/v1
kind: Broker
metadata:
  name: orders-broker
  namespace: default
  annotations:
    eventing.knative.dev/broker.class: Kafka
spec:
  config:
    apiVersion: v1
    kind: ConfigMap
    name: kafka-broker-config
    namespace: knative-eventing
```

#### 2. KafkaSource：把 Kafka topic 变成 CloudEvents 灌进 Broker

```yaml
apiVersion: sources.knative.dev/v1beta1
kind: KafkaSource
metadata:
  name: orders-source
spec:
  bootstrapServers:
  - kafka.kafka.svc:9092
  topics:
  - orders
  consumerGroup: orders-knative-source
  sink:
    ref:
      apiVersion: eventing.knative.dev/v1
      kind: Broker
      name: orders-broker
```

#### 3. Knative Service：处理函数

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: order-processor
spec:
  template:
    spec:
      containers:
      - image: registry.example.com/order-processor:v1
```

#### 4. Trigger：把"特定类型的事件"路由到这个 Service

```yaml
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: order-created-trigger
spec:
  broker: orders-broker
  filter:
    attributes:
      type: com.example.order.created
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: order-processor
```

效果：Kafka topic `orders` 的每条消息变成 CloudEvent → Broker → 过滤 `type=com.example.order.created` → Knative Service。Service 没流量时扩到 0，事件来了自动启动。

### 2.6 GraalVM Native + Spring Boot + Knative

把 JVM Spring Boot 包成 GraalVM Native，冷启动从 5s 降到 50ms。

#### Spring Boot 3 项目结构

```xml
<!-- pom.xml -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.graalvm.buildtools</groupId>
            <artifactId>native-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

#### Dockerfile（多阶段）

```dockerfile
FROM ghcr.io/graalvm/graalvm-community:21 AS builder
RUN gu install native-image
WORKDIR /app
COPY . .
RUN ./mvnw -Pnative native:compile -DskipTests

FROM gcr.io/distroless/base-debian12
COPY --from=builder /app/target/myapp /myapp
ENTRYPOINT ["/myapp"]
EXPOSE 8080
```

镜像构建（耗时 5~10 分钟）：

```bash
docker build -t registry.example.com/myapp:native-v1 .
docker push registry.example.com/myapp:native-v1
```

#### Knative Service

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: myapp-native
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/min-scale: "0"
        autoscaling.knative.dev/max-scale: "20"
    spec:
      containers:
      - image: registry.example.com/myapp:native-v1
        resources:
          requests:
            cpu: 200m
            memory: 64Mi      # ★ Native 镜像内存小很多
          limits:
            memory: 256Mi
        readinessProbe:
          httpGet:
            path: /actuator/health
          initialDelaySeconds: 0    # ★ Native 启动 50ms，不需要 delay
          periodSeconds: 1
```

冷启动测试：

```bash
# 等扩到 0
sleep 90

# 测冷启动
time curl http://myapp-native.default.example.com/hello
# real    0m0.180s     # ★ 包括网络 + 冷启动
```

JVM 版本同条件下：

```bash
time curl http://myapp-jvm.default.example.com/hello
# real    0m5.420s
```

差 30 倍。

### 2.7 自定义 Scaler（External Scaler）

KEDA 内置 50+ Scaler，但如果你的事件源是自家系统（比如自研 MQ），需要写 External Scaler。

接口（gRPC，KEDA 周期调用）：

```protobuf
service ExternalScaler {
  rpc IsActive(ScaledObjectRef) returns (IsActiveResponse);
  rpc StreamIsActive(ScaledObjectRef) returns (stream IsActiveResponse);
  rpc GetMetricSpec(ScaledObjectRef) returns (GetMetricSpecResponse);
  rpc GetMetrics(GetMetricsRequest) returns (GetMetricsResponse);
}
```

Go 实现骨架：

```go
type Scaler struct{}

func (s *Scaler) IsActive(ctx context.Context, ref *pb.ScaledObjectRef) (*pb.IsActiveResponse, error) {
    queueName := ref.ScalerMetadata["queue"]
    depth, err := myQueueClient.Depth(queueName)
    if err != nil {
        return nil, err
    }
    return &pb.IsActiveResponse{Result: depth > 0}, nil
}

func (s *Scaler) GetMetricSpec(ctx context.Context, ref *pb.ScaledObjectRef) (*pb.GetMetricSpecResponse, error) {
    return &pb.GetMetricSpecResponse{
        MetricSpecs: []*pb.MetricSpec{{
            MetricName: "queueDepth",
            TargetSize: 100,        // 单 Pod 目标深度
        }},
    }, nil
}

func (s *Scaler) GetMetrics(ctx context.Context, req *pb.GetMetricsRequest) (*pb.GetMetricsResponse, error) {
    queueName := req.ScaledObjectRef.ScalerMetadata["queue"]
    depth, _ := myQueueClient.Depth(queueName)
    return &pb.GetMetricsResponse{
        MetricValues: []*pb.MetricValue{{
            MetricName:  "queueDepth",
            MetricValue: int64(depth),
        }},
    }, nil
}
```

KEDA 端引用：

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: my-queue-scaler
spec:
  scaleTargetRef:
    name: worker
  minReplicaCount: 0
  maxReplicaCount: 30
  triggers:
  - type: external
    metadata:
      scalerAddress: my-external-scaler.default.svc:6000
      queue: tasks
```

> **口诀**：External Scaler 是兜底方案。能用内置 50+ Scaler 就别自己写 —— 内置的处理了 reconnect、auth、metric 缓存等一堆细节。

---

## 3. 实测数据：冷启动、节省、突发流量

### 3.1 不同语言的冷启动对比

测试方法：扩到 0 后 sleep 90s，发起请求，记录从请求发出到收到第一字节的时间（TTFB）。镜像已 pre-pull 到节点。

测试机器：3 节点 K8s 1.28，每节点 4C8G，本地 NVMe，Knative 1.13 默认配置。

| 语言/框架 | 镜像大小 | TTFB（冷） | TTFB（热） |
| --- | --- | --- | --- |
| Go (net/http) | 12 MB | **180ms** | 8ms |
| Node.js (express) | 80 MB | **350ms** | 12ms |
| Python (FastAPI) | 90 MB | **800ms** | 15ms |
| Java Spring Boot 3 (JVM) | 280 MB | **5400ms** | 10ms |
| Java Spring Boot 3 (GraalVM Native) | 95 MB | **190ms** | 9ms |
| Java Quarkus (GraalVM Native) | 70 MB | **130ms** | 8ms |
| Rust (axum) | 8 MB | **120ms** | 5ms |

**关键发现**：
- Go / Rust 的天然优势：编译型 + 小镜像 + 快启动
- JVM 是冷启动重灾区，主要时间在 JIT、Spring 反射扫描、Bean 初始化
- GraalVM Native 把 Java 拉到了 Go 的水平
- Python 比想象的慢（因为依赖加载多）

### 3.2 GraalVM Native 后的冷启动改善

同一个 Spring Boot 3 应用（含 30 个 Controller、连 PostgreSQL）：

| 模式 | 启动时间 | 内存（RSS）| 镜像大小 | 编译时间 |
| --- | --- | --- | --- | --- |
| JVM (HotSpot) | 5.4s | 350 MB | 280 MB | 30s |
| GraalVM Native | 0.05s | 80 MB | 95 MB | **6 min** |

代价：
- 编译慢 12 倍
- 反射、动态代理需要写 reflect-config.json（Spring AOT 自动生成大部分）
- 调试比 JVM 难

收益：
- 冷启动 100 倍
- 内存 4 倍

**云原生 / Serverless 场景 GraalVM Native 是 Java 的唯一活路**。

### 3.3 Scale-to-Zero 的实际资源节省

场景：100 个微服务，每个服务平均日活 4 小时（其余时间闲）。每个服务永久 Pod 占用 0.2 CPU + 256MB。

| 部署模式 | 总 CPU 占用 | 总内存占用 | 月成本（按云厂商定价）|
| --- | --- | --- | --- |
| 永久 Pod | 20 CPU | 25 GB | 100% 基线 |
| HPA min=1 | 20 CPU | 25 GB | 100% |
| Knative Scale-to-Zero | 平均 ~3 CPU | ~4 GB | **~17%** |

节省 83%。但代价是：
- 冷启动用户感知（必须配 GraalVM 或 min-scale=1 for hot services）
- 调度压力（频繁创建销毁 Pod）
- Image pull 流量（如果节点没缓存）

**真实生产建议**：核心服务 `min-scale=1` 永远保留 1 副本；非核心、突发性服务用 scale-to-zero。

### 3.4 KEDA vs HPA：突发流量的对比

测试场景：消息队列 lag 在 30s 内从 0 飙到 50000，看扩容速度。

| 方案 | 扩到目标副本数耗时 | 滞后 | 准确度 |
| --- | --- | --- | --- |
| HPA (CPU 70%) | **120s** | CPU 指标天然滞后 30s | 间接，不准 |
| HPA (Custom Metric: lag/1000) | 60s | Prometheus 抓取 30s + HPA 决策 30s | 较准 |
| **KEDA (Kafka lag scaler)** | **45s** | KEDA pollingInterval 30s | **直接、准确** |

KEDA 胜在：
1. **直接读源**，不经过 Prometheus 中转
2. **专为这种场景设计**，决策逻辑更简单
3. **能扩到 0**，HPA 不能

但 KEDA 也有坑：
- pollingInterval 不能太短（默认 30s），太短会压垮事件源
- KEDA Operator 是单点，重启时所有 ScaledObject 短暂不工作

### 3.5 Knative vs OpenFaaS 性能对比

同一个 Go HTTP 函数（返回 hello），同集群部署：

| 指标 | Knative | OpenFaaS |
| --- | --- | --- |
| 冷启动 | 200ms | 350ms（NATS 触发更慢）|
| 热请求 P99 | 12ms | 15ms |
| 单 Pod QPS | 8000 | 6500 |
| 扩到 0 | 默认支持 | 需要配置 idler 组件 |
| 事件驱动 | Eventing 模块（CloudEvents 标准）| 内置但非标准 |
| 灰度发布 | 一等公民 | 需要外部工具 |
| 学习曲线 | 较陡 | 较平 |

Knative 在性能、标准化上胜出；OpenFaaS 在易用性、轻量上胜出。

> **口诀**：Knative 是"K8s 原生 Serverless 的 de facto 标准"；OpenFaaS 适合小集群快速上手；KEDA 是 Knative 之外的事件驱动扩缩选项。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① Java Spring 冷启动 5s —— "用户首次访问转圈"

#### 现象

Spring Boot 3 应用部署成 Knative Service，min-scale=0。低频接口扩到 0 后，下一个用户请求要等 5~6s 才返回，浏览器一直转圈。监控显示：扩容 + Pod ready 总耗时 5400ms，其中应用 init 占 4500ms。

#### 根因

Spring Boot 启动慢的几个根因：
1. **JVM 启动**：JIT 预热、类加载（~500ms）
2. **Spring 上下文初始化**：扫描 @Component、初始化 BeanFactory（~2s）
3. **AutoConfiguration**：spring.factories 里几十个自动配置类全部检查（~1s）
4. **依赖初始化**：DataSource 连接池初始化、HikariCP 建连（~1s）
5. **WebServer 启动**：Tomcat 初始化（~500ms）

每一项都是几百毫秒，加起来就 5s。

#### 修复

**方案 A：永远保留 1 副本（最简单）**

```yaml
metadata:
  annotations:
    autoscaling.knative.dev/min-scale: "1"   # 不扩到 0
```

适合核心高频服务，代价是失去"按需付费"。

**方案 B：GraalVM Native Image（最有效）**

参见 [2.6](#_26-graalvm-native--spring-boot--knative)。冷启动从 5400ms 降到 190ms，下降 95%。

代价：
- 镜像编译时间长（5~10 分钟）
- 需要适配 reflect / proxy / resources（Spring Boot 3 的 AOT 自动生成大部分）
- 第三方库可能不兼容（每个库要确认 native 兼容性）

**方案 C：CRaC（Coordinated Restore at Checkpoint，JDK 21+）**

```bash
# 启动到稳定状态
java -XX:CRaCCheckpointTo=/checkpoint -jar app.jar
# 触发 checkpoint
jcmd <pid> JDK.checkpoint
# 下次启动从 checkpoint 恢复
java -XX:CRaCRestoreFrom=/checkpoint
```

冷启动可降到 ~100ms。坑：checkpoint 文件包含运行时状态（包括 socket、文件描述符），云原生场景需要小心处理。

**方案 D：分层启动 + 提前准备好镜像缓存**

- 用 distroless 减小镜像（拉取快）
- 节点配 `imagePullPolicy: IfNotPresent` 避免重复拉
- 用 DaemonSet pre-pull 关键镜像

#### 验证

```bash
# 多次冷启动取平均
for i in {1..10}; do
  kubectl scale --replicas=0 deployment/myapp-jvm
  sleep 60
  time curl -o /dev/null -s http://myapp.default.example.com/hello
done
```

修复前平均 5.4s，方案 B 后 190ms。

> **口诀**：JVM Serverless 必踩坑。**核心服务 min-scale=1，非核心服务上 GraalVM Native**。

---

### 案例 ② Scale-to-Zero 后 DB 连接重新建立慢 —— "扩到 0 后第一个请求慢 2 秒"

#### 现象

Knative Service 扩到 0 后，第一个请求耗时 2.5s。监控显示应用启动只用 200ms（已 GraalVM Native），但 readinessProbe 通过到响应业务的间隔有 2s 空白。

#### 根因

应用代码：

```java
@Bean
public DataSource dataSource() {
    HikariConfig config = new HikariConfig();
    config.setJdbcUrl("jdbc:postgresql://postgres:5432/orders");
    config.setMinimumIdle(5);
    config.setMaximumPoolSize(20);
    return new HikariDataSource(config);
}
```

启动时 Hikari 异步初始化连接池，应用 ready 时连接池**还没建好**。第一个请求来时：
1. 应用拿连接 → 连接池没空闲连接 → 创建新连接
2. 创建连接 = TCP 握手 + TLS 握手 + Postgres 鉴权 + 初始化 schema 检查
3. 单个连接建立 ~200~500ms
4. 后续请求要补满 minimumIdle，再创建几个连接

总计第一个请求等 1.5~2s。

#### 修复

**方案 A：readinessProbe 检查连接池是否就绪**

```java
@RestController
public class HealthController {
    @Autowired DataSource dataSource;

    @GetMapping("/ready")
    public ResponseEntity<String> ready() throws SQLException {
        // 强制拿一个连接验证
        try (Connection c = dataSource.getConnection()) {
            c.createStatement().execute("SELECT 1");
        }
        return ResponseEntity.ok("ready");
    }
}
```

```yaml
readinessProbe:
  httpGet:
    path: /ready
  initialDelaySeconds: 0
  periodSeconds: 1
  failureThreshold: 30
```

应用启动后必须等连接池建好才 ready，Activator 把请求 hold 在 buffer 里等 ready。

**方案 B：减少 minimumIdle，按需建连**

```java
config.setMinimumIdle(0);   // 先不预建，按需
config.setConnectionTimeout(5000);
```

代价：第一个请求仍然需要建连。但后续连接复用，问题只在第一次。

**方案 C：连接池预热脚本（init container）**

```yaml
initContainers:
- name: warm-db
  image: postgres:15
  command: ["psql", "-h", "postgres", "-U", "user", "-d", "orders", "-c", "SELECT 1"]
```

不解决根本问题（init container 跑完容器才启动），但能确保 DB 端预热。

**方案 D：Knative 配置缓冲**

```yaml
metadata:
  annotations:
    autoscaling.knative.dev/initial-scale: "1"   # 第一次启动直接到 1
    serving.knative.dev/progress-deadline: "60s"
```

#### 验证

```bash
# 扩到 0 后多次测
for i in {1..5}; do
  kubectl scale --replicas=0 ksvc/myapp
  sleep 60
  time curl http://myapp/api/order/1
done
```

修复前 2.5s，修复后 350ms。

> **口诀**：冷启动不只是"应用启动"，还包括"应用依赖（DB / Redis / Kafka）的连接建立"。**readinessProbe 必须验证关键依赖**。

---

### 案例 ③ KEDA 触发器配置错误导致不扩缩 —— "Kafka 堆积百万消息无人处理"

#### 现象

Kafka topic `orders` 突然堆了 100 万消息，监控告警 lag 持续涨。KEDA ScaledObject 已配置，Deployment replicas 仍为 0。`kubectl describe scaledobject` 看不到错误。

#### 根因

`kubectl get scaledobject orders-scaler -o yaml`：

```yaml
spec:
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka.kafka.svc:9092
      consumerGroup: order-group-v1     # ★ 业务实际用 order-group-v2
      topic: orders
      lagThreshold: "100"
```

业务侧消费组改名为 `order-group-v2`，但 KEDA 配置里还是 v1。KEDA poll 的是 `order-group-v1` 的 lag —— 这个组**不存在**，lag = 0，KEDA 认为不需要扩。

更糟的是：KEDA 默认对"组不存在"不报错，只是 metric 返回 0。

#### 修复

**第一步**：找到正确的 consumerGroup。

```bash
kafka-consumer-groups.sh --bootstrap-server kafka:9092 --list
# 输出所有 group，找到业务实际用的
```

**第二步**：修正 ScaledObject。

```yaml
spec:
  triggers:
  - type: kafka
    metadata:
      consumerGroup: order-group-v2
      ...
```

**第三步**：装更严的监控。

```bash
# 监控 KEDA 自身的 metric
kubectl get --raw /apis/external.metrics.k8s.io/v1beta1/namespaces/default/s0-kafka-orders | jq

# 看 KEDA Operator 日志
kubectl logs -n keda deployment/keda-operator --tail=200
```

**第四步**：Kafka lag 双重监控（不能只信 KEDA）。

直接用 Burrow 或 kafka-exporter 把 lag 暴露给 Prometheus，配独立告警 `kafka_consumergroup_lag > 10000` 持续 2 分钟。这样即使 KEDA 配错，业务监控仍能发现问题。

#### 验证

```bash
# 给 topic 灌测试消息
kafka-producer-perf-test.sh --topic orders --num-records 1000 --record-size 100 \
  --throughput 100 --producer-props bootstrap.servers=kafka:9092

# 观察 KEDA 是否扩
kubectl get deployment order-consumer -w
```

修复后 30 秒内开始扩，2 分钟内消化堆积。

> **口诀**：KEDA 配置的 consumerGroup / queueName 必须与业务实际一致。**KEDA 自身指标 + 业务源指标双监控，谁错都能发现**。

---

### 案例 ④ 事件丢失 —— "Broker 重启时一批订单事件丢了"

#### 现象

Knative Eventing + Kafka Broker 部署。某次集群升级 Broker 重启，重启期间 Kafka 有几百条新消息进来。重启完成后，业务发现这几百条对应的 Knative Service **从未收到**。

#### 根因

Knative Eventing 的 Broker（基于 Kafka 实现的 KafkaSource → Broker → Trigger 链路）：

```
Kafka topic ──► KafkaSource (consumer)
                    │
                    │ 转换成 CloudEvent，发到 Broker（内部 topic）
                    ▼
                Broker 内部 topic ──► Dispatcher
                                          │
                                          │ HTTP POST 到 Subscriber
                                          ▼
                                    Knative Service
```

每一跳都可能丢：
1. **KafkaSource → Broker**：如果 KafkaSource 进程崩溃后才 commit offset，会丢
2. **Broker → Dispatcher**：内部 topic 的复制因子 < 3 时，单 broker 故障可能丢
3. **Dispatcher → Service**：HTTP POST 失败时是否重试？重试到死信队列吗？

实际事故的根因是 Dispatcher 配置：

```yaml
apiVersion: eventing.knative.dev/v1
kind: Trigger
spec:
  delivery:
    retry: 0          # ★ 不重试！
    deadLetterSink:    # ★ 没配死信
      ...
```

Subscriber 重启期间，Dispatcher 拿到 503，因为 retry=0 直接丢弃。

#### 修复

**方案 A：必配 retry + 死信队列**

```yaml
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: order-trigger
spec:
  broker: orders-broker
  delivery:
    retry: 5
    backoffPolicy: exponential
    backoffDelay: PT1S       # 1s, 2s, 4s, 8s, 16s
    deadLetterSink:
      ref:
        apiVersion: v1
        kind: Service
        name: dlq-handler
  filter:
    attributes:
      type: com.example.order.created
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: order-processor
```

死信里的事件可以人工处理或自动重投。

**方案 B：保证消费幂等**

死信和重试都依赖于 Subscriber 是幂等的。事件 ID 必须能用于去重：

```java
@PostMapping("/")
public ResponseEntity<Void> handle(@RequestHeader("ce-id") String eventId,
                                    @RequestBody Order order) {
    if (processedEventDao.exists(eventId)) {
        return ResponseEntity.ok().build();   // 已处理过，直接 ack
    }
    process(order);
    processedEventDao.save(eventId);
    return ResponseEntity.ok().build();
}
```

**方案 C：Broker 高可用配置**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kafka-broker-config
  namespace: knative-eventing
data:
  default.topic.partitions: "10"
  default.topic.replication.factor: "3"     # ★ 至少 3
  bootstrap.servers: kafka.kafka.svc:9092
```

#### 验证

故意在 Broker 升级时灌入消息，确认 Subscriber 能收到全部（可能延迟，但不丢）。

```bash
# 灌测试消息
for i in {1..100}; do
  kafkacat -P -b kafka:9092 -t orders -k "test-$i" \
    <<< '{"orderId": '$i'}'
done

# 同时滚 Broker
kubectl rollout restart -n knative-eventing deployment/kafka-broker

# 验证 Subscriber 处理日志包含全部 100 条
kubectl logs deploy/order-processor | grep -c "processed"
```

> **口诀**：Eventing 链路上每一跳都可能丢。**retry + deadLetter + 幂等**三件套缺一不可。

---

### 案例 ⑤ 多 Revision 流量切换异常 —— "灰度切了 100% 还有 1% 走旧版本"

#### 现象

Knative Service 灰度，配置 Traffic 100% 到 v2，已等 5 分钟。监控仍发现 ~1% 请求落到 v1。

#### 根因

可能根因有多个：

1. **Ingress 缓存**：Ingress（Kourier / Istio）的 Endpoint 缓存有 TTL，旧路由短暂存在
2. **Pod 优雅退出**：v1 的 Pod 在收到 SIGTERM 后还有 in-flight 请求要处理完
3. **Knative Activator 缓存**：Activator 内部的路由表更新有秒级延迟
4. **客户端长连接**：HTTP keep-alive 让客户端连到 v1 的 Pod 上不释放

排查方法：

```bash
# 1. 查 Route 状态
kubectl describe route hello

# 2. 查实际流量
kubectl get pods -l serving.knative.dev/service=hello -o wide
# 看 v1 和 v2 的 Pod 状态

# 3. Ingress 配置
kubectl get -n kourier-system svc kourier
kubectl logs -n kourier-system deployment/3scale-kourier-control --tail=100
```

通常根因是**长连接**：客户端用 HTTP/1.1 keep-alive 或 HTTP/2 多路复用，连到 v1 的 Pod 后就不切换，直到连接关闭。

#### 修复

**方案 A：客户端配置短连接 / 禁用 keep-alive**（不可能要求所有客户端这么做）

**方案 B：v1 Pod 主动断开连接**

在 v1 Service 配置 readinessProbe 失败 → 让客户端连接被服务端主动 close。

```yaml
# v1 灰度结束后，把 v1 的 minScale 设成 0
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/min-scale: "0"
```

让 v1 自然扩到 0，旧连接随之释放。

**方案 C：用 Tag 路由 + 客户端切换**

```yaml
spec:
  traffic:
  - revisionName: hello-v1
    percent: 0
    tag: v1
  - revisionName: hello-v2
    percent: 100
    tag: v2
```

需要旧版本时通过 `v1.hello.example.com` 显式访问。

**方案 D：理解"100% 切换不是瞬间生效"**

接受：流量切换有 30s ~ 2min 的过渡期。生产建议 100% 切换后再观察 5 分钟才宣布发布完成。

#### 验证

```bash
# 持续打流量 5 分钟，统计落到各 Revision 的占比
for i in {1..1000}; do
  curl -s -o /dev/null -w "%{http_code}\n" http://hello/version
done | sort | uniq -c
```

> **口诀**：Knative 流量切换不是瞬间。客户端长连接 + Ingress 缓存让 100% 切换有过渡期。**接受过渡 + 监控 + 等够时间**。

---

### 案例 ⑥ Knative + Service Mesh 兼容性 —— "Istio sidecar 让 Knative 探活失败"

#### 现象

集群已有 Istio。某天给 namespace 打上 `istio-injection=enabled`，所有 Knative Service 突然变成 NotReady。报错：

```
Revision "hello-00001" failed: Container failed with: container readiness probe failed
```

#### 根因

Istio 的 sidecar（Envoy）在 Pod 启动时**晚于业务容器** ready。Knative 的 readinessProbe 检查业务容器的端口，请求先到 Envoy → Envoy 还没 ready → 返回错误 → Knative 判定 Revision 失败。

更深层的冲突：
- Knative Activator → Pod 之间走的是 K8s Service IP，会被 Istio sidecar 拦截
- Activator 不在 mesh 里，sidecar 的 mTLS 会拒绝它
- Knative 自己维护的"流量切分"和 Istio VirtualService 互相覆盖

#### 修复

**方案 A：Knative 官方文档：用 Istio 作为 Knative 的 Ingress**

```yaml
# 安装 Knative 时选 Istio 作为 networking layer
# config-network ConfigMap 里：
data:
  ingress.class: istio.ingress.networking.knative.dev
```

Knative 把流量切分等工作委托给 Istio，避免冲突。

**方案 B：Knative Service 加 Istio 排除注解**

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
spec:
  template:
    metadata:
      annotations:
        sidecar.istio.io/inject: "false"   # ★ 不注入 Envoy
```

简单粗暴，缺点是这个 Service 失去 mesh 能力（mTLS、流量观测）。

**方案 C：调整 Pod 启动顺序**

Istio 1.7+ 支持 `holdApplicationUntilProxyStarts`：

```yaml
metadata:
  annotations:
    proxy.istio.io/config: |
      holdApplicationUntilProxyStarts: true
```

业务容器等 Envoy ready 后才启动，避免 ready 检查时序问题。但增加了启动延迟（雪上加霜）。

**方案 D：Activator 加进 mesh（高级）**

把 knative-serving 也打上 `istio-injection=enabled`，Activator 的 sidecar 也在 mesh 里。需要仔细配置 mTLS 模式，否则集群级故障。

#### 验证

```bash
kubectl describe ksvc hello | grep -A5 Conditions
# 所有 condition 应该 True
```

> **口诀**：Knative + Istio 是两个有重叠职责的系统。**要嘛让 Istio 当 Knative 的 networking，要嘛在关键服务排除 sidecar 注入**。混跑必踩。

---

### 案例 ⑦ 长任务超时 —— "Knative 默认 30s，PDF 生成跑不完"

#### 现象

业务用 Knative Service 跑 PDF 生成，平均耗时 60s。结果客户端 30s 时收到 504。日志：

```
upstream request timeout
```

#### 根因

Knative Service 的两层超时：
1. **`timeoutSeconds`**（Revision 级，默认 300s）：单请求最大耗时
2. **Activator 的请求超时**（默认 600s，Knative 1.10+）

但**第三层超时**才是凶手：Ingress / Service Mesh 的超时。Kourier 默认 30s，Istio 的 VirtualService 默认 0（无限）但被 Envoy 全局策略覆盖。

#### 修复

**方案 A：调 Knative Service 的 timeoutSeconds**

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
spec:
  template:
    spec:
      timeoutSeconds: 600    # 10 分钟
      containers:
      - image: ...
```

**方案 B：调 Ingress 超时**

Kourier:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-kourier
  namespace: knative-serving
data:
  request-timeout: "600s"
```

Istio:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: hello
spec:
  http:
  - timeout: 600s
    route:
    - destination:
        host: hello.default.svc.cluster.local
```

**方案 C：长任务异步化（推荐）**

PDF 生成这种长任务**不该用同步 HTTP**。改成：
1. 用户提交请求 → 立即返回 jobId
2. 后端把任务塞进队列（Kafka / Redis）
3. KEDA 触发 worker（Knative Service）按 lag 扩缩
4. 用户轮询 jobId 或 WebSocket 通知

```java
@PostMapping("/generate-pdf")
public ResponseEntity<JobResponse> generate(@RequestBody PdfRequest req) {
    String jobId = UUID.randomUUID().toString();
    queue.send(new PdfJob(jobId, req));
    return ResponseEntity.accepted().body(new JobResponse(jobId));
}
```

异步化天然适配 Serverless：worker 服务可以扩到 0 + 按 lag 扩缩。

#### 验证

```bash
# 长请求 540s
time curl -X POST http://hello/long-task
```

#### 拓展

Knative 不适合**真正的长任务**（小时级）。这种场景考虑：
- K8s Job / CronJob
- Argo Workflows / Tekton
- 自定义 Operator

> **口诀**：Knative 同步 HTTP 适合 < 5min 的任务。长任务必须异步化，配合 KEDA 做"队列驱动 worker"。

---

### 案例 ⑧ 函数日志聚合困难 —— "Pod 频繁创建销毁，日志查不全"

#### 现象

某个 Knative Service 每天扩缩几百次，单 Pod 平均存活 2 分钟。要查"昨天 14:30 这个请求的完整链路"时，对应 Pod 早就销毁了，`kubectl logs` 拿不到。

#### 根因

K8s 日志的根本机制：日志是写到 Pod 所在节点的 stdout/stderr，再由 kubelet 收集到 `/var/log/containers/`。Pod 一销毁，节点上的日志文件**会被 kubelet 在保留窗口后清理**（默认几小时）。

Serverless 高频销毁场景下，**日志必须实时采集到中心化系统**。

#### 修复

**方案 A：DaemonSet 采集 + 中心存储**

```yaml
# 每个节点跑一个 fluent-bit/vector
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: vector
spec:
  template:
    spec:
      containers:
      - name: vector
        image: timberio/vector:latest
        volumeMounts:
        - name: var-log
          mountPath: /var/log
          readOnly: true
        - name: config
          mountPath: /etc/vector
      volumes:
      - name: var-log
        hostPath:
          path: /var/log
```

Vector 配置：

```toml
[sources.k8s]
type = "kubernetes_logs"

[transforms.parse]
type = "remap"
inputs = ["k8s"]
source = '''
  . |= parse_json!(.message) ?? {}
'''

[sinks.loki]
type = "loki"
inputs = ["parse"]
endpoint = "http://loki:3100"
labels.app = "{{ kubernetes.labels.app }}"
labels.revision = "{{ kubernetes.labels[\"serving.knative.dev/revision\"] }}"
```

查询时按 trace_id 找：

```logql
{app="hello"} |= "trace_id=abc123"
```

**方案 B：应用直接 push 日志到中心**

```java
// Logback config
<appender name="LOKI" class="com.github.loki4j.logback.Loki4jAppender">
    <http>
        <url>http://loki:3100/loki/api/v1/push</url>
    </http>
    <format>
        <label>
            <pattern>app=myapp,revision=${KNATIVE_REVISION}</pattern>
        </label>
    </format>
</appender>
```

绕过节点采集，但增加应用复杂度。

**方案 C：分布式追踪 + 日志关联**

每条日志带 trace_id，关联 Jaeger / Tempo 的 trace：

```java
private static final Logger log = LoggerFactory.getLogger(MyClass.class);

public void handle(Request req) {
    String traceId = MDC.get("traceId");
    log.info("processing request, traceId={}", traceId);
}
```

通过 trace_id 串起跨 Pod、跨服务的所有日志。

#### 验证

灰度后查 24 小时前的某个具体请求 trace_id，能完整查到所有日志。

> **口诀**：Serverless = Pod 短命 = 日志必须实时外发。**DaemonSet 采集 + 集中存储 + trace_id 串联**是标配。

---

### 案例 ⑨ 跨 Region 容灾 —— "主 region 挂了 Knative 服务全断"

#### 现象

业务部署在 us-east-1 的 K8s 集群，Knative Service 扩缩正常。某次 us-east-1 IDC 故障，业务全断。备 region us-west-2 也有 K8s，但服务没自动切。

#### 根因

Knative 是**集群本地组件**，没有跨集群能力。要做跨 region 容灾，必须在更上层处理：

1. **DNS 层**：跨 region 的 Anycast / GSLB
2. **多集群部署**：每个 region 一份 Knative Service
3. **状态层**：DB / Kafka 跨 region 复制

#### 修复

**多活架构（推荐）**

```
                    Cloudflare / Route 53 GSLB
                    ┌─────────────────────────┐
                    │  hello.example.com      │
                    │  health-check based     │
                    │  routing                │
                    └────┬────────────────┬───┘
                         │                │
                ┌────────▼─────┐  ┌──────▼──────┐
                │  us-east-1   │  │  us-west-2  │
                │  K8s Cluster │  │  K8s Cluster│
                │  Knative     │  │  Knative    │
                │  Service A   │  │  Service A  │
                └──────┬───────┘  └──────┬──────┘
                       │                 │
                       └────────┬────────┘
                                │
                        ┌───────▼────────┐
                        │  Aurora Global │ (DB 跨 region)
                        │  Kafka MirrorMaker
                        └────────────────┘
```

每个 region 独立部署同一份 Knative Service。GSLB 按健康检查路由。任一 region 挂了，自动切走。

**主备架构（成本低）**

主 region 100% 流量，备 region `min-scale: 0`。主挂了 GSLB 切到备 region，备 region 冷启动接管。冷启动期间用户感知 30s 中断。

**配置示例（GSLB）**

```hcl
# Terraform with Route53
resource "aws_route53_health_check" "east" {
  fqdn              = "hello-east.example.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30
}

resource "aws_route53_record" "hello" {
  zone_id = var.zone_id
  name    = "hello"
  type    = "A"

  set_identifier = "east"
  failover_routing_policy { type = "PRIMARY" }
  health_check_id = aws_route53_health_check.east.id
  alias {
    name = "hello-east.example.com"
  }
}
```

#### 验证

模拟主 region 故障：

```bash
# 把主 region 的 ksvc 全部 scale 0 模拟挂
kubectl --context us-east-1 patch ksvc hello \
  --type merge -p '{"spec":{"template":{"metadata":{"annotations":{"autoscaling.knative.dev/max-scale":"0"}}}}}'

# 5 分钟内观察流量是否切到 us-west-2
curl http://hello.example.com/region
# 应该返回 us-west-2
```

> **口诀**：Knative 是**单集群组件**。跨 region 容灾必须在 DNS / 多集群 / 数据复制层做，**Knative 自身解决不了**。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 服务定义层
- [ ] 每个 Knative Service 明确写 `min-scale` 和 `max-scale`，**不依赖默认值**
- [ ] 核心高频服务设 `min-scale: 1`，避免冷启动用户感知
- [ ] `containerConcurrency` 经过压测确定（不是抄默认 100）
- [ ] `target` 设为 `containerConcurrency` 的 70~80%，留 buffer
- [ ] `timeoutSeconds` 与业务实际处理时长匹配（默认 300s 够吗？）
- [ ] resources.requests / limits 都填，不要让 Knative 用默认值

### 冷启动优化层
- [ ] JVM 应用必须考虑 GraalVM Native（5s → 200ms）
- [ ] Native 不可行的，关键服务设 `min-scale: 1`
- [ ] readinessProbe 检查"业务真正可服务"（含 DB 连接等关键依赖）
- [ ] 镜像大小 < 200MB（distroless / scratch / alpine）
- [ ] 节点 image pre-pull（DaemonSet 或 cluster-init）

### KEDA 层
- [ ] KEDA Operator 高可用部署（≥ 2 副本）
- [ ] ScaledObject 的 consumerGroup / queueName 与业务实际一致
- [ ] cooldownPeriod 与冷启动代价匹配（频繁 0↔1 切换有代价）
- [ ] minReplicaCount = 0 时确认能接受冷启动
- [ ] 业务侧独立监控（Burrow / kafka-exporter）作为 KEDA 的对照

### Eventing 层
- [ ] Trigger 必配 `delivery.retry > 0` 和 `deadLetterSink`
- [ ] Subscriber 实现幂等（基于 ce-id 去重）
- [ ] Broker（Kafka）topic 副本数 ≥ 3
- [ ] CloudEvent 关键字段（type / source / id）规范化
- [ ] 死信队列有人工处理 / 自动重投流程

### 流量与发布
- [ ] 灰度发布走 traffic split，不重启 Pod
- [ ] 100% 切换后等 5 分钟再宣布完成（长连接过渡期）
- [ ] 旧 Revision 保留至少 24h 便于回滚
- [ ] 跨 Service 调用走 K8s Service DNS，不依赖 Knative URL

### 网络与 Mesh
- [ ] Knative + Istio 选 Istio 做 networking layer 或显式禁用 sidecar 注入
- [ ] Activator 的 burst-capacity 按业务突发流量调
- [ ] Ingress 超时 ≥ Service timeoutSeconds

### 可观测性
- [ ] 日志走 DaemonSet 实时采集到中心存储
- [ ] 应用日志携带 trace_id（OpenTelemetry / W3C Trace Context）
- [ ] 关键 metric: Knative `revision_request_count` / `revision_request_latencies`
- [ ] KEDA metric: `keda_metrics_adapter_scaled_object_metric_value`
- [ ] 冷启动次数监控（每天 N 次冷启动是健康范围？）

### 容灾
- [ ] 多 region 部署 Knative + GSLB
- [ ] 状态层（DB / Kafka）跨 region 复制
- [ ] 故障切换演练每季度一次

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### CNCF Serverless 生态
- **OpenFunction**（CNCF 沙箱）：在 Knative 之上做 FaaS 抽象，支持事件驱动函数 + 同步函数
- **Fission**：另一个 K8s FaaS，预热 Pod 池实现毫秒级冷启动
- **Kubeless**（已 archive，但思路值得看）：用 ConfigMap 直接装函数代码

### Dapr 多语言运行时
- **Dapr**（Distributed Application Runtime）：sidecar 模式提供 state / pub-sub / binding 等能力
- 与 Knative 配合时能极大简化"事件驱动微服务"
- Dapr 的 Pluggable Component 让你扩展自己的状态存储 / 消息中间件

### 公有云 FaaS 对比
- **AWS Lambda**：Firecracker microVM，1ms 计费精度，layer / extension 模型
- **Cloudflare Workers**：V8 Isolate，全球 300+ PoP，Durable Objects 做有状态
- **Vercel Functions**：基于 AWS Lambda 做的开发者友好封装
- **Google Cloud Functions / Cloud Run**：Cloud Run 底层就是 Knative

### WebAssembly + Serverless
- **WasmEdge**（CNCF 沙箱）：Rust / C / Go 编译成 wasm，秒级冷启动 + 沙箱安全
- **Spin**（Fermyon）：开发者友好的 wasm 函数框架
- **Wasmer / Wasmtime**：通用 wasm 运行时
- 趋势：wasm 可能取代容器作为 Serverless 的标准运行时

### Edge Serverless
- **Knative on Edge**：把 Knative 跑在边缘节点（K3s）
- **Cloudflare Workers**：天然边缘
- **Crane / KCloud Edge**：腾讯的边缘 Serverless 方案
- 边缘场景的关键：极致冷启动 + 跨节点状态

### Stateful Serverless
- **Cloudflare Durable Objects**：单实例强一致的状态管理（actor 模型）
- **AWS Lambda + EFS**：函数读写共享文件系统
- **Knative + Volume**：挂载 PVC，但与扩缩冲突
- 趋势：Serverless 不再只能 stateless，actor / FaaS-with-state 是研究热点

### Crane（FinOps）
- **Crane**（腾讯）：K8s 资源优化平台，做 VPA / 智能扩缩 / 闲置资源回收
- 与 Knative 配合可以更精细地管"按量付费"

### Knative 进阶
- **AppCD**：Knative 的应用部署工具
- **TriggerMesh**：把 Knative Eventing 做成 Serverless Integration Platform
- 阅读 Knative `pkg/autoscaler/scaling/autoscaler.go` 源码理解 KPA 决策算法

---

## 7. 引用

- Knative 官方文档：[knative.dev](https://knative.dev/docs/)
- KEDA 官方文档：[keda.sh](https://keda.sh/docs/)
- CloudEvents 规范：[cloudevents.io](https://cloudevents.io/)
- *Berkeley View on Serverless Computing*（2019）：Serverless 学术综述
- *Firecracker: Lightweight Virtualization for Serverless Applications*（NSDI 2020）：AWS Lambda 底层
- *Cloudflare Workers: How V8 Isolates Power 10M Apps*（Cloudflare 博客）
- Knative 1.13 源码：[github.com/knative/serving](https://github.com/knative/serving)
- KEDA 2.13 源码：[github.com/kedacore/keda](https://github.com/kedacore/keda)
- 《Cloud Native Patterns》—— Cornelia Davis
- CNCF Serverless Landscape：[landscape.cncf.io/serverless](https://landscape.cncf.io/serverless)

---

> 配套：[Kubernetes 实战](/article/k8s/k8s-essentials) | [Operator 与 CRD](/article/k8s/operator) | [GraalVM Native Image](/article/jvm/graalvm)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你觉得这个深度对，告诉我下一篇该写哪个。
