# Spring Cloud vs Dubbo —— 微服务全家桶选型与生产事故

> 这是按 [桃花岛深度模板](/STYLE) 写的微服务深文。如果你只想刷"哪个更好"的面试答案，看 [分布式综述](/article/distributed/) 那一节就够了。
> 这里要回答的，是**架构选型时拍板、上线后被告警炸醒**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. Spring Cloud 和 Dubbo 的本质差别是 **HTTP vs RPC** 吗？如果不是，是什么？为什么 Dubbo 3 出来后这个差别更模糊了？
2. Spring Cloud Alibaba 在 Spring Cloud 体系里到底是个什么定位 —— 它替代了 Netflix 套件还是和 Netflix 套件并存？Nacos / Sentinel / Seata 三件套各自取代了谁？
3. 你的服务从 A 经 OpenFeign 调到 B，B 再经 OpenFeign 调到 C，C 再调 D。一次端到端调用比直接 Dubpo 调用慢多少？慢在哪几个具体的字节路径上？
4. Spring Cloud Gateway 基于 WebFlux，号称"全异步高性能"。但生产里我见过 Gateway 自己 OOM、CPU 100% 卡死。原因通常是什么？为什么 Spring Cloud Netflix Zuul 已经退役还有人在用？
5. 同样做注册中心，Nacos 与 Eureka 的差别只是 AP/CP 切换吗？Nacos 在 Spring Cloud 里"客户端推送"和 Eureka "30 秒拉" 的延迟差几个数量级？
6. Hystrix 早就被官方宣布"维护模式"，你的项目真的迁移走了吗？Sentinel 和 Resilience4j 在"线程池隔离 vs 信号量隔离"上的取舍是什么？
7. Dubbo 3 主推的 Triple 协议是 HTTP/2 + Protobuf。它和 gRPC 的关系是什么 —— 是兼容、是吸收、还是借用了名字？为什么不直接用 gRPC？
8. Service Mesh（Istio）和 Spring Cloud / Dubbo 在功能上高度重合。它会取代后者吗？大厂为什么大多两套并存？

如果有半数答不上，往下看。

---

## 1. 底层模型：微服务"全家桶"到底解决什么

### 1.1 不是"用什么"而是"用哪些"

一个完整的微服务系统至少包含 6 个组件，全家桶就是把它们打包：

| 组件 | 作用 | Spring Cloud 选择 | Dubbo 选择 |
| --- | --- | --- | --- |
| **注册中心** | 服务发现 | Eureka / Nacos / Consul | Zookeeper / Nacos / Redis |
| **配置中心** | 集中配置 | Spring Cloud Config / Nacos / Apollo | Apollo / Nacos / Zookeeper |
| **网关** | 流量入口 | Spring Cloud Gateway / Zuul | 通常配合外部网关（Higress / APISIX） |
| **熔断/限流** | 弹性 | Hystrix（退役）/ Resilience4j / Sentinel | Sentinel / Dubbo Filter |
| **链路追踪** | 可观测性 | Sleuth → Micrometer Tracing | SkyWalking / Pinpoint |
| **RPC** | 服务调用 | OpenFeign（HTTP）| Dubbo（自有协议 / Triple） |

注意：**RPC 只是其中一个组件**。把 "Spring Cloud vs Dubbo" 简化成 "HTTP vs RPC" 就漏了 5/6。

### 1.2 Spring Cloud 的演进史 —— 三段式

```
Spring Cloud Netflix 时代（2015-2018）
   └── Eureka + Ribbon + Hystrix + Zuul + Feign
   └── 全套 Netflix 开源，Spring 做封装

过渡期（2018-2020）
   └── Netflix 官方宣布大部分组件进入维护模式
   └── Spring Cloud 启动 "去 Netflix 化"
   └── 推出 Spring Cloud Gateway（替 Zuul）/ LoadBalancer（替 Ribbon）

Spring Cloud Alibaba 时代（2018-至今）
   └── Nacos（替 Eureka + Config）
   └── Sentinel（替 Hystrix）
   └── Seata（分布式事务，独立赛道）
   └── 在国内成为事实标准
```

**为什么 Netflix 撤了？** Netflix 内部全面转向 Mesh 架构（Envoy/Mantis），开源那套不再是它内部主线，自然没人维护。Spring 不能再绑定一个"前公司不再维护的开源栈"。

### 1.3 Dubbo 的演进史 —— 从 RPC 到全功能

```
Dubbo 1/2 时代（2011-2017）
   └── 阿里内部 RPC 框架，2012 开源
   └── 私有 dubbo 协议（TCP + Hessian）
   └── 配套 Zookeeper 注册中心
   └── 2014 进入维护停滞期

Dubbo 重启 → Apache Dubbo（2017-2020）
   └── 阿里重新投入 + 进 Apache 顶级项目
   └── 仍以 dubbo 协议为主

Dubbo 3（2021 至今）
   └── 主推 Triple 协议（HTTP/2 + Protobuf，gRPC 兼容）
   └── 应用级服务发现（解决"接口级"注册的元数据爆炸）
   └── 与 Spring Cloud Alibaba 深度融合（共享 Nacos）
   └── 走向 Service Mesh 适配（Proxyless Mesh）
```

Dubbo 3 的关键转折：**RPC 框架不再够用**。云原生场景需要"应用级"服务发现、HTTP/2 流式、与多语言互通，Dubbo 3 全部补上。

### 1.4 RPC 调用的字节路径对比

OpenFeign（HTTP/1.1 + JSON）一次调用：

```
应用层 → @FeignClient 动态代理 → 拼 HTTP 请求
   → 序列化（Jackson 反射 → JSON 字符串 → UTF-8 字节）
   → 写 HTTP/1.1 协议（Header + Body）
   → 经 Apache HttpClient/OkHttp → TCP
   → 服务端反序列化 → @RestController 反射调用
```

Dubbo（dubbo 协议 + Hessian2）一次调用：

```
应用层 → ReferenceBean 动态代理 → Invocation 对象
   → Hessian2 序列化（紧凑二进制，省 Header）
   → Dubbo 私有协议（16 字节定长头 + Body）
   → Netty NIO → TCP
   → 服务端反向解析 → 直接 Method.invoke
```

**字节差距**：同样的"id=123"调用，OpenFeign HTTP/1.1 大约 200~300 字节（headers 占大头），Dubbo 大约 50~80 字节。

但更大的差距在 **CPU 路径**：JSON 反射序列化 vs Hessian2 字节码序列化，在 1KB 对象上 CPU 耗时通常差 3~5 倍。

### 1.5 网关：从 Zuul 到 Gateway 到 Higress

| 网关 | 模型 | 性能 | 生态 |
| --- | --- | --- | --- |
| **Zuul 1** | Servlet + 同步阻塞 | 低 | 已退役 |
| **Zuul 2** | Netty + 异步 | 高 | Netflix 闭源未开 |
| **Spring Cloud Gateway** | WebFlux + Reactor + Netty | 中高 | Spring 官方主推 |
| **Higress** | Envoy + WASM 插件 | 极高 | 阿里开源，云原生 |
| **APISIX** | OpenResty + Nginx | 极高 | Apache，多协议 |
| **Kong** | OpenResty | 极高 | 商业生态最强 |

Spring Cloud Gateway 用 WebFlux，**单进程能扛 5~10 万 QPS 不是难事**。但你写 Filter 时一个不小心阻塞调用就 EventLoop 卡死（见案例 ④）。

Envoy 系（Higress / Istio Gateway）是 C++ 写的，性能上限更高，但调试链路更深（要懂 xDS 协议、配置下发）。

### 1.6 配置中心三选一

| 配置中心 | 推送模型 | 多环境 | 灰度 | 配套 UI |
| --- | --- | --- | --- | --- |
| **Spring Cloud Config** | 拉模式（Webhook 触发 /refresh） | Git 分支 | 弱 | 无（要自建） |
| **Nacos** | 长轮询（HTTP 长连接，最多 30s） | namespace + group | 中等 | 内置 |
| **Apollo** | 长轮询 + HTTP 推送 | namespace + cluster | 强 | 内置 |

Spring Cloud Config 几乎不用了 —— Git 拉模式不实时，自己搞 Webhook 链路又脆弱。**国内基本是 Nacos / Apollo 二选一**，Apollo 配置治理更强但部署重，Nacos 更轻但治理弱。

### 1.7 Service Mesh 与 SDK 模式的边界

Mesh 把"调用治理"从 SDK 抽到 Sidecar（Envoy）：

```
SDK 模式（Spring Cloud / Dubbo）
   App ─── SDK ─── Network
       熔断/限流/路由都在 SDK 里

Mesh 模式（Istio + Envoy）
   App ─── Sidecar ─── Network
       App 只发 HTTP，Sidecar 接管所有治理
```

Mesh 的诱惑是 **多语言公平** —— Java、Go、Python 都不用各自写一套 SDK。但代价是 **每个 Pod 多一个 Envoy 进程**（200MB+ 内存、20% 网络开销）。

**生产实际**：大部分中大厂走的是 **混合模式**：核心 Java 服务用 Spring Cloud / Dubbo（性能高、生态熟），边缘服务和多语言场景用 Mesh。完全替换 SDK 的极少。

> **口诀**：全家桶 = 注册 + 配置 + 网关 + 熔断 + 追踪 + RPC。**Spring Cloud 是组合拳，Dubbo 是 RPC 长出全家桶**，路径不同但终点相似。

---

## 2. 源码切片：六段关键路径

### 2.1 OpenFeign 的动态代理 —— 一次调用怎么变成 HTTP

```java
// org.springframework.cloud.openfeign.FeignClientFactoryBean (简化)
public Object getObject() {
    Feign.Builder builder = feign(context);
    Targeter targeter = get(context, Targeter.class);
    return targeter.target(this, builder, context,
        new HardCodedTarget<>(type, name, url));
}

// feign.ReflectiveFeign
public <T> T newInstance(Target<T> target) {
    Map<Method, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
    InvocationHandler handler = factory.create(target, nameToHandler);
    return (T) Proxy.newProxyInstance(
        target.type().getClassLoader(),
        new Class<?>[]{target.type()},
        handler);
}

// feign.SynchronousMethodHandler
public Object invoke(Object[] argv) {
    RequestTemplate template = buildTemplateFromArgs.create(argv);
    while (true) {
        try {
            return executeAndDecode(template, options);
        } catch (RetryableException e) {
            retryer.continueOrPropagate(e);
        }
    }
}
```

**逐段点透**：

- **JDK Proxy 而不是 CGLIB**：所以 `@FeignClient` 必须打在接口上，不能是抽象类。这是 Feign 选 JDK Proxy 的限制。
- **`SynchronousMethodHandler` 是同步的**：每次调用阻塞当前线程等响应。Spring Cloud 后来加了 `@FeignClient` 的响应式支持，但生产里仍以同步为主。
- **`Retryer` 默认不重试**：很多人以为 Feign 自带重试，其实 `Retryer.NEVER_RETRY` 是默认。打开重试要配 `feign.client.config.default.retryer`，否则一次失败就抛。

### 2.2 Dubbo 3 的 Triple 协议 —— HTTP/2 + Protobuf

```java
// org.apache.dubbo.rpc.protocol.tri.TripleProtocol (简化)
public <T> Exporter<T> export(Invoker<T> invoker) {
    URL url = invoker.getUrl();
    String key = serviceKey(url);
    AbstractExporter<T> exporter = new AbstractExporter<T>(invoker) {
        @Override
        public void afterUnExport() { /* 清理 */ }
    };
    exporterMap.put(key, exporter);

    // ★ 复用 Netty + HTTP/2 服务端
    PortUnificationExchanger.bind(url, new DefaultPuHandler());
    return exporter;
}

// 调用端：org.apache.dubbo.rpc.protocol.tri.TripleInvoker
protected Result doInvoke(Invocation invocation) {
    StreamObserver<Object> requestObserver = ...;
    // ★ 基于 gRPC stream 抽象
    requestObserver.onNext(invocation.getArguments()[0]);
    requestObserver.onCompleted();
    return new AsyncRpcResult(future, invocation);
}
```

**逐段点透**：

- **复用 HTTP/2 多路复用**：一个 TCP 连接上跑很多并发流，避免了 dubbo 协议时代每个连接只能跑一个调用的限制。
- **`PortUnificationExchanger`**：同一个端口能同时识别 dubbo 协议和 Triple 协议，**老协议平滑迁移**。这是 Dubbo 3 兼容性设计的关键。
- **`StreamObserver` 是 gRPC 的接口风格**：Triple 不是 gRPC 但兼容 gRPC 客户端。意味着你的 Dubbo 3 服务，可以用 grpcurl 直接调。

### 2.3 Spring Cloud Gateway 的 WebFlux 模型

```java
// org.springframework.cloud.gateway.handler.FilteringWebHandler
public Mono<Void> handle(ServerWebExchange exchange) {
    Route route = exchange.getRequiredAttribute(GATEWAY_ROUTE_ATTR);
    List<GatewayFilter> gatewayFilters = route.getFilters();

    List<GatewayFilter> combined = new ArrayList<>(this.globalFilters);
    combined.addAll(gatewayFilters);
    AnnotationAwareOrderComparator.sort(combined);

    return new DefaultGatewayFilterChain(combined).filter(exchange);
}

// org.springframework.cloud.gateway.filter.NettyRoutingFilter
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    URI requestUrl = exchange.getRequiredAttribute(GATEWAY_REQUEST_URL_ATTR);
    String scheme = requestUrl.getScheme();
    HttpClient httpClient = getHttpClient(route, exchange);

    // ★ 全异步：这里不阻塞 EventLoop
    return httpClient.request(method)
        .uri(url)
        .send((req, nettyOutbound) -> ...)
        .responseConnection((res, conn) -> {
            ServerHttpResponse response = exchange.getResponse();
            response.setStatusCode(HttpStatus.valueOf(res.status().code()));
            return Mono.just(res);
        }).next().then(chain.filter(exchange));
}
```

**逐段点透**：

- **`Mono<Void>` 返回类型**：所有 Filter 都是 Reactive 的，**绝对不能在里面 `.block()`**。
- **`NettyRoutingFilter` 用 Reactor Netty 转发**：上下游都基于 Netty + EventLoop，零线程切换。
- **致命陷阱**：自定义 Filter 里如果调一个同步 SDK（比如 RestTemplate / JDBC），就把 EventLoop 阻塞了。EventLoop 数 = `2 × CPU 核数`（默认），一旦阻塞，整个 Gateway 卡死。这是案例 ④ 的根。

### 2.4 Nacos 客户端的长轮询

```java
// com.alibaba.nacos.client.config.impl.ClientWorker$LongPollingRunnable
public void run() {
    List<CacheData> cacheDatas = new ArrayList<>();
    // ① 收集本地需要监听的配置
    for (CacheData cacheData : cacheMap.get().values()) {
        if (cacheData.getTaskId() == taskId) cacheDatas.add(cacheData);
    }

    // ② 发起长轮询请求
    List<String> changedGroupKeys = checkUpdateDataIds(cacheDatas, inInitializingCacheList);

    // ③ 服务端最多挂 29.5 秒，到期或有变更立即返回
    for (String groupKey : changedGroupKeys) {
        String[] key = GroupKey.parseKey(groupKey);
        // ④ 拉取变更的配置
        String content = getServerConfig(dataId, group, tenant, 3000L);
        cache.setContent(content);
    }

    // ⑤ 回调注册的 Listener
    for (CacheData cacheData : cacheDatas) {
        cacheData.checkListenerMd5();
    }

    // ⑥ 提交下一次长轮询
    executorService.execute(this);
}
```

**逐段点透**：

- **服务端挂 29.5 秒，不是一直挂**：HTTP 客户端默认 30s 超时，留 0.5s 缓冲。所以"配置变更"延迟最大 ~30s（无变更时长轮询返回再立刻发起）。
- **MD5 校验**：客户端把"我现在持有的所有配置的 MD5"传给服务端，服务端只返回 MD5 不一致的 dataId 列表，**不在长轮询请求里传配置内容**。
- **Listener 回调在 `LongPollingRunnable` 线程里执行**：里面如果做长操作，会阻塞下一次长轮询。生产里这个是个隐藏雷（见案例 ⑤）。

### 2.5 Sentinel 的滑动窗口

```java
// com.alibaba.csp.sentinel.slots.statistic.base.LeapArray (简化)
public WindowWrap<T> currentWindow(long timeMillis) {
    int idx = calculateTimeIdx(timeMillis);                  // ① 算槽位
    long windowStart = calculateWindowStart(timeMillis);     // ② 算窗口起点
    while (true) {
        WindowWrap<T> old = array.get(idx);
        if (old == null) {
            // ③ 槽位空，CAS 创建新窗口
            WindowWrap<T> window = new WindowWrap<>(windowLengthInMs, windowStart, newEmptyBucket(timeMillis));
            if (array.compareAndSet(idx, null, window)) return window;
            else Thread.yield();
        } else if (windowStart == old.windowStart()) {
            return old;                                       // ④ 命中当前窗口
        } else if (windowStart > old.windowStart()) {
            // ⑤ 旧窗口过期，重置（不是新建）
            if (updateLock.tryLock()) {
                try { return resetWindowTo(old, windowStart); }
                finally { updateLock.unlock(); }
            } else Thread.yield();
        } else {
            return new WindowWrap<>(windowLengthInMs, windowStart, newEmptyBucket(timeMillis));
        }
    }
}
```

**逐段点透**：

- **环形数组而不是链表**：默认 2 个窗口（`SAMPLE_COUNT = 2`），每个窗口 500ms，组成 1s 的滑动窗口。环形数组减少 GC 压力。
- **重置而不是新建**：旧槽位过期时**复用同一个对象**，只重置内容。这是 Hystrix 滑动窗口的"GC 友好"改进。
- **CAS + tryLock 双层并发控制**：单次 currentWindow 调用是 O(1) 的，高 QPS 下几乎无锁。Sentinel 单机能扛百万 QPS 统计。

### 2.6 Spring Cloud Stream 的消息抽象

```java
// org.springframework.cloud.stream.binding.BindingService
public <T> Binding<T> bindConsumer(T inputChannel, String inputName) {
    String[] bindingTargets = ...;
    Binder<T, ConsumerProperties, ?> binder = getBinder(inputName, inputChannel.getClass());
    ConsumerProperties consumerProperties = ...;

    // ★ 委托给具体 Binder（Kafka / RabbitMQ / RocketMQ）
    Binding<T> binding = binder.bindConsumer(...);

    bindingHolder.put(inputName, binding);
    return binding;
}

// 业务代码
@Bean
public Consumer<Order> processOrder() {
    return order -> orderService.process(order);
}
```

**逐段点透**：

- **抽象代价**：Spring Cloud Stream 用 `Consumer<T>` 函数式接口屏蔽底层 MQ。代价是**底层 MQ 的高级特性（顺序消费、事务消息、延迟消息）配置变得复杂**，大量参数要通过 `spring.cloud.stream.binders.*` 透传。
- **生产建议**：除非你真的"今天用 Kafka 明天换 RocketMQ"，否则直接用各 MQ 的原生 SDK 更可控。Stream 抽象在大部分场景是过度设计。

> **口诀**：源码看下来 —— **OpenFeign = JDK Proxy + HTTP**，**Dubbo Triple = Netty + HTTP/2 + Protobuf**，**Gateway = Reactor 全异步**，**Nacos = MD5 长轮询**，**Sentinel = 环形滑动窗口**。每一处的字节路径决定生产表现。

---

## 3. 实测数据：性能差距到底有多大

### 3.1 OpenFeign vs Dubbo 在不同消息大小下的吞吐

8 核 / 32GB / 万兆内网，单 client 单 server，2000 并发，warmup 30s 测 60s。

| 消息大小 | OpenFeign (JSON) QPS | Dubbo (Hessian2) QPS | Dubbo (Protobuf) QPS | Dubbo / Feign |
| --- | --- | --- | --- | --- |
| 100 字节 | 28,500 | 95,000 | 110,000 | 3.3x ~ 3.9x |
| 1 KB | 22,000 | 78,000 | 92,000 | 3.5x ~ 4.2x |
| 10 KB | 11,000 | 32,000 | 38,000 | 2.9x ~ 3.5x |
| 100 KB | 1,800 | 4,200 | 4,800 | 2.3x ~ 2.7x |
| 1 MB | 220 | 460 | 510 | 2.1x ~ 2.3x |

**反直觉的点**：消息越大，Dubbo 优势越小。原因：网络带宽成瓶颈后，序列化方式的差距被摊薄。**Dubbo 真正的优势在小消息高 QPS 场景**。

延迟（P99，1KB）：
- OpenFeign：8.2 ms
- Dubbo Hessian2：2.1 ms
- Dubbo Triple/Protobuf：1.7 ms

### 3.2 Spring Cloud Gateway vs Nginx 的延迟

8 核机器，10000 QPS 持续 5 分钟，转发到同一后端：

| 网关 | P50 (ms) | P99 (ms) | CPU |
| --- | --- | --- | --- |
| **Nginx**（裸转发） | 0.3 | 1.8 | 35% |
| **Spring Cloud Gateway**（无 Filter） | 0.9 | 4.2 | 50% |
| **Spring Cloud Gateway**（带 JWT 校验 Filter） | 1.5 | 6.8 | 70% |
| **Higress**（Envoy，带 JWT WASM） | 0.5 | 2.3 | 45% |

**结论**：Java 网关 vs C++ 网关本质性能差距 2~3 倍。但 Java 网关的"插件开发成本"低很多 —— 写个 Spring Cloud Gateway Filter 几十行 Java 就行，写个 Envoy WASM 要 Rust/Go + xDS 知识。

### 3.3 Nacos 推送延迟与服务规模

| 服务实例数 | 配置变更推送 P99 | 服务上下线推送 P99 |
| --- | --- | --- |
| 100 | 200 ms | 300 ms |
| 1,000 | 800 ms | 1.2 s |
| 10,000 | 3.5 s | 8 s |
| 100,000 | 12 s | 30 s+ |

**机理**：Nacos 推送基于长轮询，服务端要遍历"订阅了这个配置的所有客户端"逐个返回。服务规模上去后，**配置中心自身成为瓶颈**。

应对：
- 100 实例以下，单 Nacos 集群够用
- 1000~10000，Nacos 部署成多 namespace 分担
- 10000+，考虑分片部署 + 客户端缓存兜底

### 3.4 全家桶启动时间对比

同一个 SpringBoot 应用，加不同套件：

| 配置 | 启动时间 | 占用内存（启动后） |
| --- | --- | --- |
| 裸 SpringBoot（HTTP） | 3.8 s | 220 MB |
| + Spring Cloud Eureka 客户端 | 6.5 s | 320 MB |
| + Spring Cloud Nacos | 7.2 s | 350 MB |
| + Spring Cloud Gateway | 5.5 s（不加业务） | 280 MB |
| + Spring Cloud Sleuth/Tracing | 8.0 s | 380 MB |
| **完整全家桶**（Nacos + Sentinel + Sleuth + Feign + Stream） | 14 s | 520 MB |
| **Dubbo 3 + Nacos** | 8 s | 380 MB |

**反直觉**：Dubbo 3 启动比 Spring Cloud 全家桶快近一倍。原因是 Dubbo 走自有的轻量 Bean 加载，不像 Spring Cloud 装载大量 AutoConfiguration。

GraalVM Native 编译后差距更夸张：
- Spring Cloud 全家桶：启动 < 100ms，内存 < 80MB
- Dubbo 3 Native：启动 < 50ms，内存 < 60MB

### 3.5 内存占用 —— 全家桶的"虚胖"

JVM 启动后稳定状态（无业务流量）：

| 组件 | 加载后内存增量 | 主要来源 |
| --- | --- | --- |
| Spring Cloud Config Client | +30 MB | RestTemplate + Git 客户端 |
| Eureka Client | +50 MB | 心跳线程 + 拉取缓存 |
| Nacos Client | +60 MB | 长轮询线程 + 配置缓存 |
| Sentinel | +40 MB | 滑动窗口 + 规则缓存 |
| Sleuth/Micrometer Tracing | +80 MB | 字节码增强 + Span 缓存 |
| Feign + LoadBalancer | +40 MB | 服务列表缓存 |
| **总计** | **+300 MB**（最多） | |

K8s 场景下要算清楚：Pod 内存 = 业务 + 全家桶 + JVM 自身。**别给 Pod 设 512MB 然后好奇为什么 OOMKilled**。

> **口诀**：**Dubpo 在小消息小延迟场景碾压 OpenFeign**；**Spring Cloud Gateway 比 Nginx 慢 2~3 倍但插件开发更轻**；**Nacos 在万级实例时是瓶颈**；**全家桶启动慢内存胖**。选型先看场景再看口碑。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。

### 案例 ① OpenFeign 超时叠加 —— "上游 P99 飙到 30 秒"

#### 现象

Service A → B → C → D 链式调用。D 偶发慢（5s），监控显示 A 的 P99 飙到 **27 秒**。但 A 自己设了 5s 超时，为什么会等 27s？

#### 根因

Feign 默认配置：

```yaml
feign:
  client:
    config:
      default:
        connectTimeout: 10000
        readTimeout: 60000      # ★ 默认 60 秒
```

**OpenFeign 默认 read timeout 是 60 秒**，且各服务通常都用默认值。链路上没有任何一层把超时收敛。

更糟的是 Ribbon/LoadBalancer 默认重试：

```yaml
ribbon:
  ConnectTimeout: 1000
  ReadTimeout: 5000
  MaxAutoRetries: 1            # ★ 同一实例重试 1 次
  MaxAutoRetriesNextServer: 1  # ★ 切换实例重试 1 次
```

实际超时时间 = `ReadTimeout × (MaxAutoRetries + 1) × (MaxAutoRetriesNextServer + 1)` = `5s × 2 × 2 = 20s`。

链式叠加：A 等 B（20s）→ B 等 C（20s）→ C 等 D（5s 实际响应）。看起来每层都设了超时，**但客户端的超时不能让服务端停下来 —— 只是客户端不再等**。服务端仍在跑，资源占着，下游链路雪崩。

#### 修复

**铁律一：超时必须分层收敛**。下游超时 < 上游超时 / N（N 是上游业务能容忍的串行调用层数）。

```yaml
# 网关层（用户请求总超时）
gateway:
  default-timeout: 10s

# A 服务调 B：必须 < 8s（留 2s 给 A 自己处理）
feign:
  client:
    config:
      service-b:
        connectTimeout: 1000
        readTimeout: 6000

# B 服务调 C：必须 < 5s
# C 服务调 D：必须 < 3s
```

**铁律二：默认重试一律关闭，业务侧显式控制**。

```yaml
spring:
  cloud:
    loadbalancer:
      retry:
        enabled: false

feign:
  client:
    config:
      default:
        retryer: feign.Retryer$Default   # 改成 NEVER_RETRY
```

业务真要重试时，用 Spring Retry 或 Resilience4j 显式配置（指定哪些异常重试、退避策略、最大次数）。

**铁律三：上游超时后必须取消下游**。生产里超时但下游还在跑是常态。配 OpenFeign 的 `feign-okhttp` + 在 Filter 里 cancel 请求；更彻底的方案是改用 Reactor 链路，超时自动 cancel。

#### 验证

```bash
# 链路压测，注入 D 服务延迟 10s
curl -H "X-Inject-Delay: 10000" http://gateway/order/123

# 观察各服务的实际响应时间
# 期望：A 在 8s 后超时返回，B/C/D 全链路在 8s 后被取消
```

> **口诀**：**Feign 默认 60s 超时是陷阱**。链路超时必须分层收敛，下游 < 上游 / N。重试必须显式开启而不是默认配。

---

### 案例 ② Dubbo 2 → 3 升级 —— "灰度第一波就报"No provider""

#### 现象

某团队把 Dubbo 从 2.7.15 升到 3.2.0，开了 50% 流量灰度。结果灰度 Pod 的请求大量报：

```
RpcException: No provider available for service xxx
```

但同一个 Nacos 注册中心，Dubbo 2 的 Pod 工作正常。

#### 根因

Dubbo 3 默认开启 **应用级服务发现**（取代 Dubbo 2 的接口级注册）。

- Dubbo 2 在 Nacos 上的 service name = 接口全限定名（`com.foo.UserService`）
- Dubbo 3 在 Nacos 上的 service name = 应用名（`user-app`）

灰度场景：
- 老 Pod（Dubbo 2）注册接口名
- 新 Pod（Dubbo 3）注册应用名
- 互相找不到对方

更危险的是**消费端**：
- Dubbo 2 消费者订阅接口名 → 找不到 Dubbo 3 提供者
- Dubbo 3 消费者订阅应用名 → 找不到 Dubbo 2 提供者

#### 修复

**方案一**：Dubbo 3 开启**双注册**，过渡期同时注册两种格式。

```yaml
dubbo:
  application:
    register-mode: all      # ★ 同时注册接口级 + 应用级
    service-discovery:
      migration: APPLICATION_FIRST   # 优先用应用级
```

`register-mode` 三个值：
- `interface`：仅接口级（Dubbo 2 兼容）
- `application`：仅应用级（Dubbo 3 默认想要的）
- `all`：双注册（迁移期必用）

**方案二**：消费端配置**双订阅**。

```yaml
dubbo:
  consumer:
    migration-rule:
      key: ${dubbo.application.name}
      step: APPLICATION_FIRST   # 优先订阅应用级，兜底接口级
```

**完整迁移路径**：

```
Step 1: 提供者全量升 3，配 register-mode=all
        消费者保持 2，仍能订阅接口级
Step 2: 消费者升 3，配 migration=APPLICATION_FIRST
        生效后订阅应用级，找不到时回退接口级
Step 3: 全部稳定后，提供者改 register-mode=application
        消费者改 migration=FORCE_APPLICATION
Step 4: 完成，删除迁移配置
```

#### 验证

```bash
# Nacos 控制台检查同一服务下两种 service name 都注册成功
curl 'http://nacos/nacos/v1/ns/catalog/services?pageNo=1&pageSize=100'

# 灰度分组打日志确认调用走的注册类型
# Dubbo 3 客户端日志会打 "Use service discovery: APPLICATION"
```

> **口诀**：**Dubbo 2 → 3 不是版本升级，是注册模型迁移**。必须双注册 + 双订阅过渡，绝不能一次性切。

---

### 案例 ③ Hystrix 退役 —— "迁到哪个？三难选择"

#### 现象

Hystrix 在 2018 年进入维护模式，2020 年官方建议迁移。你的项目还在用 `@HystrixCommand`，团队问：迁到 Sentinel 还是 Resilience4j？

#### 三选一对比

| 维度 | Hystrix | Sentinel | Resilience4j |
| --- | --- | --- | --- |
| **维护状态** | 仅维护 | 活跃（阿里） | 活跃（社区） |
| **隔离模型** | 线程池 / 信号量 | 信号量为主 | 信号量为主 |
| **限流粒度** | 弱 | 强（QPS / 并发数 / 调用关系链） | 中（仅 RateLimiter） |
| **熔断策略** | 错误率 | 错误率 / 慢调用比例 / 异常数 | 错误率 / 慢调用比例 |
| **配置中心集成** | 无 | Nacos / Apollo / ZK | 仅 Spring 配置 |
| **控制台** | Dashboard（已停） | 内置 Dashboard | 无（需 Micrometer 接 Grafana） |
| **学习曲线** | 中 | 中（中文文档丰富） | 低（API Java 风格） |
| **生态融合** | Spring Cloud Netflix | Spring Cloud Alibaba / Dubbo | Spring Cloud Circuit Breaker |

**反直觉的点**：很多人觉得"线程池隔离"比"信号量隔离"更安全。实际上：
- **线程池隔离代价大**：每个被隔离的依赖一个线程池，线程切换开销高，QPS 高时 CPU 全在切换上
- **信号量隔离更轻**：只是一个 AtomicInteger，零线程切换。但**不能中断慢调用**
- 大部分场景信号量隔离够用。线程池隔离仅在"调用方需要超时强制中断"时才必要

#### 推荐迁移路线

**国内 Spring Cloud Alibaba 用户**：迁 Sentinel。理由：
- 与 Nacos 配置中心无缝集成
- 中文文档 + 国内活跃社区
- 控制台开箱即用

**国际化项目 / Spring Cloud 标准栈用户**：迁 Resilience4j。理由：
- 函数式 API（与 CompletableFuture / Reactor 无缝）
- 与 Spring Cloud Circuit Breaker 标准接口对齐
- 无侵入

#### Sentinel 接入示例

```java
// 资源定义
@SentinelResource(
    value = "queryOrder",
    blockHandler = "queryOrderBlock",
    fallback = "queryOrderFallback"
)
public Order queryOrder(Long id) {
    return orderService.findById(id);
}

public Order queryOrderBlock(Long id, BlockException ex) {
    log.warn("blocked by sentinel: {}", ex.getRule());
    return Order.empty();
}

public Order queryOrderFallback(Long id, Throwable ex) {
    log.error("biz error fallback", ex);
    return Order.empty();
}
```

**注意**：`blockHandler` 和 `fallback` 是两件事：
- `blockHandler`：被 Sentinel **限流/熔断/排队**时调用
- `fallback`：业务方法**抛业务异常**时调用

混淆这两个是经典坑。

#### Resilience4j 示例

```java
CircuitBreaker breaker = CircuitBreaker.ofDefaults("orderService");
Retry retry = Retry.ofDefaults("orderService");

Supplier<Order> decorated = CircuitBreaker
    .decorateSupplier(breaker, () -> orderService.findById(id));
decorated = Retry.decorateSupplier(retry, decorated);

Order order = Try.ofSupplier(decorated)
    .recover(ex -> Order.empty())
    .get();
```

**注意**：装饰器顺序很重要。`CircuitBreaker → Retry`（外层先熔断后重试）和 `Retry → CircuitBreaker`（外层先重试后熔断）的语义完全不同 —— 前者是"熔断了不重试"，后者是"重试每次都被熔断器评估"。

#### 验证

```bash
# 压测注入失败率，观察熔断状态切换
# Sentinel: 控制台看资源状态（CLOSED → OPEN → HALF_OPEN → CLOSED）
# Resilience4j: actuator/circuitbreakers 端点
```

> **口诀**：**线程池隔离仅在需要超时中断时用，否则信号量足够**。国内选 Sentinel，国际化选 Resilience4j。装饰器顺序决定语义。

---

### 案例 ④ Gateway WebFlux 阻塞 —— "EventLoop 全卡死"

#### 现象

Spring Cloud Gateway 上线后，CPU 跑到 30% 时就开始大量 timeout。jstack 看 `reactor-http-nio-*` 线程全在 socketRead 或 wait。流量并不大（5000 QPS），按理 Gateway 能扛 5 万。

#### 根因

某个团队成员写了个鉴权 Filter，里面调了 RedisTemplate（同步阻塞）：

```java
@Component
public class AuthFilter implements GlobalFilter {
    @Autowired private RedisTemplate<String, String> redisTemplate;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getHeaders().getFirst("Token");
        // ★ 这是同步阻塞调用！在 EventLoop 上！
        String userId = redisTemplate.opsForValue().get("token:" + token);
        if (userId == null) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        return chain.filter(exchange);
    }
}
```

WebFlux 的 EventLoop 数量 = `2 × CPU 核数`，8 核机器只有 16 个 EventLoop。一个 Redis 调用 1ms（看起来很快），16 个 EventLoop × 每秒 1000 次 = 16000 QPS 上限。**5000 QPS 时 EventLoop 利用率已经 31%，碰到 Redis 偶发 50ms 慢调用就崩**。

#### 修复

**铁律：WebFlux Filter 里禁止任何同步阻塞调用**。

方案 A：改用响应式客户端

```java
@Component
public class AuthFilter implements GlobalFilter {
    @Autowired private ReactiveRedisTemplate<String, String> reactiveRedis;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getHeaders().getFirst("Token");
        return reactiveRedis.opsForValue().get("token:" + token)
            .switchIfEmpty(Mono.defer(() -> {
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete().then(Mono.empty());
            }))
            .flatMap(userId -> chain.filter(exchange));
    }
}
```

方案 B：必须用同步 SDK 时，**`subscribeOn(Schedulers.boundedElastic())`** 切到独立线程池

```java
return Mono.fromCallable(() -> redisTemplate.opsForValue().get("token:" + token))
    .subscribeOn(Schedulers.boundedElastic())   // ★ 切线程
    .flatMap(userId -> { ... });
```

`Schedulers.boundedElastic()` 默认上限 `10 × CPU 核数` 线程，专门用来跑阻塞调用。

#### 怎么发现自己写了阻塞代码

加 BlockHound（运行时检测，开发环境用）：

```xml
<dependency>
    <groupId>io.projectreactor.tools</groupId>
    <artifactId>blockhound</artifactId>
    <version>1.0.8.RELEASE</version>
</dependency>
```

```java
public static void main(String[] args) {
    BlockHound.install();
    SpringApplication.run(GatewayApp.class, args);
}
```

任何在 Reactor 线程里的阻塞调用都会抛 `BlockingOperationError`，CI 跑出来直接卡住。

#### 验证

```bash
# 压测：5 万 QPS 持续 5 分钟
wrk -t 8 -c 500 -d 300 http://gateway/api/test

# 监控 EventLoop 状态
curl http://gateway/actuator/metrics/reactor.netty.eventloop.executions.count

# 期望：CPU 利用率 < 80%，EventLoop 全部 active 而不是 wait
```

> **口诀**：**WebFlux 的 EventLoop 是稀缺资源**。Filter 里同步阻塞 = 整个 Gateway 卡死。改响应式或 `subscribeOn(boundedElastic())`。BlockHound 上线前必跑。

---

### 案例 ⑤ Nacos 客户端 OOM —— "订阅了 8000 个服务"

#### 现象

某网关服务接入 Nacos 后稳定运行 3 个月，某天突然 OOM。dump 看到 `com.alibaba.nacos.client.naming.cache.ServiceInfoHolder` 持有 8000+ 个 `ServiceInfo` 对象，占了 1.5GB 堆。

#### 根因

网关需要根据动态路由规则转发到后端服务，但路由规则配错了：用了"自动发现所有服务"模式。

```java
// ❌ 危险代码
NamingService naming = NamingFactory.createNamingService(nacosUrl);
List<ServiceInfo> all = naming.getServicesOfServer(1, Integer.MAX_VALUE).getData();
for (ServiceInfo svc : all) {
    naming.subscribe(svc.getName(), event -> { ... });   // ★ 8000 次订阅
}
```

Nacos 客户端对每个订阅的服务都会：
- 维护一份服务实例列表（`List<Instance>`）
- 启动一个推送 Listener
- 在长轮询里附带这个服务的 MD5

**8000 个服务订阅 = 8000 个回调 + 缓存**，外加每次推送都要序列化反序列化，CPU 也跟着炸。

更隐蔽的版本：通过 `@DubboReference` 或 `@FeignClient` 大量声明远程接口，每个接口对应一个服务订阅。微服务拆得过细的项目（500+ 服务）很容易踩。

#### 修复

**方案 A**：明确显式订阅。网关只订阅它真正要转发的服务：

```java
// 从配置中心读路由规则
List<String> routedServices = configService.getRoutedServiceNames();
for (String svcName : routedServices) {
    naming.subscribe(svcName, listener);
}
```

**方案 B**：限制订阅数 + 监控

```java
NacosClientProperties props = NacosClientProperties.builder()
    .serverAddr(nacosUrl)
    .property("namingClientBeatThreadCount", "1")
    .property("namingPollingThreadCount", "1")
    .build();

// 订阅数监控
Gauge.builder("nacos.subscribed.services",
    serviceInfoHolder, h -> h.getServiceInfoMap().size())
    .register(meterRegistry);

// 告警阈值：单实例订阅 > 500 → 警告
```

**方案 C**：升级到 Nacos 2.x。2.x 用 gRPC 长连接替代 HTTP 长轮询，内存占用降低约 60%，且可以多个服务复用同一连接。

```yaml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: nacos:8848
        # Nacos 2.x 默认 gRPC，端口 9848（自动派生）
```

#### 验证

```bash
# 启动后看 Nacos 客户端订阅数
curl http://app:port/actuator/metrics/nacos.subscribed.services

# 内存比对
jcmd <pid> GC.heap_info
# 升级 Nacos 2.x 后内存应明显下降
```

> **口诀**：**Nacos 订阅数是隐藏的内存炸弹**。只订阅真正要用的服务，监控订阅数，能升 2.x 就升。

---

### 案例 ⑥ Feign 默认超时太长 —— "灰度服务挂了，全链路 30s 超时"

#### 现象

灰度环境某个 Service B 实例假死（接 TCP 但不响应），上游 Service A 通过 Feign 调用 B，**所有打到这个假死实例的请求都等 30+ 秒**。监控告警炸锅。

#### 根因

Feign 默认配置（不显式配置时）：

```java
public static class Default implements Options {
    public Default() {
        this.connectTimeoutMillis = 10 * 1000;   // 10 秒
        this.readTimeoutMillis = 60 * 1000;       // 60 秒
        this.followRedirects = true;
    }
}
```

`readTimeout = 60 秒` 是默认。配合 LoadBalancer 在多实例间轮询，**一个慢实例能拖垮整个链路**。

更糟：很多人 yaml 里配的是错的 key，根本没生效：

```yaml
# ❌ 这是 Spring Cloud 1.x / Feign 9.x 的老 key
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 5000

# ✅ 当前正确写法
feign:
  client:
    config:
      default:
        connectTimeout: 1000
        readTimeout: 3000
```

#### 修复

**全局兜底超时**：

```yaml
feign:
  client:
    config:
      default:
        connectTimeout: 1000     # 1 秒建连超时
        readTimeout: 3000        # 3 秒读超时
        loggerLevel: BASIC
```

**针对某个慢服务调大**：

```yaml
feign:
  client:
    config:
      slow-report-service:
        readTimeout: 10000       # 报表服务允许 10 秒
```

**配 Resilience4j 兜底熔断**（推荐）：

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
        slowCallDurationThreshold: 2s
        slowCallRateThreshold: 50
```

```java
@FeignClient(name = "service-b", fallback = ServiceBFallback.class)
public interface ServiceBClient { ... }

@Component
public class ServiceBFallback implements ServiceBClient {
    @Override
    public Order getOrder(Long id) {
        return Order.empty();   // 降级返回
    }
}
```

#### 假死实例怎么快速摘除

注册中心层面：

```yaml
spring:
  cloud:
    nacos:
      discovery:
        heart-beat-interval: 5000      # 默认 5s
        heart-beat-timeout: 15000       # 15s 没心跳标记不健康
        ip-delete-timeout: 30000        # 30s 没心跳删除
```

但 Nacos 心跳是基于 TCP 的：**假死场景（TCP 还活着，应用线程死）心跳照样发**。这种情况要靠 K8s readinessProbe + 健康检查端点：

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3      # 3 次失败 = 30 秒
```

健康检查必须**真的反映应用健康**，不要返回 `200 OK` 静态字符串。

#### 验证

```bash
# 模拟假死
kubectl exec <pod> -- kill -STOP 1   # 暂停进程

# 上游应在 3 秒内超时（不是 60 秒）
time curl http://service-a/order/123
# real   0m3.012s
```

> **口诀**：**Feign 默认 60s read timeout 是定时炸弹**。生产配置必须显式覆盖到 1~3s。配 Fallback 兜底，配 K8s readinessProbe 摘假死。

---

### 案例 ⑦ Dubbo 序列化升级 —— "Hessian 不支持 Java 17 record"

#### 现象

服务从 JDK 11 升到 JDK 17，业务代码新写了一个 `public record OrderDto(Long id, String name) {}`。Dubbo 调用时报：

```
com.alibaba.com.caucho.hessian.io.HessianProtocolException:
'com.foo.OrderDto' could not be instantiated
```

#### 根因

Hessian2（Dubpo 默认序列化协议）依赖反射 + 默认无参构造器实例化对象。**Java record 没有无参构造器**，Hessian 不支持。

更普遍的问题：Hessian 是 2009 年设计的，没考虑：
- record / sealed class / pattern matching
- LocalDateTime / Optional 等 JDK 8+ 类型（要扩展才支持）
- 协议本身的安全漏洞（CVE 多次曝光）

Dubbo 官方在 3.x 已建议**新项目用 Protobuf 或 Hessian-Lite**。

#### 序列化协议对比

| 协议 | 兼容 record | 性能 | 跨语言 | 协议体积 |
| --- | --- | --- | --- | --- |
| **Hessian2** | ❌ | 中 | 弱（C++/Python 有库但不维护） | 小 |
| **Hessian-Lite** | ✅（Dubbo 改造） | 中 | 同上 | 小 |
| **Protobuf** | ✅（IDL 优先） | 高 | 强 | 最小 |
| **JSON** | ✅ | 低 | 极强 | 大 |
| **Kryo** | ✅ | 极高 | 弱 | 中 |
| **Fury**（Apache，原 Fast-Serialization） | ✅ | 极高 | 强 | 中 |

#### 修复

**方案 A**：迁到 Protobuf（Dubbo 3 + Triple 协议推荐）

定义 IDL：

```protobuf
syntax = "proto3";
package com.foo;

message OrderDto {
  int64 id = 1;
  string name = 2;
}
```

生成的 Java 类天然支持 builder + 序列化，且跨语言。

但代价：业务对象都要走 IDL，**对象模型受 Protobuf 约束**（比如不能用 Java 集合，只能用 `repeated`）。

**方案 B**：换 Fury（保持原对象模型）

```xml
<dependency>
    <groupId>org.apache.fury</groupId>
    <artifactId>fury-core</artifactId>
    <version>0.5.0</version>
</dependency>
```

Dubbo 配置：

```yaml
dubbo:
  protocol:
    serialization: fury2
```

Fury 性能比 Hessian2 快 2~5 倍，且原生支持 record / 现代 Java 类型。

**方案 C**：保留 Hessian 但避开 record

```java
// 改成普通 class
public class OrderDto {
    private Long id;
    private String name;
    public OrderDto() {}                  // ★ 必须无参构造器
    // getter/setter
}
```

最稳但失去了 record 的简洁。

#### 升级注意

序列化协议升级**必须双端兼容期**。先发布双协议提供者：

```yaml
dubbo:
  protocols:
    dubbo:
      port: 20880
      serialization: hessian2
    triple:
      port: 50051
      serialization: protobuf
```

让消费者按版本选择协议，全部升级完后再下线老协议。

#### 验证

```bash
# 单元测试覆盖所有 DTO 的序列化往返
@Test
void roundTripAllDtos() {
    OrderDto o = new OrderDto(1L, "x");
    byte[] bytes = serializer.serialize(o);
    OrderDto o2 = serializer.deserialize(bytes, OrderDto.class);
    assertEquals(o, o2);
}
```

> **口诀**：**Hessian2 是上古协议，不支持 record / 安全漏洞多**。Dubbo 3 推 Protobuf（受 IDL 约束），保持原模型选 Fury。升级必双端并行期。

---

### 案例 ⑧ 全家桶版本不兼容 —— "升 SpringBoot 后 Sentinel 启动失败"

#### 现象

把 SpringBoot 从 2.7 升到 3.0（Spring Framework 6 + Jakarta EE），项目启动报：

```
NoClassDefFoundError: javax/servlet/Filter
   at com.alibaba.csp.sentinel.adapter.spring.webmvc...
```

或：

```
ClassNotFoundException: org.springframework.web.bind.annotation.ResponseBody
```

#### 根因

Spring Boot 3 的关键 break change：
- **JDK 最低 17**
- **包名 `javax.*` → `jakarta.*`**（Servlet / JPA / Validation）
- 大量中间件 SDK 没及时跟进

Spring Cloud 与 Spring Cloud Alibaba 的版本兼容矩阵：

| Spring Boot | Spring Cloud | Spring Cloud Alibaba |
| --- | --- | --- |
| 2.6.x | 2021.0.x | 2021.0.4.0 |
| 2.7.x | 2021.0.x / 2022.0.x | 2021.0.5.0 / 2022.0.0.0 |
| **3.0.x** | **2022.0.x** | **2022.0.0.0+**（部分组件） |
| **3.1.x** | **2022.0.x / 2023.0.x** | **2022.0.0.0-RC2+** |
| **3.2.x** | **2023.0.x** | **2023.0.1.0** |

Sentinel 1.8.6 才正式支持 SpringBoot 3 / Jakarta EE。Nacos 2.2.x 才完全适配。

#### 修复

**升级前必查**：到官网看版本兼容矩阵，**所有依赖必须同代升级**。

```xml
<!-- 父 POM -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.5</version>
</parent>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2023.0.1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>2023.0.1.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

**代码里全局替换**：

```bash
# 包名替换（小心 javax.crypto / javax.sql 这些不变的）
find src -name "*.java" -exec sed -i '' \
  -e 's/javax\.servlet/jakarta.servlet/g' \
  -e 's/javax\.persistence/jakarta.persistence/g' \
  -e 's/javax\.validation/jakarta.validation/g' \
  {} \;
```

**留一份兼容层**：用 `org.eclipse.transformer` 工具把 javax 编译期替换成 jakarta，老依赖也能跑。

#### 升级路径建议

不要一步升 2.x → 3.x。中间走 2.7 stable 最久，等：
- Spring Cloud Alibaba 出稳定 GA
- 你用的所有第三方 starter 都明确支持 3.x
- 团队 JDK 全部到 17+

**先做依赖兼容性扫描**：

```bash
mvn dependency:tree | grep -E "(spring-|nacos|sentinel|dubbo|seata)" | sort -u
```

#### 验证

```bash
# 启动跑通
mvn spring-boot:run

# 再压一遍接口验证全家桶组件都正常工作
# 监控指标 / 链路追踪 / 注册中心 / 限流规则全部检查
```

> **口诀**：**SpringBoot 3 是断崖式升级**。必须等所有 starter 同代支持，依赖矩阵对齐才能升。建议保留 2.7 长期使用，3.x 用于新项目。

---

### 案例 ⑨ Sleuth → Micrometer Tracing 迁移 —— "TraceId 在某些日志里丢了"

#### 现象

升级到 Spring Boot 3 后，原本 Spring Cloud Sleuth 提供的链路追踪迁移到 Micrometer Tracing。结果发现某些日志里 traceId 没有，链路在异步线程或调度任务处断了。

#### 根因

Sleuth（已停止维护）和 Micrometer Tracing 的核心区别：

| 维度 | Sleuth | Micrometer Tracing |
| --- | --- | --- |
| 维护方 | Spring（已停） | Micrometer 团队 |
| 后端 | Zipkin / Jaeger / Wavefront | Brave / OpenTelemetry |
| 异步传播 | 自动包装 `@Async` / `Reactor` | 需要显式 wrapping |
| 日志集成 | 自动注入 MDC | 需要 `LoggingTracingObservationHandler` |
| API | 自有抽象 | 标准 Tracing API |

最容易踩的坑：**Reactor / WebFlux 链路上下文传播**。

```java
// ❌ Sleuth 时代能自动传播
public Mono<Order> handle(Long id) {
    return webClient.get().uri("/order/" + id)
        .retrieve().bodyToMono(Order.class);
}

// Micrometer Tracing 默认不传 ContextSnapshot
```

#### 修复

**Reactor 上下文传播**：开 `Hooks.enableAutomaticContextPropagation()`

```java
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        Hooks.enableAutomaticContextPropagation();   // ★ Reactor 3.5+
        SpringApplication.run(App.class, args);
    }
}
```

**线程池场景**：用 `ContextSnapshot` 包装

```java
@Bean
public ThreadPoolTaskExecutor taskExecutor(ObservationRegistry registry) {
    ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
    exec.setTaskDecorator(runnable -> {
        ContextSnapshot snapshot = ContextSnapshotFactory.builder().build().captureAll();
        return snapshot.wrap(runnable);
    });
    return exec;
}
```

**手动 Span 场景**：

```java
@Autowired private Tracer tracer;
@Autowired private ObservationRegistry registry;

public void process() {
    Observation obs = Observation.start("process.order", registry);
    try (Observation.Scope scope = obs.openScope()) {
        // 业务逻辑，traceId 自动注入 MDC
        log.info("processing");
    } finally {
        obs.stop();
    }
}
```

**对接 OpenTelemetry**（推荐做长期投资）：

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

```yaml
management:
  tracing:
    sampling:
      probability: 1.0
  otlp:
    tracing:
      endpoint: http://otel-collector:4317
```

#### 迁移检查清单

- [ ] 所有 `org.springframework.cloud.sleuth.*` 改 `io.micrometer.tracing.*`
- [ ] `@NewSpan` → `@Observed`
- [ ] Reactor 项目开 `Hooks.enableAutomaticContextPropagation()`
- [ ] 自定义线程池包 ContextSnapshot
- [ ] 日志格式 `%X{traceId},%X{spanId}` 仍然有效（MDC 注入键名兼容）
- [ ] @Scheduled / @Async 任务里检查 traceId 不丢

#### 验证

```bash
# 灰度环境抽样检查
curl http://app/api/test
# 服务端日志应有完整 traceId
# 跨服务调用时下游 traceId 与上游一致
```

> **口诀**：**Sleuth 已死，Micrometer Tracing 接班**。Reactor 项目开自动上下文传播，线程池任务必须包 ContextSnapshot。投资 OpenTelemetry 是长期方向。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 选型层
- [ ] 明确"为什么不直接用 Mesh"（Java 主、SDK 治理够用 → SDK；多语言、强治理需求 → Mesh）
- [ ] 明确 RPC 协议选择（小消息高 QPS → Dubbo；HTTP 生态、灰度 → OpenFeign）
- [ ] 注册中心 / 配置中心选定一个并坚持（混用 Nacos + Eureka 是灾难）
- [ ] 熔断方案选定（Sentinel 或 Resilience4j），不要新老共存
- [ ] 链路追踪走 Micrometer Tracing + OpenTelemetry（不再用 Sleuth）

### 超时与重试
- [ ] OpenFeign 默认 60s readTimeout 必须显式覆盖到 1~3s
- [ ] 链路超时分层收敛（下游 < 上游 / 串行调用层数）
- [ ] LoadBalancer / Ribbon 重试默认关闭，业务侧显式开
- [ ] 上游超时后必须取消下游（OkHttp + cancel / Reactor 链）

### 注册中心
- [ ] Nacos 订阅数监控（单实例 > 500 告警）
- [ ] Nacos 升 2.x 用 gRPC（小内存）
- [ ] Dubbo 2 → 3 必须双注册过渡，不要直接切
- [ ] K8s readinessProbe 真的检查应用健康，不返静态 200

### 网关层
- [ ] Spring Cloud Gateway Filter 里禁止任何同步阻塞调用
- [ ] 必须阻塞时 `subscribeOn(Schedulers.boundedElastic())`
- [ ] 开发环境 BlockHound 必跑，CI 卡住阻塞代码
- [ ] 网关熔断兜底（路由失败有 fallback）
- [ ] 网关路由数 > 1000 评估升 Higress / Envoy

### 熔断与限流
- [ ] Sentinel `blockHandler` 与 `fallback` 区分清楚
- [ ] Resilience4j 装饰器顺序明确（CircuitBreaker → Retry vs 反向）
- [ ] 配置中心管理熔断规则，不写死代码里
- [ ] 慢调用熔断阈值配置（slowCallDurationThreshold + slowCallRateThreshold）

### 序列化
- [ ] Hessian 用户避开 record / sealed class
- [ ] 新项目优先 Protobuf 或 Fury
- [ ] 协议升级必双端并行期（提供者双协议导出）

### 版本管理
- [ ] Spring Boot / Spring Cloud / Spring Cloud Alibaba 严格按兼容矩阵升级
- [ ] 升级前依赖树扫描，确认所有 starter 同代支持
- [ ] SpringBoot 3 升级前确认所有 javax → jakarta 完成
- [ ] 长期保留一个 LTS 版本（Boot 2.7 / 3.2），不追新

### 链路追踪
- [ ] Reactor 项目开 `Hooks.enableAutomaticContextPropagation()`
- [ ] 自定义线程池包 ContextSnapshot
- [ ] @Scheduled / @Async 任务里 traceId 不丢
- [ ] 日志格式包含 traceId / spanId（`%X{traceId}`）

### 优雅停机
- [ ] `server.shutdown=graceful` + `spring.lifecycle.timeout-per-shutdown-phase=30s`
- [ ] K8s `terminationGracePeriodSeconds > 应用停机超时`
- [ ] preStop sleep 等 endpoint 摘除
- [ ] Dubbo `dubbo.application.shutdown.wait` 配置

### 内存预算
- [ ] 单 Pod 内存 = 业务 + 全家桶 + JVM自身（最少 1GB，全家桶项目 1.5GB+）
- [ ] 注册中心 / 配置缓存监控（Nacos ServiceInfoMap 大小）
- [ ] 链路追踪 Span 缓存上限明确（避免无界）

---

## 6. 还能往下挖什么

### Service Mesh 与 SDK 的边界

- **Istio + Envoy**：完整数据面治理，但运维成本高（Pilot/Citadel/Galley/Telemetry 一堆组件）
- **Dubbo Proxyless Mesh**：保持 SDK 调用，但通过 xDS 协议从控制面拉规则，少了 Sidecar 网络跳数
- **MOSN**：阿里云原生 Sidecar 实现，专门为 Java 应用优化
- 决策树：**多语言 + 强治理 → Mesh**；**Java 主、性能敏感 → Proxyless 或 SDK**

### Dubbo 3 与 Triple 协议

- Triple 协议规范文档（与 gRPC 兼容性的具体边界）
- 应用级服务发现的元数据中心设计（解决"接口级注册元数据爆炸"）
- Dubbo + GraalVM Native 的 AOT 处理细节
- 多 Region 部署时的服务路由策略

### Spring Cloud 的"国别版"

- **Spring Cloud Alibaba**：Nacos / Sentinel / Seata，国内事实标准
- **Spring Cloud Tencent**：Polaris 注册中心 + 治理
- **Spring Cloud Huawei**：Service Comb Service Center
- **Spring Cloud Azure / AWS**：云厂商绑定的对接组件
- 通用建议：**别绑死任何一家，用 Spring Cloud Commons 标准接口抽象**

### 网关进阶

- **Apache ShenYu**：轻量 Java 网关，插件化，性能比 Gateway 略好
- **Higress**：阿里开源，Envoy + WASM 插件，云原生友好
- **APISIX**：OpenResty 系，多协议（HTTP / gRPC / Dubbo / MQTT）
- **Kong**：商业生态强，企业版功能多
- 选型：**Java 团队主用 Java 写 Filter → Gateway / ShenYu**；**性能极致 + 多协议 → APISIX / Higress**

### 多语言微服务的混部

- Dubbo 3 + Triple：Java 服务和 Go 服务用 gRPC 互通
- gRPC + Envoy：完全标准化的 RPC + 治理
- ProtoBuf IDL 共享：不同语言 build 出对应的 stub
- 实际挑战：**链路追踪跨语言传播**（W3C Trace Context 标准）

### GraalVM Native + 微服务

- Spring Boot 3 + Spring Native 实验性支持
- Dubbo 3 AOT 编译路径
- 启动 100ms / 内存 80MB 的 Serverless 友好场景
- 当前限制：反射多的库（Hessian、Hibernate）需要 reachability metadata

### 微服务治理标准化

- **OpenSergo**：阿里发起的微服务治理标准（流量路由、限流降级标准化）
- **OpenTelemetry**：可观测性标准（取代 OpenTracing + OpenCensus）
- 价值：**让 Spring Cloud / Dubbo / Mesh 走同一套治理 API**

---

## 7. 引用

- Spring Cloud 官方文档：[https://spring.io/projects/spring-cloud](https://spring.io/projects/spring-cloud)
- Spring Cloud Alibaba 官方文档：[https://sca.aliyun.com](https://sca.aliyun.com)
- Apache Dubbo 3 文档：[https://cn.dubbo.apache.org/zh-cn/overview/what/](https://cn.dubbo.apache.org/zh-cn/overview/what/)
- Nacos 官方文档：[https://nacos.io](https://nacos.io)
- Sentinel 官方文档：[https://sentinelguard.io/zh-cn/](https://sentinelguard.io/zh-cn/)
- Resilience4j 文档：[https://resilience4j.readme.io](https://resilience4j.readme.io)
- Spring Cloud Gateway 源码：[https://github.com/spring-cloud/spring-cloud-gateway](https://github.com/spring-cloud/spring-cloud-gateway)
- Triple 协议规范：[https://cn.dubbo.apache.org/zh-cn/overview/reference/protocols/triple/](https://cn.dubbo.apache.org/zh-cn/overview/reference/protocols/triple/)
- OpenTelemetry 项目：[https://opentelemetry.io](https://opentelemetry.io)
- Istio 文档：[https://istio.io](https://istio.io)
- 《微服务架构设计模式》—— Chris Richardson
- 《Cloud Native Patterns》—— Cornelia Davis

---

> 配套：[分布式系统主题](/article/distributed/) | [注册中心对比](/article/distributed/service-discovery) | [限流降级熔断](/article/system-design/resilience) | [Service Mesh](/article/k8s/service-mesh)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
