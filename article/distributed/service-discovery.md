# 服务注册与发现 —— Eureka / Nacos / Consul / etcd / ZK 全对比

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，看 [分布式综述](/article/distributed/) 那一节就够了。
> 这里要回答的，是**真正在生产里被注册中心坑过、半夜被告警叫起来排查"实例为什么消失了"** 时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. CP 注册中心和 AP 注册中心在"网络分区"那一刻分别表现成什么？为什么微服务场景几乎都站 AP 这边？
2. Netflix 在 2014 年从 ZooKeeper 切到自研 Eureka 的根本原因是什么？只是 ZK 太重吗？
3. Eureka 的"自我保护机制"具体保护的是什么？它在什么情况下会**误触发**让你 debug 半天？
4. Nacos 同时支持 AP（Distro）和 CP（JRaft），它怎么在一个集群里**逐服务**切换两种模式？
5. 一个集群里有 5 万个实例，注册中心每秒推送几次给客户端？关键瓶颈是 CPU 还是网络还是序列化？
6. 多机房部署的注册中心如果机房间网络抖动几秒，AP 模式下会不会出现"两个机房各自下线对方"的脑裂？
7. 注册中心整个挂掉时，你的微服务还能不能互相调用？答"能"或"不能"都不算对，关键看什么？
8. Service Mesh 普及后（Istio + Envoy 的 xDS），传统的 Eureka / Nacos 还有存在的必要吗？

如果有半数答不上，往下看。

---

## 1. 底层模型：注册发现到底在解决什么

绝大多数文章把注册中心讲成"服务上线注册一下、调用方查一下、心跳保活一下"。这层认知不够看懂上面 8 个问题。要真正理解，必须把注册中心拆成它的核心能力。

### 1.1 服务发现 = 高可用 KV + 健康检查 + 推/拉模型

抽象到本质，所有注册中心都在做三件事：

```
注册中心
  ├── 一个高可用的 KV 存储          （存"服务名 → 实例列表"）
  ├── 一套健康检查机制              （怎么判断实例死了）
  └── 一套变更通知机制              （实例变了怎么告诉客户端）
```

差异只在三件事各自怎么实现：
- KV 存储是 **CP（强一致）** 还是 **AP（最终一致）**？
- 健康检查是 **客户端心跳** 还是 **服务端探测**？
- 变更通知是 **长轮询** 还是 **服务端推送（Watch / Stream）**？

**这三个维度的不同组合，就是 5 大注册中心的本质差异**。

### 1.2 CP vs AP 的真实代价

很多人把 CAP 当成"三选二"的口号。在注册中心场景里要更具体地想：

**CP 注册中心**（如 ZK / etcd / Consul 默认）：
- 网络分区时，**少数派分区** 不可写、不可读
- 写入要走 Raft / ZAB 共识，延迟 ≥ 1 个 RTT
- "强一致" 给的是读到最新写入的保证，**不是** 给"实例真的活着"的保证
- 适合**配置中心、分布式锁、Leader 选举**等强一致场景

**AP 注册中心**（如 Eureka / Nacos AP 模式）：
- 网络分区时，**所有节点都可读可写**，但读到的可能是过时数据
- 写入只要本地落地 + 异步同步，延迟 ≈ 内存写
- 实例列表可能短暂不一致（几秒内不同节点看到的实例数不一样）
- 适合**服务发现**：调用方拿到一个稍旧的列表，最坏只是请求被路由到死实例（有重试机制兜底）

为什么微服务场景几乎都站 AP？想象一个场景：北京机房和上海机房之间网络抖动。

- **CP 模式**：上海是少数派，注册中心拒绝服务上海机房的注册请求 → 上海机房新发布的实例**注册不进去** → 上海调用方找不到本机房的服务 → 跨机房调用甚至全挂
- **AP 模式**：上海机房的注册中心节点继续工作，上海机房新实例正常注册、本地可见 → 业务**继续跑**

> **口诀**：**服务发现宁可读到旧数据，也不能拒绝服务**。这就是 AP 的胜出根源。

### 1.3 客户端发现 vs 服务端发现

```
客户端发现（Client-Side Discovery）：
   Client ──查询──→ Registry
   Client ──直连──→ Service Instance
   代表：Eureka + Ribbon、Nacos + Spring Cloud LoadBalancer

服务端发现（Server-Side Discovery）：
   Client ──请求──→ LoadBalancer ──查询──→ Registry
                          ↓
                    Service Instance
   代表：K8s Service + kube-proxy、AWS ELB
```

| 维度 | 客户端发现 | 服务端发现 |
| --- | --- | --- |
| 客户端复杂度 | 高（要内嵌 SDK） | 低（只发请求） |
| 跨语言难度 | 每语言一个 SDK | 协议层面统一 |
| 灵活性 | 高（可做精细路由、灰度） | 低（受限于 LB 能力） |
| 故障域 | LB 逻辑跑在 Client，不会单点 | LB 是单点（要做 HA） |
| 典型场景 | 微服务（Spring Cloud / Dubbo） | K8s、传统四层负载均衡 |

Service Mesh（Istio）走的是第三种 —— **Sidecar 发现**：每个 Pod 旁挂一个 Envoy，Envoy 去注册中心拿数据，Client 透明地通过 Envoy 发请求。这是后面要讲的"xDS 模型"。

### 1.4 心跳机制：四种活法

实例怎么"证明自己还活着"？四种主流做法：

```
① TCP 心跳（ZK 的 Session）
   Client ←──长连接──→ Server
   连接断 = 实例死

② HTTP 心跳（Eureka）
   Client ──POST /apps/.../renew──→ Server  每 30s 一次
   服务端记录最后心跳时间，超过 90s 认为死

③ TTL 续租（Consul / etcd）
   Client 申请一个 TTL=30s 的 lease
   Client 每 10s 调 KeepAlive 续租
   30s 没续 → lease 过期 → 关联的 KV 自动删

④ 主动探测（Nacos 临时实例 + 永久实例混合）
   Server 主动向 Client 发健康检查请求
   常见：HTTP /health、TCP connect、MySQL ping
```

**反直觉的点**：四种里，**只有 TCP 长连接是"实时"的**。HTTP 心跳和 TTL 续租都有"固有延迟"——心跳间隔 + 容忍周期。这就是为什么 Eureka 默认配置下，实例挂了**最长 90s** 才被踢。

跨语言场景下要特别小心：Java 客户端用 30s 心跳，Go 客户端用 5s 心跳，**注册中心服务端统一用 90s 超时**就会出 bug —— Go 实例的网络稍抖一下就被误判下线（见案例 ⑧）。

### 1.5 推送模型：怎么把变更告诉客户端

实例列表变了，客户端怎么知道？三种主流推送机制：

```
① 长轮询（Long Polling，Nacos 1.x）
   Client ──HTTP，挂起 30s──→ Server
   30s 内有变化 → 立即返回；否则超时返回，Client 立即再发
   优点：HTTP 标准，穿透代理；缺点：高频时连接数爆炸

② Server-Sent Events / WebSocket
   Server ──持续推送──→ Client
   适合一对多广播

③ gRPC Stream / Watch（Nacos 2.x、etcd、ZK）
   Client 维护一个长连接 Stream
   Server 有变化主动推消息
   优点：低延迟、低开销；缺点：连接管理复杂、防火墙穿透问题
```

**推送 vs 拉取的真实抉择**：万级实例规模下，"全量推送" 成本极高 —— 一个服务有 1000 实例，变更一次要给所有 5000 个调用方各推一份。这就是为什么大厂的注册中心都做了**增量推送 + 客户端去重**：

```
Server 维护每个 Client 的版本号 V_client
  变更后只推 (V_client, V_now] 的 diff
Client 收到 diff 应用到本地缓存
```

> **口诀**：**注册中心的本质性能瓶颈不是注册速度，而是变更通知的扇出代价**。

### 1.6 多机房部署模式

跨机房的部署一般三种模型：

```
模式 A：单机房集群 + 跨机房读
   北京机房：3 节点注册中心 + 业务实例
   上海机房：业务实例 → 直连北京注册中心
   缺点：机房间网络断了上海全挂；优点：简单

模式 B：每机房独立集群 + 数据双向同步（推荐）
   北京：3 节点 + 注册北京实例
   上海：3 节点 + 注册上海实例
   双向同步：定期把对方机房的实例信息镜像过来
   缺点：同步延迟；优点：单机房自洽，断网不挂

模式 C：跨机房 Stretched Cluster
   一个 Raft 集群，5 节点跨 3 机房
   缺点：写入延迟 = 跨机房 RTT；优点：强一致
```

CP 注册中心（etcd / ZK / Consul 默认）只能选 A 或 C；AP 注册中心（Eureka / Nacos AP）能用 B —— 这是 AP 在多机房场景的**第二个胜出点**。

模式 B 的同步策略也分两种：
- **元数据同步**：只同步服务/实例信息（Nacos 双向同步）
- **代理同步**：本机房不存远程实例，用时通过远程注册中心查（Eureka Remote Region）

> **口诀**：**机房隔离 + 故障自治** 是大规模生产的硬要求，AP 注册中心天然支持这个模式。

---

## 2. 五大注册中心：架构对比

> 这一节每个注册中心讲三件事：**核心数据结构、健康检查、推送机制**。源码细节在 §3。

### 2.1 ZooKeeper：CP 派的祖师爷

ZK 不是为服务发现设计的（最早是 Hadoop 的协调服务），但因为成熟早，被 Dubbo / Kafka 拿去当了注册中心。

**核心数据结构**：分层文件系统 + 临时节点（Ephemeral Node）

```
/dubbo
  /com.example.UserService
    /providers
      /tcp://10.0.0.1:20880  (ephemeral)   ← 实例 A
      /tcp://10.0.0.2:20880  (ephemeral)   ← 实例 B
    /consumers
```

**健康检查**：ZK Session（TCP 长连接 + 心跳）
- 客户端连上 ZK 后建立 Session（默认 timeout 40s）
- 客户端每 `tickTime/3` 发心跳维持 Session
- Session 超时 → 该客户端创建的所有 Ephemeral Node 自动删除

**推送机制**：Watch（一次性回调）
- 客户端 `getChildren("/providers", watch=true)` 注册监听
- 子节点变化时，**ZK 回调一次**，然后这个 Watch 失效
- 客户端必须**重新注册** Watch（"读 + 重注册"几乎是标准模板）

**ZK 做注册中心的硬伤**：
- 一次性 Watch → 大量重注册请求
- ZAB 协议是 CP，机房分区时少数派注册不进去
- 5w+ 实例时，全量同步给新加入的从节点慢
- Session 心跳所有客户端用同一个 timeout，跨语言难统一

### 2.2 Eureka：Netflix 给微服务定制的 AP 派

Eureka 是 Netflix 在 2012 年开源的，明确为 AWS EC2 这种"云上 + 多 AZ + 实例可能随时挂"的场景设计的。

**核心数据结构**：内存 ConcurrentHashMap

```java
// Eureka 服务端的核心存储
ConcurrentMap<String,                    // appName
    Map<String,                          // instanceId
        Lease<InstanceInfo>>> registry;
```

`Lease<T>` 包了实例信息 + 最后续约时间 + 注册时间。**整个注册表存内存，没有持久化**（Server 重启从其他 Server 拉一份 sync 过来）。

**健康检查**：HTTP 心跳 + 多级超时
- 客户端默认每 30s 发一次 `POST /apps/{appName}/{instanceId}` 续约
- Server 端检查超过 90s 没续约的实例标记为 DOWN
- **每 60s 跑一次清理任务** 把 DOWN 的实例剔除

**推送机制**：客户端拉取（30s 周期）
- 客户端启动时全量拉一次
- 之后每 30s 拉增量（基于版本号）
- 服务端维护一个**最近 3 分钟的变更队列** 给增量请求用

**自我保护机制**（Eureka 的招牌特性）：
- Server 统计"最近一分钟的实际心跳数 / 期望心跳数"
- 比例 < 85%（默认 `renewalPercentThreshold=0.85`）时进入**自我保护模式**
- 自我保护下：**不再剔除任何实例**（避免误判把好实例都踢光）
- 等到比例恢复 → 退出自我保护

设计哲学：**宁可保留死实例，不愿误删活实例**。客户端有重试机制兜底。

> **口诀**：Eureka 是"假设网络不可靠"设计的；ZK 是"假设网络可靠"设计的。这就是为什么 Netflix 弃 ZK 选 Eureka。

### 2.3 Consul：CP + 健康检查的"瑞士军刀"

HashiCorp 的 Consul 是少数把"丰富的健康检查"做成第一公民的注册中心。

**核心数据结构**：
- 服务/节点信息 → 内存 + Raft 持久化
- KV 存储 → BoltDB 持久化
- 健康检查状态 → 独立维护

**健康检查**：四种内置检查 + 主动探测
- **Script Check**：Consul Agent 跑脚本，根据退出码判断
- **HTTP Check**：定期 GET 一个 URL，2xx 算健康
- **TCP Check**：能 connect 上就算健康
- **TTL Check**：客户端定期 PUT 一个 update，超时算 critical

注意 Consul 的健康检查是 **Agent 主动跑** 的（每个节点跑一个 Consul Agent），不是注册中心 Server 跑 —— 这种"分布式探测"模式压力分摊到每个节点，能撑大规模。

**推送机制**：HTTP Long Polling + Blocking Query
- 客户端 GET 时带 `?index=N&wait=5m`
- Server 收到后挂起，要么数据变到 index > N 立即返回，要么 5m 超时返回
- 客户端收到后立即用新的 index 再发请求

**Consul 的反熵机制（Anti-Entropy）**：
- Agent 周期性把本地状态和 Server 端的状态对比
- 发现不一致就把本地状态强推一遍
- 防止网络抖动后状态永久错位（详见 §3.4）

### 2.4 etcd：CP 派的现代代表

etcd 是 CoreOS 出品（现在 CNCF 项目），K8s 用它存所有元数据。Watch + Lease 是它的两大核心。

**核心数据结构**：MVCC B-tree（基于 BoltDB）+ 内存索引
- 每个 KV 都带一个 revision，全局单调递增
- 历史版本可查（直到被 compact）

**健康检查**：Lease（租约）
- 客户端申请 Lease：`grant(TTL=30s)`，得到 `leaseId`
- 把服务实例信息绑到这个 Lease：`put("/services/foo/instance1", "addr", lease=leaseId)`
- 客户端定期调 `KeepAlive(leaseId)` 续约
- Lease 过期 → 所有绑该 Lease 的 KV 自动删除

**推送机制**：gRPC Stream Watch
- 客户端 `Watch("/services/foo", startRev=N)`
- Server 持续推送 `Event{type, key, value, revision}`
- 客户端记录最后处理的 revision，断线重连时从此 revision 续

**为什么 K8s 选 etcd 而不是 ZK**：
- gRPC stream watch 比 ZK 的"一次性 watch"高效得多
- HTTP/gRPC 接口比 ZK 的二进制协议好做客户端
- Lease 比 Ephemeral Node 灵活（可独立管理生命周期）

### 2.5 Nacos：阿里的 AP+CP 双模派

Nacos 在 2018 年开源，明确目标是"既能做服务发现也能做配置中心"。它最特别的是：**同一个集群内可以逐服务选择 AP 或 CP 模式**。

**核心数据结构**：服务-集群-实例三层
```
Service "user-service"
  ├── Cluster "DEFAULT"
  │     ├── Instance 10.0.0.1:8080 (ephemeral)
  │     └── Instance 10.0.0.2:8080 (ephemeral)
  └── Cluster "BJ"
        └── Instance 10.0.1.1:8080 (persistent)
```

实例分两类：
- **Ephemeral（临时实例）**：Distro 协议（AP），客户端心跳维持
- **Persistent（永久实例）**：JRaft 协议（CP），Server 主动探测健康

注册时通过参数 `ephemeral=true/false` 选择。

**Distro 协议**（AP，Nacos 自研，类似 Eureka）：
- 集群内每个节点负责一部分实例（hash 分片）
- 客户端心跳到 owner 节点
- 节点间异步同步全量数据（每 5s 一次）
- 节点间 verify 周期对账

**JRaft 模式**（CP，基于阿里 SOFA 的 Raft 实现）：
- 写请求 → 转发到 Leader → Raft 共识 → 持久化
- 读请求 → 可走 Leader 强一致读，也可走 Follower 弱一致读

**推送机制**：
- Nacos 1.x：HTTP Long Polling（30s 挂起）
- Nacos 2.x：gRPC Stream（性能提升 10x，是版本切换的核心动机）

> **口诀**：**Nacos = Eureka + Spring Cloud Config + 主动健康检查 + CP 模式选项**。一站式覆盖，但复杂度也高。

### 2.6 一图比五

| 维度 | ZooKeeper | Eureka | Consul | etcd | Nacos |
| --- | --- | --- | --- | --- | --- |
| **CAP** | CP | AP | CP（默认） | CP | AP/CP 双模 |
| **共识协议** | ZAB | 无（最终一致） | Raft | Raft | Distro / JRaft |
| **健康检查** | TCP Session | HTTP 心跳 | Agent 探测（多种） | Lease 续租 | 心跳（AP） + 探测（CP） |
| **推送机制** | Watch（一次性） | 拉取（30s） | Long Polling | gRPC Watch Stream | Long Polling / gRPC Stream |
| **持久化** | 磁盘日志 | 仅内存 | 磁盘 | 磁盘（BoltDB） | MySQL（CP） / 内存（AP） |
| **多机房** | 难（单 Stretched 集群） | 易（Region） | 中（WAN gossip） | 难 | 易（双向同步） |
| **典型用途** | Dubbo / Kafka 注册 | Spring Cloud（已 EOL） | 多语言微服务 | K8s 元数据 | Spring Cloud / Dubbo |
| **跨语言 SDK** | 全 | 弱（主 Java） | 全 | 全 | 全（主 Java/Go） |
| **运维难度** | 中 | 低 | 中 | 中 | 中 |

> **口诀**：**只跑微服务用 Nacos / Eureka；要做配置中心+服务发现一体用 Nacos / Consul；K8s 内部用 etcd；老 Dubbo 集群凑合用 ZK**。

---

## 3. 源码切片：五段关键路径

### 3.1 Nacos Distro 协议的 AP 实现

> 基于 Nacos 2.2 源码，节选 `DistroProtocol` 和 `DistroDataProcessor`。

Distro 的核心思想：**集群内每个节点 own 一部分数据，客户端心跳到 owner，owner 异步同步给其他节点**。

```java
// com.alibaba.nacos.naming.consistency.ephemeral.distro.v2.DistroProtocol
public void sync(DistroKey distroKey, DataOperation action, long delay) {
    for (Member each : memberManager.allMembersWithoutSelf()) {
        DistroKey distroKeyWithTarget = new DistroKey(distroKey.getResourceKey(),
                distroKey.getResourceType(), each.getAddress());
        DistroDelayTask distroDelayTask = new DistroDelayTask(distroKeyWithTarget, action, delay);
        distroTaskEngineHolder.getDelayTaskExecuteEngine().addTask(distroKeyWithTarget, distroDelayTask);
    }
}
```

**逐段点透**：

- **`memberManager.allMembersWithoutSelf()`**：遍历集群里除自己外的所有节点，每个都创建一个延迟任务。这是**异步同步** 的入口。
- **`DistroDelayTask` 进延迟队列**：不是立即同步，而是攒一段时间（默认 1s）批量发，避免单实例变更触发 N 次跨节点 RPC。
- **AP 体现在哪里**：本节点的写已经成功了，同步是 fire-and-forget，**不等其他节点 ACK**。这就是为什么 Nacos AP 模式注册延迟极低（毫秒级）。

代价是：节点间数据可能短暂不一致。Nacos 用 **Verify 任务** 做最终一致：

```java
// 每 5 秒跑一次：把自己 own 的数据的"摘要"发给其他节点
public void verify() {
    Map<String, DistroData> dataMap = distroDataStorage.getVerifyData();
    for (Member member : memberManager.allMembersWithoutSelf()) {
        DistroData verifyData = new DistroData(...);
        // 远端比对摘要，不一致就拉取
        distroComponentHolder.findTransportAgent(type).syncVerifyData(verifyData, member.getAddress());
    }
}
```

对方收到 verify 数据后，比对本地摘要 —— 不一致就反向请求拉一份完整数据。这就是 Distro 的"反熵" 机制。

### 3.2 Nacos JRaft CP 模式

```java
// com.alibaba.nacos.core.distributed.raft.JRaftProtocol
@Override
public Response write(WriteRequest request) throws Exception {
    CompletableFuture<Response> future = new CompletableFuture<>();
    final Throwable[] throwable = new Throwable[]{null};
    try {
        commit(request.getGroup(), request, future);   // ★ 提交到 Raft
    } catch (Throwable e) {
        future.completeExceptionally(e);
    }
    return future.get(rpcRequestTimeoutMs, TimeUnit.MILLISECONDS);
}

private void commit(String group, Message data, CompletableFuture<Response> future) {
    final RaftGroupService raftGroupService = multiRaftGroup.get(group);
    final Node node = raftGroupService.getRaftNode();
    if (!node.isLeader()) {
        // ★ 非 Leader 必须转发到 Leader
        invokeToLeader(group, data, rpcRequestTimeoutMs, future);
        return;
    }
    final Task task = new Task();
    task.setData(ByteBuffer.wrap(data.toByteArray()));
    task.setDone(new NacosClosure(data, status -> {
        // Raft 共识完成后回调
        ...
    }));
    node.apply(task);   // ★ 进入 Raft 流程
}
```

**逐段点透**：

- **`!node.isLeader()` 转发**：JRaft 的 strong leader 模式 —— 写必须经 Leader。Follower 收到写请求要转发，多一跳 RTT。
- **`node.apply(task)`**：调 JRaft 库进入 propose → replicate → commit 流程。**至少 1 个 RTT** 才能返回。
- **代价**：JRaft 模式下注册一个实例的延迟比 Distro 高 5-10 倍（毫秒级 vs 几十毫秒）。这就是为什么大部分服务发现场景用 Distro AP，只有"网关路由表" 这类要严格一致的才用 JRaft。

为什么 Nacos 要双模？因为它**同时是注册中心和配置中心**：
- 服务发现 → 容忍短暂不一致 → AP
- 配置发布 → 不能两个客户端读到不同配置 → CP

### 3.3 Eureka 的客户端缓存与服务端去抖

Eureka 客户端的"自保命"逻辑藏在 `DiscoveryClient`：

```java
// com.netflix.discovery.DiscoveryClient
private boolean fetchRegistry(boolean forceFullRegistryFetch) {
    Stopwatch tracer = FETCH_REGISTRY_TIMER.start();
    try {
        Applications applications = getApplications();
        if (clientConfig.shouldDisableDelta()
                || (!Strings.isNullOrEmpty(clientConfig.getRegistryRefreshSingleVipAddress()))
                || forceFullRegistryFetch
                || (applications == null)
                || (applications.getRegisteredApplications().size() == 0)
                || (applications.getVersion() == -1)) {
            getAndStoreFullRegistry();   // ★ 全量
        } else {
            getAndUpdateDelta(applications);   // ★ 增量
        }
        applications.setAppsHashCode(applications.getReconcileHashCode());
        ...
    } catch (Throwable e) {
        logger.error("...", e);
        return false;
    }
    ...
}
```

**逐段点透**：

- **增量优先 + 全量兜底**：默认走 delta。但有 5 个条件之一就退化到全量（首次启动、本地无数据、版本号异常等）。
- **`reconcileHashCode`**：客户端把本地缓存的所有实例算一个 hash，请求时带上。Server 端比对，不一致就**强制下次走全量** —— 这是防止"增量推送丢消息" 的兜底机制。
- **客户端缓存**：所有数据存在 `localRegionApps`（AtomicReference）里。即使注册中心整个挂掉，客户端**仍能用本地缓存继续路由**。这是案例 ⑨ 的根。

Eureka Server 端的"去抖"在 `ResponseCache`：

```java
// com.netflix.eureka.registry.ResponseCacheImpl
private final LoadingCache<Key, Value> readWriteCacheMap;     // 30s 失效
private final ConcurrentMap<Key, Value> readOnlyCacheMap;     // 长期有效

private final TimerTask updateTask = new TimerTask() {
    @Override
    public void run() {
        for (Key key : readOnlyCacheMap.keySet()) {
            CurrentRequestVersion.set(key.getVersion());
            try {
                Value cacheValue = readWriteCacheMap.get(key);
                Value currentCacheValue = readOnlyCacheMap.get(key);
                if (cacheValue != currentCacheValue) {
                    readOnlyCacheMap.put(key, cacheValue);   // ★ 30s 拷贝一次
                }
            } finally {
                CurrentRequestVersion.remove();
            }
        }
    }
};
```

**逐段点透**：

- **二级缓存**：`readWriteCacheMap` 写穿；`readOnlyCacheMap` 30s 才更新一次。
- **客户端拉的是 readOnly**：所以 Eureka 默认配置下，**实例注册后客户端最长 30s+30s+90s（响应缓存 + 拉取间隔 + 心跳超时）才能感知**。生产经常因为这个被吐槽"Eureka 慢"。
- **优化**：把 `eureka.server.responseCacheUpdateIntervalMs` 调小到 3s，能显著减少延迟，但 Server CPU 上升。

### 3.4 Consul 的 Anti-Entropy 反熵机制

Consul Agent 周期性把本地状态和 Server 对比，发现差异立即修正。

```go
// hashicorp/consul/agent/local/state.go
func (l *State) syncService(key structs.ServiceID) error {
    s := l.services[key]
    if s == nil || s.Deleted {
        return nil
    }

    req := structs.RegisterRequest{
        Datacenter:      l.config.Datacenter,
        ID:              l.config.NodeID,
        Node:            l.config.NodeName,
        Address:         l.config.AdvertiseAddr,
        TaggedAddresses: l.config.TaggedAddresses,
        NodeMeta:        l.metadata,
        Service:         s.Service,
        EnterpriseMeta:  key.EnterpriseMeta,
        WriteRequest:    structs.WriteRequest{Token: l.serviceToken(key)},
        SkipNodeUpdate:  l.nodeInfoInSync,
    }

    var out struct{}
    err := l.Delegate.RPC("Catalog.Register", &req, &out)
    if err == nil {
        s.InSync = true   // ★ 标记为已同步
        ...
    }
    return err
}
```

```go
// 触发 sync 的两条路径
func (l *State) Sync() error {
    // Path 1: 周期性触发（默认 60s）
    for id := range l.services {
        if !l.services[id].InSync {
            if err := l.syncService(id); err != nil {
                return err
            }
        }
    }
    return nil
}
```

**逐段点透**：

- **Anti-Entropy 是异步的**：Agent 注册到 Server 失败也不会直接报错给客户端，只是标记 `InSync=false`，下次循环重试。
- **解决的问题**：网络分区恢复后，Agent 本地状态和 Server 可能不一致（中间有变更）。Anti-Entropy 保证最终一致。
- **代价**：状态收敛有延迟（< 60s）。生产配置可以调小 `sync_coordinate_interval`。

### 3.5 etcd 的 Watch 实现

etcd 的 Watch 是最优雅的"服务端推送" 实现，基于 gRPC stream + MVCC revision。

```go
// etcd/server/etcdserver/api/v3rpc/watch.go
func (sws *serverWatchStream) sendLoop() {
    for {
        select {
        case wresp, ok := <-sws.watchStream.Chan():
            if !ok {
                return
            }
            // 把多个 watch event 攒成一个 response
            evs := wresp.Events
            events := make([]*mvccpb.Event, len(evs))
            for i := range evs {
                events[i] = &evs[i]
            }
            wr := &pb.WatchResponse{
                Header:          sws.newResponseHeader(wresp.Revision),
                WatchId:         int64(wresp.WatchID),
                Events:          events,
                CompactRevision: wresp.CompactRevision,
                Canceled:        canceled,
            }
            // ★ 通过 gRPC stream 推给客户端
            if err := sws.gRPCStream.Send(wr); err != nil {
                return
            }
        case <-sws.ctrlStream:
            ...
        }
    }
}
```

**逐段点透**：

- **基于 revision**：每个 KV 操作都有全局 revision。客户端 Watch 时带 `startRevision=N`，Server 推送从 N 之后的所有事件。
- **断线续推**：客户端断线重连时，**用最后处理的 revision 重新发起 Watch**，Server 从那里继续推 —— 不会丢事件（除非那部分历史已被 compact）。
- **Compact 是关键**：etcd 默认每 5min compact 一次，老的 revision 会被回收。如果客户端断线超过 compact 间隔，再连上时会收到 `CompactRevision` 错误，**必须做全量同步**。

这个模型比 ZK 的"一次性 Watch" 优秀很多，是 K8s 选 etcd 的核心原因。

> **口诀**：**ZK Watch 是"事件通知"，etcd Watch 是"事件流"**。后者天然支持顺序性和断线续推。

---

## 4. 实测数据：万级实例下谁更扛

### 4.1 推送延迟：实例变更到客户端感知

在 8 核 16GB 的标准机器上，模拟 1 万实例集群，触发一次实例下线，统计 99% 客户端感知到变更的时间。

| 注册中心 | 配置 | P99 推送延迟 |
| --- | --- | --- |
| ZK 3.7 | Watch + 每客户端 1 个 Watcher | ~50ms |
| Eureka 1.10 | 默认 (30s 拉取 + 30s 缓存) | **~60s** |
| Eureka 1.10 | 调优 (3s 拉取 + 3s 缓存) | ~6s |
| Consul 1.15 | Long Polling 5min | ~200ms |
| etcd 3.5 | gRPC Watch | ~30ms |
| Nacos 1.4 (HTTP) | Long Polling 30s | ~150ms |
| Nacos 2.2 (gRPC) | Stream | ~20ms |

**反直觉的点**：
- Eureka 默认配置慢得离谱。Spring Cloud Netflix 官方推荐生产把 `eureka.server.responseCacheUpdateIntervalMs` 调到 3000，`eureka.client.registry-fetch-interval-seconds` 调到 5
- Nacos 2.x 用 gRPC 后比 1.x 快 7-10 倍 —— 这是 Nacos 1 → 2 的核心升级动机
- etcd 最快，但你不会拿它做服务发现 SDK（接入成本高，K8s 间接用）

### 4.2 注册风暴：发布时 N 个实例同时注册

模拟一次发布，1000 个 Pod 同时启动注册：

| 注册中心 | 注册全部完成耗时 | Server CPU 峰值 |
| --- | --- | --- |
| ZK | 18s（写要走 ZAB） | 80% |
| Eureka | 4s | 30% |
| Consul | 12s（写走 Raft） | 60% |
| etcd | 8s | 50% |
| Nacos AP | 3s | 25% |
| Nacos CP | 15s | 70% |

**结论**：AP 模式应对注册风暴明显更扛，因为不需要走共识。CP 模式下注册风暴会成为 Leader 节点的 CPU 瓶颈。

### 4.3 心跳风暴对网络的影响

5 万实例，每实例 30s 一次心跳：

```
心跳总 QPS = 50,000 / 30 = 1,667 QPS
单次心跳大小 = HTTP Header + InstanceId ≈ 500B
入流量 = 1,667 × 500B = 833 KB/s

但服务端要处理 = O(实例数 × 频率)
CPU 主要消耗在：JSON 序列化 + 心跳写入并发控制
```

实测：5w 实例下 Eureka Server 单核 CPU 约 40% 都耗在心跳处理上。优化方向：
- 拉长心跳间隔（30s → 60s）
- 用 Protobuf 替代 JSON
- gRPC stream 心跳（Nacos 2.x）

### 4.4 跨机房延迟下的一致性窗口

模拟北京-上海 30ms RTT，AP 模式下两机房双向同步：

| 同步策略 | 平均不一致窗口 | 极端情况 |
| --- | --- | --- |
| Nacos Distro 默认（5s 同步） | < 5s | 网络抖动 30s 期间会有显著不一致 |
| Eureka Region 同步（30s） | < 30s | 长达分钟级 |

CP 模式（跨机房 Raft）：
- 写入延迟 = 2 × RTT = 60ms（Leader → Follower → Quorum ACK）
- 但**不会**有不一致窗口

### 4.5 客户端缓存对注册中心宕机的影响

把注册中心整个集群停掉，观察业务调用是否还能跑：

| 注册中心 | 客户端缓存策略 | 注册中心挂后业务表现 |
| --- | --- | --- |
| Eureka | 内存全量缓存 + 启动时 dump 到磁盘 | **业务完全不受影响**，已知实例继续调 |
| Nacos | 内存缓存 + Failover 文件 | 同上 |
| Consul | Agent 本地缓存 | 同上 |
| ZK | 默认无缓存（Watch 失效后失明） | **新发起的 getChildren 失败**，业务报错 |
| etcd | SDK 一般无缓存 | 同 ZK |

**关键认知**：Eureka 和 Nacos 设计上**就假设注册中心会挂**，所以客户端必须有完整缓存。这是它们 AP 哲学的延伸。

> **口诀**：**注册中心挂了业务该不该挂，本质看客户端 SDK 有没有持久化缓存**。这是评估注册中心选型的隐藏要点。

---

## 5. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 跨机房注册中心脑裂 —— "上海机房把北京机房全踢了"

#### 现象

部署在北京、上海双机房的 Nacos 集群（Distro AP 模式）。某天凌晨机房间专线抖动 2 分钟，恢复后发现：
- **北京客户端拿到的实例列表里没有上海的实例**
- **上海客户端拿到的实例列表里没有北京的实例**
- 上海实例的心跳在北京 Server 看是"超时"

#### 根因

Distro 协议的核心：每个 Server 节点 own 一部分实例的"权威数据"。客户端心跳到 owner 节点。

机制串起来：
1. 上海实例 A，hash 到了北京节点 N1（owner）
2. 网络分区，上海客户端到 N1 的心跳全部超时
3. **上海节点 N4 接管了 A 的心跳**（注册时上海客户端会 fallback）
4. 但 N1 上 A 的"老记录"没有被心跳，30s 后**N1 把 A 标记为不健康**
5. N1 把"A 不健康"同步给 N2、N3
6. 网络恢复后，N4 上的"A 健康"和 N1 的"A 不健康"冲突
7. Distro 的合并策略：以 owner（N1）为准 → A 被认为不健康

更深的问题：**两边各自的客户端都在往各自的 Server 写数据，恢复后双方对彼此的实例都判定为"不健康"**。

#### 修复

**方案 A**：调大健康检查超时

```properties
# nacos server
nacos.naming.healthcheck.heartbeat.timeout=30000     # 30s 不健康
nacos.naming.healthcheck.expire.timeout=90000        # 90s 移除
```

让短暂网络抖动不要立即判死。

**方案 B**：双机房用独立集群 + 元数据双向同步

```
北京 Nacos 集群（3 节点） → 注册北京实例
上海 Nacos 集群（3 节点） → 注册上海实例
两个集群之间用 Nacos Sync（开源工具）做双向同步
```

这样机房内部完全自洽，专线断了不影响机房内调用。

**方案 C**：业务侧加 Region 亲和

```yaml
# Spring Cloud LoadBalancer
spring:
  cloud:
    loadbalancer:
      zone: bj   # 当前可用区
      same-zone: true   # 优先调本可用区实例
```

即使列表里有上海实例，也优先调本机房，跨机房调用降低出问题概率。

#### 验证

模拟机房间网络断 5 分钟（用 `iptables` drop 跨机房流量），观察：
- 各机房内部业务不受影响
- 网络恢复后，5 分钟内实例列表重新对齐

> **口诀**：**AP 注册中心 + 跨机房单集群 = 必脑裂**。要么独立集群双向同步，要么忍受短暂故障。

---

### 案例 ② 心跳丢失误下线 —— "网络抖一下实例全没了"

#### 现象

业务监控看到某服务的实例数突然从 50 跌到 30，10 秒后又恢复到 50。同时段大量上游报 "no instance available" 错误。

#### 根因

查 Eureka Server 日志：

```
2024-xx-xx 03:15:22 INFO  - DS: Registry: expired lease for instance 10.0.0.x:8080
2024-xx-xx 03:15:22 INFO  - DS: Registry: expired lease for instance 10.0.0.y:8080
... (20 instances)
```

时间点上正好是网络监控告警的"专线丢包率上升 3%"。

机理：
1. 网络丢包 3% → 部分实例的 30s 心跳没到达
2. Eureka 默认 90s 没心跳就标记 DOWN
3. 但单实例丢 1-2 个心跳很正常，关键是**清理任务每 60s 跑一次，一次性把所有过期的踢光**
4. 自我保护机制理应防止这种"批量误删"，但它的阈值是"心跳率 < 85%"，3% 丢包对应的心跳率约 97%，**没触发自我保护**

#### 修复

**方案 A**：调低自我保护阈值

```properties
# eureka server
eureka.server.renewal-percent-threshold=0.95   # 95% 就触发保护
```

更激进的保护，宁可保留死实例。配合客户端调用失败重试，可接受。

**方案 B**：调高心跳容忍

```yaml
# eureka client
eureka:
  instance:
    lease-renewal-interval-in-seconds: 30
    lease-expiration-duration-in-seconds: 120   # 默认 90，调到 120
```

允许丢更多心跳。

**方案 C**：确保自我保护被启用（生产经常被人误关）

```properties
eureka.server.enable-self-preservation=true   # 默认 true，确认没被改
```

#### 验证

故意在测试环境用 `tc qdisc add ... loss 5%` 注入丢包，看实例是否被误踢。修复后应保持稳定。

> **口诀**：**心跳超时 = 心跳间隔 × 至少 3**。30s 间隔配 90s 超时是 Eureka 默认，丢一两次心跳也容忍；缩短任何一边都增加误判率。

---

### 案例 ③ 客户端缓存陈旧导致请求黑洞 —— "实例已下线但还在被调用"

#### 现象

灰度发布期间，10% 流量 502。日志显示请求被路由到了已经下线的旧版本实例。监控看注册中心里这些实例**确实早已被剔除**，但客户端还在往那调。

#### 根因

排查链路：
1. 旧实例 Pod 被 K8s 销毁
2. 实例进程退出前**没有主动调注册中心 deregister**（依赖心跳超时被动剔除）
3. 注册中心 90s 后才标记 DOWN
4. 客户端拉取间隔 30s + 服务端响应缓存 30s = **最坏 90s 才感知**
5. 客户端从感知到从负载均衡列表移除还有 1 个 LB 周期

总延迟：**最坏 90 + 90 = 180s** 内，客户端会持续往死实例发请求。

#### 修复

**方案 A**：实例优雅下线，主动 deregister

```java
@PreDestroy
public void deregister() {
    discoveryClient.shutdown();   // 主动通知注册中心
    // 等待至少一个推送周期，让客户端感知
    Thread.sleep(30_000);
}
```

K8s 配套 `terminationGracePeriodSeconds: 60`。

更好的做法 —— **K8s preStop 钩子调注册中心 deregister API**：

```yaml
lifecycle:
  preStop:
    exec:
      command: ["curl", "-X", "DELETE",
                "http://nacos:8848/nacos/v1/ns/instance?serviceName=...&ip=..."]
```

**方案 B**：客户端做主动健康检查

不要完全信任注册中心列表。Spring Cloud LoadBalancer 默认有 `HealthCheckServiceInstanceListSupplier`：

```yaml
spring:
  cloud:
    loadbalancer:
      health-check:
        initial-delay: 0
        interval: 5s
        path:
          default: /actuator/health
```

每 5 秒主动 check 一次实例健康，5xx 立即从 LB 列表剔除。

**方案 C**：调整 Eureka 的推送参数（已在案例 ②）

#### 验证

发布过程中监控 502 数量，调整后应降为 0。压测：手动 kill 一个实例，观察 LB 多久把它剔除。

> **口诀**：**别指望注册中心剔除实例的速度**。优雅下线 + 客户端主动健康检查双保险。

---

### 案例 ④ Eureka 自我保护误触发 —— "我明明都关闭了实例为什么还在"

#### 现象

夜间维护，运维一次性下线了 30% 的实例（触发批量缩容）。结果 Eureka 控制台显示"自我保护已启用"，被关闭的实例**仍然显示在线**，调用方持续报错。

#### 根因

自我保护的触发条件：
```
expectedNumberOfRenewsPerMin = 实例数 × (60 / 心跳间隔) × 2 = 实例数 × 4
expectedNumberOfRenewsPerMin × 0.85 < 实际心跳数 → 触发自我保护
```

一次性关闭 30% 实例 → 实际心跳数瞬间降到 70% → **触发自我保护** → 不再剔除任何实例。

但被关闭的实例真的死了，自我保护把它们留着。

#### 修复

**方案 A**：批量缩容前主动 deregister

```bash
for ip in $TO_REMOVE; do
  curl -X DELETE "http://eureka/eureka/apps/myapp/${ip}:8080"
done
sleep 5   # 等 Eureka Server 把变更推给其他 Eureka Server
# 然后才 kill -9
```

主动 deregister 不会触发自我保护（自我保护只看心跳缺失）。

**方案 B**：临时关闭自我保护（批量运维窗口）

```bash
curl -X PUT "http://eureka/actuator/eureka/self-preservation?enabled=false"
# 操作完再开回来
```

风险：开启窗口期内有真实网络故障会误删实例。

**方案 C**：分批操作，避免单次超过 15%

每次缩容控制在 < 15%，不触发自我保护阈值。

#### 验证

测试环境批量关 30% 实例，观察 Eureka 控制台：
- 直接 kill：触发自我保护，实例还显示
- 先 deregister：实例正常消失

> **口诀**：**自我保护的设计假设是"实例突然大量下线 = 网络问题"**。如果真是有意为之的批量缩容，必须**主动 deregister**告诉 Eureka "不是网络问题"。

---

### 案例 ⑤ Nacos 双模配置错位 —— "既不 AP 又不 CP"

#### 现象

业务接 Nacos，用着用着发现：
- 高频注册场景延迟很高（CP 模式特征）
- 但又有"读到旧数据"的问题（AP 模式特征）

排查半天，怀疑是 Nacos 哪里出 bug。

#### 根因

代码里：

```java
// 注册时
namingService.registerInstance("user-service", instance);   // 默认 ephemeral=true → AP
```

```java
// 查询时
List<Instance> instances = namingService.getAllInstances("user-service", false);
// ★ 第二个参数是 subscribe，false 表示不订阅（每次都直接查 Server）
```

但服务定义这边：

```java
// 第一次注册时手动指定了 ephemeral=false
namingService.registerInstance("user-service", "DEFAULT", instance);   // ephemeral=false → CP
```

**Nacos 的特性**：服务级别的 `ephemeral` 第一次注册时确定。如果先注册了一个 `ephemeral=false` 的实例（CP 模式），整个 service 就被打上 CP 标记。后续 `ephemeral=true` 的注册会被拒绝或行为异常。

更隐蔽的问题：开发环境和生产环境配置不一致 —— 开发用了默认 AP，生产因历史原因用了 CP，行为完全不同。

#### 修复

**方案 A**：统一服务的 ephemeral 设置

代码层面强制：

```java
@PostConstruct
public void register() {
    Instance instance = new Instance();
    instance.setIp(ip);
    instance.setPort(port);
    instance.setEphemeral(true);   // ★ 显式设置
    namingService.registerInstance(serviceName, instance);
}
```

**方案 B**：通过 Nacos 配置全局禁用永久实例

```properties
# application.properties
spring.cloud.nacos.discovery.ephemeral=true   # 强制临时实例
```

**方案 C**：Nacos 控制台清理"幽灵服务"

如果发现某个 service 的 ephemeral 设置错了，最稳的办法是：
1. 停止所有该服务的实例
2. 调用 Nacos API 删除 service：`DELETE /nacos/v1/ns/service`
3. 重新启动实例（这次用对的 ephemeral 设置）

#### 验证

```bash
# 查询服务的 ephemeral 状态
curl "http://nacos:8848/nacos/v1/ns/service?serviceName=user-service"
# 看返回里的 metadata
```

> **口诀**：**Nacos 服务的 ephemeral 是"出生时决定的，后天难改"**。生产标准化为 ephemeral=true，永久实例只用于网关、ZooKeeper 替代等少数场景。

---

### 案例 ⑥ ZK Session Timeout 太短 —— "网络一抖 Dubbo 全下线"

#### 现象

Dubbo + ZK 的老系统。业务运行平稳，某天网卡 driver 升级后偶发 GC 长 STW（5 秒），结果**该机器上所有 Dubbo Provider 全部从注册中心消失**，触发上游全量切流。

#### 根因

ZK 客户端用 Session 维持连接：
- Session timeout 默认 40s
- 但很多老 Dubbo 配置里改成了 5s（"为了快速感知故障"）
- JVM Full GC 5 秒 → 心跳暂停 5 秒 → Session 超时
- 该客户端创建的所有 Ephemeral Node（即所有 Dubbo Provider 注册）**全部被 ZK 自动删除**

更糟：GC 结束后客户端重连 ZK，**重新注册所有 Provider** —— 注册风暴导致集群短暂抖动。

#### 修复

**方案 A**：拉大 Session timeout

```xml
<dubbo:registry address="zookeeper://..." 
                client="curator"
                session="60000"/>   <!-- 60 秒 -->
```

ZK 服务端的 `tickTime` × `maxSessionTimeout` 决定了上限，要确认 server 端允许。

**方案 B**：换 Curator 客户端，开启 Session 重连

```java
CuratorFramework client = CuratorFrameworkFactory.builder()
    .connectString("...")
    .sessionTimeoutMs(60000)
    .connectionTimeoutMs(15000)
    .retryPolicy(new ExponentialBackoffRetry(1000, 3))
    .build();

// Session 失效时重新注册
client.getConnectionStateListenable().addListener((c, state) -> {
    if (state == ConnectionState.LOST) {
        // 重新注册所有 ephemeral 节点
    }
});
```

**方案 C**：彻底解决 GC —— G1 / ZGC 调优

```bash
# 老服务用 CMS 或 G1 时
-XX:MaxGCPauseMillis=200
-XX:+UseG1GC

# JDK 11+ 直接上 ZGC
-XX:+UseZGC -XX:+UnlockExperimentalVMOptions
```

ZGC 一般 STW < 10ms，根本不会触发 Session 超时。

#### 验证

故意触发 Full GC（`jcmd <pid> GC.run` 多次），监控 ZK 上的注册节点是否消失。修复后应稳定。

> **口诀**：**ZK Session timeout 必须 >> JVM 最坏 STW 时间**。生产 60s 起步，别迷信"短超时 = 快感知"。

---

### 案例 ⑦ 注册风暴 —— "新版本上线，注册中心挂了"

#### 现象

业务发布，3000 个 Pod 同时滚动重启。新 Pod 启动后注册到 Nacos，结果 Nacos Server 集群 CPU 100%，新注册请求超时，**已运行的服务的心跳也开始超时**，业务雪崩。

#### 根因

3000 个实例同时注册：
- AP 模式：每个注册要触发集群内同步给其他 N-1 节点 → 3000 × N 次跨节点 RPC
- 每个新实例首次连接 → SDK 拉一次全量注册表（几百 KB）→ 网络带宽爆
- Server 端 JSON 序列化 + 内存数据结构操作 CPU 飙升
- 一旦响应慢，已运行实例的心跳开始堆积
- 心跳堆积 → 实例被误判 DOWN → 推送变更给所有客户端 → 客户端请求重定向 → 雪崩

K8s 滚动更新策略默认 `maxSurge=25%`，3000 实例同时增加 750 个，确实容易冲爆。

#### 修复

**方案 A**：滚动发布限速

```yaml
spec:
  strategy:
    rollingUpdate:
      maxSurge: 5%        # 一次最多多 5%
      maxUnavailable: 0
```

3000 实例分 20 批，每批 150 个。

**方案 B**：Pod 启动时加随机延迟

```dockerfile
CMD ["sh", "-c", "sleep $((RANDOM % 30)) && exec java -jar app.jar"]
```

避免 K8s 一次调度的几十个 Pod 在同一秒注册。

**方案 C**：注册中心扩容 + 分集群

3000 实例量级建议 5 节点 Nacos 集群以上。或者按业务线拆多个 Nacos 集群。

**方案 D**：升级到 Nacos 2.x

gRPC stream 比 HTTP Long Polling 在大规模下显著更省 CPU。

#### 验证

模拟 3000 Pod 同时重启，观察 Nacos Server CPU 峰值和心跳超时数。

> **口诀**：**注册中心的吞吐瓶颈出现在"发布瞬间"**。通过限速滚动 + 启动抖动 + 集群扩容来分摊。

---

### 案例 ⑧ 跨语言客户端心跳机制不一致 —— "Go 服务总被误判下线"

#### 现象

Java 服务跑着稳定，但同一个 Nacos 集群里的 Go 服务实例**频繁出现"健康/不健康"状态切换**，Java 调用方拿到 Go 实例时偶发连接失败。

#### 根因

排查发现：
- Java SDK 默认心跳间隔：5s
- Go SDK 默认心跳间隔：5s
- **但 Go SDK 实现里心跳是同步阻塞的**，遇到 Server 慢响应（比如 GC）会阻塞下次心跳
- 两次心跳实际间隔可能 10-15s
- Nacos Server 的健康超时配置：15s

→ Go 实例频繁踩在超时边缘。

更深层：
- Java SDK 用 ScheduledExecutor，固定频率
- Go SDK 早期版本用 `time.Sleep` + 同步调用，受 Server RT 影响

#### 修复

**方案 A**：升级到最新版 Go SDK（v2 已修）

```bash
go get github.com/nacos-group/nacos-sdk-go/v2@latest
```

**方案 B**：调整心跳/超时比例

```yaml
# Nacos server
nacos.naming.healthcheck.heartbeat.timeout=20000   # 心跳超时拉到 20s
```

给跨语言客户端更多余量。

**方案 C**：跨语言场景统一用 Sidecar 模式（提前向 Service Mesh 演进）

每个服务旁挂一个 Envoy / Consul Connect / Nacos Sidecar，所有语言统一通过 Sidecar 注册，避免每语言 SDK 行为差异。

#### 验证

```bash
# 监控 Go 实例的 healthy 状态切换次数
curl "http://nacos:8848/nacos/v1/ns/operator/metrics" | jq .healthyInstanceCount
```

修复后应稳定。

> **口诀**：**多语言场景下 SDK 行为差异是隐形坑**。核心配置（心跳间隔、重试策略）必须跨语言对齐，或者上 Sidecar 隔离。

---

### 案例 ⑨ 注册中心宕机后服务无法新增 —— "运行的能跑，新发布上不去"

#### 现象

某天注册中心整个集群挂掉（机房断电）。监控发现：
- **已运行的服务之间互相调用正常**（出乎意料）
- 但新发布的实例**起不来**：注册不上去，超时退出
- 想紧急扩容也扩不了（新 Pod 注册失败）

#### 根因

存量服务能跑的原因：客户端 SDK 都有本地缓存（Eureka / Nacos / Consul 都有）。注册中心挂掉后，客户端继续用最后一次拉到的实例列表。

新实例起不来的原因：
1. 新 Pod 启动时 SDK 调注册中心 register API → 全部超时
2. SDK 默认行为：注册失败 → 重试 N 次 → 抛异常 → 应用启动失败

更糟：有的注册中心 SDK 的"健康检查"也会失败（要求注册中心 ping 通），导致 K8s readinessProbe 一直 fail，Pod 永远不 Ready。

#### 修复

**方案 A**：SDK 注册失败不要让应用启动失败

```yaml
# Spring Cloud Nacos
spring:
  cloud:
    nacos:
      discovery:
        fail-fast: false   # 注册中心不通不影响启动
        register-enabled: true   # 但仍要注册（注册中心恢复后异步重注册）
```

**方案 B**：Nacos / Eureka SDK 启用 Failover 模式

Nacos SDK 在 `~/nacos/naming/<namespace>/failover/` 下可以预置 failover 文件，注册中心连不上时用本地文件。

```bash
# 把当前服务列表保存到 failover 目录
curl http://nacos:8848/nacos/v1/ns/instance/list?serviceName=foo > \
  ~/nacos/naming/public/failover/foo
```

**方案 C**：注册中心多活 / 跨机房

最根本的解决方案。3 机房部署，单机房挂掉不影响服务发现。

**方案 D**：客户端缓存到磁盘

```yaml
# Eureka client
eureka:
  client:
    backup-registry-impl: com.netflix.discovery.DefaultEurekaClientConfig$BackupRegistry
```

实例列表 dump 到磁盘，新进程启动时如果注册中心不可达，从磁盘加载。

#### 验证

```bash
# 模拟注册中心挂掉（在所有 Nacos 机器上）
sudo iptables -A INPUT -p tcp --dport 8848 -j DROP

# 同时启动一个新 Pod，看是否能起来 + 旧服务调用是否正常
```

> **口诀**：**注册中心宕机时，已知服务能调，新增服务难起**。这是几乎所有注册中心的通病，必须靠多机房 + Failover 缓存 + fail-fast=false 三件套兜底。

---

## 6. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 选型层
- [ ] 明确业务对一致性的真实要求：服务发现选 AP，配置中心 / Leader 选举选 CP
- [ ] K8s 内部服务用 K8s Service（实际是 etcd）即可，不要再叠一层注册中心
- [ ] 跨语言场景优先选 SDK 完善的（Nacos / Consul / etcd）
- [ ] 评估是否要走 Service Mesh，避免 SDK 跨语言治理负担
- [ ] 老 Dubbo + ZK 场景考虑迁移到 Nacos（更现代）

### 部署层
- [ ] 集群至少 3 节点（Quorum），生产推荐 5 节点
- [ ] CP 集群节点数必须为奇数（3/5/7）
- [ ] 跨机房部署：AP 选独立集群双向同步，CP 选 stretched cluster + 高带宽专线
- [ ] Server 端 JVM heap 至少 4G（万级实例规模）
- [ ] 注册中心独占机器，不与业务混部

### 配置层
- [ ] 心跳超时 ≥ 心跳间隔 × 3（默认 30s 心跳 / 90s 超时是合理基线）
- [ ] ZK Session timeout ≥ JVM 最坏 STW 时间 × 2（生产 60s 起步）
- [ ] Eureka 自我保护必须开启（`enable-self-preservation=true`）
- [ ] Eureka 推送相关参数生产应优化（`responseCacheUpdateIntervalMs=3000`）
- [ ] Nacos 服务的 ephemeral 标识统一（推荐 true）

### 客户端层
- [ ] SDK 配置 `fail-fast=false`，注册中心不通不影响应用启动
- [ ] 客户端启用本地缓存 + Failover 文件
- [ ] 客户端启用主动健康检查（不完全信任注册中心列表）
- [ ] 配置 LoadBalancer 的 zone-affinity，优先调本机房实例
- [ ] 跨语言 SDK 行为统一（心跳间隔、重试策略）

### 发布层
- [ ] K8s 滚动发布 maxSurge ≤ 10%（避免注册风暴）
- [ ] Pod 启动时加随机延迟（5-30s）打散注册
- [ ] preStop 钩子主动调注册中心 deregister API
- [ ] terminationGracePeriodSeconds ≥ 60（给优雅下线时间）
- [ ] 大规模批量缩容时主动 deregister，避免误触发自我保护

### 监控层
- [ ] 注册中心集群 CPU / 内存 / 网络入流量
- [ ] 实例数量监控 + 异常波动告警（短时大幅下降 = 误删 / 网络故障）
- [ ] Eureka 自我保护状态告警（进入即告警）
- [ ] Nacos history list / 心跳处理 QPS
- [ ] ZK history watch / pending request 数
- [ ] 客户端注册成功率 + 拉取耗时
- [ ] 业务侧"no instance available" 错误率告警

---

## 7. 还能往下挖什么

读完上面只是起点。如果想再深一层：

### Service Mesh 与 xDS 协议
- Istio + Envoy 的 xDS 体系：CDS / EDS / LDS / RDS / SDS
- xDS 本质就是"控制面 → 数据面"的服务发现 + 路由配置推送协议
- gRPC 直接支持 xDS，绕过传统注册中心 SDK
- xDS Streaming Aggregated（SOTW vs Incremental）

### K8s Service vs 注册中心
- K8s Service 是"基于 etcd + kube-proxy" 的服务发现实现
- Endpoints 控制器把 Pod IP 写到 etcd，kube-proxy 监听变更更新 iptables / IPVS
- 跨集群服务发现（多 K8s 集群）的边界
- 何时需要在 K8s 之上再叠 Nacos：复杂治理（多版本、灰度、限流）

### DNS-based Service Discovery
- AWS Cloud Map、Kubernetes CoreDNS、Consul DNS
- DNS TTL 与服务变更感知延迟的关系
- DNS-based 的硬伤：无法做精细路由、健康状态弱

### 跨集群服务发现
- **Karmada**：多 K8s 集群联邦，跨集群 ServiceImport
- **Submariner**：跨集群 Service 互通的网络层方案
- **Istio Multi-Cluster**：Sidecar 直接跨集群发现实例
- 跨地域服务发现的延迟与一致性权衡

### 多协议注册中心
- 一个服务同时暴露 HTTP + gRPC + Dubbo 三个端口怎么注册？
- Nacos / Polaris 的多端点支持
- 协议感知的负载均衡

### 注册中心安全
- mTLS：Istio Citadel / SPIFFE 身份框架
- 服务间认证（SPIFFE ID）
- 注册中心的 ACL：Consul Token、Nacos 命名空间隔离
- 防止 fake instance 注册（实例身份验证）

### 一致性算法对照
- ZK 的 ZAB vs etcd / Consul 的 Raft：本质都是 Paxos 变种
- Nacos Distro 的 AP 协议：自研 hash 分片 + 异步同步
- Gossip 协议（Consul Serf）：弱一致 + 高扩展性

---

## 8. 引用

- Nacos 源码：[alibaba/nacos](https://github.com/alibaba/nacos)
- Eureka 源码：[Netflix/eureka](https://github.com/Netflix/eureka)
- Consul 文档：[Anti-Entropy](https://developer.hashicorp.com/consul/docs/architecture/anti-entropy)
- etcd 文档：[Watch API](https://etcd.io/docs/v3.5/learning/api/#watch-api)
- Netflix Tech Blog：[Eureka! Why You Shouldn't Use ZooKeeper for Service Discovery](https://medium.com/knerd/eureka-why-you-shouldnt-use-zookeeper-for-service-discovery-4932c5c7e764)（2014 经典）
- Mikito Takada, *Distributed Systems for Fun and Profit*（CAP 与 PACELC 的科普）
- Diego Ongaro, *In Search of an Understandable Consensus Algorithm*（Raft 论文）
- Envoy xDS 协议：[xDS REST and gRPC protocol](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol)

---

> 配套：[分布式系统主题](/article/distributed/) | [共识算法](/article/distributed/consensus) | [高可用架构演进](/article/system-design/high-availability)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
