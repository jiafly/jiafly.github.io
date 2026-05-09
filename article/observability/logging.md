# 日志架构深入 —— 结构化日志、采集、存储与查询的工业级实践

> 这是按 [桃花岛深度模板](/STYLE) 写的可观测性系列第二篇。如果你只想要面试口诀，看 [可观测性总览](/article/observability/) 那一节就够了。
> 这里要回答的，是**日志体系上 TB 级、ES 集群被打挂、PII 泄漏告警凌晨三点把你叫起来**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上可以跳过：

1. 你的服务一天产生 500GB 日志，开了 ELK，每次告警要查日志却要等 30s 出结果 —— ES 慢在哪一层？瓶颈是查询、是 mapping、还是 segment merge？
2. 同事说"我们换 Loki，省 80% 成本"，你怎么验证？Loki 凭什么便宜，代价是什么？
3. 一行 Java 异常栈被切成了 17 条独立日志，搜索时拼不回来 —— 在哪一层修？Filebeat 配置该怎么写？
4. `WARN` 日志里你 `log.warn("user login failed: {}", request)`，request 里有手机号 —— 监管检查时这算不算 PII 泄漏？怎么从架构上消除而不是靠人记得脱敏？
5. 你给 Loki 加了一个 label `request_id`，第二天 index 涨到 200GB，查询全超时 —— 为什么？高基数 label 的死亡半径有多大？
6. ClickHouse 存日志号称比 ES 省 10 倍，你怎么设计表结构？分区键、排序键、`Bloom Filter` 怎么选？
7. 链路追踪有 traceId，日志里也写了 traceId，但在 Kibana 里点 traceId 跳到链路系统跳错了 —— 故障在哪一层？
8. 你在 Filebeat 里配了 `output.kafka`，结果业务高峰期 Kafka 反压、Filebeat 内存涨到 4GB OOM —— 设计上哪里错了？

如果有半数答不上，往下看。

---

## 1. 底层模型：日志的 5 个生命周期

绝大多数文章把"日志架构"画成 `Filebeat → Kafka → Logstash → ES → Kibana` 一条线就结束。这层抽象解释不了上面 8 个问题。要看懂它们，必须把日志拆成 **5 个独立的工程问题**。

### 1.1 五段流水线

```
   生成              采集              传输              存储              查询
 ┌──────┐        ┌────────┐        ┌──────┐        ┌────────┐        ┌────────┐
 │ App  │ stdout │ Agent  │  push  │ MQ / │  pull  │ ES /   │  query │ Kibana │
 │ Logger│──────►│Filebeat│───────►│Kafka │───────►│ Loki / │◄───────│ Grafana│
 │  SDK │  file  │Fluent  │        │ HTTP │        │ CH / S3│        │ CLI    │
 └──────┘        └────────┘        └──────┘        └────────┘        └────────┘
   ↑                ↑                  ↑                ↑                ↑
 结构化           buffer/            队列          压缩/索引/         查询语言
 vs 文本          多行合并          反压策略        分片/冷热         聚合优化
```

每一段都有自己的失败模式，每一段都能成为瓶颈。一个工业级日志体系要在 **5 段都做对**。

### 1.2 结构化日志 vs 非结构化

**非结构化（文本）**：

```
2025-05-09 14:32:11 INFO  [biz-pool-3] OrderService - user 12345 placed order ORD-998 with amount 89.50
```

**结构化（JSON）**：

```json
{"@timestamp":"2025-05-09T14:32:11.123Z","level":"INFO","thread":"biz-pool-3","logger":"OrderService","msg":"order placed","userId":12345,"orderId":"ORD-998","amount":89.50,"traceId":"abc123"}
```

差别不在"好不好看"，在**机器可读性**：
- 文本日志要查 `userId=12345` 必须正则解析整行 —— 在 ES 里走 `match_phrase`，每次都全文倒排
- JSON 日志在 ES 里 `userId` 是个独立字段，走 `term` 查询，**单字段倒排索引秒回**

代价：
- JSON 比文本大 2~3 倍（key 名重复 + 引号 + 转义）
- 写日志多一次序列化开销（~1µs/条）
- 字段多了 ES mapping 容易爆（见案例 ⑤ 在 Loki 上的对应版本）

### 1.3 三种采集模式

| 模式 | 部署 | 适用场景 | 痛点 |
| --- | --- | --- | --- |
| **DaemonSet Agent** | 每 Node 一个 Filebeat/Fluent-bit | K8s 容器日志（stdout） | 节点 IO 共用；多业务混采 |
| **Sidecar Agent** | 每 Pod 一个 Agent 容器 | 业务有特殊采集需求；多租户隔离 | Pod 资源翻倍；运维复杂 |
| **SDK 直送** | 业务进程里 Logback Appender 直发 | 低延迟告警链路；金融场景 | 业务进程背锅；网络抖动直接影响业务 |

DaemonSet 是默认选择。Sidecar 用于"这个业务必须独立 buffer / 独立鉴权"。SDK 直送用于"日志即告警"的关键链路。

### 1.4 传输模式：push vs pull

- **Push（Filebeat → Kafka / ES）**：Agent 主动推。简单，但下游挂了 Agent 自己 buffer，buffer 满 → OOM 或丢日志
- **Pull（Loki / Promtail 模式）**：Agent 写本地文件 + 暴露 endpoint，下游主动拉。背压天然 —— 拉不动就慢，不会把 Agent 撑爆

绝大多数生产是 push 模式 + 中间加 Kafka 做削峰：

```
App ──► local file ──► Filebeat ──► Kafka ──► Logstash ──► ES
                                       ↑
                                    削峰 + 重放
                                    保证下游挂了 Agent 不挂
```

### 1.5 存储引擎对比

| 引擎 | 索引模式 | 查询能力 | 存储成本 | 写入吞吐 | 适用 |
| --- | --- | --- | --- | --- | --- |
| **Elasticsearch** | 全字段倒排 | 全文 + 聚合 + 复杂 query | 高（1~2x 原文） | 中 | 复杂查询、传统首选 |
| **Loki** | 仅索引 label | 按 label 过滤后 grep | **极低（0.1x 原文）** | 高 | 已有 Prometheus 标签体系 |
| **ClickHouse** | 列式 + 稀疏索引 | SQL + 高速聚合 | 低（0.1~0.3x） | **极高** | 大规模、需 SQL 分析 |
| **S3 + 索引** | 对象存储 + 外部索引 | 慢但便宜 | **最低** | 高 | 归档、审计、低频查询 |

**关键认知**：选型不是"哪个更好"，是"你需要怎么查"。
- 如果 90% 查询是"按 service + level 过滤后 grep 关键字" → Loki
- 如果你要做"按 region / 按 endpoint 聚合 P99 错误率" → ES 或 ClickHouse
- 如果只是合规审计 → S3 归档

### 1.6 查询语言对比

```
# Lucene (ES / Kibana)
level:ERROR AND service:order AND msg:"timeout" AND @timestamp:[now-1h TO now]

# LogQL (Loki)
{service="order", level="ERROR"} |= "timeout" | json | duration > 1s

# SQL (ClickHouse)
SELECT count(), service FROM logs
WHERE level = 'ERROR' AND msg LIKE '%timeout%'
  AND ts BETWEEN now() - INTERVAL 1 HOUR AND now()
GROUP BY service ORDER BY 1 DESC
```

LogQL 故意设计成 Prometheus PromQL 的"姊妹语言"，做 metric/log 关联查询体验最一致。SQL 最强但写起来冗长。Lucene 适合自由文本搜索。

> **口诀**：日志架构是 **5 段独立工程**。每段都要单独评估失败模式。把日志当成"一条流水线"看就死。

---

## 2. 工具与配置切片

> 每段切片 < 30 行 + 逐段点透。版本：Logback 1.4 / Filebeat 8.x / Fluent-bit 2.x / Vector 0.34 / Loki 2.9 / ClickHouse 23.x。

### 2.1 SLF4J + Logback 的 JSON Encoder（含 MDC）

依赖 `logstash-logback-encoder` 是事实标准：

```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```

`logback-spring.xml`：

```xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>spanId</includeMdcKeyName>
            <includeMdcKeyName>userId</includeMdcKeyName>
            <customFields>{"service":"order","env":"prod"}</customFields>
            <fieldNames>
                <timestamp>@timestamp</timestamp>
                <message>msg</message>
                <thread>thread</thread>
                <logger>logger</logger>
            </fieldNames>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="STDOUT" />
    </root>
</configuration>
```

**逐段点透**：

- **`includeMdcKeyName` 显式列举**：不要 `<includeMdc>true</includeMdc>` 全收 —— MDC 里可能藏着别的库往里塞的乱七八糟字段，污染 ES mapping
- **`customFields` 写死 service/env**：每条日志都带 service 标签是 100% 必须的，靠 K8s 那边再注入容易遗漏
- **fieldNames 改成 `@timestamp`**：ES 默认时间字段是 `@timestamp`，不改就是给自己挖坑
- **不用 `KeyValuePair` / `Marker` 等高级 API**：保持 JSON 扁平，方便所有工具消费

业务代码用 SLF4J 的 `argument` 占位符让结构化字段自动出现在 JSON 里：

```java
log.info("order placed",
    kv("orderId", orderId),
    kv("userId", userId),
    kv("amount", amount));
```

`kv` 来自 `net.logstash.logback.argument.StructuredArguments.kv`。这样 JSON 输出会自带这些字段，**不需要拼字符串**。

### 2.2 Filebeat 配置 + 多行日志合并

K8s DaemonSet 模式典型配置 `filebeat.yml`：

```yaml
filebeat.autodiscover:
  providers:
    - type: kubernetes
      node: ${NODE_NAME}
      hints.enabled: true
      templates:
        - condition:
            equals:
              kubernetes.namespace: prod
          config:
            - type: container
              paths:
                - /var/log/containers/*-${data.kubernetes.container.id}.log
              multiline.type: pattern
              multiline.pattern: '^\d{4}-\d{2}-\d{2}'  # 行首是日期 → 新日志
              multiline.negate: true
              multiline.match: after
              multiline.max_lines: 500
              multiline.timeout: 5s

processors:
  - add_kubernetes_metadata: ~
  - drop_fields:
      fields: ["agent", "ecs", "host", "input", "log.offset"]

queue.mem:
  events: 4096
  flush.min_events: 512
  flush.timeout: 1s

output.kafka:
  hosts: ["kafka-1:9092", "kafka-2:9092"]
  topic: 'logs-%{[kubernetes.namespace]}'
  partition.round_robin:
    reachable_only: true
  required_acks: 1
  compression: snappy
  max_message_bytes: 1000000
```

**逐段点透**：

- **`multiline.pattern` + `negate: true` + `match: after`**：含义是"不匹配该 pattern 的行都附加到上一条日志"。这是把 Java 异常栈合并成一条的核心。**`multiline.max_lines: 500` 必须设**，否则一个无限循环堆栈能把单条日志撑到 GB 级
- **`multiline.timeout: 5s`**：如果 5s 内没有新行进来，强制 flush —— 否则最后一条日志可能永远卡在 buffer 里
- **`drop_fields`**：Filebeat 默认会塞一堆 `agent.*`、`ecs.*`、`host.*` 元数据。每条日志多 1KB，TB 级日志能省 30% 存储
- **`queue.mem.events: 4096`**：内存 queue 大小。配大了下游挂的时候 OOM；配小了高峰期丢日志。**4096 是 8.x 的合理默认**
- **`required_acks: 1`**：Kafka 的 leader 写入即 ack。`0` 太激进（leader 挂了丢），`-1` 等所有 replica 太慢

### 2.3 Fluent-bit 的 stream processor

Fluent-bit 比 Filebeat 更轻量（C 写的，内存 ~10MB vs Filebeat ~80MB），它的杀手锏是 **stream processor** —— 可以在 Agent 端做 SQL 风格的日志过滤聚合：

```ini
[INPUT]
    Name              tail
    Path              /var/log/containers/*.log
    Parser            docker
    Tag               kube.*
    Mem_Buf_Limit     50MB

[FILTER]
    Name              kubernetes
    Match             kube.*
    Kube_URL          https://kubernetes.default.svc:443
    Merge_Log         On
    K8S-Logging.Parser On

[FILTER]
    Name              modify
    Match             kube.*
    Remove_wildcard   kubernetes_

[OUTPUT]
    Name              loki
    Match             kube.*
    Host              loki.observability.svc
    Port              3100
    Labels            job=fluentbit, namespace=$kubernetes['namespace_name']
```

外加 stream processor 文件 `stream_processor.conf`：

```sql
[STREAM_TASK]
    Name   error_count
    Exec   CREATE STREAM errors WITH (tag='errors') AS
           SELECT COUNT(*) AS cnt FROM TAG:'kube.*' WHERE level = 'ERROR' WINDOW TUMBLING (10 SECOND);
```

**逐段点透**：

- **Stream processor 在 Agent 端就把"每 10 秒错误数"算出来直接送 Prometheus**：避免日志全量到中心后再聚合，省 90% 网络
- **`Mem_Buf_Limit: 50MB`**：到达上限会 pause input。Fluent-bit 默认行为比 Filebeat 的"暴涨"安全
- **`Merge_Log: On`**：自动把 stdout 里的 JSON 字段合并到根，业务的 JSON 日志直接成为顶层字段
- **`Remove_wildcard: kubernetes_`**：Loki 场景下，K8s 元数据全删（避免高基数 label，见案例 ⑤）

### 2.4 Vector 的 transforms 与多 sink

Vector（Datadog 出品的 Rust 写的 Agent）的优势是**统一处理 logs/metrics/traces** + **强大的 VRL（Vector Remap Language）**：

```toml
[sources.app_logs]
  type = "kubernetes_logs"

[transforms.parse_json]
  type = "remap"
  inputs = ["app_logs"]
  source = '''
    . = parse_json!(.message)
    .timestamp = parse_timestamp!(.["@timestamp"], "%Y-%m-%dT%H:%M:%S%.3fZ")
    if .level == "ERROR" {
      .severity_num = 5
    }
    # PII 脱敏：手机号
    .msg = replace(.msg, r'1[3-9]\d{9}', "***-PHONE-***")
  '''

[transforms.sample_info]
  type = "sample"
  inputs = ["parse_json"]
  rate = 10                     # INFO 日志 10:1 采样
  exclude.type = "vrl"
  exclude.source = '.level != "INFO"'

[sinks.loki]
  type = "loki"
  inputs = ["sample_info"]
  endpoint = "http://loki:3100"
  labels.service = "{{ service }}"
  labels.level = "{{ level }}"
  encoding.codec = "json"

[sinks.s3_archive]
  type = "aws_s3"
  inputs = ["parse_json"]                # ★ 全量归档，不采样
  bucket = "my-log-archive"
  compression = "zstd"
  batch.max_bytes = 100000000
```

**逐段点透**：

- **VRL 在 Agent 端做 PII 脱敏**：从架构上消除 PII 泄漏（业务代码漏脱敏 → Agent 兜底，见案例 ⑦）
- **多 sink 路由不同采样率**：Loki 进 INFO 采样后的版本（查询用），S3 进全量（合规/审计用）
- **`exclude` 让 sample 只对 INFO 生效**：ERROR/WARN 永远不采样
- **`compression = "zstd"`**：对日志 zstd 比 gzip 压缩比好 30%，CPU 开销低 50%（见 3.5 的实测）

### 2.5 Loki 的 chunk + index 模型

Loki 的核心设计是 **"index 只索引 label，原文 chunk 化压缩存对象存储"**。

```
日志流（一组 label 唯一确定一条 stream）：
{service="order", level="ERROR", env="prod"}
                       │
                       ▼
                  ┌────────────┐
                  │   Stream   │
                  └────────────┘
                       │
       ┌───────────────┼───────────────┐
       ▼               ▼               ▼
   ┌───────┐       ┌───────┐       ┌───────┐
   │Chunk 1│       │Chunk 2│       │Chunk 3│  ← gzip/snappy 压缩
   │~1MB   │       │~1MB   │       │~1MB   │     存到 S3
   │~2小时 │       │~2小时 │       │~2小时 │
   └───────┘       └───────┘       └───────┘

Index 只记录："这组 label 在 t1~t2 时间段的 chunk 在 S3 哪个对象里"
```

配置 `loki.yaml` 关键段：

```yaml
limits_config:
  ingestion_rate_mb: 10
  ingestion_burst_size_mb: 20
  max_label_name_length: 1024
  max_label_value_length: 4096
  max_label_names_per_series: 30           # ★ 防高基数 label 关键
  cardinality_limit: 100000

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: s3
      schema: v13
      index:
        prefix: loki_index_
        period: 24h

chunk_store_config:
  max_look_back_period: 720h               # 30 天

storage_config:
  tsdb_shipper:
    active_index_directory: /loki/tsdb-index
    cache_location: /loki/tsdb-cache
  aws:
    s3: s3://us-east-1/my-loki-bucket
```

**逐段点透**：

- **`max_label_names_per_series: 30`**：单个 stream 的 label 数上限。超了直接拒绝写入。是防"开发把 request_id / user_id 加成 label" 的防火墙
- **`schema: v13` + `store: tsdb`**：v13 是 Loki 2.9 的最新 schema，TSDB 索引比老的 boltdb 快一倍
- **chunk 周期 `period: 24h`**：每 24h 一个 index 文件，方便按天滚动删除冷数据
- **`ingestion_rate_mb: 10`**：单租户每秒最多 10MB。生产必须按容量规划设，不然爆 OOM

### 2.6 ClickHouse 表设计（按时间分区 + 物化视图）

ClickHouse 存日志的核心模式：**按天分区 + 时间倒序排序 + Bloom Filter 索引**。

```sql
CREATE TABLE logs (
    ts          DateTime64(3, 'UTC'),
    service     LowCardinality(String),
    level       LowCardinality(String),
    trace_id    String,
    span_id     String,
    msg         String,
    fields      Map(String, String),
    INDEX idx_trace trace_id TYPE bloom_filter(0.01) GRANULARITY 1,
    INDEX idx_msg  msg       TYPE tokenbf_v1(32768, 3, 0) GRANULARITY 1
)
ENGINE = MergeTree
PARTITION BY toYYYYMMDD(ts)
ORDER BY (service, level, ts)
TTL toDate(ts) + INTERVAL 30 DAY DELETE,
    toDate(ts) + INTERVAL 7  DAY TO VOLUME 'cold'
SETTINGS storage_policy = 'hot_cold';
```

**逐段点透**：

- **`LowCardinality(String)`**：service/level 这种值很少的列用 `LowCardinality` 包裹，CH 内部用字典编码，存储和查询都快 5~10 倍
- **`PARTITION BY toYYYYMMDD(ts)`**：按天分区让"删 30 天前数据" = "drop partition"，O(1) 操作；且按时间范围查询天然只扫相关分区
- **`ORDER BY (service, level, ts)`**：CH 是稀疏索引，按这个顺序排序意味着 `WHERE service='x' AND level='ERROR'` 是顺序扫描小段
- **`bloom_filter(0.01)`**：trace_id 是高基数随机字符串，全表扫慢；Bloom Filter 让"指定 trace_id" 的查询直接跳过 99% 数据块
- **`tokenbf_v1`**：对 msg 做分词 Bloom Filter，让 `msg LIKE '%timeout%'` 也走索引
- **TTL 双段**：7 天后从 hot 卷（NVMe）迁到 cold 卷（HDD/S3），30 天彻底删

物化视图做"日志即指标"（见 6.1）：

```sql
CREATE MATERIALIZED VIEW error_rate_5min
ENGINE = AggregatingMergeTree
PARTITION BY toYYYYMMDD(ts)
ORDER BY (service, ts)
AS SELECT
    toStartOfFiveMinute(ts) AS ts,
    service,
    countState() AS error_cnt
FROM logs
WHERE level = 'ERROR'
GROUP BY service, ts;
```

写入 logs 时 CH 自动维护这个 5 分钟错误数的预聚合表。Grafana 查"过去 24h 各服务错误率"时直接查这个视图，**比扫原始 logs 快 1000 倍**。

### 2.7 OpenTelemetry Logs 协议

OTel Logs 是日志的"统一传输协议"（Protocol Buffer over gRPC/HTTP），目标是**让 logs/metrics/traces 走同一根管道**。

数据模型核心字段：

```protobuf
message LogRecord {
  fixed64 time_unix_nano = 1;
  fixed64 observed_time_unix_nano = 11;
  SeverityNumber severity_number = 2;
  string severity_text = 3;
  AnyValue body = 5;                       // 日志主体（结构化或文本）
  repeated KeyValue attributes = 6;
  bytes trace_id = 9;                      // ★ 与 trace 自动关联
  bytes span_id = 10;
}
```

业务代码（Java）用 OTel SDK：

```java
Logger logger = openTelemetry.getLogsBridge()
    .loggerBuilder("order-service")
    .setInstrumentationVersion("1.0.0")
    .build();

logger.logRecordBuilder()
    .setSeverity(Severity.ERROR)
    .setBody("order processing failed")
    .setAttribute(AttributeKey.stringKey("orderId"), orderId)
    .setAttribute(AttributeKey.stringKey("userId"), userId)
    .emit();
```

**逐段点透**：

- **`trace_id` 是协议级字段**：不依赖业务在日志里 `MDC.put("traceId", ...)`，OTel SDK 自动从当前 context 取
- **`severity_number` 用整数**（1=TRACE, 5=DEBUG, 9=INFO, 13=WARN, 17=ERROR, 21=FATAL）：跨语言统一，避免字符串比较的坑
- **`AnyValue body`**：可以是字符串、数字、Map、Array —— 兼容结构化和非结构化日志
- **OTel Collector 在中间做协议转换**：业务都说 OTLP，下游可以是 ES/Loki/CH，业务无感切换

> **口诀**：Agent 端做尽可能多的事 —— 解析、过滤、采样、脱敏、预聚合。每往后传一层，成本翻倍。

---

## 3. 实测数据：选型不是拍脑袋

> 以下数据来自社区基准测试 + 生产运维经验，不同硬件、不同 query 形态会有出入，仅供量级参考。

### 3.1 ELK vs Loki vs ClickHouse 在 TB 级下的查询延迟

测试环境：5TB 日志（30 天）、3 节点集群、相同查询。

| 查询类型 | ES (8.x) | Loki (2.9) | ClickHouse (23.x) |
| --- | --- | --- | --- |
| 过去 5 分钟 ERROR 日志 (~10K 条) | 0.3s | **0.2s** | 0.4s |
| 过去 1 小时按 service 聚合错误数 | 1.2s | 8s | **0.3s** |
| 过去 24 小时全文搜 "timeout" | 4s | 35s | 6s |
| 过去 7 天按 trace_id 找日志（命中 100 条） | 0.8s | 25s（如果 trace_id 不是 label） | **0.1s**（有 Bloom Filter） |
| 过去 30 天 P99 错误率趋势图 | 18s | 跑不出来 | **2s**（物化视图） |

**结论**：
- **告警触发后的实时查询** → ES / Loki 都行
- **聚合分析、长时间窗** → ClickHouse 一骑绝尘
- **全文搜索** → ES 仍是体验最好的，Loki 的 grep 模式适合"按 label 缩小后再搜"
- **trace_id 关联** → CH 加 Bloom Filter 最快；Loki 必须把 trace_id 做成 label（高基数风险）；ES 默认 keyword 字段也快

### 3.2 不同采集 Agent 的 CPU / 内存开销

10000 行/秒的日志输入，单 Agent 实例：

| Agent | CPU | 内存 | 二进制大小 | 启动时间 |
| --- | --- | --- | --- | --- |
| **Filebeat** | ~15% | ~80MB | 80MB | 2s |
| **Fluent-bit** | ~5% | **~10MB** | **2MB** | 0.3s |
| **Fluentd** | ~25% | ~120MB | 50MB | 5s |
| **Vector** | ~8% | ~30MB | 50MB | 0.5s |
| **Logstash** | ~50% | ~500MB | 200MB | 30s |

**结论**：
- **K8s DaemonSet 首选 Fluent-bit**：每个节点都跑，资源越省越好
- **业务侧 Sidecar / 复杂 transform 选 Vector**：VRL 比 Fluent-bit 的 Lua 更强大
- **Logstash 现在只在"有 jruby 历史包袱"的场景用**：新项目别选

### 3.3 文本 vs JSON 日志的存储成本

100 万条相同业务日志：

| 格式 | 原始大小 | gzip 后 | zstd 后 | snappy 后 |
| --- | --- | --- | --- | --- |
| 文本（`logger.info("x={} y={}", x, y)`） | **120MB** | 18MB | 16MB | 25MB |
| JSON（含字段名） | 280MB | 32MB | 28MB | 45MB |
| JSON + drop 元数据字段 | 200MB | 24MB | 21MB | 35MB |

**结论**：
- JSON 原始大 2.3 倍，但**压缩后只大 1.5 倍**（字段名重复，压缩率高）
- ELK / Loki 存储成本主要是**压缩后的体积**，所以"JSON 比文本贵 50%"是正确的预算
- 这 50% 的代价换来"可结构化查询" —— **永远值得**

### 3.4 采样 10% 后的诊断有效性

某电商生产观测：

| 日志级别 | 全量 | 采样后保留 | 诊断关键事件覆盖率 |
| --- | --- | --- | --- |
| ERROR / FATAL | 100% | **100%（不采样）** | 100% |
| WARN | 100% | 100% | 100% |
| INFO | 100% | 10% | **97%**（关键路径用 trace 兜底） |
| DEBUG | 100%（仅故障时开） | 1% | 60% |

**关键设计**：
- ERROR/WARN 永远不采样
- INFO 按 trace_id 一致性采样（同一个请求要么全采要么全丢，避免一条链路日志只剩一半）
- DEBUG 平时不开，故障时用动态日志级别开启目标 service

一致性采样的实现（VRL）：

```
hash = to_int(slice!(.trace_id, 0, 8), 16) % 10
if hash == 0 { . }   # 取 1/10
```

### 3.5 日志压缩比（gzip / zstd / snappy）

100GB 原始 JSON 日志：

| 算法 | 压缩后 | 压缩 CPU | 解压 CPU | 适用 |
| --- | --- | --- | --- | --- |
| **gzip** (level 6) | 14GB | 慢 | 中 | 兼容性好，老系统 |
| **zstd** (level 3) | **12GB** | **快** | **快** | 现代首选 |
| **snappy** | 22GB | **极快** | **极快** | Kafka 传输（吞吐优先） |
| **lz4** | 25GB | 极快 | 极快 | 内存中间结果 |

**结论**：
- **存储用 zstd**：压缩比好 + CPU 低
- **Kafka 传输用 snappy**：批次小、解压快、CPU 低
- **gzip 已经没有任何理由**继续用，除非下游强依赖

> **口诀**：先压缩后存储是基本功。**zstd 是 2025 年的默认**，旧的 gzip / snappy 该升级就升级。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① ES 集群被告警日志打挂 —— "凌晨 3 点 Kibana 全红"

#### 现象

凌晨 3 点告警系统疯狂报"日志收集延迟"。登 Kibana 全红，所有 visualization 转圈出不来。ES 集群 status `red`，多个 shard `unassigned`。

业务侧无故障，但**所有依赖日志告警的链路全部失灵**。

#### 根因

排查发现 2 点 47 分某个 batch 服务出 bug，每秒打 5 万条 ERROR 日志（平时 50 条）。Filebeat 拼命采、Logstash 拼命收、ES 拼命写。

ES 写入路径的瓶颈在于 **Lucene 的 segment 创建** + **mapping 解析**：

1. ES 收到 JSON → 解析每个字段判断 mapping → 不存在的字段动态创建 mapping
2. 每条 ERROR 里 `exception.stackTrace` 字段含完整堆栈，每条堆栈都不一样 → 触发 **mapping explosion**（字段数突破 1000 默认上限）
3. mapping 操作要 master 协调，master 节点 CPU 100%
4. master 卡住 → 分片重平衡卡住 → shard `unassigned`

**核心问题**：ES 的 schema-on-write 模式对"日志这种字段不固定"的场景天然不友好。

#### 修复

**临时**：

```bash
# 1. 关掉 Filebeat 那条日志的 input（改 conf 或在 K8s 给 Pod 打标签）
# 2. ES 删掉今天的 index（如果数据没价值）
curl -XDELETE "es:9200/logs-2025.05.09"

# 3. 提高 mapping 上限（缓兵之计）
curl -XPUT "es:9200/logs-*/_settings" -d '{"index.mapping.total_fields.limit": 2000}'
```

**长效**：

1. **业务侧加日志限流**（Logback 自带 `DuplicateMessageFilter`）：

```xml
<turboFilter class="ch.qos.logback.classic.turbo.DuplicateMessageFilter">
    <allowedRepetitions>5</allowedRepetitions>
    <cacheSize>1000</cacheSize>
</turboFilter>
```

2. **Filebeat → Kafka 加缓冲**（不是 Filebeat → ES 直连）：

```yaml
output.kafka:
  hosts: ["kafka:9092"]
  topic: 'logs-raw'
```

Kafka 自动削峰，ES 慢了不会反压到 Filebeat。

3. **ES 模板严格约束 mapping**：

```json
{
  "index_patterns": ["logs-*"],
  "template": {
    "mappings": {
      "dynamic": "false",
      "properties": {
        "@timestamp": {"type": "date"},
        "level": {"type": "keyword"},
        "service": {"type": "keyword"},
        "msg": {"type": "text"},
        "trace_id": {"type": "keyword"},
        "exception": {"type": "object", "enabled": false}
      }
    }
  }
}
```

`dynamic: false` 让未声明字段被存储但**不索引**，避免 mapping 爆炸。`exception.enabled: false` 让异常字段完全不索引（要查的话用 `_source` 全文 grep）。

4. **告警链路独立**：核心告警别走 ES，走 Prometheus + Alertmanager。日志只是辅助诊断。

#### 验证

```bash
# 模拟暴增日志，观察 Kafka 积压、ES 写入延迟、master CPU
# 期望：Kafka 削峰生效；ES 写入限速但不挂；master 平稳
kubectl exec -it batch-bug-simulator -- /bin/sh -c 'while true; do echo error; done'

# 监控指标
curl es:9200/_cat/indices?v
curl es:9200/_cluster/health
```

> **口诀**：ES 集群挂的 90% 是 mapping 问题。**严格 dynamic + Kafka 削峰 + 业务限流**三件套必须配。告警链路别压在日志上。

---

### 案例 ② Filebeat OOM —— "采集不过来日志"

#### 现象

业务高峰期 Filebeat 容器内存涨到 4GB OOM 重启，重启后又涨。Pod 频繁 restart，CPU 也飙高。**业务日志开始大量丢失**。

#### 根因

Filebeat 的内存模型：

```
读文件 → in-memory queue (events: 4096) → output (Kafka/ES)
                  ↑
        如果 output 慢，queue 不会丢，会等
        但 events 是事件数，不是字节数
        单条 1MB 的大日志 × 4096 = 4GB OOM
```

排查 dump 看到大量 `large stacktrace` 日志（业务有个组件 catch + log 异常栈，每条 200KB）。

更糟：业务把 `multiline.max_lines` 调到 10000，单条日志能撑到 2MB。

外加 Kafka 那边在做 broker 重启，Filebeat output 卡 30s，queue 堆满。

#### 修复

**1. 限制单条日志大小**：

```yaml
processors:
  - drop_event:
      when:
        regexp:
          message: '^.{100000,}$'    # 超过 100KB 直接丢
  - truncate_fields:
      fields: ["message"]
      max_bytes: 10000               # 截到 10KB
```

**2. 用字节数而非事件数限制 queue**：

```yaml
queue.mem:
  events: 1024
  flush.min_events: 256
  flush.timeout: 1s
```

Filebeat 没有直接的 `queue.size_bytes`，但通过 `events` × 期望平均大小估算。1024 × 10KB = 10MB queue 上限。

**3. 配 `harvester_limit` 防止打开太多文件**：

```yaml
filebeat.inputs:
  - type: container
    paths: [...]
    harvester_limit: 100             # 单个 input 最多并发读 100 个文件
    close_inactive: 5m               # 5 分钟没新行的文件关掉
    close_renamed: true              # 文件被改名（log rotate）立即关
    clean_inactive: 24h              # 24 小时不活跃的 file state 清掉
```

**4. 资源限制 + 重启**：

```yaml
resources:
  requests:
    memory: 200Mi
    cpu: 100m
  limits:
    memory: 500Mi                    # ★ 撞到上限直接 OOM 重启，避免拖累节点
    cpu: 500m
```

**5. 业务侧改造**：

- 异常栈日志改用 `LoggingMeterRegistry` 之类的指标化方案
- 不要在 hot path 里 `log.error("xxx", e)`，做采样

#### 验证

```bash
# 压测：业务每秒产生 1MB 大日志 × 100 条
# 期望：Filebeat 内存稳定 < 500MB，丢弃过大日志，不 OOM

kubectl top pod -l app=filebeat
kubectl logs filebeat-xxx | grep "drop_event"
```

> **口诀**：Filebeat OOM 90% 是单条日志过大。**控制单条 + 控制 queue + 控制 harvester** 三件套。

---

### 案例 ③ 多行日志被切碎 —— "Java 异常栈散成 17 条"

#### 现象

业务报"看到 NPE 但点不开异常栈"。在 Kibana 里看到一条 `NullPointerException`，下面跟着 17 条 `at com.example.xxx`，每条独立。搜异常类找不到完整栈。

#### 根因

Java 异常栈的格式：

```
2025-05-09 14:32:11 ERROR [pool-3-thread-1] com.example.OrderService - failed to process
java.lang.NullPointerException: order is null
    at com.example.OrderService.process(OrderService.java:42)
    at com.example.OrderService.handle(OrderService.java:30)
    at java.base/java.lang.Thread.run(Thread.java:842)
```

容器 stdout 是逐行写的。Filebeat 默认按行采集，**不知道哪几行属于同一个事件**。

如果你没配 multiline，Filebeat 把每行当独立日志送出去：第一行有 timestamp 是完整 JSON，下面的 `at xxx` 行变成 `{"message": "    at com.example...", "timestamp": <采集时间>}`，丢失了上下文。

#### 修复

**Filebeat 端配多行合并**（首选 —— 在数据源最近处处理）：

```yaml
filebeat.inputs:
  - type: container
    paths: [...]
    multiline.type: pattern
    multiline.pattern: '^\d{4}-\d{2}-\d{2}'   # 行首是日期 = 新事件
    multiline.negate: true                     # 不匹配该 pattern 的行
    multiline.match: after                     # 附加到上一条
    multiline.max_lines: 500                   # 单事件最多 500 行
    multiline.timeout: 5s
```

**业务侧用 JSON 输出避免该问题**（更优）：

用 `LogstashEncoder` 后整个异常栈进入单个 JSON 字段：

```json
{
  "@timestamp": "...",
  "level": "ERROR",
  "msg": "failed to process",
  "stack_trace": "java.lang.NullPointerException: order is null\n\tat com.example..."
}
```

stdout 是单行，Filebeat 不需要做 multiline，直接送。

**Loki / Promtail 也支持多行合并**：

```yaml
scrape_configs:
  - job_name: kubernetes-pods
    pipeline_stages:
      - multiline:
          firstline: '^\d{4}-\d{2}-\d{2}'
          max_wait_time: 3s
          max_lines: 500
```

**最坑的是 Docker JSON log driver 自己也会切**：单条超过 16KB 时 Docker 会拆成多个 JSON 对象（log split）。Filebeat 8.x 有 `combine_partial: true` 自动合：

```yaml
filebeat.inputs:
  - type: container
    parsers:
      - container:
          stream: stdout
          format: docker
      - multiline:
          type: pattern
          pattern: '^\d{4}'
          negate: true
          match: after
```

#### 验证

```bash
# 触发一个有异常栈的请求
curl http://service/trigger-npe

# Kibana 查最近 10s 的日志
GET logs-*/_search
{
  "query": {"match": {"level": "ERROR"}},
  "sort": [{"@timestamp": "desc"}],
  "size": 1
}
# 期望：返回 1 条事件，stack_trace 字段完整含所有 "at" 行
```

> **口诀**：**多行合并必须做，且越靠近源越好**。最优解是业务直接出 JSON，异常栈作为字段；次优解是 Filebeat 端 multiline。

---

### 案例 ④ 采样错过关键诊断 —— "故障时正好这一条没采到"

#### 现象

线上偶发"用户下单失败"。打开日志查这个 userId，**只查到下单前的几条 INFO，关键的"调用支付接口"那一条没了**。开发说"INFO 是 10% 采样的"。

#### 根因

简单的 10% 采样（`hash(line) % 10 == 0`）有两个致命问题：

1. **同一个请求的多条日志可能不都被采**：A 步骤被采、B 步骤没被采，链路断裂
2. **采样没有上下文感知**：故障路径和正常路径同等概率被丢，但故障路径才是诊断关键

#### 修复

**改造一：按 trace_id 一致性采样**

```
sampled = hash(trace_id) % 10 == 0
```

同一个 trace 要么全采要么全不采。一个用户的下单全链路日志要么完整、要么完全没有。

VRL 实现：

```
.sample_decision = (to_int!(slice!(.trace_id, 0, 8), 16) % 100) < 10
if !.sample_decision && .level == "INFO" {
    abort
}
```

**改造二：错误路径反向不采样**

如果一个 trace 出现过 ERROR，则该 trace 所有日志（包括之前的 INFO）必须保留。

实现方式 A（贵但简单）：业务自己在日志里打 `outcome: success/failure`，采样器看到 failure 就回查。
实现方式 B（架构）：用 OTel Collector 的 `tail_sampling_processor`，缓存最近 N 秒的 trace，如果该 trace 任意 span 标记 error → 整个 trace 落地。

```yaml
processors:
  tail_sampling:
    decision_wait: 10s
    num_traces: 100000
    policies:
      - name: error-policy
        type: status_code
        status_code: { status_codes: [ERROR] }
      - name: probabilistic-policy
        type: probabilistic
        probabilistic: { sampling_percentage: 10 }
```

**改造三：日志全量保留 + 采样仅作用于"查询"**

让 Vector 写两路：S3 全量 + Loki 采样。日常查询走 Loki 快，故障时查 S3 全量（用 Athena / Trino）。

```toml
[sinks.loki_sampled]
  inputs = ["sample_info"]
  ...

[sinks.s3_full]
  inputs = ["parse_json"]    # 不经过 sample
  type = "aws_s3"
  ...
```

#### 验证

```bash
# 故意让某 service 1% 的请求触发 ERROR
# 跑 10000 次请求，统计 ERROR trace 的日志完整率

# 期望：ERROR trace 的日志 100% 完整；INFO trace 10% 完整
```

> **口诀**：**采样必须按 trace_id 一致性、且错误路径优先保留**。简单的 hash(line)%N 采样是错的。

---

### 案例 ⑤ Loki 高基数 label —— "index 一夜涨到 200GB"

#### 现象

开发听说 Loki 便宜，把 `request_id`、`user_id`、`trace_id` 都加成 label：

```yaml
labels:
  service: order
  level: ERROR
  request_id: "${request_id}"
  user_id: "${user_id}"
```

第二天 Loki 集群告警："index 异常增长"。监控看 index 从 5GB 涨到 200GB，查询全部超时。

#### 根因

Loki 的核心模型：**一组唯一 label 值组合 = 一个 stream**。

- service=order, level=ERROR, request_id=req-1 → stream A
- service=order, level=ERROR, request_id=req-2 → stream B
- service=order, level=ERROR, request_id=req-3 → stream C
- ...

每个 stream 在 index 里都有独立条目。如果 `request_id` 是请求级唯一的随机值，**每个请求一个 stream**：

- 1000 QPS × 86400s = 8640 万个 stream / 天

每个 stream 的 chunk 也是孤立的（因为 label 不同），没法合并压缩。chunk 数爆炸 → S3 小文件爆炸 → 查询要扫描的 chunk 元数据爆炸 → 查询超时。

#### 修复

**铁律：Loki 的 label 只用于"维度选择"，不用于"事件标识"**。

label 候选规则：
- ✅ service / level / env / region / cluster（值域固定且小）
- ❌ request_id / trace_id / user_id / order_id（高基数）
- ❌ status_code（如果是任意整数）；若仅 200/4xx/5xx 几个值 OK

trace_id 怎么搜？放进 **日志正文（line）**，不放 label。LogQL 用 `|` 过滤：

```logql
{service="order", level="ERROR"} |= "trace_id=abc123"

# 或者 JSON 解析后过滤
{service="order"} | json | trace_id="abc123"
```

虽然慢一点（要扫 chunk），但**总成本远低于把 trace_id 做成 label**。

**配置层加防火墙**：

```yaml
limits_config:
  max_label_names_per_series: 15            # 单 stream label 数上限
  cardinality_limit: 100000                 # 全局 stream 数上限
  max_streams_per_user: 10000               # 单用户 stream 数

# 拒绝高基数 label 的写入
ingester:
  max_chunks_per_query: 2000000
```

**用 Promtail 的 stage 提取 label 后立即删除高基数字段**：

```yaml
pipeline_stages:
  - json:
      expressions:
        level: level
        service: service
        request_id: request_id
  - labels:
      level:
      service:
      # ★ request_id 不放 labels（即使 json 解析了）
  - labeldrop:
      - request_id                         # 显式从 metadata 里删掉
```

**事后清理**：

```bash
# 删除高基数 stream（Loki 2.6+）
curl -X POST 'http://loki:3100/loki/api/v1/delete' \
  -d 'query={request_id=~".+"}' \
  -d 'start=2025-05-08T00:00:00Z' \
  -d 'end=2025-05-09T00:00:00Z'
```

#### 验证

```bash
# 查当前总 stream 数
curl -s 'http://loki:3100/metrics' | grep loki_ingester_memory_streams

# 单查询 stream 数
logcli query '{service="order"}' --stats

# 期望：总 stream 数 < 10万；单查询返回 stream < 1000
```

> **口诀**：**Loki label = Prometheus label**。维度选择用 label，事件标识用日志正文。`trace_id` 从来都不该是 label。

---

### 案例 ⑥ ClickHouse 表设计错误 —— "查询慢 100 倍"

#### 现象

用 ClickHouse 存日志，最简单的查询都要 30 秒：

```sql
SELECT count() FROM logs
WHERE service = 'order' AND level = 'ERROR'
  AND ts > now() - INTERVAL 1 HOUR
```

`EXPLAIN` 看到全表扫描，扫了 50 亿行。

#### 根因

最初表结构：

```sql
CREATE TABLE logs (
    id UInt64,
    ts DateTime,
    service String,
    level String,
    msg String
)
ENGINE = MergeTree
ORDER BY id;            -- ★ 错的关键
```

`ORDER BY id` 让 CH 按 id（递增）建稀疏索引。`WHERE service='x'` 没法走索引。

`PARTITION BY` 也没设，所有数据在一个分区，TTL 删除等于全表 rewrite。

`service` / `level` 是普通 String，CH 每次比较都做字符串运算。

#### 修复

按 2.6 的设计重建：

```sql
CREATE TABLE logs_v2 (
    ts DateTime64(3),
    service LowCardinality(String),         -- ★ 字典编码
    level LowCardinality(String),
    trace_id String,
    msg String,
    INDEX idx_trace trace_id TYPE bloom_filter(0.01) GRANULARITY 4
)
ENGINE = MergeTree
PARTITION BY toYYYYMMDD(ts)                 -- ★ 按天分区
ORDER BY (service, level, ts)               -- ★ 顺序匹配查询模式
TTL toDate(ts) + INTERVAL 30 DAY DELETE;
```

数据迁移：

```sql
INSERT INTO logs_v2 SELECT
    toDateTime64(ts, 3) AS ts,
    service, level, trace_id, msg
FROM logs
WHERE ts > now() - INTERVAL 30 DAY;
```

**关键认知**：CH 的 `ORDER BY` 决定**物理存储顺序** + **稀疏索引顺序**。设计原则：

1. **从基数低到高**排：`service`（10 个值）→ `level`（5 个）→ `ts`（连续）
2. **最常 WHERE 的列放最前**
3. **时间字段放最后**（因为同一 service+level 内时间是连续的）

`PARTITION BY` 决定**物理目录划分**：
- 太粗（按月）：单分区太大，TTL 删除慢
- 太细（按小时）：分区数爆炸，metadata 开销大
- **按天 (`toYYYYMMDD`)** 是日志场景的甜点

**进阶：物化视图预聚合**

```sql
CREATE MATERIALIZED VIEW error_stats_5m
ENGINE = SummingMergeTree
PARTITION BY toYYYYMMDD(ts)
ORDER BY (service, ts)
AS SELECT
    toStartOfFiveMinute(ts) AS ts,
    service,
    sum(if(level = 'ERROR', 1, 0)) AS error_cnt,
    sum(if(level = 'WARN',  1, 0)) AS warn_cnt
FROM logs_v2
GROUP BY service, ts;
```

Grafana 查"过去 24h 错误率趋势" → 直接查 5 分钟粒度的预聚合表，**1000x 提速**。

#### 验证

```sql
-- 改造前后查询时间对比
SELECT count() FROM logs WHERE service = 'order' AND level = 'ERROR'
  AND ts > now() - INTERVAL 1 HOUR;
-- 期望：从 30s → < 1s

-- 看分区情况
SELECT partition, count(), formatReadableSize(sum(bytes_on_disk))
FROM system.parts WHERE table = 'logs_v2' AND active
GROUP BY partition ORDER BY partition DESC;
```

> **口诀**：CH 表设计 = `PARTITION BY 时间` + `ORDER BY 低基数→高基数→时间` + `LowCardinality + Bloom Filter`。这三条做到，性能 10x。

---

### 案例 ⑦ 日志泄漏 PII —— "合规检查发现手机号明文"

#### 现象

合规团队抽查发现，业务日志里有大量明文手机号、身份证号、银行卡号。监管要求 7 天内整改，否则停业务。

#### 根因

代码里类似这种：

```java
log.info("user login: {}", request);
// User 的 toString() 包含 phone="13800138000"
```

或：

```java
log.warn("payment failed for {}", payment);
// Payment.toString() 含 cardNo="6225..."
```

依赖业务开发"记得脱敏"是不可靠的。1000 个开发里有 999 个偶尔会忘。

#### 修复

**多层防御**：

**第一层：业务侧消除 toString 隐患**

用 Lombok 的 `@ToString.Exclude`：

```java
@ToString
public class User {
    private Long id;
    private String name;
    @ToString.Exclude private String phone;
    @ToString.Exclude private String idCard;
}
```

或用自定义注解 + 字节码增强：

```java
@PII(type = PIIType.PHONE)
private String phone;
```

`toString` 自动输出 `phone="138****8000"`。

**第二层：日志框架统一脱敏**

实现 Logback 的 `MessageConverter`：

```java
public class PiiMaskConverter extends ClassicConverter {
    private static final Pattern PHONE = Pattern.compile("1[3-9]\\d{9}");
    private static final Pattern ID_CARD = Pattern.compile("\\d{17}[\\dXx]");

    @Override
    public String convert(ILoggingEvent event) {
        String msg = event.getFormattedMessage();
        msg = PHONE.matcher(msg).replaceAll("***-PHONE-***");
        msg = ID_CARD.matcher(msg).replaceAll("***-IDCARD-***");
        return msg;
    }
}
```

```xml
<encoder>
    <pattern>%d %level %logger - %piiMask</pattern>
</encoder>
```

但这个方案有性能开销（每条日志走正则）+ 漏洞（混淆字符绕过）。

**第三层：Agent 端兜底**

在 Vector / Fluent-bit 端加 transform，无论业务怎么写，**出节点前必然脱敏**：

```toml
[transforms.pii_mask]
  type = "remap"
  source = '''
    .msg = replace(.msg, r'1[3-9]\d{9}', "***-PHONE-***")
    .msg = replace(.msg, r'\d{17}[\dXx]', "***-IDCARD-***")
    .msg = replace(.msg, r'\d{16,19}', "***-CARD-***")
  '''
```

**第四层：分级访问 + 审计**

合规要求"必要时能解密"的场景，用**保留密文 + KMS 解密**：

- 日志里写 `phone_enc=ENC(...)` 加密版
- 解密接口走审批流，每次解密留审计

**第五层：扫描 + 阻断**

定期跑日志全文扫描（grep 正则），命中就告警：

```bash
clickhouse-client --query "
SELECT service, count()
FROM logs
WHERE match(msg, '1[3-9]\\d{9}|\\d{17}[\\dXx]')
  AND ts > now() - INTERVAL 1 DAY
GROUP BY service
HAVING count() > 0"
```

#### 验证

```bash
# 业务侧故意打一条带手机号的日志，从 Kibana 查询确认已脱敏
log.info("test phone 13800138000")

# 期望：Kibana 看到 "test phone ***-PHONE-***"
```

> **口诀**：PII 防御**多层兜底**。靠业务记得脱敏 = 早晚出事。**Agent 端必须有最后一道兜底**。

---

### 案例 ⑧ trace ID 与日志关联失败 —— "点过去跳错链路"

#### 现象

业务接入了 SkyWalking，配置 Logback 输出 trace_id。Kibana 里点 trace_id 跳到 SkyWalking 链路系统，**有时跳出来的是另一个完全不相关的 trace**。

#### 根因

排查发现：

1. 业务用 `MDC.put("traceId", tracer.currentTraceId())`，但 SkyWalking 用 segmentId 不是 traceId
2. 部分异步任务的 MDC 没传递（见 [线程池案例 ⑤](/article/java/threadpool#案例--mdc--traceid-失踪-链路追踪在线程池断了)），traceId 是空或残留上次的
3. Kibana 跳转 URL 用的是 traceId，但 SkyWalking 那边按 segmentId 索引

**根本问题**：trace 上下文的传递在多个组件、多个线程边界、多个协议（HTTP / gRPC / Kafka）之间容易破碎。

#### 修复

**统一 trace_id 标准 —— OpenTelemetry**

OTel 规定 trace_id 是 16 字节十六进制（32 字符）。所有组件（应用、SDK、collector、后端）都遵守。

业务侧用 OTel SDK：

```java
Span span = Span.current();
String traceId = span.getSpanContext().getTraceId();
MDC.put("trace_id", traceId);
MDC.put("span_id", span.getSpanContext().getSpanId());
```

或用 OTel 的 Logback Appender 自动注入：

```xml
<appender name="OTLP" class="io.opentelemetry.instrumentation.logback.appender.v1_0.OpenTelemetryAppender">
</appender>
```

它读取当前 `Context` 自动写入 trace_id / span_id 字段，**不需要业务手动 MDC**。

**线程池透传**：

```java
ExecutorService pool = Context.taskWrapping(rawPool);

// 或显式
Context ctx = Context.current();
pool.submit(() -> ctx.wrap(actualWork).run());
```

**Kafka / RPC 跨进程透传**：

OTel 自动注入 W3C `traceparent` header：

```
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
```

下游 OTel SDK 自动从 header 提取并续上 context。

**Kibana 跳转链接配置**：

```json
{
  "field": "trace_id",
  "value": "{{value}}",
  "type": "url",
  "url_template": "https://jaeger.internal/trace/{{value}}"
}
```

**测试链路完整性**：

```java
// 故意跨 5 个 service + 1 个 Kafka，最后查日志
// 期望：每个 service 的日志都有相同的 trace_id；Jaeger 里也能找到
```

#### 验证

```bash
# 1. 发起一个请求
curl -H "x-trigger-test: 1" http://entry-service/api

# 2. 从响应里拿到 trace_id（OTel 标准是 traceparent header）
trace_id=$(curl -i http://entry-service/api | grep traceparent | cut -d- -f2)

# 3. 在所有 5 个 service 的日志里搜
for svc in entry order payment ship notify; do
  logcli query "{service=\"$svc\"} |= \"$trace_id\""
done
# 期望：每个 service 都有 1+ 条日志

# 4. Jaeger 查同一 trace_id
curl http://jaeger:16686/api/traces/$trace_id
# 期望：返回完整 trace
```

> **口诀**：trace_id 关联**必须从协议层统一**。OTel 是当前最佳实践，自动处理 MDC、跨线程、跨进程。

---

### 案例 ⑨ 日志采集延迟导致告警滞后 —— "故障 5 分钟才报警"

#### 现象

线上故障 14:32 发生，告警 14:37 才触发。监控曲线显示业务 14:32 就异常，错误日志的告警却晚了 5 分钟。客户已经在群里炸了。

#### 根因

告警链路：业务打日志 → Filebeat 采 → Kafka → Logstash → ES → ElastAlert 查询 → 触发告警。

每一段都有延迟：

| 环节 | 典型延迟 |
| --- | --- |
| Filebeat scan_frequency 默认 10s | 0~10s |
| Kafka 端到端 | < 1s |
| Logstash 批处理 (`flush_size: 500`) | 0~5s |
| ES refresh_interval 默认 1s | 0~1s |
| ElastAlert 默认 1 分钟轮询 | 0~60s |
| 告警通道（钉钉/Slack）发送 | 0~10s |

**累计 ~80s 是常见的；最坏 5 分钟不夸张**。

#### 修复

**告警永远不要走"日志查询"链路。走 metric 链路**。

正确架构：

```
业务 → 打日志 + 同时打 metric (logback-metrics 或 OTel 双写)
                      ↓                              ↓
                   日志链路                    Prometheus
                  （秒级延迟）                  （秒级 scrape）
                      ↓                              ↓
                  ES/Loki                      Alertmanager
                  （查询用）                    （秒级触发）
```

具体做法：

**业务侧用 Micrometer 把"错误"作为 metric 暴露**：

```java
Counter errorCounter = Counter.builder("biz.error.count")
    .tag("type", "order")
    .register(meterRegistry);

try {
    process(order);
} catch (Exception e) {
    errorCounter.increment();
    log.error("order failed", e);
}
```

Prometheus 抓 metric，告警规则：

```yaml
groups:
- name: biz
  rules:
  - alert: HighOrderErrorRate
    expr: rate(biz_error_count{type="order"}[1m]) > 10
    for: 30s
    annotations:
      summary: "订单错误率超过阈值"
```

延迟：scrape 15s + for 30s = **45s 内告警**，比走日志快 4~5 倍。

**日志只用作"告警出来了之后跳过去看上下文"**。

**如果一定要从日志告警**（比如老系统改不动）：

- Loki + Promtail 的 `metrics` stage 在 Agent 端就把日志转 metric：

```yaml
pipeline_stages:
  - regex:
      expression: 'level=(?P<level>\w+)'
  - metrics:
      log_lines_total:
        type: Counter
        config:
          action: inc
          match_all: true
      log_errors_total:
        type: Counter
        source: level
        config:
          value: ERROR
          action: inc
```

Promtail 暴露 metric，Prometheus 抓。延迟同 metric 链路。

#### 验证

```bash
# 故意触发故障，分别记录：
# t0 = 故障发生时刻
# t1 = Prometheus 告警触发时刻
# t2 = 日志告警触发时刻
# 期望：t1 - t0 < 60s；t2 - t0 可能 > 3min

curl -X POST http://service/inject-error
date  # t0

# 钉钉/Slack 告警时间
```

> **口诀**：**告警走 metric，不走日志**。日志查询的延迟不可控；metric 是秒级的。日志的角色是"告警来了之后查根因"。

---

## 5. 上线检查清单

打印贴墙 / 加进 SRE Code Review checklist。

### 业务侧（生成层）

- [ ] 日志输出 JSON 格式（`LogstashEncoder` 或等价）
- [ ] 每条日志都有 `service` / `env` / `level` / `@timestamp` 字段
- [ ] trace_id / span_id 通过 OTel 自动注入，不依赖业务 MDC
- [ ] 异步任务（线程池、Kafka consumer）的 MDC 透传已配置
- [ ] User / Payment 等含 PII 的对象的 `toString` 已脱敏
- [ ] 异常栈不在 hot path 里循环打印（每秒不超过 N 条）
- [ ] 用 Micrometer 把"业务错误"也暴露为 metric（告警走 metric 不走日志）

### 采集层（Agent）

- [ ] Filebeat / Fluent-bit 配置了 `multiline.max_lines` 和 `multiline.timeout`
- [ ] 单条日志大小限制（截断或丢弃）
- [ ] queue 大小按"事件数 × 平均字节"估算，限制内存上限
- [ ] Pod resource limit 配置 `memory.limit`，撞上限直接重启
- [ ] `harvester_limit` / `close_inactive` 防止打开太多文件
- [ ] Agent 端做了 PII 脱敏 transform（兜底）
- [ ] Agent 端做了一致性采样（按 trace_id），ERROR/WARN 不采样

### 传输层

- [ ] Filebeat → Kafka，不是 Filebeat → ES 直连（削峰）
- [ ] Kafka topic 按 namespace / service 分流
- [ ] Kafka 用 snappy / zstd 压缩
- [ ] 消费者侧（Logstash / Vector）有限速，不打挂下游

### 存储层

- [ ] ES 模板设 `dynamic: false` 或 `strict`，禁止动态字段
- [ ] ES `index.mapping.total_fields.limit` 显式设值（默认 1000 易爆）
- [ ] ES index 按时间滚动 + ILM 冷热分层 + TTL 删除
- [ ] Loki 的 label 数限制 (`max_label_names_per_series`) 显式配
- [ ] Loki 不允许 trace_id / request_id 等高基数字段做 label
- [ ] ClickHouse 按天分区 + 低基数排序键 + LowCardinality 类型
- [ ] ClickHouse 关键查询配物化视图预聚合

### 查询 & 告警

- [ ] 关键告警走 Prometheus metric，不走日志查询
- [ ] 日志查询有 query DSL 上限（max docs / max fields）
- [ ] Kibana / Grafana 跳转 trace_id 链接已测试
- [ ] 长时间窗查询走 ClickHouse / Athena，不查 ES 全量

### 合规层

- [ ] PII 字段（手机/身份证/银行卡）有脱敏 + 扫描告警
- [ ] 日志保留期限符合监管（金融 5+ 年，通用 6 个月）
- [ ] 日志访问有审计（谁查了什么 trace_id）
- [ ] 加密字段需要审批流才能解密

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### 日志即指标（log-based metrics）

把"统计错误率"这种聚合操作前置到 Agent 端：
- Promtail 的 `metrics` stage
- Vector 的 `log_to_metric` transform
- Fluent-bit 的 stream processor
- Datadog Logs 的 generated metrics

**好处**：metric 比日志便宜 100x；告警延迟降到秒级；历史指标永久保留，原始日志按需保留。

**陷阱**：标签维度爆炸（高基数 metric 比高基数 log 还致命）；动态规则难管理。

### eBPF 自动注入日志

不改业务代码，用 eBPF 钩内核 / libc / JVM 的关键函数自动产生日志：
- HTTP 请求自动注入 access log（不依赖框架）
- 系统调用层抓取所有网络 IO
- 工具：Cilium Tetragon、Pixie、Parca

**适用**：遗留系统补观测；安全审计场景；性能 profiling。

### 日志的 AI 异常检测

NLP 模型对日志做："这条日志和过去 30 天 99% 的日志不一样" 判定：
- ELK X-Pack 的 anomaly detection
- 自训 transformer（log embedding）
- LLM 总结故障日志（GPT/Claude）

**实战经验**：异常检测对"已知模板的偏差"有用（比如某 service 突然出现新 ERROR pattern）；对"全新故障"误报率高。

### GDPR / 数据安全合规

- 日志的"可被遗忘"要求：用户删号后 N 天内日志要清掉相关 PII
- 跨境数据传输：欧盟用户的日志不能存到美国节点
- 日志加密：传输 TLS + 静态 KMS 加密 + 字段级加密
- 工具：HashiCorp Vault Transit Engine、AWS KMS、阿里 KMS

### 长尾分析与热点

- 日志做"用户行为分析"（漏斗、留存、转化）
- 异常聚类（用 simhash 找出"95% 相似的报错"）
- 日志驱动的 capacity planning（看流量曲线规划扩容）

### SaaS 日志服务

- **Datadog Logs**：贵但功能全，"Live Tail" + "Log Patterns" 体验顶级
- **Splunk**：传统企业首选，SPL 查询语言学习曲线陡
- **Sumo Logic**：定位中端
- **阿里云 SLS / 腾讯云 CLS**：国内首选，按写入量计费

**自建 vs SaaS 决策点**：日均 < 100GB 用 SaaS 划算；> 1TB 自建 ROI 高。中间地带按团队精力定。

### 自研日志平台的成本曲线

某中厂日志成本曲线（参考）：

| 日均量 | 方案 | 月成本 | 备注 |
| --- | --- | --- | --- |
| 10GB | Datadog Logs | $300 | 闭眼 SaaS |
| 100GB | ELK on K8s | $1500 (含人力) | 自建甜点开始 |
| 1TB | Loki + S3 | $3000 | Loki 真便宜的区间 |
| 10TB | ClickHouse 集群 | $15000 | 必须自建，需专职 SRE |
| 100TB+ | 自研 (类 Scuba) | $100000 | 大厂级别投入 |

---

## 7. 引用

- [Elasticsearch: The Definitive Guide](https://www.elastic.co/guide/en/elasticsearch/guide/current/index.html)
- [Loki Architecture](https://grafana.com/docs/loki/latest/get-started/architecture/) —— Cortex 的姊妹项目
- [ClickHouse Documentation: MergeTree](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/mergetree)
- [OpenTelemetry Logs Specification](https://opentelemetry.io/docs/specs/otel/logs/)
- [Vector Documentation](https://vector.dev/docs/) —— Datadog 出品的统一 Agent
- Honeycomb 博客：*"What's a Wide Event?"* —— 结构化日志的工程论
- Google SRE Book Chapter 6: *"Monitoring Distributed Systems"*
- [Lyft 工程博客 ELK 优化系列](https://eng.lyft.com/) —— 大规模 ELK 实战
- 阿里云 SLS [日志最佳实践](https://help.aliyun.com/product/28958.html)
- Brendan Gregg, *Systems Performance* —— 日志与性能分析的关联

---

> 配套：[可观测性主题](/article/observability/) | [Prometheus 指标设计](/article/observability/prometheus) | [链路追踪深入](/article/observability/tracing)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
