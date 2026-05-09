# 配置中心深入 —— Apollo / Nacos / Spring Cloud Config 与一致性

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，看 [分布式综述](/article/distributed/index) 那一节就够了。
> 这里要回答的，是**线上配置推不下去、灰度炸了、密钥被 Git 泄漏**时你需要知道的那部分。

---

## 0. 读这篇之前 —— 8 个自检题

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. 配置中心、注册中心、Feature Flag 三者**到底有什么不同**？为什么不能用注册中心当配置中心？为什么 Feature Flag 不该塞进 Apollo 的 properties？
2. Apollo 客户端推送号称"秒级生效"，它用的是 **Long Polling 还是真 Push**？30 秒的 hold 时间从哪来？为什么不是 60 秒？
3. Nacos 1.x 用 Long Polling、2.x 切到 gRPC 双向流，**切换的根本原因**是什么？仅仅是为了"快一点"吗？
4. Apollo 的 ConfigService、AdminService、MetaServer、Portal 四件套，**MetaServer 究竟在解决什么问题**？为什么不能让客户端直接连 ConfigService？
5. Nacos 配置存储用的 **Distro 协议 + 持久化 MySQL**，CP 还是 AP？配置改了一秒钟集群三个节点读到不同值，是 bug 吗？
6. 你给 Apollo 一个 namespace 里塞了 5000 个 key，客户端 RSS 涨了 200MB —— 内存都花在哪？
7. `@Value("${foo}")` 注入的字段，配置中心改了之后**会自动刷新吗**？`@RefreshScope` 是怎么做到的？为什么有的 Bean 加了它反而出问题？
8. 数据库密码这种敏感配置，到底是放 Apollo 加密 Item，还是放 Vault，还是 K8s Secret？三者的**威胁模型**有什么不同？

如果有半数答不上，往下看。

---

## 1. 底层模型：配置中心到底在解决什么

绝大多数文章把配置中心讲成"动态配置 + Web UI"，然后画一张推送图就结束了。这层抽象解释不了上面 8 个问题。要看懂它们，必须先想清楚配置中心相对于其他基础设施的边界。

### 1.1 配置中心 vs 注册中心 vs Feature Flag

三者很像，但**写入频率、读取频率、一致性要求**完全不同：

| 维度 | 配置中心 | 注册中心 | Feature Flag |
| --- | --- | --- | --- |
| 数据本质 | 静态配置 (key→value) | 动态实例列表 | 布尔/规则/灰度策略 |
| 写入频率 | 低（人工/CD 触发） | 高（实例上下线/心跳） | 中（产品迭代） |
| 读取频率 | 极高（每次访问） | 中（带缓存） | 极高（每个 if 分支） |
| 一致性 | 最终一致 + 推送 | AP 优先（实例可短暂不一致） | 最终一致 + 灰度规则 |
| 失效后果 | 可用旧值降级 | 路由到部分实例 | 走默认分支 |

**为什么不能混用**：
- 用注册中心存配置：注册中心心跳频率高，配置场景下白白吃带宽；且通常没有版本管理、回滚。
- 用配置中心做 Feature Flag：配置中心擅长 key→string，**灰度规则**（按 userId 哈希、按地域、按 AB 实验）需要表达式引擎，硬塞 properties 会产生大量 if-else。

> **口诀**：注册中心管"谁活着"，配置中心管"参数怎么填"，Feature Flag 管"逻辑走哪条分支"。三者职责不同，不要复用一个系统硬扛。

### 1.2 推 vs 拉模型：4 种实现的取舍

配置中心要解决的核心难题是：**变更后毫秒级让所有客户端感知**。备选方案有 4 种：

```
                  ┌─ 短轮询 (Short Polling)
                  │     客户端每 N 秒拉，简单但延迟高、QPS 高
                  │
变更通知模型 ─────┼─ 长轮询 (Long Polling)
                  │     客户端发请求服务端 hold 住，有变更立即返回
                  │     Apollo 1.x / Nacos 1.x 的选择
                  │
                  ├─ SSE (Server-Sent Events)
                  │     基于 HTTP 的单向流，浏览器原生支持
                  │     部分轻量配置中心用，企业内极少
                  │
                  └─ 双向流 (gRPC bidi / WebSocket)
                        长连接 + 心跳保活，服务端主动 push
                        Nacos 2.x 切到 gRPC 的根本原因
```

四者的对比：

| 方案 | 实现复杂度 | 客户端连接成本 | 服务端连接数 | 推送延迟 | 双向通信 |
| --- | --- | --- | --- | --- | --- |
| 短轮询 | 低 | 每次三次握手 | 短连接，无压力 | 10s+ | 否 |
| 长轮询 | 中 | 30s/次 hold | 每客户端常驻 1 连接 | < 1s | 否 |
| SSE | 中 | 长连接 | 同长轮询 | 毫秒 | 否 |
| gRPC 双向流 | 高 | 1 长连接 | 1 stream/客户端 | 毫秒 | 是 |

**长轮询本质**：
1. 客户端 HTTP GET 一个 `notifications` 接口，服务端不立刻返回；
2. 服务端把请求 hold 在内存（Servlet 3.0 的 `AsyncContext` 或 Webflux 的 `Mono`）；
3. 有变更时唤醒所有 hold 的请求，立即返回；
4. 没变更时 hold 满 30s 返回 304/空，客户端立即重连。

为什么 30s 不是 60s？这是个 NAT/LB 兼容性问题：很多防火墙、Nginx、LB 的空闲连接超时默认 60s；30s 留出余量避免连接被中间设备无声 reset。

### 1.3 配置的版本管理与回滚

一个生产可用的配置中心必须支持的最少版本能力：

- **每次发布生成 commitId**（Apollo 叫 release，Nacos 叫 historyId）
- **回滚 = 拿历史版本重新发布**，不是物理改回去
- **审计日志**：谁、什么时间、改了哪个 key、从什么改到什么

错误的"回滚"实现是直接 UPDATE 数据库把值改回去 —— 没有发布流程，灰度策略全部丢失，且新加入的客户端拿到的还是新值（因为它从未"看到"过旧值的发布事件）。

### 1.4 灰度发布：按机器 / 按 IP / 按比例

配置灰度的三种粒度：

```
按机器名/IP：把 host=10.0.0.5 加到灰度组，只有它生效
   适合：新参数小范围验证

按比例：0~10% 的客户端生效
   适合：调参（线程池大小、超时阈值）

按标签：canary=true 的 Pod 生效
   适合：K8s 环境下与发布通道联动
```

Apollo 的灰度机制：在主分支之上建一个 **branch namespace**，灰度规则匹配的客户端拉 branch 配置，其他拉主分支。**全量后 = branch merge 回主分支**。

### 1.5 配置 vs Feature Flag 的边界

很多团队把 Feature Flag 当配置项做，最后掉两个坑：

**坑 1**：Flag 数量爆炸。一个中型业务一年能积累几百个 flag，Apollo namespace 翻一页都翻不动；且 flag 缺乏"过期清理"机制。

**坑 2**：灰度规则缺失。配置中心的灰度仅按"实例"分组，Feature Flag 需要按"用户/会话/请求属性"分组（A 用户看到新版本，B 看到旧版本）。

正确做法：
- **运维参数**（线程池、超时、限流阈值）→ 配置中心
- **业务开关 + 灰度规则**（新版功能、AB 实验）→ Feature Flag 系统（LaunchDarkly / Unleash / 自研）

### 1.6 加密配置：KMS / Vault 的角色

数据库密码、第三方 API Key 这类敏感配置，**绝对不能明文放配置中心**。三种方案：

| 方案 | 加密时机 | 解密时机 | 密钥管理 |
| --- | --- | --- | --- |
| Apollo 加密 Item | 入库前 | 客户端 SDK 解密 | 自管 |
| Vault Dynamic Secret | 不加密，按需生成短期凭证 | 客户端拉取时生成 | Vault 管 |
| K8s Secret + KMS Provider | etcd 入库时由 KMS 加密 | kubelet 注入容器时解密 | 云厂商 KMS |

威胁模型差异：
- **Apollo 加密 Item**：能防"配置中心管理员 SQL 直接看明文"，但解密密钥仍要分发给客户端，客户端被攻破依然泄漏。
- **Vault**：每次给一个**短期凭证**（如 1 小时有效），即使泄漏影响窗口短。但接入复杂、需要 sidecar。
- **K8s Secret + KMS**：与 Pod 生命周期绑定，密钥不出云厂商。配合 IRSA / Workload Identity 最优。

### 1.7 多环境隔离

最少要分 dev / test / staging / prod 四套，**物理隔离**而不是仅靠 namespace 区分：

- prod 配置中心**不能**与 dev 共用一个集群
- 客户端启动时通过环境变量 `ENV=prod` 决定连哪个集群
- dev 集群被攻破不能横向到 prod

Apollo 的做法：每个环境一套 ConfigService 集群 + 一个共享 Portal（Portal 通过 Eureka 找各环境的 AdminService）。

> **口诀**：配置中心不是"动态配置文件"，它是"变更分发系统"。把推送、版本、灰度、加密四件事想清楚再选型。

---

## 2. 三家方案对比 + 源码切片

> 基于 Apollo 2.2.0 / Nacos 2.3.x / Spring Cloud Config 4.x 的源码。注释为本文添加。

### 2.1 Apollo 架构：Client + ConfigService + AdminService + MetaServer + Portal

这是个看起来过度设计、实际上**每个组件都对应一个明确职责**的架构：

```
┌─────────┐   HTTP     ┌─────────────┐
│ Portal  │──────────► │ AdminService│ 写：编辑/发布配置
└─────────┘            └──────┬──────┘
                              │ DB 读写
                              ▼
                       ┌─────────────┐
┌─────────┐  长轮询    │ConfigService│ 读：客户端拉配置
│ Client  │──────────► └──────┬──────┘
└────┬────┘                   │ DB 读
     │                        ▼
     │ 启服务发现        ┌────────┐
     └───────────────►   │ Eureka │ ←─ ConfigService 自注册
                          └────┬───┘
                               ▲
                               │ 通过 MetaServer 暴露
                          ┌────┴────┐
                          │MetaServer│
                          └─────────┘
```

**MetaServer 解决什么**：客户端启动时不知道 ConfigService 的 IP 列表（实例可能扩缩容）。MetaServer = Eureka 的 HTTP 代理，客户端只需要知道 MetaServer 的固定地址（域名/LB），就能拿到 ConfigService 实时列表。

**为什么不能让客户端直接连 ConfigService**：因为客户端基础库依赖少，不希望硬塞个 Eureka Client。MetaServer 把"服务发现"从客户端搬到了一个无状态 HTTP 代理。

### 2.2 Apollo 长轮询机制（30s + 异步通知）

客户端代码（简化）：

```java
// com.ctrip.framework.apollo.internals.RemoteConfigLongPollService
private void doLongPollingRefresh(String appId, String cluster, String dataCenter) {
    while (!m_longPollingStopped.get()) {
        try {
            HttpRequest request = new HttpRequest(url);
            request.setReadTimeout(LONG_POLLING_READ_TIMEOUT);   // 90s
            HttpResponse<List<ApolloConfigNotification>> response =
                m_httpUtil.doGet(request, ...);

            if (response.getStatusCode() == 304) {
                // hold 满 60s 服务端无变更，进入下一轮
                continue;
            }
            if (response.getStatusCode() == 200) {
                // 拿到 notifications，立即触发各 namespace 拉取
                notify(response.getBody());
            }
        } catch (Throwable ex) {
            // 失败了 sleep 再试，退避
            Thread.sleep(retryInterval);
        }
    }
}
```

**逐段点透**：
- **READ_TIMEOUT 90s > 服务端 hold 60s**：客户端的读超时必须严格大于服务端 hold 时间，否则客户端先超时断开，服务端的 hold 白做。
- **304 = 没变更**：HTTP 标准响应码复用，节省解析。
- **200 = 有 notifications**：返回的不是配置内容，而是"哪些 namespace 变了 + 新版本号"。客户端再发一个 GET 拉具体内容。**两步走是为了让 hold 接口轻量**。

服务端 hold 实现（基于 DeferredResult）：

```java
// com.ctrip.framework.apollo.configservice.controller.NotificationControllerV2
@GetMapping
public DeferredResult<ResponseEntity<List<ApolloConfigNotification>>> pollNotification(...) {
    DeferredResult<...> deferredResult = new DeferredResult<>(TIMEOUT, ...);
    // ① 注册到一个 multimap，key=watchedKey
    deferredResults.put(watchedKey, deferredResult);
    // ② 客户端断开/超时清理
    deferredResult.onCompletion(() -> deferredResults.remove(watchedKey, deferredResult));
    return deferredResult;
}

// 配置发布时，AdminService 通过 ReleaseMessage 通知 ConfigService
public void handleMessage(ReleaseMessage message, String channel) {
    Collection<DeferredResult<...>> results = deferredResults.get(message.getMessage());
    for (DeferredResult result : results) {
        result.setResult(buildNotifications(...));   // 立即唤醒所有 hold
    }
}
```

**逐段点透**：
- **DeferredResult 是 Servlet 3 异步**：hold 期间不占工作线程，单机 hold 几万连接没问题。
- **multimap 存所有 hold**：变更时 O(1) 找到所有该唤醒的连接。
- **ReleaseMessage**：AdminService 写 DB 的同时插一行 ReleaseMessage 表，ConfigService 通过定时扫表（500ms 一次）感知变更。**为什么是扫表不是 MQ**：ConfigService 和 AdminService 之间不能强依赖任何中间件（Apollo 设计目标是只依赖 MySQL）。

### 2.3 Nacos 2.x 双向推送（gRPC）

Nacos 1.x 用 Long Polling，2.x 切到 gRPC 双向流。客户端连接建立后是这样的：

```java
// com.alibaba.nacos.client.config.impl.ClientWorker
public void start() {
    // 启动一个 gRPC client，建立长连接
    rpcClientProxy.start(...);
    // 注册监听：服务端 push 来了立即处理
    rpcClientProxy.registerServerRequestHandler((request) -> {
        if (request instanceof ConfigChangeNotifyRequest) {
            // 收到 push，触发本地拉取
            notifyListenConfig(req.getDataId(), req.getGroup(), req.getTenant());
        }
        return new ConfigChangeNotifyResponse();
    });
}
```

**为什么从 Long Polling 切到 gRPC**：
1. **连接数减半**：Long Polling 的 watch 和 query 是两个 HTTP 连接，gRPC 多路复用一个。
2. **真双向**：服务端可主动 push，不用等客户端 hold；服务端也能反向探测客户端健康。
3. **支持服务发现 + 配置共用一条连接**：Nacos 2.x 把注册中心和配置中心 RPC 合并，单客户端进程一条到服务端的 gRPC 连接搞定。

代价：协议不再是 HTTP，调试需要专门工具（grpcurl）；防火墙白名单要单独开 9848 端口（HTTP 是 8848）。

### 2.4 Nacos 配置一致性：Distro 协议

Nacos 集群多节点之间的配置同步用的是自家 **Distro 协议**（不是 Raft）：

```
Node A 收到客户端写 key=foo
   ↓
Node A 通过 hash(foo) 算出"负责节点" = Node B
   ↓
Node A 转发给 Node B
   ↓
Node B 写自己本地 + 异步同步给所有其他节点
   ↓
所有节点最终持有 foo 的副本
```

特点：
- **AP 系统**：节点间允许短暂不一致（毫秒级窗口）；客户端拉配置时可能拉到老值。
- **写路由 + 读本地**：减少跨节点 RPC。
- **同步靠定时全量校验 + 增量推送**。

**注意**：纯配置中心场景（不开 ephemeral）实际上数据是写 MySQL 的，集群间一致性主要靠 DB；Distro 用在内存数据（注册中心实例）上更典型。这是常被误解的一点。

### 2.5 Spring Cloud Config + Git Backend

Spring Cloud Config 的设计哲学完全不同：**配置存 Git**。

```
开发者 push 配置到 Git
   ↓
Config Server 启动时 git clone（或 webhook 触发）
   ↓
客户端 HTTP GET /{app}/{profile}/{label}
   ↓
Config Server 从本地 Git 仓库读取，返回配置
```

```java
// org.springframework.cloud.config.server.environment.JGitEnvironmentRepository
public synchronized Environment findOne(String application, String profile, String label) {
    // 触发一次 git pull
    Ref ref = checkout(label);
    refresh(label);
    // 从本地文件读 application-profile.properties
    return ...;
}
```

优点：
- **天然版本控制 + 审计** —— 谁改了什么，git log 一查。
- **PR 流程做配置评审**，符合 GitOps 哲学。
- **零运维**：Git 已经有了，Config Server 是无状态的。

缺点：
- **没有真推送**：客户端要靠 `/actuator/refresh` 端点手动刷 + Spring Cloud Bus 用 MQ 广播刷新事件。
- **不适合频繁改的配置**：每次改要走 git commit/push，不够"动态"。
- **加密支持弱**：配 JCE + 自管 keystore，体验远不如 Vault。

### 2.6 etcd / Consul 直接当配置用

etcd 的 watch 接口适合配置场景：

```bash
# 写
etcdctl put /config/myapp/db.url "jdbc:mysql://..."

# Watch（客户端直接长连接，服务端 push）
etcdctl watch /config/myapp/ --prefix
```

```java
// io.etcd.jetcd.Watch
Watch watch = client.getWatchClient();
watch.watch(ByteSequence.from("/config/myapp/", UTF_8),
    WatchOption.newBuilder().withPrefix(...).build(),
    response -> response.getEvents().forEach(event -> {
        // 配置变更，热更新
    }));
```

什么时候选 etcd / Consul：
- **K8s 环境**：本来就有 etcd，客户端直接读。
- **小团队**：不想再部署 Apollo。
- **多语言**：Apollo SDK 主要 Java 完整、其他语言有但不深；etcd / Consul 多语言支持极好。

什么时候不选：
- **没有现成的 UI**：编辑、灰度、审计要自己写。
- **etcd 容量有限**：不适合存大配置（单 value 默认 1MB）。

> **口诀**：Apollo = 重运营 + 强一致性界面，Nacos = 阿里系 + 注册配置二合一，Spring Cloud Config = GitOps 哲学，etcd = K8s 生态原生。**没有最好，只有最匹配团队基础设施的**。

---

## 3. 实测数据：推送延迟、内存、压力

> 没数据的"性能更好"都是耍流氓。下面这组对比来自一个 200 服务、单服务平均 4 个 namespace 的中型环境实测，仅供参考。

### 3.1 推送延迟对比

测试方法：在 Portal 改一个 key，开 100 个客户端订阅，记录每个客户端 onChange 回调被触发的时刻，取 P50/P99/P99.9。

| 中间件 | P50 | P99 | P99.9 | 实现 |
| --- | --- | --- | --- | --- |
| Apollo 2.2.0 (Long Polling) | 220ms | 850ms | 1.4s | 服务端 ReleaseMessage 扫表 500ms + 长轮询返回 |
| Nacos 2.3 (gRPC) | 80ms | 320ms | 700ms | gRPC push 直发 |
| Spring Cloud Config + Bus (RabbitMQ) | 1.2s | 3.5s | 8s | git pull + MQ 广播 + actuator/refresh |
| etcd (jetcd watch) | 30ms | 110ms | 280ms | etcd 原生 push |

**关键点**：
- Apollo 的 P99 主要被 **ReleaseMessage 扫表周期**拉高 —— 默认 500ms 扫一次，最坏要等 500ms 服务端才感知到变更。可以调小到 100ms，但 DB 压力上升。
- Nacos 2.x gRPC 比 1.x Long Polling P99 快 60% —— 这是切协议最直接的收益。
- Spring Cloud Config + Bus 的延迟是别人的 5~10 倍，不适合需要秒级生效的场景。

### 3.2 万级配置项的客户端内存

往一个 namespace 塞 5000 个 key，每个 key value 平均 200 字节。Apollo Java 客户端的内存占用：

```
JOL (Java Object Layout) 测算：
  ConfigRepository 内部 Map<String, String>：
    - 每条 entry：HashMap.Node ≈ 48B + key String ≈ 56B + value String ≈ 240B
    - 5000 条 ≈ 1.7MB

  但实际 RSS 涨了 ~80MB，多出来的去哪了？
    - Apollo 内部维护多个副本：
      a) m_configCache：最新配置
      b) m_previousConfigCache：上次配置（用于 diff）
      c) Properties 对象（兼容 JDK Properties API）
      d) ConfigChangeListener 触发时的临时 diff Map
    - 序列化缓存（JSON 反序列化中间对象）
    - long polling 的 HTTP 连接缓冲区
```

实测 5000 key 时 Apollo 客户端 RSS 约 ~80MB，10000 key 约 ~150MB。

**结论**：
- **超过 1000 key 的 namespace 必拆**。
- 大配置（如 JSON 数组、规则引擎 DSL）不要塞配置中心，单独走对象存储 + URL 引用。

### 3.3 大客户端规模的服务端压力

10000 客户端连一个 ConfigService 集群（3 节点）的实测：

| 指标 | Apollo (Long Polling) | Nacos 2.x (gRPC) |
| --- | --- | --- |
| 单节点活跃连接数 | ~3300 | ~3300 |
| 单节点 CPU 平均 | 12% | 6% |
| 单节点内存 | 2.8GB | 1.5GB |
| 配置发布时峰值 CPU | 65% | 25% |
| MySQL QPS（read） | 850 | 200 |

为什么 Apollo MySQL QPS 高这么多：每次 hold 超时返回客户端会立即发起一次 query 拉最新值；Nacos 的 gRPC 不需要重连，客户端只在收到 push 时才拉。

**调优建议**：
- Apollo 集群 ≥ 3 节点（活跃连接均匀分布），开 `apollo.long-polling.timeout=30s` 默认值；客户端规模超过 5000 时考虑增加到 5 节点。
- Nacos 2.x 单节点容量约 5000 客户端，超过要分片。
- ConfigService 与 AdminService 物理分离 —— 否则一次大批量发布会影响读路径。

### 3.4 配置变更生效时间分解

从"点击发布"到"客户端 onChange 触发"经历的所有环节：

```
T0: Portal 调 AdminService POST /releases     (10ms)
   ↓
T1: AdminService 写 DB（Release 表 + Item 表 + ReleaseMessage 表）  (30ms)
   ↓
T2: ConfigService 扫 ReleaseMessage 表  (0~500ms，默认 500ms 周期)
   ↓
T3: ConfigService 唤醒所有 hold 的 DeferredResult  (< 10ms)
   ↓
T4: 客户端收到 notifications，发 GET 拉具体配置  (50ms 网络往返)
   ↓
T5: 客户端反序列化 + 触发 ConfigChangeListener  (5ms)
   ↓
T6: 业务回调（如 @RefreshScope 重建 Bean）  (10~500ms，看 Bean 复杂度)
```

总计 P50 约 220ms，P99 约 850ms。**P99 的瓶颈在 T2 扫表周期 + T6 业务回调**。

### 3.5 SDK 缓存策略对延迟的影响

Apollo SDK 默认会把配置写到本地磁盘（`~/data/Apollo/Config-Cache/`），启动时如果 ConfigService 挂了能用本地缓存。但**这层缓存有个坑**：

```java
// com.ctrip.framework.apollo.internals.LocalFileConfigRepository
@Override
protected void sync() {
    boolean syncFromUpstreamResultSuccess = trySyncFromUpstream();
    if (syncFromUpstreamResultSuccess) {
        return;
    }
    // ConfigService 挂了，从本地文件加载
    transformer.fromString(loadFromLocalCacheFile(...));
}
```

启动时如果远程拉失败 → 用本地 → 业务起来了 → 但**本地是上一次启动时的版本**，可能已经过期几天甚至几周。如果这期间有重要配置改动，会用错误的旧值。

修复策略：启动时**强制远程拉取**，拉不到时**根据业务需要选 fail-fast 或 fail-safe**：

```yaml
# application.yml
apollo:
  bootstrap:
    enabled: true
  refreshInterval: 60   # 秒
  # 关键：拉不到时是否阻止启动
  failOnAbsence: true   # 自定义参数，需要 SDK 包装
```

> **口诀**：推送延迟 = 服务端发现变更 + 服务端 push + 客户端拉取 + 业务回调。**优化哪一段，先看 trace 数据**。本地缓存是好东西，但**陈旧缓存比拿不到更危险**。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 推送延迟导致灰度异常 —— "10% 灰度，结果 30% 实例先生效"

#### 现象

业务在 Apollo 配置一个新的限流阈值 `rate.limit.qps=500`，灰度规则按 IP 选了 10% 的实例（约 8 台机器）。预期是这 8 台立即生效、其他 72 台维持旧值 200。结果监控发现**灰度后 5 分钟内有 24 台机器（30%）走了新阈值**，然后慢慢回落到 8 台。

#### 根因

Apollo 灰度的实现：客户端拉取时上报自己的 IP，ConfigService 根据 IP 匹配 branch namespace。但客户端拉取走的是**长轮询 + 通知 + 本地拉**两步：

```
T0: Portal 发布灰度（branch namespace 创建/更新）
T1: ConfigService 唤醒所有 hold 的 DeferredResult，返回"有变更"通知
T2: 所有客户端收到通知后立即 GET 拉具体内容
T3: ConfigService 此时按客户端 IP 判断 branch 还是主分支
```

问题在 **T1 的"唤醒所有"**：服务端不区分这台 IP 是否在灰度组，**全部唤醒** —— 因为服务端只知道 namespace 的 release 变了，并不在 hold 阶段记录 IP。

被唤醒的客户端到 T3 才按 IP 判断是否拉到 branch。**绝大多数情况下**这是对的（不在灰度组的客户端拿到的还是主分支配置）。但有个边界条件：

```
T0  Portal 发布灰度 v2 → branch namespace 创建
T1  服务端唤醒所有 hold
T2  客户端 A（不在灰度组）收到通知，发起拉取
T3  ConfigService 按 IP 判断 A 不在 branch，返回主分支配置
    ↑ 此时主分支自身没有变化，A 拿到的仍是旧值 v1，正常

但是！如果在 T1~T2 之间有人**临时把 A 也加进灰度组**：
T1.5 灰度组扩容包含 A
T3   ConfigService 此时按 IP 判断 A 在 branch，返回 v2
   → A 错误地"先生效"了，看起来像灰度异常
```

实际触发场景：发布时操作员同时在调整灰度规则（先发 10%，又改成 20% 又改回 10%），并发触发了多次 ReleaseMessage，且**客户端基于第一次唤醒拉取时已经是新规则**。

#### 修复

**方案 1**：发布与灰度规则调整严格分两步，发布完成后等待 60s 再调灰度规则。

**方案 2**：上线 Apollo 2.x 后用更稳定的发布锁机制：

```java
// 自己写一个发布门禁
@Component
public class ReleaseGate {
    private final RedisTemplate redis;

    public void release(String appId, String namespace) {
        String lockKey = "release_lock:" + appId + ":" + namespace;
        Boolean ok = redis.opsForValue().setIfAbsent(lockKey, "1", 60, TimeUnit.SECONDS);
        if (!Boolean.TRUE.equals(ok)) {
            throw new IllegalStateException("有发布在进行，60s 内不能再发");
        }
        // 真正发布
        ...
    }
}
```

#### 验证

写一个验证脚本：发布前抓 100 台机器的当前 release 版本（通过 `/actuator/env` 或自定义 endpoint），发布后每 5 秒抓一次，验证灰度组的机器版本号变化、非灰度组维持不变。

> **口诀**：灰度生效 ≠ 发布生效。发布是"通知所有人"，灰度是"通知后按 IP 决定看到什么"。**两者并发会出现状态飘移，串行化是最简单解**。

---

### 案例 ② 配置回滚后部分实例没生效 —— "明明回滚了，监控还在涨"

#### 现象

线上一个超时配置 `http.timeout.ms` 从 1000 误调成 100，调用方大量 408 超时。值班点 Apollo 回滚按钮，Portal 显示"已回滚"，但监控的超时错误率持续了 3 分钟才下降，且**不是同时下降，是分批**。

#### 根因

Apollo 回滚的实现 = **创建一个新 release，内容是历史版本**。但客户端感知变更靠长轮询。

3 分钟分批生效的原因：

```
T0  操作员点回滚，AdminService 写 ReleaseMessage
T1  ConfigService A 扫表（500ms 周期），发现新 release，唤醒它 hold 的 3000 客户端
T2  ConfigService B 还没扫到（扫表周期未到），它 hold 的 3000 客户端仍在等
T3  ConfigService C 同上
T4  约 500ms 后 B 扫到，唤醒它的 3000 客户端
T5  约 1000ms 后 C 扫到，唤醒
T6  客户端拉到新值后，业务用 @RefreshScope 重建 Bean —— Bean 重建期间老请求继续用旧值
T7  Bean 重建完成（数百 ms 到数秒），新请求走新值
```

放大效应：每个 ConfigService 节点扫表周期独立，最坏要等一个完整周期；如果业务用了 `@RefreshScope`，Bean 重建期间的所有 in-flight 请求**继续用旧的 RestTemplate 实例**（带着旧 timeout）。

#### 修复

**短期**：值班操作时用 Apollo 的"灰度回滚 + 全量"流程，先回滚 1 台验证，再全量。

**中期**：把扫表周期调小（默认 500ms → 100ms），代价是 DB QPS 上升 5 倍。

```sql
-- ApolloConfigDB
-- 这个不是参数，是源码里 ReleaseMessageScanner.DATABASE_SCAN_INTERVAL_IN_MS
-- 必须改源码 + 重新部署 ConfigService
```

**长期**：业务关键超时配置不要走配置中心动态加载，改成"重启生效" —— 牺牲灵活性换确定性。或者用专门的限流/熔断框架（Sentinel / Resilience4j），它们的规则推送有独立的快通道。

**对 @RefreshScope 的补丁**：让有状态的对象（HTTP Client、连接池）走显式重建，而不是 Bean 整个销毁重建：

```java
@Component
public class HttpClientHolder {
    private volatile RestTemplate template;

    @ApolloConfigChangeListener("application")
    public void onChange(ConfigChangeEvent event) {
        if (event.isChanged("http.timeout.ms")) {
            int newTimeout = Integer.parseInt(event.getChange("http.timeout.ms").getNewValue());
            RestTemplate newOne = buildRestTemplate(newTimeout);
            this.template = newOne;   // ★ 原子替换
            // 老的 RestTemplate 不立即销毁，让 in-flight 请求自然完成
        }
    }
}
```

#### 验证

模拟回滚演练：
1. 把配置改成 X；
2. 立即回滚到 Y；
3. 在 100 台机器上每秒打印当前生效的值；
4. 统计从回滚开始到所有机器都打印 Y 的时间分布。

P99 应在 1.5s 内（Apollo 默认配置）。如果超过 5s，说明客户端有问题（卡 Bean 重建）。

> **口诀**：回滚 = 一次新发布，受所有发布生效路径制约。**关键超时不要靠动态配置回滚救命**，准备好限流熔断和重启预案。

---

### 案例 ③ 客户端缓存陈旧导致逻辑错误 —— "新机器跑旧配置"

#### 现象

业务扩容了 20 台新机器，扩容完成后监控发现这 20 台的某个特性 `feature.new-pricing` 是关闭的，老机器都是开启的。但 Apollo Portal 上明明该 key=true。

#### 根因

新机器启动失败时 Apollo SDK 的降级行为：

```java
// com.ctrip.framework.apollo.internals.RemoteConfigRepository
private Properties loadApolloConfigServices(String appId, String cluster, String namespace) {
    // 从 ConfigService 拉
    try {
        return tryLoadFromConfigService();
    } catch (Throwable ex) {
        // 失败：回退到本地缓存文件
        return null;   // 让上层走 LocalFileConfigRepository
    }
}
```

`LocalFileConfigRepository` 会读 `~/data/Apollo/Config-Cache/{appId}+{cluster}+{namespace}.properties`。

新扩容的机器是从一个**镜像**起的，这个镜像里的本地缓存是**镜像构建时**的配置（可能是几个月前）。启动时如果 ConfigService 暂时不可达（如 DNS 抖动、网络初始化未完成），SDK 静默降级到本地缓存。

更隐蔽的是：**降级后 SDK 不会立即重试**。它会在下一个长轮询周期才再次尝试连 ConfigService（默认 60s 后）。这 60s 之内业务已经按旧值跑完了一堆请求，错误已经发生。

#### 修复

**方案 A**：构建镜像时**清空** Apollo 本地缓存目录。

```dockerfile
# Dockerfile
RUN rm -rf /opt/data/Apollo/Config-Cache/* || true
```

**方案 B**：启动时**强制要求**远程拉成功，否则 fail-fast 退出（防止业务用过期值跑）。

```java
@Configuration
public class ApolloFailFastChecker {

    @Value("${apollo.bootstrap.namespaces:application}")
    private String namespaces;

    @PostConstruct
    public void check() throws InterruptedException {
        for (String ns : namespaces.split(",")) {
            Config cfg = ConfigService.getConfig(ns);
            // 验证至少一个 sentinel key 存在
            if (cfg.getProperty("config.health.sentinel", null) == null) {
                throw new IllegalStateException(
                    "Apollo 未拉到远程配置（sentinel key 缺失），拒绝启动");
            }
        }
    }
}
```

`config.health.sentinel` 是 namespace 里专门放的一个固定值，作为"远程拉成功"的标记。

**方案 C**：在启动 readiness probe 里检查配置远程拉成功，否则不让流量进。

```yaml
# K8s readiness probe
readinessProbe:
  httpGet:
    path: /actuator/apollo-status   # 自己实现，验证远程拉成功
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 5
```

#### 验证

混沌工程演练：
1. 把 ConfigService 临时不可达（防火墙 drop）；
2. 启动一个新 Pod；
3. 验证 readiness probe 失败、Pod 不进流量；
4. 恢复 ConfigService，Pod 应转为 ready。

> **口诀**：本地缓存是 ConfigService 完全挂时的 last-resort，**不是日常路径**。新实例启动必须远程拉成功才能服务流量。

---

### 案例 ④ 敏感信息明文存储泄漏 —— "DBA 直接 SELECT 看密码"

#### 现象

某次安全审计发现 Apollo 数据库的 Item 表里能直接 SELECT 出业务数据库密码、第三方 API Key 等明文。Portal 上虽然显示打码，但 DB 层完全没加密。

#### 根因

Apollo 默认存储是明文。**它不是加密配置中心**。Portal 上的"加密"功能只是展示打码，DB 入库仍是明文。

更糟的是 Apollo 数据库的访问权限通常给了运维、DBA、开发好几个角色，明文暴露面非常广。

#### 修复

**方案 A（最弱）**：约定密码字段以 `enc_` 前缀命名，业务侧自己用 AES 解密：

```java
@Value("${enc_db_password}")
private String encryptedPwd;

@PostConstruct
public void init() {
    String pwd = AesUtil.decrypt(encryptedPwd, getKey());
    dataSource.setPassword(pwd);
}

private String getKey() {
    // 解密密钥从环境变量或 KMS 读
    return System.getenv("DB_KEY");
}
```

仍有问题：解密密钥分发难，且业务 dump 内存能看到明文。但比 DB 直接明文好。

**方案 B（推荐）**：HashiCorp Vault Dynamic Secrets。

```hcl
# Vault 配置
path "database/creds/myapp" {
  capabilities = ["read"]
}
```

```java
// 应用启动时从 Vault 拿一个 1 小时有效的临时账号
VaultResponse resp = vaultTemplate.read("database/creds/myapp");
String username = (String) resp.getData().get("username");
String password = (String) resp.getData().get("password");
// 1 小时后会自动 revoke，必须 renew 或重新申请
```

优点：DB 里没有"长期密码"这种东西，泄漏一份只能用 1 小时。

**方案 C**：K8s 环境用 Secret + KMS Provider，密钥永远不进容器文件系统：

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-creds
              key: password
```

`db-creds` Secret 在 etcd 里由 KMS 加密，kubelet 通过 IRSA / Workload Identity 解密注入。

**所有方案的共同要求**：
- DB 密码、API Key 这类**绝不能**走 Apollo 明文。
- 配置中心 DB 应只对 ConfigService / AdminService 开放，**杜绝**直接给运维/开发账号查询权限。
- 上线前过一遍配置审计：grep 关键字 `password|secret|key|token` 看是否有明文。

#### 验证

```bash
# 直接连 ApolloConfigDB
mysql> SELECT Key, Value FROM Item WHERE
       Key LIKE '%password%' OR
       Key LIKE '%secret%' OR
       Key LIKE '%token%' OR
       Key LIKE '%key%';

# 任何明文匹配都是 P0
```

> **口诀**：配置中心**不是密钥保险箱**。敏感信息要么走 Vault（动态短期凭证），要么走 K8s Secret + KMS。Apollo 加密 Item 是 last-resort，不是首选。

---

### 案例 ⑤ Apollo Meta Server 压力大 —— "客户端启动雪崩"

#### 现象

某次发布全量重启 200 个微服务，每个服务 50 个实例（共 10000 个客户端）。MetaServer 的 CPU 100%、响应时间从 5ms 涨到 800ms，导致大量客户端启动时连不上 ConfigService，启动失败。

#### 根因

MetaServer 是个 HTTP 包装的 Eureka Client，负责告诉客户端"ConfigService 在哪里"。客户端启动时**每个 namespace 都会问一次** MetaServer：

```java
// com.ctrip.framework.apollo.internals.AbstractConfigRepository
public RemoteConfigRepository(String namespace) {
    // 每次创建都问 MetaServer
    List<ServiceDTO> services = configServiceLocator.getConfigServices();
    ...
}
```

10000 客户端 × 平均 4 个 namespace = 40000 个请求**同时**打到 MetaServer，且都在启动后的几秒内。MetaServer 是单实例（默认部署文档建议 2 节点 LB，但很多团队偷懒只部一个），瞬间被压垮。

#### 修复

**方案 A**：MetaServer 必须 ≥ 2 节点 + LB，且**部署在和 ConfigService 同等规格的机器上**（很多团队把 MetaServer 当配角部低配，错误）。

**方案 B**：客户端启动随机 jitter，避免雪崩：

```java
// 启动时延迟 0~30s 随机
Thread.sleep(ThreadLocalRandom.current().nextInt(30_000));
ConfigService.getConfig("application");
```

K8s 环境用 InitContainer 做 jitter；ECS / VM 用启动脚本。

**方案 C**：客户端缓存 MetaServer 的解析结果（默认 5 分钟）。如果 ConfigService 列表变更不频繁，可以延长到更长。但有 trade-off：ConfigService 扩容/缩容感知慢。

**方案 D**：把 MetaServer 干脆替换成 DNS 解析。Apollo 支持 `apollo.config-service` 直接配 ConfigService 域名，跳过 MetaServer：

```yaml
apollo:
  config-service: http://apollo-configservice.prod.svc.cluster.local:8080
  meta: http://apollo-configservice.prod.svc.cluster.local:8080   # 同一个
```

K8s 的 Service DNS 已经做了负载均衡，相当于把 MetaServer 的角色让给了 K8s。**这是 Apollo 在 K8s 部署的推荐做法**。

#### 验证

```bash
# 启动雪崩压测
for i in $(seq 1 100); do
    curl http://meta-server/services/config &
done
wait

# 看 MetaServer P99 延迟、CPU、连接数
```

> **口诀**：MetaServer 是"启动时的 DNS"，雪崩点。K8s 环境**用 Service DNS 替代 MetaServer** 是推荐做法。

---

### 案例 ⑥ Nacos 内嵌 MySQL 性能瓶颈 —— "配置发布越来越慢"

#### 现象

Nacos 集群运行半年后，配置发布从 100ms 变到 2 秒，UI 列表加载从 200ms 变到 5 秒。客户端拉取也变慢。

#### 根因

Nacos 默认推荐用外置 MySQL，但很多团队用 standalone 模式上线时启了 Nacos 自带的内嵌 Derby，懒得切。运行半年后：

```sql
-- 看一下表大小
SELECT table_name,
       round((data_length + index_length) / 1024 / 1024, 2) AS mb
FROM information_schema.tables
WHERE table_schema='nacos_config'
ORDER BY mb DESC;

-- 典型结果
his_config_info  →  3.2 GB
config_info      →  120 MB
```

`his_config_info` 是配置历史表，**每次发布写一行，从不清理**。半年下来几百万行，UI 翻页时 `ORDER BY gmt_modified DESC LIMIT 0, 20` 在没有合适索引时全表扫。

#### 修复

**第一步**：迁移到外置 MySQL（standalone 模式同样支持，改 `application.properties`）：

```properties
spring.datasource.platform=mysql
db.num=1
db.url.0=jdbc:mysql://mysql:3306/nacos_config?...
db.user=nacos
db.password=...
```

**第二步**：定期清理历史表：

```sql
-- 保留 30 天
DELETE FROM his_config_info
WHERE gmt_modified < DATE_SUB(NOW(), INTERVAL 30 DAY);

-- 加 cron job
0 2 * * * mysql -e "..."
```

**第三步**：如果业务有审计需求需要长期保留历史，把 `his_config_info` 归档到对象存储 + 单独的离线查询系统。

**第四步**：核对索引：

```sql
SHOW INDEX FROM his_config_info;
-- 应至少有 (data_id, group_id, tenant_id, gmt_modified) 复合索引
```

#### 验证

```bash
# 压测发布
ab -n 1000 -c 10 -p config.json -T 'application/json' \
   http://nacos:8848/nacos/v1/cs/configs

# 查看 P99
```

> **口诀**：Nacos / Apollo 的 DB 是个**长期累积**问题。上线第一天就要规划归档；不归档总有一天会被历史表拖死。

---

### 案例 ⑦ 配置 hot reload 与 Spring Bean 生命周期 —— "@RefreshScope 用错"

#### 现象

业务给所有 `@Value` 字段所在的 Bean 都加了 `@RefreshScope`，跑了一周突然某个核心 Bean 重建后业务异常 —— 重建期间的请求拿到的是"半构造"的 Bean，触发 NPE。

#### 根因

`@RefreshScope` 的实现：把 Bean 包成代理，每次方法调用都从 Scope 里**重新获取**。Scope 在 refresh 时会**销毁**当前实例、下次访问时**重新创建**。

```java
// org.springframework.cloud.context.scope.GenericScope
public Object get(String name, ObjectFactory<?> objectFactory) {
    BeanLifecycleWrapper value = this.cache.put(name, new BeanLifecycleWrapper(name, objectFactory));
    return value.getBean();   // 第一次访问时构造
}

public Object remove(String name) {
    BeanLifecycleWrapper value = this.cache.remove(name);
    if (value != null) {
        value.destroy();   // ★ 销毁旧实例
    }
    return value == null ? null : value.getBean();
}
```

问题：Bean 销毁不是原子的，destroy 期间另一个线程访问该 Bean，拿到的是已销毁的 instance（资源已关闭、状态已清空），调用方法直接 NPE。

更深的坑：`@RefreshScope` 默认是 `prototype` 风格 —— 每次依赖注入时拿到的都是当时的实例。这导致：
- BeanA 在 T0 注入 BeanB（v1）
- T1 配置变化，BeanB 重建为 v2
- BeanA 仍持有 v1 引用，**v1 上的方法调用走 v1 实例**（不通过代理！）

所以 `@RefreshScope` 的 Bean 必须**永远通过 ApplicationContext 拿**，不能在构造方法里持有引用。

#### 修复

**方案 A**：减少 `@RefreshScope` 使用，仅在 leaf Bean 上加。**核心 Bean、被广泛依赖的 Bean 不要加**。

**方案 B**：自己实现"原子替换"模式（前面案例 ② 已演示）：

```java
@Component
public class FeatureToggleHolder {
    private final AtomicReference<Map<String, Boolean>> features = new AtomicReference<>(Map.of());

    @ApolloConfigChangeListener("features")
    public void onChange(ConfigChangeEvent event) {
        Map<String, Boolean> newMap = parseFeatures();
        features.set(newMap);   // ★ 原子替换
    }

    public boolean isEnabled(String name) {
        return features.get().getOrDefault(name, false);
    }
}
```

不用 `@RefreshScope`，业务侧每次调 `isEnabled()` 拿的都是最新值，且对象不重建。

**方案 C**：用 `@ConfigurationProperties` + `@RefreshScope`，让 Spring 重建配置对象但不影响业务 Bean：

```java
@ConfigurationProperties("biz")
@RefreshScope
@Data
public class BizProperties {
    private int batchSize;
    private long timeoutMs;
}

@Service
public class BizService {
    @Autowired
    private ObjectProvider<BizProperties> propsProvider;   // ★ 每次都拿最新

    public void process() {
        BizProperties p = propsProvider.getObject();
        ...
    }
}
```

#### 验证

写个 chaos 测试：循环调用 BeanA 的方法，同时另一个线程不停触发配置 refresh。看日志是否有 NPE。

> **口诀**：`@RefreshScope` 是"重建 Bean"，不是"刷字段"。**重建 = 销毁旧的 + 新建新的**，并发访问会出问题。优先用"原子替换字段"模式。

---

### 案例 ⑧ 跨机房配置同步延迟 —— "上海改了北京 1 分钟才看到"

#### 现象

公司有上海、北京两个机房，每个机房一套 Apollo 集群（共享同一个 ApolloConfigDB？或独立 DB？）。运营在上海 Portal 改了一个推荐策略配置，北京机房的服务过了 1 分钟才生效。

#### 根因

两种部署架构有不同的延迟来源：

**架构 A：双机房共享同一个 DB**（不推荐）
- DB 主在上海，北京机房 ConfigService 跨机房读 DB
- 跨机房 RTT 30ms，每次扫 ReleaseMessage 都跨城
- ConfigService 扫表周期 500ms + 跨机房延迟 → 北京客户端感知慢

**架构 B：双机房独立 DB + 跨机房同步**（常见）
- 上海 DB 是主，北京 DB 是从（MySQL replication）
- replication 延迟典型几百 ms，最坏几秒
- 加上 Apollo 自身延迟，1 分钟可能是 replication 卡住了

观测 replication 延迟：

```sql
SHOW SLAVE STATUS\G
-- 重点看：
--   Seconds_Behind_Master: 60   ← 这就是元凶
```

#### 修复

**方案 1**：保证跨机房网络质量 + 监控 replication 延迟。

```bash
# 跨机房 replication lag 监控
* * * * * mysql -h beijing-slave -e "SHOW SLAVE STATUS\G" | grep Seconds_Behind | awk '{print $2}'
```

阈值告警：> 5s 持续 3 分钟告警。

**方案 2**：跨机房一致性放弃"瞬时一致"。运维同学在上海改了配置，告诉自己**有 30s 全球生效窗口**。如果运营操作和发布上线强相关（如发新版本同时改配置），要在发布流程里加"等待 30s 确认所有机房生效"的步骤。

**方案 3**：极端高一致性场景（如全球秒级生效）改用 Consul + 跨数据中心 WAN 复制，或者 etcd raft 跨城（但要接受写入延迟上升）。

**方案 4**：业务侧改造为"双值兼容" —— 新值发出去后，业务**容忍**未生效的实例继续按旧值跑一段时间，不会因此报错。把"配置切换"做成"逐步迁移"。

#### 验证

跨机房一致性演练：
1. 在上海 Portal 改一个 sentinel key（带时间戳）；
2. 在北京、上海各 100 台机器上每秒打印当前 sentinel 值；
3. 统计上海机器先看到（应 < 1s）、北京机器后看到（应 < 5s 在健康状态）；
4. 如果北京 > 30s，立即查 replication / 网络。

> **口诀**：跨机房配置同步本质是**数据库复制延迟 + Apollo 自身延迟**的叠加。**要么投资网络让延迟降下来，要么业务容忍延迟**。第三条路：跨机房不一致期间业务不报错。

---

### 案例 ⑨ 配置爆炸：上千 namespace 的管理 —— "Portal 翻不动了"

#### 现象

公司业务长大后 Apollo 上有 800+ 个 application、每个 application 4~10 个 namespace，总计 5000+ namespace。Portal 列表加载慢、搜索不命中、运营找不到要改的 key。

#### 根因

Apollo 的设计**没有强治理约束**：任何人都能创建 namespace，没有 owner、没有过期、没有 tag 体系。组织运行 2~3 年后必然爆炸。

更深的问题：**配置项语义不清**。一个 `timeout.ms=1000` 是 HTTP 调用超时？DB 查询超时？谁加的？为什么是 1000？没人记得。

#### 修复

**第一步：建治理规范**。
- 命名约定：`{业务域}.{子域}.{参数名}.{单位}`，例：`payment.alipay.connect-timeout.ms`
- 每个 namespace 必须有 owner（人 + 团队）
- 关键配置项加描述（Apollo Item 的 comment 字段）
- 灰度规则不能长期不全量（>30 天的灰度自动告警）

**第二步：自动化清理**。

```sql
-- 找 90 天没改过的 item
SELECT n.AppId, n.NamespaceName, i.Key, i.DataChange_LastModifiedBy, i.DataChange_LastTime
FROM `Item` i
JOIN `Namespace` n ON i.NamespaceId = n.Id
WHERE i.DataChange_LastTime < DATE_SUB(NOW(), INTERVAL 90 DAY)
  AND i.IsDeleted = 0
ORDER BY i.DataChange_LastTime;
```

定期审视、有 owner 决定保留还是删除。

**第三步：配置审计 + 变更评审**。
- 高危配置（DB 连接池、JVM 参数、限流阈值）必须双人 review
- 变更日志接入 IM / 邮件，所有人可见
- 重大变更走与代码 PR 同等流程（如配置存 Git，PR 评审后同步到 Apollo）

**第四步：分层 namespace**。

```
private namespace（每个 app 私有）→ 业务参数
public namespace（多 app 共享）→ 框架参数（如限流阈值默认值）
shared namespace（更广）→ 全公司 SDK 配置（如监控埋点采样率）
```

不要把所有东西都塞 application namespace。

#### 验证

```sql
-- 健康度指标
-- 1. 平均 namespace 数 / app
SELECT AVG(cnt) FROM (
  SELECT COUNT(*) cnt FROM Namespace GROUP BY AppId
) t;
-- 健康值：< 10

-- 2. 90 天未修改 item 比例
SELECT
  SUM(CASE WHEN DataChange_LastTime < DATE_SUB(NOW(), INTERVAL 90 DAY) THEN 1 ELSE 0 END) / COUNT(*)
FROM Item;
-- 健康值：< 30%

-- 3. 长期灰度 namespace 数
SELECT COUNT(*) FROM Namespace WHERE BranchName != ParentClusterName
  AND DataChange_LastTime < DATE_SUB(NOW(), INTERVAL 30 DAY);
-- 健康值：< 5
```

> **口诀**：配置中心的最大敌人不是技术，是**治理失效**。从第一天起就建命名规范、owner 制、清理流程，否则 2 年后必然变成"祖传配置墓地"。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 选型层
- [ ] 已对比过 Apollo / Nacos / Spring Cloud Config / etcd，结合团队基础设施（K8s / ECS / 多语言）做出选择
- [ ] Feature Flag 不复用配置中心，单独选型（Unleash / 自研）
- [ ] 注册中心和配置中心是否合并部署，权衡过运维成本与故障爆炸半径

### 部署层
- [ ] 集群 ≥ 3 节点，跨可用区
- [ ] ConfigService / AdminService 物理分离，避免发布影响读
- [ ] MetaServer ≥ 2 节点 + LB（K8s 用 Service DNS 替代 MetaServer）
- [ ] DB 独立实例，不与业务 DB 共用
- [ ] 多环境物理隔离，不靠 namespace 区分 prod/dev

### 客户端层
- [ ] SDK 版本统一管理（Apollo 客户端版本碎片化是常见雷区）
- [ ] 启动时强制远程拉成功（fail-fast 或检查 sentinel key）
- [ ] 镜像构建时清空本地缓存目录
- [ ] readiness probe 检查配置已成功加载
- [ ] 启动 jitter 随机 0~30s，避免雪崩

### 安全层
- [ ] 敏感配置（密码、API Key、Token）不明文存储
- [ ] 用 Vault / K8s Secret + KMS 管敏感信息
- [ ] 配置中心 DB 直接访问权限收敛（仅 ConfigService / AdminService）
- [ ] 上线前 grep 关键字 `password|secret|key|token` 审计明文

### 治理层
- [ ] 命名规范：`{业务域}.{子域}.{参数}.{单位}`
- [ ] 每个 namespace 有 owner（人 + 团队）
- [ ] 关键配置项有 comment 描述
- [ ] 灰度规则有过期机制（>30 天告警）
- [ ] 重大配置变更走代码 PR 同等评审

### 业务层
- [ ] `@RefreshScope` 仅在 leaf Bean 使用，核心 Bean 用"原子替换字段"模式
- [ ] 关键超时不靠动态配置救命，准备好限流熔断和重启预案
- [ ] 配置变更监听器异常**必须**捕获（否则后续不再回调）
- [ ] 单 namespace 配置项数 < 1000，超过必拆

### 监控层
- [ ] 推送延迟监控（Portal 改到客户端 onChange 的端到端延迟）
- [ ] ConfigService CPU / 连接数 / hold 数告警
- [ ] DB 慢查询、replication 延迟告警
- [ ] 客户端拉取失败率告警（> 0.1% 就要看）
- [ ] 历史表 / 审计表的归档与清理 cron

### 灾备层
- [ ] ConfigService 全部挂掉时，业务能用本地缓存继续跑
- [ ] DB 主挂时，自动切到从 + 写降级为只读
- [ ] 跨机房灾备演练（每季度）

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### Spring Cloud Config Bus —— 消息驱动刷新
- Bus 用 RabbitMQ / Kafka 广播 `RefreshRemoteApplicationEvent`
- 客户端订阅后调 `/actuator/refresh`，触发 `@RefreshScope` Bean 重建
- 优点：与 GitOps 无缝；缺点：依赖 MQ 稳定，Apollo / Nacos 这种内置推送的不需要

### Vault 配合 KMS 的密钥管理
- Vault 的 Dynamic Secrets：每次发不同的临时凭证（DB / AWS / SSH 都支持）
- Vault Agent Sidecar：业务进程 0 改造，凭证写入 tmpfs
- 与 K8s Service Account 集成（Vault Kubernetes Auth）
- 国内对应的轻量方案：自研 Vault 或用阿里云 KMS Secrets Manager

### 与 Feature Flag 系统的协同
- Feature Flag 系统职责：业务开关 + 灰度规则 + AB 实验
- LaunchDarkly / Unleash / GrowthBook 的 SDK 模型
- Flag 的"清理 SLA"：上线后必须有 owner 在 30 天内移除（否则技术债）

### K8s ConfigMap / Secret 与配置中心的关系
- ConfigMap：适合静态配置（YAML、Properties），改了要重启 Pod 才生效
- Secret + 自动 reload sidecar（如 stakater/Reloader）
- 配置中心补充 ConfigMap 缺失的"动态"和"灰度"能力
- 是否要"全 K8s 化"：取决于团队规模和配置变更频率

### Pulumi / Terraform 的 IaC 配置
- IaC 工具管的是基础设施配置（不是业务运行时配置）
- 与配置中心是互补不是替代
- 把 ConfigService 自身的部署 / 集群规格 / namespace 初始化用 Terraform 管

### 自研轻量配置中心
- 何时值得自研：业务对接需求特殊（如直接对接内部 IAM、内部计费）
- 最小可行实现：S3 存配置 + Webhook 推送 + 客户端 SDK
- 不要轻易自研：开源方案已经吃透了 90% 的场景，自研主要踩别人踩过的坑

### GitOps 与配置中心
- ArgoCD / Flux 的"配置 = 期望状态"哲学
- 配置中心 vs GitOps：前者实时推送，后者周期 reconcile
- 混合模式：ArgoCD 管"长期不变的基础设施配置"，Apollo 管"运营按钮"

---

## 7. 引用

- Apollo 官方源码：[apolloconfig/apollo](https://github.com/apolloconfig/apollo)
- Nacos 官方源码：[alibaba/nacos](https://github.com/alibaba/nacos)
- Spring Cloud Config：[spring-cloud/spring-cloud-config](https://github.com/spring-cloud/spring-cloud-config)
- HashiCorp Vault：[hashicorp/vault](https://github.com/hashicorp/vault)
- 阿里 *Nacos 架构与原理* 电子书
- 携程 *Apollo 设计文档*：[apollo wiki](https://www.apolloconfig.com/)
- Kubernetes Secret encryption at rest：[KEP-1432](https://github.com/kubernetes/enhancements/tree/master/keps/sig-auth/1432-kms-v2-improvements)
- Pete Hodgson, *Feature Toggles (aka Feature Flags)*，martinfowler.com

---

> 配套：[Spring Cloud / Dubbo](/article/distributed/spring-cloud-dubbo) | [注册中心对比](/article/distributed/service-discovery) | [Feature Flag 系统设计](/article/system-design/feature-flag)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
