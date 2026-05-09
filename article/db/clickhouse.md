# ClickHouse 深入 —— 列存、MergeTree、物化视图与 OLAP 实战

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想知道 ClickHouse "是个 OLAP 列存数据库" 这种概要，去任意一篇综述都够了。
> 这里要回答的，是**你接手了一个百亿行的 ClickHouse 集群、半夜被 "Too many parts" 告警叫醒**时你需要知道的那部分。

---

## 0. 读这篇之前 —— 8 个自检题

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. 同样查 "近 7 天某商品的下单量 GROUP BY 城市"，ClickHouse 比 MySQL 快几十上百倍 —— **本质差别**是什么？仅是 "列存比行存快" 这一句够吗？
2. ClickHouse 的 `PRIMARY KEY` 和 MySQL 的 `PRIMARY KEY` **完全是两回事**。前者**不是唯一约束**。它到底是什么？写错了会出现什么现象？
3. `ORDER BY` 和 `PARTITION BY` 经常被搞混。给一个 100 亿行的事件表，**你打算怎么定这两个字段**？为什么选择是这样？
4. MergeTree 的 part 是怎么产生的？什么时候 merge？为什么会出现 "Too many parts" 错误？这个错误的本质是 ClickHouse 在保护谁？
5. 物化视图（Materialized View）的 "增量" 到底是什么意思？它是 INSERT 触发的还是定时跑的？主表写入失败和物化视图写入失败之间是什么关系？
6. `ReplacingMergeTree` / `SummingMergeTree` / `AggregatingMergeTree` 看名字都像 "去重 / 聚合"。**它们的去重/聚合什么时候发生**？SELECT 立即看到结果吗？
7. `Distributed` 表和本地表是什么关系？为什么写入要写本地表而读用分布式表？JOIN 在分布式集群里有个 "死穴"，是哪条？
8. `max_memory_usage` 没设的集群，一个用户的大查询能不能把整个 ClickHouse 节点打挂？

如果有半数答不上，往下看。

---

## 1. 底层模型

绝大多数人对 ClickHouse 的认知止于 "OLAP 列存"，但这层抽象解释不了上面 8 个问题。要看懂它们，必须下沉到 MergeTree 的物理布局和写入/查询路径。

### 1.1 列存的核心优势：IO + 压缩 + 向量化

行存（MySQL InnoDB）按行打包到 page 里：

```
page: [row1 col1 col2 col3 col4 col5][row2 col1 col2 col3 col4 col5]...
```

查询 `SELECT col2, col3 FROM t WHERE col1 > 100`，你要扫一整页才能拿到 col2 + col3，**带宽全花在你不需要的列上**。

列存按列分别存储：

```
col1.bin: [row1.col1, row2.col1, row3.col1, ...]
col2.bin: [row1.col2, row2.col2, row3.col2, ...]
col3.bin: [row1.col3, row2.col3, row3.col3, ...]
```

只读你要的两列。**IO 量直接砍掉 N 倍**（N = 表的列数）。

但这只是开胃菜。真正的差距来自：

- **压缩比**：同一列的值类型/分布相似，LZ4 / ZSTD 压缩比能到 10:1 甚至 30:1。100GB 行存 → 5~10GB 列存。
- **向量化执行**：列存天然是按列的 array，CPU 一次处理一批（SIMD），分支预测命中率极高。
- **稀疏索引**：不需要为每行建索引，按 "块" 索引就够。索引体积小到能完全装内存。

> **口诀**：列存的优势 = IO 减少 + 高压缩比 + 向量化 + 稀疏索引。**单独一个 "列存" 解释不了几十倍的差距，是这四件事一起干的活**。

### 1.2 ClickHouse 架构：Server / 分片 / 副本

```
                    ┌──────────────────────┐
                    │   client / proxy     │
                    └──────────┬───────────┘
                               │
       ┌──────────────────────┼──────────────────────┐
       │                      │                      │
   ┌───▼────┐            ┌────▼───┐            ┌────▼───┐
   │ shard1 │            │ shard2 │            │ shard3 │
   │ ┌────┐ │            │ ┌────┐ │            │ ┌────┐ │
   │ │rep1│ │            │ │rep1│ │            │ │rep1│ │
   │ ├────┤ │            │ ├────┤ │            │ ├────┤ │
   │ │rep2│ │            │ │rep2│ │            │ │rep2│ │
   │ └────┘ │            │ └────┘ │            │ └────┘ │
   └────────┘            └────────┘            └────────┘
                               │
                       ┌───────┴────────┐
                       │ ZooKeeper /    │
                       │ ClickHouse     │
                       │ Keeper         │
                       └────────────────┘
```

- **shard（分片）**：数据按某个 key 水平切到不同 shard，每个 shard 存一部分数据
- **replica（副本）**：同一个 shard 的多份完整拷贝，做 HA 和读扩展
- **ZooKeeper / Keeper**：协调副本之间的复制状态，分布式 DDL 也走它

注意一个反直觉的点：**ClickHouse 没有 "主节点"**。所有节点对等，集群拓扑写在每个节点的 `remote_servers` 配置里。这意味着集群扩容比 MySQL 那种 "主从" 灵活，但配置错了也更容易翻车。

### 1.3 MergeTree 引擎家族

ClickHouse 的核心引擎不是 "ClickHouse"，而是 MergeTree 家族。常用 6 个：

| 引擎 | 用途 |
| --- | --- |
| **MergeTree** | 默认引擎，最普通的列存表 |
| **ReplacingMergeTree** | 后台 merge 时按主键去重，保留最新版本 |
| **SummingMergeTree** | 后台 merge 时把数值列按主键聚合（求和） |
| **AggregatingMergeTree** | 后台 merge 时按主键聚合（任意聚合函数，配合 `-State` / `-Merge`） |
| **CollapsingMergeTree** | 用 `sign` 列标记 "添加 / 删除"，merge 时折叠 |
| **VersionedCollapsingMergeTree** | Collapsing 的有版本号版本，乱序写入也能正确折叠 |

加上 `Replicated` 前缀就是带副本的版本（`ReplicatedMergeTree` / `ReplicatedReplacingMergeTree` ...）。

**关键认知**：所有 "去重 / 聚合" 都发生在**后台 merge** 时，不是写入时也不是查询时。这是后面案例 ① 和案例 ④ 的根。

### 1.4 数据分区（Partition）+ 主键索引（稀疏索引）

每个 MergeTree 表在磁盘上是这样的：

```
/var/lib/clickhouse/data/db/table/
├── 20251101_1_1_0/         ← part 目录：分区_最小块号_最大块号_合并层级
│   ├── columns.txt
│   ├── count.txt
│   ├── primary.idx          ← 稀疏索引（每 8192 行一条索引）
│   ├── col1.bin            ← 列数据
│   ├── col1.mrk2           ← 列的标记文件，索引到 bin 的偏移
│   ├── col2.bin
│   ├── col2.mrk2
│   └── ...
├── 20251101_2_2_0/
├── 20251101_3_3_0/
└── 20251102_4_4_0/
```

**目录名编码了一切**：
- `20251101` = 分区键（这里按天分区）
- `1_1_0` = 块号区间 [1, 1]，merge 层级 0（原始 part）
- merge 后变成 `20251101_1_3_1`：合并了 [1,3] 三个 part，层级升到 1

**主键索引是稀疏的**。MySQL InnoDB 的索引是 "每行一条索引项"，ClickHouse 默认 **每 `index_granularity = 8192` 行一条索引项**。

```
primary.idx 内容（举例：主键是 user_id）
[user_id=100, user_id=8500, user_id=15200, user_id=22000, ...]
```

查 `WHERE user_id = 9000`，二分定位到落在 `[8500, 15200]` 这个 granule（颗粒）里 → 只读这一个 8192 行的块。

代价：必须读完整 8192 行（不能直接跳到第 9000 行）。但这个代价可以忽略 —— 因为列存按块批量读才是高效的。

### 1.5 写入路径：part → background merge

```
INSERT 一批数据
   │
   ▼
1. 排序（按 ORDER BY 字段）
2. 拆分到对应的分区
3. 每个分区生成一个新 part（独立目录）
4. 写入 primary.idx + 列文件
5. 写完通知 ZK（Replicated 表）
   │
   ▼
   后台 merge 线程异步合并 part
   策略：把小的合并成大的，类似 LSM-Tree 的层级合并
   merge 后的 part 留几分钟才删旧 part（防止读端正在用）
```

**关键约束**：每次 INSERT 至少生成一个 part。如果你每秒 INSERT 1000 次（每次 1 行），每秒会产生 1000 个 part。merge 跟不上 → part 数累积 → "Too many parts" 错误。

ClickHouse 的官方建议：**单次 INSERT 至少几千~几万行，频率不要超过每秒几次**。

`max_parts_in_total = 100000` 是默认上限，超过就报错保护集群（merge 线程被压垮会让查询变慢、磁盘碎片暴涨）。

### 1.6 查询路径：分布式表 → 本地表 → 列读取 → 向量化

查询一条 SQL 的全过程：

```
SELECT count() FROM events_dist WHERE date = '2025-11-01' AND city = 'Beijing'

   ① 接收节点解析 SQL
   ② events_dist 是 Distributed 表 → 发请求到所有 shard 的本地表
       events_local（events_dist 的 underlying）
   ③ 每个 shard 独立执行：
       - 用 partition key (date) 直接定位到 20251101_* part
       - 用 primary key 索引找到候选 granule
       - 读 city 列的对应 granule（解压、跳过 mark）
       - 向量化执行 city = 'Beijing' 过滤
       - 在每个 granule 上 count()
   ④ 接收节点把每个 shard 的局部 count 加起来
   ⑤ 返回客户端
```

每一步都做了 IO / CPU 的最大压缩。这就是 "为什么 ClickHouse 这么快" 的工程答案。

### 1.7 物化视图：插入触发的增量计算

ClickHouse 的物化视图（Materialized View）**不是定时跑的**，而是 INSERT 触发的。

```sql
-- 主表
CREATE TABLE events (...) ENGINE = MergeTree ORDER BY ...;

-- 物化视图：每秒/每分钟的 PV 聚合
CREATE MATERIALIZED VIEW events_pv_per_minute
ENGINE = SummingMergeTree
ORDER BY (minute, page)
AS SELECT
    toStartOfMinute(ts) AS minute,
    page,
    count() AS pv
FROM events
GROUP BY minute, page;
```

行为：
- 每次 `INSERT INTO events ...`，ClickHouse 用这批新数据**当成 SELECT 的输入**，跑一遍物化视图的 SELECT
- 结果作为一条新 INSERT 写入物化视图的存储表
- 历史数据不会自动 backfill

这就是 "增量" 的含义：**只对新写入的批次做计算，不重算历史**。

> **口诀**：物化视图 = INSERT 触发器 + 独立存储表。它**不能事务保证主表和视图的一致性**（案例 ④）。

---

## 2. 源码切片

> 基于 ClickHouse 24.x 主干源码。注释为本文添加。

### 2.1 MergeTreeData::write 的核心

```cpp
// src/Storages/MergeTree/MergeTreeDataWriter.cpp 简化
MergeTreeData::MutableDataPartPtr MergeTreeDataWriter::writeTempPart(
    BlockWithPartition & block_with_partition,
    const StorageMetadataPtr & metadata_snapshot,
    ContextPtr context)
{
    Block & block = block_with_partition.block;

    // ① 按 ORDER BY 排序（这是 part 内部有序的根本）
    auto sort_description = getSortDescription(metadata_snapshot);
    if (!isAlreadySorted(block, sort_description))
        stableSortBlock(block, sort_description);

    // ② 创建 part 的目录、初始化 columns.txt 等元数据
    auto new_data_part = data.createPart(part_name, ...);
    new_data_part->setColumns(columns, infos);

    // ③ 真正写列文件
    auto out = std::make_unique<MergedBlockOutputStream>(
        new_data_part, metadata_snapshot, columns,
        compression_codec, ...);
    out->write(block);
    out->writeSuffixAndFinalizePart(new_data_part);

    return new_data_part;
}
```

**逐段点透**：

- **① 必须排序**：part 内部按 ORDER BY 有序是 ClickHouse 整套查询机制的前提。primary.idx 才能用二分；merge 时才能用归并。如果你 INSERT 的数据已经按 ORDER BY 有序，能省掉一次大开销 —— 这就是 "按时间顺序导入数据更快" 的底层原因。
- **② 一次 INSERT 一个 part**：这就是为什么频繁小 INSERT 会导致 part 爆炸的根。
- **③ MergedBlockOutputStream**：每列独立的 stream，独立的压缩 codec。LZ4 / ZSTD 在这里被调用。

### 2.2 PartLog 与 part 元数据

```sql
-- 系统表 system.parts，每个 part 一条记录
SELECT
    partition,
    name,
    rows,
    bytes_on_disk,
    data_compressed_bytes,
    data_uncompressed_bytes,
    marks,
    level,
    active
FROM system.parts
WHERE table = 'events'
ORDER BY modification_time DESC LIMIT 5;
```

```
┌─partition─┬─name──────────────┬────rows─┬─bytes_on_disk─┬─compressed─┬─uncompressed─┬──marks─┬─level─┬─active─┐
│ 20251101  │ 20251101_1_120_2  │ 9830400 │   1.2 GiB     │   1.2 GiB  │   8.4 GiB    │   1200 │   2   │   1    │
│ 20251101  │ 20251101_121_140_1│  819200 │  120 MiB      │  120 MiB   │  720 MiB     │    100 │   1   │   1    │
└───────────┴───────────────────┴─────────┴───────────────┴────────────┴──────────────┴────────┴───────┴────────┘
```

观察：
- `level` 越大表示 merge 过越多次
- `active = 1` 才是 "对查询可见" 的 part；`active = 0` 是已合并但还没物理删除的旧 part
- `data_compressed_bytes / data_uncompressed_bytes` 就是这一 part 的压缩比

`system.part_log` 记录每次 part 操作的事件（创建、合并、删除、下载），是排查问题的金矿：

```sql
SELECT event_type, event_time, part_name, rows, error, exception
FROM system.part_log
WHERE table = 'events' AND event_time > now() - 3600
ORDER BY event_time DESC LIMIT 20;
```

### 2.3 MergeTask 后台合并

```cpp
// src/Storages/MergeTree/MergeTask.cpp 极度简化
bool MergeTask::executeImpl()
{
    // ① 选出要合并的 parts（小的优先合并）
    auto parts = selectPartsToMerge(...);
    if (parts.empty()) return false;

    // ② 估算合并后的 part 大小，预占磁盘
    reserveSpace(...);

    // ③ 用归并排序合并多个有序 part
    //    每个 part 内部已经按 ORDER BY 有序
    //    多路归并 → 输出仍然按 ORDER BY 有序
    auto merger = std::make_shared<MergingSortedTransform>(
        sort_description, parts);

    // ④ 对于 ReplacingMergeTree / SummingMergeTree 等
    //    在这里执行去重 / 求和 / 折叠 ★
    auto specialized = wrapMergeForEngine(merger, engine_type);

    // ⑤ 输出到新 part
    while (auto block = specialized->read())
        merged_part_writer->write(block);

    // ⑥ 提交：原子地用新 part 替换旧 parts
    commitPart(new_part, source_parts);
    return true;
}
```

**逐段点透**：

- **③ 多路归并**：part 内部有序 + 归并 → 合并后仍有序。这保证了无论 merge 多少次，主键索引永远有效。
- **④ 引擎特殊逻辑在 merge 时执行**：这是 "ReplacingMergeTree 不能保证 SELECT 立即去重" 的根。SELECT 看到的是 **part 内已经合并完成 + part 之间未合并** 的混合状态。
- **⑥ 原子提交**：merge 完成前，旧 parts 仍然 `active = 1`、新 part 不可见。提交瞬间通过文件系统 rename + 内存元数据切换实现原子。

### 2.4 PrimaryKey 二分查找

```cpp
// src/Storages/MergeTree/MergeTreeIndexReader.cpp 简化
MarkRanges MergeTreeIndexReader::filterMarkRanges(
    const KeyCondition & key_condition,
    const MarkRanges & ranges)
{
    MarkRanges result;
    for (const auto & range : ranges) {
        // 在 [begin, end] 这段 marks 范围内做二分
        size_t left = range.begin;
        size_t right = range.end;

        while (left < right) {
            size_t mid = (left + right) / 2;
            // 取 primary.idx 中第 mid 条索引项作为这块的边界
            auto index_value = primary_index[mid];
            if (key_condition.mayBeTrueInRange(index_value, ...))
                result.add(MarkRange{mid, mid + 1});
            // 二分缩范围
            ...
        }
    }
    return result;
}
```

**逐段点透**：

- **mayBeTrueInRange**：这个名字精确反映了稀疏索引的本质 —— 只能告诉你 "这一段可能有匹配的行"，不能告诉你具体行。
- 命中之后**整个 granule（默认 8192 行）都要读**，再在内存里精确过滤。所以 ClickHouse 的查询不适合 "查单条记录"（没有 hash 索引或 B+ 树那种 O(log n) 精确定位）。
- **这就是为什么 PRIMARY KEY 不是唯一约束**：它只是用来构建稀疏索引的排序键。多条相同主键的行可以并存。

### 2.5 Distributed::executeQuery

```cpp
// src/Storages/StorageDistributed.cpp 极度简化
QueryPipeline StorageDistributed::read(...)
{
    // ① 解析 cluster 配置：哪些 shard、每个 shard 哪个副本可用
    auto cluster = context->getCluster(cluster_name);

    // ② 把 SELECT 改写成针对本地表的查询
    //    SELECT count() FROM events_dist
    //    → SELECT count() FROM events_local 发到每个 shard
    auto modified_query_ast = rewriteQueryForLocalTable(query, ...);

    // ③ 并行向所有 shard 发请求
    auto pipes = std::vector<Pipe>{};
    for (const auto & shard : cluster->getShardsInfo()) {
        // 选一个 healthy replica
        auto connection = pickReplicaWithLoadBalancing(shard, ...);
        // 发出去
        pipes.emplace_back(createRemoteSourcePipe(
            connection, modified_query_ast, ...));
    }

    // ④ 把每个 shard 的结果做局部聚合 → 接收节点再做最终聚合
    return mergeShardResults(pipes);
}
```

**逐段点透**：

- **③ 并行扇出**：N 个 shard 并行查，理论上 N 倍加速。但**有个隐藏代价**：每个 shard 都要返回中间结果给接收节点。如果中间结果很大（比如 `GROUP BY user_id` 但 user_id 基数千万），接收节点的内存可能爆炸 —— 这是案例 ⑨。
- **④ 两阶段聚合**：每个 shard 先做局部聚合（`AggregatingTransform`），再把局部状态发给接收节点合并。这是 ClickHouse 比 "把数据全拉过来再聚合" 快的关键。

### 2.6 Aggregator 的向量化聚合

```cpp
// src/Interpreters/Aggregator.cpp 极度简化
void Aggregator::executeImpl(
    AggregatedDataVariants & result,
    const ColumnRawPtrs & key_columns,
    const ColumnRawPtrs & aggregate_columns,
    size_t rows)
{
    // ① 选择最优的哈希表实现（根据 key 类型 + 基数）
    //    单 key int → AggregationMethodOneNumber
    //    多 key string → AggregationMethodKeysFixed
    //    高基数 → 两阶段哈希（two-level hash）
    auto & method = result.aggregation_method;

    // ② 一次处理一批行（向量化）
    typename Method::State state(key_columns, ...);
    for (size_t i = 0; i < rows; ++i) {
        // 计算 key 的 hash 并查 / 插入哈希表
        auto emplace_result = state.emplaceKey(method.data, i, ...);

        // 在对应的聚合状态上累加
        for (size_t col = 0; col < aggregate_columns.size(); ++col)
            aggregate_functions[col]->add(
                emplace_result.iterator->getMapped() + offsets[col],
                aggregate_columns[col], i, ...);
    }
}
```

**逐段点透**：

- **① 哈希表选择**：ClickHouse 有十几种 hash table 实现，根据 key 类型和数据规模动态选最快的。`two-level hash` 是高基数的关键 —— 把 hash table 拆成 256 个子表，并行 merge 时无锁。
- **② 一行一行 add**：但每次 `add()` 都是高度 inline 的、无虚函数调用的（通过模板特化）。这就是向量化执行的具体表现。

### 2.7 物化视图的 attachToTable

```cpp
// src/Storages/StorageMaterializedView.cpp 极度简化
void StorageMaterializedView::pushBlock(const Block & block, ...)
{
    // 物化视图被注册成主表的 "下游"，主表 INSERT 时调用这里
    auto select_context = createContextWithBlock(block);

    // 用 INSERT 进来的 block 当输入，跑物化视图的 SELECT
    auto interpreter = std::make_unique<InterpreterSelectQuery>(
        view_query, select_context, ...);
    auto pipeline = interpreter->execute();

    // 把 SELECT 的结果 INSERT 到物化视图的存储表（通常是 .inner.* 表）
    auto insert = std::make_unique<InterpreterInsertQuery>(
        insert_query_for_inner_table, ...);
    insert->execute().pipeline.run();
}
```

**逐段点透**：

- **block 是 INSERT 进来的那批数据**：所以物化视图的 SELECT 看到的不是全表，是 "刚 INSERT 进来的这一批"。这就是 "增量" 的实现。
- **失败处理是关键**：默认情况下，物化视图的 INSERT 失败**不会**回滚主表的 INSERT，主表已经写成功了。这就是案例 ④ 的根因。

---

## 3. 实测数据

### 3.1 列存 vs 行存的查询对比

测试场景：1 亿行 `events` 表（30 列），查询 `SELECT city, count() FROM events WHERE date = '2025-11-01' GROUP BY city`。

机器：8 核 / 32GB / NVMe SSD。

| DB | 表大小（磁盘） | 查询耗时 | 扫描行数 |
| --- | --- | --- | --- |
| MySQL（无索引） | 28 GB | 145 s | 1 亿 |
| MySQL（date 索引） | 30 GB | 18 s | 200 万（按 date） |
| **ClickHouse**（按 date 分区） | **3.2 GB** | **0.18 s** | 200 万（仅扫 date 分区） |

ClickHouse 表大小**小一个数量级**（压缩），查询快**100 倍**。这不是单一因素的胜利，是 IO + 压缩 + 向量化叠加的结果。

### 3.2 不同压缩算法的 CPU vs 压缩比

ClickHouse 支持 `LZ4`（默认）、`ZSTD`、`Delta + LZ4` 等。同一份 1 亿行表的对比：

| codec | 表大小 | INSERT 速度 | SELECT 速度（冷启动） |
| --- | --- | --- | --- |
| `NONE` | 11 GB | 180 MB/s | 230 ms（IO bound） |
| `LZ4`（默认） | 3.2 GB | 165 MB/s | 180 ms |
| `ZSTD(1)` | 2.4 GB | 130 MB/s | 175 ms |
| `ZSTD(9)` | 1.9 GB | 70 MB/s | 175 ms |
| `Delta, ZSTD(3)`（适合时间序列） | 1.6 GB | 110 MB/s | 165 ms |

观察：
- LZ4 是 **CPU 极快、压缩比一般** 的甜点
- ZSTD 在冷查询场景占优势（IO 节省超过解压成本）
- 时间戳 / 单调递增列上加 `Delta` 编码差异巨大

> **口诀**：**默认用 LZ4**；**冷数据 / 历史分区可改 ZSTD**；**时间序列加 Delta**。

### 3.3 PRIMARY KEY 选择对查询性能

同一表，不同的 ORDER BY 方案，查询 `WHERE user_id = 12345 AND date = '2025-11-01'`：

| ORDER BY | primary.idx 大小 | 查询耗时 | 扫描行数 |
| --- | --- | --- | --- |
| `(date)` | 12 KB | 850 ms | 200 万（一整天） |
| `(user_id)` | 12 KB | 65 ms | 8192（一个 granule） |
| `(date, user_id)` | 12 KB | 60 ms | 8192 |
| `(user_id, date)` | 12 KB | 65 ms | 8192 |

观察：
- 主键里**有 user_id** 的方案，查询快 10+ 倍
- `(date, user_id)` 和 `(user_id, date)` 区别不大 —— 因为已经按分区限定了 date，主键内 date 在前在后差异不大

> **口诀**：**ORDER BY 的第一列 = 你最常 WHERE 的列**。如果有多个高频 WHERE 列，按基数从低到高排（也有反对意见，工程上多测试）。

### 3.4 不同 MergeTree 变体的写入吞吐

同一份数据写入：

| 引擎 | 写入吞吐（行/s） | merge 后磁盘 | 备注 |
| --- | --- | --- | --- |
| `MergeTree` | 350 万 | 12 GB | 无去重 |
| `ReplacingMergeTree` | 340 万 | 12 GB（merge 前）→ 8 GB | merge 时去重，慢 |
| `SummingMergeTree` | 330 万 | 12 GB → 4 GB | 数值列 sum |
| `AggregatingMergeTree`（含 -State） | 280 万 | 12 GB → 5 GB | 聚合状态体积大 |

观察：
- 写入差异不大（写入路径基本一样）
- merge 后差异显著，看你的数据重复 / 聚合特征

### 3.5 分布式查询的扇出代价

3 shard 集群，查询 `SELECT user_id, count() FROM events_dist GROUP BY user_id LIMIT 100`：

| user_id 基数 | 中间结果体积（每 shard 返回） | 查询耗时 |
| --- | --- | --- |
| 1 万 | 200 KB × 3 | 0.4 s |
| 100 万 | 20 MB × 3 | 1.8 s |
| **1 亿** | **2 GB × 3** | **OOM 接收节点** |

高基数 GROUP BY 在分布式场景**特别危险**。解决方案：
- 用 `GROUP BY user_id WITH ROLLUP` 之类减少基数
- 用 `distributed_group_by_no_merge = 0` 关掉合并（但 LIMIT 语义就不对了）
- 改写：先在每 shard 取 TOP N 再合并

> **口诀**：分布式 GROUP BY，**关注的不是 SELECT 行数，而是 GROUP BY 字段的基数**。基数 = 中间结果行数。

---

## 4. 生产案例 9 则

### 案例 ① 误以为 PRIMARY KEY 是唯一约束 —— "同一订单出现两次"

#### 现象

业务报表里某个订单 ID 出现了两次，金额翻倍。直接 `SELECT * FROM orders WHERE order_id = 'O123'` 返回 2 条记录。

但同事检查 DDL：

```sql
CREATE TABLE orders (
    order_id String,
    user_id  UInt64,
    amount   Decimal(18, 2),
    ts       DateTime
) ENGINE = MergeTree
PARTITION BY toYYYYMM(ts)
ORDER BY (user_id, order_id);
```

`ORDER BY (user_id, order_id)` —— 主键不就是 `order_id` 吗？怎么能重复？

#### 根因

ClickHouse 的 `PRIMARY KEY` / `ORDER BY` **不是 MySQL 那种唯一约束**。它的语义是 "主键索引使用的排序键"，**不阻止重复**。

ClickHouse 不做 INSERT 时去重的原因：
- 列存数据库的设计目标是 "高吞吐写入 + 高吞吐查询"，不是 "单行 CRUD"
- 写入时去重要查全表，违背列存哲学

那 ReplacingMergeTree 呢？它的去重是在 **后台 merge** 时按主键去重 —— **merge 前 SELECT 仍然能看到重复行**。

业务上这条订单可能是因为：
- 上游 Kafka 重复消费
- 应用重试时没幂等
- 跨 shard 路由错误，同一订单写到不同 shard

#### 修复

**方案 A**：用 `ReplacingMergeTree` + `FINAL` 关键字。

```sql
CREATE TABLE orders (
    order_id String,
    user_id  UInt64,
    amount   Decimal(18, 2),
    ts       DateTime,
    version  UInt64               -- 版本号，更大的覆盖更小的
) ENGINE = ReplacingMergeTree(version)
PARTITION BY toYYYYMM(ts)
ORDER BY (order_id);

-- 查询时用 FINAL 强制实时去重
SELECT * FROM orders FINAL WHERE order_id = 'O123';
```

`FINAL` 让查询时即时合并，但**性能代价大**（要把每个 part 内的相同 key 行全找出来合并）。**只在小范围查询用**。

**方案 B**：业务层保证幂等。最稳，最快。在写入前用 Redis SET NX 检查，重复就跳过。

**方案 C**：定期跑 `OPTIMIZE TABLE orders FINAL`。强制后台立即 merge。但大表 `OPTIMIZE FINAL` 极慢、阻塞写、不推荐生产用。

**方案 D**（最推荐）：`SELECT` 时显式去重。

```sql
SELECT order_id, argMax(amount, version) AS amount
FROM orders
WHERE order_id = 'O123'
GROUP BY order_id;
```

`argMax(amount, version)` 选 version 最大的那条。这是 OLAP 场景标准做法。

#### 验证

```sql
-- 写入前后对比
SELECT order_id, count() AS dup
FROM orders
GROUP BY order_id
HAVING dup > 1
LIMIT 10;
```

> **口诀**：**ClickHouse 的 PRIMARY KEY 不是唯一约束**。ReplacingMergeTree 也只是 "终将去重"，不是 "立即去重"。SELECT 端必须用 `argMax` / `FINAL` / 业务幂等三选一。

---

### 案例 ② TTL 配置错误 —— "数据 '消失' 了"

#### 现象

业务发现 30 天前的数据查不到了。DBA 看 `system.parts` 没有那么早的分区。但他们没主动 DROP 过。

#### 根因

某次表结构变更时加了 TTL：

```sql
ALTER TABLE events MODIFY TTL ts + INTERVAL 30 DAY;
```

写错了！原本是想 "归档到冷盘"，结果 `TTL ... DELETE`（默认行为）= **直接删除**。

ClickHouse 的 TTL 语义：
- `TTL expr DELETE`（默认）→ part 完全过期就**直接删**
- `TTL expr TO DISK 'cold'` → 移到指定卷
- `TTL expr TO VOLUME 'archive'` → 移到指定 volume
- `TTL expr GROUP BY ... SET col = aggr(col)` → 聚合压缩

更阴险的是 **列级 TTL**：

```sql
ALTER TABLE events MODIFY COLUMN raw_payload String TTL ts + INTERVAL 7 DAY;
```

这个 `raw_payload` 列在 7 天后**被设为默认值（空字符串）**，行不删。看起来 "数据丢了" 但其实是某一列被清了。

#### 修复

**修复一：可观测性**。任何 TTL 变更必须走评审。

```sql
-- 检查表上的所有 TTL 规则
SELECT name, ttl_expression, partition_key, sorting_key
FROM system.tables
WHERE database = 'default' AND name = 'events';

-- 检查列级 TTL
DESCRIBE TABLE events SETTINGS describe_include_subcolumns = 1;
```

**修复二：误删之后的恢复**。

如果 part 还没被物理删除（默认 `old_parts_lifetime = 480` 秒），可以从 `detached/` 目录恢复：

```bash
ls /var/lib/clickhouse/data/db/events/detached/
# 看到 ignored_xxx_yyy 这种目录
```

```sql
-- 把 detached 的 part 恢复
ALTER TABLE events ATTACH PARTITION '202510';
```

如果已被物理清理，只能从备份恢复（你的备份策略最好用 `clickhouse-backup` 工具）。

**修复三：TTL 的安全使用**。

```sql
-- 推荐：先标记 TTL 但延后生效，留时间观察
ALTER TABLE events MODIFY TTL ts + INTERVAL 90 DAY;  -- 长一点

-- 推荐：TTL 配 TO DISK 而非 DELETE，删除靠手动 ALTER ... DROP PARTITION
ALTER TABLE events MODIFY TTL ts + INTERVAL 30 DAY TO DISK 'cold';
```

#### 验证

```sql
-- 看 part 的 TTL 信息
SELECT
    partition,
    name,
    rows,
    delete_ttl_info_min,
    delete_ttl_info_max,
    move_ttl_info.expression,
    move_ttl_info.min,
    move_ttl_info.max
FROM system.parts
WHERE table = 'events' AND active;
```

> **口诀**：TTL 默认是 DELETE，且**删除是异步且不可恢复**（除非在 `old_parts_lifetime` 内）。**生产 TTL 推荐用 TO DISK，删除靠手动 DROP PARTITION**。

---

### 案例 ③ part 数过多触发 "Too many parts" —— "INSERT 全部失败"

#### 现象

凌晨 2 点告警：所有 `INSERT INTO events` 都报错：

```
DB::Exception: Too many parts (300). Merges are processing significantly slower than inserts.
```

业务停摆。

#### 根因

业务侧把日志通过 Flink 写入 ClickHouse，每 100ms 一个 batch，每个 batch 几百行 → **每秒 ~10 个 INSERT** → 每秒 10 个 part → merge 跟不上 → part 数达到 300 上限。

`max_parts_in_total` / `parts_to_throw_insert` 是 ClickHouse 的保护机制。如果不限制：
- merge 线程被压垮
- 查询要扫的 part 越来越多 → 查询变慢
- 磁盘文件碎片暴涨

观测：

```sql
-- 哪些表 part 多
SELECT database, table, count() AS parts
FROM system.parts
WHERE active
GROUP BY database, table
ORDER BY parts DESC LIMIT 10;

-- 每分钟有多少 part 被创建
SELECT toStartOfMinute(event_time) AS minute, count()
FROM system.part_log
WHERE event_type = 'NewPart' AND event_time > now() - 3600
GROUP BY minute
ORDER BY minute;
```

#### 修复

**根本方案**：**改写入模式**。

```
✗ 错：每 100ms 一批，几百行
✓ 对：每 5~10s 一批，几万~几十万行
```

具体方法：
- Flink/Kafka 消费方加 buffer，攒够再写
- 或用 ClickHouse 的 `Buffer` 引擎做缓冲表

```sql
CREATE TABLE events_buffer AS events
ENGINE = Buffer(default, events,
    16,        -- 16 个并发缓冲层
    10, 100,   -- min / max time（秒）触发刷新
    10000, 1000000,    -- min / max rows
    10000000, 100000000);  -- min / max bytes

-- 业务写 events_buffer，Buffer 引擎自动批量刷到真实表
INSERT INTO events_buffer VALUES (...);
```

但 Buffer 引擎有局限（节点重启数据丢失），生产更推荐 Kafka + clickhouse-kafka-connect 或 Flink CDC。

**临时缓解**：调大上限。

```sql
-- 临时调整（但治标不治本）
ALTER TABLE events MODIFY SETTING parts_to_throw_insert = 600;
ALTER TABLE events MODIFY SETTING max_parts_in_total = 200000;
```

**调优 merge 线程**：

```xml
<!-- config.xml -->
<background_pool_size>32</background_pool_size>          <!-- 默认 16 -->
<background_merges_mutations_concurrency_ratio>4</background_merges_mutations_concurrency_ratio>
```

#### 验证

```sql
-- 写入恢复后 part 数应回到几十
SELECT count() FROM system.parts WHERE table = 'events' AND active;

-- 单表平均 INSERT 大小（行数）
SELECT
    avg(rows) AS avg_rows_per_part,
    avg(bytes_on_disk) AS avg_bytes_per_part
FROM system.part_log
WHERE event_type = 'NewPart' AND table = 'events' AND event_time > now() - 600;
```

健康指标：单 part 至少 **几万行**，且**每秒 INSERT 数 < 5**。

> **口诀**：**ClickHouse 不是 Kafka**。批量 + 低频写入是设计假设。"Too many parts" 永远是写入模式问题，不是 ClickHouse 慢。

---

### 案例 ④ 物化视图写入失败但主表成功 —— "聚合和明细对不上"

#### 现象

报表组反馈：明细表（events）有 100 万条数据，物化视图聚合表（events_pv）算出来只有 99.7 万。**少了 0.3%**。

#### 根因

物化视图的 INSERT 是**主表 INSERT 的副作用**，但**默认不在事务里**。如果物化视图的 INSERT 失败（比如 schema 变更、磁盘满、目标表挂了），**主表已经写成功**，物化视图就丢了这一批。

ClickHouse 24.x 默认行为：

| 设置 | 默认 | 影响 |
| --- | --- | --- |
| `materialized_views_ignore_errors` | 0 | 视图写入失败抛错，但不回滚主表 |
| `parallel_view_processing` | 0 | 多视图串行写 |
| `optimize_on_insert` | 1 | 写入时尝试合并 |

注意：`materialized_views_ignore_errors = 0` 不代表 "主表回滚"。它只是 "把错误抛给客户端"。客户端如果不重试，数据就是丢了。

更隐蔽的问题：**视图 SELECT 的逻辑出错**也会导致丢数据。比如 SELECT 里用了 `assumeNotNull`，遇到 null 行就失败，那一批整批跳过。

#### 修复

**方案 A**：用 `INSERT INTO events SETTINGS insert_quorum = 'auto'` + 应用层重试。失败必须知道，重试必须幂等。

**方案 B**：监控物化视图的 lag。

```sql
-- 主表行数
SELECT count() FROM events WHERE date = today();

-- 视图行数（聚合后）
SELECT sum(pv) FROM events_pv WHERE minute >= toStartOfDay(today());

-- 两者按业务规则应该一致
```

把这个差值做成监控，每分钟跑。差超过阈值（比如 0.1%）告警。

**方案 C**：用 `system.errors` 和 `system.query_log` 找视图失败：

```sql
SELECT
    type,
    event_time,
    query_id,
    exception
FROM system.query_log
WHERE type = 'ExceptionWhileProcessing'
  AND query LIKE '%events_pv%'
  AND event_time > now() - 86400
ORDER BY event_time DESC LIMIT 20;
```

**方案 D**：最稳的架构 —— 用 Kafka 做缓冲，物化视图失败的数据回到 Kafka 重试。把 "物化视图" 改成 "下游消费者 + 单独的目标表"。

#### 验证

跑一周后对比明细表与聚合表。差异 < 业务容忍阈值才算修复。

> **口诀**：物化视图**不是事务保证**的。主表成功不代表视图成功。**生产场景必须监控 lag + 失败重试**。

---

### 案例 ⑤ 分布式 JOIN 性能崩塌 —— "数据没共置"

#### 现象

3 shard 集群，业务做 `events JOIN users ON events.user_id = users.user_id`，单 shard 测试 0.3s，分布式查询 28s。

#### 根因

ClickHouse 的分布式 JOIN 默认行为：**右表会被广播到所有节点**。

```sql
SELECT * FROM events_dist e JOIN users_dist u ON e.user_id = u.user_id
```

执行流程（错的方式）：
1. 接收节点先查 users_dist 拿到全部 users 数据 → 拉到接收节点
2. 把 users 表分发到所有 shard
3. 每个 shard 用本地 events JOIN 内存中的 users

如果 users 表 1 亿行 × 200 字节 = 20GB，光网络传输就要 20GB × 3 shard = 60GB。这就是 28s 的来源。

更严重的版本：右表大到内存放不下 → OOM。

#### 修复

**方案 A**：**数据共置**（co-location）。让 events 和 users 按相同的 sharding key 切片，且同 user_id 落同 shard。

```sql
-- events_dist
CREATE TABLE events_dist AS events_local
ENGINE = Distributed(cluster, default, events_local, user_id);

-- users_dist
CREATE TABLE users_dist AS users_local
ENGINE = Distributed(cluster, default, users_local, user_id);
```

然后查询用 `GLOBAL JOIN` 改成 **本地 JOIN**：

```sql
-- 直接 JOIN 本地表，不走 GLOBAL
SELECT * FROM events_dist e
JOIN users_local u ON e.user_id = u.user_id;
```

每个 shard 在本地完成 JOIN。前提：**业务保证同一 user 的 events 和 users 一定在同一 shard**。这是数据建模阶段就要决定的。

**方案 B**：右表小到能完全广播（Dictionary）。

```sql
CREATE DICTIONARY users_dict (
    user_id UInt64,
    name String,
    city String
) PRIMARY KEY user_id
SOURCE(MYSQL(...))
LAYOUT(HASHED())
LIFETIME(3600);

-- 用 dictGet 替代 JOIN
SELECT
    user_id,
    dictGet('users_dict', 'name', user_id) AS name,
    count()
FROM events_dist
GROUP BY user_id;
```

适合维度表（< 几千万行），常驻内存，访问 O(1)。

**方案 C**：用 `GLOBAL JOIN` 显式控制。

```sql
SELECT * FROM events_dist e GLOBAL JOIN users_dist u ON e.user_id = u.user_id;
```

`GLOBAL` 显式声明 "右表传给每个 shard"。比方案 A 慢但好理解。

#### 验证

```sql
EXPLAIN PIPELINE
SELECT * FROM events_dist e JOIN users_local u ON e.user_id = u.user_id;
```

观察是否有 `Distributed` 节点把右表往外发。本地 JOIN 不应该有这一步。

> **口诀**：分布式 JOIN 死穴 = **右表广播**。要么数据共置 + 本地 JOIN，要么右表用 Dictionary。**ClickHouse 不是 PostgreSQL，JOIN 永远是性能瓶颈**。

---

### 案例 ⑥ ZooKeeper / Keeper 性能瓶颈 —— "Replicated 表写入巨慢"

#### 现象

集群从 100 个 ReplicatedMergeTree 表扩到 500 个，INSERT 速度从 50 万行/s 跌到 5 万行/s。同时 ZooKeeper 出现大量 sync timeout 告警。

#### 根因

每个 ReplicatedMergeTree 表的每次 INSERT 都要在 ZK 上：
1. 在 `/clickhouse/tables/{shard}/{table}/replicas/{replica}/queue/queue-` 下创建 znode（注册 part）
2. 在 `/clickhouse/tables/{shard}/{table}/log/log-` 下创建 znode（同步日志）
3. 等其他副本拉取并 ack

500 表 × 每秒几次 INSERT = 每秒几千次 ZK 写入。原生 ZooKeeper 的写入吞吐瓶颈是 **几千 QPS**（不是查询，是写）。

ZK 还有更隐蔽的问题：
- ZK 的 snapshot 文件太大（几 GB） → 重启慢、内存压力大
- ZK 的 `jute.maxbuffer` 默认 1MB，超大 part 元信息会爆

#### 修复

**方案 A**：迁移到 **ClickHouse Keeper**。Keeper 是 ClickHouse 自研的 ZK 协议兼容实现，C++ 写、性能比 ZK 高 5~10 倍，吃内存少。

```xml
<!-- config.xml -->
<keeper_server>
    <tcp_port>9181</tcp_port>
    <server_id>1</server_id>
    <log_storage_path>/var/lib/clickhouse/coordination/log</log_storage_path>
    <snapshot_storage_path>/var/lib/clickhouse/coordination/snapshots</snapshot_storage_path>

    <coordination_settings>
        <operation_timeout_ms>10000</operation_timeout_ms>
        <session_timeout_ms>30000</session_timeout_ms>
        <raft_logs_level>information</raft_logs_level>
    </coordination_settings>

    <raft_configuration>
        <server>
            <id>1</id>
            <hostname>node1</hostname>
            <port>9234</port>
        </server>
        ...
    </raft_configuration>
</keeper_server>
```

ClickHouse Cloud 直接用 Keeper，没有 ZK。生产新部署强烈推荐 Keeper。

**方案 B**：减少 ZK 调用。

- 减少表数量（合表 + 用业务字段区分）
- 增大 INSERT 批次（之前一秒 10 次小 INSERT，改一秒 1 次大 INSERT，ZK 调用减 10 倍）
- 关掉不必要的 `insert_quorum`（默认 0 就好）

**方案 C**：ZK 不动的情况下调优。

```ini
# zoo.cfg
maxClientCnxns=2000
autopurge.snapRetainCount=5
autopurge.purgeInterval=1
preAllocSize=131072
snapCount=3000000

# JVM heap 8~16GB（不要更大，影响 GC）
```

#### 验证

```sql
-- ZK / Keeper 操作延迟（毫秒）
SELECT
    metric,
    value,
    description
FROM system.events
WHERE metric LIKE '%ZooKeeper%' OR metric LIKE '%Keeper%'
ORDER BY value DESC LIMIT 10;
```

也可以看 ZK 自己的 `mntr` 命令（`echo mntr | nc localhost 2181`）。

> **口诀**：**Replicated 集群 + 表多 = ZK/Keeper 瓶颈**。新部署直接 Keeper；老 ZK 集群优先减少表数量和 INSERT 频率。

---

### 案例 ⑦ 大查询打挂集群 —— "max_memory_usage 没配"

#### 现象

业务方临时跑一个 SQL 探查："`SELECT user_id, groupArray(event) FROM events GROUP BY user_id`"。半小时后整个 ClickHouse 节点 OOM 重启。所有正在跑的查询全部失败。

#### 根因

`groupArray(event)` 把每个 user 的所有 events 拼成一个 array。如果某个 user 有 100 万条 event 且每条 1KB，单个 user 就要 1GB 内存。1000 万 user × 大数组 → 几百 GB 内存。

ClickHouse 默认**没有强制的查询级内存限制**（社区版）。一个大查询能吃光整个节点的内存。

#### 修复

**方案 A**：**全局 / 用户级 / 查询级三道闸**。

```xml
<!-- users.xml -->
<profiles>
    <default>
        <!-- 单查询最大内存（5GB） -->
        <max_memory_usage>5000000000</max_memory_usage>

        <!-- 单用户所有查询合计最大内存（10GB） -->
        <max_memory_usage_for_user>10000000000</max_memory_usage_for_user>

        <!-- 服务器总内存上限（基于物理内存的百分比，0.9 = 90%） -->
        <max_server_memory_usage_to_ram_ratio>0.9</max_server_memory_usage_to_ram_ratio>

        <!-- 超内存怎么办？溢出到磁盘 -->
        <max_bytes_before_external_group_by>3000000000</max_bytes_before_external_group_by>
        <max_bytes_before_external_sort>3000000000</max_bytes_before_external_sort>

        <!-- 查询超时 -->
        <max_execution_time>300</max_execution_time>

        <!-- 限制并发 -->
        <max_concurrent_queries_for_user>20</max_concurrent_queries_for_user>
    </default>

    <!-- 给探查 / 临时查询专门一个低限的 profile -->
    <readonly>
        <readonly>1</readonly>
        <max_memory_usage>2000000000</max_memory_usage>
        <max_execution_time>60</max_execution_time>
        <max_concurrent_queries_for_user>5</max_concurrent_queries_for_user>
    </readonly>
</profiles>
```

**方案 B**：监控 + 自动 KILL 大查询。

```sql
-- 找超过 5 分钟的查询
SELECT
    query_id,
    user,
    elapsed,
    formatReadableSize(memory_usage) AS mem,
    substring(query, 1, 100) AS q
FROM system.processes
WHERE elapsed > 300
ORDER BY memory_usage DESC;

-- KILL
KILL QUERY WHERE query_id = 'xxx';
```

写个 cron 任务自动 KILL 跑超 30 分钟、内存超 10GB 的查询。

**方案 C**：用资源组（24.4+ 引入 Workload）。

```sql
CREATE WORKLOAD analytics IN default_resource (
    max_memory_usage = 5_000_000_000,
    max_concurrent_threads = 16
);
```

#### 验证

跑一个故意写错的大查询，应该能命中限制然后失败，而不是把节点打挂。

```sql
-- 这个会被内存限制截断
SELECT user_id, groupArray(event) FROM events GROUP BY user_id;

-- 错误信息应该是
-- DB::Exception: Memory limit (for query) exceeded: would use 5.21 GiB, ...
```

> **口诀**：**ClickHouse 单查询能吃光节点**。生产必须配 `max_memory_usage` + `max_execution_time` + `max_concurrent_queries_for_user`。**默认配置下集群是裸奔状态**。

---

### 案例 ⑧ ALTER 大表卡死 —— "改个字段类型整库不可用"

#### 现象

业务想把 `events.amount` 从 `Float64` 改成 `Decimal(18, 2)`：

```sql
ALTER TABLE events MODIFY COLUMN amount Decimal(18, 2);
```

跑了 10 分钟没返回，业务监控全红。

#### 根因

ClickHouse 的 ALTER 在很多情况下要**重写所有 part**：
- `MODIFY COLUMN`（类型变更）→ 重写
- `DROP COLUMN`（旧列）→ 重写删除该列文件
- 加 `INDEX` → 重新构建索引

重写期间每个 part 都要解压 → 转换 → 重新压缩 → 写新 part。100 亿行表可能跑几小时。

更严重的是：ALTER 时的 mutation 操作会**占用 merge 线程**。merge 跟不上 → "Too many parts" → 写入失败。

观察 mutation 进度：

```sql
SELECT
    database, table, mutation_id, command,
    parts_to_do, is_done,
    create_time, latest_fail_time
FROM system.mutations
WHERE NOT is_done
ORDER BY create_time;
```

#### 修复

**预防一**：永远先在小表 / 测试环境验证 ALTER。

**预防二**：评估 ALTER 类型。

```
✓ 元数据级 ALTER（秒级完成）：
  - ADD COLUMN（默认值）
  - RENAME COLUMN
  - 改 ORDER BY 的 ASC/DESC
  - 改 TTL 表达式

✗ 重写型 ALTER（小时级）：
  - MODIFY COLUMN（类型变更）
  - DROP COLUMN
  - 大批量 UPDATE（ALTER TABLE ... UPDATE）
  - DELETE（ALTER TABLE ... DELETE）
```

**预防三**：用 `Materialized` 列做 "在线类型变更"。

```sql
-- 加个新列（默认值是从旧列转换）
ALTER TABLE events ADD COLUMN amount_decimal Decimal(18, 2)
MATERIALIZED toDecimal64(amount, 2);

-- 后台慢慢 backfill 历史数据
ALTER TABLE events MATERIALIZE COLUMN amount_decimal IN PARTITION '202509';
ALTER TABLE events MATERIALIZE COLUMN amount_decimal IN PARTITION '202510';
-- ...

-- 业务切换到新列
-- 一段时间后 DROP 旧列
```

按分区做，每次只重写一个分区的部分 part，影响可控。

**事中**：取消 mutation。

```sql
KILL MUTATION WHERE mutation_id = 'xxx';
```

但已经处理过的 part 不会回滚，部分 part 类型新、部分类型旧。需要清理。

**长期**：用 Replicated 集群 + 滚动 ALTER。一个副本 ALTER，另一副本服务。

#### 验证

ALTER 期间监控：

```sql
SELECT
    parts_to_do_names,
    parts_to_do,
    is_done,
    latest_failed_part,
    latest_fail_reason
FROM system.mutations
WHERE table = 'events' AND NOT is_done;
```

`parts_to_do` 应该单调下降。如果不下降，看 `latest_fail_reason`。

> **口诀**：**ClickHouse 的 ALTER 不是 DDL，是 DML**。MODIFY COLUMN 在大表上 = 重写所有 part。生产分区滚动 + Materialized 列做平滑迁移。

---

### 案例 ⑨ 跨分片查询的内存放大

#### 现象

3 shard 集群，单 shard 跑 `SELECT user_id, count() FROM events_local GROUP BY user_id` 占内存 2GB。改成 `events_dist` 跑同样查询，**接收节点内存占到 18GB**。

#### 根因

分布式 GROUP BY 的执行：
1. 每个 shard 在本地做 GROUP BY，得到 `(user_id → count)` 的局部聚合结果
2. 把局部聚合结果发给接收节点
3. 接收节点把所有 shard 的结果**再做一次 GROUP BY 合并**

接收节点的内存 = 所有 shard 局部结果的总和。如果每个 shard 有 1 亿个不同 user_id，接收节点要持有 3 亿个 `(user_id, partial_count)` —— 远超单 shard 的 1 亿。

这就是 "分布式聚合内存放大" 现象。

#### 修复

**方案 A**：用 `distributed_aggregation_memory_efficient`。

```sql
SELECT user_id, count()
FROM events_dist
GROUP BY user_id
SETTINGS distributed_aggregation_memory_efficient = 1,
         max_threads = 8;
```

启用后，每个 shard 把局部结果**按 hash 分桶**发出，接收节点流式合并 —— 不需要一次性持有所有数据。代价是网络往返略多。

**方案 B**：减少 GROUP BY 基数。能用 `GROUP BY toStartOfHour(ts), city` 这种粗粒度就别 GROUP BY 高基数字段。

**方案 C**：用 `GROUP BY ... LIMIT BY`。

```sql
SELECT user_id, count()
FROM events_dist
GROUP BY user_id
ORDER BY count() DESC
LIMIT 100;
```

这种 "TOP N" 场景，每个 shard 只需返回自己的 TOP N，接收节点合并后再 TOP N。内存可控。

**方案 D**：用 `SAMPLE` 采样估算。

```sql
SELECT user_id, count() * 10 AS estimated_count
FROM events_dist SAMPLE 0.1
GROUP BY user_id;
```

10% 采样，结果误差可接受场景下，内存和耗时都降一个数量级。

#### 验证

```sql
-- 看接收节点的内存峰值
SELECT
    query_id,
    formatReadableSize(memory_usage) AS mem,
    formatReadableSize(peak_memory_usage) AS peak
FROM system.query_log
WHERE query LIKE '%events_dist%' AND type = 'QueryFinish'
ORDER BY peak_memory_usage DESC LIMIT 5;
```

> **口诀**：分布式 GROUP BY 的内存 = N × 单 shard 局部结果。**高基数 GROUP BY 必须开 `distributed_aggregation_memory_efficient`**。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 表设计层
- [ ] `PRIMARY KEY` / `ORDER BY` 的第一列是最常 WHERE 的列
- [ ] `PARTITION BY` 用时间字段（按天 / 按月，不要按小时 —— 分区数会爆）
- [ ] 分区数控制在几百以内，不要无限拆分
- [ ] 选对 MergeTree 变体（Replacing / Summing / Aggregating）并理解 "merge 时才生效"
- [ ] 高频写表必须 `Replicated*MergeTree`，不要单副本
- [ ] 列压缩选 LZ4（默认），冷数据 ZSTD，时间序列加 Delta

### 写入层
- [ ] **单次 INSERT 至少几千~几万行**
- [ ] **INSERT 频率每秒不超过几次**
- [ ] 业务方写 ClickHouse 必须经过 Kafka / Buffer 引擎缓冲
- [ ] 监控 `system.parts` 中每表 part 数 < 100
- [ ] 监控 `Too many parts` 错误，告警

### 查询层
- [ ] 用 Distributed 表查询，不要直接查 _local 表
- [ ] 大 GROUP BY 必须 `distributed_aggregation_memory_efficient = 1`
- [ ] JOIN 优先用 Dictionary，其次本地 JOIN（数据共置），实在不行用 GLOBAL JOIN
- [ ] 单查询配 `max_memory_usage`（5GB 起）
- [ ] 单用户配 `max_memory_usage_for_user`（10GB 起）
- [ ] `max_execution_time`（300s 起）
- [ ] `max_concurrent_queries_for_user`（20 起）

### 物化视图层
- [ ] 物化视图与主表的行数 / 聚合值定期对账
- [ ] 视图 SELECT 不用 `assumeNotNull` / 强类型转换（容易整批失败）
- [ ] 失败可重试架构（Kafka → 多消费者）

### TTL & 数据生命周期
- [ ] TTL 优先用 TO DISK，不要用 DELETE
- [ ] DROP PARTITION 必须人工审核
- [ ] 备份用 `clickhouse-backup`，每天全量 + 每小时增量

### 集群层
- [ ] 新集群用 ClickHouse Keeper，不用 ZooKeeper
- [ ] 监控 ZK / Keeper 的写入延迟（< 100ms）
- [ ] 副本数 ≥ 2（HA）
- [ ] `system.replication_queue` 队列不堆积（< 100）

### 监控层
- [ ] 接 Prometheus exporter
- [ ] 关键指标：QPS、慢查询、part 数、merge 队列长度、复制队列、ZK 延迟
- [ ] `system.errors` 表每天扫一次
- [ ] 慢查询日志（`log_queries = 1`，`log_query_threads = 1`）

---

## 6. 还能往下挖什么

### ClickHouse Cloud 与 Keeper
- ClickHouse Cloud 的存算分离架构（数据在对象存储，计算节点无状态）
- Keeper 替代 ZK 的工程价值：少一层运维、一致性更强、性能 5~10 倍
- Keeper 的 NuRaft（C++ 实现）vs ZK 的 ZAB 协议

### 与 Apache Pinot / Druid 的对比
- **Druid**：偏实时摄入（Kafka 强集成），架构复杂（Coordinator / Historical / Broker / MiddleManager 分离），适合实时仪表盘
- **Pinot**：LinkedIn 开源，类似 Druid 但更轻，星型 schema 友好
- **ClickHouse**：通用 OLAP，单节点性能极强，分布式架构简单
- 选型建议：通用 OLAP / 灵活 SQL / 离线 + 准实时 → ClickHouse；纯实时秒级 → Druid

### 数据湖（Iceberg / Delta Lake）+ ClickHouse
- ClickHouse 的 `Iceberg` / `DeltaLake` / `Hudi` 表引擎，可以直接查对象存储里的湖表
- 用法：ETL 后数据写到 Iceberg，ClickHouse 做查询层
- 价值：解耦存储和计算，多个引擎共享同一份数据（Spark 写、ClickHouse 读、Trino 联邦）

### Window Functions
- ClickHouse 22.3+ 支持完整的窗口函数（`ROW_NUMBER()` / `LAG` / `LEAD` / `RANK`）
- 但性能不如普通聚合 —— 因为要全量排序
- 大数据量时优先考虑 `arrayJoin` + `arrayEnumerate` 等数组技巧

### 流处理（ClickPipes / Materialized PostgreSQL）
- **ClickPipes**：Cloud 版的托管摄入服务（Kafka / Kinesis → CH）
- **Materialized PostgreSQL**：PG 的 logical replication 直接写 CH
- **Materialized MySQL**：MySQL binlog 直接写 CH（实时数仓核心组件）
- 这些把 ETL 简化到 "声明" 级别

### 实时数仓（OLAP 实时化）
- 传统 Lambda 架构（流 + 批）正在被 "OLAP 实时化" 替代
- ClickHouse + Kafka + 物化视图 = 端到端秒级延迟
- 与 Apache Doris、StarRocks 的同台竞争（这俩定位很像 ClickHouse，国内更流行）

### 与 Spark / Flink 的协作
- Spark 写 ClickHouse：用 `clickhouse-spark` connector，注意批量写 + JDBC 限制
- Flink → ClickHouse：用 `flink-clickhouse-sink`，sink 端做 batch 缓冲
- 反向：ClickHouse 数据导回 Hive / Iceberg 做 ML 训练

### HTTP 协议 vs Native 协议
- ClickHouse 的 Native 协议（TCP 9000）比 HTTP（8123）快 2~3 倍
- 但 HTTP 通用性强、跨防火墙易、便于调试
- 高吞吐 ETL 用 Native；BI 工具 / 临时查询用 HTTP

---

## 7. 引用

- ClickHouse 官方文档：[https://clickhouse.com/docs](https://clickhouse.com/docs)
- ClickHouse 源码：[github.com/ClickHouse/ClickHouse](https://github.com/ClickHouse/ClickHouse)
- 论文：*Column-oriented Database Systems*（VLDB Tutorial 2009）
- Lemire et al., *Decoding billions of integers per second through vectorization*（向量化经典论文）
- Yandex 原始 ClickHouse 设计文档（2016）
- 朱凯，《ClickHouse 原理解析与应用实践》—— 中文最系统的资料
- ClickHouse Meetup 视频集（YouTube ClickHouse 频道）
- Altinity Blog —— ClickHouse 商业公司，技术深度博客

---

> 配套：[Elasticsearch 深入](/article/db/elasticsearch) | [日志架构](/article/observability/logging) | [MySQL 索引深入](/article/db/mysql-index)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
