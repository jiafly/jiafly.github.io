# 分布式链路追踪 —— OpenTelemetry / SkyWalking / Jaeger 工程化全景

> 这是按 [桃花岛深度模板](/STYLE) 写的链路追踪深文。如果你只是想知道"什么是 Trace / Span"，看 [可观测性总览](/article/observability/) 那一节够了。
> 这里要回答的，是**接入了 SkyWalking 但链路总在线程池处断、上线后 Span 把 ES 打挂、Service Map 怎么也对不上真实拓扑**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过：

1. W3C Trace Context 的 `traceparent` 头长什么样？里面 4 个字段分别是什么含义？为什么把 sampled flag 放在 traceparent 里而不是 tracestate？
2. 你接了 SkyWalking 后，业务里 `executor.submit(task)` 的 task 里日志没 traceId、Span 也对不上 —— 链路在线程池处断了。**根因**是什么？为什么 SkyWalking Agent "号称无侵入"还会断？
3. **头采样**（head-based sampling）和**尾采样**（tail-based sampling）的核心区别是什么？为什么"全采样错误请求"必须用尾采样？尾采样的工程代价是什么？
4. 一个新项目让你选 OpenTelemetry / SkyWalking / Jaeger，你的决策树是什么？三者最关键的差异点（不是"功能列表"）在哪？
5. 一个服务每秒 5 万 QPS、平均 20 个 Span/请求，全采样 Span 数据 = 100 万/秒。按 ES 一条 Span ~1KB 算，一天产生约 86GB —— 你怎么把这个量降到可接受范围？给 3 种方法和它们的代价。
6. 跨 RPC（HTTP / gRPC / Dubbo）的 context 传递机制是什么？跨 MQ（Kafka / RocketMQ）呢？为什么 MQ 场景"接收方时间"和"发送方时间"画到一张时间线上会非常奇怪？
7. Service Map（服务拓扑图）是从 Span 数据推导出来的，不是配置出来的。**推导算法**是什么？你看到 Service Map 上有一根从 A 指向 B 的边，但实际上 A 从来不调 B —— 可能是什么原因？
8. eBPF 实现的链路追踪（Pixie / Beyla）号称"零侵入"。它能拿到哪些信息？拿不到哪些？为什么不能完全替代 SDK 接入？

如果有半数答不上，往下看。

---

## 1. 底层模型：Trace、Span、Context Propagation

绝大多数文章把链路追踪讲成"加个 Agent，自动有图"，把 Trace 和 Span 的概念草草带过。这层抽象解释不了上面 8 个问题。要看懂它们，必须下沉到三个层次：**数据模型、传播协议、采样存储**。

### 1.1 Trace / Span / SpanContext 数据模型

一条调用链的物理结构：

```
Trace（一次端到端调用，由 traceId 标识）
 ├── Span A（一段工作单元，有 spanId、parentSpanId、起止时间、属性）
 │    ├── Span B
 │    │    └── Span D
 │    └── Span C
 └── Span E（与 A 同 trace 但跨服务边界）
```

每个 Span 至少包含：

| 字段 | 含义 | 例子 |
| --- | --- | --- |
| `traceId` | 整条链路的全局唯一 ID | 16 字节，128 bit |
| `spanId` | 本 Span 的 ID | 8 字节，64 bit |
| `parentSpanId` | 父 Span 的 ID（根 Span 为空） | 8 字节 |
| `name` | 操作名 | `GET /api/order` |
| `startTime` / `endTime` | 起止时间（纳秒精度） | `1700000000123456789` |
| `attributes` | 键值对（标签） | `{"http.status_code": 200}` |
| `events` | 时间点事件 | `{"timestamp": ..., "name": "exception"}` |
| `status` | OK / Error / Unset | |
| `kind` | SERVER / CLIENT / PRODUCER / CONSUMER / INTERNAL | 决定 Span 在拓扑里怎么连 |

**SpanContext** 是 Span 中"必须跨进程传递的最小子集"：`traceId + spanId + traceFlags + traceState`。它就是后面 W3C Trace Context 协议要传输的内容。

### 1.2 W3C Trace Context 标准

2020 年成为 W3C Recommendation，是目前最主流的跨语言传递格式。HTTP 头形式：

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
                ▲    ▲                                ▲                ▲
                │    │                                │                │
              version traceId(16字节)             parentSpanId(8字节)  flags
```

4 段含义：

| 段 | 长度 | 含义 |
| --- | --- | --- |
| `version` | 1 字节（hex 2 字符） | 协议版本，目前固定 `00` |
| `traceId` | 16 字节（hex 32 字符） | 全局 trace ID，全 0 非法 |
| `parentSpanId` | 8 字节（hex 16 字符） | 上游 Span ID（接收方的 parent） |
| `flags` | 1 字节（hex 2 字符） | bit 0 = sampled（该 trace 是否被采样） |

**为什么把 sampled flag 放在 traceparent 而不是 tracestate？** 因为 sampled 决策必须**全链路一致** —— 上游决定采样，下游必须跟着采样，否则 trace 链路不全。把它放在标准化的 traceparent 里，所有 vendor 的 SDK 都必须遵守。

**tracestate** 是 vendor 特定的扩展：

```
tracestate: vendor1=value1,vendor2=value2
```

每个 vendor 可以塞自己的元数据（如 SkyWalking 的 segment ID、阿里 ARMS 的 EagleEye ID）。最多 32 个 key、512 字节。

> 早期百花齐放：Zipkin 用 `X-B3-TraceId` 系列、SkyWalking 用 `sw8`、阿里用 `EagleEye-TraceId`。**现在新项目应直接选 W3C，避免互通成本**。

### 1.3 头采样 vs 尾采样

采样是工程上**最关键也最容易做错**的决策。

**头采样（head-based sampling）**：
- 决策时机：trace 入口（第一个 Span 创建时）
- 决策依据：traceId 哈希、固定比例（如 1%）、或上游已决策的 sampled flag
- 优点：实现极简，开销极低（决策一次，所有服务跟随）
- 缺点：**采样在你看到结果之前就决定了** —— 错误请求 / 慢请求很可能没采到

```
请求进入 → traceparent flags = 01（采样）→ 整链路所有 Span 都上报
请求进入 → traceparent flags = 00（不采样）→ 整链路所有 Span 都丢
```

**尾采样（tail-based sampling）**：
- 决策时机：等整条 trace 跑完
- 决策依据：链路里有没有 error、总耗时是否 > P99 阈值、是否命中关键服务
- 优点：**100% 采样错误和慢请求**
- 缺点：必须先把 trace 全收下来缓存（按 traceId 聚合），才能决策；**Collector 内存压力大、要分布式（同 traceId 必须在同一个 Collector）**

```
所有 Span → Collector 按 traceId 缓存（窗口 30s）→ 窗口结束触发决策 → 该丢的丢
```

工程上常见的是**混合**：
- 头采样兜底（如 1%）保证有"基础流量画像"
- 尾采样针对错误 / 慢请求做"全采样"

> 业内常说"头采样更便宜、尾采样更聪明"。但**尾采样在 Collector 层的负担、内存、网络重定向**远比想象的复杂。中等规模团队没专人维护时，**头采样 + 错误前后的"放大窗口"**（错误发生后 N 秒内强制采样）是更务实的选择。

### 1.4 Context Propagation：HTTP / gRPC / MQ 三种通道

跨进程传递 SpanContext 必须依赖底层协议提供"自定义元数据"的能力。

**HTTP**：放在 Header 里（W3C 标准就是 HTTP Header）。

```
GET /api/order HTTP/1.1
Host: order-service
traceparent: 00-4bf92f...-00f067aa...-01
tracestate: ot=p:8;r:62
```

**gRPC**：放在 Metadata 里（gRPC 的 Metadata 也是 Header 风格）。

```java
Metadata md = new Metadata();
md.put(Metadata.Key.of("traceparent", ASCII_STRING_MARSHALLER), value);
```

**MQ（Kafka / RocketMQ）**：放在消息 Header 里（Kafka 0.11+ 支持 record headers）。

```java
ProducerRecord<String, String> record = new ProducerRecord<>("topic", "key", "value");
record.headers().add("traceparent", value.getBytes());
```

MQ 场景的**特殊性**：
- 发送方的 Span 是 PRODUCER kind，结束于"消息发出去"
- 接收方的 Span 是 CONSUMER kind，开始于"消费消息"
- 中间可能间隔几秒、几分钟、甚至几小时（消息堆积）
- 画到一张甘特图上看起来像"中间一段巨大的空白" —— 这是**正常的**，不是 bug

> 跨 MQ 的链路追踪经常被忽略：发送方接了，但消费者侧没装 Agent / 装了但没正确从 Header 解出 context。结果 MQ 之后整条链路都断了。**生产巡检时把 PRODUCER / CONSUMER 配对检查列入 checklist**。

### 1.5 Span 的存储与查询模型

Span 数据的特点：
- **写多读少**（采集量大，查询量小但要快）
- **时序性强**（按时间分片）
- **高基数**（traceId、spanId 几乎不重复）
- **关联查询**（按 traceId 聚合 Span 形成链路）

主流存储选择：

| 存储 | 优点 | 缺点 | 谁在用 |
| --- | --- | --- | --- |
| **Elasticsearch** | 查询灵活、生态好 | 写入开销大、磁盘占用大、TCO 高 | Jaeger / SkyWalking 默认 |
| **Cassandra** | 写入吞吐高、水平扩展好 | 查询能力弱、运维复杂 | Jaeger 早期 |
| **ClickHouse** | 列存、压缩高、聚合快 | 实时性稍弱 | SigNoz / Uptrace / 字节跳动 |
| **HBase** | 写入快、扩展好 | 二级索引弱 | 早期 Zipkin |
| **自研 TSDB** | 完全契合追踪场景 | 投入大 | 阿里 EagleEye、Google Dapper |

近两年的趋势是 **ClickHouse 替代 ES**：相同硬件下 ClickHouse 的存储压缩率高 5-10 倍，查询聚合快 3-5 倍，特别适合 Span 这种"高基数 + 时序 + 聚合"的负载。

### 1.6 OpenTelemetry 的"统一三件套"愿景

OpenTelemetry（OTel）是 CNCF 毕业项目，由 OpenTracing + OpenCensus 合并而来。它的核心愿景是：

```
Application
    │
    ▼
OpenTelemetry SDK（数据生成）
    │
    ▼
OpenTelemetry Protocol (OTLP，统一传输协议)
    │
    ▼
OpenTelemetry Collector（统一采集层）
    │
    ├──► Jaeger（看 Trace）
    ├──► Prometheus（看 Metrics）
    ├──► Loki（看 Logs）
    └──► 任意厂商后端（New Relic / Datadog / 阿里 ARMS / ...）
```

三件套统一：**Trace / Metrics / Logs 同一个 SDK、同一个协议、同一个采集层**。这是过去十年可观测性领域最重要的标准化运动。

| 维度 | OpenTelemetry | SkyWalking | Jaeger |
| --- | --- | --- | --- |
| 定位 | **标准 + SDK + Collector** | 端到端 APM 平台 | 后端存储 + UI |
| 数据生成 | SDK / Auto Agent（多语言） | Java Agent 字节码增强 | 依赖 SDK 或 OTel |
| 采集 | OTel Collector | OAP Server | 依赖 Agent / SDK |
| 存储 | 不绑定 | ES / H2 / TiDB / BanyanDB | ES / Cassandra / Badger |
| UI | 无（用 Jaeger / Grafana） | 自带 | 自带 |
| Service Map | 通过 Collector + 后端 | **自带、强项** | 弱 |
| 性能（agent） | 较好 | 略高（字节码改造多） | 依赖 SDK |

**怎么选**：
- 多语言、追求标准化、想避免厂商锁定 → **OpenTelemetry**（推荐 SDK + Collector，UI 用 Jaeger / Grafana Tempo）
- Java 为主、要"开箱即用的 APM 平台"（拓扑、告警、SQL 分析） → **SkyWalking**
- 只要一个看 Trace 的后端 + UI → **Jaeger**（搭配任何 SDK）

> 趋势是 **OpenTelemetry 成为事实标准**：SkyWalking 已支持 OTLP 接入，Jaeger 主推 OTel SDK 弃用自有 client。新项目 2024+ 直接 OTel。

---

## 2. 源码切片：上下文传播的工程细节

> 基于 OpenTelemetry Java SDK 1.32、SkyWalking Java Agent 9.x。注释为本文添加。

### 2.1 OpenTelemetry SDK 的 ContextStorage（ThreadLocal）

OTel 的 `Context` 是不可变对象，挂在 ThreadLocal 上：

```java
// io.opentelemetry.context.ThreadLocalContextStorage
enum ThreadLocalContextStorage implements ContextStorage {
    INSTANCE;

    private static final ThreadLocal<Context> THREAD_LOCAL_STORAGE =
        new ThreadLocal<>();

    @Override
    public Scope attach(Context toAttach) {
        Context beforeAttach = current();
        if (toAttach == beforeAttach) return Scope.noop();

        THREAD_LOCAL_STORAGE.set(toAttach);
        return () -> {
            // ★ Scope.close() 必须恢复原 Context，否则后续任务串
            if (current() != toAttach) {
                logger.log(Level.FINE, "Context in storage not the expected ...");
            }
            THREAD_LOCAL_STORAGE.set(beforeAttach);
        };
    }

    @Override
    public Context current() {
        Context current = THREAD_LOCAL_STORAGE.get();
        return current == null ? Context.root() : current;
    }
}
```

**逐段点透**：

- **Context 是不可变（immutable）的**：每次 `with(key, value)` 返回新 Context，旧 Context 不变。这是为了让"父 Span 的 context"在子任务里也可以独立修改而不互相影响。
- **`attach` 返回 Scope，必须 `try-with-resources` 用**：忘记 close → ThreadLocal 残留 → 下一个任务串上下文。这是后面案例 ⑤ 的根。
- **`Scope.close()` 内的 `if (current() != toAttach)` 警告**：如果你 close 的不是当前正在 attach 的 Context，说明上下文嵌套逻辑出错，会打 fine 级别日志。生产上看到这条日志要严查。

### 2.2 SkyWalking Java Agent 的字节码增强

SkyWalking Agent 的核心机制是 **JavaAgent + Byte Buddy** 字节码改造：

```java
// org.apache.skywalking.apm.agent.SkyWalkingAgent
public static void premain(String agentArgs, Instrumentation instrumentation) {
    // ① 加载所有 plugin（每个 plugin 描述"对哪个类的哪个方法做什么增强"）
    final PluginFinder pluginFinder = new PluginFinder(
        new PluginBootstrap().loadPlugins());

    // ② 用 Byte Buddy 注册 Transformer
    new AgentBuilder.Default(byteBuddy)
        .ignore(...)
        .type(pluginFinder.buildMatch())            // 匹配 Class
        .transform((builder, typeDescription, classLoader, module, pd) -> {
            return new Transformer(pluginFinder).transform(
                builder, typeDescription, classLoader, module);
        })
        .with(new Listener())
        .installOn(instrumentation);
}
```

每个插件定义"在哪个方法的入口/出口/异常处插什么代码"。例如 `tomcat-7.x-8.x` 插件拦截 `org.apache.catalina.core.StandardHostValve#invoke`，在入口处：
- 从 HTTP Header 解出上游 traceparent
- 创建 EntrySpan（SERVER kind）
- 把 SpanContext 放进 ContextManager

```java
// TomcatInvokeInterceptor 简化
@Override
public void beforeMethod(EnhancedInstance objInst, Method method,
                         Object[] allArguments, Class<?>[] argumentsTypes,
                         MethodInterceptResult result) {
    Request request = (Request) allArguments[0];
    ContextCarrier carrier = new ContextCarrier();
    CarrierItem next = carrier.items();
    while (next.hasNext()) {
        next = next.next();
        next.setHeadValue(request.getHeader(next.getHeadKey()));   // ★ 从 HTTP Header 提取
    }
    AbstractSpan span = ContextManager.createEntrySpan(
        request.getRequestURI(), carrier);
    span.setComponent(ComponentsDefine.TOMCAT);
    SpanLayer.asHttp(span);
    Tags.URL.set(span, request.getRequestURL().toString());
}
```

**逐段点透**：

- **EnhancedInstance 是注入的接口**：Byte Buddy 给被增强的类加了 `EnhancedInstance` 接口，提供 `getSkyWalkingDynamicField` / `setSkyWalkingDynamicField` —— 让 Agent 能在不修改源类结构的前提下挂数据。
- **ContextCarrier**：把 SpanContext 序列化为可放进 Header 的字符串。SkyWalking 自有协议是 `sw8`，OTel 接入后可换 `traceparent`。
- **`createEntrySpan` 内部**：从当前线程 ContextManager 取 TraceSegment（一个服务内的 Span 链），如果没有就新建，并把 carrier 里的上游链路信息接上。

### 2.3 跨线程池 Context 传递（Runnable 装饰器）

ThreadLocal 不会自动跨线程。OTel 提供 `Context.taskWrapping`：

```java
// io.opentelemetry.context.Context
default Runnable wrap(Runnable runnable) {
    return () -> {
        try (Scope ignored = makeCurrent()) {       // ★ 在 worker 上 attach
            runnable.run();
        }
    };
}

default Executor wrap(Executor executor) {
    return command -> executor.execute(current().wrap(command));
}

default ExecutorService wrap(ExecutorService executor) {
    return new ContextPropagatingExecutorService(executor);
}
```

正确用法：

```java
ExecutorService raw = Executors.newFixedThreadPool(4);
ExecutorService wrapped = Context.taskWrapping(raw);

// 提交任务时自动捕获当前 Context，在 worker 线程上恢复
wrapped.submit(() -> {
    Span span = Span.current();   // ★ 拿到的是提交线程的 Span
});
```

**逐段点透**：

- **`current().wrap(command)` 在提交时调用**：此时 `current()` 是提交线程的 Context。返回的 Runnable 闭包捕获了它。
- **worker 执行时 `makeCurrent()` 把闭包里的 Context attach 到 worker 的 ThreadLocal**：try-with-resources 保证 worker 跑完后恢复。
- **没有这层包装，worker 调用 `Span.current()` 拿到的是 `Span.getInvalid()`**（noop Span），所有 instrumentation 都"看不见"父 Span。

SkyWalking 的等价机制：`org.apache.skywalking.apm.toolkit.activation.runnable.RunnableInterceptor` —— 但**默认不开**！需要业务代码主动用 `@TraceCrossThread` 注解或换成 toolkit 提供的 `RunnableWrapper`。

```java
// 用 @TraceCrossThread 注解（需引入 apm-toolkit-trace）
@TraceCrossThread
public class MyTask implements Runnable { ... }

// 或包一层
executor.submit(RunnableWrapper.of(() -> { ... }));
```

> SkyWalking "无侵入"在跨线程池场景**站不住** —— 你要么改业务代码加注解，要么用 toolkit 包装。这是案例 ① 的根。

### 2.4 跨 MQ Context 传递（Header 注入）

发送侧（Producer）：

```java
// OTel KafkaProducerInstrumentation 简化
public ProducerRecord<K, V> beforeSend(ProducerRecord<K, V> record) {
    Context parent = Context.current();
    SpanBuilder builder = tracer.spanBuilder(record.topic() + " send")
        .setSpanKind(SpanKind.PRODUCER)
        .setParent(parent);
    Span span = builder.startSpan();

    // ★ 把 traceparent 注入消息 Header
    propagator.inject(Context.current().with(span), record.headers(), SETTER);
    return record;
}

// SETTER 实现
private static final TextMapSetter<Headers> SETTER = (headers, key, value) -> {
    headers.remove(key);
    headers.add(key, value.getBytes(StandardCharsets.UTF_8));
};
```

接收侧（Consumer）：

```java
public void onMessage(ConsumerRecord<K, V> record) {
    Context parent = propagator.extract(Context.current(), record.headers(), GETTER);
    Span span = tracer.spanBuilder(record.topic() + " process")
        .setSpanKind(SpanKind.CONSUMER)
        .setParent(parent)                        // ★ 接续上游
        .startSpan();
    try (Scope scope = span.makeCurrent()) {
        // 真正处理消息
        handle(record);
    } finally {
        span.end();
    }
}
```

**逐段点透**：

- **`propagator.inject` / `extract`** 是 OTel 的核心抽象，把"具体协议"（HTTP Header / Kafka Header / gRPC Metadata）和"标准格式"（W3C / B3）解耦。Setter 和 Getter 决定怎么读写底层载体。
- **CONSUMER Span 的 parent 是上游 PRODUCER**：所以同一条消息的发送和消费会落在同一个 trace 下，但中间有时间间隔。
- **如果 MQ 中间件用了重试 / 死信队列**：每次重试 Consumer 都会创建新 Span，parent 不变，trace 看起来会有多个并列的 CONSUMER 分支 —— 这是好事，方便诊断重试链路。

### 2.5 Sampler 接口与 ParentBased / TraceIdRatioBased

OTel SDK 的采样接口：

```java
public interface Sampler {
    SamplingResult shouldSample(
        Context parentContext,
        String traceId,
        String name,
        SpanKind spanKind,
        Attributes attributes,
        List<LinkData> parentLinks);

    String getDescription();
}

// 决策结果：丢弃 / 记录但不采样 / 记录且采样
public enum SamplingDecision {
    DROP,
    RECORD_ONLY,
    RECORD_AND_SAMPLE;
}
```

最常用两个内置实现：

```java
// 1. 按 traceId hash 等比例采样
public final class TraceIdRatioBasedSampler implements Sampler {
    private final long idUpperBound;        // ratio * Long.MAX_VALUE

    @Override
    public SamplingResult shouldSample(...) {
        // 取 traceId 后 8 字节作为 long，与上界比较
        return Math.abs(getTraceIdRandomPart(traceId)) < idUpperBound
            ? POSITIVE_SAMPLING_RESULT
            : NEGATIVE_SAMPLING_RESULT;
    }
}

// 2. 跟随父 Span 的决策
public final class ParentBasedSampler implements Sampler {
    private final Sampler root;
    @Override
    public SamplingResult shouldSample(Context parentContext, ...) {
        SpanContext parent = Span.fromContext(parentContext).getSpanContext();
        if (parent.isValid()) {
            // ★ 父 Span 有效 → 跟随父 Span 的 sampled flag
            return parent.isSampled() ? RECORD_AND_SAMPLE : DROP;
        }
        // 根 Span → 走真正的策略（通常是 TraceIdRatio）
        return root.shouldSample(parentContext, ...);
    }
}
```

生产推荐配置：

```java
Sampler sampler = Sampler.parentBased(
    Sampler.traceIdRatioBased(0.01)          // 根 Span 1% 采样
);
```

含义：根 Span 按 1% 采样；子 Span 跟随根的决策（被采样的 trace 全部 Span 都上报，未采样的全丢）。**保证 trace 完整性**。

**反例**：如果上游 ParentBased、下游纯 TraceIdRatioBased：

```
A（采样了，发出 sampled=1） → B（自己又随机 1%）→ 99% 概率 B 不采样
→ trace 看起来 A 之后什么都没有
```

这是**所有相关服务必须采样配置一致**的根本原因。看到链路"半截"，先查 Sampler 配置。

### 2.6 Span Processor 的 BatchSpanProcessor

Span 生成后不能立即上报（一条一次 RPC 太贵），需要批量：

```java
// io.opentelemetry.sdk.trace.export.BatchSpanProcessor
public final class BatchSpanProcessor implements SpanProcessor {
    private final Queue<ReadableSpan> queue;        // 默认 2048
    private final ScheduledExecutorService worker;
    private final long scheduleDelayNanos;          // 默认 5s
    private final int maxExportBatchSize;           // 默认 512
    private final long exporterTimeoutNanos;        // 默认 30s

    @Override
    public void onEnd(ReadableSpan span) {
        if (!span.getSpanContext().isSampled()) return;   // ★ 未采样的不入队
        if (!queue.offer(span)) {
            droppedSpans.incrementAndGet();              // ★ 队满直接丢
        }
    }

    private void exportCurrentBatch() {
        List<SpanData> batch = drainBatch(maxExportBatchSize);
        try {
            CompletableResultCode result = spanExporter.export(batch);
            result.join(exporterTimeoutNanos, TimeUnit.NANOSECONDS);
        } catch (Throwable t) {
            // 失败的 batch 直接丢，不重试
        }
    }
}
```

**逐段点透**：

- **`isSampled()` 检查在入队前**：未采样的 Span 直接丢弃，连内存都不占。这是头采样比尾采样便宜的本质。
- **队满直接丢**：`droppedSpans` 计数器是生产必须监控的指标。Drop 高 = 批量发送速度跟不上生成速度 = 后端写入慢或 Exporter 配置太小。
- **失败不重试**：链路追踪的设计哲学是 **"丢一些 Span 比拖垮业务好"**。Exporter 失败的 batch 直接丢，绝不阻塞业务线程。

> Span Processor 是接入链路追踪后**业务感知最强的开销点**：每个 Span 入队涉及一次 CAS，每个 batch export 占网络带宽。配置不当（batch 太小、queue 太满）会显著拖慢服务。

---

## 3. 实测数据：采样、Agent 开销与存储压力

> 以下数据为 8 核 / 16GB / 千兆网 / SSD 环境的典型值，仅供参考。你的负载请自测。

### 3.1 不同采样率下的开销与诊断有效性

测试场景：Spring Boot 微服务，5 个服务串联，每请求约 25 个 Span。基线 QPS 5000，P99 22ms。

| 采样率 | CPU 增量 | P99 增量 | 出问题时诊断率 | 后端存储 |
| --- | --- | --- | --- | --- |
| 0%（不接） | 基线 | 基线 | 0%（盲飞） | 0 |
| 1% | +1.5% | +0.3ms | 30%（错误请求很可能漏采） | ~3GB/天 |
| 10% | +3% | +0.7ms | 70% | ~30GB/天 |
| 100% | +12% | +2.5ms | 100% | ~300GB/天 |
| 100% + 尾采样过滤 | +12%（采集端） | +2.5ms | 100%（错误全采） | ~10GB/天 |

**结论**：
- 1% 头采样几乎无感，但**错误诊断率惨不忍睹**
- 100% 全采 + 尾采样过滤是兼顾观察性和成本的最优解，但 Collector 复杂度上一个量级
- 中等团队的务实选择：**头采样 5% + 错误请求强制采样**（业务代码里发现 error 时调 `Span.current().setAttribute("force.sample", true)`）

### 3.2 SkyWalking Agent vs OpenTelemetry Agent 性能对比

同一个 Spring Boot 应用，分别接 SkyWalking Java Agent 和 OpenTelemetry Java Agent，跑 sysbench 风格压测。

| 指标 | 不接 Agent | OpenTelemetry Agent | SkyWalking Agent |
| --- | --- | --- | --- |
| 启动时间 | 3.2s | 4.8s（+1.6s） | 5.5s（+2.3s） |
| QPS（4 线程） | 12,500 | 12,100（-3%） | 11,800（-5.6%） |
| P99 延迟 | 8.2ms | 8.5ms（+0.3） | 8.9ms（+0.7） |
| 内存（RSS） | 480MB | 560MB（+80MB） | 620MB（+140MB） |
| CPU（idle 时） | 0.3% | 0.5% | 0.8% |

**为什么 SkyWalking 略重**：插件覆盖更广（默认开 30+ 个插件），ContextManager 内部有更多 ThreadLocal 操作，自带 metric & profiling 也消耗资源。

**优化建议**：
- 关闭不需要的插件：`agent.ignore_suffix`、`plugin.exclude_plugins`
- 调小 batch size、缩短 flush 间隔（避免 Span 堆积内存）
- 关闭 traceProfile（性能剖析）—— 默认就关了，不要乱开

### 3.3 头采样 vs 尾采样的存储成本

同样 5000 QPS、25 Span/请求 的场景：

| 方案 | Collector CPU | Collector 内存 | 后端写入量 | 错误覆盖率 |
| --- | --- | --- | --- | --- |
| 1% 头采样 | 1 核 | 200MB | 1.25 万 Span/秒 | 1%（差） |
| 10% 头采样 | 1.5 核 | 250MB | 12.5 万 Span/秒 | 10% |
| 100% 全采 | 4 核 | 500MB | 125 万 Span/秒 | 100% |
| **100% + 尾采样**（5% 通过 + 错误全过） | **8 核** | **6GB** | 11 万 Span/秒 | **100%** |

尾采样的代价：
- **Collector 内存**：必须缓存窗口期内所有 trace（典型 30s 窗口）
- **Collector CPU**：要按 traceId 哈希 + 缓冲 + 决策
- **集群拓扑**：多 Collector 时同 traceId 必须路由到同一个 Collector（用 LB 的 sticky 或前置 router）

> 尾采样是个"高级武器" —— 用对了省 90% 存储且不丢错误；用错了 Collector 自己变成瓶颈。**没有专人维护 Collector 拓扑，先别上**。

### 3.4 高基数 Span 数量对存储的压力

ES 里一条典型 Span 约 1KB（含 attributes / events / resource）。5000 QPS × 25 Span = 12.5 万 Span/秒：

- 全采集：~10.8 GB/小时，~260 GB/天
- 三副本（ES 默认）：~780 GB/天

ClickHouse 同等数据压缩后：~80 GB/天（含 3 副本）。**约 ES 的 1/10**。

**索引成本**：ES 的 inverted index 对每个 attribute 都建索引，单条 Span 写入 CPU 约是 ClickHouse 的 5-8 倍。生产 ES 集群的 CPU 70%+ 花在 indexing 上很常见。

### 3.5 链路完整率随服务数的衰减

链路里只要有**一个**服务的 Sampler 没正确配置 / Agent 挂了 / context 透传断了，整条 trace 就不全。

假设单服务"链路正确率" = 99%，则全链路完整率：

| 链路深度 | 全链路完整率 |
| --- | --- |
| 3 服务 | 97.0% |
| 5 服务 | 95.1% |
| 10 服务 | 90.4% |
| 20 服务 | 81.8% |
| 50 服务 | 60.5% |

**结论**：服务数翻倍，"看到完整链路"的概率 **指数级下降**。中大型微服务架构里"链路完整率 < 80%"是常态。

> 这是 Service Mesh 推动 sidecar 注入追踪的核心动机：把 context 透传从"业务代码自己保证"下沉到 sidecar，单点正确性大幅提高。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 跨线程池追踪丢失 —— "异步任务 Span 全没了"

#### 现象

接了 SkyWalking 后，业务里调用 `executor.submit(task)` 的 task 内部：
- 日志里 traceId 是空（用 `%X{tid}` 打不出来）
- SkyWalking 看不到 task 内部的下游调用 Span
- 主链路只到 `executor.submit` 这一行，之后断了

#### 根因

SkyWalking 的 ContextManager 把 TraceSegment 挂在 ThreadLocal 上。`executor.submit(task)` 把 task 提交到 worker 线程，worker 的 ThreadLocal 里没有这个 TraceSegment。

更糟的是：worker 是**池里复用**的。上一个任务残留的 TraceSegment 可能被下一个任务"看到" → **traceId 串了**。

虽然 SkyWalking 号称"字节码无侵入"，但**默认不增强用户业务里的 Runnable**（不知道哪个是业务的）。需要业务代码主动配合。

#### 修复

**方案 A**：用 SkyWalking 的 `apm-toolkit-trace` 包装。

```xml
<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-trace</artifactId>
    <version>9.2.0</version>
</dependency>
```

```java
import org.apache.skywalking.apm.toolkit.trace.RunnableWrapper;
import org.apache.skywalking.apm.toolkit.trace.CallableWrapper;

executor.submit(RunnableWrapper.of(() -> {
    // 任务逻辑：traceId、TraceSegment 都自动透传
}));

executor.submit(CallableWrapper.of(() -> compute()));
```

或注解：

```java
@TraceCrossThread
public class BizTask implements Runnable { ... }
```

**方案 B**（OpenTelemetry）：用 `Context.taskWrapping`

```java
ExecutorService raw = Executors.newFixedThreadPool(4);
ExecutorService wrapped = Context.taskWrapping(raw);

wrapped.submit(() -> {
    // Span.current() 能拿到提交线程的 Span
});
```

**方案 C**（最优雅）：直接在线程池层装饰

```java
public class TracingExecutor implements ExecutorService {
    private final ExecutorService delegate;

    @Override
    public void execute(Runnable command) {
        Context ctx = Context.current();
        delegate.execute(() -> {
            try (Scope ignored = ctx.makeCurrent()) {
                command.run();
            }
        });
    }
    // 其他方法略
}
```

**方案 D**（治本）：用阿里 [TransmittableThreadLocal](https://github.com/alibaba/transmittable-thread-local)（TTL）配合 SkyWalking 的 TTL 适配器，**业务代码完全无感**。

```java
ExecutorService ttlExecutor = TtlExecutors.getTtlExecutorService(rawExecutor);
```

#### 验证

```java
@Test
void traceShouldPropagateAcrossPool() {
    Span parent = tracer.spanBuilder("parent").startSpan();
    try (Scope s = parent.makeCurrent()) {
        wrappedExecutor.submit(() -> {
            String childTraceId = Span.current().getSpanContext().getTraceId();
            assertEquals(parent.getSpanContext().getTraceId(), childTraceId);
        }).get();
    } finally {
        parent.end();
    }
}
```

> **口诀**：链路追踪 + 线程池 = 必须装饰。SkyWalking 默认不装饰用户 Runnable；OTel 提供 `taskWrapping`；终极方案是 TTL。这与 [线程池深文](/article/java/threadpool) 的案例 ⑤ 互为印证。

---

### 案例 ② 跨 MQ 追踪丢失 —— "Header 没透传"

#### 现象

订单服务发 Kafka 消息，库存服务消费。两个服务都接了 OTel：
- 订单服务能看到 `kafka send` Span
- 库存服务能看到 `kafka process` Span
- 但**两个 Span 不在同一个 trace 里** —— 各自有独立的 traceId

#### 根因

排查后发现：库存服务的 Kafka 消费代码用了一个**自定义封装**：

```java
// 业务自己封装的消息中间件 SDK
public abstract class AbstractKafkaConsumer<T> {
    public final void consume(ConsumerRecord<String, String> raw) {
        T msg = JsonUtil.parse(raw.value(), payloadType);    // ★ 只取 value，丢了 headers
        doConsume(msg);
    }
    protected abstract void doConsume(T msg);
}
```

业务把 Kafka 原始 `ConsumerRecord` 解析成业务对象后就不传了。OTel Agent 增强的是 `KafkaConsumer.poll()`，但**业务真正"开始处理"的是 `doConsume(msg)` 这一层**，到那里 headers 已经丢了，traceparent 没人提取。

#### 修复

**方案 A**：业务封装层自己做 propagation。

```java
public abstract class AbstractKafkaConsumer<T> {
    public final void consume(ConsumerRecord<String, String> raw) {
        // ★ 在业务调用前 extract context
        Context parent = GlobalOpenTelemetry.getPropagators()
            .getTextMapPropagator()
            .extract(Context.current(), raw.headers(), KAFKA_HEADER_GETTER);

        Span span = tracer.spanBuilder(topic + " process")
            .setSpanKind(SpanKind.CONSUMER)
            .setParent(parent)
            .startSpan();

        try (Scope s = span.makeCurrent()) {
            T msg = JsonUtil.parse(raw.value(), payloadType);
            doConsume(msg);
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR);
            throw e;
        } finally {
            span.end();
        }
    }
}

private static final TextMapGetter<Headers> KAFKA_HEADER_GETTER =
    new TextMapGetter<>() {
        @Override
        public Iterable<String> keys(Headers headers) {
            return StreamSupport.stream(headers.spliterator(), false)
                .map(Header::key).collect(Collectors.toList());
        }
        @Override
        public String get(Headers headers, String key) {
            Header h = headers.lastHeader(key);
            return h == null ? null : new String(h.value(), StandardCharsets.UTF_8);
        }
    };
```

**方案 B**：让 OTel Agent 增强位置覆盖你的封装层。OTel Java Agent 默认增强 `org.apache.kafka.clients.consumer.KafkaConsumer`，能在 poll 时打 Span。但如果业务用 spring-kafka 的 `@KafkaListener`，它的 instrumentation 是 `spring-kafka` 插件，要确认开启。

**方案 C**：业务封装别"吃"原始 record，把 record 透传给业务方法。

#### 验证

```bash
# 1. 在订单发送侧捕获 traceId
grep "kafka send" order-service.log | head -1
# trace_id=4bf92f3577b34da6a3ce929d0e0e4736

# 2. 在库存消费侧用同一 traceId 查
grep "4bf92f3577b34da6a3ce929d0e0e4736" inventory-service.log
# 应该能看到 kafka process Span，且 parent 正确
```

> **口诀**：MQ 链路追踪 = Producer 注 + Consumer 抽。业务封装层是常见黑洞 —— 必须确认 headers 一路透传到真正调用 `extract` 的位置。

---

### 案例 ③ 头采样导致核心错误漏采 —— "看不见的 5xx"

#### 现象

线上 5xx 告警，去 SkyWalking 查具体出错的请求链路 —— **居然找不到**。监控指标显示这个时间段确实有几十个 5xx，但 trace 列表里一条都没有。

#### 根因

采样配置：

```yaml
# OpenTelemetry SDK 配置
otel.traces.sampler: parentbased_traceidratio
otel.traces.sampler.arg: 0.01     # 1% 头采样
```

业务真实 QPS = 5000，错误率 = 0.5%（25 个/秒）。1% 头采样意味着错误请求被采样的概率 = 1% × 0.5% × 5000 = **每秒 0.25 个**。换算下来，4 秒才有一个错误请求被采到。如果你的告警查询窗口是 1 分钟，能看到 ~15 个错误请求 trace —— **大概率漏掉关键的那个**。

#### 修复

**方案 A**：基于业务结果的"动态尾采样"。在业务发现错误时主动标记采样：

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(Exception.class)
    public ResponseEntity<?> handle(Exception e) {
        Span span = Span.current();
        if (span != null) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR);
            // ★ 标记本 trace 必须采样
            span.setAttribute("sampling.priority", 1);
        }
        log.error("request failed", e);
        return ResponseEntity.internalServerError().build();
    }
}
```

这需要配合"基于属性的尾采样" Sampler 或 OTel Collector 的 tail sampling processor。

**方案 B**：在 OTel Collector 上开 tail sampling processor。

```yaml
# otel-collector-config.yaml
processors:
  tail_sampling:
    decision_wait: 10s              # 等 10s 让所有 Span 到齐
    num_traces: 100000              # 缓冲 trace 数
    expected_new_traces_per_sec: 10000
    policies:
      - name: errors-policy
        type: status_code
        status_code:
          status_codes: [ERROR]
      - name: slow-policy
        type: latency
        latency:
          threshold_ms: 1000        # > 1s 全采
      - name: probabilistic-policy
        type: probabilistic
        probabilistic:
          sampling_percentage: 5    # 其他 5%

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [tail_sampling]
      exporters: [otlphttp]
```

**方案 C**：最简单 —— 错误率高的时候**临时调高头采样率**。某些 SDK 支持运行时改采样率，配合配置中心动态推。

#### 验证

```bash
# 上线后跑一段时间，对照
错误请求数（业务监控）          —— 假设 1000
错误 trace 数（追踪后端）        —— 应该 ≈ 1000（尾采样后）

# 查看 collector tail_sampling metric
curl http://collector:8888/metrics | grep tail_sampling
# otelcol_processor_tail_sampling_sampling_decision_total
# 看 sampled / not_sampled 比例
```

> **口诀**：低头采样率下"采集到错误"是个**概率游戏**，错误率越低越漏采。**核心服务必须用尾采样或动态采样**，不要用纯固定比例。

---

### 案例 ④ Span 数据爆炸 —— "每秒 10 万 Span 打挂 ES"

#### 现象

接入链路追踪两周后，运维告警 ES 集群磁盘 90%、写入延迟 5s+。一查发现 trace index 一天写入 200GB。集群只有 5 节点 × 2TB SSD，撑不了一周。

#### 根因

复合原因：
1. **采样率配的是 100%**（开发图省事）
2. 一些"无价值"的 Span 没过滤：健康检查 `/actuator/health` 每秒被 K8s 探一次，每次就一堆 Span
3. 一个内部服务暴力打 attribute：每个 Span 把整个请求 body 塞进 attribute（10KB+）
4. ORM 框架（MyBatis）默认每条 SQL 一个 Span，一个业务请求 30+ SQL

10000 QPS × 30 Span/请求 × 5KB Span size = **1.5 GB/秒**。一天 130TB（未压缩）。

#### 修复

**层 1：采样率合理化**

```yaml
otel.traces.sampler: parentbased_traceidratio
otel.traces.sampler.arg: 0.1       # 10%
```

**层 2：在 Collector 过滤无价值 Span**

```yaml
processors:
  filter:
    traces:
      span:
        # 丢健康检查
        - 'attributes["http.target"] == "/actuator/health"'
        - 'attributes["http.target"] == "/metrics"'
        # 丢内部探活
        - 'attributes["http.user_agent"] matches "kube-probe.*"'

  # 限制 attribute 大小
  attributes:
    actions:
      - key: http.request.body
        action: delete              # 直接删
      - key: db.statement
        action: hash                # 哈希后保留（既能聚合又不占空间）
```

**层 3：SDK 侧关闭无意义 instrumentation**

```yaml
otel.instrumentation.jdbc.statement-sanitizer.enabled: true
otel.instrumentation.jdbc.enabled: false      # 不要每条 SQL 一个 Span
otel.instrumentation.spring-data.enabled: true # 改用 Repository 粒度
```

**层 4：换 ClickHouse / Tempo 这类压缩比高的存储**

ClickHouse 写入 + 压缩后约是 ES 的 1/10 空间。Grafana Tempo 直接对象存储，成本极低（但查询能力弱）。

**层 5：冷热分离**

- 1 天内：高速 SSD（用于实时排障）
- 7 天内：标准 SSD
- 30 天内：对象存储（cold tier）
- 30 天外：删除 / 归档到便宜对象存储

#### 验证

```bash
# 调整后 24h 观察
curl 'http://es:9200/_cat/indices/jaeger-span-*?v&s=index'
# 看每天 size 应在团队预算内

# Collector 的 droppedSpans 应该接近 0
curl http://collector:8888/metrics | grep dropped
```

> **口诀**：Span 量是个**乘数效应**（QPS × Span/请求 × Span size）。任何一个维度爆了都会打挂存储。**接入前先估算量级，留 5x 余量**。

---

### 案例 ⑤ TraceId 串了 —— "MDC 复用导致两个请求 ID 相同"

#### 现象

日志里看到两条不同请求的日志带着**相同的 traceId**，但实际上它们是不同用户、不同时间发的请求。

```
[traceId=abc123] user=alice action=login
[traceId=abc123] user=bob action=checkout
```

排查发现两条日志确实是不同的请求，但都被链路追踪绑到了同一个 trace 上。

#### 根因

业务代码里有一段"自定义 MDC"：

```java
@Component
public class TraceIdFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain) {
        String traceId = ((HttpServletRequest) req).getHeader("X-Trace-Id");
        if (traceId == null) traceId = generateTraceId();
        MDC.put("traceId", traceId);
        try {
            chain.doFilter(req, resp);
        }
        // ★ 缺 finally 里 MDC.remove()
    }
}
```

Servlet 容器（Tomcat）的工作线程是池化复用的。线程跑完一个请求，MDC 残留没清。下一个请求进来，业务代码先调下游获取真实 traceId 再 put —— 但**有些早期日志（filter 链上层）打的是上一个请求残留的 traceId**。

更糟的是 SkyWalking 的 `mdc-adapter` 插件会自动把 SkyWalking 的 traceId 填到 MDC，但业务代码又手动 put 了一个，互相覆盖。

#### 修复

**A**：所有 MDC put 必须 try-finally remove

```java
public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain) {
    MDC.put("traceId", traceId);
    try {
        chain.doFilter(req, resp);
    } finally {
        MDC.clear();              // ★ 一定要清
    }
}
```

或者更稳：

```java
Map<String, String> backup = MDC.getCopyOfContextMap();
try {
    MDC.put("traceId", traceId);
    chain.doFilter(req, resp);
} finally {
    if (backup == null) MDC.clear();
    else MDC.setContextMap(backup);
}
```

**B**：别自己生成 traceId，**用追踪框架的**

```java
// SLF4J + OTel
String traceId = Span.current().getSpanContext().getTraceId();

// SkyWalking
String traceId = TraceContext.traceId();
```

接 logback 的 MDC 适配器自动注入：

```xml
<!-- skywalking -->
<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-logback-1.x</artifactId>
</dependency>
```

```xml
<!-- logback-spring.xml -->
<encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
    <layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.TraceIdPatternLogbackLayout">
        <Pattern>%d %-5level [%tid] %logger - %msg%n</Pattern>
    </layout>
</encoder>
```

`%tid` 自动取 SkyWalking traceId，无需业务自己 put。

**C**：用 OTel logs auto-instrumentation —— 自动把 traceId / spanId 注入 LogRecord，业务零代码。

#### 验证

跑一组并发请求（不同 userId），grep 日志按 traceId 聚类，确认每个 traceId 只对应一个 userId。

```bash
awk '/traceId=/ {match($0, /traceId=([^ ]+)/, t); match($0, /user=([^ ]+)/, u); print t[1], u[1]}' app.log \
  | sort -u | awk '{print $1}' | sort | uniq -d
# 输出应为空（没有重复 traceId 对应不同 user）
```

> **口诀**：MDC 是 ThreadLocal，**线程池复用必清**。最稳的做法是不自己 put，让追踪框架的 logging 适配器代劳。

---

### 案例 ⑥ 异步任务丢链路 —— "CompletableFuture 没装饰"

#### 现象

接 OTel 后业务用 `CompletableFuture` 做异步并发：

```java
CompletableFuture<Order> orderF = CompletableFuture.supplyAsync(this::loadOrder);
CompletableFuture<User> userF = CompletableFuture.supplyAsync(this::loadUser);
return orderF.thenCombine(userF, OrderDto::new).join();
```

链路里能看到 `loadOrder` 和 `loadUser` 的下游调用，但它们的 parentSpanId 是空，**和主链路连不上**，看起来像两条独立的孤儿 trace。

#### 根因

`CompletableFuture.supplyAsync(supplier)`（不带 Executor 的重载）默认用 `ForkJoinPool.commonPool()`。这个池的线程池没被任何 OTel 增强 wrap，跨线程时 Context 丢失。

更隐蔽：即使你传了 Executor，`thenCombine` 的回调是在哪个线程跑？取决于哪个 future 后完成、是不是 Async 后缀的方法 —— **可能是任意线程**。Context 不一定能传过去。

#### 修复

**方案 A**：所有 Executor 都先 wrap

```java
ExecutorService bizPool = Executors.newFixedThreadPool(8);
ExecutorService tracedPool = Context.taskWrapping(bizPool);

CompletableFuture<Order> orderF = CompletableFuture.supplyAsync(this::loadOrder, tracedPool);
CompletableFuture<User> userF = CompletableFuture.supplyAsync(this::loadUser, tracedPool);
```

注意：`thenCombine` / `thenApply` 不带 Async 时在**完成 future 的线程**或**调用线程**上跑（取决于完成时机），这两条线程都已经被 wrap，Context 会跟随。

**方案 B**：用 OTel 的 `CompletableFutureWrapper`

```java
import io.opentelemetry.context.Context;

Context current = Context.current();
CompletableFuture<Order> orderF = CompletableFuture.supplyAsync(() -> {
    try (Scope s = current.makeCurrent()) {
        return loadOrder();
    }
}, bizPool);
```

**方案 C**：直接禁用 commonPool 的使用 —— 业务规范里写明 "`CompletableFuture` 必须传 Executor"，code review 强制把关。

OTel Java Agent 1.30+ 实际上已经增强了 `ForkJoinPool` 和 `CompletableFuture`，但**不是所有路径都覆盖**。建议自己 wrap 兜底。

#### 验证

```java
@Test
void completableFutureShouldPropagateContext() {
    Span parent = tracer.spanBuilder("parent").startSpan();
    String parentTraceId;
    try (Scope s = parent.makeCurrent()) {
        parentTraceId = Span.current().getSpanContext().getTraceId();
        String childTraceId = CompletableFuture.supplyAsync(() ->
            Span.current().getSpanContext().getTraceId(), tracedPool
        ).join();
        assertEquals(parentTraceId, childTraceId);
    } finally {
        parent.end();
    }
}
```

> **口诀**：`CompletableFuture.supplyAsync` 不传 Executor = 用 commonPool = 上下文传不过去。**强制传池 + 池要 wrap**，二者缺一不可。

---

### 案例 ⑦ Reactor 链路丢失 —— "Schedulers.boundedElastic 没绑 Context"

#### 现象

WebFlux + Reactor 的应用接 OTel，发现链路在 `.subscribeOn(Schedulers.boundedElastic())` 这种地方断掉。Mono / Flux 链上的下游 Span parent 错乱。

#### 根因

Reactor 不靠 ThreadLocal 传上下文，它有自己的 **Reactor Context**（`Context` 这个名字撞车了，注意区分）。Reactor 的执行模型是"发布者 / 订阅者"，subscribe 时穿过的线程不固定。

OTel 的 Context 默认挂在 ThreadLocal —— Reactor 切线程时 ThreadLocal 不会跟着走。

#### 修复

**方案 A**：开启 Reactor 的 ContextPropagation 库（Reactor 3.5+）

```java
// 引依赖
// io.micrometer:context-propagation
// io.projectreactor:reactor-core (3.5+)

// 应用启动
import io.micrometer.context.ContextRegistry;

Hooks.enableAutomaticContextPropagation();
```

启用后，Reactor 自动在 subscribe 时捕获当前线程的 ThreadLocal（包括 OTel Context），在调度切线程时还原。**业务代码完全无感**。

**方案 B**（手动）：把 OTel Context 显式塞进 Reactor Context

```java
public Mono<Order> findOrder(String id) {
    return Mono.deferContextual(ctx -> {
        Context otelCtx = ctx.get(Context.class);
        try (Scope s = otelCtx.makeCurrent()) {
            return Mono.just(loadOrder(id));
        }
    });
}

// 在入口处把 OTel Context 塞进 Reactor Context
.contextWrite(reactorCtx -> reactorCtx.put(Context.class, Context.current()))
```

繁琐，不推荐。能用方案 A 就用 A。

**方案 C**：用 `reactor-tools` + OTel 的 reactor instrumentation

OTel Java Agent 自带 `reactor` instrumentation，`Hooks.enableAutomaticContextPropagation()` 在 Agent 模式下默认会被自动调用。**手动接入 SDK 模式时容易漏**。

#### 验证

```java
@Test
void reactorShouldPropagateContext() {
    Hooks.enableAutomaticContextPropagation();

    Span parent = tracer.spanBuilder("parent").startSpan();
    try (Scope s = parent.makeCurrent()) {
        String childTraceId = Mono.fromCallable(() ->
            Span.current().getSpanContext().getTraceId()
        ).subscribeOn(Schedulers.boundedElastic())
         .block();
        assertEquals(parent.getSpanContext().getTraceId(), childTraceId);
    } finally {
        parent.end();
    }
}
```

> **口诀**：Reactor + 链路追踪 = 必开 `Hooks.enableAutomaticContextPropagation()`。OTel Agent 模式默认开，SDK 模式必须自己开。

---

### 案例 ⑧ Sampler 配置不一致 —— "上游头采样、下游全采，链路不全"

#### 现象

A、B、C 三个服务串联调用。看 trace 列表只有 1% 的 trace 是完整的，其他 99% 都是"只有 A 没有 B/C" 或 "只有 C 没有 A"。

#### 根因

排查后发现各服务的 Sampler 配置：

| 服务 | Sampler |
| --- | --- |
| A | `parentbased_traceidratio` 1% |
| B | `traceidratio` 100% （没用 parentbased） |
| C | `parentbased_traceidratio` 1% |

B 服务的配置错误：用的是 `traceidratio` 而不是 `parentbased_traceidratio`。这意味着**B 不看 A 传过来的 sampled flag，自己重新摇号**：

- A 采样了（1%）→ B 又摇号 100% 采样 → C parentbased 跟随 B 采样 → trace 完整：1% × 100% = 1%
- A 没采样（99%）→ B 摇号 100% 也采样 → 但 B 上游没 Span（A 的 Span 没上报）→ trace 只有 B/C，看起来"半截"

更糟的场景：B 自己摇 1%，C parentbased 跟随：

- A 采样 + B 不采样 → trace 只有 A
- A 采样 + B 采样 → trace 完整（概率 1% × 1% = 0.01%）

#### 修复

**所有服务的 Sampler 必须一致**，并且都用 `parentbased_xxx`：

```yaml
# 所有服务都这样配
otel.traces.sampler: parentbased_traceidratio
otel.traces.sampler.arg: 0.1     # 根 Span 10% 采样，下游全跟随
```

或者考虑用 **JaegerRemoteSampler / OTel sampling configuration service** —— 集中下发采样规则，避免散落配置漂移。

#### 验证

```bash
# 抓一段时间的 trace，统计完整率
# 完整 = trace 包含所有应被调用的服务的 Span
# 完整率应 ≈ 100%（采样的 trace 都完整）
```

> **口诀**：链路完整性 = **所有服务 Sampler 决策一致**。必用 `parentbased_*` 让下游跟随上游。集中配置 + 巡检漂移。

---

### 案例 ⑨ Service Map 不准 —— "Span 父子关系错乱导致拓扑虚连接"

#### 现象

SkyWalking 的 Service Topology 上看到一根从 `gateway` 指向 `report-service` 的边。但你 100% 确定 gateway 从来不调 report-service。

#### 根因

Service Map 的推导算法（简化）：

```
对每个 trace 里的每个 Span：
  if span.kind == CLIENT 且 parent 也存在：
    upstream = parent.service.name
    downstream = span.attributes["peer.service"] 或 "net.peer.name"
    在拓扑图上加一条边 upstream → downstream
```

问题：`parent.service.name` 怎么确定？通常是**通过 parentSpanId 找上一个 Span，看它属于哪个 service**。如果 parentSpanId 错了 / 串了，拓扑就错。

具体场景：
1. 业务里某段代码用了"假上下文"：手动 new 了一个 SpanContext，parentSpanId 写死成另一个不相关的值
2. 跨线程池没 wrap，子任务的 parent 指向了**线程池里上一个任务残留的 Span**
3. 测试代码的 Span 和生产 Span 共用了 traceId 空间（极少见但出过）

#### 修复

**A**：禁用所有"手写 SpanContext"。如果业务真的需要构造 Span，必须用 `tracer.spanBuilder()`，让 SDK 自己分配 ID。

**B**：跨线程池场景必须 wrap（呼应案例 ①）。

**C**：在 Collector 上加 trace 完整性校验，丢弃明显异常的 trace：

```yaml
processors:
  tail_sampling:
    policies:
      - name: drop-orphan-spans
        type: composite
        composite:
          - type: span_count
            span_count:
              min_spans: 2          # 单 Span 的 trace 多半是孤儿，丢掉
```

**D**：定期对照 Service Mesh / 配置中心里的实际依赖关系，发现"虚连接"立即排查。

#### 验证

```bash
# 拉 SkyWalking 的拓扑 API，对比预期依赖
curl 'http://oap:12800/graphql' -d '{"query": "{ getServiceTopology(...) { ... } }"}'

# diff 实际依赖文件
diff topology-actual.json topology-expected.json
```

> **口诀**：Service Map 是 Span 父子关系的副产品。父子关系一错（手写 / 串线程），拓扑就出虚边。**拓扑异常先查 trace 数据本身的健康度**。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 标准与协议层
- [ ] 跨服务 propagation 协议统一（推荐 W3C Trace Context，弃用 B3 / sw8 自有协议）
- [ ] 多语言项目里所有 SDK 都支持同一个 propagation 协议
- [ ] tracestate 中放置的 vendor 数据 < 512 字节
- [ ] HTTP / gRPC / MQ 三类通道的 propagation 都验证过

### 采样层
- [ ] 所有服务的 Sampler 配置一致（建议 `parentbased_traceidratio`）
- [ ] 采样率与业务流量匹配（核心服务 5-10%，边缘服务 1%）
- [ ] 错误请求 / 慢请求有"强制采样"机制（业务层标记 + Collector tail sampling）
- [ ] Sampler 配置可动态调整（配合配置中心）
- [ ] 健康检查 / 内部探活 / metrics 端点的 Span 在 Collector 层过滤掉

### 上下文传播层
- [ ] 所有线程池都用 `Context.taskWrapping` 或 SkyWalking toolkit 包装
- [ ] `CompletableFuture.supplyAsync` 永远传 Executor，且 Executor 是 wrap 过的
- [ ] Reactor / WebFlux 项目开 `Hooks.enableAutomaticContextPropagation()`
- [ ] MQ 生产/消费两侧都 inject / extract Header
- [ ] MDC put 后 try-finally 清理（或用追踪框架的日志适配器自动注入）

### 性能与开销层
- [ ] BatchSpanProcessor 的 queue size、flush interval 经过压测调优
- [ ] `droppedSpans` 指标接入监控，> 0 告警
- [ ] Agent 接入后的 P99、CPU、内存开销在可接受范围（典型 < 10%）
- [ ] 禁用不必要的 instrumentation（ORM 单 SQL 一 Span 通常应关）

### 数据治理层
- [ ] Span attribute 不放大对象（请求 body / 响应 body / SQL 完整字符串）
- [ ] 高基数字段（userId / orderId）只放在 attribute，不放在 metric label
- [ ] SQL 等 attribute 走 sanitizer / hash，避免 PII 泄漏
- [ ] 存储有 retention 策略 + 冷热分离

### 后端与可观测性层
- [ ] Service Map / Topology 与实际依赖对账（防虚边）
- [ ] 链路完整率监控（trace 跨 N 个 service 时完整率 > 80%）
- [ ] 后端存储有容量监控、写入延迟监控、查询延迟监控
- [ ] Collector 自身有可观测性（CPU / 内存 / queue size）
- [ ] 关键 trace 能在 5s 内查询出来

### 灰度与回滚层
- [ ] Agent 升级前先在小流量服务灰度
- [ ] Sampler 配置变更走配置中心，不改代码
- [ ] 链路追踪挂了不影响业务（Exporter 失败要 fallback，绝不抛异常出业务）
- [ ] 有"一键关闭追踪"的开关

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### eBPF 实现的零侵入追踪
- **Pixie**（New Relic 开源）：基于 eBPF 在内核层抓 syscall / TCP / HTTP，零业务代码改动
- **Beyla**（Grafana）：自动发现 HTTP / gRPC 服务，输出 OTLP
- 优势：不改一行代码，覆盖所有语言
- 局限：拿不到业务上下文（业务自定义的 attribute、跨 RPC 的 traceId 透传），所以**不能完全替代 SDK** —— 适合做"基础流量观察"，业务深度追踪还得 SDK

### OpenTelemetry Logs 与 Trace 的关联
- OTel Logs Specification：日志数据模型与 Span / traceId 自动关联
- 在日志里自动注入 traceId / spanId（Java logback / log4j2 已有 OTel 适配）
- 在 trace UI 里直接跳转到这条 trace 关联的日志（Loki / ES 都支持）

### Service Mesh 下的自动追踪
- **Istio + Envoy**：在 sidecar 层做 propagation，业务代码无需改动
- 但 **sidecar 只能保证"跨进程"传递**，进程内（线程池 / 异步任务）还是要 SDK
- 实际工程中 Istio + OTel SDK 双管齐下

### 长尾分析（Tail Latency Analysis）
- 不只看 P99，看 P99.9 / P99.99 的链路画像
- Critical Path Analysis：从慢 trace 里找耗时最长的 Span 链
- 论文：Google *The Tail at Scale*（Dean & Barroso, 2013）

### 因果追踪（Causal Profiling）
- 论文：*Coz: Finding Code that Counts with Causal Profiling*（OSDI '15）
- 不是看"哪段代码最慢"，而是看"加速哪段代码对总延迟最有效"
- 与 trace 数据结合，能更精准地指导优化

### 业务追踪与基础追踪的协调
- 链路追踪天生只关注"系统调用"，但业务往往关心"业务事件"（用户下单的完整状态机）
- 常见做法：Span 上加业务 attribute（orderId / step），后端做"业务视角"聚合
- 高级做法：结合 CDC（Change Data Capture）+ 链路，把数据库变更也纳入 trace

### 链路追踪与 SLO 闭环
- SLO（Service Level Objective）告警触发后自动拉相关 trace 做 Root Cause Analysis
- 工具：Grafana SLO + Tempo + Pyroscope（profiling）
- 终极愿景：告警 → 自动定位到具体代码行

---

## 7. 引用

- W3C [Trace Context Recommendation](https://www.w3.org/TR/trace-context/)（2020）
- OpenTelemetry [Specification](https://github.com/open-telemetry/opentelemetry-specification) / [Java SDK](https://github.com/open-telemetry/opentelemetry-java)
- Google Dapper 论文：*Dapper, a Large-Scale Distributed Systems Tracing Infrastructure*（2010）
- SkyWalking 官方文档：[Apache SkyWalking](https://skywalking.apache.org/docs/)
- Jaeger 官方文档：[Jaeger Tracing](https://www.jaegertracing.io/docs/)
- Grafana Tempo：[grafana.com/docs/tempo](https://grafana.com/docs/tempo/)
- Pixie eBPF tracing：[px.dev](https://px.dev/)
- *Distributed Tracing in Practice* —— Austin Parker et al., O'Reilly 2020
- 阿里云 ARMS 链路追踪最佳实践（公开文档）

---

> 配套：[可观测性主题](/article/observability/) | [Prometheus 指标设计](/article/observability/prometheus) | [线程池生产实战](/article/java/threadpool)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。下一篇候选：[日志采集与查询](/article/observability/logging) / [eBPF 可观测性](/article/observability/ebpf) —— 想看哪个告诉我。
