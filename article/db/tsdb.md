# 时序数据库深入 —— InfluxDB / TimescaleDB / VictoriaMetrics / Prometheus

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想速记选型，看 [DB 综述](/article/db/) 那一节够了。
> 这里要回答的，是**写入打满、查询超时、磁盘膨胀、监控本身被监控搞挂**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. 同样写 100 万 metric/s，为什么 InfluxDB 内存占 16GB，VictoriaMetrics 只占 2GB？两者用的索引结构差别在哪？
2. Prometheus 一个 block 是 2 小时 —— 这个 2 小时是怎么算出来的？短一点（10 分钟）或长一点（1 天）会怎样？
3. TimescaleDB 的 hypertable 切了上千个 chunk，单条简单查询从 50ms 变 5s —— 谁在背锅？怎么修？
4. 你给一个新 metric 加了个 `user_id` label，第二天监控告警"InfluxDB 内存爆了" —— 描述这条因果链，包含具体数据结构名字。
5. 同一份 1 年的样本数据，原始 16 字节/点，VictoriaMetrics 实际占多少字节/点？InfluxDB v2 占多少？背后是哪些压缩算法在叠加？
6. InfluxDB 的 Continuous Query 和 TimescaleDB 的 Continuous Aggregate，**两者在"何时算"和"算什么"上的本质区别**是什么？哪个更容易"拖慢主集群"？
7. 你想存"3 年的 1 分钟原始数据 + 永久的 1 小时降采样" —— 用 Prometheus 怎么做？用 VictoriaMetrics 怎么做？两者的存储成本差异为什么有 5~10 倍？
8. 时序数据 vs 日志数据 vs 关系数据 —— 边界在哪？同一份"用户行为"数据，什么场景适合塞 ClickHouse 而不是 InfluxDB？

如果有半数答不上，往下看。

---

## 1. 底层模型：时序数据为什么要单独造轮子

绝大多数文章把时序数据库讲成"时间戳 + value"的列存，然后画一个 LSM 树就结束。这层认知不够回答上面 8 个问题。要看懂它们，必须从**数据本身的物理特征**开始。

### 1.1 时序数据的四个特征

| 特征 | 含义 | 对存储的要求 |
| --- | --- | --- |
| **追加为主** | 99% 是 INSERT，几乎不 UPDATE | 写优化（LSM 而非 B+ Tree） |
| **按时间查** | 90% 查询带 `WHERE time > X` | 时间分区（Partition by Time） |
| **聚合为主** | `avg / sum / rate / max` 远多于点查 | 列存（cache 友好） |
| **可衰减** | 老数据精度可降低（1s → 1m → 1h） | 降采样 + 分级存储 |

通用关系数据库（MySQL）撑不住这种负载。给你一张 30 天的监控表，单表 100 亿行，B+ Tree 索引就要几十 GB；range scan 还要回表。专用 TSDB 把这四个特征都吃进设计里。

### 1.2 索引：倒排（Inverted Index）与高基数挑战

时序数据有个核心抽象 —— **series**（序列）：

```
metric_name + {label1=v1, label2=v2, ...} → 一条独立的时间序列
```

例如：

```
http_requests_total{method=GET, path=/api/user, status=200, host=web01}
```

每个 series 是一组 (timestamp, value) 点。**series 的总数叫 cardinality（基数）**，是时序数据库性能的命门。

为什么？因为查询往往是 `sum by (path)` 这种"按某个 label 聚合"，需要先**根据 label 找出符合条件的所有 series**，然后再取这些 series 的样本。这个"按 label 找 series"的操作，靠的是**倒排索引**：

```
方向 1（正排）：series_id → metric_name + labels
方向 2（倒排）：每个 label 值 → 它包含的 series_id 集合（postings list）
```

例：
```
postings["status=200"]    = [series_1, series_5, series_42, ...]
postings["path=/api/user"] = [series_5, series_99, series_42, ...]
```

求"status=200 AND path=/api/user" → 两个 list 求**交集**。

> **高基数灾难**：如果你给 metric 加了 `user_id` 这种 label，每个用户 = 一条新 series。1000 万用户 = 1000 万 series。倒排索引的 key 数量爆炸 → 内存撑不住 → 查询交集时 list 大到爆。这是案例 ① 的根。

### 1.3 列式存储 + 时间分区

时序数据进入物理存储时按"列 + 时间块"组织：

```
Block (e.g. 2 小时)
├── timestamps[]       → 用 delta-of-delta 编码
├── values[]           → 用 Gorilla XOR / DoubleDelta 压缩
└── series_id          → 这一列属于哪条 series
```

为什么列存？因为聚合查询读"某一列的所有值"，而不是"某一行的所有列"。列存让 IO 只读需要的列，且同列数据类型一致 → 压缩比远高于行存。

时间分区的作用：
- 查询 `WHERE time > now()-1h` 直接跳过 `now()-2h` 之前的所有 block
- 老 block 可以独立压缩、归档、删除（删一个目录就完事，不用 DELETE）
- block 内的数据按时间有序，二分查找定位区间起点

### 1.4 降采样（Downsampling）与保留策略（Retention）

原始 1s 精度的数据，存满 1 个月就要几个 TB。但"3 个月前的 CPU 使用率"你要的精度是多少？很可能 1 小时一个点就够了。

降采样：把高精度数据按窗口聚合（avg / max / p99），写到一个粗粒度表里：

```
原始：    1s 精度，保留 7 天
降采样 1：1m 精度，保留 30 天
降采样 2：1h 精度，保留 1 年
降采样 3：1d 精度，保留永久
```

存储成本几乎线性下降：1s 精度 1 周 vs 1d 精度 1 年，前者数据点是后者的 ~17000 倍。

各家方案差别：
- **Prometheus**：原生不支持降采样，要靠 Recording Rules 自己写规则
- **InfluxDB**：Continuous Query（v1） / Tasks（v2） / 自动 Compaction（v3）
- **TimescaleDB**：Continuous Aggregate + 增量刷新
- **VictoriaMetrics**：商业版有 downsampling，开源版没有
- **Thanos / Cortex**：对接 Prometheus 之后做降采样（5m + 1h）

### 1.5 Continuous Query / Continuous Aggregate

降采样不能让用户每次查询时算 —— 必须**预计算**。两种做法：

**InfluxDB Continuous Query (v1)**：定期跑一个 SQL，把 5 分钟前的数据聚合写到新表。
```sql
CREATE CONTINUOUS QUERY cq_5m ON mydb
BEGIN
  SELECT mean(value) INTO downsampled.5m FROM cpu
  GROUP BY time(5m), host
END
```

问题：每跑一次都是一个**全表扫描** + 写入。在主库上跑，**会和写入争 IO**。

**TimescaleDB Continuous Aggregate**：增量物化视图，只算"新进来的数据"。
```sql
CREATE MATERIALIZED VIEW cpu_5m
WITH (timescaledb.continuous) AS
SELECT time_bucket('5m', time) AS bucket, host, AVG(value)
FROM cpu GROUP BY bucket, host;

SELECT add_continuous_aggregate_policy('cpu_5m',
  start_offset => INTERVAL '1 hour',
  end_offset   => INTERVAL '5 minutes',
  schedule_interval => INTERVAL '5 minutes');
```

只重算"上次刷新之后新写入的桶"，开销小一个数量级。这是后面案例 ⑥ 的修复方向。

### 1.6 写入路径：WAL → MemTable → SSTable

几乎所有 TSDB 的写入路径都长成这样（LSM 风格）：

```
Client write
   │
   ▼
┌──────────────────┐
│  WAL (Write-Ahead Log)  │   ← 顺序写盘，保证 crash 不丢
└─────────┬────────┘
          ▼
┌──────────────────┐
│  MemTable        │   ← 内存中，按 series 组织
└─────────┬────────┘
          ▼ (满了或定时 flush)
┌──────────────────┐
│  SSTable / Block │   ← 不可变，列式压缩
└─────────┬────────┘
          ▼ (后台)
┌──────────────────┐
│  Compaction      │   ← 合并小文件，去重
└──────────────────┘
```

差异点：
- **Prometheus**：MemTable 叫 "Head Block"，2 小时一切片，落盘成 block
- **InfluxDB v1**：用 TSM File（Time-Structured Merge Tree），更像 LSM
- **InfluxDB v3 (IOx)**：直接用 Apache Arrow + Parquet + 对象存储
- **TimescaleDB**：基于 PG 的 WAL，Hypertable 切 chunk
- **VictoriaMetrics**：MergeTree 风格（致敬 ClickHouse），写入 → parts → merge

### 1.7 五大方案对比（一张表看清边界）

| 维度 | Prometheus | InfluxDB v2 | InfluxDB v3 (IOx) | TimescaleDB | VictoriaMetrics |
| --- | --- | --- | --- | --- | --- |
| **底层** | 自研 TSDB | TSM | Arrow + Parquet | PG + Hypertable | MergeTree-like |
| **查询语言** | PromQL | Flux / InfluxQL | SQL (DataFusion) | SQL | PromQL + MetricsQL |
| **降采样** | Recording Rules | Tasks | 自动 | Continuous Aggregate | 商业版 |
| **集群** | 单机（靠 Thanos/Cortex） | OSS 单机 / Cloud | Cloud-native | 多节点（社区版有限） | 原生支持 |
| **保留** | 默认 15 天 | 灵活 | 灵活 + S3 分层 | 灵活 + 压缩 | 灵活 |
| **基数容忍** | 中（million 级降速） | 中 | 高 | 高 | **极高** |
| **典型场景** | K8s 监控 | IoT / 通用 | 云原生 + 大规模 | 业务时序 + 关系数据混查 | 大规模指标 |
| **生态** | Grafana 一等公民 | Telegraf / Chronograf | Grafana | PG 生态全用 | Grafana 一等公民 |

> **口诀**：**指标监控选 Prometheus / VictoriaMetrics；业务时序混合关系查询选 TimescaleDB；IoT / 通用 telemetry 选 InfluxDB；超大规模高基数选 VictoriaMetrics**。这是粗略指南，每条有反例。

---

## 2. 五种主流方案切片

> 每段挑核心代码 / SQL，配逐段点透。版本明确写在小节头。

### 2.1 InfluxDB v2 / v3 —— 从 TSM 到 Arrow

InfluxDB v1/v2 的存储层叫 **TSM (Time-Structured Merge Tree)**，文件结构：

```
shard/
  ├── 000000001-000000001.tsm    # 数据文件（不可变）
  ├── 000000001-000000001.tsi    # 倒排索引
  └── _wal/
        └── _00001.wal           # WAL
```

TSM 文件内部分块：

```
┌─────────────┬─────────────┬───────────┬──────────┐
│  Header     │  Blocks     │  Index    │  Footer  │
└─────────────┴─────────────┴───────────┴──────────┘
                  │
                  ▼
            ┌──────────────────────────┐
            │  series_key + timestamps │   ← Delta-of-Delta
            │  + values                │   ← Gorilla XOR / Snappy
            └──────────────────────────┘
```

InfluxDB v3 (IOx) 是个完全重写的版本，关键变化：

- **存储**：从 TSM 换成 **Apache Parquet**（列式 + 行组）
- **执行引擎**：Apache Arrow + DataFusion（向量化执行）
- **查询语言**：原生 SQL（不再推 Flux）
- **存算分离**：Parquet 文件直接放 S3

最小化的写入路径（v3 简化）：

```rust
// influxdb_iox/ingester
async fn handle_write(req: WriteRequest) -> Result<()> {
    let lp = parse_line_protocol(&req.body)?;
    // ① 解析成内存表（Arrow RecordBatch）
    let batch = lp_to_record_batch(lp)?;
    // ② 写 WAL（Parquet 之前的耐久层）
    wal.append(&batch).await?;
    // ③ 进 ingester 内存
    in_memory.merge(batch);
    // ④ 满了或定时，flush 成 Parquet 上 S3
    if in_memory.size() > threshold {
        let parquet = in_memory.to_parquet();
        object_store.put(&parquet).await?;
    }
}
```

**逐段点透**：

- **为什么换 Parquet？** Parquet 是事实上的湖仓标准。换它意味着 InfluxDB 数据可以被任何 Spark / Trino / DuckDB 直接查，**降低锁定**。
- **为什么 DataFusion？** Rust 写的 SQL 引擎，向量化执行 + Arrow 内存格式，性能逼近 ClickHouse；同时不用自己维护 SQL parser。
- **代价**：v3 不兼容 v1/v2 的 Flux 和 InfluxQL（v3 只支持 SQL 子集）。**v1 → v3 是大手术**，案例 ④ 展开。

### 2.2 TimescaleDB —— Hypertable 与 Chunk

TimescaleDB 是 PostgreSQL 的扩展（不是 fork）。核心抽象是 **Hypertable**：对用户像普通表，对引擎是按时间切的一堆 chunk。

```sql
-- 建表（普通 PG 语法）
CREATE TABLE metrics (
  time  TIMESTAMPTZ NOT NULL,
  host  TEXT NOT NULL,
  cpu   DOUBLE PRECISION,
  mem   DOUBLE PRECISION
);

-- 转成 hypertable，按 time 切
SELECT create_hypertable('metrics', 'time',
  chunk_time_interval => INTERVAL '1 day');
```

之后每天的数据会自动落到一个独立的 chunk（实际是 PG 的 child table + 约束排除）。

```
metrics (hypertable, 父表)
├── _hyper_1_1_chunk    (data of 2026-05-01)
├── _hyper_1_2_chunk    (data of 2026-05-02)
├── _hyper_1_3_chunk    (data of 2026-05-03)
└── ...
```

查询 `WHERE time > '2026-05-02'` 时，PG 的 constraint exclusion 会跳过 chunk 1。这叫**chunk pruning**，是 TimescaleDB 性能的关键。

**列式压缩**（compression）：

```sql
ALTER TABLE metrics SET (
  timescaledb.compress,
  timescaledb.compress_segmentby = 'host',     -- 按 host 分段
  timescaledb.compress_orderby   = 'time DESC' -- 段内按时间倒序
);

SELECT add_compression_policy('metrics', INTERVAL '7 days');
```

逻辑：7 天前的 chunk 自动从行存换成列存（PG 内部表示为单行存 array）。**典型压缩比 10x ~ 30x**，但压缩后 chunk 不再支持 UPDATE / DELETE（除非先解压）。

**chunk 数过多的隐患**：每个 chunk 是一个独立 PG 表，规划 query 时要过一遍约束。**chunk > 1000 时，规划开销 > 执行开销**。这是案例 ③ 的根。

> **口诀**：TimescaleDB = PG + 时间切片 + 列压缩。**chunk_time_interval 选错，全盘皆输** —— 太小 chunk 数爆，太大压缩/删除粒度粗。

### 2.3 Prometheus TSDB —— 2 小时 Block 模型

> Prometheus TSDB 详细已在 [Prometheus 指标设计](/article/observability/prometheus) 讲过，本节做对比定位。

Prometheus 把时间切成 **2 小时 block**：

```
data/
├── 01HXXXXXXXXXXXXXXXXXXXXXXX/    # block 1 (2h)
│   ├── chunks/
│   ├── index
│   ├── meta.json
│   └── tombstones
├── 01HYYYYYYYYYYYYYYYYYYYYYYY/    # block 2 (2h)
└── wal/
```

为什么是 2 小时？是个工程权衡：

- **太短**（如 10 分钟）：block 数太多，每次查询要打开几十个文件
- **太长**（如 1 天）：head block（内存中的当前块）太大，OOM 风险

2h 是 head 内存占用 vs 文件碎片的甜点。

后台 compaction 把相邻 block 合并：2h + 2h → 6h → 1d → 7d，最大不超过保留期的 10%。这就是 Prometheus 的"分级合并"。

Prometheus 的局限：
- **单机为主**（remote_write 后才能扩）
- **不原生支持降采样**（要靠 Recording Rules）
- **不适合长期存储**（默认 15 天，长期靠 Thanos/Cortex/Mimir/VictoriaMetrics）

### 2.4 VictoriaMetrics —— MergeTree 风格的极致压缩

VictoriaMetrics（VM）的核心设计借鉴了 ClickHouse 的 **MergeTree**：

```
data/
├── parts/
│   ├── 0_2_2_0/    # part: minRowID_maxRowID_level_mutationVersion
│   │   ├── timestamps.bin
│   │   ├── values.bin
│   │   ├── index.bin
│   │   └── metaindex.bin
│   ├── 0_5_3_0/
│   └── ...
```

每个 part 是不可变的，后台 merger 持续合并小 part 成大 part。这种设计：

- 写入只追加（不锁老数据）
- 合并时去重 + 重压缩
- 查询要扫多个 part，但每个 part 内部高度有序

**压缩**：VM 用了**多层级压缩**：

```
原始时间戳 (8B)
  → Delta-of-Delta (变长，常见 1~2B)
  → ZSTD 整块压缩

原始 value (8B)
  → Gorilla XOR (常见 1~2B)
  → ZSTD 整块压缩
```

VM 实测平均 **0.4 ~ 1 字节/点**，比 InfluxDB v2（约 2 ~ 4 字节/点）小 5~10x。这就是为什么"同一份数据 VM 占 1/5 InfluxDB 的盘"。

**索引**：VM 用了"延迟索引构建"。写入时只往 mergeset 里塞，倒排索引在 merge 时构建。代价是新写入的数据短期内查询慢一点（毫秒级），但写入吞吐显著高。

**集群版**（vmcluster）三组件：
```
vmagent / Prometheus  ──remote_write──►  vminsert  ──hash──►  vmstorage × N
                                                                    ▲
Grafana ───────────────────────────────►  vmselect ──fanout──┘
```

vmstorage 之间无主备 —— 每个 vmstorage 存一部分 series（按 series_id 哈希）。**水平扩展几乎线性**。

### 2.5 QuestDB —— 低延迟的另一种思路

QuestDB 是用 Java + 一些 native 写的，主打"亚毫秒查询延迟"。它的核心特点：

- 列存 + 时间分区（按天/月/年）
- **JIT 编译查询**（把过滤条件编译成机器码）
- SQL 兼容（基本是 PostgreSQL 方言）
- 单机为主

适合场景：**金融行情、量化回测**这种对单条查询延迟极度敏感的场景。

```sql
-- QuestDB 建表
CREATE TABLE trades (
  ts TIMESTAMP,
  symbol SYMBOL,    -- 内部字典编码
  price DOUBLE,
  qty DOUBLE
) TIMESTAMP(ts) PARTITION BY DAY;
```

`SYMBOL` 类型是 QuestDB 的招牌：把字符串映射成 int，存的是 int，查询时再翻译。**和 InfluxDB 的 tag 一样的思路，但只在 schema 层显式化**。

> **口诀**：**Prometheus** 监控、**TimescaleDB** 关系混查、**InfluxDB** 通用 + IoT、**VictoriaMetrics** 大规模高基数、**QuestDB** 极致低延迟。每个工具有它的甜点区，强行跨界用就会踩本文剩余案例。

---

## 3. 实测数据：写入、压缩、查询、基数

> 没数据的"性能更好"都是耍流氓。下面给一组同等条件下的对比，数据来源于 TSBS（Time Series Benchmark Suite）公开报告 + 部分自测。

### 3.1 写入吞吐对比

测试场景：10 万 series，每 series 1s 一个点，单机（16C / 64G / NVMe SSD）持续写 10 分钟。

| 方案 | 持续写入吞吐 (points/s) | CPU | 内存 |
| --- | --- | --- | --- |
| **VictoriaMetrics 1.95** | **3,200,000** | 35% | 2.1 GB |
| **InfluxDB v2** | 1,100,000 | 60% | 5.8 GB |
| **TimescaleDB 2.13** | 580,000 | 70% | 8.2 GB |
| **Prometheus 2.50** | 750,000 | 45% | 4.5 GB |
| **InfluxDB v3 (cloud)** | 2,400,000 | n/a | n/a |

**结论性口诀**：

- **VictoriaMetrics 写入最快**：MergeTree 写只追加，索引延迟构建
- **TimescaleDB 写入最慢**：每条数据要走 PG 的 WAL + 索引维护，代价是它有完整 SQL 能力
- Prometheus 是"被动 pull"，吞吐取决于 scrape interval × target 数

### 3.2 高基数场景下的内存占用

把 series 数从 10 万拉到 1000 万（假设你给某 metric 加了高基数 label）：

| 方案 | 10 万 series 内存 | 1000 万 series 内存 | 增长比 |
| --- | --- | --- | --- |
| Prometheus | 4 GB | **18 GB**（频繁 OOM） | 4.5x |
| InfluxDB v2 | 5 GB | **22 GB** | 4.4x |
| VictoriaMetrics | 2 GB | 6 GB | 3x |
| TimescaleDB | 3 GB | 7 GB | 2.3x |

VictoriaMetrics 和 TimescaleDB 在高基数下表现明显好。原因：
- VM 倒排索引用了 mergeset，索引数据本身可以压缩 + spilled to disk
- TimescaleDB 把 label 当成普通列，PG 的 B+ Tree 索引天然不怕基数（只是慢）

> **基数告警阈值**：对 Prometheus，**单实例 series 数 > 200 万**就该警惕；> 1000 万必须考虑 VictoriaMetrics / 拆实例。

### 3.3 聚合查询延迟

查询 `SUM by (host) over last 24h` 在 1000 host × 1 metric × 1s 精度下：

| 方案 | 不开降采样 | 开降采样到 1m | 开降采样到 5m |
| --- | --- | --- | --- |
| Prometheus | 5.2 s | (需 Recording Rule) | - |
| InfluxDB v2 | 3.8 s | 0.4 s | 0.12 s |
| TimescaleDB（无 cagg） | 7.5 s | - | - |
| TimescaleDB + Continuous Aggregate | - | **0.08 s** | 0.03 s |
| VictoriaMetrics | 2.1 s | (商业版) | - |

**结论**：**降采样能把查询延迟降 1~2 个数量级**。指标只看一周以上趋势的图表，基本都该用降采样数据，而不是原始数据。

### 3.4 长期存储成本（压缩比）

存 1 年、1 万 series、1s 精度的原始大小（裸 16B/点）= 5 TB。各方案实际占盘：

| 方案 | 实际占盘 | 字节/点 | 压缩比 |
| --- | --- | --- | --- |
| 原始（裸 16B/点） | 5 TB | 16.0 | 1x |
| InfluxDB v2 | 1.1 TB | 3.5 | 4.6x |
| Prometheus | 1.4 TB | 4.4 | 3.6x |
| TimescaleDB（启用压缩） | 480 GB | 1.5 | 10.5x |
| **VictoriaMetrics** | **180 GB** | **0.58** | **27.7x** |

VM 的压缩比为什么离谱？三个叠加：
1. **Delta-of-Delta** 时间戳：连续点 timestamp 几乎全是 1s 间隔，编码后 1~2 bit
2. **Gorilla XOR** value：连续点 value 变化小，XOR 后高位全 0
3. **ZSTD** 块压缩：在前两层基础上再压一遍

> **冷知识**：Facebook 2015 的 Gorilla 论文（"Gorilla: A Fast, Scalable, In-Memory Time Series Database"）是几乎所有现代 TSDB 压缩的祖宗。VM 在 Gorilla 之上又叠了 ZSTD，吃尽了空间。

### 3.5 高基数 label 对各方案的影响

在 metric 上加一个 100 万取值的 label（比如 `request_id`）会发生什么：

| 方案 | 写入降速 | 查询降速 | 内存涨幅 | 备注 |
| --- | --- | --- | --- | --- |
| Prometheus | -85% | -95%（OOM） | +400% | 几乎不可用 |
| InfluxDB v2 | -60% | -70% | +300% | 缓慢死亡 |
| VictoriaMetrics | -25% | -40% | +80% | 还能扛 |
| TimescaleDB | -20% | -50% | +60% | 还能扛 |

> **铁律**：**永远不要把"无界值"放进 label**。`user_id`、`request_id`、`session_id`、随机数、URL 中的查询参数 —— 这些都是 metric 杀手。它们应该进**日志**或**trace**，不进 metric。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 高基数 metric 撑爆数据库 —— "加了一个 label，第二天集群挂了"

#### 现象

某团队为了"方便排查"，给 `http_requests_total` 加了 `user_id` label：

```yaml
# 业务代码（伪）
http_requests_total{method="GET", path="/api", user_id="42"}.inc()
```

第二天告警：Prometheus 实例 OOM，重启后 5 分钟又 OOM；接收的 vmagent 提示 `too many series`；Grafana 几乎所有图打不开。

#### 根因

每个不同的 `user_id` 值 = 一条独立 series。日活 100 万用户 = 100 万 × N（N = 其他 label 组合数）series 一夜暴涨。

倒排索引 postings list 长度爆炸：

```
postings["method=GET"]    = [series_1, series_2, ..., series_100000000]   ← 1 亿
postings["path=/api"]     = [series_1, series_2, ..., series_100000000]
```

任何 `WHERE method=GET AND path=/api` 都要做 1 亿 × 1 亿 的 list 求交，内存秒爆。

更糟：Prometheus 的 head block 在内存里维护**所有活跃 series 的索引**。series 数翻 100 倍，head block 内存翻 100 倍，OOM 是必然。

#### 修复

**临时**：紧急把 user_id label 删掉。

```yaml
# vmagent 加 relabel，丢弃这个 label
metric_relabel_configs:
  - action: labeldrop
    regex: user_id
```

**长效**：建立 cardinality 监控 + label 上线 review。

```promql
# Prometheus
topk(20, count by (__name__)({__name__=~".+"}))
# 或用 prometheus_tsdb_head_series 看趋势
prometheus_tsdb_head_series

# VictoriaMetrics 提供专门 API
curl http://vm:8428/api/v1/status/tsdb
```

label 上线 review checklist：
- [ ] label 取值是不是有界（< 1000 个独立值）
- [ ] label 取值是不是和 user / request / session 这种"无界主体"绑定
- [ ] label 取值是不是来自 URL / IP / random 这种"高基数源"

如果是高基数维度，要么去掉，要么换"日志 + ClickHouse"或"trace + Tempo/Jaeger"。

**进阶**：实在要存高基数维度时用 VM 的 `streaming aggregation`：

```yaml
# vmagent 配置：在写入前就聚合掉高基数维度
streamAggrConfig:
  - match: 'http_requests_total'
    interval: 1m
    outputs: [total]
    by: [method, path, status]   # 聚合掉 user_id
```

这样写到存储的就是聚合后的低基数数据，user_id 维度被牺牲了但保住了集群。

#### 验证

```promql
# 看 series 数变化
prometheus_tsdb_head_series

# 看每个 metric 的 series 数 top
topk(10, count by (__name__)({__name__=~".+"}))
```

修复后 series 数应该回到加 label 前的水平。

> **口诀**：**label 取值必须有界**。无界的维度是 metric 的死敌，应该丢给日志和 trace。

---

### 案例 ② 历史数据降采样配置错误导致丢失 —— "三年的数据看不到了"

#### 现象

业务团队配了"原始数据 7 天 + 1m 降采样 30 天 + 1h 降采样永久"。运行半年后业务方反馈"3 个月前的图全是空的"。

#### 根因

InfluxDB v1 的常见错配：

```sql
-- 原始 retention policy：7 天
CREATE RETENTION POLICY raw ON metrics
DURATION 7d REPLICATION 1 DEFAULT;

-- 1m 降采样 retention policy：30 天
CREATE RETENTION POLICY rp_1m ON metrics
DURATION 30d REPLICATION 1;

-- 1h 降采样 retention policy：永久
CREATE RETENTION POLICY rp_1h ON metrics
DURATION INF REPLICATION 1;

-- Continuous Query：原始 → 1m
CREATE CONTINUOUS QUERY cq_1m ON metrics
BEGIN
  SELECT mean(value) INTO rp_1m.cpu_1m
  FROM raw.cpu GROUP BY time(1m), host
END;

-- 看似漏了：1m → 1h 这条 CQ 没建！
```

结果：
- 原始数据 7 天后被清理（正常）
- 1m 降采样 30 天后被清理（正常）
- **1h 降采样表里啥都没有**（因为没人往里写）

更隐蔽的版本：**CQ 建了但语法错了**。CQ 失败默认不抛错，只在 `_internal` 数据库里写错误日志，没人看。

#### 修复

**立即**：从备份恢复原始数据，重新跑历史聚合：

```sql
-- 一次性回填：把过去 30 天的原始数据聚合成 1h
SELECT mean(value) INTO rp_1h.cpu_1h
FROM raw.cpu
WHERE time > now() - 30d
GROUP BY time(1h), host;
```

但前提是原始数据还在备份里。如果只剩 30 天的 1m 数据，那只能从 1m 降采样到 1h（精度损失但能恢复趋势）。

**长效**：

1. **建立 CQ 健康监控**：
```sql
-- 查 CQ 执行错误
SELECT * FROM "_internal"."monitor"."continuous_queries"
WHERE time > now() - 1h AND error != ''
```

2. **降采样链条形式化检查**：
```python
# 把降采样链条写成配置，启动时校验
DOWNSAMPLE_CHAIN = [
    ('raw',   None,   '7d'),
    ('rp_1m', '1m',   '30d'),
    ('rp_1h', '1h',   'INF'),
]

def verify():
    for prev, curr in pairwise(DOWNSAMPLE_CHAIN):
        # 检查 CQ 是否存在 + 语法正确
        # 检查 retention 是否一致
        ...
```

3. **数据完整性巡检**：定时跑一个 SQL，检查每个降采样表"最近一个数据点"的时间。如果超过预期间隔，告警。

#### 验证

```sql
-- 每个表的最新数据点时间应该和当前时间差不超过预期间隔
SELECT MAX(time) FROM rp_1h.cpu_1h;  -- 应该是 < 1h 以内
SELECT MAX(time) FROM rp_1m.cpu_1m;  -- 应该是 < 1m 以内
```

> **口诀**：**降采样链条必须从原始数据一路通到永久保存**。配完之后**第二天**和**保留期到期前**各做一次完整性巡检，不要等用户报。

---

### 案例 ③ TimescaleDB chunk 数过多 —— "简单 SELECT 从 50ms 变 5s"

#### 现象

TimescaleDB hypertable，刚建表时单条 `SELECT * FROM metrics WHERE host='web01' AND time > now()-1h LIMIT 10` 是 50ms。运行半年后变成 5s。EXPLAIN 显示 plan time 比 execute time 长 100 倍。

#### 根因

```sql
-- 当时建表
SELECT create_hypertable('metrics', 'time',
  chunk_time_interval => INTERVAL '1 hour');   -- ★ 1 小时一个 chunk
```

半年 = 4380 个 chunk。每条 SQL 的查询规划阶段，PG 必须遍历所有 chunk 的 child table，对每个 chunk 检查 constraint 是否能裁剪。**chunk 数 > 1000 之后，规划开销线性上涨**。

更糟：每个 chunk 的统计信息（pg_statistic）也要单独维护。`ANALYZE` 也变得越来越慢。

#### 修复

**立即**：合并历史 chunk。TimescaleDB 提供了 `move_chunk` 但不能直接 merge。曲线方法：

```sql
-- 创建一个新 hypertable，chunk_time_interval 调到 1 天
CREATE TABLE metrics_new (LIKE metrics INCLUDING ALL);
SELECT create_hypertable('metrics_new', 'time',
  chunk_time_interval => INTERVAL '1 day');

-- 拷贝旧数据
INSERT INTO metrics_new SELECT * FROM metrics;

-- 切换
BEGIN;
ALTER TABLE metrics RENAME TO metrics_old;
ALTER TABLE metrics_new RENAME TO metrics;
COMMIT;

-- 老表分批删
```

**长效**：
1. 重新规划 chunk_time_interval。**经验公式**：
   ```
   chunk_time_interval ≈ 你想让"最近热数据"放进 25% 内存的时长
   ```
   - 64G 内存 → 16G 给热数据 → 假设单天数据 4G → chunk_interval = 4 天
   - 实际再保守一点取 1 天 / 1 周

2. 启用压缩：旧 chunk 压缩后内存占用 / 文件数大幅下降

```sql
-- 7 天前的 chunk 自动压缩
ALTER TABLE metrics SET (
  timescaledb.compress,
  timescaledb.compress_segmentby = 'host'
);
SELECT add_compression_policy('metrics', INTERVAL '7 days');
```

3. 启用 `timescaledb.enable_chunk_append = on`（默认 on，但旧版本要确认）：让 PG 在规划时跳过显然不在范围的 chunk。

4. 设保留策略，自动清理过老的 chunk：
```sql
SELECT add_retention_policy('metrics', INTERVAL '90 days');
```

#### 验证

```sql
-- 看 chunk 数
SELECT count(*) FROM timescaledb_information.chunks
WHERE hypertable_name = 'metrics';

-- 看单 SQL 的规划时间
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT * FROM metrics WHERE host='web01' AND time > now()-1h LIMIT 10;
-- Planning Time 应该 < 5ms
```

> **口诀**：**TimescaleDB chunk 数 < 1000** 是经验红线。chunk 太小是慢性自杀。建表时算好 chunk_time_interval，定期监控 chunk 总数。

---

### 案例 ④ InfluxDB v1 → v2 / v3 迁移踩坑

#### 现象

业务把 InfluxDB v1 升级到 v2，准备过几个月再升 v3。结果：
- 部分 dashboard 报错 "function not supported in Flux"
- Continuous Query 的语义微妙变化导致数据重复
- v3 公告只支持 SQL，团队发现 Flux 脚本要全部重写

#### 根因

InfluxDB 三代变化巨大：

| 维度 | v1 | v2 | v3 |
| --- | --- | --- | --- |
| 查询语言 | InfluxQL | Flux（主推） + InfluxQL | SQL（DataFusion） |
| 存储 | TSM | TSM | Parquet (Arrow) |
| 任务系统 | Continuous Query | Tasks | Tasks + 自动 |
| 数据组织 | database + retention policy | bucket | bucket + 自动 partition |
| API | /write, /query | /api/v2/* | /api/v3/* + Flight SQL |

v1 → v2 的关键陷阱：

1. **Database/RP → Bucket** 概念变了。原本一个 db 多 rp，v2 一个 bucket 一个 retention。**迁移时要把 db.rp 拍平成多个 bucket**。

2. **CQ → Task**。但 Task 是 Flux 写的。Flux 是函数式 + 管道风格，不是 SQL：

```flux
// Flux 版的 5m 降采样 task
option task = {name: "downsample_5m", every: 5m}

from(bucket: "raw")
  |> range(start: -task.every)
  |> filter(fn: (r) => r._measurement == "cpu")
  |> aggregateWindow(every: 5m, fn: mean)
  |> to(bucket: "downsampled")
```

3. **InfluxQL 子集**。v2 仍然支持 InfluxQL 但是子集，部分函数（subquery、复杂正则）不支持。Grafana dashboard 要逐个验证。

v2 → v3 是更大的手术：
- Flux **被弃用**，v3 只支持 SQL
- Token / 权限模型重做
- 部分写入路径行为不同（v3 自动去重重复时间戳）

#### 修复

**v1 → v2 迁移最佳实践**：

```bash
# 1. 用官方工具 export（兼容性最好）
influxd export --bucket-id <bucket> --output-path /backup

# 2. 在 v2 上 import
influx restore /backup --bucket newbucket

# 3. CQ 手工翻译成 Task
# （没有自动工具，必须人工翻译，做好测试）

# 4. dashboard 用 InfluxDB Cloud Migration Tool 检查兼容性
```

**v2 → v3 迁移**：
- 短期：双写 v2 和 v3，新数据走 v3，老数据查 v2
- 长期：把 Flux 改写成 SQL（v3 的 SQL 是 DataFusion 方言）

```sql
-- v3 SQL 示例
SELECT
  date_bin(INTERVAL '5 minutes', time) AS bucket,
  host,
  AVG(usage) AS avg_usage
FROM cpu
WHERE time > now() - INTERVAL '1 hour'
GROUP BY bucket, host
ORDER BY bucket;
```

**关键原则**：**任何 v 间迁移都先在 staging 跑全量 dashboard 自动化测试**。建一个 dashboard 截图对比工具（v1 截图 vs v2 截图）。

#### 验证

```bash
# 抽样 100 个查询，v1 vs v2 结果对比，差异容忍 < 1%
python compare_v1_v2.py --queries queries.json --tolerance 0.01
```

> **口诀**：**InfluxDB 三代之间不是平滑升级，是替换**。迁移成本很高，迁移前先评估"留 v1 + 引入 VictoriaMetrics 做新需求"是不是更现实。

---

### 案例 ⑤ 写入风暴 —— "单批次过大导致内存峰值"

#### 现象

某 IoT 网关每 5 分钟把缓存的 10 万设备数据一次性 batch 写入 InfluxDB。InfluxDB 进程的内存呈"锯齿状"周期性飙到 30G，几次差点 OOM。

#### 根因

InfluxDB 写入路径的**内存放大**：
- 客户端发送 100MB Line Protocol
- 服务端 parse 成内存 Series（约 3x 放大 → 300MB）
- 进 cache（可能再次拷贝）
- 触发 WAL flush + cache → TSM
- 在 TSM compaction 时还要再拷一次

`5 分钟一批 + 单批 100MB` 实际峰值就接近 1GB，再叠加多个客户端 + 并发 compaction，内存峰值就上 30G。

#### 修复

**写入侧**：把单批拆小、提高频率。

```python
# Bad: 5min 一批，单批 100k
buffer = []
for point in stream:
    buffer.append(point)
    if len(buffer) >= 100000:
        client.write(buffer)
        buffer.clear()

# Good: 1s 一批 + 上限 5k
buffer = []
last_flush = time.time()
for point in stream:
    buffer.append(point)
    if len(buffer) >= 5000 or time.time() - last_flush > 1:
        client.write(buffer)
        buffer.clear()
        last_flush = time.time()
```

或者用 **Telegraf** 这种成熟 collector，它自带 batching + retry + backpressure。

**InfluxDB 侧**：调几个关键参数（v2）。

```toml
[storage]
# WAL 同步频率
wal-fsync-delay = "100ms"

# Cache 多大触发 snapshot
cache-snapshot-memory-size = "256m"     # 默认 25m，IoT 场景可以调大避免频繁 flush
cache-max-memory-size = "1g"            # 硬上限

# Compaction 最大并发
compact-throughput = "48m"
compact-throughput-burst = "48m"
```

**容量层**：上限保护。

```toml
# v2 全局写限流
[write]
max-concurrent-write-limit = 0   # 0 = 不限，建议设具体值
max-enqueued-write-limit = 0     # 队列上限
```

或者在 ingress 层加限流（Nginx / Envoy）：

```nginx
limit_req_zone $binary_remote_addr zone=influx:10m rate=100r/s;
```

#### 验证

```bash
# 监控关键指标
curl http://localhost:8086/metrics | grep -E "(cache_memory|compactions|wal_bytes)"

# 内存曲线应该平滑（不再是锯齿）
```

> **口诀**：**写入是流不是批**。client 侧小批量高频；服务端给 cache / compaction 设硬上限。突发 batch 最大的 TSDB 杀手。

---

### 案例 ⑥ Continuous Query 拖慢主集群 —— "聚合一跑，业务图全卡"

#### 现象

InfluxDB v1 上线了一组 5m 降采样 CQ。每到 5 分钟整点，监控大屏卡住 5~10 秒。jstack 看不出问题，但 InfluxDB 的 `_internal` 数据库显示 "queryWriteOk" 短时间归零。

#### 根因

CQ 在 InfluxDB v1 / v2（task）里是**和正常查询/写入共享同一个执行池**的。CQ 跑的是"全表 GROUP BY time(5m)"扫描，扫几亿点的查询会：

- 占满所有 query worker
- 占满 IO（读海量原始数据）
- 占满内存（聚合中间结果）

主集群的正常查询/写入此时只能排队。

更糟的常见配错：

```sql
CREATE CONTINUOUS QUERY cq_5m ON db
RESAMPLE FOR 1h                      -- ★ 每次往前算 1 小时
BEGIN
  SELECT mean(value) INTO downsampled.cpu_5m
  FROM cpu GROUP BY time(5m), host
END
```

`RESAMPLE FOR 1h` 是说"每次跑回头算过去 1 小时"。如果你的 CQ 频率是 5 分钟，那等于每 5 分钟把过去 1 小时全算一遍 —— **同一段数据被算了 12 次**。CPU 直接打满。

#### 修复

**方案 A**：CQ 错峰 + 收窄回溯窗口。

```sql
-- 不要让所有 CQ 都在整点跑
CREATE CONTINUOUS QUERY cq_5m_a ON db RESAMPLE EVERY 5m FOR 10m ...
CREATE CONTINUOUS QUERY cq_5m_b ON db RESAMPLE EVERY 5m FOR 10m ...
-- 加上偏移让它们错开
```

**方案 B**：迁到 TimescaleDB 的 Continuous Aggregate。CA 是**增量**的，只算上次刷新后新进来的数据。

```sql
-- 增量物化视图
CREATE MATERIALIZED VIEW cpu_5m
WITH (timescaledb.continuous) AS
SELECT time_bucket('5m', time) AS bucket, host, AVG(value)
FROM cpu GROUP BY bucket, host;

-- 调度策略
SELECT add_continuous_aggregate_policy('cpu_5m',
  start_offset => INTERVAL '1 hour',     -- 重算最近 1h（处理迟到数据）
  end_offset   => INTERVAL '5 minutes',  -- 不算最近 5 分钟（数据可能未稳定）
  schedule_interval => INTERVAL '5 minutes');
```

**方案 C**：彻底分离写入集群和聚合集群。InfluxDB Enterprise / Cloud / Cortex / Mimir 都支持把 ingester 和 querier/ruler 分开部署。

#### 验证

```promql
# 监控 CQ 跑的耗时
internal.queryDurationNs

# CQ 跑期间业务查询的 P99 不应有尖刺
http_request_duration_seconds{quantile="0.99"}
```

CQ 跑的时间窗口内业务查询延迟无变化 = 修复成功。

> **口诀**：**CQ / Task 必须和业务查询资源隔离，且要用增量算法**。InfluxDB v1 的 CQ 是踩坑地图，能换 TimescaleDB CA 或 Prometheus Recording Rules 就换。

---

### 案例 ⑦ 跨时区查询的混乱 —— "同一时刻的数据，不同人看出来不一样"

#### 现象

国内开发用 UTC+8 的浏览器，看 Grafana 显示某指标在 `2026-05-01 08:00` 的值是 100。
新加坡同事看到的同一行（`2026-05-01 08:00`）显示是 200。
DBA 在数据库里 `SELECT` 出来又是另一个值。

#### 根因

时序数据的时间是**纯 epoch 毫秒/纳秒**，没有时区信息。但展现层都要做时区转换：

- Grafana：根据 user 浏览器时区或 Org 设置渲染
- Prometheus：UTC（不可配）
- InfluxDB：内部 UTC，但 InfluxQL 支持 `tz('Asia/Shanghai')`
- TimescaleDB：基于 PG 的 `timestamptz` 和 session timezone

混乱场景：
- Grafana 默认 "Browser" 时区，用户浏览器不同 → 不同人看到不同时间标签
- 业务 SQL `WHERE time > '2026-05-01 00:00:00'` 不带时区 → 解释成 server timezone
- 上游 client 写入时用 `localtime` 而不是 UTC → 数据本身就错位

#### 修复

**铁律**：

1. **写入侧永远用 UTC epoch**。Line Protocol 直接发 `nanosecond timestamp`，client 不要做任何"友好转换"。
2. **存储侧按 UTC 存**（默认就是）。
3. **显示侧明确时区**。Grafana 给每个 dashboard 显式设时区，不要用 "Browser"。
4. **SQL 里所有时间字面量带时区**。

```sql
-- 错：歧义
SELECT * FROM metrics WHERE time > '2026-05-01 00:00:00';

-- 对：明确
SELECT * FROM metrics WHERE time > '2026-05-01 00:00:00+08:00';

-- TimescaleDB 推荐
SELECT * FROM metrics
WHERE time > timestamptz '2026-05-01 00:00:00 Asia/Shanghai';
```

5. **Grafana dashboard JSON 里**：

```json
{
  "timezone": "Asia/Shanghai",   // 不要用 "browser"
  ...
}
```

6. **聚合时窗口对齐到时区**。比如"按天聚合"，用户期望是"北京时间的天"，但 UTC 0 点和北京 0 点差 8 小时。InfluxQL 里：

```sql
SELECT mean(value)
FROM cpu
GROUP BY time(1d, 8h)  -- 偏移 8h，让窗口对齐到 UTC+8 的 0 点
```

TimescaleDB：

```sql
SELECT time_bucket('1 day', time, 'Asia/Shanghai') AS day, AVG(value)
FROM cpu GROUP BY day;
```

#### 验证

跨地区团队约定：所有 dashboard 截图必须显示时区标识（`(UTC+8)` 之类）。出现争议时先对齐时区再讨论数据。

> **口诀**：**时间数据 = epoch + 时区显示约定**。存储无时区，写入用 UTC，展现明确时区，SQL 字面量带时区。任一环节模糊就出鬼。

---

### 案例 ⑧ 时序与关系数据混查的性能 —— "join 一下就 30 秒"

#### 现象

业务在 TimescaleDB 上把订单时序数据（events）和用户表（users）join：

```sql
SELECT u.country, COUNT(*)
FROM events e JOIN users u ON e.user_id = u.id
WHERE e.time > now() - INTERVAL '7 days'
GROUP BY u.country;
```

events 表 50 亿行（hypertable），users 表 1000 万行（普通表）。查询 30 秒。

#### 根因

执行计划里看到的是 `Hash Join`，PG 把 events 7 天的所有匹配行（数千万）取出来，再用 hash 表 lookup users。问题：

1. 7 天的 events 跨越多个 chunk，每个 chunk 都要扫
2. events 没有 `(time, user_id)` 复合索引，user_id 过滤没法下推
3. join 的中间结果集很大

#### 修复

**方案 A**：物化高频查询。

```sql
CREATE MATERIALIZED VIEW events_by_country_daily
WITH (timescaledb.continuous) AS
SELECT
  time_bucket('1 day', e.time) AS day,
  u.country,
  count(*) AS cnt
FROM events e JOIN users u ON e.user_id = u.id
GROUP BY day, country;
```

但 join 在 cagg 里有限制（不是所有 join 都支持），可能要自己写定时任务。

**方案 B**：把维度反范式化进时序表。

```sql
-- 写入时就把 country 一起写进 events
ALTER TABLE events ADD COLUMN country TEXT;

-- 然后查询不需要 join
SELECT country, COUNT(*)
FROM events
WHERE time > now() - INTERVAL '7 days'
GROUP BY country;
```

代价：users 表的 country 改了，events 历史数据怎么同步？要权衡。

**方案 C**：把时序数据 ETL 到 ClickHouse。**ClickHouse 在大表 join 大表场景明显优于 TimescaleDB**。

```sql
-- ClickHouse
SELECT u.country, count(*)
FROM events e JOIN users u ON e.user_id = u.id
WHERE e.time > now() - INTERVAL 7 DAY
GROUP BY u.country
SETTINGS join_algorithm = 'hash';
```

ClickHouse 的 vectorized hash join + dictionary join 能把这个查询从 30s 降到 2s。

**方案 D**：维度数据入 dictionary（ClickHouse）或 enum（PG）。

```sql
-- PG enum
CREATE TYPE country_t AS ENUM ('CN', 'US', 'JP', ...);
```

#### 验证

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ... -- 修复后单查询应在 2s 以内
```

> **口诀**：**时序表 join 大表是反模式**。要么物化，要么反范式化，要么换 ClickHouse。强行 join 就准备 30 秒。

---

### 案例 ⑨ 备份恢复 —— "block 损坏 + snapshot 只剩一半"

#### 现象

某 Prometheus 实例机器宕机，磁盘部分扇区损坏。重启后 Prometheus 报 `corrupted block`，启动失败。备份策略是每天 `cp -r data/` 到 NAS，最近一次备份是 18 小时前。

#### 根因

Prometheus 的 `cp -r` 备份**不安全**：

1. block 是不可变的，但**复制过程中可能正在生成新 block**（compaction 写新 block + 删老 block）
2. WAL 在持续写入
3. `meta.json` 在某些时刻可能与实际文件不一致

`cp -r` 拷出来的备份，**内部状态可能不一致**，恢复时启动失败。

正确做法是用 Prometheus 的 **snapshot API**：

```bash
# 触发 snapshot：会在 data/snapshots/ 下生成一个原子的、一致的快照
curl -XPOST http://prom:9090/api/v1/admin/tsdb/snapshot

# 返回值：{"data":{"name":"20260509T153045Z-7b8f9c..."},"status":"success"}

# 再 rsync 这个 snapshot 目录到远端
rsync -av data/snapshots/20260509T153045Z-* backup-server:/backup/
```

snapshot 是**硬链接**到现有 block 文件，所以瞬时完成且不占空间（直到这些文件被 compaction 删掉）。snapshot 的语义是"包含截至 snapshot 时刻的所有完整 block，但不包含 head（内存中的 2h）"。

#### 修复

**事中**：

```bash
# 找到损坏的 block
ls data/ | xargs -I {} promtool tsdb analyze data/{}

# 删掉损坏的 block（损失这 2h 的数据）
rm -rf data/01HXXXXXXXXXXXXXXXXXXXXXXX/

# 启动
./prometheus
```

**长效**：

1. **用 snapshot API 备份，不要 cp -r**：

```bash
# 自动化脚本
#!/bin/bash
NAME=$(curl -sXPOST http://localhost:9090/api/v1/admin/tsdb/snapshot | jq -r .data.name)
rsync -avz --remove-source-files /data/snapshots/$NAME/ backup:/prom-backup/$(date +%F)/
```

注意：开 snapshot API 要在启动参数里加 `--web.enable-admin-api`。

2. **head block 的丢失** 通过 WAL 弥补。WAL 是 segment 文件，每个段约 128MB，按需 fsync。崩溃后启动会 replay WAL 重建 head block。所以备份时要**同时备份 WAL** 才能完整恢复 head 数据。

3. **长期归档** 用 Thanos / Cortex / VictoriaMetrics 接 remote_write，把数据上 S3。Prometheus 本身不应该承担"长期可靠存储"的职责。

```yaml
# Prometheus remote_write 到 VM
remote_write:
  - url: http://vm:8428/api/v1/write
```

VM 自带集群 + 复制 + 持久化，比 Prometheus 单机可靠得多。

4. **跨集群多写**（关键场景）：

```yaml
# 同时写两个集群
remote_write:
  - url: http://vm-cluster-1:8428/api/v1/write
  - url: http://vm-cluster-2:8428/api/v1/write
```

#### 验证

```bash
# 定期做"恢复演练"：从备份起一个临时实例，跑一些查询验证数据完整
docker run -v /backup/2026-05-09:/data prom/prometheus
curl 'http://localhost:9090/api/v1/query?query=up'
```

> **口诀**：**Prometheus 备份用 snapshot API，不用 cp -r**。长期可靠存储靠 remote_write 到 VM/Thanos。备份必须有"恢复演练"才算数。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### Schema / 基数层
- [ ] 每个 metric 的 label 取值数 (cardinality) 经过评估，单 label < 1000
- [ ] 禁止把 `user_id` / `request_id` / `session_id` / random / URL 查询参数 当 label
- [ ] 上线新 label 之前先在 staging 跑 24 小时，看 series 增量
- [ ] 监控 `prometheus_tsdb_head_series` / VM `vm_cache_entries`，单实例 series 数有上限告警

### 写入层
- [ ] 写入用小批量（< 5000 点）+ 高频率（< 1s），不用大批量低频
- [ ] 写入 client 用成熟 SDK / Telegraf / vmagent，不要自己拼 Line Protocol
- [ ] 写入路径有限流和熔断（防止突发流量打挂存储）
- [ ] 时间戳一律用 UTC epoch；client 不要做时区转换

### 存储层
- [ ] 配 retention policy，且降采样链条**从原始一路通到永久**
- [ ] 降采样配置完成 24 小时后做完整性巡检
- [ ] TimescaleDB chunk_time_interval 经过计算，chunk 总数 < 1000
- [ ] TimescaleDB 旧 chunk 启用列式压缩，过期 chunk 自动删除
- [ ] InfluxDB 的 cache / compaction 内存上限明确设置

### 查询层
- [ ] Continuous Query / Task 用增量算法（如 TimescaleDB cagg），不要全表扫
- [ ] 长查询有 timeout（PromQL `query.timeout` / SQL `statement_timeout`）
- [ ] 高频查询 dashboard 用降采样数据，原始数据只在排查时用
- [ ] 时间字面量带时区，Grafana dashboard 显式设时区

### 运维层
- [ ] Prometheus 用 snapshot API 备份，**不用 cp -r**
- [ ] 关键集群 remote_write 双写（跨可用区或跨集群）
- [ ] 长期数据归档到 VM / Thanos / S3（Prometheus 本机只存近 15 天）
- [ ] 每月做一次"备份恢复演练"
- [ ] InfluxDB / TimescaleDB 大版本升级前在 staging 跑全量 dashboard 截图对比

### 监控本身
- [ ] 监控系统自身的 series 数、内存、磁盘、查询 P99
- [ ] 监控系统不要用自己监控自己（独立的小 Prometheus 监控大 Prometheus）
- [ ] 关键告警有冗余通道（电话 + 钉钉 + 邮件）
- [ ] 节假日前做容量演练，确认能扛过流量峰值

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### InfluxDB v3 (IOx + DataFusion + Arrow)
- 从 TSM 到 Parquet 的存储层重构
- DataFusion 的查询规划与执行（向量化、多线程、SIMD）
- Apache Arrow Flight SQL 协议（替代 HTTP/JSON）
- 与 Iceberg / Delta Lake 的潜在协同（Parquet 共用）

### 与 ClickHouse / Druid / Pinot 在监控场景的对比
- ClickHouse 的 MergeTree vs VictoriaMetrics 的 MergeTree
- Druid 的 segment + 实时摄取层（IndexTask）
- Pinot 的 startree index（针对高维聚合）
- 哪些场景这些 OLAP 引擎能替代 TSDB（提示：高维聚合 + join）

### 物联网（IoT）场景的时序方案
- 海量小设备（百万 device）+ 低频数据（5 min/点）的设计模式
- 边缘聚合（device → gateway → cloud）的层级架构
- MQTT / CoAP 接入层与 TSDB 的解耦
- 边缘时序数据库（如 EMQX 的 ekuiper）

### Telegraf / OpenTelemetry 接入
- Telegraf 200+ input plugin 的最佳实践
- OpenTelemetry Metrics / Traces / Logs 三合一的统一收集
- OTLP 协议与 Prometheus exposition format 的互转
- 用 OTel Collector 做 fan-out（一份数据写 Prometheus + 写 ClickHouse）

### 与 Grafana / Chronograf 的协作
- Grafana 9+ 的 Explore 模式与 TSDB 的深度集成
- Loki / Tempo / Mimir 全家桶的"分而治之"
- Chronograf（InfluxDB 自带）vs Grafana 的取舍
- 自定义 Datasource Plugin 接入私有 TSDB

### 时序预测与异常检测
- Facebook Prophet 在容量规划场景的应用
- 基于 STL（Seasonal-Trend decomposition using Loess）的异常检测
- 工业界对接 ML：把 TSDB 的数据导出到 Spark / TF Serving
- VictoriaMetrics 的 anomaly-detection 子项目

### 边缘时序数据库
- TDengine（涛思数据）的 stable + subtable 模型
- Greptime DB（Rust + DataFusion）的多租户设计
- 嵌入式 TSDB（如 Akumuli、SiriDB）

---

## 7. 引用

- VictoriaMetrics 文档与博客：[VictoriaMetrics docs](https://docs.victoriametrics.com/)
- TimescaleDB 文档：[TimescaleDB docs](https://docs.timescale.com/)
- InfluxDB v3 (IOx) 设计：[InfluxDB IOx GitHub](https://github.com/influxdata/influxdb_iox)
- Prometheus TSDB 论文：Fabian Reinartz, *Writing a Time Series Database from Scratch*（2017）
- Facebook Gorilla 论文：[Gorilla: A Fast, Scalable, In-Memory Time Series Database](https://www.vldb.org/pvldb/vol8/p1816-teller.pdf)（VLDB 2015）
- TSBS 基准：[Time Series Benchmark Suite](https://github.com/timescale/tsbs)
- Apache Arrow / DataFusion：[Arrow project](https://arrow.apache.org/)
- ClickHouse MergeTree：[ClickHouse docs](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/mergetree)

---

> 配套：[Prometheus 指标设计](/article/observability/prometheus) | [ClickHouse 深入](/article/db/clickhouse) | [日志架构](/article/observability/logging)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
