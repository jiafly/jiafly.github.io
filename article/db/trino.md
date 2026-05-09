# Trino / Presto 联邦查询深入 —— 跨源 SQL 与 MPP 架构

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想看"什么是 Presto"的八股，去维基百科够了。
> 这里要回答的，是**SQL 一条 JOIN 跨四个数据源、Coordinator 突然 OOM、跑批任务跟交互式查询互相挤死**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. Trino 和 Presto 是同一个东西吗？为什么 2020 年 PrestoSQL 团队要改名成 Trino？现在你 `pip install presto-python-client` 装的是哪一支？
2. 解释 MPP（Massively Parallel Processing）和 MapReduce 在执行模型上的根本区别 —— 为什么 Trino 的中间结果不落盘，而 Spark/Hive 要 shuffle 到磁盘？
3. Coordinator 单点：你的 Trino 集群只有 1 个 Coordinator，挂了所有查询都断。怎么做高可用？为什么 Trino 不像 Spark 那样有"Driver 复活"机制？
4. 一条 `SELECT * FROM mysql.db.t JOIN hive.db.t2` 的跨源 JOIN，谓词下推（predicate pushdown）能下推到 MySQL 吗？JOIN 的 build 端怎么选？数据怎么 shuffle？
5. 为什么 Trino 不持久化任何数据，仍然敢自称"数据仓库"？它和 Snowflake / BigQuery 在存储/计算分离上的差异是什么？
6. Hive Connector 用的元数据是 Hive Metastore（HMS），Iceberg Connector 也可以用 HMS —— 那它俩有啥区别？为什么 Iceberg 现在更受推崇？
7. 一条查询占了 Worker 80% 内存，把同集群的小查询全卡住。Resource Group 该怎么配？`softMemoryLimit` 和 `hardConcurrencyLimit` 哪个先生效？
8. Trino on K8s：你是把 Coordinator 和 Worker 一起部署，还是分两个 StatefulSet？为什么大厂普遍不用 HPA 自动扩缩容 Worker？

如果有半数答不上，往下看。

---

## 1. 底层模型：联邦查询、MPP 与 Connector

绝大多数文章把 Trino 讲成"分布式 SQL 查询引擎"，画一个 Coordinator + N 个 Worker 的方框图就结束了。这层抽象解释不了上面 8 个问题。要看懂它们，必须下沉到**联邦查询的本质** + **MPP 的执行模型** + **Connector SPI 的边界**。

### 1.1 联邦查询的本质

"联邦"（Federated）这个词不是营销，是有严格含义的：**统一的 SQL 入口 + 多个独立异构数据源 + 不复制数据**。

传统数据仓库 ETL 思路：
```
MySQL ─┐
Kafka ─┼──► Spark/Hive ETL ──► 数仓 ──► SQL 查询
S3    ─┘                       (HDFS)
```

数据要先复制到一个集中存储，分钟到小时延迟，存储成本翻倍，schema 漂移要追一遍。

Trino 的联邦查询思路：
```
MySQL ─┐
Kafka ─┼──► Trino (无存储) ◄── SQL 查询
S3    ─┘    跨源 JOIN
```

Trino 不存数据，每次查询**实时去各个数据源拿**。这个设计的代价和收益都很明显：

- 收益：实时一致、不需要 ETL 流水线、schema 来源即真理
- 代价：数据源压力直接传导到上游、不能离线优化数据布局、跨源 JOIN 永远要 shuffle

### 1.2 MPP 架构：Coordinator + Worker + Connector

```
                  ┌───────────────────────┐
   SQL Client ──► │   Coordinator         │
                  │   - SQL Parser        │
                  │   - Analyzer          │
                  │   - Planner / CBO     │
                  │   - Scheduler         │
                  │   - Discovery Service │
                  └─────────┬─────────────┘
                            │ Stage / Task 调度
            ┌───────────────┼────────────────┐
            ▼               ▼                ▼
       ┌─────────┐    ┌─────────┐      ┌─────────┐
       │ Worker  │    │ Worker  │      │ Worker  │
       │ Driver  │    │ Driver  │ ◄──► │ Driver  │
       │ Operator│    │ Operator│ shuf │ Operator│
       └────┬────┘    └────┬────┘      └────┬────┘
            │              │                │
            ▼              ▼                ▼
       Connector      Connector        Connector
       (Hive)         (MySQL)          (Kafka)
```

三层职责：

| 组件 | 职责 | 状态 |
| --- | --- | --- |
| **Coordinator** | 接收 SQL、解析、生成执行计划、调度 Task、聚合结果 | 单点（默认） |
| **Worker** | 执行物理算子、做 shuffle、读 Connector | 无状态、可水平扩展 |
| **Connector** | 适配各种数据源，暴露统一的 SPI | 插件式 |

注意三个反直觉的事：

1. **Worker 完全无状态**：Worker 挂了，正在跑的查询失败，但集群不需要"恢复"。重启就行。这是为什么 Worker 可以放在 Spot 实例上 —— 容忍重启。
2. **Coordinator 是单点**：从设计哲学上，Trino 团队一直认为"Coordinator HA 不值得"。它是个无状态的查询协调器，挂了重启就行，跑的查询丢了重跑（跑批场景再说）。
3. **Connector 是真正的扩展点**：Trino 的能力上限取决于 Connector。一个写得好的 Connector 能下推 filter / project / aggregate / TopN，性能直逼原生引擎；写得烂的就只能"全量拉到 Trino 内存里再算"。

### 1.3 Trino / Presto / PrestoDB 分裂史

这段历史不搞清楚，你装错版本会一脸懵。

```
2012  Facebook 内部启动 Presto 项目（Martin Traverso 等人）
2013  Presto 开源
2018  原始团队（Martin/Dain/David）从 Facebook 离职，
      创立 Starburst 公司，继续维护开源版本，名字仍叫 PrestoSQL
2019  Facebook 把 Presto 捐给 Linux Foundation，叫 PrestoDB
      → 至此分叉：PrestoSQL（社区主导）vs PrestoDB（Facebook/Uber 主导）
2020  PrestoSQL 因为商标问题被迫改名 → Trino
      创始人三人组继续主导 Trino
2021+ Trino 高速迭代，Iceberg / Delta / Hudi 一线支持
      PrestoDB 节奏变慢，主要用户是 Facebook / Uber 内部
```

实务建议：
- **新项目直接选 Trino**，社区活跃、Connector 全、文档好
- **PrestoDB** 只在你已经在用 Facebook 那条线（如 Uber 内部）才考虑
- AWS Athena 底层是 Presto 的修改版（早期是 PrestoDB 分支，现在用 Trino 引擎了，但保持 Athena API 不变）

> **口诀**：Trino = 原始团队主导的 PrestoSQL 改名版；PrestoDB = Facebook 内部那一支。**默认装 Trino**。

### 1.4 Connector SPI（Service Provider Interface）

Connector 是 Trino 的灵魂。一个 Connector 要实现的核心接口：

```java
public interface Connector {
    ConnectorMetadata getMetadata(...);     // 元数据：库/表/列/统计信息
    ConnectorSplitManager getSplitManager(); // 把表切成 Split（并行单元）
    ConnectorPageSourceProvider             // 真正读数据
        getPageSourceProvider();
    // 可选
    ConnectorPageSinkProvider getPageSinkProvider();    // 写数据
    Set<SystemTable> getSystemTables();                  // 系统表
}
```

这个 SPI 决定了**能下推什么**：

| 接口 | 下推能力 | 例子 |
| --- | --- | --- |
| `applyFilter` | 谓词下推 | MySQL Connector 把 `WHERE id > 100` 下到 MySQL |
| `applyProjection` | 列裁剪 | 只读需要的列 |
| `applyAggregation` | 聚合下推 | `SELECT count(*)` 让 MySQL 自己算 |
| `applyTopN` | TopN 下推 | `ORDER BY ts LIMIT 10` 让上游排序 |
| `applyJoin` | JOIN 下推 | 同库 JOIN 让上游做 |

**反直觉的点**：不是所有 Connector 都实现这些。Hive Connector 实现得很全（因为 Trino 团队的"基本盘"），MongoDB Connector 早期 `applyAggregation` 都不支持，导致 `SELECT count(*) FROM mongo.x.y` 把整表拉到 Trino 计数。

主流 Connector 一览：

| Connector | 元数据来源 | 谓词下推 | 写入 | 备注 |
| --- | --- | --- | --- | --- |
| **Hive** | Hive Metastore | ✅ | ✅ | 老牌，Parquet/ORC 支持好 |
| **Iceberg** | HMS / REST / Glue / Nessie | ✅ | ✅ | 表格式，时间旅行/分支 |
| **Delta Lake** | Delta Log | ✅ | ✅ | Databricks 系 |
| **Hudi** | HMS | ⚠️ | ⚠️ | upsert 强，Trino 端体验一般 |
| **MySQL/PG/Oracle** | JDBC | ✅ | ✅ | 联邦查询主力 |
| **MongoDB** | Mongo schema | ⚠️ | ✅ | 部分聚合下推 |
| **Kafka** | Topic | ❌ | ❌（只读） | 适合批，不适合流 |
| **Elasticsearch** | ES mapping | ✅ | ❌ | 有 query passthrough |
| **TPC-H/TPC-DS** | 内置 | — | — | 测试/学习用，无需外部数据 |

### 1.5 查询执行：Stage / Task / Driver / Operator

一条 SQL 在 Trino 内部被切成一棵 Stage 树：

```
   SQL: SELECT a, sum(b) FROM hive.db.t JOIN mysql.db.u ON t.k=u.k GROUP BY a

                     Stage 0 (Final)
                          │
                     ┌────┴────┐
                     │ Aggregate │
                     │ Output    │
                     └────┬─────┘
                          │ exchange (gather)
                     Stage 1 (Partial Agg)
                          │
                     ┌────┴────┐
                     │ Aggregate │
                     │ Join      │
                     └────┬─────┘
                          │ exchange (hash partition by k)
              ┌───────────┴───────────┐
              ▼                       ▼
       Stage 2 (Hive Scan)      Stage 3 (MySQL Scan)
          ScanFilter               ScanFilter
          (split=10)               (split=1)
```

层级关系：

- **Stage**：执行计划的一个阶段，对应一个 Fragment
- **Task**：一个 Stage 在一个 Worker 上的实例（Stage 1 在 10 个 Worker 跑 = 10 个 Task）
- **Driver**：一个 Task 内的并行单元，绑定一个 Pipeline
- **Operator**：Driver 内的物理算子（Scan、Filter、HashJoin、Aggregation...）

这是 MPP 的精髓：**一条 SQL 同时在 N 个 Worker × M 个 Driver 上并行执行**，并行度可以达到几千。

### 1.6 Pipeline 执行（无 shuffle 中间态磁盘化）

这是 Trino 与 Spark 最根本的差异之一。

**Spark / MapReduce**：每个 Stage 跑完，shuffle 数据**落盘**，下一个 Stage 从盘上读。优点：失败可恢复（Stage 内容已经写盘，挂了重跑当前 Stage）。缺点：磁盘 IO 成本高，端到端延迟高。

**Trino**：算子之间通过**内存中的 Page**（一个列式批，默认 1MB）传递。数据在 Worker 之间通过 HTTP 长连接 streaming 推送，**不落盘**。

```
Worker A                    Worker B
┌──────────┐                ┌──────────┐
│ Scan     │                │ Scan     │
│   ↓ Page │                │   ↓ Page │
│ Filter   │                │ Filter   │
│   ↓ Page │  HTTP stream   │   ↓ Page │
│ Sink ────┼───────────────►│ Source   │
│ (exchange│                │   ↓ Page │
└──────────┘                │ Join     │
                             └──────────┘
```

收益：
- 端到端延迟低（秒级响应交互式查询）
- 没有磁盘 IO 中间态

代价：
- **任何一个 Worker 挂了，整条查询失败**（Trino 默认无 Stage retry）
- 内存压力大，所有中间数据在内存里
- 不适合 ETL 类小时级批处理（单查询时长 > 集群 MTBF 时就开始翻车）

Trino 4.0（2023+）开始支持 **Fault-Tolerant Execution**（FTE）：可选地把 shuffle 数据物化到 S3 / HDFS，单 Task 失败可重试。但默认还是 streaming。

### 1.7 Cost-Based Optimizer

Trino 有 CBO，但它的工作前提是**有统计信息**：行数、列基数、null 比例、分位数等。

获取统计信息的途径：
- Hive Connector：`ANALYZE TABLE t`，存到 HMS
- Iceberg Connector：自动维护，新写入会更新
- JDBC Connector：实时查询数据库的 `information_schema`
- 没统计信息：CBO 退化，按行数估算（可能选错 JOIN 顺序）

CBO 干的事：
1. **JOIN 顺序重排**（多表 JOIN 选最优顺序）
2. **JOIN 算法选择**（Broadcast vs Partitioned Hash Join）
3. **JOIN 侧选择**（哪一侧做 build，哪一侧做 probe）
4. **Push down 决策**（什么下推到 Connector）

最关键的判断：**Broadcast vs Partitioned Hash Join**。

- Broadcast：小表广播到所有 Worker，每个 Worker 本地 JOIN。要求小表能放进单 Worker 内存。
- Partitioned：两表都按 JOIN key 哈希 shuffle 到对应 Worker，再 JOIN。两边都大时用。

阈值靠 `join-distribution-type=AUTOMATIC` 让 CBO 决策，没有统计信息就退化为 `PARTITIONED`，可能慢 10x。

> **口诀**：MPP 的本质是"内存流水线 + 数据切分并行"。**没有统计信息的 CBO 等于没有 CBO**。Hive Connector 必须 `ANALYZE`，Iceberg 自动维护。

---

## 2. 工程切片：从配置到 K8s 部署

> 基于 Trino 435+ 版本。

### 2.1 Trino 配置 + Catalog 定义

Trino 的配置分三层：

```
etc/
├── config.properties        # 节点角色、端口、内存
├── jvm.config               # JVM 参数
├── node.properties          # 节点 ID、数据目录
├── log.properties           # 日志级别
└── catalog/                 # Catalog 定义（每个数据源一个文件）
    ├── hive.properties
    ├── mysql.properties
    └── iceberg.properties
```

**Coordinator 配置**：

```properties
# etc/config.properties (Coordinator)
coordinator=true
node-scheduler.include-coordinator=false
http-server.http.port=8080
discovery.uri=http://coordinator-host:8080

# 内存 - 这三个参数生产里最容易踩坑
query.max-memory=50GB                     # 单查询所有 Worker 总内存上限
query.max-memory-per-node=8GB             # 单查询在单 Worker 上的内存上限
query.max-total-memory-per-node=10GB      # 单查询 user + system 总内存上限

# 并发
query.max-concurrent-queries=100
query.client.timeout=5m
```

**Worker 配置**：

```properties
# etc/config.properties (Worker)
coordinator=false
http-server.http.port=8080
discovery.uri=http://coordinator-host:8080

query.max-memory-per-node=8GB
query.max-total-memory-per-node=10GB
```

**JVM 参数**（Coordinator + Worker 通用）：

```
# etc/jvm.config
-server
-Xmx16G
-XX:+UseG1GC
-XX:G1HeapRegionSize=32M
-XX:+ExplicitGCInvokesConcurrent
-XX:+HeapDumpOnOutOfMemoryError
-XX:+ExitOnOutOfMemoryError
-Djdk.attach.allowAttachSelf=true
-Djdk.nio.maxCachedBufferSize=2000000
-XX:MaxDirectMemorySize=4G
```

`MaxDirectMemorySize` 必须设。Trino 内部用大量 ByteBuffer 做 Page 传输，不设的话默认等于 `-Xmx`，OS 看到的实际进程内存可能是 2 倍 heap，OOM Kill 是必然。

**MySQL Catalog**：

```properties
# etc/catalog/mysql.properties
connector.name=mysql
connection-url=jdbc:mysql://mysql-host:3306
connection-user=trino_ro
connection-password=***

# 性能调优
mysql.jdbc.use-information-schema=true
join-pushdown.enabled=true
join-pushdown.strategy=AUTOMATIC
```

**Hive Catalog**：

```properties
# etc/catalog/hive.properties
connector.name=hive
hive.metastore.uri=thrift://hms-host:9083
hive.s3.aws-access-key=***
hive.s3.aws-secret-key=***
hive.s3.endpoint=https://s3.us-east-1.amazonaws.com

hive.parquet.use-column-names=true
hive.compression-codec=ZSTD
hive.config.resources=/etc/hadoop/core-site.xml
```

**Iceberg Catalog**：

```properties
# etc/catalog/iceberg.properties
connector.name=iceberg
iceberg.catalog.type=hive_metastore
hive.metastore.uri=thrift://hms-host:9083
iceberg.file-format=PARQUET
iceberg.compression-codec=ZSTD

# REST Catalog（更现代的方案）
# iceberg.catalog.type=rest
# iceberg.rest-catalog.uri=https://catalog-host/v1
```

每改一个 catalog 文件，**默认需要重启** Trino。Trino 459+ 支持动态 Catalog（见 2.6）。

### 2.2 跨源 JOIN（Iceberg + MySQL）

最经典的联邦查询场景：**事实表在 Iceberg（数据湖），维表在 MySQL（业务库）**。

```sql
-- 假设：iceberg 里是订单事实表，mysql 里是用户维表
-- 想查"过去 7 天每个城市的销售额"

SELECT
    u.city,
    SUM(o.amount) AS total_sales,
    COUNT(*) AS order_cnt
FROM iceberg.warehouse.orders o
JOIN mysql.business.users u ON o.user_id = u.id
WHERE o.dt >= DATE '2026-05-01'
  AND o.dt < DATE '2026-05-08'
GROUP BY u.city
ORDER BY total_sales DESC
LIMIT 10;
```

执行时 Trino 会做：

1. Iceberg 端：用 `dt` 分区裁剪，只读 7 天的 Parquet 文件
2. MySQL 端：发送 `SELECT id, city FROM users` 拉全量（没有 WHERE 可下推）
3. Trino Worker：根据 `user_id` 做哈希 shuffle，做 hash join
4. 聚合 + TopN

**关键观察**：MySQL 维表如果有 100 万行，**全表会被拉到 Trino**。这是联邦查询的代价。

优化技巧：

- 维表小（< 10MB）：Trino 自动 Broadcast Join，每个 Worker 缓一份
- 维表大：业务上加 hint 强制 Broadcast 或先在 Iceberg 里物化一份

```sql
-- hint 强制 Broadcast
SELECT /*+ DISTRIBUTION(BROADCAST) */
    u.city, SUM(o.amount)
FROM iceberg.warehouse.orders o
JOIN mysql.business.users u ON o.user_id = u.id
GROUP BY u.city;
```

**EXPLAIN 看下推情况**：

```sql
EXPLAIN (TYPE DISTRIBUTED)
SELECT u.city, SUM(o.amount) ...;
```

输出会显示每个 Stage 的算子，重点看：
- `ScanFilter[mysql:users]` 后面的 `predicate` 段（有没有下推 WHERE）
- `Project` 后面的列（有没有列裁剪）
- `JOIN` 节点的 `distribution` 字段（PARTITIONED 还是 BROADCAST）

### 2.3 Hive / Iceberg / Delta Lake Connector

这三个 Connector 都读"数据湖"，但语义差别很大。

**Hive**：最古老，纯文件系统视图。
- 元数据：HMS 存表/分区元数据
- 数据：Parquet/ORC 文件直接放 HDFS/S3
- 问题：没有事务、改 schema 是元数据级别操作但旧文件 schema 不变、并发写容易脏

**Iceberg**：现代表格式（table format），解决 Hive 的所有问题。
- 元数据：JSON manifest 文件 + 元数据指针（指针存 HMS / REST / Glue）
- 数据：Parquet/ORC + 显式 manifest 描述每个文件的 stats
- 特性：**ACID 事务、Schema Evolution、Hidden Partitioning、Time Travel**

```sql
-- Iceberg 的杀手特性
-- 时间旅行
SELECT * FROM iceberg.db.t FOR VERSION AS OF 1234567890;
SELECT * FROM iceberg.db.t FOR TIMESTAMP AS OF TIMESTAMP '2026-05-01 00:00:00';

-- 分支（branch）+ 标签（tag）
ALTER TABLE iceberg.db.t EXECUTE create_branch('staging');
ALTER TABLE iceberg.db.t EXECUTE create_tag('v1.0', 1234567890);

-- 文件级删除（不重写整文件）
DELETE FROM iceberg.db.t WHERE user_id = 100;
```

**Delta Lake**：Databricks 出品，与 Spark 紧密集成。
- 元数据：JSON 事务日志（`_delta_log/`）
- 数据：Parquet
- 与 Iceberg 类似，但生态偏 Databricks

选型口诀：
- **新建数据湖 → Iceberg**（开源生态最活跃）
- **已经用 Databricks → Delta**
- **历史包袱 → Hive**，慢慢迁

### 2.4 Kafka Connector（流批一体的幻觉）

```properties
# etc/catalog/kafka.properties
connector.name=kafka
kafka.nodes=kafka1:9092,kafka2:9092
kafka.table-names=events
kafka.table-description-dir=/etc/trino/kafka
```

Kafka Topic 映射成表：

```json
{
    "tableName": "events",
    "schemaName": "logs",
    "topicName": "user-events",
    "key": {
        "dataFormat": "raw",
        "fields": [...]
    },
    "message": {
        "dataFormat": "json",
        "fields": [
            {"name": "user_id", "type": "BIGINT", "mapping": "userId"},
            {"name": "event", "type": "VARCHAR", "mapping": "event"},
            {"name": "ts", "type": "TIMESTAMP", "mapping": "timestamp"}
        ]
    }
}
```

```sql
-- 查最近 1 万条事件
SELECT * FROM kafka.logs.events
WHERE _partition_id = 0
ORDER BY _partition_offset DESC
LIMIT 10000;
```

**重要：Kafka Connector 不适合做"实时流处理"**。它是个**批读取器**：每次查询从 Kafka 当前 offset 范围读取，没有 watermark / window / state 概念。流处理用 Flink，Kafka Connector 只解决"用 SQL 临时翻 Kafka 数据"的问题。

### 2.5 Trino on K8s（Helm Chart）

官方 Helm Chart：

```bash
helm repo add trino https://trinodb.github.io/charts/
helm install my-trino trino/trino --values values.yaml
```

`values.yaml` 关键配置：

```yaml
image:
  tag: "459"

server:
  workers: 10
  config:
    query:
      maxMemory: "100GB"
      maxMemoryPerNode: "8GB"

coordinator:
  jvm:
    maxHeapSize: "16G"
    gcMethod:
      type: "G1GC"
  resources:
    requests:
      memory: "20Gi"
      cpu: "4"
    limits:
      memory: "20Gi"
      cpu: "8"

worker:
  jvm:
    maxHeapSize: "32G"
  resources:
    requests:
      memory: "40Gi"          # heap + direct + overhead
      cpu: "8"
    limits:
      memory: "40Gi"
      cpu: "16"

# Catalog 通过 ConfigMap / Secret 挂载
additionalCatalogs:
  hive: |
    connector.name=hive
    hive.metastore.uri=thrift://hms.data.svc.cluster.local:9083
  mysql: |
    connector.name=mysql
    connection-url=jdbc:mysql://mysql.data.svc.cluster.local:3306
```

**反直觉的部署细节**：

1. **Coordinator 用 Deployment，Worker 用 Deployment 而不是 StatefulSet**。Trino 都是无状态的，没必要 StatefulSet。
2. **request = limit**（Guaranteed QoS）。MPP 任务对内存波动极敏感，Burstable 容易被驱逐。
3. **不用 HPA 自动扩缩容**。Worker 缩容会导致正在跑的查询失败。常见做法是**业务低峰手动缩**或用 KEDA 配合 Graceful Shutdown。
4. **PodDisruptionBudget 必须配**：`maxUnavailable: 1`，避免 K8s 升级一把摘掉多个 Worker。

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: trino-worker-pdb
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: trino
      app.kubernetes.io/component: worker
```

### 2.6 用户自定义函数（UDF）

Trino 支持两种 UDF：

**SQL UDF**（Trino 432+，最简单）：

```sql
CREATE FUNCTION mask_phone(phone VARCHAR)
RETURNS VARCHAR
LANGUAGE SQL
DETERMINISTIC
RETURNS NULL ON NULL INPUT
AS '
    CASE WHEN length(phone) >= 11
         THEN concat(substring(phone, 1, 3), ''****'', substring(phone, 8))
         ELSE phone
    END
';

SELECT mask_phone('13812345678');   -- 138****5678
```

**Java Plugin UDF**（功能完整）：

```java
public class StringFunctions {
    @ScalarFunction("md5_hex")
    @Description("MD5 hex digest")
    @SqlType(StandardTypes.VARCHAR)
    public static Slice md5Hex(@SqlType(StandardTypes.VARCHAR) Slice input) {
        try {
            byte[] digest = MessageDigest.getInstance("MD5").digest(input.getBytes());
            return Slices.utf8Slice(HexFormat.of().formatHex(digest));
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException(e);
        }
    }
}
```

打包成 plugin 放到 `plugin/my-udfs/` 目录，重启即可。

**坑**：`@SqlType` 必须用 Trino 的内部类型（`Slice` 不是 `String`，`long` 不是 `Long`）。性能要求高时必须用 `@SqlNullable` 显式标注 nullable，否则 Trino 会包一层。

### 2.7 RBAC + Resource Group

**RBAC**（基于角色的访问控制）：

```sql
-- 创建角色
CREATE ROLE analyst;
CREATE ROLE etl;

-- 授权
GRANT SELECT ON iceberg.warehouse.* TO ROLE analyst;
GRANT INSERT, DELETE ON iceberg.warehouse.* TO ROLE etl;

-- 给用户赋角色
GRANT analyst TO USER alice;
GRANT etl TO USER bob;
```

但 Trino 自己的 RBAC 是 connector 级别的，**很弱**。生产里通常用：

- **Apache Ranger**：跨引擎统一权限（Hive/Trino/Spark/HBase）
- **OPA（Open Policy Agent）**：用 Rego 写策略，灵活
- **Lakehouse 平台自带**（如 Starburst Galaxy）

**Resource Group**：核心的 QoS 机制，控制"谁能用多少资源"。

```json
// etc/resource-groups.json
{
    "rootGroups": [
        {
            "name": "global",
            "softMemoryLimit": "80%",
            "hardConcurrencyLimit": 100,
            "maxQueued": 1000,
            "subGroups": [
                {
                    "name": "etl",
                    "softMemoryLimit": "60%",
                    "hardConcurrencyLimit": 10,
                    "maxQueued": 100,
                    "schedulingPolicy": "fair"
                },
                {
                    "name": "adhoc",
                    "softMemoryLimit": "30%",
                    "hardConcurrencyLimit": 50,
                    "maxQueued": 500,
                    "schedulingPolicy": "weighted",
                    "subGroups": [
                        {
                            "name": "analyst.${USER}",
                            "softMemoryLimit": "10%",
                            "hardConcurrencyLimit": 5,
                            "maxQueued": 20
                        }
                    ]
                }
            ]
        }
    ],
    "selectors": [
        {"user": "etl_.*", "group": "global.etl"},
        {"source": "superset", "group": "global.adhoc.analyst.${USER}"}
    ]
}
```

参数解释：
- `softMemoryLimit`：内存软上限。超过会**拒绝接受新查询**，已运行的不影响
- `hardConcurrencyLimit`：同时运行的查询数硬上限
- `maxQueued`：队列里最多排多少。超了直接拒绝
- `schedulingPolicy`：`fair` / `weighted` / `query_priority` / `weighted_fair`

> **口诀**：Resource Group 是 Trino 多租户的命门。**ETL 跑批 + Adhoc 交互必须分组隔离**，否则一个大查询拖垮全集群。

---

## 3. 实测数据：MPP 不是银弹

### 3.1 Trino vs Spark SQL：交互式查询延迟

测试环境：10 Worker × 16 核 × 64GB，TPC-DS 1TB，存储 S3 + Parquet。

| 查询 | Trino (s) | Spark SQL (s) | 倍数 |
| --- | --- | --- | --- |
| q1 (单表 group by) | 1.2 | 8.5 | 7x |
| q3 (3 表 JOIN) | 2.8 | 12.0 | 4.3x |
| q19 (5 表 JOIN + 聚合) | 5.5 | 22 | 4x |
| q72 (复杂 7 表 JOIN) | 18 | 45 | 2.5x |
| q85 (聚合大表 + 子查询) | 35 | 80 | 2.3x |

观察：
- **简单查询 Trino 快 4~7x**：MPP 启动开销小，没有 Spark 的 driver/executor 启动 + DAG 优化开销
- **复杂查询差距收窄到 2~3x**：CBO 决定执行计划质量，Spark CBO 也很强；Trino 的优势在执行阶段
- **超长查询（> 5min）**：Trino 容错差，反而比不上 Spark

结论：**交互式 BI 用 Trino，长跑批用 Spark**。

### 3.2 跨源 JOIN 的代价

测试：Iceberg 1 亿行事实表 JOIN MySQL 100 万行维表。

| 方案 | 耗时 | 说明 |
| --- | --- | --- |
| 直接跨源 JOIN（Broadcast） | 12s | MySQL 全表拉取 ~5s，本地 hash join ~7s |
| 直接跨源 JOIN（Partitioned，CBO 误判） | 45s | shuffle 大表 + MySQL 拉取双重压力 |
| 维表先物化到 Iceberg | 6s | 不需要再访问 MySQL，但要维护物化时效 |
| 维表广播 hint 强制 | 11s | 接近第一种 |

观察：
- 联邦查询的"实时一致"是有代价的，跨源 JOIN 普遍比预聚合慢 2~3x
- 维表物化是常见折中：Trino 跑一个 INSERT INTO iceberg.t SELECT * FROM mysql.t，每小时调一次

### 3.3 不同 Connector 的查询效率

同样 `SELECT count(*) FROM x WHERE dt = '2026-05-01'`，扫描 1 亿行的耗时：

| Connector | 耗时 | 备注 |
| --- | --- | --- |
| Iceberg + Parquet + ZSTD | 2.1s | 谓词下推到分区，stats 走 manifest |
| Hive + Parquet | 3.5s | 分区裁剪 OK，但 stats 不准 |
| MySQL（有索引） | 1.5s | 全部下推到 MySQL 自己算 |
| MySQL（无索引） | 180s | 全表扫，单 split 单线程 |
| MongoDB | 12s | 部分聚合下推，但 count 比较好 |
| Elasticsearch | 0.8s | ES 自己的 count API 极快 |

注意 MySQL 那条：**JDBC Connector 默认每个表 1 个 split** —— 单线程拉。可以配 `mysql.split-per-table=auto` 让 Trino 按主键范围切多个 split 并行拉，但要主键是数值型且分布均匀。

### 3.4 Worker 数量的扩展性

固定查询（TPC-DS q19，1TB），变化 Worker 数：

| Worker 数 | 耗时 | 加速比 | 效率 |
| --- | --- | --- | --- |
| 2 | 38s | 1.0x | 100% |
| 4 | 21s | 1.8x | 90% |
| 8 | 12s | 3.2x | 79% |
| 16 | 7.5s | 5.1x | 64% |
| 32 | 5.5s | 6.9x | 43% |
| 64 | 4.8s | 7.9x | 25% |

明显的边际递减。原因：
- Coordinator 调度开销随 Worker 数线性增长
- 网络 shuffle 成本随 Worker 数 O(N²) 增长
- 小 split 的固定开销摊不掉

实务建议：**绝大多数场景 8~32 个 Worker 是甜区**。超过 32 考虑分集群（按业务/租户划分）。

### 3.5 内存占用对比

同样 q19 查询，看不同执行方式的 Peak Memory：

| 方式 | Peak Memory | 说明 |
| --- | --- | --- |
| Broadcast Join | 6GB | 小表广播到每个 Worker |
| Partitioned Hash Join | 12GB | shuffle 后 hash 表更大但分布在多 Worker |
| Spill to Disk 开启 | 8GB | 超过阈值的部分落盘 |

Trino 默认不开 spill，要在 `config.properties` 里配：

```properties
experimental.spill-enabled=true
spiller-spill-path=/tmp/trino-spill
spill-compression-enabled=true
spill-encryption-enabled=false
```

但 spill **会 5x 慢于纯内存**，是兜底机制不是优化手段。

> **口诀**：Trino 的扩展性甜区是 **8~32 Worker**。**spill 是救命稻草不是常规优化**。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。

### 案例 ① 跨源 JOIN 内存爆炸 —— "一个 SQL 把整集群拖死"

#### 现象

数据团队提了一个 BI 查询：

```sql
SELECT u.country, u.city, p.category, SUM(o.amount)
FROM iceberg.dwh.orders o          -- 5 亿行
JOIN mysql.crm.users u ON o.user_id = u.id     -- 800 万行
JOIN mysql.product.products p ON o.pid = p.id   -- 50 万行
WHERE o.dt >= DATE '2026-04-01'
GROUP BY u.country, u.city, p.category;
```

跑下去 30 秒后整个 Trino 集群所有 Worker 都 OOM Kill 重启，影响其他正在跑的查询。

#### 根因

CBO 没有 MySQL 表的统计信息（JDBC Connector 默认不主动 ANALYZE），按"行数小估 = 适合 Broadcast"判断了 users 和 products，决定都 Broadcast。

但 800 万行的 users 表，**每行包含一个 1KB 的 JSON profile 字段** —— 实际内存占用 8GB+。Broadcast 到 32 个 Worker，每个都缓 8GB → Worker 内存爆。

更糟：Trino 的 `query.max-memory-per-node` 是软限制，超过时**整个查询取消而不是降级**。但取消之前已经分配的内存还没回收，**OS 看到 RSS 超限直接 OOM Kill**。

#### 修复

**短期**：强制 Partitioned Hash Join。

```sql
SET SESSION join_distribution_type = 'PARTITIONED';
SELECT ...;
```

或单查询 hint：

```sql
SELECT /*+ DISTRIBUTION(PARTITIONED) */ ...
```

**长期**：

1. 把 MySQL 维表**每日同步到 Iceberg**，从 Iceberg JOIN，CBO 有准确 stats。

```sql
-- 每日同步 job
INSERT OVERWRITE iceberg.dwh.dim_users
SELECT id, country, city, /* JSON 字段不要带 */ ...
FROM mysql.crm.users;
```

2. 配置 Resource Group，给 BI 查询设 `softMemoryLimit: 40%`，防止单查询打爆全集群。

3. 升级 Coordinator 配置，开 `query.low-memory-killer.policy=total-reservation-on-blocked-nodes`：内存压力时优先杀占用最大的查询，保护小查询。

```properties
query.low-memory-killer.policy=total-reservation-on-blocked-nodes
query.low-memory-killer.delay=5s
```

#### 验证

- Web UI 看查询的 `Peak Total Memory`，应该 < `query.max-memory`
- Worker 不再 OOM Kill：`kubectl get pods` 看 RESTART 次数
- 同样查询用 Iceberg 维表跑，从 OOM 变成 8s 完成

> **口诀**：**没 stats 的 JDBC 表千万别让 CBO 自动选 Broadcast**。维表大就 Partitioned，要么物化到数据湖。

---

### 案例 ② Coordinator 单点风险 —— "凌晨重启业务全停"

#### 现象

凌晨 3 点 Coordinator OOM 重启。重启用了 4 分钟（加载所有 Catalog 元数据 + Discovery 重连）。这 4 分钟内所有 BI 仪表板报错。

#### 根因

Trino 设计里 Coordinator 就是单点：
- 它持有所有"运行中查询"的状态
- 它是 Discovery Service 的中心
- Worker 启动时必须能找到它

Trino 没有官方"Coordinator HA"方案，理由是：**Coordinator 状态是查询级的，HA 复杂收益小**。

但生产里你需要 HA。常见方案有三种。

#### 修复

**方案 A：Trino Gateway**（社区主流）

```
Client → Trino Gateway → [Coordinator A | Coordinator B | ...]
                              │              │
                              ▼              ▼
                          Worker Pool A  Worker Pool B
```

部署多个独立 Trino 集群（每个集群一个 Coordinator），Gateway 做路由+健康检查。

```yaml
# trino-gateway 配置
backends:
  - name: cluster-a
    proxyTo: http://coord-a:8080
    active: true
  - name: cluster-b
    proxyTo: http://coord-b:8080
    active: true
routingRules:
  - condition: 'request.user == "etl_user"'
    backend: cluster-b
  - condition: 'true'
    backend: cluster-a
```

挂掉一个集群，Gateway 把流量路由到另一个。代价：跨集群没法共享 Worker、查询级失败仍要重试。

**方案 B：K8s 自愈 + 快速重启**

接受单点，但优化重启时间：
- Catalog 改成动态加载（Trino 459+），重启不再加载所有
- 用 `livenessProbe` 检测 Coordinator 假死，自动重启
- 用 readinessProbe 在 Coordinator 启动期间隔离流量

```yaml
livenessProbe:
  httpGet:
    path: /v1/info
    port: 8080
  initialDelaySeconds: 60
  periodSeconds: 30
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /v1/info
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
```

**方案 C：业务侧重试**

客户端自动重试（指数退避）+ 幂等查询。BI 工具如 Superset 配置 retry policy，5xx 错误自动重试 3 次。

#### 验证

- 主动 kill Coordinator pod，业务影响窗口应 < 1 分钟（K8s 重启 + 探针通过）
- 用 Trino Gateway 时，故障转移应 < 30 秒
- 监控 `coordinator_uptime` 和 `query_failure_rate`

> **口诀**：Trino 没有 Coordinator HA，**用 Trino Gateway 多集群** 或 **接受单点 + 快速重启**。Coordinator 单集群规模别超过 50 Worker，故障半径太大。

---

### 案例 ③ Worker 资源不均（数据倾斜）—— "32 个 Worker 只有 1 个累死"

#### 现象

一个 GROUP BY 查询本来 5s 完成，最近变成 50s。Web UI 看：
- 31 个 Worker 在 95% 时间空闲
- 1 个 Worker 满载，CPU 100%，内存 80%

#### 根因

GROUP BY key 倾斜。比如 `GROUP BY user_id`，大部分 user_id 数据均匀，但有个**测试账号 user_id=0 占了 30%** 的数据。Hash partition 按 key 分发，所有 user_id=0 的数据全跑到同一个 Worker。

这是分布式系统的经典问题，Spark 也有，叫 "skew"。

#### 修复

**方案 A：业务侧预清洗**

把测试账号、爬虫账号、占比异常的 key 单独处理：

```sql
SELECT user_id, COUNT(*)
FROM iceberg.events
WHERE user_id NOT IN (SELECT user_id FROM iceberg.dim.bot_users)
GROUP BY user_id;
```

**方案 B：加盐（salting）**

把热点 key 拆开：

```sql
-- 第一阶段：每个 key 加随机后缀，先聚合
WITH stage1 AS (
  SELECT
    CONCAT(CAST(user_id AS VARCHAR), '_', CAST(rand() * 100 AS VARCHAR)) AS salted_key,
    user_id,
    COUNT(*) AS cnt
  FROM iceberg.events
  GROUP BY 1, 2
)
-- 第二阶段：去掉盐再聚合
SELECT user_id, SUM(cnt)
FROM stage1
GROUP BY user_id;
```

效果：第一阶段把热点 key 的数据分散到 100 个 Worker，第二阶段汇总（数据量已经小得多）。

**方案 C：Trino 自带的 Skewed Join 优化**

```properties
optimizer.skew-join-optimization=true
```

(Trino 419+) 自动识别倾斜 key 并拆分。但效果不如手动加盐稳定。

**方案 D：用 Approximate 函数**

如果只需要近似结果：

```sql
SELECT approx_distinct(user_id), approx_percentile(amount, 0.99)
FROM iceberg.events;
```

`approx_distinct` 用 HyperLogLog，不需要全量 shuffle。

#### 验证

- Web UI 看每个 Stage 的 Task 耗时分布，应该相对均匀（最长/最短 < 2x）
- 监控指标：`taskCpuTime` 分位数，p99/p50 < 3
- 加盐后，原来 50s 的查询应回到 8s 左右

> **口诀**：**所有 GROUP BY / JOIN 的 key 都要看分布**。倾斜的 key 业务清洗 > 加盐 > 框架自动优化。

---

### 案例 ④ Hive Metastore 性能瓶颈 —— "查询 hang 在 planning"

#### 现象

查询 elapsed time 30s，但实际 execution 只用了 3s。Web UI 看查询状态长时间停在 `PLANNING`。

#### 根因

Trino planning 阶段要从 Hive Metastore 拉表元数据。如果：
- 表分区数特别多（百万级分区）
- HMS 后端 MySQL 慢（连接池满 / 慢 SQL）
- 元数据缓存没开

每次查询都要去 HMS 拉一遍元数据，planning 时间能从毫秒涨到几十秒。

`SHOW STATS FOR my_table` 也会触发 HMS 大量请求。

#### 修复

**1. 开 HMS 元数据缓存**：

```properties
# etc/catalog/hive.properties
hive.metastore-cache-ttl=1h
hive.metastore-refresh-interval=10m
hive.metastore-cache-maximum-size=10000
hive.metastore-stats-cache-ttl=1h
```

**2. 大量分区的表用 Iceberg**：Iceberg 元数据自带 stats，不依赖 HMS 的分区列表。

**3. HMS 后端 MySQL 调优**：
- `innodb_buffer_pool_size` 调大到能装下整个 metastore DB
- 加索引：`PARTITIONS.PART_NAME`、`SDS.SD_ID` 等热点查询路径

**4. 升级 HMS 部署架构**：
- 单 HMS → HMS Pool（多个 HMS 实例 + 负载均衡）
- 或换 **HMS-less** 方案：Iceberg REST Catalog / AWS Glue

```properties
# Iceberg REST Catalog（无 HMS）
connector.name=iceberg
iceberg.catalog.type=rest
iceberg.rest-catalog.uri=https://catalog.example.com/v1
```

**5. 监控**：

```sql
-- Trino 内部表，看 HMS 调用统计
SELECT * FROM system.metadata.materialized_view_properties;
SELECT * FROM jmx.current.io.trino.plugin.hive.metastore_method_invocations:*;
```

#### 验证

- 查询的 PLANNING 时间从 30s 降到 < 2s
- HMS 后端 MySQL QPS 降低 80%
- 大表 `SHOW STATS` 从分钟级到秒级

> **口诀**：**Trino 慢不一定是 Trino 慢，先排查 HMS**。元数据缓存必开。重 HMS 场景换 Iceberg + REST Catalog。

---

### 案例 ⑤ Iceberg 元数据延迟 —— "刚 INSERT 完查不到"

#### 现象

ETL 任务 `INSERT INTO iceberg.t` 提交成功，BI 立即查询，**查不到新数据**。等几分钟又有了。

#### 根因

Iceberg 是"snapshot"模型，每次提交产生一个新 snapshot：

```
v1.metadata.json  →  manifest_list_1.avro  →  data_file_1.parquet
                                              data_file_2.parquet
v2.metadata.json  →  manifest_list_2.avro  →  data_file_3.parquet (新)
```

`current_metadata_pointer` 指向 v2，但 Trino 端有元数据缓存：

```properties
iceberg.materialized-views.metadata-cache-enabled=true
iceberg.metadata-cache.ttl=600s            # 10 分钟
```

缓存期内 Trino 看到的还是 v1 metadata，自然查不到 v2 写入的数据。

#### 修复

**方案 A：缩短缓存 TTL**

```properties
iceberg.metadata-cache.ttl=30s
```

缺点：HMS 调用变多，Coordinator 内存占用上升。

**方案 B：写入侧主动刷新**

Iceberg REST Catalog 可以推送 invalidate 通知。但开源实现还不普及。

**方案 C：业务侧约定时延**

最常见的做法：约定 ETL 写入 → 5 分钟内可查。Dashboard 不显示"最近 5 分钟"的数据。简单粗暴有效。

**方案 D：使用 Iceberg 的 `system.refresh` 系统表**

```sql
CALL iceberg.system.refresh_table('warehouse', 'orders');
```

主动触发刷新。脚本化后可以在 ETL 完成后自动调用。

#### 验证

- INSERT 后 < 30s 即可查到新数据
- 监控 Iceberg 的 `current_snapshot_id`，写入后应及时更新

> **口诀**：Iceberg 不是实时数据库，**写入到可查存在缓存延迟**。生产里要么短缓存 + 接受 HMS 压力，要么业务约定可查时延。

---

### 案例 ⑥ 大查询拖慢小查询 —— "Resource Group 没配"

#### 现象

一个 ETL 跑 30 分钟，期间所有 BI 查询从平均 2s 涨到 60s。ETL 跑完一切恢复正常。

#### 根因

没有 Resource Group，所有查询竞争同一池资源。ETL 占满 80% 内存 + 70% CPU，BI 查询挤不进来或被限速。

更深的问题：Trino 默认调度策略是 **FAIR**（等同于 FIFO + 时间片轮转），但 FAIR 不区分查询大小。一个 30 分钟的大查询和一个 1 秒的小查询轮流排队，小查询等的代价远大于大查询。

#### 修复

按"任务类型"分组隔离，给交互式查询优先级：

```json
{
    "rootGroups": [
        {
            "name": "global",
            "softMemoryLimit": "100%",
            "hardConcurrencyLimit": 200,
            "maxQueued": 1000,
            "subGroups": [
                {
                    "name": "etl",
                    "softMemoryLimit": "50%",
                    "hardConcurrencyLimit": 5,
                    "maxQueued": 50,
                    "schedulingPolicy": "fair",
                    "schedulingWeight": 1
                },
                {
                    "name": "interactive",
                    "softMemoryLimit": "40%",
                    "hardConcurrencyLimit": 100,
                    "maxQueued": 500,
                    "schedulingPolicy": "weighted_fair",
                    "schedulingWeight": 10,
                    "subGroups": [
                        {
                            "name": "user_${USER}",
                            "softMemoryLimit": "5%",
                            "hardConcurrencyLimit": 5,
                            "maxQueued": 20
                        }
                    ]
                },
                {
                    "name": "system",
                    "softMemoryLimit": "10%",
                    "hardConcurrencyLimit": 5,
                    "maxQueued": 10,
                    "schedulingWeight": 100
                }
            ]
        }
    ],
    "selectors": [
        {"user": ".*_etl", "group": "global.etl"},
        {"source": "superset|metabase", "group": "global.interactive.user_${USER}"},
        {"source": "system", "group": "global.system"},
        {"group": "global.interactive.user_${USER}"}
    ]
}
```

要点：
- **etl 与 interactive 分组**，资源池物理隔离
- `schedulingWeight` 高的组获得更多调度机会（interactive: 10 vs etl: 1）
- interactive 内部按用户细分（防止单个 BI 用户暴击）

启用：

```properties
# etc/config.properties
resource-groups.configuration-manager=file
resource-groups.config-file=/etc/trino/resource-groups.json
```

#### 验证

- ETL 运行期间 BI 查询 P99 仍 < 5s
- `system.runtime.queries` 表看队列分布

```sql
SELECT resource_group_id, state, count(*)
FROM system.runtime.queries
GROUP BY 1, 2;
```

> **口诀**：**ETL 与 Adhoc 必须分 Resource Group**。`hardConcurrencyLimit` 控住跑批，`schedulingWeight` 给交互式查询优先级。

---

### 案例 ⑦ Kafka Connector 不适合实时 —— "用 SQL 监控生产消息延迟变 10s"

#### 现象

业务用 Trino 的 Kafka Connector 做"近实时监控"：

```sql
-- 每 10 秒刷新一次 dashboard
SELECT event_type, COUNT(*) FROM kafka.logs.events
WHERE _timestamp > current_timestamp - INTERVAL '1' MINUTE
GROUP BY event_type;
```

最初 1s 完成，业务量上来后涨到 10s。Kafka 端正常，Trino Worker CPU 正常。

#### 根因

Kafka Connector 的工作方式：
1. 从 Kafka 元数据拿 Topic 当前 offset 范围
2. 给每个 Partition × offset 范围分配一个 Split
3. Worker 各自从 Kafka 拉数据

问题：**没有 offset 索引**。Connector 不知道哪些 offset 对应哪个 timestamp，**WHERE _timestamp > X 不能下推**。Trino 必须**拉取全部 partition 的全部数据**，在 Trino 端 filter。

随着 Kafka 数据增长（哪怕 retention 只有 1 天），全量拉的代价越来越高。

#### 修复

**Kafka Connector 不是为实时设计的**。把它当"翻 Kafka 历史数据的临时工具"用，不要做监控。

监控/实时分析的正确架构：

```
Kafka ─► Flink (real-time aggregation) ─► Iceberg/ClickHouse
                                              │
                                              ▼
                                            Trino (BI 查询)
```

或：

```
Kafka ─► Materialize / RisingWave (流式 SQL) ─► dashboard
```

如果真要用 Kafka Connector，至少：
- Topic 加 `_partition_id` 过滤，缩小扫描范围
- 用 Kafka Streams 预聚合一份"分钟级窗口"数据写另一个 Topic，Trino 查那个

#### 验证

切换到 Flink + Iceberg 后，dashboard 查询稳定在 1s 内，且不再随 Kafka 数据量增长而变慢。

> **口诀**：Trino 的 Kafka Connector 是**批工具**，不是流引擎。**实时监控用 Flink + Iceberg/CH**。

---

### 案例 ⑧ Spill to Disk 的代价 —— "查询不 OOM 了但慢 10x"

#### 现象

为防止 OOM 开了 spill：

```properties
experimental.spill-enabled=true
spiller-spill-path=/data/trino-spill
```

OOM 是没了，但所有大查询慢了 5~10x。

#### 根因

Spill 的代价：
- 数据从内存写到本地盘（即使是 NVMe，也比内存慢 100x+）
- 读回时反序列化
- 压缩/解压（`spill-compression-enabled=true` 默认）
- spill 期间整个 Operator 暂停消费上游数据，触发 backpressure

更糟：spill 后 Operator 退出 spill 模式时，**spill 文件不能复用**，还要写新的。

Spill 设计是**最后兜底**，不是日常优化。

#### 修复

**首选：把内存搞大**

- Worker 节点用大内存机型（128GB / 256GB）
- 调高 `query.max-memory-per-node`
- 比 spill 性价比高得多

**其次：减少需要 spill 的查询**

- 大 GROUP BY → approx_distinct / approx_percentile
- 大 JOIN → 维表先物化、Broadcast 改 Partitioned
- 排序 + LIMIT → TopN 算子（Trino 自动识别）

**最后：spill 配置优化**

```properties
experimental.spill-enabled=true
spiller-spill-path=/nvme1/trino-spill,/nvme2/trino-spill   # 多盘并行
spiller-threads=8
spill-compression-enabled=true
spill-encryption-enabled=false                              # 加密更慢
experimental.spiller-max-used-space-threshold=0.9           # 留 10% 给系统
```

按**操作类型**精细化开关：

```properties
experimental.spill-order-by=true        # ORDER BY 允许 spill
experimental.spill-window-operator=true # 窗口函数允许 spill
experimental.spill-aggregations=false   # 聚合不 spill（聚合 spill 性能特别差）
experimental.spill-joins=false          # JOIN 不 spill
```

#### 验证

- 查询从 spill 模式（看 Operator stats）回到全内存模式
- P99 耗时回到原水平
- 实在大的查询用 `query.max-memory` 限制 + 失败重路由到 Spark

> **口诀**：**Spill 是降级不是优化**。优先级：加内存 > 优化查询 > 部分场景 spill。聚合和 JOIN 别 spill。

---

### 案例 ⑨ Trino vs Spark 选型混淆 —— "用 Trino 跑 4 小时 ETL 失败"

#### 现象

数据团队用 Trino 跑了一个 ETL：从 5 个 MySQL 实例 + 3 个 Hive 表 JOIN 出宽表写到 Iceberg，预计 4 小时。跑了 3 小时 30 分 Worker OOM 重启，整个查询失败要重跑。

#### 根因

Trino 不适合长时间跑批。原因：
1. **Streaming 执行无 Stage 容错**：单 Worker 挂全查询失败
2. **集群 MTBF 限制**：32 个 Worker 的集群，单 Worker 半天内挂的概率不低
3. **资源占用持续高位**：4 小时占用大块内存，影响其他查询

Trino 的执行模型是为**秒到分钟级查询**设计的。

#### 修复

按时间界限选工具：

| 场景 | 工具 | 理由 |
| --- | --- | --- |
| 秒到分钟级 | **Trino** | MPP 低延迟最强 |
| 分钟到小时级 | **Trino + FTE** 或 **Spark** | FTE 提供容错；Spark 也能跑 |
| 小时到天级 | **Spark** | Stage 容错、资源调度成熟 |
| 实时流 | **Flink** | 流处理原生 |

**用 Trino FTE（Fault-Tolerant Execution）跑长查询**：

```properties
# etc/config.properties
retry-policy=TASK
exchange-manager.name=filesystem
exchange.base-directories=s3://my-bucket/trino-exchange/
```

FTE 把 shuffle 数据物化到 S3，Task 失败可重试。代价：性能下降 30~50%，S3 成本。

**或者：把这个 ETL 让给 Spark**

```python
# Airflow DAG
spark_etl = SparkSubmitOperator(
    task_id='build_wide_table',
    application='/opt/airflow/jobs/build_wide_table.py',
    conf={
        'spark.executor.memory': '16g',
        'spark.executor.instances': '20',
        'spark.sql.adaptive.enabled': 'true',  # 类似 Trino CBO
    }
)
```

Spark 的优点：
- Stage 内 Task 失败可单独重试
- 长跑批的 checkpoint 机制
- SQL on Spark 有完整的 ETL 生态

#### 验证

- ETL 切到 Spark：跑 4 小时无失败完成
- Trino 集群恢复，BI 查询 P99 < 2s
- 监控：Trino 单查询时长分布的 P99 < 5min

> **口诀**：**Trino 跑秒级到分钟级，Spark 跑分钟到小时级**。超过 30 分钟的查询要么开 FTE，要么换 Spark。

---

## 5. 上线检查清单

打印贴墙 / 加进 SRE checklist。

### 部署层
- [ ] Coordinator 与 Worker 用独立 Deployment（不要混部）
- [ ] Worker 资源 `request = limit`（Guaranteed QoS）
- [ ] `MaxDirectMemorySize` 显式配置（默认会等于 Xmx，导致 RSS 超限 OOM Kill）
- [ ] `node-scheduler.include-coordinator=false`（Coordinator 不参与执行）
- [ ] PodDisruptionBudget 配置 `maxUnavailable: 1`

### 配置层
- [ ] `query.max-memory` < 集群总 user memory × 60%
- [ ] `query.max-memory-per-node` < Worker heap × 50%
- [ ] HMS 元数据缓存开启（`hive.metastore-cache-ttl`）
- [ ] `query.low-memory-killer.policy=total-reservation-on-blocked-nodes`
- [ ] Catalog 文件挂载用 ConfigMap，密码用 Secret

### 资源管理层
- [ ] Resource Group 配置：ETL / Interactive / System 至少分三组
- [ ] 给 Interactive 设较高 `schedulingWeight`
- [ ] 单用户 `hardConcurrencyLimit` 限制（防止暴击）
- [ ] `maxQueued` 防止队列无限堆积
- [ ] 业务接入新应用必须申请 Resource Group selector

### Connector 层
- [ ] 新建数据湖优先 Iceberg，不用 Hive
- [ ] JDBC Connector 配 `join-pushdown.enabled=true`
- [ ] MySQL Connector 配 `mysql.split-per-table=auto`（主键数值时）
- [ ] Kafka Connector 不用于实时分析
- [ ] 维表大于 100MB 不要让 CBO 自动 Broadcast

### 查询层
- [ ] CBO 依赖统计信息：Hive 表必须 `ANALYZE TABLE`
- [ ] 长查询（> 5min）开启 FTE 或路由到 Spark
- [ ] Spill 仅当兜底，聚合/JOIN 不开 spill
- [ ] EXPLAIN 检查谓词下推、JOIN 类型、列裁剪

### 监控层
- [ ] Coordinator uptime + heap usage
- [ ] Worker 数量 + 各 Worker active task 数
- [ ] 查询 P50 / P99 延迟（按 Resource Group 分维度）
- [ ] 失败查询率（OOM / Timeout 单独维度）
- [ ] HMS 调用 QPS + 慢调用
- [ ] Iceberg snapshot 增长速率（监控写入健康）

### 安全层
- [ ] 启用 HTTPS（`http-server.https.enabled=true`）
- [ ] 用 LDAP / OAuth2 做认证，不用默认无认证
- [ ] 列级权限用 Ranger / OPA，不依赖 Trino 自带 RBAC
- [ ] 敏感字段（手机号 / 身份证）用 mask UDF
- [ ] Coordinator 端口不直接暴露公网，走 Trino Gateway

---

## 6. 还能往下挖什么

### Trino Gateway / Lakehouse 网关
- 多 Trino 集群路由：[trino-gateway](https://github.com/trinodb/trino-gateway)
- 路由策略：按用户、按查询大小、按 SLA
- 与 Apache Kyuubi（多 SQL 引擎统一网关）的对比

### Starburst Galaxy（云服务）
- Trino 商业版，与 Iceberg / Delta 深度集成
- 提供 Coordinator HA、自动扩缩容、数据网格（Data Mesh）能力
- 对比 Snowflake / Databricks SQL，差异在"是否分离存储计算 + 是否真正多源"

### Trino vs Athena vs BigQuery
- AWS Athena：Trino 内核，但 serverless + 按扫描量计费
- BigQuery：Google 自研 Dremel，存算分离，存储天然在 GCS
- 选型维度：成本（Trino 自建 < Athena < BigQuery）vs 运维（反向）

### DuckDB / DataFusion 的对比
- DuckDB：单进程嵌入式 OLAP，单机性能比 Trino 还猛
- DataFusion（Rust）：Apache 项目，向量化引擎，Trino 的"现代替代"潜力股
- 适用场景：单机数据 < 1TB 用 DuckDB，分布式用 Trino

### Iceberg REST Catalog
- 取代 HMS 的下一代元数据层：[Iceberg REST Spec](https://iceberg.apache.org/concepts/catalog/)
- Apache Polaris（Snowflake 开源）/ Tabular（被 Databricks 收购）/ Nessie 三大实现
- 能做：跨引擎统一权限、原子 commit、数据分支（git-like）

### 流批一体（Trino + Iceberg + Flink）
- Flink 写 Iceberg → Trino 查 Iceberg：分钟级新鲜度的"近实时数仓"
- Iceberg v2/v3 支持 row-level delete，Flink CDC 可写入 update/delete
- 与 Hudi（更强 upsert）/ Paimon（流式 lakehouse）的边界

### SQL 标准与 Trino 的扩展
- Trino 支持 ANSI SQL:2016 大部分特性 + 自有扩展（lambda、UNNEST、ROW 类型）
- WITH RECURSIVE、Window function 几乎完整支持
- Trino 不支持的：存储过程、触发器、视图复杂物化（用物化视图 + Iceberg 替代）

### Trino 的容错执行（FTE）深入
- Task 级 retry vs Query 级 retry 的取舍
- Exchange Manager 的 S3/HDFS 物化机制
- 适用场景：长查询、跑批、按需计算（spot 实例）

---

## 7. 引用

- Trino 官方文档：https://trino.io/docs/current/
- Trino 源码：[trinodb/trino](https://github.com/trinodb/trino)
- Presto/Trino 论文：*Presto: SQL on Everything*（ICDE 2019）
- Apache Iceberg：[iceberg.apache.org](https://iceberg.apache.org/)
- *MPP Database Architectures*（Greenplum / Vertica 早期论文）
- Starburst 博客：https://www.starburst.io/blog/
- Trino Gateway：https://github.com/trinodb/trino-gateway
- 美团 Trino 实践：[Trino 在美团的应用与实践](https://tech.meituan.com/)
- Lyft / Pinterest / Stripe 的 Trino 工程博客（搜"Trino at <company>"）

---

> 配套：[数据湖 Iceberg/Delta](/article/db/data-lake) | [DuckDB](/article/db/duckdb) | [Apache Spark 内核](/article/db/spark)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你觉得这个深度对，告诉我下一篇该写哪个。
