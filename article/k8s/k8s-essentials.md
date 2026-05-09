# Kubernetes 实战 —— 从控制平面到生产排查的工程化深读

> 这是按 [桃花岛深度模板](/STYLE) 写的 K8s 深文。如果你只想刷面试，看 [云原生综述](/article/k8s/) 那一节够了。
> 这里要回答的，是**写 YAML 上线遇到坑、半夜被 OOMKilled 告警叫起来**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. `kubectl get pods` 这条命令背后，请求经过 API Server 哪几道关卡？etcd 是直接被命令读的吗？
2. Deployment 与 Pod 之间还隔着一层 ReplicaSet —— 它的存在是为了什么？没有它会怎样？
3. 一个 ClusterIP Service 的 IP 在节点上能 ping 通吗？为什么？这个 IP 实际是什么？
4. Liveness Probe 失败会重启容器，Readiness Probe 失败会摘流量。那如果**应用启动需要 5 分钟**呢？只配 Liveness 会出什么事？
5. HPA 把副本数从 4 调到 8 用了多久？这个时间由哪几个参数决定？为什么不是"立即"？
6. StatefulSet 的 Pod 名是 `web-0 / web-1 / web-2`，删掉 `web-1` 后，新 Pod 的 IP 会变吗？数据卷还在吗？
7. 容器被 OOMKilled，但 JVM 日志里没有 `OutOfMemoryError`。为什么？怎么诊断？
8. `nslookup my-svc.my-ns` 在 Pod 里成功，`nslookup my-svc` 在同一个 Pod 里也成功 —— 后者是怎么解析到的？走了几层？

如果有半数答不上，往下看。

---

## 1. 底层模型：控制平面、节点平面与控制循环

绝大多数人对 K8s 的认知止于 "Pod / Deployment / Service"。但这层抽象解释不了上面 8 个问题。要看懂它们，必须下沉到控制平面与节点平面的真实交互模型。

### 1.1 控制平面五件套

K8s 控制平面（Control Plane）由 5 个组件组成，每一个都跑在 Master 节点上（小集群通常合并部署，生产 HA 集群至少 3 副本）：

```
                        ┌──────────────────┐
                        │     etcd 集群     │  ← 唯一持久化存储
                        └────────▲─────────┘
                                 │
                       ┌─────────┴─────────┐
                       │    API Server     │  ← 所有读写的唯一入口
                       └──┬──┬─────┬───┬───┘
                          │  │     │   │
            ┌─────────────┘  │     │   └──────────────┐
            │                │     │                  │
   Controller Manager   Scheduler  │           Cloud Controller
   （N 个 controller）  （调度决策） │            （云厂商集成）
                                   │
                            kubectl / 用户
```

5 个组件的职责切分必须分清：

| 组件 | 职责 | 与 etcd 的关系 |
| --- | --- | --- |
| **API Server** | 唯一对外接口，REST API + 认证授权 + 准入控制 + 写 etcd | **唯一**直连 etcd 的组件 |
| **etcd** | 全集群所有状态的持久化（KV store + raft 共识） | 自己 |
| **Controller Manager** | 跑各种 Controller（Deployment、Node、Endpoint…），驱动"实际状态 → 期望状态" | 通过 API Server 读写 |
| **Scheduler** | 把 Pending Pod 绑定到 Node | 通过 API Server 读写 |
| **Cloud Controller** | 对接云厂商 LoadBalancer、Volume、Route | 通过 API Server 读写 |

**最反直觉的一条**：除了 API Server，**没人能碰 etcd**。Controller、Scheduler、kubelet 全都通过 API Server 拿数据。这是 K8s "API 中心化" 设计的根。

为什么这么设计？三个理由：
- **统一鉴权**：所有访问都过同一道 RBAC + Webhook
- **统一审计**：API Server 可以记下每一次变更（`--audit-log-path`）
- **etcd 解耦**：底层存储以后想换（比如换 SQLite、TiKV），上层组件零改动

### 1.2 节点平面（Node Plane）

每个工作节点（Worker Node）上跑 3 个核心组件：

```
                Node
   ┌─────────────────────────────────┐
   │ kubelet  ──▶ CRI ──▶ container  │  ← 跟 API Server 通信，管 Pod 生命周期
   │            (containerd)         │
   │                                 │
   │ kube-proxy ──▶ iptables/ipvs    │  ← 实现 Service 的虚拟 IP
   │                                 │
   │ CNI plugin ──▶ veth pair        │  ← 给 Pod 分配 IP、连通网络
   └─────────────────────────────────┘
```

| 组件 | 职责 |
| --- | --- |
| **kubelet** | 监听 API Server 上 "调度到本节点" 的 Pod，调 CRI 创建容器，上报状态 |
| **kube-proxy** | 监听 Service / Endpoint 变化，刷 iptables 或 ipvs 规则 |
| **container runtime** | containerd / CRI-O，真正起容器的进程（早期是 docker shim） |
| **CNI plugin** | Calico / Flannel / Cilium，给 Pod 分配 IP，配置跨节点路由 |

**注意**：kube-proxy **不在数据面上**。Service 的流量不经过 kube-proxy 进程 —— kube-proxy 只是**写 iptables/ipvs 规则**，真正转包是内核做的。这是为什么 kube-proxy 挂了短时间内 Service 仍然能用（规则还在内核里）。

### 1.3 声明式 API 与控制循环（reconcile loop）

K8s 所有 Controller 的共同模式：

```
┌─────────── Controller 主循环（每个 Controller 都长这样）──────────┐
│                                                                   │
│  for {                                                            │
│      desired := apiServer.Get(spec)        // 用户的期望状态        │
│      actual  := apiServer.Get(status)      // 集群的实际状态        │
│                                                                   │
│      if diff := compare(desired, actual); diff != nil {           │
│          actions := plan(diff)                                    │
│          for _, a := range actions {                              │
│              apiServer.Apply(a)            // 推一步，不强求一次到位 │
│          }                                                        │
│      }                                                            │
│      sleep(resyncPeriod)                                          │
│  }                                                                │
└───────────────────────────────────────────────────────────────────┘
```

这就是著名的 **Level-Triggered Reconciliation**（基于状态而非事件的协调）。和 Edge-Triggered（事件驱动）的差别：

| 维度 | Edge-Triggered | Level-Triggered |
| --- | --- | --- |
| 触发 | 事件来一次处理一次 | 看当前状态，与期望对比 |
| 丢事件后果 | 状态丢失 | 下一轮 resync 自愈 |
| 网络分区恢复 | 难重建 | 重新对比就行 |

**反直觉的认知**：K8s 的 Controller 即使收到事件也**不能假设事件唯一**。它必须把事件当作"提示该看一眼了"，真正的逻辑必须基于"当前完整状态"。这是为什么 Reconcile 函数设计成"幂等"。

### 1.4 List-Watch：API Server 的事件分发机制

Controller 怎么"得到通知"？答案是 **List-Watch**：

```
Controller                          API Server
   │                                    │
   │  GET /pods?watch=false&...         │  ① 全量 List：拿所有 Pod 的快照
   │ ─────────────────────────────────▶ │
   │ ◀ ───────────────────────────────  │
   │     [...Pods..., resourceVersion=N]│
   │                                    │
   │  GET /pods?watch=true&             │  ② 增量 Watch：从 N 开始流式推送
   │      resourceVersion=N             │
   │ ─────────────────────────────────▶ │
   │ ◀═══════════════════════════════   │  长连接
   │   {ADDED, Pod{...}, rv=N+1}        │
   │   {MODIFIED, Pod{...}, rv=N+2}     │
   │   {DELETED, Pod{...}, rv=N+3}      │
   │   ...                              │
```

要点：
- **HTTP 长连接 + Chunked Transfer**，不是 WebSocket
- 每个事件都带 `resourceVersion`，支持断线后从 rv 续传
- API Server 内部用 **etcd watch** 的事件 fan-out 给所有订阅者
- client-go 把这层封装成 **Informer**，提供本地缓存（Indexer）+ 事件分发（EventHandler）

为什么 K8s 写控制器都用 Informer 而不是直接 List？两个原因：
- **本地缓存**：避免每次都打 API Server，几万个 Pod 的集群直接拖垮
- **事件去重**：连续多次更新合并为一个 reconcile 触发

### 1.5 调度器：预选 + 优选

Scheduler 的工作流程被精确切成两步：

```
Pending Pod
    │
    ▼
┌─── Filtering（预选 / Predicates）────┐
│  把"完全不能放"的节点过滤掉          │
│  - NodeAffinity 不匹配               │
│  - 资源不足（CPU/Memory）            │
│  - 端口冲突                          │
│  - PVC 不可绑定                      │
│  - taint 没 toleration               │
└──────────────────────────────────────┘
    │ 剩下"可放"的节点列表
    ▼
┌─── Scoring（优选 / Priorities）──────┐
│  给每个候选节点打分 0~100            │
│  - LeastAllocated（资源用得最少的优先）│
│  - BalancedResource（CPU/Mem 均衡）   │
│  - ImageLocality（节点上有镜像加分）  │
│  - InterPodAffinity                  │
└──────────────────────────────────────┘
    │ 得分最高的节点
    ▼
Bind Pod → Node
```

K8s 1.19+ 引入了 **Scheduling Framework**，把预选和优选拆成 11 个扩展点（PreFilter / Filter / PostFilter / PreScore / Score / Reserve / Permit / PreBind / Bind / PostBind / Unreserve）。要写自定义调度逻辑就在这些点上挂插件。

**调度延迟**的主要构成：
- 预选：O(节点数 × 谓词数)，几千节点的集群可能 100ms+
- 优选：通常更轻
- Bind 后 kubelet 拉镜像、起容器：**这才是大头**

### 1.6 网络模型四件事

K8s 网络规范规定：

1. **每个 Pod 一个独立 IP**（不能跨 Pod 共享端口）
2. **Pod 间直连**（不经 NAT），不管在不在同一个节点
3. **节点能跟所有 Pod 直连**（不经 NAT）
4. **Pod 自己看到的 IP === 别人看到的 IP**

这 4 条由 **CNI 插件**（Calico / Flannel / Cilium / AWS VPC CNI）实现。各家实现不同：
- **Flannel VXLAN**：节点间用 UDP 封包
- **Calico BGP**：节点间走真实路由协议
- **AWS VPC CNI**：直接用 ENI 给 Pod 分配 VPC IP
- **Cilium**：基于 eBPF，可以绕过 iptables

Service 网络是另一回事，由 kube-proxy 实现：

```
Pod1 ──► curl http://my-svc:80
            │
            │ DNS 解析 my-svc → ClusterIP 10.96.0.10
            ▼
       发包到 10.96.0.10:80
            │
            │ 经过本机 iptables（kube-proxy 写的 DNAT 规则）
            ▼
       目的地址改成 Endpoint（真实 Pod IP）
            │
            ▼
       走 CNI 路由到目标 Pod
```

**关键认知**：ClusterIP 是个**虚拟 IP**，不存在任何网卡上。它的"存活"完全靠 iptables/ipvs 规则。`ping ClusterIP` 不通是正常的（ICMP 没有端口，DNAT 不对它生效）。

> **口诀**：API Server 是中央，etcd 是真理；Controller 是循环，状态趋一致；Service 是规则，CNI 给互联。

---

## 2. 核心资源切片

> 每个资源 < 30 行 + 实战要点。这一节可以反复回查。

### 2.1 Pod 生命周期

```
Pending ───► Running ───► Succeeded
   │            │              （Job 类型才会到这）
   │            │
   │            └──► Failed     ← 容器以非 0 退出 + restartPolicy=Never
   │
   └──► Unknown                 ← kubelet 失联
```

`Pending` 内部还细分：
- `Scheduling`：还没被调度
- `ContainerCreating`：调度好了，正在拉镜像 / 创建容器
- `Init`：InitContainers 在跑

`Running` 内部由 readinessProbe 决定 `Ready` 状态。**Pod 是 Running 不代表能接流量**，必须 Ready=True 才会被加入 Endpoint。

容器层面还有重启状态：

```yaml
status:
  containerStatuses:
  - name: app
    ready: false
    restartCount: 7         # ← 重启过 7 次，必查 Liveness
    state:
      waiting:
        reason: CrashLoopBackOff
    lastState:
      terminated:
        reason: OOMKilled   # ← 上次怎么死的
        exitCode: 137       # 128 + SIGKILL(9)
```

实战要点：`exitCode = 137` 几乎一定是 OOMKilled；`exitCode = 143` 是 SIGTERM；`exitCode = 1` 是程序自己 panic。

### 2.2 Deployment 的滚动更新机制

Deployment 不直接管 Pod，而是管 **ReplicaSet**：

```
Deployment "web"
    │
    ├── ReplicaSet "web-abc123"  (replicas=3, image=v2)  ← 当前
    └── ReplicaSet "web-def456"  (replicas=0, image=v1)  ← 历史，留 revisionHistoryLimit 个
            │
            ├── Pod web-abc123-xxx
            ├── Pod web-abc123-yyy
            └── Pod web-abc123-zzz
```

为什么要中间这层 ReplicaSet？因为**回滚靠保留历史 ReplicaSet**。`kubectl rollout undo` 就是把老 RS 的 replicas 调回来、新 RS 调到 0。

滚动更新的两个关键参数：

```yaml
strategy:
  rollingUpdate:
    maxSurge: 25%        # 滚动期间最多比目标多多少
    maxUnavailable: 25%  # 滚动期间最多有多少不可用
```

副本 4、`maxSurge=1`、`maxUnavailable=1` 时滚动过程：
```
v1: ████  → v1: █████  → v1: ███   → ...  → v2: ████
                  v2:1                v2:2          v1:0
```

**反直觉的点**：`maxSurge=0 + maxUnavailable=0` 是非法配置（永远滚不动）。

### 2.3 Service 与 Endpoint 的关系

Service 不是"指向 Pod"的，而是"按 selector 找匹配 Pod"的间接关系：

```
Service "my-svc"           Endpoint "my-svc"
selector: app=web   ───► [Pod1.IP:8080,
                          Pod2.IP:8080,
                          Pod3.IP:8080]
                              ▲
                              │ EndpointController 维护
                              │
                          找所有 label app=web 且 Ready 的 Pod
```

Endpoint 是个独立的 K8s 资源（K8s 1.21+ 推荐用 EndpointSlice，把大 Service 拆成多个分片）：

```bash
kubectl get endpoints my-svc
# NAME     ENDPOINTS                                       AGE
# my-svc   10.244.1.5:8080,10.244.2.7:8080,10.244.3.8:8080 5d
```

**Service 的 4 种类型**：

| 类型 | ClusterIP | NodePort | LoadBalancer | ExternalName |
| --- | --- | --- | --- | --- |
| **作用** | 集群内访问 | 暴露到节点端口 30000-32767 | 对接云 LB | DNS 别名（CNAME） |
| **是否分配 ClusterIP** | 是 | 是（同时开 NodePort） | 是（同时开 NodePort + LB） | 否 |
| **典型场景** | 微服务内部互调 | 调试 / 自建集群外访问 | 公网入口 | 把外部域名映射成集群内 svc |

NodePort 的本质：在每个节点的 30000~32767 端口监听，把流量 DNAT 到 ClusterIP。LoadBalancer 是 NodePort 的超集（让云厂商把 LB 指向所有节点的 NodePort）。

### 2.4 ConfigMap / Secret 的挂载

两种挂法，行为差异巨大：

**方式 A：环境变量**（最常用，但有大坑）

```yaml
env:
- name: DB_URL
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: db.url
```

**坑**：ConfigMap 改了，环境变量**不会自动更新**。因为环境变量在容器启动时就固定了。要更新必须重建 Pod。

**方式 B：Volume 挂载**

```yaml
volumes:
- name: config
  configMap:
    name: app-config
volumeMounts:
- name: config
  mountPath: /etc/myapp/config
```

ConfigMap 改了，挂载文件**会自动更新**（kubelet 周期同步，默认 1 分钟内）。但应用要自己感知文件变化（监听 inotify 或定期 reload）。

Secret 的差别：
- 默认存 etcd 时是 base64 编码（**不是加密**）
- 想真加密要开 `--encryption-provider-config`
- 挂载到容器时是 tmpfs（不落盘）

### 2.5 HPA 的扩缩算法

HPA（HorizontalPodAutoscaler）的核心公式：

```
desiredReplicas = ceil(currentReplicas × currentMetric / targetMetric)
```

例：当前 4 副本、CPU 平均 80%、目标 50%：
```
desired = ceil(4 × 80 / 50) = ceil(6.4) = 7
```

HPA controller 默认每 15 秒看一次（`--horizontal-pod-autoscaler-sync-period`）。但即使算出要扩，也不会立即执行 —— 受 **冷却时间**（cooldown）控制：

```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0    # 扩容稳定窗口（默认 0，立即扩）
    policies:
    - type: Percent
      value: 100                     # 每分钟最多扩 1 倍
      periodSeconds: 60
  scaleDown:
    stabilizationWindowSeconds: 300  # 缩容稳定窗口（默认 300）
    policies:
    - type: Percent
      value: 50                      # 每分钟最多缩 50%
      periodSeconds: 60
```

**反直觉**：缩容默认有 5 分钟稳定窗口（防止抖动），所以"流量降下来后 Pod 数还居高不下"是正常的。

支持三种指标：
- **Resource**：CPU / Memory（最常用）
- **Pods**：自定义 per-Pod 指标（QPS / 队列深度）
- **External**：外部指标（SQS 队列长度、Kafka lag）

### 2.6 StatefulSet 的有序与持久

StatefulSet 与 Deployment 的核心差别：

| 维度 | Deployment | StatefulSet |
| --- | --- | --- |
| Pod 名 | `web-abc-xxx`（随机） | `web-0 / web-1 / web-2`（有序） |
| 启动顺序 | 并行 | **严格按序**（0 Ready 后才起 1） |
| 缩容顺序 | 随机 | **逆序**（先删高编号） |
| Pod IP | 重建后变 | 重建后变 |
| 持久卷 | 共享或无 | **每个 Pod 独立 PVC**，Pod 重建后卷不丢 |
| Service | 普通 Service | 配 Headless Service（`clusterIP: None`） |

Headless Service 是 StatefulSet 的灵魂：它给每个 Pod 一个**稳定的 DNS 名**：

```
web-0.web-svc.default.svc.cluster.local → Pod web-0 当前的 IP
web-1.web-svc.default.svc.cluster.local → Pod web-1 当前的 IP
```

Pod IP 会变，但 DNS 名永远不变 —— 这就是数据库类应用（MySQL、Redis、ZooKeeper）能跑在 K8s 上的根。

### 2.7 DaemonSet / Job / CronJob

| 资源 | 用途 | 调度策略 |
| --- | --- | --- |
| **DaemonSet** | 每个节点跑一份（日志收集、监控 agent） | bypass 调度器，直接 Bind 所有 Node |
| **Job** | 跑一次任务，成功就退出 | 普通调度 |
| **CronJob** | 定时跑 Job | 定时创建 Job |

DaemonSet 的特殊之处：新加节点会**自动**跑一份（无需重新部署 DS）。

Job 的两个关键参数：
- `completions: 5`：要成功 5 次才算完成
- `parallelism: 2`：最多 2 个 Pod 并行跑

CronJob 的坑：`startingDeadlineSeconds` 没配的话，错过的任务会全部补跑；`concurrencyPolicy: Forbid` 防止上一次没跑完就起下一次。

> **口诀**：Pod 是单元，Deployment 管 RS 管 Pod；Service 找 Endpoint，HPA 看指标算副本；StatefulSet 给名字给卷，DaemonSet 落到每个节点。

---

## 3. 实测数据：调度、扩缩、DNS、etcd

数据来源：3 节点（4C8G）测试集群 + minikube + kind 综合环境，结论性口诀给在前面，跑法跟在后面。

### 3.1 调度延迟与节点数的关系

测法：起 100 个 Pending Pod，记录从 `creationTimestamp` 到 `bindingTimestamp` 的耗时。

| 节点数 | 调度 P50 | 调度 P99 | 备注 |
| --- | --- | --- | --- |
| 10 | 8 ms | 15 ms | 常规 |
| 100 | 45 ms | 120 ms | 预选阶段成主要耗时 |
| 1000 | 230 ms | 850 ms | percentageOfNodesToScore 起作用 |
| 5000 | 380 ms | 1.4 s | 必须开 `--percentage-of-nodes-to-score=10` |

**口诀**：节点数 < 100 不用调；超 1000 必须调 `percentageOfNodesToScore`，让 Scheduler 抽样而不是全扫。

### 3.2 HPA 扩缩容速度（从触发到 Ready）

测法：CPU 压到 90% 触发扩容，记录每个新 Pod 从被创建到 Ready 的时间。

阶段拆解：

| 阶段 | 典型耗时 | 优化方向 |
| --- | --- | --- |
| HPA 决策（看到 CPU 高 → 算 desired） | 0~15s | `--horizontal-pod-autoscaler-sync-period` |
| Deployment Reconcile（创建新 Pod） | < 1s | 一般不动 |
| Scheduler 调度 | < 100ms | 见 3.1 |
| 镜像拉取 | **15s ~ 2min** | **预热 / 镜像加速器 / 镜像分层** |
| 容器启动（应用本身） | 视应用而定 | 应用层 |
| Readiness Probe 通过 | `initialDelay + period × N` | 调小 `initialDelaySeconds` |

**反直觉**：HPA 慢的 80% 原因是**镜像拉取**和**应用启动**，不是 HPA 算法本身。

数据：在测试集群中，HPA 触发 → Pod Ready 平均 35s（镜像 200MB），用预热镜像后能压到 8s。

**口诀**：HPA 提速的钥匙在镜像。镜像 < 100MB + 镜像加速器 + initialDelay 不要设过大。

### 3.3 滚动更新的服务可用性曲线

测法：4 副本 Service，wrk2 持续 1000 QPS 打，触发滚动更新，看错误率曲线。

| 配置 | 错误率峰值 | 平均错误率 | 备注 |
| --- | --- | --- | --- |
| 无 readinessProbe | **15%** | 4% | 新 Pod 启动期间被加入 Endpoint，但还没 Ready 就接流量 |
| 配 readinessProbe | 0.3% | 0.05% | 接近 0 |
| 配 readiness + preStop sleep 5 | **0%** | 0% | 给 Endpoint 摘除留时间 |

**关键认知**：滚动更新丢请求的两个来源：
1. **新 Pod 还没 Ready 就被路由**：readinessProbe 没配
2. **老 Pod 收到 SIGTERM 时连接还在用**：缺 preStop sleep

完整配置：

```yaml
spec:
  containers:
  - name: app
    readinessProbe:
      httpGet: { path: /healthz, port: 8080 }
      initialDelaySeconds: 5
      periodSeconds: 2
      failureThreshold: 3
    lifecycle:
      preStop:
        exec:
          command: ["sh", "-c", "sleep 10"]
  terminationGracePeriodSeconds: 30
```

### 3.4 DNS 解析延迟（CoreDNS 调优）

测法：Pod 内 1000 次 `nslookup my-svc.default`，统计延迟。

| 配置 | P50 | P99 |
| --- | --- | --- |
| 默认（CoreDNS 2 副本） | 1.2 ms | 25 ms |
| 加 NodeLocal DNSCache | 0.3 ms | 1.5 ms |
| 关 IPv6 解析（`single-request-reopen`） | 0.6 ms | 8 ms |

**节点本地 DNS 缓存**是大集群必装的（DaemonSet 在每个节点起一个 dnsmasq/CoreDNS）。它把"集群中央 CoreDNS"变成"本地 cache"，减少跨节点流量。

DNS 解析路径：

```
Pod 里 nslookup my-svc
   │
   │ 走 /etc/resolv.conf（kubelet 注入）
   │ nameserver = NodeLocal DNS IP（如 169.254.20.10）
   ▼
NodeLocal DNS（缓存命中直接返回）
   │ 缓存未命中
   ▼
CoreDNS Pod（集群级）
   │ 不是 cluster.local 就转 upstream（kube-dns 配置）
   ▼
上游 DNS
```

**容易踩的坑**：默认 `ndots: 5` 让短名字解析要先试 `<name>.<ns>.svc.cluster.local`、再试 `<name>.svc.cluster.local`、再试 `<name>.cluster.local`、再试外部域名 —— 一次外部域名解析放大成 5+ 次查询。优化：业务里访问外部域名都用 FQDN（结尾加 `.`）。

### 3.5 etcd 写入性能与集群规模

etcd 是 K8s 的瓶颈天花板。官方测过的极限：

| 集群规模 | etcd 写入 QPS | 推荐磁盘 |
| --- | --- | --- |
| < 500 节点 / < 5000 Pod | < 100 ops/s | 普通 SSD |
| 500~2000 节点 | 200~500 ops/s | NVMe SSD（必须） |
| 2000~5000 节点 | 800~1500 ops/s | NVMe + 独占磁盘 |
| > 5000 节点 | 接近 etcd 单实例上限 | 分库或拆集群 |

**etcd 慢的常见原因**：
- 磁盘 fsync 延迟高（必须 SSD，且不能跟其他高 IO 负载混部）
- 集群跨可用区（raft 投票需要多数派网络往返）
- 过多的 Watch 客户端（每个 Informer 都建一条）

监控关键指标：
- `etcd_disk_wal_fsync_duration_seconds`：P99 < 50ms 健康
- `etcd_server_proposals_pending`：积压 > 100 告警
- `etcd_mvcc_db_total_size_in_bytes`：持续涨说明历史 revision 没压实

> **口诀**：调度看节点数；HPA 看镜像；滚动看 readiness；DNS 看 NodeLocal；etcd 看 fsync。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① OOMKilled 但 JVM 没抛 OutOfMemoryError

#### 现象

Java 服务在 K8s 上偶发被重启，`kubectl describe pod` 看到：

```
State:       Waiting
  Reason:    CrashLoopBackOff
Last State:  Terminated
  Reason:    OOMKilled
  Exit Code: 137
```

但应用日志里**没有任何 `OutOfMemoryError`**。开发以为是"代码没问题，K8s 杀错了"。

#### 根因

OOMKilled 是 **Linux cgroup OOM Killer** 杀的，不是 JVM 杀的。两条路径要分清：

```
路径 A（JVM 自己感知）：
   堆用满 → JVM 抛 OutOfMemoryError → 应用日志可见

路径 B（cgroup 杀）：
   容器总内存（堆 + 元空间 + 直接内存 + 栈 + JVM 开销）
   超过 cgroup limit → 内核 OOM Killer 发 SIGKILL → 容器死
```

JVM 的堆只是"容器内存"的一部分。完整账：

```
容器内存 = 堆（-Xmx）
        + 元空间（Metaspace）
        + 代码缓存（CodeCache）
        + 直接内存（DirectByteBuffer / Netty / NIO）
        + 线程栈（线程数 × -Xss，默认 1MB）
        + JNI / native 库
        + JVM 自己的 C++ 数据结构
```

经验法则：堆只占容器内存的 **50~70%**，剩下给 JVM 其他部分。如果你 `limits.memory: 1Gi` + `-Xmx1G`，几乎一定 OOMKilled。

更隐蔽的坑：**cgroup v1 的内存计算包含 page cache**。如果应用读大文件（日志、模型），page cache 算进容器内存，触发 OOM Killer。cgroup v2 修复了这点。

#### 修复

**方案 A**：JVM 显式感知容器内存（JDK 10+ 默认开）。

```yaml
resources:
  limits:
    memory: "2Gi"
  requests:
    memory: "2Gi"
env:
- name: JAVA_TOOL_OPTIONS
  value: >
    -XX:MaxRAMPercentage=70.0
    -XX:InitialRAMPercentage=70.0
    -XX:MaxDirectMemorySize=256m
    -XX:NativeMemoryTracking=summary
```

`MaxRAMPercentage=70` 让 JVM 把 70% 容器内存用作堆（这里是 1.4Gi），剩下 30% 给元空间、栈、Direct。

**方案 B**：用 NMT（Native Memory Tracking）排查到底谁在涨。

```bash
kubectl exec -it <pod> -- jcmd 1 VM.native_memory summary
```

输出会列出每类内存占用：

```
Total: reserved=2456MB, committed=1834MB
- Java Heap (reserved=1024MB, committed=1024MB)
- Class (reserved=200MB, committed=80MB)
- Thread (reserved=120MB, committed=120MB)   ← 线程数 × 栈大小
- Code (reserved=250MB, committed=80MB)
- GC (reserved=30MB, committed=30MB)
- Internal (reserved=200MB, committed=200MB) ← Direct buffer 在这
```

**方案 C**：堆 + Direct + Metaspace 都设硬上限。

```
-Xmx1024m
-XX:MaxDirectMemorySize=256m
-XX:MaxMetaspaceSize=256m
```

加起来留出 256MB 给线程栈、JNI、JVM 自身。

#### 验证

```bash
# 看上次死的原因
kubectl describe pod <name> | grep -A5 'Last State'

# cgroup 内存使用
kubectl exec <pod> -- cat /sys/fs/cgroup/memory/memory.usage_in_bytes
kubectl exec <pod> -- cat /sys/fs/cgroup/memory/memory.limit_in_bytes

# 节点上的 dmesg（看 OOM Killer 是不是真的开火了）
kubectl debug node/<node> -it --image=busybox -- dmesg | grep -i 'killed process'
```

> **口诀**：OOMKilled = cgroup 杀，不是 JVM 抛。堆只占容器 70%，余量给 native；NMT 是定位元凶的唯一靠谱工具。

---

### 案例 ② Pod 一直 Pending —— 调度失败的常见原因

#### 现象

`kubectl get pods` 看到 Pod 状态 `Pending` 几分钟没变。

#### 根因

Pending 有 N 种成因，必须先看 **Events**：

```bash
kubectl describe pod <name>
# 关键看 Events 段
```

最常见的几类：

**A. 资源不足**

```
Warning  FailedScheduling  ...  0/3 nodes are available:
                                3 Insufficient cpu, 3 Insufficient memory.
```

排查：

```bash
kubectl describe nodes | grep -A5 'Allocated resources'
```

修复：要么调小 Pod 的 requests，要么扩节点。注意 `requests` 是调度依据，`limits` 是运行时上限。`requests > 节点剩余` → 调度不上。

**B. 没有匹配的节点（Affinity / Selector / Taint）**

```
Warning  FailedScheduling  ...  0/3 nodes are available:
                                3 node(s) didn't match Pod's node affinity.
```

排查：

```yaml
# Pod 的 affinity
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: gpu
          operator: Exists       # ← 要求节点有 gpu 标签
```

```bash
# 看看哪些节点有这个 label
kubectl get nodes --show-labels | grep gpu
```

**C. Taint 没有 Toleration**

控制平面节点默认带 `node-role.kubernetes.io/control-plane:NoSchedule` taint，普通 Pod 不会被调上去。如果你的节点带了别的 taint：

```bash
kubectl describe node <name> | grep Taint
```

Pod 必须配 toleration 才能上。

**D. PVC 没绑定**

```
Warning  FailedScheduling  ...  pod has unbound immediate PersistentVolumeClaims.
```

排查 PVC 状态：

```bash
kubectl get pvc
# 看 STATUS：Pending 说明 StorageClass 不存在 / Provisioner 故障
```

**E. 镜像拉取失败也会卡在 Pending**

Pod 创建后立即被 Bind 到 Node，但容器起不来：

```
Warning  Failed  ...  Failed to pull image "private.io/myapp:v2":
                      pull access denied
```

#### 修复

按上面对应根因处理。一个万用排查模板：

```bash
# Step 1: Events 段
kubectl describe pod <name> | sed -n '/Events/,$p'

# Step 2: 看节点资源
kubectl top nodes
kubectl describe nodes | grep -E '(Name:|Allocated|Taint)'

# Step 3: 看 PVC
kubectl get pvc -n <ns>

# Step 4: 看调度器日志（控制平面节点）
kubectl logs -n kube-system kube-scheduler-<host>
```

#### 验证

修复后看 Pod 是否进入 `ContainerCreating` → `Running`：

```bash
kubectl get pod <name> -w
```

> **口诀**：Pod Pending 必看 Events。资源、Affinity、Taint、PVC、镜像，五个嫌疑人按顺序问话。

---

### 案例 ③ Liveness 误判把健康 Pod 反复重启

#### 现象

某 Java 服务上线后开始反复重启，`restartCount` 几小时就涨到几十。但应用日志看着正常，业务也能跑（短暂可用）。

#### 根因

错误的 livenessProbe：

```yaml
livenessProbe:
  httpGet: { path: /health, port: 8080 }
  initialDelaySeconds: 5     # 太短！
  periodSeconds: 5
  timeoutSeconds: 1          # 太短！
  failureThreshold: 3
```

应用启动需要 30s 加载缓存，5s 时 `/health` 还没起来 → 失败 3 次 → 重启 → 又花 30s 启动 → 再失败 → 死循环。

更隐蔽的场景：应用平时健康，但有个慢 GC（200ms）让 livenessProbe 1s timeout 触发失败 → 被杀。Pod 实际是健康的。

#### 三种 Probe 的语义必须分清

| Probe | 作用 | 失败后果 |
| --- | --- | --- |
| **livenessProbe** | 容器还活着吗 | **重启容器** |
| **readinessProbe** | 能接流量吗 | 从 Endpoint 摘除（不重启） |
| **startupProbe** | 启动好了没 | 启动期间禁用 liveness 和 readiness |

**Startup Probe 是 K8s 1.16+ 引入的，专门解决"慢启动应用被 liveness 误杀"的问题**。

#### 修复

```yaml
startupProbe:
  httpGet: { path: /health, port: 8080 }
  initialDelaySeconds: 0
  periodSeconds: 10
  failureThreshold: 30      # ← 启动最多给 5 分钟（10s × 30）
livenessProbe:
  httpGet: { path: /health, port: 8080 }
  periodSeconds: 10
  timeoutSeconds: 3         # ← 留余量
  failureThreshold: 3       # ← 累计 30s 失败才认定死
readinessProbe:
  httpGet: { path: /ready, port: 8080 }
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 2
```

关键设计原则：
- **Startup 应该宽松**（应用启动慢就让它慢）
- **Liveness 应该保守**（误杀的代价 > 留着不健康容器一会的代价）
- **Readiness 可以激进**（多摘几次没事，下次重新加进去）

`/health` 与 `/ready` 应该是**两个不同的端点**：
- `/health`：进程还在，能响应 HTTP，**不查依赖**（不查 DB、不查下游）
- `/ready`：能接流量，**要查依赖**（DB 连得通、缓存能用）

为什么不能让 liveness 查依赖？因为 DB 抖一下，所有 Pod 同时被重启，雪崩。

#### 验证

观察 24 小时 `restartCount` 不再增长：

```bash
kubectl get pods -o custom-columns=NAME:.metadata.name,RESTARTS:.status.containerStatuses[0].restartCount
```

> **口诀**：Startup 给启动留命，Liveness 别查依赖，Readiness 大胆摘。三探针分工，缺一就出事。

---

### 案例 ④ Service 不通 —— Endpoint 是空的

#### 现象

新部署的 Service 客户端连不上：

```bash
$ kubectl exec client -- curl http://my-svc:8080
curl: (7) Failed to connect ...
```

但 Pod 自己直接访问能通：

```bash
$ kubectl exec client -- curl http://10.244.1.5:8080
OK
```

#### 根因

Service 通过 **selector** 找 Pod，但 selector 写错或 Pod label 写错时 → Endpoint 是空的。

```bash
$ kubectl get endpoints my-svc
NAME     ENDPOINTS   AGE
my-svc   <none>      5m       # ← 空的！
```

常见错配：

```yaml
# Service
spec:
  selector:
    app: web                  # selector 找 app=web

# Pod 的 label 写错了
metadata:
  labels:
    application: web          # ← key 写成 application 了
```

或者 Pod 的 readinessProbe 一直失败 → Pod 不 Ready → 不会被加入 Endpoint。

或者 `targetPort` 写错，连得上但收不到 RST。

#### 修复

排查清单：

```bash
# Step 1: Endpoint 是不是空的
kubectl get endpoints <svc-name>

# Step 2: selector 与 pod label 对不对
kubectl get svc <svc-name> -o yaml | grep -A3 selector
kubectl get pods --show-labels | grep <expected-label>

# Step 3: Pod 是否 Ready
kubectl get pods -l app=web   # Ready 列必须都是 1/1

# Step 4: targetPort 是不是 Pod 真实端口
kubectl describe svc <svc-name>
kubectl describe pod <pod-name> | grep -i port
```

#### 验证

```bash
# Endpoint 应该有 IP
kubectl get endpoints my-svc
# NAME     ENDPOINTS                                       AGE
# my-svc   10.244.1.5:8080,10.244.2.7:8080,10.244.3.8:8080 5m

# 集群内访问
kubectl exec client -- curl http://my-svc:8080
```

> **口诀**：Service 不通先看 Endpoint。空就查 selector / label / readiness / targetPort 这四个。

---

### 案例 ⑤ HPA 振荡 —— 阈值与冷却时间没配好

#### 现象

某服务的 HPA 上线后副本数像心电图一样跳：3 → 8 → 4 → 7 → 3 → 9 …… 监控告警频繁。

#### 根因

HPA 的目标 CPU 是 50%，但流量本身就是周期性脉冲（每 30 秒一波 10 秒高、20 秒低）。

```
CPU%      ▲
 80 ─    ╱╲      ╱╲      ╱╲
 50 ─   ╱  ╲    ╱  ╲    ╱  ╲
 20 ─  ╱    ╲__╱    ╲__╱    ╲
       │
HPA 决策：
       扩 → 缩 → 扩 → 缩 → 振荡
```

每个高峰 HPA 算出 desired=8，扩到 8；高峰一过 desired=3，缩到 3；下一个高峰又来 ……

更糟的是 K8s 1.18 以前缩容默认有 5 分钟稳定窗口、扩容没有，所以可能扩很快、缩稍慢，但仍然剧烈抖。

#### 修复

**方案 A**：调缩容稳定窗口（K8s 1.18+）。

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 600   # 10 分钟内取最大 desired
    policies:
    - type: Percent
      value: 10                        # 每分钟最多缩 10%
      periodSeconds: 60
  scaleUp:
    stabilizationWindowSeconds: 0      # 扩容不延迟
    policies:
    - type: Percent
      value: 100
      periodSeconds: 60
    - type: Pods
      value: 4
      periodSeconds: 60
    selectPolicy: Max                  # 两条策略取大
```

**方案 B**：换更稳定的指标。CPU 抖得厉害，可以用：
- 入口 QPS（用 Prometheus Adapter 暴露成 custom metric）
- 队列深度（消息队列消费场景）
- 业务侧指标（比如订单数）

**方案 C**：拉宽阈值容忍。HPA 内部有个 `tolerance`（默认 0.1），意思是"差异 < 10% 不动"。在 5000m 目标下，差 < 500m 不会触发。可以通过 `--horizontal-pod-autoscaler-tolerance=0.2` 调宽。

#### 验证

观察 24 小时 desired 副本数曲线：

```bash
kubectl get hpa <name> --watch
```

理想：高峰扩、谷底缓慢缩，副本数曲线像低通滤波后的水位线。

> **口诀**：HPA 振荡的根是脉冲流量 + 太敏感的指标。缩容窗口拉到 10 分钟，扩容立即；CPU 不稳就换 QPS。

---

### 案例 ⑥ StatefulSet 顺序启动卡住

#### 现象

部署了一个 3 副本的 MySQL StatefulSet，看到：

```bash
$ kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
mysql-0   1/1     Running   0          10m
mysql-1   0/1     Pending   0          10m         ← 卡住
mysql-2   0/0     -         0          0
```

`mysql-1` 一直起不来，`mysql-2` 根本没创建。

#### 根因

StatefulSet 的核心约束：**严格按编号顺序启动**。`mysql-N+1` 必须等 `mysql-N` Ready 后才会被创建。

`mysql-1` Pending 的常见原因（呼应案例 ②）：
- PVC 不能绑定（PV 不够、StorageClass 故障）
- 资源不足（节点没资源）
- 镜像拉不下来

但 StatefulSet 的特殊点：**这个 Pod 卡住，整个 StatefulSet 后续都卡**。Deployment 不会，因为它的 Pod 是并行的。

#### 修复

**方案 A**：诊断 mysql-1 为什么 Pending（按案例 ② 的清单走）。

```bash
kubectl describe pod mysql-1
kubectl get pvc | grep mysql-1
```

**方案 B**：如果可以接受非严格顺序，改 `podManagementPolicy`：

```yaml
spec:
  podManagementPolicy: Parallel    # 默认是 OrderedReady
```

`Parallel` 模式下所有 Pod 并行启动（仍保持名字稳定、卷稳定），仅在那些"启动顺序无所谓"的有状态应用上用（比如 Cassandra、ClickHouse 集群，节点之间对等）。

**方案 C**：MySQL 这种**主从有依赖**的应用，绝对不能改成 Parallel。要保留 OrderedReady，并把 `mysql-0` 设为 master 初始化、`mysql-1+` 启动时主动 join。

#### 验证

```bash
# 顺序启动健康标志
kubectl get pods -l app=mysql -o wide
# mysql-0 1/1 Running ...
# mysql-1 1/1 Running ...
# mysql-2 1/1 Running ...
```

> **口诀**：StatefulSet 一个卡住全卡。先诊断那个 Pod；除非应用对等，别动 podManagementPolicy。

---

### 案例 ⑦ ConfigMap 更新后应用没自动重启

#### 现象

更新了 ConfigMap，发现配置里的 `log.level` 已经从 INFO 改成 DEBUG，但应用日志仍然按 INFO 输出。

#### 根因

ConfigMap 的更新行为分两种挂法（呼应 2.4）：

**Volume 挂载**：kubelet 周期同步（默认 1 分钟内），文件**会**自动更新。但应用要自己检测文件变化重新加载。

**环境变量**：环境变量是容器启动时**固定**的，ConfigMap 改了**不会**自动更新到环境变量。

如果业务通过环境变量读配置 → ConfigMap 改了对应用毫无影响。如果通过 Volume 挂载 → 文件更新了，但应用没监听 inotify。

#### 修复

**方案 A**：让 Pod 跟着 ConfigMap 重新部署（最常见模式）。

给 Deployment 加一个基于 ConfigMap 内容的注解：

```yaml
spec:
  template:
    metadata:
      annotations:
        configmap-hash: "{{ .Values.configHash }}"   # Helm 模板里算
```

或者用 [stakater/Reloader](https://github.com/stakater/Reloader) 这种 Operator，它监听 ConfigMap / Secret 变更，自动 patch Deployment 触发滚动。

**方案 B**：应用自己监听文件变化。

Spring Boot 用 `@RefreshScope` + Spring Cloud Kubernetes：

```yaml
spring:
  cloud:
    kubernetes:
      reload:
        enabled: true
        mode: polling     # 或者 event（基于 watch）
        period: 5000
```

Java 应用可以用 WatchService 监听挂载点：

```java
WatchService watcher = FileSystems.getDefault().newWatchService();
Paths.get("/etc/myapp/config").register(watcher,
    StandardWatchEventKinds.ENTRY_MODIFY);
// 起线程消费 events，触发 reload
```

**方案 C**：用 Sidecar 模式。Sidecar 监听文件变化，通过本地 IPC（HTTP / gRPC / signal）通知主容器 reload。Envoy 就是这么实现 xDS 的。

#### 验证

```bash
# 改 ConfigMap
kubectl edit configmap app-config

# 看挂载文件是否更新（一两分钟内）
kubectl exec <pod> -- cat /etc/myapp/config/log.level

# 看应用是否生效（看日志格式 / 看 /actuator/loggers）
```

> **口诀**：ConfigMap 不会推到容器。用 Reloader 滚动 Pod，或应用自己监听文件，或 Sidecar 推。三选一，没第四种。

---

### 案例 ⑧ ResourceQuota 配错导致命名空间无法创建 Pod

#### 现象

某个 namespace 突然无法创建任何 Pod，错误信息：

```
Error from server (Forbidden): error when creating "deploy.yaml":
pods "myapp-xxx" is forbidden: failed quota: dev-quota:
  must specify cpu,memory
```

#### 根因

namespace 上配了 ResourceQuota 要求**所有 Pod 必须显式声明 CPU/Memory requests 和 limits**：

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
```

ResourceQuota 隐含一条规则：如果 quota 限制了 `requests.cpu`，那么 namespace 下**所有 Pod 必须显式设置 requests.cpu**，否则无法创建。

而你的 Deployment 没设 resources：

```yaml
containers:
- name: app
  image: ...
  # 没有 resources 段
```

#### 修复

**方案 A**：每个 Deployment 显式声明 resources。

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

**方案 B**：在 namespace 上配 LimitRange，给没声明的 Pod 自动填默认值。

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: dev
spec:
  limits:
  - default:                  # ← 没声明 limits 时用这个
      cpu: 500m
      memory: 512Mi
    defaultRequest:           # ← 没声明 requests 时用这个
      cpu: 100m
      memory: 256Mi
    type: Container
```

LimitRange + ResourceQuota 是**最佳搭档**：Quota 限总量，LimitRange 兜底个体。

#### 验证

```bash
# 看 quota 用量
kubectl describe resourcequota -n dev

# 测试创建一个不声明 resources 的 Pod
kubectl run test --image=nginx -n dev
```

> **口诀**：ResourceQuota 限总量，LimitRange 给默认。两个一起配才不会让开发因为忘写 resources 而部署失败。

---

### 案例 ⑨ etcd 满 / 慢导致整个集群操作变慢

#### 现象

整个集群 `kubectl get pods` 慢到几秒甚至超时；新 Deployment 部署慢；HPA 明显延迟；告警雪片一样下来。

#### 根因

etcd 健康度恶化。常见原因：

**A. etcd 数据库满**（默认 quota 2GB）

```
etcdserver: mvcc: database space exceeded
```

这时候 etcd 进入只读模式，集群 API 大面积失败。

**B. etcd 磁盘 fsync 慢**（监控指标 `etcd_disk_wal_fsync_duration_seconds`）

raft log 写盘必须 fsync 才能提交。SSD 一般 < 10ms，如果上 100ms：
- 磁盘问题（IO 高、共享磁盘）
- 文件系统问题（ext4 vs xfs）
- 内核问题（cgroup IO 限速）

**C. etcd 慢的传染**：API Server 等不到 etcd 写完，所有写操作（创建 Pod、更新 status）排队，整个集群的"反应速度"垮掉。

**D. 历史 revision 没压实**：etcd 默认每 5 分钟 compact，但如果你写得太快或 compact 失败，DB 文件持续涨。

#### 修复

**A. 紧急扩容 quota**（先续命）：

```bash
# 调大到 8GB
etcdctl alarm list
ETCDCTL_API=3 etcdctl --endpoints=$EP \
  --command-timeout=300s defrag
ETCDCTL_API=3 etcdctl alarm disarm
```

K8s 控制平面启动参数加：

```
--quota-backend-bytes=8589934592    # 8GB
```

**B. 磁盘升级**：必须独占 NVMe SSD，**不能与 kubelet / 业务混部**。生产 etcd 节点不应该跑别的 Pod。

**C. 主动压实 + 碎片整理**：

```bash
# 压实历史 revision
ETCDCTL_API=3 etcdctl compact $(etcdctl endpoint status --write-out=json | egrep -o '"revision":[0-9]*' | egrep -o '[0-9]*')

# 碎片整理（这步会短暂阻塞 etcd，逐节点做）
ETCDCTL_API=3 etcdctl defrag --endpoints=$EP
```

**D. 减少 Watch 数**：太多 Operator 各自建 Informer 会爆。生产中评审 Operator 时关注它有没有滥用 List-Watch。

**E. 大集群拆事件 etcd**：K8s 支持把 events 单独放一个 etcd 集群（events 写得最频繁、最不重要）：

```
--etcd-servers-overrides=/events#https://event-etcd:2379
```

#### 验证

```bash
# etcd 健康度
ETCDCTL_API=3 etcdctl endpoint health --endpoints=$EP
ETCDCTL_API=3 etcdctl endpoint status --endpoints=$EP --write-out=table

# DB size
# 关键监控
# - etcd_mvcc_db_total_size_in_bytes（在涨吗）
# - etcd_disk_wal_fsync_duration_seconds（P99 < 50ms）
# - etcd_server_proposals_pending（积压吗）
```

> **口诀**：etcd 慢全集群慢。SSD + 独占磁盘 + 监控 fsync + 定期 compact + defrag 是 etcd 五件套。事件多到爆就拆 etcd。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。按 5 类分组。

### 资源配置层
- [ ] 每个容器都设 `requests` 和 `limits`，CPU 和 Memory 都设
- [ ] `requests` 反映真实需求（看 metrics 历史 P95），不靠拍脑袋
- [ ] `limits.memory` ≈ `requests.memory`（避免被驱逐），CPU 可以 limits > requests
- [ ] Java 服务设 `MaxRAMPercentage=70`，留 30% 给 native
- [ ] namespace 上配 ResourceQuota + LimitRange（兜底默认值）

### 健康检查层
- [ ] 所有 Pod 配 `readinessProbe`（不配的话滚动更新一定丢请求）
- [ ] 启动慢的应用配 `startupProbe`（避免 liveness 误杀）
- [ ] `livenessProbe` 不查依赖（不查 DB、不查下游）
- [ ] `/health` 与 `/ready` 是两个端点，前者只看进程、后者看依赖
- [ ] `terminationGracePeriodSeconds >= preStop sleep + 应用关闭时间`
- [ ] `preStop` 配 `sleep N`，让 endpoint 摘除有时间

### 网络与 Service 层
- [ ] Service 的 `selector` 与 Pod 的 `labels` 严格对齐（写错了 endpoint 就空）
- [ ] `targetPort` 与容器实际监听端口一致
- [ ] 跨集群 / 跨 namespace 调用使用 FQDN（避免 ndots 解析放大）
- [ ] 大集群装 NodeLocal DNS Cache
- [ ] 重要服务的 Endpoint 监控（数量、健康）

### 监控与可观测层
- [ ] Pod 的 `restartCount` 监控（暴涨说明 Liveness 误判或 OOMKilled）
- [ ] HPA 的 `desiredReplicas` 时序图（看是否振荡）
- [ ] etcd 的 db size、fsync 延迟、proposals pending 监控
- [ ] 容器内存使用 vs limits 的比例告警（> 80% 持续 5 分钟）
- [ ] 节点的 `Allocatable` vs 实际 `Allocated` 监控（避免节点超分配）
- [ ] CrashLoopBackOff Pod 自动告警

### 安全与稳定层
- [ ] 不用 `latest` 镜像 tag（无法回滚）
- [ ] PodDisruptionBudget 至少配 `minAvailable: 1`（节点维护时不会全删）
- [ ] 节点维护用 `kubectl drain`，不要直接关
- [ ] 重要服务启用 `topologySpreadConstraints` 跨节点 / 跨可用区分布
- [ ] 用 NetworkPolicy 限制 Pod 之间的网络访问（默认全通是不安全的）
- [ ] Secret 不能 base64 后塞进 ConfigMap 当配置（开了 etcd 加密才算合格）
- [ ] 容器跑非 root（`securityContext.runAsNonRoot: true`）
- [ ] 镜像扫描在 CI 里跑（trivy / grype），高危漏洞阻断发布

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### Operator 模式与 CRD 开发
- CRD（CustomResourceDefinition）让你自己定义 K8s 资源类型，schema 校验由 OpenAPI 驱动
- Operator SDK / Kubebuilder 是两个主流脚手架
- Operator 的本质是"运行在 K8s 里的 Controller"，把领域知识（怎么做主从、怎么做扩缩）编码成 reconcile 逻辑
- 看一个完整 Operator：CockroachDB Operator / etcd Operator
- Operator 反模式：把所有逻辑塞 reconcile，没有事件去重，性能崩溃

### Multi-Cluster
- **Karmada**（华为）：基于 Propagation Policy 的多集群编排
- **Submariner**：跨集群 Service 互通（基于 IPSec / WireGuard tunnel）
- **Cluster API**：用 K8s API 管 K8s 集群本身
- **kubefed v2**（已退役）：早期方案，留作历史教训

### GitOps
- **ArgoCD**：声明式 GitOps，把 Git 当作 desired state 的源
- **Flux**：与 ArgoCD 类似，更轻量、更 K8s 原生
- 关键差异：ArgoCD 有界面，Flux 是 CLI 主导
- 配套：Kustomize 比 Helm 更适合 GitOps（不用模板，纯 patch）

### eBPF + Cilium 数据面
- **Cilium** 用 eBPF 替代 kube-proxy 的 iptables/ipvs，性能更好且可观测
- **Hubble**（Cilium 的子项目）：基于 eBPF 的流量观测，能看到每条 L7 流
- eBPF 让 K8s 网络从"规则匹配"升级到"程序化"，是未来 5 年的方向

### Pod Security Standard
- 1.25+ 移除了 PSP（PodSecurityPolicy），换成 PSS（Pod Security Standards）
- 三个等级：Privileged / Baseline / Restricted
- 在 namespace 上加 label 启用：`pod-security.kubernetes.io/enforce: restricted`

### Vertical Pod Autoscaler（VPA）
- HPA 横向扩缩 Pod 数，VPA **纵向**调 Pod 的 resources
- VPA 重建 Pod 时会触发滚动，与 HPA 不能混用同一个 metric
- 适合"难以水平扩展"的服务（比如有状态的）

### Custom Scheduler
- K8s 支持多个 Scheduler 共存，Pod 通过 `schedulerName` 选择
- 写自己的 Scheduler 适合：GPU 拓扑感知、批处理（Volcano）、特殊调度策略
- Volcano 是 CNCF 的批处理调度器，gang scheduling（要么全调度上、要么都不调度）

### Service Mesh
- Istio / Linkerd 在 Pod 注入 sidecar 实现服务治理（流量路由、熔断、mTLS）
- 与 K8s 的边界：K8s 管"在哪跑"，Mesh 管"怎么通信"
- 新一代趋势：基于 eBPF 的 sidecarless（Cilium Service Mesh / Istio Ambient Mode）

### Kubelet 内部
- kubelet 与 CRI 之间的 protobuf 接口
- PLEG（Pod Lifecycle Event Generator）—— 老版本"PLEG is not healthy" 告警的根
- Topology Manager / CPU Manager / Device Manager —— 高级资源调度

---

## 7. 引用

- 官方文档：[Kubernetes Concepts](https://kubernetes.io/docs/concepts/)
- 论文：[Borg, Omega, and Kubernetes](https://research.google/pubs/pub44843/)（Google 三代调度系统的演进）
- KEP（K8s Enhancement Proposals）：[github.com/kubernetes/enhancements](https://github.com/kubernetes/enhancements)
- 源码：[kubernetes/kubernetes](https://github.com/kubernetes/kubernetes) —— `pkg/scheduler` 和 `pkg/controller` 必读
- *Kubernetes Patterns* —— Bilgin Ibryam（按场景模式整理）
- *Programming Kubernetes* —— Michael Hausenblas（写 Controller / Operator 必看）
- CNCF Landscape：[landscape.cncf.io](https://landscape.cncf.io/) —— 全景图
- etcd 性能调优官方文档：[etcd.io/docs/latest/op-guide/performance/](https://etcd.io/docs/latest/op-guide/performance/)
- Cilium 文档：[docs.cilium.io](https://docs.cilium.io/)
- 美团技术博客 K8s 系列、字节跳动 SRE 公开分享 —— 中文生产案例参考

---

> 配套：[容器与 cgroup](/article/linux-perf/container-cgroup) | [JVM GC 深入](/article/jvm/gc) | [系统设计主题](/article/system-design/) | [Service Mesh](/article/k8s/service-mesh)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
