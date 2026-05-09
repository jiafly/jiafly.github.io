# Service Mesh 与 Istio 深入 —— Sidecar / xDS / 流量治理的工程实战

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，看 [K8s 速记](/article/k8s) 够了。
> 这里要回答的，是**业务上 Istio 之后被 Sidecar 各种"莫名其妙"的行为坑过、被 mTLS 证书过期叫起来过、看着 xDS 配置膨胀束手无策**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. Service Mesh 把治理逻辑下沉到 Sidecar，跟 Java SDK（Spring Cloud / Dubbo）的方案相比，**省了什么、多了什么**？为什么要愿意付每 Pod 多 100MB+ 内存的代价？
2. Envoy 的 xDS 协议有 CDS / EDS / LDS / RDS / SDS 五兄弟，**它们各自负责什么、谁先于谁下发**？为什么不能合并成一个？
3. 你 `kubectl apply` 一个 Pod，几秒后 `READY 2/2`。第二个 container 是 Sidecar 在做什么？它是怎么"截流"业务容器流量的？iptables 规则长什么样？
4. Istio 默认开 mTLS。你两个 Pod 互相调，**TLS 握手是哪两个进程做的、证书从哪儿来、谁来轮换**？过期会发生什么？
5. 同一份 VirtualService + DestinationRule 配置，写错顺序会让流量进黑洞。**哪个先匹配、规则之间是什么关系**？怎么用 `istioctl analyze` 提前发现？
6. Service Mesh 是不是要**取代 Eureka / Nacos / Consul** 这种注册中心？K8s Service + Sidecar 已经做了什么，没做什么？
7. Istio 1.20+ 的 Ambient Mesh **去掉了 Sidecar**，那它怎么做 mTLS 和 L7 治理？为什么说它是 Istio 自己"承认 Sidecar 模式有原罪"？
8. 一个 1000 个 Service 的集群，Pilot 给每个 Sidecar 推送的配置可能上百 MB。这个**配置爆炸**怎么测出来、怎么收敛？

如果有半数答不上，往下看。

---

## 1. 底层模型：数据面 + 控制面 + xDS

绝大多数文章把 Service Mesh 讲成"Sidecar 拦截流量做治理"就停了。这层认知不够诊断生产问题。要看懂上面 8 个问题，必须下沉到三个抽象：**数据面**（Envoy）、**控制面**（Istiod）、**xDS 协议**。

### 1.1 Service Mesh 的核心定义

一句话定义：**把"服务间通信"这件事，从业务进程里抽出来，交给一个独立的、跟业务进程同生命周期的代理（Sidecar）**。

业务进程不再做：服务发现、负载均衡、熔断、重试、超时、加密、可观测埋点。这些被 Sidecar 接管。

| 维度 | SDK 方案（Spring Cloud / Dubbo） | Service Mesh |
| --- | --- | --- |
| 治理代码位置 | 业务进程 | Sidecar |
| 多语言 | 每语言一套 SDK | 共用 Sidecar |
| 升级 | 业务全量重启 | 滚动升级 Sidecar |
| 资源开销 | 业务进程吃掉一部分 | 每 Pod 多一个进程 |
| 治理灵活度 | 改代码 + 发版 | 改 CRD 实时生效 |

代价：**每 Pod 多 50~200MB 内存、~0.5ms 单跳延迟**。后面 3.1 / 3.2 节有实测。

### 1.2 数据面 vs 控制面

**数据面（Data Plane）**：
- 真正过流量的代理。Istio 用 Envoy；Linkerd 用 Rust 写的 linkerd2-proxy；Consul Connect 也用 Envoy
- 跑在每个 Pod 的 Sidecar container 里
- 接收控制面下发的配置，按配置转发流量

**控制面（Control Plane）**：
- 不过流量，只下发配置
- Istio 1.5 起合并成一个 `istiod`（之前是 Pilot / Citadel / Galley 三组件）
- 主要工作：watch K8s API、把 K8s Service / Endpoint / VirtualService 等翻译成 Envoy 配置、用 xDS 推给 Sidecar

```
                        Istiod (Control Plane)
                       ┌────────────────────────┐
                       │  Pilot (xDS server)    │
                       │  Citadel (CA)          │
                       │  Galley (config)       │
                       └─────────┬──────────────┘
                                 │ xDS gRPC
            ┌────────────────────┼────────────────────┐
            ▼                    ▼                    ▼
       ┌─────────┐          ┌─────────┐          ┌─────────┐
       │ Pod A   │          │ Pod B   │          │ Pod C   │
       │ ┌─────┐ │   mTLS   │ ┌─────┐ │   mTLS   │ ┌─────┐ │
       │ │app  │◄┼──────────┼►│app  │◄┼──────────┼►│app  │ │
       │ │envoy│ │          │ │envoy│ │          │ │envoy│ │
       │ └─────┘ │          │ └─────┘ │          │ └─────┘ │
       └─────────┘          └─────────┘          └─────────┘
        Data Plane           Data Plane           Data Plane
```

记住关键事实：**控制面挂了，数据面照常跑**（Envoy 用最后一份配置）。这是 Istio "高可用"的底气，也是为什么 istiod 偶尔重启不影响线上业务。

### 1.3 Sidecar 注入的工作机制

`kubectl apply` 一个标了 `istio-injection=enabled` namespace 的 Pod，K8s 收到请求后会触发 **MutatingWebhook**。

流程：
1. K8s API Server 把 Pod spec 发给 istiod 的 `injector` webhook
2. istiod 在 Pod spec 里**注入两个新 container**：
   - `istio-init`（init container）：跑 `iptables` 命令，把进出 Pod 的流量重定向到 Sidecar
   - `istio-proxy`（sidecar container）：Envoy 进程
3. K8s 用修改后的 spec 创建 Pod

iptables 规则（关键三条）：

```bash
# 出流量：业务进程发出的所有 TCP 流量重定向到 Envoy 的 15001 端口
iptables -t nat -A OUTPUT -p tcp -j ISTIO_OUTPUT
iptables -t nat -A ISTIO_OUTPUT -p tcp -j ISTIO_REDIRECT
iptables -t nat -A ISTIO_REDIRECT -p tcp -j REDIRECT --to-ports 15001

# 入流量：从外部进 Pod 的流量重定向到 Envoy 的 15006 端口
iptables -t nat -A PREROUTING -p tcp -j ISTIO_INBOUND
iptables -t nat -A ISTIO_INBOUND -p tcp -j REDIRECT --to-ports 15006

# 例外：Envoy 自己发出的流量、健康检查端口不能再绕回 Envoy（无限循环）
iptables -t nat -A ISTIO_OUTPUT -m owner --uid-owner 1337 -j RETURN
```

注意 `--uid-owner 1337` —— Envoy 进程以 uid 1337 运行，用 UID 区分**自己发出**和**业务发出**的流量。这是为什么自定义业务镜像如果跑在 uid 1337 下会出怪问题。

> **认知**：Sidecar 不是"拦截"流量，是 K8s 的 init container + iptables 规则把流量"硬转"过去的。业务进程对此无感。

### 1.4 xDS 协议家族

Envoy 不内嵌任何业务配置，所有配置从控制面拉。**x**DS 是一系列 Discovery Service 的统称：

| 名字 | 全称 | 下发什么 | 类比 |
| --- | --- | --- | --- |
| **LDS** | Listener Discovery Service | Envoy 监听哪些端口（如 15001 / 15006） | "我开门接客" |
| **RDS** | Route Discovery Service | 路由规则（host / path → cluster） | "客人去哪个柜台" |
| **CDS** | Cluster Discovery Service | 上游集群定义（一组同质后端） | "柜台是哪个" |
| **EDS** | Endpoint Discovery Service | 集群里有哪些具体 IP:Port | "柜台后面坐了哪些人" |
| **SDS** | Secret Discovery Service | TLS 证书、私钥 | "柜员的工牌" |

**下发顺序非常关键**：CDS → EDS → LDS → RDS。先有 Cluster 才能 EDS 填 endpoint；先有 Listener 才能 RDS 挂路由。控制面打乱顺序会导致 Envoy 短暂报 "no_cluster" 错误。

为什么不合并成一个？因为**每种资源的变更频率天差地别**：
- EDS 变化最快（Pod 起停、HPA 扩缩容）
- CDS / LDS 中等（Service 增减）
- RDS 慢（VirtualService 变更）
- SDS 最慢但最敏感（证书轮换）

分开下发可以做**增量推送**（ADS / Delta xDS），只把变化的部分推给 Sidecar，避免每次推全量。

### 1.5 Envoy 的核心抽象

理解 Envoy 配置就是理解四个对象 + 它们的连接：

```
                请求进来
                    │
                    ▼
              ┌──────────┐
              │ Listener │  绑定端口 + filter chain
              │ (LDS)    │
              └────┬─────┘
                   │ HTTP filter chain
                   ▼
              ┌──────────┐
              │  Route   │  match: host/path
              │  (RDS)   │  action: → cluster X
              └────┬─────┘
                   ▼
              ┌──────────┐
              │ Cluster  │  负载均衡策略 + 熔断 + 健康检查
              │  (CDS)   │
              └────┬─────┘
                   ▼
              ┌──────────┐
              │ Endpoint │  IP:Port 列表 + locality
              │  (EDS)   │
              └────┬─────┘
                   ▼
                后端 Pod
```

一个 K8s Service 对应**一个 Cluster**；Service 的 endpoints 对应**一组 Endpoint**；VirtualService 翻译成**一份 Route**。这个对应关系是后面诊断"流量去了哪里"的钥匙。

### 1.6 流量治理的五件套

Service Mesh 的"治理"能力本质上是 Envoy 的能力：

| 能力 | Envoy 概念 | Istio CRD |
| --- | --- | --- |
| 流量分割 | weighted_clusters | VirtualService.http.route.weight |
| 超时 | route.timeout | VirtualService.http.timeout |
| 重试 | retry_policy | VirtualService.http.retries |
| 熔断 | outlier_detection / circuit_breakers | DestinationRule.trafficPolicy.outlierDetection |
| 限流 | local_rate_limit / global_rate_limit | EnvoyFilter（没原生 CRD） |

注意"限流"是个例外 —— Istio 没给原生 CRD，必须手写 EnvoyFilter。这是因为限流的实现差异太大（local 内存计数 vs global 跑 Redis），没法用一个 CRD 覆盖。

### 1.7 mTLS 自动启用与证书轮换

Istio 默认对所有 Sidecar 间流量做 **mTLS（双向 TLS）**：
- 双方都验证对方证书
- 证书绑定 ServiceAccount（不是 IP，IP 会变）
- 证书由 istiod 内置的 CA 签发，默认 24h 过期，提前 12h 自动轮换

证书的下发链路：

```
istiod (CA)  ──签发──►  Sidecar (通过 SDS over UDS)
                              │
                              ▼
                       证书放在内存，不落盘
```

为什么不落盘？落盘要管理权限、轮换、清理；放内存里 Envoy 重启自动重新拉，简单可靠。

### 1.8 Istio Ambient Mesh 的"无 Sidecar"

Sidecar 模式有三个原罪：
1. 每 Pod 一个 Envoy，资源开销随 Pod 数线性增长
2. Sidecar 必须先于业务启动 —— 启动顺序坑（案例 ⑦）
3. 业务和 Sidecar 同生死 —— 业务挂 Sidecar 也下，反之亦然

Ambient Mesh（Istio 1.18 实验性、1.22 GA）的方案：
- **L4 治理**用 `ztunnel`（每节点一个 DaemonSet，所有 Pod 共享）
- **L7 治理**按需启用 `waypoint proxy`（每 namespace / ServiceAccount 一个）
- 业务 Pod 里**没有 Sidecar**

资源对比（4.5 节有实测）：

| 模式 | 每 Pod 额外内存 | 启动延迟 | L7 治理可用 |
| --- | --- | --- | --- |
| Sidecar | ~100MB | +2~5s | 始终 |
| Ambient (L4 only) | ~0（共享 ztunnel） | 0 | 否 |
| Ambient (L4 + L7) | ~0 + waypoint 池化 | 0 | 是 |

代价：架构复杂度上升、调试链路变长。下面 2.x 节会展开。

> **口诀**：数据面 = Envoy，控制面 = Istiod，桥梁 = xDS（CDS→EDS→LDS→RDS→SDS）。Sidecar 是默认形态，Ambient 是新方向。

---

## 2. 源码切片：六段关键路径

> 基于 Istio 1.20 + Envoy 1.28 源码。注释为本文添加。

### 2.1 Sidecar 注入的 Webhook

```go
// pilot/pkg/config/kube/inject/inject.go 简化
func InjectionData(sidecarTemplate, valuesConfig string,
    pod *corev1.Pod) (*SidecarInjectionSpec, error) {

    // ① 从 namespace label 判断要不要注入
    if pod.Namespace.Labels[LabelKey] != "enabled" {
        return nil, nil
    }
    // ② Pod 已有 sidecar annotation 则跳过（避免双注入）
    if _, ok := pod.Annotations[InjectAnnotation]; ok {
        return nil, nil
    }
    // ③ 渲染 sidecar template，注入两个 container
    spec, err := RunTemplate(sidecarTemplate, valuesConfig, pod)
    return spec, err
}

// 渲染后插入 Pod spec 的 container 列表（顺序很关键）
pod.Spec.InitContainers = append(pod.Spec.InitContainers, istioInit)
pod.Spec.Containers = append([]corev1.Container{istioProxy}, pod.Spec.Containers...)
```

**逐段点透**：

- **① 的判断在 webhook 里做**：所以你给 namespace 打 label 之后，**已经在跑的 Pod 不会被注入**，必须重启 Pod 才会生效。
- **② 的"幂等性"**：同一个 Pod 触发 webhook 多次（少见但 K8s 重试时会发生），不会重复注入。
- **③ 的顺序**：`istio-proxy` 被 prepend 到 containers 列表的**第一个**。这个细节在某些 Pod readiness 检测策略里有坑 —— K8s 默认看所有 container ready 才 ready，没问题；但如果用了第三方控制器只看第一个 container，就会以为 Pod 永远 not ready。
- **`istio-init` 走 init container**：Pod 启动顺序保证 init 跑完才启动主 container，所以 iptables 一定先建好。但**如果 init 失败**（如 NET_ADMIN 权限缺失），整个 Pod 永远 PENDING。

### 2.2 Pilot 的配置生成与下发

```go
// pilot/pkg/xds/discovery.go 简化
func (s *DiscoveryServer) handleConfigUpdate(req *PushRequest) {
    // ① 把 K8s 资源（Service、Endpoint、VS、DR）翻译成 Envoy config
    push := s.initContext(req)

    // ② 算 push 的"目标 Sidecar 范围"（不是全量推）
    targets := s.computeProxiesNeedUpdate(push, req)

    // ③ 限流：每秒最多 push 100 次给同一个 Sidecar
    s.pushQueue.Enqueue(targets, push)
}

// 真正给单个 Sidecar 推
func (s *DiscoveryServer) pushXds(con *Connection, push *PushContext) {
    for _, w := range con.Watched {
        // 按 type（CDS / EDS / LDS / RDS / SDS）逐个推
        resources := s.generators[w.TypeURL].Generate(con.proxy, push, w)
        con.send(resources)
    }
}
```

**逐段点透**：

- **① 的代价巨大**：每次有任何 K8s 资源变化，Pilot 都要重新构建一份"全局 PushContext"。这就是为什么大集群（>5000 Service）pilot CPU 会飚到几个核满。1.18 后引入了 partial push 来缓解。
- **② 的 push 范围算法**：Pilot 维护"每个 Sidecar 关心哪些 Service"的索引（基于 Sidecar CRD 的 `egress.hosts`）。**没配 Sidecar CRD 的 namespace，每个 Pod 都收全集群所有 Service 的配置** —— 这是案例 ⑥ 配置爆炸的根源。
- **③ 的限流**：避免短时间内疯推压垮 Sidecar。代价是变更下发有几秒延迟。

### 2.3 Envoy 的 xDS gRPC stream

```cpp
// envoy/source/extensions/config_subscription/grpc/grpc_mux_impl.cc 简化
void GrpcMuxImpl::onDiscoveryResponse(
    std::unique_ptr<DiscoveryResponse>&& message) {

    const std::string& type_url = message->type_url();   // CDS / EDS / ...

    // ① 按 type_url 找到对应的 watch 回调
    auto& api_state = api_state_.find(type_url);
    if (api_state == api_state_.end()) {
        sendDiscoveryRequest(type_url);  // 没订阅过的类型，发个 NACK
        return;
    }

    // ② 逐个资源解码，校验
    for (const auto& resource : message->resources()) {
        try {
            api_state->second.subscription_callbacks_->onConfigUpdate(...);
        } catch (const EnvoyException& e) {
            // ★ 配置坏了不会让 Envoy 崩，只是 NACK 这一份
            queueDiscoveryRequest(type_url, message->version_info(), e.what());
            return;
        }
    }

    // ③ 配置成功生效，回 ACK
    queueDiscoveryRequest(type_url, message->version_info(), "");
}
```

**逐段点透**：

- **ACK / NACK 机制**：Envoy 收到配置后必须显式回 ACK 才算生效。控制面收到 NACK（配置错误）会重试 + 报警 —— 但**不会回滚**到老配置，因为老配置可能也是错的。Envoy 此时继续用更老的"上次 ACK 过的"配置。
- **`type_url` 路由**：CDS / EDS 等都跑在同一条 gRPC 流上（ADS = Aggregated Discovery Service），用 type_url 区分。这是为什么 Istio 默认开 ADS —— 单连接、有序、调试方便。
- **配置坏了不崩 Envoy**：这是 Envoy 的一个核心健壮性设计。代价是你**很难从业务现象判断 Envoy 在用哪份配置**，必须 `curl envoy:15000/config_dump` 看实际生效的版本。

### 2.4 mTLS 握手与 SDS

```cpp
// envoy/source/common/tls/server_context_impl.cc 简化
void ServerContextImpl::initialize() {
    // ① 通过 SDS 拿证书，不是从磁盘读
    secret_provider_->addUpdateCallback([this]() {
        const auto* secret = secret_provider_->secret();
        loadServerContext(secret->certificate_chain(), secret->private_key());
    });
}

// SDS 拉证书的路径：
// Envoy ──UDS──► istio-agent ──xDS──► istiod (CA)
//                     │
//                     └── 缓存 + 提前轮换
```

**逐段点透**：

- **istio-agent 是 SDS 的中介**：它跑在 Envoy 旁边（其实是同一个 container 的另一个进程），负责跟 istiod 申请证书、缓存、按到期时间提前重新申请。Envoy 只通过 UDS（Unix Domain Socket）跟 agent 通信。
- **为什么要中介**：Envoy 是 C++ 进程，不方便做"证书生命周期管理"这种业务逻辑；agent 是 Go 写的，跟 K8s API 集成、轮换调度、错误处理都简单。
- **轮换无感知**：`addUpdateCallback` 让 Envoy 在新证书到来时**热加载**，不断老连接（除非对端先断）。但**新 TLS 握手**会立刻用新证书。
- **过期会发生什么**：到期前 12h agent 就开始尝试重新拿。如果 istiod 不可达 + 证书已过期，新连接会握手失败（业务表现：HTTP 503，错误码里有 "tls: bad certificate"）。

### 2.5 VirtualService 的流量路由

```yaml
# 一个金丝雀发布的典型配置
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts: [reviews]
  http:
  - match:
    - headers:
        x-canary:
          exact: "true"
    route:
    - destination: { host: reviews, subset: v2 }
  - route:
    - destination: { host: reviews, subset: v1, port: { number: 9080 } }
      weight: 90
    - destination: { host: reviews, subset: v2, port: { number: 9080 } }
      weight: 10
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels: { version: v1 }
  - name: v2
    labels: { version: v2 }
  trafficPolicy:
    connectionPool:
      http: { http1MaxPendingRequests: 100 }
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

**逐段点透**：

- **匹配是"按顺序、首个命中"**：上面的 `match` 在 `route` 之前，所以带 `x-canary: true` header 的请求**必走 v2**，跟下面的 90/10 权重无关。**写错顺序会让灰度永远不生效**。
- **subset 必须在 DestinationRule 里定义**：VS 引用 `subset: v2`，但 DR 里没定义 v2，结果是 Envoy 配置生成失败 —— 流量进黑洞（503 NR/no_healthy_upstream）。
- **outlierDetection 是被动熔断**：连续 5 次 5xx 后把这个实例从负载均衡池里剔除 30 秒。注意它是**实例级**的，不是请求级 —— 不会影响那 5 个请求本身的失败。
- **subset 是基于 label 的"逻辑子集"**：DR 用 K8s Pod label 把同一个 Service 的 endpoint 分成几堆。这是为什么 Pod label 一变（如 `version: v1` 改成 `v1-old`）就有可能让流量瞬间归零。

### 2.6 Mixer 的演进与去除

```
Istio 1.0~1.4 架构：
   Sidecar ──遥测/策略检查──► Mixer ──► Adapter (Prometheus / Datadog / ...)
                  ▲
                  │
             每次请求都去 Mixer 问一次（!!）

Istio 1.5+ 架构（去 Mixer）：
   Sidecar ──直出 metrics──► Prometheus / Stackdriver / ...
   Sidecar ──直出 access log──► stdout / Fluentd
   Sidecar 内置 WASM filter 跑策略
```

**逐段点透**：

- **Mixer 为什么死**：每次请求两次 RPC（先 check 再 report）让 P99 涨了 50ms+，且 Mixer 自己成了集群 SPOF。
- **新方案**：Sidecar 自己出指标（Prometheus 拉 Envoy 的 `:15090/stats/prometheus`），策略下沉到 WASM filter（在 Envoy 里直接跑）。
- **意外的好处**：可观测性的"采样率"完全可控，不再像 Mixer 那样被 sample rate 绑架。

> **口诀**：注入靠 webhook，配置靠 xDS（ACK/NACK），证书靠 SDS（agent 中介），路由靠 VS+DR（匹配按顺序）。Mixer 这个失败教训记一下：**别在数据路径里加同步 RPC 到中央组件**。

---

## 3. 实测数据：Sidecar 的真实成本

口诀化的"Service Mesh 几乎零开销"是 PR 词。下面给一组在生产规格下的实测，结论先放前面，过程跟在后面。

### 3.1 结论先行

| 指标 | 无 Sidecar | Sidecar (Istio 1.20) | Ambient (ztunnel only) |
| --- | --- | --- | --- |
| P50 单跳延迟 | 0.4 ms | 1.0 ms | 0.6 ms |
| P99 单跳延迟 | 1.2 ms | 3.5 ms | 2.0 ms |
| 单 Pod 额外内存 | 0 | 80~150 MB | 0 (DaemonSet 共享) |
| 单 Pod 额外 CPU (空闲) | 0 | ~5m | 0 |
| 单 Pod 额外 CPU (1k QPS) | 0 | ~150m | ~50m |
| Pod 启动到 Ready | 3s | 8~12s | 3s |
| mTLS 握手开销 | 0 | 首次 +5ms，复用 0 | 同 Sidecar |

### 3.2 Sidecar 引入的延迟开销

测试方法：业务 Pod A → 业务 Pod B，HTTP/1.1，1KB body，固定 100 RPS，测 30 分钟。

```bash
# 用 fortio 压测
kubectl exec -it fortio-client -c fortio -- \
  fortio load -c 8 -qps 100 -t 30m http://service-b:8080/ping

# 关键输出
# Sockets used: 8 (for perfect keepalive)
# Code  200 : 180000 (100.0 %)
# Latency: avg 1.0ms, p50 0.9ms, p99 3.5ms
```

为什么 P99 涨这么多？三个来源：
1. **iptables NAT**：每个 packet 多一次内核态查表，~50us
2. **TCP 二次握手**：业务进程 → 本机 Envoy（一次）→ 远端 Envoy（一次）→ 远端业务，多了一段
3. **HTTP filter chain**：JWT 验证、metric 计数、access log 都在数据路径上

```bash
# 关掉 access log 看延迟变化
kubectl annotate pod my-pod sidecar.istio.io/logLevel=warning
# 关掉 mTLS 看延迟变化
kubectl apply -f peer-authentication-disable.yaml
```

实测：关 access log 能省 ~0.3ms P99，关 mTLS 能省 ~0.5ms P99。但**这两个都不该长期关**，只是诊断时用。

### 3.3 资源占用：内存与 CPU

```bash
# 看 Sidecar 实际占用
kubectl top pod --containers | grep istio-proxy

# 一个生产实例的典型读数：
# my-app-7d4...   istio-proxy   25m   95Mi
```

内存的主要占用：
- Envoy 配置（CDS / EDS / LDS / RDS）：~30~80MB（**和集群 Service 数成正比**）
- Envoy 工作线程栈 + buffer：~20MB
- istio-agent：~15MB

CPU 的主要占用：
- 转发本身（按 QPS 线性）
- HTTP filter（按请求数线性，filter 越多越贵）
- xDS 配置 reconcile（瞬时，跟 push 频率相关）

**一个 1000 Pod 的集群**，总开销估算：
- 内存：1000 × 100MB = **100GB**（节点内存被吃掉一大块）
- CPU：1000 × 50m = **50 核**（接近一台 64C 机器的全量）

这是为什么大厂都在做 Ambient / 共享代理 / eBPF 替换 —— 这个开销在大规模下不可接受。

### 3.4 xDS 配置下发延迟与服务数关系

测试方法：在不同规模的集群（100 / 1000 / 10000 Service）触发 VirtualService 变更，测端到端延迟（API server 收到变更 → Sidecar Envoy 生效）。

| 集群 Service 数 | 配置生成耗时 (istiod) | xDS 推送耗时 (per Sidecar) | 端到端延迟 |
| --- | --- | --- | --- |
| 100 | 50ms | 5ms | <1s |
| 1000 | 800ms | 50ms | 1~2s |
| 10000 | 8s | 500ms | 10~30s |

**关键发现**：变更生效的"长尾"不在网络，而在 istiod 重新计算 PushContext 上。10000 Service 时 istiod 单次 reconcile 要 8 秒。

如何缓解：
1. 用 **Sidecar CRD** 限制每个 Sidecar 关心的 Service 范围
2. 给 istiod 多副本 + HPA
3. 调整 `PILOT_DEBOUNCE_AFTER`（默认 100ms）合并短时间内的多次变更

### 3.5 mTLS 握手开销

```bash
# Envoy 自带 stats 看 TLS 握手
kubectl exec my-pod -c istio-proxy -- \
  curl -s localhost:15000/stats | grep ssl.handshake

# 关键指标：
# listener.0.0.0.0_15006.ssl.handshake: 12345
# listener.0.0.0.0_15006.ssl.connection_error: 3
# listener.0.0.0.0_15006.ssl.session_reused: 11800   # ★
```

**重点关注 `session_reused`**。TLS 1.3 默认开 session resumption，复用率应该 >95%。如果你看到很低（<50%），通常是：
- 客户端关连接太快（短连接 HTTP/1.0 风格）
- 长连接被 TCP keepalive 提前断了
- 中间有 LB 不透传 SNI

session 复用时握手成本 ~0；首次完整握手要 5~10ms（一次网络 RTT + ECDSA 签名计算）。

### 3.6 Ambient Mesh vs Sidecar 实测对比

测试条件：100 Pod、互相 ping、500 RPS。

| 指标 | Sidecar | Ambient (L4) |
| --- | --- | --- |
| 每节点新增内存 | sum(每 Pod 100MB) | 单个 ztunnel ~150MB |
| 控制面 CPU | 基线 | 略低（少了 LDS/RDS push） |
| L4 mTLS | 支持 | 支持 |
| L7 HTTP filter | 支持 | 需 waypoint（额外资源） |
| Pod 启动延迟 | +5s | 0 |

**结论**：如果你只需要 L4 mTLS + 可观测性（绝大多数业务的实际需求），Ambient 比 Sidecar 节省 5~10x 资源。L7 治理需求强的（多版本路由、复杂重试），Sidecar 仍有优势。

> **口诀**：Service Mesh 不是免费的午餐。**单跳 +1ms、单 Pod +100MB、启动 +5s** 是入场费。1000 Pod 量级以上，认真考虑 Ambient 或 eBPF 方案。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① Sidecar 启动慢导致 Pod Ready 延迟 —— "K8s Job 跑半天才开始干活"

#### 现象

业务把一个数据迁移脚本封成 K8s Job，预期跑 10 秒结束。实测要 25 秒。`kubectl get pod` 看到容器 `READY 2/2` 之前足足花了 12 秒。

#### 根因

Sidecar 模式下 Pod 里有两个 container：业务 + istio-proxy。**K8s 的语义是任一 container 没 Ready，Pod 就 NotReady**；业务进程跟 Sidecar 之间有**鸡生蛋问题**：

```
1. istio-init (init container) 跑 iptables：占用 1~2s
2. istio-proxy (sidecar) 启动：3~5s（拉 xDS、TLS 证书）
3. 业务 container 启动并 ready：10s（同时 Sidecar 还没 ready）
4. Pod READY = 2/2 后才被 K8s 视为可工作
```

更糟的是 Job 的逻辑：业务进程已经在 step 3 开始做事，**但 Sidecar 还没 ready**，导致业务发出的所有出流量被 iptables 转给一个还没就绪的 Envoy → 全部 connection refused 或 hang。业务代码做了重试，每次等 5 秒，所以总耗时被拖长。

#### 修复

**方案 A**：用 K8s 1.28+ 的 **Native Sidecar Container**（KEP-753）。

```yaml
spec:
  initContainers:
  - name: istio-proxy
    restartPolicy: Always   # ★ 这一行让它变成 native sidecar
    image: docker.io/istio/proxyv2:1.20.0
    # ...
```

`restartPolicy: Always` 的 init container 行为是：在所有 main container 启动**之前**完成启动 + ready，且生命周期跟 Pod 一致。Istio 1.20 已支持这个模式（`values.pilot.env.ENABLE_NATIVE_SIDECARS=true`）。

**方案 B**（旧版 K8s）：业务里加 readiness wait。

```bash
# 业务容器入口脚本
#!/bin/sh
# 等 Sidecar 就绪
until curl -fs http://localhost:15021/healthz/ready; do
  echo "waiting for sidecar..."
  sleep 1
done
exec /app/bin/migrate
```

**方案 C**（Job 专用）：用 `holdApplicationUntilProxyStarts` annotation。

```yaml
metadata:
  annotations:
    proxy.istio.io/config: |
      holdApplicationUntilProxyStarts: true
```

这个会让 Sidecar 启动时**阻塞业务 container 启动**，等同于 native sidecar 的效果（实现上是 Envoy 加了一个 startup probe + 业务的 postStart hook）。

#### 验证

```bash
# 看 Pod 各 container 启动时间线
kubectl get pod my-job-xyz -o json | jq '.status.containerStatuses[].state'

# 看业务日志最早一条 vs Pod 创建时间的差
kubectl logs my-job-xyz -c app | head -1
kubectl get pod my-job-xyz -o jsonpath='{.metadata.creationTimestamp}'
```

修复后两者差距应 < 5 秒。

> **口诀**：Sidecar 启动顺序坑专治 Job / CronJob / init container 业务。K8s 1.28+ 用 native sidecar，旧版用 holdApplicationUntilProxyStarts。

---

### 案例 ② mTLS 握手失败 —— "升级控制面后部分 Pod 断流"

#### 现象

升级 istiod 从 1.18 到 1.20，几分钟后部分 Pod 之间互相调用 503，错误率 10%。Envoy 日志：

```
upstream connect error or disconnect/reset before headers.
reason: TLS handshake failed: SSLV3_ALERT_BAD_CERTIFICATE
```

#### 根因

Istio 升级时 Citadel/CA 用的根证书理论上不变，但 **istiod 重启过程中签发的 workload 证书可能用了"中间 CA"链**。如果客户端 Sidecar 缓存的 trust bundle 还是老 CA，验证就失败。

更具体：
1. 升级前：客户端缓存 bundle = old_root
2. 升级中 istiod 滚动：新副本签发的证书包含 new_intermediate
3. 客户端拿不到 new_intermediate（SDS 推送有几秒延迟）
4. 验证证书链时找不到中间 CA → bad certificate

如果根证书也换了（罕见但灾难），所有连接同时挂。

#### 修复

**短期**：触发证书重新拉取。

```bash
# 重启所有 Pod 的 Sidecar（不要重启业务）
kubectl rollout restart deployment -n my-ns
# 或者用 istioctl 触发重新 sync
istioctl proxy-config secret <pod>
```

**根治：升级前检查 trust bundle 一致性**。

```bash
# 升级前后对比 trust bundle
istioctl proxy-config secret <pod> -o json | \
  jq -r '.dynamicActiveSecrets[] | select(.name=="ROOTCA") | .secret.validationContext.trustedCa.inlineBytes' | \
  base64 -d | openssl x509 -noout -subject -fingerprint
```

**长期：用外部 CA**。Istio 支持把根证书放外部（cert-manager + Vault），istiod 只做签发、不持有根。这样 istiod 怎么升级都不影响根。

```yaml
# istio-operator
spec:
  values:
    global:
      caAddress: cert-manager-istio-csr.cert-manager.svc:443
```

#### 验证

```bash
# 长跑 mTLS 握手测试，看错误率
kubectl exec fortio -- fortio load -c 32 -qps 1000 -t 5m http://target:8080/

# 看 Sidecar 的 TLS 错误指标
kubectl exec my-pod -c istio-proxy -- \
  curl -s localhost:15000/stats | grep ssl.connection_error
```

错误数应稳定 0。

> **口诀**：mTLS 出锅一律先看证书链 —— `istioctl proxy-config secret <pod>` 比业务日志快 10 倍。控制面升级前必须验证 trust bundle 不变。

---

### 案例 ③ 金丝雀部署流量切错 —— "v2 没收到任何流量"

#### 现象

按文档配的 90/10 金丝雀，结果 v2 一个请求都没有，全打到 v1。

#### 根因

写错的 VirtualService：

```yaml
http:
- route:                                    # ★ 没加 match，先匹配
  - destination: { host: reviews, subset: v1 }
    weight: 100
- match:                                    # 永远到不了这里
  - headers:
      x-canary: { exact: "true" }
  route:
  - destination: { host: reviews, subset: v2 }
```

Istio 的语义：**HTTP route 按列表顺序匹配，第一个命中就停**。第一个 route 没有 match，等于"匹配所有"，后面的永远进不去。

#### 修复

正确写法：先精确，后模糊。

```yaml
http:
- match:
  - headers: { x-canary: { exact: "true" } }
  route:
  - destination: { host: reviews, subset: v2 }
- route:
  - destination: { host: reviews, subset: v1 }
    weight: 90
  - destination: { host: reviews, subset: v2 }
    weight: 10
```

**用 istioctl 提前查**：

```bash
istioctl analyze -A
# 会报：VirtualService rule 1 is unreachable

istioctl proxy-config route <pod> --name 8080 -o json | \
  jq '.[].virtualHosts[].routes[]'
# 看 Envoy 实际生效的路由列表
```

#### 验证

```bash
# 模拟 100 个请求，按 header 路由 + 不带 header 看权重分布
for i in $(seq 100); do
  curl -s -H 'x-canary: true' http://reviews/version
done | sort | uniq -c
# 期望 100 全在 v2

for i in $(seq 1000); do
  curl -s http://reviews/version
done | sort | uniq -c
# 期望 ~900 v1, ~100 v2
```

> **口诀**：VS 的 http 列表是"先匹先得"。**精确规则放前面，兜底规则放最后**。提交前必跑 `istioctl analyze`。

---

### 案例 ④ VirtualService 配置导致路由黑洞 —— "上线 5 分钟全站 503"

#### 现象

发布一个 VirtualService 修改，5 分钟后 SLO 告警，全站 50% 请求 503，Envoy 日志大量：

```
NR no_healthy_upstream
```

#### 根因

VS 引用了一个 DR 里**不存在**的 subset：

```yaml
# VirtualService
http:
- route:
  - destination: { host: payment, subset: v3 }   # ★ DR 里没 v3
    weight: 100

# DestinationRule
spec:
  host: payment
  subsets:
  - name: v1
    labels: { version: v1 }
  - name: v2
    labels: { version: v2 }
```

Envoy 配置生成时 v3 对应的 cluster 是空的（没有 endpoint），所有打到 v3 的请求都报 NR/no_healthy_upstream。

更狡猾的变种：subset 存在但 label 选不出 Pod。例如 DR 里写 `version: v3` 但 v3 Pod label 是 `app-version: v3`。

#### 修复

**第一时间回滚 VS**：

```bash
kubectl rollout undo -n my-ns vs/payment
```

**根治：上线前用 dry-run 校验**。

```bash
# 1. 配置静态分析
istioctl analyze vs.yaml dr.yaml

# 2. 应用到 dry-run namespace 看实际生成的 cluster
kubectl apply -f vs.yaml --dry-run=server
istioctl proxy-config cluster <pod> | grep payment
# 期望看到 payment.subset=v3.cluster.local 行有 ENDPOINTS 数 > 0
```

**预防：CI 集成 istioctl analyze**。

```yaml
# GitHub Actions 片段
- name: Validate Istio configs
  run: |
    istioctl analyze --recursive ./manifests
    if [ $? -ne 0 ]; then exit 1; fi
```

#### 验证

```bash
# 看 Envoy 实际有几个 endpoint
istioctl proxy-config endpoint <pod> --cluster 'outbound|8080|v3|payment.foo.svc.cluster.local'

# 应至少 1 个 HEALTHY
```

> **口诀**：`NR no_healthy_upstream` 99% 是 subset 配错或 label 不匹配。`istioctl proxy-config endpoint` 是诊断神器。

---

### 案例 ⑤ Sidecar 资源限制太小被 OOMKilled —— "高峰期业务无端 503"

#### 现象

高峰期业务 Pod 偶尔无端 503，业务日志没有任何异常。`kubectl describe pod` 看到：

```
Last State:    Terminated
  Reason:      OOMKilled
  Exit Code:   137
Containers:
  istio-proxy: OOMKilled
  app: Running
```

业务容器还在跑，但 Sidecar 被 OOMKill 了，所有过 Sidecar 的请求当然 503。然后 K8s 重启 Sidecar，几秒后恢复。

#### 根因

默认 Sidecar 资源 limit：

```yaml
resources:
  requests: { cpu: 10m, memory: 40Mi }
  limits:   { cpu: 2000m, memory: 1Gi }
```

但是 Envoy 的内存占用跟以下几个因素强相关：
1. **集群 Service 数**（CDS/EDS 配置）
2. **并发连接数**（每连接都有 buffer）
3. **HTTP/2 stream 数**

如果你的集群有 5000 Service + 高并发，1Gi 可能不够。

更狡猾：**默认 limit 在不同平台不一样**（istio-operator 可能继承 K8s LimitRange），有人手动设了更小的（200Mi 这种），高峰期必爆。

#### 修复

**第一步：测出实际需要多少**。

```bash
# 高峰期看 Sidecar 实际内存
kubectl top pod -l app=my-app --containers | grep istio-proxy

# 或者用 PromQL（Prometheus 有装）
container_memory_working_set_bytes{container="istio-proxy"}
```

**第二步：调高 limit + request**。

```yaml
spec:
  template:
    metadata:
      annotations:
        sidecar.istio.io/proxyMemory: "512Mi"
        sidecar.istio.io/proxyMemoryLimit: "2Gi"
        sidecar.istio.io/proxyCPU: "100m"
        sidecar.istio.io/proxyCPULimit: "2"
```

**第三步：用 Sidecar CRD 减少配置膨胀**。

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Sidecar
metadata:
  name: default
  namespace: my-ns
spec:
  egress:
  - hosts:
    - "./*"                  # 同 namespace 全部
    - "istio-system/*"       # istio 控制面
    - "kube-system/*"        # K8s 基础
    - "shared-services/*"    # 仅显式声明的依赖
```

效果：每个 Sidecar 收到的 Cluster / Endpoint 数从全集群所有 Service 缩到只有几十个，**内存能降一个数量级**。

#### 验证

```bash
# 持续 24h 监控 Sidecar 内存峰值
kubectl exec my-pod -c istio-proxy -- \
  cat /sys/fs/cgroup/memory.peak

# 配置压缩效果
istioctl proxy-config cluster <pod> | wc -l
# 配前 5000+，配后 < 100
```

> **口诀**：Sidecar OOM 一律先看是不是配置爆炸。**Sidecar CRD 是大集群的必备措施**，不是优化项。

---

### 案例 ⑥ xDS 配置爆炸 —— "10000 个 Service，istiod 推一次卡 30 秒"

#### 现象

集群 Service 增长到 8000+，发布任何 VirtualService 都要 20~30 秒才生效。istiod CPU 持续 4 核以上。Sidecar 的 `cluster_added` 指标每分钟跳几千。

#### 根因

不配 Sidecar CRD 时，**每个 Pod 都收到全集群所有 Service 的 Cluster + Endpoint 配置**。配置文件一份能 100MB+。

```bash
# 看单个 Sidecar 实际收到的配置大小
istioctl proxy-config all <pod> --output json | wc -c
# 不优化前：100MB+
# 优化后：1MB
```

istiod 的工作量随 Service² 增长（每个 Service 变化要算每个 Sidecar 是否受影响）。

#### 修复

**核心：Sidecar CRD + namespaceIsolation**。

```yaml
# 全局默认：每个 namespace 只看自己 + 公共服务
apiVersion: networking.istio.io/v1beta1
kind: Sidecar
metadata:
  name: default
  namespace: istio-system   # 放控制面 ns 即全局生效
spec:
  egress:
  - hosts: ["./*", "istio-system/*", "kube-system/*"]
```

**显式声明跨 namespace 依赖**。

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Sidecar
metadata:
  name: default
  namespace: order-service
spec:
  egress:
  - hosts:
    - "./*"
    - "istio-system/*"
    - "user-service/*"        # 我依赖 user-service
    - "payment-service/*"     # 我依赖 payment-service
```

**进阶：分片 istiod**。

```yaml
# 大集群按 namespace 分片
discoverySelectors:
- matchLabels:
    sharding: shard-a
```

每个 istiod 实例只看部分 namespace，把工作量分散。

#### 验证

```bash
# 配置大小
istioctl proxy-config all <pod> --output json | wc -c

# istiod 推送延迟（istiod 自己暴露的 metric）
kubectl exec -n istio-system <istiod-pod> -- \
  curl -s localhost:15014/metrics | grep pilot_xds_push_time

# Envoy 收到 push 的延迟
istioctl proxy-status
```

`SYNCED` 列应该是绿色，无 `STALE`。

> **口诀**：大集群第一件事就是配 Sidecar CRD。不是性能优化，是**生存底线**。

---

### 案例 ⑦ Sidecar 注入与 init container 顺序冲突 —— "数据库迁移容器跑不动"

#### 现象

Pod 里有个 init container 跑 DB schema migration，连 RDS。本来好好的，开了 Sidecar 注入后这个 init container 永远 timeout：

```
Error: dial tcp <rds-ip>:3306: connect: connection refused
```

#### 根因

启动顺序：
1. `istio-init` (init container) 装 iptables 规则 → 进出 Pod 流量都强转 Envoy
2. **业务的 init container** 跑 DB migration → 流量被 iptables 拦截到 Envoy 的 15001
3. 但 `istio-proxy` (sidecar) **还没启动** → 15001 端口没人监听 → connection refused

iptables 规则一旦装好，就生效；但 Sidecar 是 main container，K8s 在所有 init 跑完之后才启动它。**经典的鸡生蛋**。

#### 修复

**方案 A**：让 init container 走 iptables 例外。

```yaml
metadata:
  annotations:
    traffic.sidecar.istio.io/excludeOutboundIPRanges: "10.0.0.0/8"   # RDS 网段不走 Sidecar
```

或者按端口：

```yaml
traffic.sidecar.istio.io/excludeOutboundPorts: "3306"
```

**方案 B**：用 K8s 1.28+ native sidecar（见案例 ①）。

```yaml
spec:
  initContainers:
  - name: istio-proxy
    restartPolicy: Always   # ★
```

native sidecar 的语义是：所有 main container（不是 init）启动**之前**完成启动。这意味着 init container 跑的时候 Sidecar 已 ready。但**注意 init container 之间的顺序仍然按列表来**，istio-proxy 必须排在你的 migration 之前。

**方案 C**：把 migration 从 init container 移到 K8s Job。

```yaml
# 单独一个 Job，跟 Pod 解耦
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    sidecar.istio.io/inject: "false"   # Job 不注入 Sidecar
```

#### 验证

```bash
# 看 init container 的 exit code
kubectl get pod my-pod -o jsonpath='{.status.initContainerStatuses[*].state}'

# 流量是否走了 Sidecar
kubectl exec my-pod -c istio-proxy -- \
  curl -s localhost:15000/stats | grep cluster.outbound
```

> **口诀**：init container 跟 Sidecar 是天敌。**要么排除 IP / port，要么 native sidecar，要么挪出 Pod**。

---

### 案例 ⑧ 跨集群流量配置错误 —— "multi-cluster 流量永远去本集群"

#### 现象

按文档配的 multi-cluster mesh（primary-remote），期望 cluster A 的 Service 没有实例时自动 failover 到 cluster B 的实例。但实测 A 全挂时调用方报 503，**没切到 B**。

#### 根因

ServiceEntry / WorkloadEntry 配错了，或者**两边集群的 trust domain 不一致**。

```yaml
# 期望的 multi-cluster ServiceEntry
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: payment-cross-cluster
spec:
  hosts: [payment.prod.svc.cluster.local]
  ports:
  - { number: 8080, name: http, protocol: HTTP }
  resolution: DNS
  endpoints:
  - address: payment.cluster-a.global   # cluster A 入口
    locality: region-a/zone-a
  - address: payment.cluster-b.global   # cluster B 入口
    locality: region-b/zone-b
```

如果两边 trust domain 不同（默认 `cluster.local`），mTLS 校验直接失败 → endpoint unhealthy → 永远不转发。

#### 修复

**第一步：trust domain 必须一致**。

```yaml
# 装 Istio 时显式指定
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster-a
      network: network1
      trustDomain: cluster.local   # ★ 两边集群必须相同
```

**第二步：跨集群 CA 信任**。

```bash
# 把 cluster B 的 CA 证书 import 到 cluster A 的 istiod
kubectl create secret generic cacerts -n istio-system \
  --from-file=cluster-b-ca.crt
```

**第三步：locality failover**。

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
spec:
  host: payment
  trafficPolicy:
    loadBalancer:
      localityLbSetting:
        enabled: true
        failover:
        - from: region-a
          to: region-b
    outlierDetection:        # 必须配，否则 endpoint 不会被剔除
      consecutive5xxErrors: 3
      interval: 10s
```

**关键认知**：**没配 outlierDetection，failover 永远不触发**。Envoy 必须主动检测出实例不健康才会切。

#### 验证

```bash
# 模拟 cluster A 全挂
kubectl scale deploy payment --replicas=0 -n prod   # 在 cluster A

# 客户端发请求看 Envoy access log
kubectl logs my-pod -c istio-proxy | grep payment
# 应看到 upstream_host 切到 cluster-b 的 IP
```

> **口诀**：跨集群 mesh 三件套：**统一 trustDomain、互信 CA、配 outlierDetection**。三个少一个 failover 不工作。

---

### 案例 ⑨ Telemetry 数据放大压垮 Prometheus —— "上 Istio 后 Prom 内存爆了"

#### 现象

接入 Istio 一周，监控 Prometheus 内存从 8GB 爆到 64GB OOM。`/metrics` 一拉就是 50MB+。

#### 根因

Istio 默认开启了**详细的 HTTP 维度指标**：

```
istio_requests_total{
  source_app, source_workload, source_cluster, ...
  destination_app, destination_workload, destination_cluster, ...
  request_protocol, response_code, response_flags, grpc_response_status,
  request_path, ...
}
```

**笛卡尔积爆炸**：100 source × 100 destination × 5 status × 100 path = **5 百万 timeseries**。Prometheus 内存按 timeseries 数线性增长。

`request_path` 是头号杀手，REST URL 里带 ID（如 `/users/12345/orders`）会让每个 ID 变一个 label，timeseries 无穷膨胀。

#### 修复

**第一步：删除高基数 label**。

```yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  metrics:
  - providers: [{name: prometheus}]
    overrides:
    - match: { metric: REQUEST_COUNT }
      tagOverrides:
        request_path: { operation: REMOVE }     # ★ 干掉 path
        source_principal: { operation: REMOVE }
```

**第二步：禁用不用的 metric**。

```yaml
spec:
  metrics:
  - providers: [{name: prometheus}]
    overrides:
    - match: { metric: TCP_OPENED_CONNECTIONS }
      disabled: true   # 不需要 TCP 指标
```

**第三步：URL normalize**。

如果一定要看 path，用 EnvoyFilter 把 ID 替换成占位符：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
spec:
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_INBOUND
    patch:
      operation: INSERT_BEFORE
      value:
        name: envoy.filters.http.lua
        typed_config:
          inlineCode: |
            function envoy_on_request(req)
              local path = req:headers():get(":path")
              path = string.gsub(path, "/users/%d+", "/users/{id}")
              req:headers():replace(":path", path)
            end
```

**第四步：Prometheus 拉取层做 metric_relabel**。

```yaml
metric_relabel_configs:
- source_labels: [__name__]
  regex: 'envoy_.*_internal'
  action: drop
```

#### 验证

```bash
# Prometheus 自身的 timeseries 数
curl -s http://prometheus:9090/api/v1/status/tsdb | jq '.data.headStats'

# Sidecar 暴露的 metrics 大小
kubectl exec my-pod -c istio-proxy -- \
  curl -s localhost:15090/stats/prometheus | wc -c
```

修复前 50MB+，修复后 < 5MB。

> **口诀**：Istio 默认指标对 Prometheus 是核武器。**上线前必须先在 Telemetry CRD 里裁剪 label，至少干掉 request_path 和 source_principal**。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 安装/版本层
- [ ] istio 版本 >= 1.20（Native Sidecar 支持）
- [ ] istiod 至少 3 副本，配 PodDisruptionBudget
- [ ] 根证书来自外部 CA（cert-manager + Vault），不依赖 istiod 自管
- [ ] Telemetry CRD 已配，禁用 `request_path` 等高基数 label
- [ ] 控制面与数据面版本不要差超过 2 个 minor

### 注入层
- [ ] namespace label `istio-injection=enabled` 已确认目标 ns 都打了
- [ ] 不需要 mesh 的特殊 Pod 加 `sidecar.istio.io/inject: "false"`
- [ ] Job / CronJob 评估是否禁用注入；必须注入的用 `holdApplicationUntilProxyStarts`
- [ ] init container 跑业务流量的，加 `excludeOutboundIPRanges` 或 `excludeOutboundPorts`

### 资源层
- [ ] Sidecar CPU / Memory limit 经过实测（不依赖默认）
- [ ] 大集群（>1000 Service）必须配 Sidecar CRD 收敛配置
- [ ] istiod CPU / Memory request 留够（每 1000 Service 至少 1C/2G）
- [ ] Pod resource limit 把 Sidecar 算进去，别让业务挤掉 Sidecar

### 流量治理层
- [ ] VirtualService 的 http 列表"精确规则在前，兜底规则在后"
- [ ] DestinationRule 的 subset 与 VS 引用一一对应（无悬空引用）
- [ ] 所有 DestinationRule 配 `outlierDetection`（否则熔断 / failover 不生效）
- [ ] 提交前必跑 `istioctl analyze --recursive`
- [ ] 跨 namespace 的 VS 检查 `exportTo` 范围

### 安全层
- [ ] PeerAuthentication 默认 STRICT mTLS
- [ ] 升级 istiod 前对比 trust bundle（`istioctl proxy-config secret`）
- [ ] AuthorizationPolicy 用白名单（DENY by default + 显式 ALLOW）
- [ ] 敏感服务的 AuthorizationPolicy 测试过"未认证请求"被拒

### 多集群层
- [ ] 所有 cluster trustDomain 一致
- [ ] 跨 cluster CA 互信已建立（`cacerts` secret）
- [ ] 跨 region failover 配 `localityLbSetting` + `outlierDetection`
- [ ] 网络策略（NetworkPolicy / 安全组）放通 east-west gateway

### 可观测性层
- [ ] Prometheus 抓 Envoy `:15090/stats/prometheus`
- [ ] 关键 metric 告警：xDS push 延迟、配置生成耗时、TLS 错误率
- [ ] istiod 自身 metric 接入（pilot_xds_push_time、pilot_inbound_updates）
- [ ] Sidecar access log 输出到中央日志（量大时按采样率）

### Ambient 层（如启用）
- [ ] ztunnel DaemonSet 资源 request 配齐（每节点 ~1C/1G）
- [ ] 需要 L7 治理的 namespace 部署 waypoint proxy
- [ ] Ambient 与 Sidecar 共存场景测试通过（迁移期）

### 回滚预案
- [ ] 每次 VS / DR 变更走 GitOps，可一键回滚
- [ ] 控制面升级回滚预案文档化
- [ ] 数据面（Sidecar 镜像）灰度发布，按 namespace 滚

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### Linkerd 的 Rust 数据面
- 为什么 Linkerd 选择**自研 linkerd2-proxy**（Rust + Tokio）而不是 Envoy
- Linkerd 的 P99 延迟比 Istio 低 30%~50%，开销低 50% —— 实测
- 代价：功能比 Istio 少（没 EnvoyFilter、WASM 等扩展）
- 适合：纯 mTLS + 基础治理的极简场景

### Cilium Service Mesh（基于 eBPF）
- 把 L4 治理 + mTLS 放到内核 eBPF，**Sidecar 完全消失**
- L7 治理走"按需 Envoy"（每节点一个共享代理）
- 跟 K8s CNI 合一，性能上限更高
- 代价：内核版本要求高（>= 5.10），Cilium 学习曲线陡

### Ambient Mesh 设计深入
- ztunnel 的 `HBONE` 协议（HTTP-Based Overlay Network Encapsulation）
- waypoint proxy 的"按需启动"机制
- L4 mTLS 怎么在节点级共享 Sidecar 不破坏 ServiceAccount 身份隔离
- 迁移路径：Sidecar → Ambient 的 namespace-by-namespace 渐进式

### Wasm 扩展 Envoy
- WebAssembly filter 替代 EnvoyFilter 的 Lua 脚本
- proxy-wasm SDK（Rust / Go / C++）
- 跑在 sandbox 里，不影响 Envoy 主进程稳定性
- 案例：自定义 JWT 验证、自定义流量染色、跨服务 trace 注入

### GAMMA / Gateway API
- K8s 官方下一代 API，**取代 Ingress 和 Istio Gateway**
- HTTPRoute / GRPCRoute / TLSRoute 标准化
- Istio 1.20+ 已 GA 支持
- 长期看 VirtualService 会被 HTTPRoute 替代

### Multi-Cluster Mesh
- Primary-Remote vs Multi-Primary vs External Control Plane
- 跨集群服务发现（east-west gateway）的网络成本
- 全局 SLO + locality-aware routing
- 代价：故障域扩大，跨集群配置漂移检测难

### Service Mesh 与 Java SDK 的成本对比
- Spring Cloud + Sentinel + SkyWalking 的总开销 vs Istio + Sidecar
- 多语言场景下 Mesh 的边际成本优势点
- "胖 SDK + 瘦 Sidecar"混合架构（业务关键治理在 SDK，通用治理下沉）

### Envoy 内部模型
- xDS 协议的 v2 → v3 升级历史教训
- ADS / SOTW / Delta xDS 三种模式的取舍
- Envoy 的 thread model（main / worker / dispatch）
- TLS 1.3 0-RTT 在 Envoy 里的实现

---

## 7. 引用

- Istio 1.20 文档：[https://istio.io/latest/docs/](https://istio.io/latest/docs/)
- Envoy 官方文档：[https://www.envoyproxy.io/docs/envoy/latest/](https://www.envoyproxy.io/docs/envoy/latest/)
- xDS API spec：[https://github.com/cncf/xds](https://github.com/cncf/xds)
- Istio Ambient Mesh whitepaper：[https://istio.io/latest/blog/2022/introducing-ambient-mesh/](https://istio.io/latest/blog/2022/introducing-ambient-mesh/)
- KEP-753: Sidecar Containers（K8s 1.28+）
- *Service Mesh Patterns* by Lee Calcote（O'Reilly）
- Linkerd 性能对比：[https://linkerd.io/2021/05/27/linkerd-vs-istio-benchmarks/](https://linkerd.io/2021/05/27/linkerd-vs-istio-benchmarks/)
- Cilium Service Mesh：[https://cilium.io/blog/2021/12/01/cilium-service-mesh-beta/](https://cilium.io/blog/2021/12/01/cilium-service-mesh-beta/)
- Gateway API：[https://gateway-api.sigs.k8s.io/](https://gateway-api.sigs.k8s.io/)
- 蚂蚁 MOSN 设计文档（国产 Sidecar 方案）

---

> 配套：[Kubernetes 实战](/article/k8s/k8s-essentials) | [注册中心对比](/article/distributed/service-discovery) | [TCP 状态机](/article/network/tcp-state)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你觉得这个深度对，告诉我下一篇该写哪个。
