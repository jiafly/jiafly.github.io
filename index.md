<!-- 首页 -->

<h1 style="color:#E97018; margin-bottom:0.2rem;">桃花岛 · 内容地图</h1>
<p style="color:#888; margin-top:0;">写代码会真用上的那部分，按"问题 → 原理 → 源码 → 数据 → 案例"五层结构慢慢补。</p>

---

## 🔥 深度长文（推荐长期回看）

不是八股、不是综述，是**写代码遇到具体问题时回来查**的那种文章。每篇按 [问题动机 → 原理推导 → 源码切片 → 实测数据 → 生产案例 → 上线清单] 六层结构来写，详见 [写作模板](/STYLE)。

> **98 篇深文 ≈ 159,000 行**。每篇 1500-2000 行，覆盖资深 Java 架构师必备的全部技能栈。

### Java 并发与高性能
- [synchronized 内幕 —— 从对象头到锁膨胀](/article/java/synchronized) <span style="color:#888;font-size:0.9rem;">· 1535 行</span>
- [AQS 源码深读 —— 一个 int 撑起整个 J.U.C](/article/java/aqs) <span style="color:#888;font-size:0.9rem;">· 2336 行</span>
- [线程池生产实战 —— 从源码到事故复盘](/article/java/threadpool) <span style="color:#888;font-size:0.9rem;">· 1369 行</span>
- [CompletableFuture 异步编排](/article/java/completablefuture) <span style="color:#888;font-size:0.9rem;">· 1940 行</span>
- [虚拟线程深入 —— Loom / Continuation](/article/java/virtual-thread) <span style="color:#888;font-size:0.9rem;">· 1522 行</span>
- [HashMap 与 ConcurrentHashMap 深入](/article/java/hashmap) <span style="color:#888;font-size:0.9rem;">· 1727 行</span>
- [LMAX Disruptor 低延迟 —— 无锁 Ring Buffer 与机械同情](/article/java/disruptor) <span style="color:#888;font-size:0.9rem;">· 1718 行</span>
- [Java IO / NIO / AIO 深入 —— Stream 到 FFM 演进](/article/java/io-nio) <span style="color:#888;font-size:0.9rem;">· 1725 行</span>
- [Java 序列化深入 —— Jackson / Protobuf / FlatBuffers / Kryo](/article/java/serialization) <span style="color:#888;font-size:0.9rem;">· 1584 行</span>
- [Java 字节码增强 —— ASM / ByteBuddy / CGLib / Javassist](/article/java/bytecode) <span style="color:#888;font-size:0.9rem;">· 1502 行</span>
- [JPMS 模块系统与 FFM API —— Java 9+ 两次重大演进](/article/java/jpms-ffm) <span style="color:#888;font-size:0.9rem;">· 1671 行</span>

### Spring 全家桶
- [Spring Bean 生命周期与循环依赖深入](/article/java/spring-bean) <span style="color:#888;font-size:0.9rem;">· 1749 行</span>
- [Spring AOP 内幕 —— 动态代理到自调用陷阱](/article/java/spring-aop) <span style="color:#888;font-size:0.9rem;">· 1346 行</span>
- [Spring 事务传播机制深入](/article/java/spring-tx) <span style="color:#888;font-size:0.9rem;">· 1565 行</span>
- [Spring Boot 自动配置原理](/article/java/spring-boot) <span style="color:#888;font-size:0.9rem;">· 1686 行</span>
- [Spring Security 6 深入](/article/security/spring-security) <span style="color:#888;font-size:0.9rem;">· 1784 行</span>
- [Spring Cloud / Dubbo 微服务对比](/article/distributed/spring-cloud-dubbo) <span style="color:#888;font-size:0.9rem;">· 1547 行</span>

### JVM 内核
- [JVM GC 深入 —— G1/ZGC/Shenandoah](/article/jvm/gc-deep) <span style="color:#888;font-size:0.9rem;">· 1392 行</span>
- [JMM 深入 —— happens-before 与内存屏障](/article/jvm/jmm-deep) <span style="color:#888;font-size:0.9rem;">· 1546 行</span>
- [JIT 优化与字节码深入](/article/jvm/jit) <span style="color:#888;font-size:0.9rem;">· 1632 行</span>
- [GraalVM Native Image —— AOT 与 Spring Boot 3 原生镜像](/article/jvm/graalvm) <span style="color:#888;font-size:0.9rem;">· 1421 行</span>
- [Project Loom 内部 —— Continuation 与 Carrier Thread 实现](/article/jvm/loom) <span style="color:#888;font-size:0.9rem;">· 2026 行</span>
- [ClassLoader 深入 —— 双亲委派、OSGi、Spring Boot Loader](/article/jvm/classloader-deep) <span style="color:#888;font-size:0.9rem;">· 1392 行</span>
- [HotSpot C2 编译器内核 —— Sea of Nodes 与优化 Pass](/article/jvm/c2-internals) <span style="color:#888;font-size:0.9rem;">· 1479 行</span>

### 数据库 / 搜索 / 流 / 大数据
- [MySQL MVCC 与锁](/article/db/mysql-mvcc) <span style="color:#888;font-size:0.9rem;">· 912 行</span>
- [MySQL 索引深入 —— B+ 树物理结构与优化器](/article/db/mysql-index) <span style="color:#888;font-size:0.9rem;">· 1413 行</span>
- [MySQL 分库分表实战](/article/db/sharding) <span style="color:#888;font-size:0.9rem;">· 1582 行</span>
- [Redis 内核与一致性](/article/db/redis-internals) <span style="color:#888;font-size:0.9rem;">· 1847 行</span>
- [MongoDB 深入 —— 文档模型、副本集、分片](/article/db/mongodb) <span style="color:#888;font-size:0.9rem;">· 1707 行</span>
- [Elasticsearch 深入 —— 倒排索引、分片与生产事故](/article/db/elasticsearch) <span style="color:#888;font-size:0.9rem;">· 1547 行</span>
- [ClickHouse OLAP 深入 —— 列存、MergeTree、物化视图](/article/db/clickhouse) <span style="color:#888;font-size:0.9rem;">· 1465 行</span>
- [Cassandra 深入 —— LSM 树、一致性级别与运维事故](/article/db/cassandra) <span style="color:#888;font-size:0.9rem;">· 1574 行</span>
- [HBase 深入 —— LSM、Region 分裂、协处理器](/article/db/hbase) <span style="color:#888;font-size:0.9rem;">· 1709 行</span>
- [TiDB 与 NewSQL 深入 —— 分布式事务、Multi-Raft、HTAP](/article/db/tidb) <span style="color:#888;font-size:0.9rem;">· 1524 行</span>
- [Flink 流处理深入 —— Checkpoint、State、Watermark](/article/db/flink) <span style="color:#888;font-size:0.9rem;">· 1484 行</span>
- [Apache Spark 内核 —— RDD / Catalyst / Tungsten](/article/db/spark) <span style="color:#888;font-size:0.9rem;">· 1567 行</span>
- [数据湖 Iceberg / Delta Lake / Hudi 深入](/article/db/data-lake) <span style="color:#888;font-size:0.9rem;">· 1475 行</span>
- [DuckDB 嵌入式分析 —— 列存 + 向量化执行](/article/db/duckdb) <span style="color:#888;font-size:0.9rem;">· 1428 行</span>
- [时序数据库深入 —— InfluxDB / TimescaleDB / VictoriaMetrics](/article/db/tsdb) <span style="color:#888;font-size:0.9rem;">· 1371 行</span>
- [对象存储深入 —— S3 / MinIO / Ceph 与现代数据基础设施](/article/db/object-storage) <span style="color:#888;font-size:0.9rem;">· 1509 行</span>
- [Trino / Presto 联邦查询深入](/article/db/trino) <span style="color:#888;font-size:0.9rem;">· 1566 行</span>

### 分布式系统
- [共识算法 —— Paxos 到 Raft](/article/distributed/consensus) <span style="color:#888;font-size:0.9rem;">· 1271 行</span>
- [分布式锁 —— Redis / ZK / etcd 全景](/article/distributed/dist-lock) <span style="color:#888;font-size:0.9rem;">· 1598 行</span>
- [分布式事务 —— 2PC / TCC / Saga](/article/distributed/dist-tx) <span style="color:#888;font-size:0.9rem;">· 1579 行</span>
- [服务注册与发现 —— 5 大注册中心对比](/article/distributed/service-discovery) <span style="color:#888;font-size:0.9rem;">· 1396 行</span>
- [Spring Cloud / Dubbo 微服务对比](/article/distributed/spring-cloud-dubbo) <span style="color:#888;font-size:0.9rem;">· 1547 行</span>

### 中间件 / 网络
- [Kafka 存储引擎 —— 顺序写、PageCache、零拷贝](/article/mq/kafka-storage) <span style="color:#888;font-size:0.9rem;">· 1504 行</span>
- [RocketMQ 内核 —— 存储、事务、顺序、积压](/article/mq/rocketmq-deep) <span style="color:#888;font-size:0.9rem;">· 1525 行</span>
- [Apache Pulsar 深入 —— 计算存储分离、多租户、Geo-Replication](/article/mq/pulsar) <span style="color:#888;font-size:0.9rem;">· 1486 行</span>
- [Netty EventLoop 与 ByteBuf](/article/network/netty-eventloop) <span style="color:#888;font-size:0.9rem;">· 1817 行</span>
- [TCP 状态机深入 —— SYN_FLOOD 到 CLOSE_WAIT](/article/network/tcp-state) <span style="color:#888;font-size:0.9rem;">· 1420 行</span>
- [HTTP/1 → 2 → 3 演进 —— 多路复用与 QUIC](/article/network/http-evolution) <span style="color:#888;font-size:0.9rem;">· 1268 行</span>
- [gRPC 深入 —— Protobuf、流式调用、负载均衡](/article/network/grpc) <span style="color:#888;font-size:0.9rem;">· 1595 行</span>
- [WebSocket / SSE / 长连接 —— 推送 + IM + LLM 流式](/article/network/websocket) <span style="color:#888;font-size:0.9rem;">· 1773 行</span>

### 云原生 / K8s
- [Kubernetes 实战 —— 控制平面到生产排查](/article/k8s/k8s-essentials) <span style="color:#888;font-size:0.9rem;">· 1470 行</span>
- [Service Mesh 与 Istio 深入](/article/k8s/service-mesh) <span style="color:#888;font-size:0.9rem;">· 1428 行</span>
- [Operator 与 CRD 开发 —— Reconcile Loop 到生产级 Operator](/article/k8s/operator) <span style="color:#888;font-size:0.9rem;">· 1825 行</span>
- [eBPF 与 Cilium —— 内核态可编程与下一代数据面](/article/k8s/ebpf-cilium) <span style="color:#888;font-size:0.9rem;">· 1652 行</span>
- [Serverless 与 Knative / KEDA —— K8s 原生函数计算](/article/k8s/serverless) <span style="color:#888;font-size:0.9rem;">· 1816 行</span>

### 架构设计
- [DDD 战术与战略设计 —— Aggregate 到 Bounded Context](/article/architecture/ddd) <span style="color:#888;font-size:0.9rem;">· 1974 行</span>
- [API 设计与 OpenAPI —— REST/GraphQL/gRPC 选型](/article/architecture/api-design) <span style="color:#888;font-size:0.9rem;">· 2024 行</span>
- [Hexagonal/Clean/Onion 架构 —— 端口适配器与依赖反转](/article/architecture/hexagonal) <span style="color:#888;font-size:0.9rem;">· 1917 行</span>
- [事件驱动架构（EDA）与 Event Sourcing](/article/architecture/eda) <span style="color:#888;font-size:0.9rem;">· 1915 行</span>
- [微服务拆分演进 —— 单体到 Cell-based 架构](/article/architecture/microservices) <span style="color:#888;font-size:0.9rem;">· 1951 行</span>
- [多租户架构设计 —— 数据、计算、网络隔离与共享](/article/architecture/multi-tenant) <span style="color:#888;font-size:0.9rem;">· 2282 行</span>

### 系统设计
- [缓存设计 —— 击穿/雪崩/穿透/一致性](/article/system-design/cache-design) <span style="color:#888;font-size:0.9rem;">· 1467 行</span>
- [高可用架构演进 —— 单机到异地多活](/article/system-design/high-availability) <span style="color:#888;font-size:0.9rem;">· 1573 行</span>
- [限流、降级、熔断 —— Sentinel/R4j/自适应](/article/system-design/resilience) <span style="color:#888;font-size:0.9rem;">· 1282 行</span>
- [秒杀系统演进 —— 1K → 100K QPS](/article/system-design/seckill) <span style="color:#888;font-size:0.9rem;">· 1854 行</span>
- [配置中心深入 —— Apollo / Nacos / Spring Cloud Config](/article/system-design/config-center) <span style="color:#888;font-size:0.9rem;">· 1287 行</span>
- [Feature Flag 系统设计 —— 渐进式发布与试验平台](/article/system-design/feature-flag) <span style="color:#888;font-size:0.9rem;">· 1848 行</span>

### 工程实践（测试 / DevOps / 安全）
- [单元测试与 Mock —— JUnit 5 / Mockito / Testcontainers](/article/testing/unit-testing) <span style="color:#888;font-size:0.9rem;">· 1625 行</span>
- [Git 工作流深入 —— 对象模型到 Trunk-based](/article/devops/git-workflow) <span style="color:#888;font-size:0.9rem;">· 1724 行</span>
- [DevOps 与 CI/CD 流水线 —— Jenkins 到 GitOps](/article/devops/cicd) <span style="color:#888;font-size:0.9rem;">· 1828 行</span>
- [GitOps 实战 —— ArgoCD / Flux / 多集群部署](/article/devops/gitops) <span style="color:#888;font-size:0.9rem;">· 2064 行</span>
- [灰度发布与金丝雀部署 —— Argo Rollouts / Flagger](/article/devops/canary-deployment) <span style="color:#888;font-size:0.9rem;">· 1760 行</span>
- [Web 安全实战 —— OAuth2/JWT/HTTPS/OWASP](/article/security/security-deep) <span style="color:#888;font-size:0.9rem;">· 1863 行</span>
- [Spring Security 6 深入 —— 过滤器链、OAuth2 RS、方法级安全](/article/security/spring-security) <span style="color:#888;font-size:0.9rem;">· 1784 行</span>
- [WebAuthn / Passkey 实战 —— 无密码认证工程落地](/article/security/webauthn) <span style="color:#888;font-size:0.9rem;">· 1747 行</span>

### 可观测性 / 性能工程
- [Prometheus 指标设计与查询](/article/observability/prometheus) <span style="color:#888;font-size:0.9rem;">· 1793 行</span>
- [分布式链路追踪 —— OpenTelemetry / SkyWalking](/article/observability/tracing) <span style="color:#888;font-size:0.9rem;">· 1484 行</span>
- [SLO 错误预算 —— Google SRE 工程化落地](/article/observability/slo) <span style="color:#888;font-size:0.9rem;">· 1262 行</span>
- [日志架构深入 —— 结构化、采集、Loki/CH](/article/observability/logging) <span style="color:#888;font-size:0.9rem;">· 1677 行</span>
- [故障定位方法论 —— 5 分钟定位 P99 飙升](/article/observability/troubleshooting) <span style="color:#888;font-size:0.9rem;">· 1603 行</span>
- [OpenTelemetry Collector 深入 —— 统一遥测采集与转换](/article/observability/otel-collector) <span style="color:#888;font-size:0.9rem;">· 1871 行</span>
- [火焰图实战 —— perf / async-profiler / Pyroscope](/article/linux-perf/flame-graph) <span style="color:#888;font-size:0.9rem;">· 1215 行</span>
- [Linux 60 秒诊断 —— Brendan Gregg 标准动作](/article/linux-perf/60s-diagnosis) <span style="color:#888;font-size:0.9rem;">· 1506 行</span>
- [Linux 内存模型与 OOM](/article/linux-perf/memory-oom) <span style="color:#888;font-size:0.9rem;">· 1469 行</span>
- [Linux IO 模型 —— select/epoll/io_uring](/article/linux-perf/io-models) <span style="color:#888;font-size:0.9rem;">· 1413 行</span>
- [容器与 cgroup —— Java 在 K8s 里的"误读"](/article/linux-perf/container-cgroup) <span style="color:#888;font-size:0.9rem;">· 1484 行</span>

### AI 工程
- [RAG 工程化 —— 从 chunk 到评估](/article/ai/rag-engineering) <span style="color:#888;font-size:0.9rem;">· 1430 行</span>
- [Agent 设计模式深入 —— ReAct / Multi-Agent](/article/ai/agent-deep) <span style="color:#888;font-size:0.9rem;">· 1915 行</span>
- [MCP 与工具调用深入 —— 让 LLM 真正用上工具](/article/ai/mcp-deep) <span style="color:#888;font-size:0.9rem;">· 1663 行</span>
- [Prompt 工程深入 —— Few-shot 到 Chain-of-Thought](/article/ai/prompt-deep) <span style="color:#888;font-size:0.9rem;">· 1610 行</span>
- [LLM 推理引擎深入 —— vLLM / TGI / llama.cpp](/article/ai/llm-inference) <span style="color:#888;font-size:0.9rem;">· 1784 行</span>
- [Vector Database 深入 —— HNSW / IVF / Milvus / pgvector](/article/ai/vector-db) <span style="color:#888;font-size:0.9rem;">· 1489 行</span>
- [多模态 AI 工程 —— 视觉、语音、视频](/article/ai/multimodal) <span style="color:#888;font-size:0.9rem;">· 1784 行</span>
- [LangGraph 与 DSPy 深入 —— LLM 应用的状态机与编程范式](/article/ai/langgraph-dspy) <span style="color:#888;font-size:0.9rem;">· 1510 行</span>
- [LLM 微调实战 —— LoRA / QLoRA / DPO 与部署](/article/ai/finetuning) <span style="color:#888;font-size:0.9rem;">· 1755 行</span>
- [LLM 安全实战 —— Prompt Injection / Jailbreak 防御](/article/ai/llm-security) <span style="color:#888;font-size:0.9rem;">· 1889 行</span>

> 想看哪个主题的下一篇深度版，[告诉我](/article/about)。

---

## 🗺️ 广度地图

按"工程师真正要会"的维度分十大板块。每个分类的导览页都列了规划的深文清单：

<table style="width:100%; border-collapse: collapse;">
<tr>
  <td style="padding:0.5rem 0; vertical-align:top; width:50%;">

**核心基础**
- [Java 核心](/#java-核心)
- [JVM 内核](/#jvm-内核)
- [数据库 / 搜索](/#数据库--搜索)
- [架构设计](/article/architecture/)

  </td>
  <td style="padding:0.5rem 0; vertical-align:top; width:50%;">

**架构与运维**
- [分布式系统](/article/distributed/)
- [消息队列](/article/mq/)
- [网络与 RPC](/article/network/)
- [K8s 与云原生](/article/k8s/)
- [系统设计](/article/system-design/)
- [可观测性](/article/observability/)
- [Linux 与性能工程](/article/linux-perf/)
- [测试与质量](/article/testing/)
- [DevOps](/article/devops/)
- [安全工程](/article/security/)
- [AI 工程](/#ai-工程)

  </td>
</tr>
</table>

---

## 🔥 高频精选

适合 **面试准备**——按这条线刷一遍，覆盖大部分 Java 后端考点：

1. [JVM 运行时数据区](/article/jvm/jvm)
2. [垃圾收集机制](/article/jvm/gc) → 深入 [G1/ZGC/Shenandoah](/article/jvm/gc-deep)
3. [Java 内存模型 JMM](/article/jvm/jmm) → 深入 [JMM 深入](/article/jvm/jmm-deep)
4. [Java 集合框架](/article/java/collection) → 深入 [HashMap/CHM](/article/java/hashmap)
5. [并发编程速记](/article/java/concurrent) → 深入 [线程池](/article/java/threadpool) / [synchronized](/article/java/synchronized) / [AQS](/article/java/aqs)
6. [Spring 核心原理](/article/java/spring) → 深入 [AOP](/article/java/spring-aop) / [事务](/article/java/spring-tx) / [Boot](/article/java/spring-boot)
7. [MySQL 速记](/article/db/mysql) → 深入 [MVCC](/article/db/mysql-mvcc) / [索引](/article/db/mysql-index) / [分库分表](/article/db/sharding)
8. [Redis 速记](/article/db/redis) → 深入 [内核与一致性](/article/db/redis-internals)

---

## 关于本站的写作哲学

- 每个深文按 [桃花岛深度模板](/STYLE)：5 层结构 + 上线清单 + 延伸方向
- "速记/综述"专门给面试准备用，不展开源码
- "深文"才是主菜，目标是**让你写代码遇到问题时第一个想到回来查这里**
- 反对：堆术语、贴大段源码不点透、"性能更好"没数据、"可能"和"也许"满天飞

如果你想看某个主题的深度版本（或发现现有文章浅了），欢迎到 [关于站长](/article/about) 留言告诉我。
