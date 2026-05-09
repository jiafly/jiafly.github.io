# Cassandra 深入 —— LSM 树、一致性级别与运维事故

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，看 [NoSQL 综述](/article/db/sharding) 就够了。
> 这里要回答的，是**线上 Cassandra 集群一致性配错、tombstone 把查询拖死、compaction 把磁盘撑爆**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. LSM 树（Log-Structured Merge-Tree）相比 B+ 树，**写**为什么快？代价是什么？为什么"读放大"和"空间放大"是它的天敌？
2. `CONSISTENCY = QUORUM` 的"过半"是怎么算出来的？三机房 RF=3 的 QUORUM 等于多少？跨机房写一次要等几个节点 ACK？
3. Cassandra 的 "Tunable Consistency"（可调一致性）号称 `R + W > N` 就强一致 —— 这个公式在网络分区时还成立吗？
4. Partition Key（分区键）和 Clustering Key（聚簇键）在物理上是怎么布局的？为什么"一个 Partition 不要超过 100MB"是铁律？
5. Read Repair（读修复）和 Hinted Handoff（提示移交）和 Anti-Entropy Repair（反熵修复）三者的工作时机和边界各是什么？
6. STCS、LCS、TWCS 三种 compaction（压实）策略，时序数据应该选哪个？为什么选错了会让磁盘 24 小时撑爆？
7. 为什么 Cassandra 的 `BATCH` 语句**不是为高性能设计的**？它的真实用途是什么？
8. ScyllaDB（C++ 重写的 Cassandra 兼容数据库）相比原版 Cassandra，吞吐能高 5~10 倍 —— 它做对了什么？为什么 Cassandra 自己不去抄？

如果有半数答不上，往下看。

---

## 1. 底层模型：从 Wide Column 到 LSM 树

绝大多数文章把 Cassandra 讲成"分布式 NoSQL，线性扩展"就完了。这层抽象解释不了上面 8 个问题。要看懂它们，必须下沉到 Cassandra 的**数据模型 + 一致性哈希 + LSM 写读路径 + 一致性协调**这四个核心层。

### 1.1 Wide Column 数据模型

Cassandra 的数据模型不是文档（如 MongoDB），不是 K-V（如 Redis），也不是关系（如 MySQL），而是 **Wide Column**（宽列）—— 一种"按 partition 分组、每组内是 sorted map"的二维结构。

```sql
CREATE TABLE sensor_data (
    sensor_id   text,        -- partition key（决定数据落在哪个节点）
    bucket      text,        -- partition key 的一部分（复合分区键）
    ts          timestamp,   -- clustering key（决定 partition 内的物理顺序）
    metric      text,        -- clustering key
    value       double,
    PRIMARY KEY ((sensor_id, bucket), ts, metric)
) WITH CLUSTERING ORDER BY (ts DESC, metric ASC);
```

- `((sensor_id, bucket))` 是 partition key —— 整个括号一起做 hash，决定落在 token ring 哪个位置
- `ts, metric` 是 clustering key —— 同一 partition 内的所有 row 按 (ts DESC, metric ASC) 排序存储

物理上，一个 partition 的所有 row 在磁盘上**连续存放**，这是 Cassandra "范围扫描快" 的根。

> **铁律**：partition 不要超过 **100MB / 10 万 row**。超过会引发"hot partition"、读放大、compaction 卡顿，全都是后面案例的素材。

### 1.2 一致性哈希 + Token Ring + vnode

Cassandra 用**一致性哈希**（consistent hashing）把数据分布到节点。Token 范围 `[-2^63, 2^63 - 1]`，每个节点负责一段。

```
       node A
      [-2^63, -2^62)
        ┌─────────┐
node D │           │ node B
[-2^62/4, 0)       │ [-2^62, -2^62/4)
        │           │
        └─────────┘
       node C
      [0, 2^63 - 1]
```

写入流程：
1. 客户端计算 `hash(partition_key)` → 得到 token T
2. 找到 token ring 上 T 顺时针下一个节点 → "primary replica"
3. 沿环继续找 RF（Replication Factor，复制因子）- 1 个节点，作为副本

**vnode（虚拟节点）的设计**：原始一致性哈希一个物理节点占一段 token，扩缩容时数据迁移不均衡。Cassandra 1.2+ 引入 vnode，**一个物理节点占 256 段（默认）**，分散在 token ring 各处。好处：
- 扩缩容时数据迁移更均匀
- 故障时多个节点同时帮忙补数据，恢复更快

代价：单 partition 的范围扫描可能跨多个 vnode → 读路径稍复杂。

### 1.3 LSM 树：MemTable → SSTable → Compaction

**LSM 树**（Log-Structured Merge-Tree）是 Cassandra 写入的灵魂。和 B+ 树原地更新不同，LSM 把写入**追加**到内存表，再批量刷到磁盘。

```
写入路径：
  ① WAL（CommitLog）顺序追加 —— 防丢
  ② MemTable（内存中的 sorted map，跳表实现） —— 写入即返回
  ③ MemTable 写满 → flush 成 SSTable（磁盘上的不可变文件）
  ④ 多个 SSTable 累积 → Compaction 合并成更大的 SSTable

读路径：
  Bloom Filter → KeyCache → Partition Index → SSTable 数据
              ↓
         合并 MemTable + 多个 SSTable 的版本，按 timestamp 取胜者
```

为什么 LSM 写快？
- **顺序写**：MemTable flush 是顺序写入，比 B+ 树的随机写快 10~100 倍
- **无锁定位**：MemTable 内部是 ConcurrentSkipListMap，写入几乎无锁竞争
- **不更新只追加**：UPDATE 实际是写一条新的、带新 timestamp 的 row，旧版本留在老 SSTable 里

代价：
- **读放大**：一次 read 可能要查 N 个 SSTable
- **空间放大**：旧版本数据没被合并前一直占空间
- **写放大**：Compaction 重写数据，一份数据可能在它的生命周期里被写 5~10 次

LSM 是 Cassandra / RocksDB / LevelDB / HBase / ScyllaDB 共同的底层。理解它就理解了一大票 NoSQL。

### 1.4 Read Path：四层缓存合奏

读一行数据，Cassandra 要做的事远比 B+ 树复杂：

```
client read
   │
   ▼
Coordinator 节点（接客户端的那个，不一定是数据所在节点）
   │  根据 CL（Consistency Level）联系若干 replica
   ▼
Replica 节点本地 read：
   ① Bloom Filter（每个 SSTable 一个，判断 key 可能在不在，假阳性 ~1%）
   ② Row Cache（默认关闭 —— 太占内存且容易污染）
   ③ Key Cache（partition key → SSTable 内偏移）
   ④ Partition Summary + Index（在磁盘上找到 partition 的精确位置）
   ⑤ MemTable + 命中的 SSTable 数据合并（按 timestamp 取最新）
```

**为什么默认关 Row Cache？** Cassandra 的典型场景是宽列（partition 内可能上万行），缓存整个 partition 太奢侈，且写入时要 invalidate 一整 partition，命中率低、维护贵。生产里**几乎从不开 Row Cache**。

**Bloom Filter 的代价**：每行需要约 `~10 bit` 的内存。10 亿行的表 → 1.25GB 堆内存。可以通过 `bloom_filter_fp_chance` 调假阳性率，假阳性高省内存但读放大。

### 1.5 Tunable Consistency：R + W > N 的真相

Cassandra 的招牌特性：**每个查询可以指定一致性级别**（CL）。

| CL | 含义 | 典型用途 |
| --- | --- | --- |
| `ONE` | 任意一个 replica ACK 即可 | 高吞吐 / 容忍少量不一致 |
| `TWO` / `THREE` | 至少两/三个 replica ACK | 中等强度 |
| `QUORUM` | 过半 replica ACK（`N/2 + 1`） | 最常用：`R + W > N` 实现强一致 |
| `LOCAL_QUORUM` | 本数据中心过半 | 跨 DC 部署的默认推荐 |
| `EACH_QUORUM` | 每个 DC 都过半 | 跨 DC 强一致（贵） |
| `ALL` | 所有 replica 都 ACK | 几乎不用：一个节点挂就写不进 |
| `ANY` | 包括 hinted handoff —— 任意节点收到就算成功 | 不要用，几乎没强度 |

**`R + W > N` 的真实含义**：如果你**写**等过半 ACK（W=2，N=3），**读**也等过半 ACK（R=2），那么读集合和写集合**至少有一个 replica 重叠**，所以总能读到最新值。

```
N = 3, W = 2, R = 2
写：node1, node2 ACK（node3 还没写到）
读：可能命中 {node1, node2}, {node1, node3}, {node2, node3}
任意两两组合至少包含 node1 或 node2 之一 → 强一致
```

**网络分区时的破裂**：如果客户端在分区期间用 `ONE` 写到少数派节点，多数派恢复后那次写就被覆盖。Cassandra 的"AP"特性正是这种时候放弃了强一致。

> **口诀**：要强一致用 **QUORUM 写 + QUORUM 读**；跨 DC 用 **LOCAL_QUORUM**；CL=ONE 写 + CL=ONE 读 是"最快也最不一致"组合，写 ALL / 读 ONE 在跨 DC 场景几乎不可用。

### 1.6 Hinted Handoff、Read Repair、Anti-Entropy Repair

Cassandra 用**三层修复**机制对抗 replica 之间的数据漂移。

**Hinted Handoff（提示移交）**：写入时如果某个 replica 暂时下线，coordinator 把这个写"暂存"在另一个节点（hint），等 replica 上线后**回放**。

- 默认保留 3 小时（`max_hint_window`）
- 超过窗口的 hint 直接丢弃 —— 必须靠后续 repair 弥补
- hint 占磁盘空间，节点长期下线时会爆

**Read Repair（读修复）**：CL > ONE 的读，coordinator 比对各 replica 的返回。
- 发现版本不一致 → **后台异步**把最新版本写回落后 replica
- 控制比例的参数：`read_repair_chance`（已废弃，5.0+ 改为 `read_repair`）
- 仅修复**被读到的 row**，没被读到的不会修

**Anti-Entropy Repair（反熵修复，nodetool repair）**：定期跑全量比对。
- 用 **Merkle Tree（默克尔树）** 比对 replica 之间的差异
- 只传输不一致的部分
- **必须定期跑**，否则删除的数据可能"复活"（见案例 ⑤）

```
Hinted Handoff：节点短时间下线（< 3h）
Read Repair：日常读触发的就近修复
Anti-Entropy Repair：每周/每月一次的完整修复
```

三者缺一不可。生产 Cassandra 没跑 repair = 定时炸弹。

### 1.7 GC Grace 与 Tombstone 的生命周期

每条 DELETE 在 Cassandra 里都不是真正的"删除"，而是写一个**带 timestamp 的墓碑**（tombstone）。墓碑必须存活足够久，让所有 replica 都同步到这次删除，否则会出现"删除复活"。

```
T = 0:    DELETE → 写一个 tombstone（cell 级 / row 级 / range 级 / partition 级）
T = 0~10d: tombstone 留在 SSTable 里，参与所有读（必须读出来才能判断"该不该展示"）
T = 10d:  gc_grace_seconds 过期，下一次 compaction 中 tombstone 被物理清理
```

四种 tombstone 粒度：
- **Cell tombstone**：删一列（`UPDATE t SET col = null WHERE ...`）
- **Row tombstone**：删一行（`DELETE FROM t WHERE pk = ? AND ck = ?`）
- **Range tombstone**：删一段 clustering 范围（`DELETE FROM t WHERE pk = ? AND ck > ?`）
- **Partition tombstone**：删整 partition（`DELETE FROM t WHERE pk = ?`）

**Range tombstone 是最容易被误用的**：一次 DELETE 写入一个 tombstone，但读时它**会与 partition 内每行 cell 比对**，扫 1 万行就要做 1 万次比对 → 慢得离谱（见案例 ①）。

> **口诀**：**tombstone 不是"删除"，是"延期清理标记"**。`gc_grace_seconds` 是 repair 的最低频率，不能调小除非 repair 跟得上。

### 1.8 三种 Compaction：STCS / LCS / TWCS

Compaction（压实）是 LSM 的"扫地僧"—— 把多个 SSTable 合并成一个，丢掉过期的 tombstone（墓碑）和被覆盖的旧版本。**选错策略是磁盘空间和读延迟的最大杀手**。

| 策略 | 全称 | 适用 | 写放大 | 读放大 | 空间放大 |
| --- | --- | --- | --- | --- | --- |
| **STCS** | Size-Tiered | 默认；写多读少 | 中 | 高 | **高（峰值 2x）** |
| **LCS** | Leveled | 读多写少；OLTP | **高（10x+）** | 低（1~2 个 SSTable） | 低 |
| **TWCS** | Time-Window | **时序数据** | 低 | 中 | 低 |

**STCS（Size-Tiered Compaction Strategy）**：相同大小的 SSTable 凑一组合并。
- 优点：写放大最低、CPU 消耗少
- 缺点：合并时**需要双倍磁盘**（旧文件还没删，新文件已生成）→ 磁盘必须留 50% 空闲

**LCS（Leveled Compaction Strategy）**：分层（L0 → L10），每层大小 10x，每层内 partition 不重叠。
- 优点：一次读最多查 1~2 个 SSTable
- 缺点：写放大严重 —— 一行数据可能在 LCS 里被重写 10 次

**TWCS（Time-Window Compaction Strategy）**：按时间窗口分组（每天/每小时一个窗口），窗口内用 STCS，跨窗口不合并。
- 专为时序数据设计：老数据冷藏不再合并，节省 IO
- 配合 TTL：整个窗口过期 → 直接删 SSTable 文件，无需 tombstone
- 选错（用 STCS 做时序）后果：tombstone 永远扫不完（见案例 ①）

> **口诀**：写多读少 STCS；读多写少 LCS；**时序数据必须 TWCS**。换 compaction 策略要重新生成所有 SSTable，是离线大动作。

---

## 2. 源码切片：六段关键路径

> 基于 Apache Cassandra 4.1 / 5.0 源码（Java）。注释为本文添加。

### 2.0 写入入口：Keyspace.apply() 的整体编排

```java
// org.apache.cassandra.db.Keyspace
public CompletableFuture<?> apply(Mutation mutation, boolean writeCommitLog,
                                  boolean updateIndexes, boolean isDroppable,
                                  boolean isDeferrable) {
    // ① 先写 CommitLog（顺序追加 + 可配置 fsync 策略）
    CompletableFuture<CommitLogPosition> clf = writeCommitLog
        ? CommitLog.instance.add(mutation)
        : CompletableFuture.completedFuture(CommitLogPosition.NONE);

    // ② CommitLog 落盘后才 apply 到各 ColumnFamilyStore（MemTable）
    return clf.thenCompose(position -> {
        OpOrder.Group opGroup = writeOrder.start();
        try {
            for (PartitionUpdate upd : mutation.getPartitionUpdates()) {
                ColumnFamilyStore cfs = getColumnFamilyStore(upd.metadata().id);
                cfs.apply(upd, indexer, opGroup, position);
            }
            return CompletableFuture.completedFuture(null);
        } finally {
            opGroup.close();
        }
    });
}
```

**逐段点透**：

- **CommitLog 是写入耐久性的唯一保证**。MemTable 在节点崩溃时丢失，靠 CommitLog 重放恢复。
- **`commitlog_sync = batch`** 时 `CommitLog.add()` 返回的 future 在 fsync 后才完成 → 写入吞吐受 fsync 影响（千 ops/s 量级）。
- **`commitlog_sync = periodic`**（默认）每 `commitlog_sync_period_in_ms`（默认 10s）批量 fsync → 写入立即返回 → 节点 kill -9 可能丢最近 10s 的写入。生产几乎都用 periodic。
- **`OpOrder.Group`** 是 Cassandra 自研的"无锁屏障"，让 MemTable flush 时能精确知道"哪些写入已完成、可以 flush"。

### 2.1 ColumnFamilyStore.apply() —— 写入路径

```java
// org.apache.cassandra.db.ColumnFamilyStore
public CompletableFuture<CommitLogPosition> apply(PartitionUpdate update,
                                                  UpdateTransaction indexer,
                                                  OpOrder.Group opGroup,
                                                  CommitLogPosition commitLogPosition) {
    // ① 拿当前活跃的 MemTable
    Memtable mt = data.getMemtableFor(opGroup, commitLogPosition);

    // ② 写入 MemTable（线程安全的 ConcurrentSkipListMap）
    long timeDelta = mt.put(update, indexer, opGroup);

    // ③ 触发 metric 更新
    metric.samplers.get(SamplerType.WRITES).addSample(...);

    // ④ MemTable 满了 → 异步触发 flush
    maybeSwitchMemtable();

    return CompletableFuture.completedFuture(commitLogPosition);
}
```

**逐段点透**：

- **①、②** 写入 MemTable 是**几乎无锁**的 —— `ConcurrentSkipListMap` 是 lock-free。这是 Cassandra 写入吞吐高的根本原因。
- **CommitLog 在哪里写？** 在 `apply()` 之前，由 `Keyspace.apply()` 先调用 `CommitLog.add()` 顺序追加，**写完 CommitLog 才写 MemTable**。CommitLog 的 fsync 策略（`commitlog_sync`）决定了"丢数据"窗口。
- **`maybeSwitchMemtable()` 的触发**：基于 MemTable 的 heap 大小（`memtable_heap_space_in_mb`）+ offheap（`memtable_offheap_space_in_mb`）超阈值时切换。**flush 是异步的**，写入不会同步阻塞。
- **`commitlog_sync = batch`** vs **`periodic`**：batch 模式每次写都 fsync（强持久但慢，2ms+/写）；periodic 默认每 10s fsync（快但崩溃丢 10s 数据）。

### 2.2 SSTable 的索引结构 —— Bloom Filter + Partition Index

```java
// org.apache.cassandra.io.sstable.format.SSTableReader （简化）
public class SSTableReader {
    private IFilter bloomFilter;          // 整张表一个 BF
    private IndexSummary indexSummary;    // 内存中的 partition 索引摘要
    private File indexFile;               // 磁盘上的 partition index 文件
    private File dataFile;                // 实际数据

    public RowIndexEntry getPosition(DecoratedKey key, Operator op) {
        // ① Bloom Filter 快速排除
        if (!bloomFilter.isPresent(key)) {
            bloomFilterTracker.addFalsePositive(); // 实际是 true negative
            return null;
        }

        // ② Key Cache 命中？
        KeyCacheKey cacheKey = new KeyCacheKey(metadata, descriptor, key.getKey());
        RowIndexEntry cached = keyCache.get(cacheKey);
        if (cached != null) return cached;

        // ③ IndexSummary 二分定位 partition index 文件中的范围
        long sampledPosition = indexSummary.getScanPosition(key);

        // ④ 在 index 文件中线性扫描一小段，找到精确 entry
        try (FileDataInput in = indexFile.createReader(sampledPosition)) {
            while (!in.isEOF()) {
                ByteBuffer indexKey = ByteBufferUtil.readWithShortLength(in);
                RowIndexEntry indexEntry = rowIndexEntrySerializer.deserialize(in);
                int cmp = decoratedKey.compareTo(indexKey);
                if (cmp == 0) {
                    keyCache.put(cacheKey, indexEntry);
                    return indexEntry;
                }
                if (cmp < 0) return null;
            }
        }
        return null;
    }
}
```

**逐段点透**：

- **Bloom Filter 假阳性**：`bloomFilter.isPresent` 返回 true 不代表真有，1% 的假阳性率会触发**无意义的索引查找**。10 亿次查询 → 1000 万次浪费的 IO。
- **IndexSummary 是 Partition Index 的"索引"**：磁盘上的 partition index 文件可能 GB 级，全装内存装不下，所以再做一层"采样索引"。
- **Key Cache 命中率**：典型生产 50~90%。命中直接拿到 SSTable 内偏移，省两次磁盘 IO。
- **整段没有锁**：SSTable 是不可变的，多线程读不需要任何同步。这是 LSM 设计的另一份红利。

### 2.3 CompactionManager —— compaction 调度

```java
// org.apache.cassandra.db.compaction.CompactionManager （简化）
public Future<?> submitBackground(final ColumnFamilyStore cfs) {
    if (cfs.isAutoCompactionDisabled()) return null;

    // ① 让 strategy 决定挑哪些 SSTable
    AbstractCompactionStrategy strategy = cfs.getCompactionStrategy();
    Collection<AbstractCompactionTask> tasks = strategy.getNextBackgroundTasks(getDefaultGcBefore(cfs));

    if (tasks.isEmpty()) return null;

    // ② 提交到 compaction executor（独立线程池）
    return executor.submitIfRunning(() -> {
        for (AbstractCompactionTask task : tasks) {
            try {
                task.execute(active);    // 真正的合并
            } catch (Throwable t) {
                logger.error("Compaction failed for {}", cfs.name, t);
            }
        }
    });
}
```

**逐段点透**：

- **策略决定"挑哪些"**：STCS 找大小相近的；LCS 找重叠的 partition 范围；TWCS 找同时间窗口的。这是策略的核心差异。
- **compaction executor 默认 2 线程**（`concurrent_compactors`）。生产 SSD 机器可以调高（4~8），但 CPU 占用线性上升。
- **compaction 的 IO 占用**：`compaction_throughput_mb_per_sec` 限速，默认 16 MB/s。**写入压力大时这值必须调高**（64+），否则 SSTable 数量爆炸 → 读放大暴涨。
- **GC grace 的作用**：`getDefaultGcBefore(cfs)` 决定 tombstone 多老才能在 compaction 中真正删除。默认 10 天 —— 等于给 repair 留窗口。

### 2.4 SinglePartitionReadCommand —— 读时多 SSTable 合并

```java
// org.apache.cassandra.db.SinglePartitionReadCommand （高度简化）
public UnfilteredPartitionIterator queryStorage(ColumnFamilyStore cfs, ReadExecutionController ec) {
    List<UnfilteredRowIterator> iterators = new ArrayList<>();

    // ① 从 MemTable 拿
    for (Memtable memtable : cfs.getTracker().getView().getAllMemtables()) {
        UnfilteredRowIterator iter = memtable.rowIterator(partitionKey, clusteringIndexFilter);
        if (iter != null) iterators.add(iter);
    }

    // ② 从可能命中的 SSTable 拿（用 BF 过滤）
    Iterable<SSTableReader> sstables = cfs.getTracker().getView().select(SSTableSet.LIVE);
    for (SSTableReader sstable : sstables) {
        if (!sstable.mayContainAssumingKeyIsInRange(partitionKey)) continue;
        UnfilteredRowIterator iter = sstable.iterator(partitionKey, clusteringIndexFilter);
        if (iter != null) iterators.add(iter);
    }

    // ③ 合并：相同 cell 取 timestamp 大的；tombstone 覆盖一切
    return UnfilteredPartitionIterators.mergeIterators(iterators, nowInSec, command);
}
```

**逐段点透**：

- **N 个 SSTable + 1 个 MemTable 全部归并**：这就是"读放大"。SSTable 数量 = 读延迟的主要因素。
- **`mayContainAssumingKeyIsInRange`** 内部是 BF + min/max key 检查。这一步过滤掉绝大多数不相关的 SSTable。
- **tombstone 在合并时起决定作用**：cell 的 tombstone（DELETE 标记）的 timestamp 比 cell 本身新 → cell 被"删除"，但**实际数据还在 SSTable 里**。这就是为什么 tombstone 多了会拖慢读 —— 它们必须被读出来才能判断"该不该展示给客户端"。
- **`Tombstone overwhelming queries` 异常**：如果一次 query 扫到 > `tombstone_failure_threshold`（默认 10 万）个 tombstone，直接报错。这是 Cassandra 给你的最后一道保险（见案例 ①）。

### 2.5 Gossip 协议 —— 节点发现与状态同步

```java
// org.apache.cassandra.gms.Gossiper （简化）
public class Gossiper {
    // 每秒执行一次的 task
    private class GossipTask implements Runnable {
        public void run() {
            // ① 更新自己的 heartbeat
            EndpointState localState = endpointStateMap.get(FBUtilities.getBroadcastAddressAndPort());
            localState.getHeartBeatState().updateHeartBeat();

            // ② 构造 SYN 消息：自己已知的所有节点的版本号摘要
            List<GossipDigest> digests = new ArrayList<>();
            for (Map.Entry<InetAddressAndPort, EndpointState> e : endpointStateMap.entrySet()) {
                digests.add(new GossipDigest(e.getKey(), generation, version));
            }

            // ③ 选一个 live 节点发 SYN
            InetAddressAndPort to = pickRandomLiveEndpoint();
            sendGossip(to, new GossipDigestSyn(clusterName, partitionerName, digests));

            // ④ 选一个 unreachable 节点发 SYN（让它有机会重新加入）
            sendGossipToUnreachable(...);

            // ⑤ 选一个 seed 节点发 SYN（保持种子连接）
            sendGossipToSeed(...);

            // ⑥ 检查节点是否疑似下线（Phi-accrual failure detector）
            for (InetAddressAndPort node : endpointStateMap.keySet()) {
                if (FailureDetector.instance.interpret(node) > phiConvictThreshold) {
                    convict(node);
                }
            }
        }
    }
}
```

**逐段点透**：

- **每秒一次** —— 一个 100 节点的集群，每秒每个节点发 3 条 gossip，全集群 300 条/秒。**集群规模上百时 gossip 流量值得监控**。
- **Phi-accrual failure detector**：不是简单的"超时即下线"，而是计算"过去 heartbeat 间隔分布"，给一个 phi 值（"我有多确信这节点死了"）。Phi > 8（`phi_convict_threshold`）才宣告死亡。比固定超时鲁棒得多。
- **versioned**：每个 EndpointState 带一个递增的 version。两个节点 gossip 时只交换 version 更新的部分，避免全量传输。
- **seed 节点的作用**：仅用于"新节点首次启动找入口"。**seed 不是 master**，运行时和其他节点平等。把所有节点都设成 seed 是常见误解（见案例 ⑥）。

### 2.6 Repair 的 Merkle Tree 构造与 diff

```java
// org.apache.cassandra.repair.Validator （简化）
public class Validator implements Runnable {
    private MerkleTrees trees;

    public void run() {
        // ① 扫描本节点 partition range 内的所有 row
        for (UnfilteredRowIterator partition : scanPartitions()) {
            // ② 计算每行的 hash（含 timestamp + cell value）
            long rowHash = computeRowHash(partition);
            trees.addHash(partition.partitionKey().getToken(), rowHash);
        }

        // ③ 把 Merkle Tree 发回 coordinator
        MessagingService.instance().sendOneWay(
            new ValidationComplete(desc, trees), coordinator);
    }
}

// 在 coordinator 上比对
public void diffMerkleTrees(MerkleTrees a, MerkleTrees b) {
    for (Range<Token> range : a.invalids(b)) {
        // 只对不一致的 range 触发数据流式同步
        streamMismatchedData(range, fromNode, toNode);
    }
}
```

**逐段点透**：

- **Merkle Tree 是哈希树**：叶子是若干 row 的 hash，内部节点是子节点的 hash。比对两棵树时**从根开始 diff**，相同就跳过整棵子树，不同才往下钻。复杂度 O(差异数 × log 总数)。
- **构造 Merkle Tree 必须扫全表**：100GB 数据 → 几个小时的全盘 IO。这就是为什么 repair 是"重活"。
- **subrange repair**：`nodetool repair -st <start> -et <end>` 可以只修一段范围。生产做法：用 [Reaper](https://github.com/thelastpickle/cassandra-reaper) 把全表切成 256 段，每天修一段，一年修两遍。
- **incremental repair**：4.0+ 提供，标记哪些 SSTable 已被 repair 过，下次 repair 只看新数据。**但 incremental repair 历史 bug 多**，4.x 才稳定，3.x 用过的人都被坑过。

---

## 3. 实测数据：compaction、CL、ScyllaDB

### 3.0 测试环境

下面所有数据来自相同硬件：
- 10 节点 EC2 i3.4xlarge（16 vCPU / 122GB RAM / 3.8TB NVMe）
- Cassandra 4.1.3，G1GC，31GB heap
- ScyllaDB 5.4 Open Source
- 测试工具：cassandra-stress 4.1 + 自研 sysbench-cassandra fork
- 数据集：10 亿 row，每行 ~100 字节，~100GB
- 客户端在独立机器，10 Gbps 网络

### 3.1 不同 Compaction 策略下的写放大与读放大

测试：10 节点集群，单节点 16 核 / 64GB / NVMe，sysbench-cassandra 风格压测，10 亿行 / 100 GB 数据集。

| 策略 | 写放大 | 读放大 | 空间放大 | P99 读延迟 |
| --- | --- | --- | --- | --- |
| **STCS** | 3.2x | 4.5 SSTable / read | **1.8x** | 18 ms |
| **LCS** | **11x** | 1.3 SSTable / read | 1.1x | **6 ms** |
| **TWCS**（1h 窗口） | 1.5x | 2.1 SSTable / read | 1.05x | 9 ms |

观察：
- LCS 读延迟最低，但**写放大 3 倍**于 STCS —— 写多场景磁盘 IO 提前到极限
- TWCS 综合最好，但前提是数据**带 TTL 且按时间写入**（时序数据）
- STCS "读 4.5 个 SSTable" 是平均，**长尾可能 20+** —— 这是 P99 高的原因

> **口诀**：写多读少选 STCS，读多写少选 LCS，时序数据**必须** TWCS。

### 3.2 一致性级别对延迟的影响

单 DC 5 节点 RF=3 集群，QPS = 10000：

| CL | P50 写延迟 | P99 写延迟 | P50 读延迟 | P99 读延迟 |
| --- | --- | --- | --- | --- |
| `ONE` | 1.2 ms | 4 ms | 1.5 ms | 5 ms |
| `LOCAL_QUORUM` | 2.0 ms | 8 ms | 2.5 ms | 10 ms |
| `ALL` | 3.5 ms | **80 ms** | 4.0 ms | **120 ms** |

观察：
- ONE → QUORUM 大约 **+50% 延迟**，但保证一致性
- QUORUM → ALL 看起来"只多一个节点"，但 P99 翻 10 倍 —— 任何一个节点慢就拖整体（**长尾放大**）
- 跨 DC 用 EACH_QUORUM 时延迟还会因为 DC 间 RTT 再涨 30~50ms

> **口诀**：默认 LOCAL_QUORUM；非关键路径降到 ONE 提速；几乎不用 ALL（一节点慢全栈慢）。

### 3.3 Cassandra vs ScyllaDB 吞吐对比

同样的硬件、同样的 schema、同样的 sysbench 负载：

| 指标 | Cassandra 4.1 | ScyllaDB 5.4 | 差距 |
| --- | --- | --- | --- |
| 写吞吐 (ops/s) | 95k | **520k** | 5.5x |
| 读吞吐 (ops/s) | 78k | **410k** | 5.3x |
| P99 写延迟 | 8 ms | **1.8 ms** | 4.4x |
| 单核效率 (ops/core) | 6k | **32k** | 5.3x |

为什么 ScyllaDB 这么猛？
- **C++ 重写**，无 GC 长尾
- **Seastar 框架** —— 每核一个 reactor，shared-nothing，无锁
- **DPDK / 用户态网络**（可选）
- **完全异步 IO**

Cassandra 自己为什么不抄？团队规模、生态绑定、JVM 历史包袱、向后兼容压力。**新项目可以直接上 ScyllaDB**，老项目要权衡迁移成本。

> **口诀**：ScyllaDB ≈ "C++ 重写、性能 5x 的 Cassandra"，wire 兼容（同一个 CQL 协议、同样的 driver）。

### 3.4 不同节点数下的扩缩容时间

加一个新节点到现有 N 节点集群，等数据 stream 完成 + bootstrap 完成：

| 集群规模 | 单节点数据量 | 加一个新节点耗时 |
| --- | --- | --- |
| 3 节点 → 4 节点 | 100 GB | ~25 分钟 |
| 10 节点 → 11 节点 | 100 GB | ~15 分钟 |
| 50 节点 → 51 节点 | 100 GB | ~10 分钟 |
| 100 节点 → 101 节点 | 100 GB | ~8 分钟 |

观察：**集群越大，加节点越快** —— 因为更多节点同时往新节点 stream 数据。这是 vnode 设计的红利。

但：缩容（`nodetool decommission`）反向走，**节点越多越慢** —— 一个节点要把数据 stream 给一堆新归属节点。

### 3.5 大量 tombstone 对查询的影响

测试场景：单 partition 中插入 10 万行，然后删除 9.9 万行，剩 1000 行有效数据。

```sql
SELECT * FROM tbl WHERE pk = 'p1';   -- 期望返回 1000 行
```

| Tombstone 数量 | 查询耗时 | 状态 |
| --- | --- | --- |
| 0 | 8 ms | ✅ 正常 |
| 1000 | 12 ms | ✅ |
| 10000 | 65 ms | ⚠️ 慢 |
| 50000 | 320 ms | ⚠️ 警告日志 |
| 100000 | 报错 `TombstoneOverwhelmingException` | ❌ 阻断 |

Cassandra 的两道阈值（`cassandra.yaml`）：
```yaml
tombstone_warn_threshold: 1000        # 超过记 warning 日志
tombstone_failure_threshold: 100000   # 超过直接报错并终止 query
```

> **口诀**：tombstone 是 LSM 删除的代价。**一次查询扫到 1000+ tombstone 必须报警**。改 schema 比改阈值靠谱。

### 3.6 Repair 在不同集群规模下的耗时

`nodetool repair -pr <keyspace>` 单节点修复耗时（10 亿 row，100GB / 节点）：

| 集群规模 | 全量 repair（每节点） | subrange repair（256 段，每段） |
| --- | --- | --- |
| 3 节点 | 4 小时 | 1 分钟 |
| 10 节点 | 5 小时 | 1.2 分钟 |
| 50 节点 | 6 小时 | 1.5 分钟 |
| 100 节点 | 8 小时 | 2 分钟 |

观察：
- 全量 repair 时间随集群规模上升 —— 因为 Merkle Tree 比对的对手节点更多
- subrange repair 单段时间相对稳定，可以**分散在一周内每天跑几段**
- **incremental repair** 在 4.x 后稳定，能让"已 repair 的数据不再扫描"，但首次仍要全量

> **口诀**：100 节点以上集群必须用 subrange + 调度器（Reaper），不要手工 nodetool repair 全量跑。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 大量 tombstone 导致查询超时 —— "TTL 数据用了 STCS"

#### 现象

时序数据表，写入用 TTL=7 天，业务每天读最近一小时数据。上线一个月后，简单 SELECT 偶发 30s 超时。监控看磁盘 IO 飙满。

#### 根因

```sql
CREATE TABLE metrics (
    sensor_id text,
    ts timestamp,
    value double,
    PRIMARY KEY (sensor_id, ts)
) WITH CLUSTERING ORDER BY (ts DESC)
  AND default_time_to_live = 604800;   -- 7 天
-- ★ 默认 compaction = STCS
```

机理串起来：
1. STCS 不感知"时间维度"，老数据和新数据混在一个 SSTable 里
2. TTL 到期后**不立即删除**，先变成 tombstone（墓碑）
3. tombstone 必须等 `gc_grace_seconds`（默认 10 天）后 compaction 才能真正清理
4. 一周的 TTL + 10 天 gc_grace = tombstone 在 SSTable 里待 17 天才被清
5. 读"最近一小时"也要扫过这堆历史 tombstone（因为它们和有效数据混在同一个 SSTable）

`nodetool tablestats metrics`：

```
Average tombstones per slice (last five minutes): 28453
Maximum tombstones per slice (last five minutes): 178293
```

每次读平均扫 2.8 万个 tombstone，最大 17.8 万 → 触发 `TombstoneOverwhelmingException`。

#### 修复

**方案 A**（推荐）：换 TWCS。

```sql
ALTER TABLE metrics WITH compaction = {
    'class': 'TimeWindowCompactionStrategy',
    'compaction_window_unit': 'HOURS',
    'compaction_window_size': 1
};
```

每小时一个时间窗口 → 7 天后整个窗口 SSTable 直接删除（不需要 tombstone 处理）→ 读"最近一小时"只扫一个 SSTable。

**方案 B**：缩短 `gc_grace_seconds`（前提：repair 频率配套）。

```sql
ALTER TABLE metrics WITH gc_grace_seconds = 86400;  -- 1 天
```

注意：**修改后必须每天跑 repair**，否则删除可能"复活"（见案例 ⑤）。

**方案 C**：单 partition 不要堆太多数据，加 bucket 分散：

```sql
PRIMARY KEY ((sensor_id, day_bucket), ts)
-- day_bucket = '2026-05-09'，每天一个新 partition
```

#### 验证

```bash
# 改完后等 1~2 个 compaction 周期，重新看
nodetool tablestats metrics | grep -i tombstone

# 监控指标
nodetool tablehistograms metrics
```

`Average tombstones per slice` 应降到 < 100。

> **口诀**：TTL 数据 + STCS = tombstone 地狱。**时序数据必须 TWCS**。Range tombstone 比单 row tombstone 更危险，因为它在读时与每行 cell 比对。

#### 延伸：unchecked_tombstone_compaction

5.0 引入 `unchecked_tombstone_compaction = true` 选项：让 compaction **跳过那些"过老但没法立刻删"的 tombstone**（因为参与的 SSTable 可能含有更老的版本）。配合 `tombstone_compaction_interval`（默认 1 天）和 `tombstone_threshold`（默认 0.2，即 SSTable 中 tombstone 占比 > 20% 时触发）使用，能让顽固 tombstone 提前清掉。代价是可能短暂出现"删除复活"，需要业务能容忍或确保 repair 跟得上。

---

### 案例 ② Hot Partition —— "单分区把节点打挂"

#### 现象

10 节点集群，监控发现某 2 个节点 CPU 100%，其他节点 30%。重启那两个节点后，几分钟又被打挂。

#### 根因

`nodetool tablestats` 看 partition 大小分布：

```
Compacted partition maximum bytes: 1879048192   # 1.79 GB ★
Compacted partition mean bytes: 4096
```

平均 4KB，最大**1.79GB** —— 一个 partition 占了整个集群一份数据的几个百分点。

业务表设计：

```sql
CREATE TABLE user_events (
    event_type text,    -- partition key （只有几十个值）
    event_id timeuuid,
    payload text,
    PRIMARY KEY (event_type, event_id)
);
```

`event_type` 只有几十个枚举值（"login", "click", "view"...）。其中 "click" 占了 60% 流量 → 所有 click 事件都打到同一个 partition → 该 partition 所在的 RF=3 个 replica 节点被打爆。

更糟：partition 1.79GB → 每次读这个 partition 的 compaction 也是 GB 级 IO → 磁盘和 CPU 双爆。

#### 修复

**方案 A**：拆 partition key —— 加 bucket。

```sql
CREATE TABLE user_events (
    event_type text,
    bucket int,         -- ★ 新增分桶
    event_id timeuuid,
    payload text,
    PRIMARY KEY ((event_type, bucket), event_id)
);
```

写入时 `bucket = hash(event_id) % 100` —— 把 1 个 partition 拆成 100 个 → 流量均匀。

读取时如果按 event_type 全扫，要并发查 100 个 partition（用 `IN` 或循环并发）。**这是 NoSQL 数据建模的经典权衡：写均衡 vs 读简单**。

**方案 B**：业务分流 —— click 这种热事件单独表，其他事件共用一表。

#### 监控

```bash
# 找 hot partition
nodetool toppartitions <keyspace> <table> 10000
```

每分钟跑一次，超过 100MB 的 partition 报警。

#### 验证

改完 schema、重新写入数据后，节点 CPU 应该均衡到 30~40% 一致；`nodetool tablestats` 的最大 partition 大小 < 100MB。

> **口诀**：partition key 必须**高基数**且**访问均匀**。基数低的字段（枚举、status）千万别做 partition key。

#### 延伸：how to detect hot partition early

不用等节点挂了再发现。两个早期信号：
- `nodetool tablehistograms <ks> <tbl>` 看 partition size 分布的 P99 / Max 列。Max 比 Mean 高 100x → hot partition 嫌疑
- `nodetool toppartitions <ks> <tbl> 5000` 实时采样，列出 5 秒内访问最多的 partition。生产建议每 5 分钟跑一次，超过阈值入告警

---

### 案例 ③ 一致性级别配错 —— "写 ONE 读 ONE，数据时有时无"

#### 现象

业务上线后用户偶尔反馈"刚下的订单刷新一下就没了"。日志里没报错，订单表数据也确实在 —— 但客户端某次查询返回的是空。

#### 根因

业务为了性能，全部用了 `CL=ONE`：

```python
session.execute("INSERT INTO orders ...", consistency_level=ConsistencyLevel.ONE)
session.execute("SELECT * FROM orders WHERE ...", consistency_level=ConsistencyLevel.ONE)
```

3 副本场景：
1. 写 ONE：coordinator 把请求发给 3 个 replica，**任意 1 个 ACK 就返回成功**。其他 2 个可能此刻还在网络途中或 GC 卡住。
2. 读 ONE：coordinator 随机挑 1 个 replica 读。
3. 写到 node1，读打到 node2（还没复制过去）→ 读不到

`R + W > N` 公式：1 + 1 = 2，**不大于 N=3** → 不强一致。

更糟：节点偶发 GC、网络抖动会让 hint 持续累积，恢复后才异步回放 → 几分钟内"消失"的概率会更高。

#### 修复

```python
# 推荐组合：QUORUM 写 + QUORUM 读 (R+W=4 > N=3)
session.default_consistency_level = ConsistencyLevel.LOCAL_QUORUM
```

更精细的策略：
- 不可丢的写（订单、支付）：`LOCAL_QUORUM`
- 高频日志类写（埋点、监控）：`ONE`（接受偶发不一致换性能）
- 跨 DC 强一致：`EACH_QUORUM`（贵，但每个 DC 都过半）

代码里别在每个调用都写 CL —— 在 driver 配置全局默认 + 在特殊场景显式 override。

#### 验证

```python
# 测试：并发写 + 立即读，1000 次，统计读不到的比例
import concurrent.futures
miss = 0
def write_then_read(i):
    global miss
    oid = f"order-{i}"
    session.execute("INSERT INTO orders (id, amount) VALUES (%s, 100)", [oid])
    rs = session.execute("SELECT * FROM orders WHERE id = %s", [oid])
    if not rs.one(): miss += 1

with concurrent.futures.ThreadPoolExecutor(50) as ex:
    list(ex.map(write_then_read, range(1000)))
print(f"miss rate: {miss}/1000")
```

CL=ONE/ONE 时 miss > 0；改成 LOCAL_QUORUM/LOCAL_QUORUM 后应为 0。

> **口诀**：**`R + W > N` 才强一致**。默认 LOCAL_QUORUM；非关键场景再降级。永远别用 `ANY`。

#### 延伸：SERIAL / LOCAL_SERIAL 与 LWT

需要 CAS（Compare-And-Set）语义时，Cassandra 提供 **Lightweight Transactions（LWT）**：

```sql
INSERT INTO users (id, email) VALUES ('u1', 'a@x.com') IF NOT EXISTS;
UPDATE accounts SET balance = 90 WHERE id = 'u1' IF balance = 100;
```

LWT 用 **Paxos** 协议在 partition 内做共识，CL 用 `SERIAL` / `LOCAL_SERIAL`：
- 一次 LWT = **4 轮往返**（prepare / promise / propose / accept）→ 比普通写慢 4~10 倍
- 仅在"必须保证唯一性 / 原子性"的场景用（注册唯一性、扣款）
- 永远不要在高并发热点 partition 上用 LWT —— 会退化成串行

---

### 案例 ④ Compaction 跟不上 —— "磁盘 24 小时撑爆"

#### 现象

集群跑了几个月一直稳定。某天业务流量翻倍，第二天告警"磁盘 95%"，扩容来不及。

#### 根因

`nodetool compactionstats`：

```
pending tasks: 1283
   compaction type   keyspace   table     completed   total   unit   progress
   Compaction        ks         tbl       234 GB      567 GB  bytes  41.27%
```

**1283 个 pending compaction**。正常应该是个位数。

机理：
1. 写入翻倍 → MemTable flush 速度翻倍 → SSTable 数量增长翻倍
2. 默认 `compaction_throughput_mb_per_sec = 16` —— compaction 吞吐被限速
3. 16MB/s 跟不上写入产生的速度 → SSTable 累积
4. SSTable 越多，每次 compaction 越大 → 进一步堆积
5. STCS 还需要"双倍空间"才能 compact —— 磁盘塞满后 compaction 直接停摆

观察 SSTable 数量：

```bash
nodetool cfstats ks.tbl | grep "SSTable count"
SSTable count: 14723
```

正常 < 100。1.4 万 SSTable → 一次读要查 BF + 索引数百次 → P99 飙到秒级。

#### 修复

**临时**（救火）：

```bash
# 调高 compaction 吞吐限速
nodetool setcompactionthroughput 256

# 调高并发 compactor 数（吃 CPU 但能消化积压）
nodetool setconcurrentcompactors 8
```

```yaml
# cassandra.yaml 永久配置
compaction_throughput_mb_per_sec: 64    # 从 16 提到 64
concurrent_compactors: 4                # 默认按核数自动算，可手工提
```

**临时清磁盘**：

```bash
# snapshot 占空间，先清
nodetool clearsnapshot

# 强制 major compaction（一次合并所有 SSTable，慎用：会产生一个超大 SSTable）
nodetool compact ks tbl
```

**长期**：
- 压力测试时就把"compaction 跟不上"作为 SLA 边界
- 监控 `pending compactions`，> 100 持续 10 分钟告警
- 容量规划留 50% 磁盘空闲（STCS 必需）

#### 验证

`nodetool compactionstats` 的 pending 应回落到个位数；SSTable count 回到 < 100。

> **口诀**：compaction 是 LSM 的"碎纸机"。**写入吞吐和 compaction 吞吐必须 1:1 设计**，磁盘必须留 50% 空间，监控 pending compactions。

#### 延伸：什么时候必须迁出 STCS

如果你看到下面任意一个信号，应当评估迁 LCS 或 TWCS：
- 单个 SSTable 已超过 1TB（STCS 滚雪球到极限）
- 磁盘空闲率 < 30% 且降不下来
- compaction 一跑 IO 100% 持续小时级

迁移过程：`ALTER TABLE ... WITH compaction = {...}` 后**所有 SSTable 重写一遍**，可能要 1~3 天。期间限速避免影响业务（`nodetool setcompactionthroughput 32`），分节点分批执行。

---

### 案例 ⑤ Repair 失败 —— "删除的数据复活了"

#### 现象

业务执行 `DELETE FROM users WHERE id = 'x'`，删除成功。一周后用户反馈这条数据**自己回来了**。

#### 根因

Cassandra 的删除是**写一个 tombstone**，不是物理删除。tombstone 有寿命：

```yaml
gc_grace_seconds: 864000   # 默认 10 天
```

机理：
1. DELETE 在 node1, node2 写成功（CL=QUORUM），**node3 此刻不在线**
2. 10 天后 `gc_grace_seconds` 过期，node1、node2 的 tombstone 在 compaction 中被清理
3. node3 上线 —— 它身上还有那条**未删除**的数据
4. 读到 node3 时，发现它有 row、其他节点没 row 也没 tombstone → coordinator 认为这是新写入 → 触发 read repair → 把 row 写回 node1, node2 → **数据复活**

`gc_grace_seconds` 的存在意义：**给 repair 留窗口**。10 天内必须跑一次 repair 把 tombstone 同步到所有 replica，否则就是这种现场。

#### 修复

**铁律**：**每个节点必须在 `gc_grace_seconds` 之内跑一次完整 repair**。

```bash
# 推荐用 Cassandra Reaper 自动调度
docker run -d -p 8080:8080 thelastpickle/cassandra-reaper

# Reaper Web UI 配置：
# - 每 7 天对每个 keyspace 跑一次 repair（gc_grace 是 10 天，留 3 天 buffer）
# - 用 subrange repair 拆成 256 段，避免单次扫全表
# - 错峰执行（凌晨低峰期）
```

如果不用 Reaper，手工脚本：

```bash
# 每周二凌晨 2 点
0 2 * * 2 nodetool repair -pr keyspace_name >> /var/log/cassandra-repair.log
```

`-pr` (primary range) 让每个节点只 repair 自己 primary 的 range，避免重复。**所有节点都跑 -pr** 才能覆盖全集群。

**已经发生数据复活怎么办？**
1. 找出复活数据的范围
2. 重新执行 DELETE
3. 立即跑 repair 确保所有 replica 同步删除
4. 等 `gc_grace_seconds` 后 tombstone 清理

#### 验证

监控指标：

```bash
# Reaper 暴露的 metric
curl http://reaper:8080/metrics | grep repair_completion

# 业务侧用 canary 表：每天写一条 + DELETE，30 天后检查是否复活
```

> **口诀**：**Cassandra 没跑 repair = 删除会复活**。`gc_grace_seconds = 10 天` 意味着每 10 天内必须 repair 一次。生产用 Reaper。

#### 延伸：节点长期下线超过 gc_grace 怎么办

如果节点已下线超过 `gc_grace_seconds`：**不要直接让它重新加入**。它身上的"老数据"会和其他节点的 tombstone 比对失败，触发数据复活。

正确做法：
1. `nodetool removenode <host_id>` 把节点从集群剔除
2. 清掉该节点本地数据目录
3. 作为新节点重新 bootstrap 加入 —— 数据会从其他 replica 完整流式同步过来

类似的反模式：把硬盘备份（一周前的）直接还原到节点 —— 同样会复活已删数据。

---

### 案例 ⑥ JVM Heap 配置错误 —— "CMS 升 G1 后 P99 翻倍"

#### 现象

把 JVM 从 CMS 升到 G1（追上社区推荐），希望降低 GC 暂停。结果 P99 写延迟从 8ms 涨到 18ms。

#### 根因

老配置（CMS）：

```bash
-Xms16G -Xmx16G
-XX:+UseConcMarkSweepGC
-XX:+CMSParallelRemarkEnabled
-XX:NewRatio=1
-XX:SurvivorRatio=8
-XX:MaxTenuringThreshold=1
```

新配置（G1，照搬默认推荐）：

```bash
-Xms16G -Xmx16G
-XX:+UseG1GC
-XX:G1RSetUpdatingPauseTimePercent=5
-XX:MaxGCPauseMillis=200
```

问题出在两点：
1. **G1 在 16GB 堆下表现一般**：G1 设计是为 32GB+ 大堆。16GB 时 region 数量少（默认 region 大小 8MB → 2048 个 region），调度收益不明显。
2. **Cassandra 的内存特点**：MemTable + Bloom Filter + Key Cache + offheap memory 占大头，**heap 里短命对象多**。CMS 的 ParNew + Tenured 模型对这种 workload 反而高效。

观察 G1 GC 日志：

```
[Eden: 7168M(7168M)->0B(7168M) Survivors: 1024M->1024M
 Heap: 14.2G(16G)->8.1G(16G)] 850ms
```

850ms 的 mixed GC —— 直接打爆 P99。

#### 修复

**方案 A**（推荐 Cassandra 4.x+）：用 G1 但堆调到 31GB（避免压缩指针失效）。

```bash
-Xms31G -Xmx31G
-XX:+UseG1GC
-XX:MaxGCPauseMillis=300
-XX:G1HeapRegionSize=16M
-XX:InitiatingHeapOccupancyPercent=70
-XX:G1ReservePercent=20
-XX:ParallelGCThreads=16
-XX:ConcGCThreads=4
```

为什么 31GB？JVM 在 32GB 以下用"压缩指针"（compressed oops），对象引用 4 字节而非 8 字节，节省 1/3 内存。一旦超 32GB 退化为非压缩指针，反而吃更多内存。

**方案 B**（Cassandra 4.0+ 特有）：试 ZGC（低延迟 GC）。

```bash
-XX:+UseZGC
-XX:+ZGenerational      # JDK 21+
-Xmx31G
```

ZGC 暂停 < 10ms，几乎无 STW。但 CPU 占用更高（约 +15%），且需要 JDK 17+ 配套。

**方案 C**：堆别开太大（< 16GB），多余内存让 OS page cache 用。Cassandra 大量数据在 page cache 里，比堆缓存效率高。

#### 验证

```bash
# 观察 GC 暂停时间分布
java -Xlog:gc*:file=gc.log:time,uptime,level,tags ...
# 用 GCViewer / GCEasy 分析

# 业务指标
P99 写延迟稳定 < 10 ms
Full GC 频率 < 1 次/天
```

> **口诀**：**Cassandra 堆 < 32GB（压缩指针）；G1 适合大堆，CMS 仍是中堆首选**。剩余物理内存让 OS 做 page cache。

#### 延伸：offheap 的诱惑与陷阱

`memtable_allocation_type` 可以选 `heap_buffers` / `offheap_buffers` / `offheap_objects`。Offheap 把 MemTable 部分挪到堆外，**减轻 GC 压力**。但：
- `offheap_objects` 在 Cassandra 4.x 仍标记为实验，4.1 偶发内存泄漏的 bug
- offheap 的内存分配/释放不走 GC，**RSS 内存可能远超 `-Xmx`**，OS 看到的内存占用让人困惑
- 容器化（K8s）部署一定要算上 offheap 的内存，否则会被 cgroup OOMKilled

通用建议：先用默认 `heap_buffers`，G1 调好；只有当 GC 暂停明显成为瓶颈才尝试 offheap。

---

### 案例 ⑦ 跨数据中心复制延迟 —— "EACH_QUORUM 写超时"

#### 现象

跨北京 / 上海双 DC 部署，应用用 `EACH_QUORUM` 写，希望两 DC 都强一致。某天写延迟 P99 从 30ms 飙到 5s，大量超时。

#### 根因

`nodetool status` 看上海 DC 节点状态：

```
Datacenter: shanghai
====================
Status=Up/Down  |/ State=Normal/Leaving/Joining/Moving
--  Address          Load     Tokens  Owns  Host ID
DN  10.20.1.5        145GB    256     ?     ...     ★ Down
DN  10.20.1.6        148GB    256     ?     ...     ★ Down
UN  10.20.1.7        146GB    256     ?     ...
```

上海 DC 3 节点 RF=3，挂了 2 个 → 上海 DC 没法满足 LOCAL_QUORUM（需要 2 个节点 ACK）→ EACH_QUORUM 永远等不到上海 DC 的 quorum → 5s 超时。

更深层原因：
1. 北京 DC 写入正常
2. 上海 DC 节点过载（compaction 跟不上）→ 内存爆 → OOM 重启 → 进入"半死状态"
3. Phi failure detector 把这 2 个节点判定为 DOWN
4. EACH_QUORUM 写失败

#### 修复

**临时**（恢复业务）：

```python
# 降级为 LOCAL_QUORUM（只要本 DC 强一致）
session.execute(stmt, consistency_level=ConsistencyLevel.LOCAL_QUORUM)
```

注意：这意味着上海 DC 此刻可能读到旧数据，**业务侧要接受**这种降级。

**长期**：
- 上海 DC 节点扩容 + 修 compaction 配置
- 关键路径**默认 LOCAL_QUORUM**，跨 DC 强一致只在极少数场景用 EACH_QUORUM
- 监控跨 DC 延迟（`StreamingMetrics`）和单 DC 节点存活率
- 跨 DC 用 `NetworkTopologyStrategy`：

```sql
CREATE KEYSPACE app WITH replication = {
    'class': 'NetworkTopologyStrategy',
    'beijing': 3,
    'shanghai': 3
};
```

不要用 SimpleStrategy（不感知拓扑，跨 DC 表现异常）。

**Driver 配置 DC 亲和性**：

```python
from cassandra.policies import DCAwareRoundRobinPolicy
cluster = Cluster(
    contact_points=['10.10.1.1'],
    load_balancing_policy=DCAwareRoundRobinPolicy(local_dc='beijing'),
    # 默认只发给 local DC，节点全挂时才 fallback 到 remote
)
```

#### 验证

监控告警：
- `org.apache.cassandra.metrics:type=ClientRequest,scope=Write,name=Timeouts` 持续 > 0
- 跨 DC 节点存活率 < 100%
- `nodetool tpstats` 看 `MutationStage` pending > 1000

> **口诀**：**默认 LOCAL_QUORUM，EACH_QUORUM 谨慎用**。跨 DC 部署必须用 NetworkTopologyStrategy + DCAwareRoundRobinPolicy。

#### 延伸：snitch 的选择

Cassandra 用 **snitch** 告诉自己"哪些节点在哪个 DC / rack"。常见选项：
- `SimpleSnitch`：所有节点同 DC 同 rack（单 DC 测试）
- `GossipingPropertyFileSnitch`（推荐）：每个节点本地配置 `cassandra-rackdc.properties`，gossip 互相同步
- `Ec2Snitch` / `Ec2MultiRegionSnitch`：AWS 部署的便利选项，自动识别 region / AZ
- `PropertyFileSnitch`：全集群共用一份 `cassandra-topology.properties`，加节点要改文件**慎用**

切换 snitch 是高风险操作 —— 节点对 DC / rack 的认知突变会触发数据全量重平衡。一定提前演练。

---

### 案例 ⑧ Batch 写入误用 —— "用 BATCH 反而更慢"

#### 现象

为了"批量写入提速"，业务把 100 条 INSERT 改用 BATCH 包起来：

```sql
BEGIN BATCH
INSERT INTO orders (id, ...) VALUES (1, ...);
INSERT INTO orders (id, ...) VALUES (2, ...);
... (100 条)
APPLY BATCH;
```

结果发现：写入吞吐**降到原来的 1/3**，coordinator 节点 CPU 飙升。

#### 根因

**Cassandra 的 BATCH 不是为高性能设计的**。BATCH 主要用途有两个：
1. **同 partition 多 row 的原子写**（轻量，性能好）
2. **跨 partition 的逻辑原子性保证**（"Logged Batch"，借助 batchlog 实现，**很重**）

跨 partition BATCH 流程：
1. coordinator 把整个 BATCH 写入 `system.batches` 表（batchlog） → **多一次磁盘 IO**
2. coordinator 同时复制 batchlog 到另外 2 个节点（防止 coordinator 挂掉后 batch 丢） → **多 2 倍网络流量**
3. coordinator 解析 BATCH，把每条 INSERT 路由到对应的 partition 所在节点
4. 等所有 INSERT ACK 后，删除 batchlog
5. 任何一条失败 → 后台异步重试整个 batch

100 条跨 partition INSERT 用 BATCH = **300 次额外的网络/磁盘 IO + 一个 coordinator 单点压力**。

正确用法：
- **同 partition 的多 row** → BATCH 高效（合并成一次 mutation）
- **跨 partition 的多条独立写入** → 用 `concurrent_async_queries`（异步并发提交）

#### 修复

```python
# ❌ 错误：跨 partition 用 BATCH
batch = BatchStatement()
for order in orders:  # orders 跨多个 partition
    batch.add(insert_stmt, [order.id, ...])
session.execute(batch)

# ✅ 正确：异步并发
from cassandra.concurrent import execute_concurrent_with_args
execute_concurrent_with_args(
    session,
    insert_stmt,
    [(order.id, ...) for order in orders],
    concurrency=50
)
```

如果一定要原子性（金融场景），用 `LOGGED BATCH` 但接受性能代价；不需要原子性用 `UNLOGGED BATCH` 跳过 batchlog（但仍是 coordinator 串行）—— 几乎没用，不如异步并发。

```sql
-- UNLOGGED 跳过 batchlog，性能稍好，但仍单 coordinator 串行
BEGIN UNLOGGED BATCH
... APPLY BATCH;
```

#### 验证

```python
import time
t0 = time.time()
# 跑 1000 次：BATCH vs concurrent
print(f"elapsed: {time.time() - t0:.2f}s")
```

concurrent 应比 BATCH 快 3~10 倍。

> **口诀**：**Cassandra 的 BATCH 是"原子性工具"不是"性能工具"**。批量提速用 driver 的 concurrent API。

#### 延伸：BATCH 的 size 阈值告警

Cassandra 在 BATCH 太大时会主动报警，阈值在 `cassandra.yaml`：

```yaml
batch_size_warn_threshold_in_kb: 5      # 超过 5KB 写 warning 日志
batch_size_fail_threshold_in_kb: 50     # 超过 50KB 直接拒绝
```

如果你的业务日志里频繁看到 `Batch of size XXX exceeds the configured warning threshold` —— 不是去调高阈值，而是**评估为什么有这么大的 BATCH**。99% 的情况是误用。

---

### 案例 ⑨ 数据建模错误 —— "用 SQL 思维设计 Cassandra 表"

#### 现象

DBA 出身的同学按 MySQL 思路建表：标准化、外键、二级索引到处加。结果跑起来到处是性能坑：JOIN 不支持、IN 查询慢、二级索引 scatter-gather 拖死集群。

#### 根因

Cassandra 的数据建模哲学：**Query-First Design**（查询驱动建模）。和关系数据库**完全相反**。

| 维度 | MySQL（SQL 思维） | Cassandra（NoSQL 思维） |
| --- | --- | --- |
| 设计起点 | 实体关系 ER 图 | **业务查询模式** |
| 范式 | 高范式（避免冗余） | **反范式（鼓励冗余）** |
| JOIN | 普遍 | **不支持，必须冗余** |
| 二级索引 | 普遍 | **谨慎用**（仅高基数 + 低频查询） |
| 一张表服务多查询 | 常见 | **一张表一种查询**（Query Per Table） |

错误示例：

```sql
-- ❌ SQL 思维
CREATE TABLE users (id uuid PRIMARY KEY, name text, email text);
CREATE TABLE orders (id uuid PRIMARY KEY, user_id uuid, amount decimal);
CREATE INDEX ON orders (user_id);    -- 二级索引

-- 查询：某用户的所有订单
SELECT * FROM orders WHERE user_id = ?;
```

这个查询会走二级索引 → 二级索引在 Cassandra 里是**每个节点本地的索引** → 查询时 coordinator 必须问**所有节点**有没有这个 user_id 的数据 → scatter-gather → 节点数越多越慢。

正确做法：

```sql
-- ✅ Cassandra 思维：按查询建表
CREATE TABLE orders_by_user (
    user_id uuid,
    order_id timeuuid,
    amount decimal,
    PRIMARY KEY (user_id, order_id)
) WITH CLUSTERING ORDER BY (order_id DESC);

-- 查询：直接命中 partition
SELECT * FROM orders_by_user WHERE user_id = ?;
```

如果还要按 order_id 查（订单详情）：

```sql
-- 再建一张表（数据冗余！）
CREATE TABLE orders_by_id (
    order_id timeuuid PRIMARY KEY,
    user_id uuid,
    amount decimal
);
```

写入时**同时写两张表**（应用层保证一致性，或用 Materialized View）。**磁盘空间换查询性能** —— 这是 NoSQL 的核心权衡。

#### 修复

建模流程：
1. 先列**所有业务查询**（"按用户查订单"、"按订单 ID 查详情"、"按时间范围查"...）
2. **每个查询设计一张表**：partition key = 查询的等值条件，clustering key = 范围/排序条件
3. 检查每个 partition 大小 < 100MB / 10 万 row
4. 检查写入是否均衡（无 hot partition）
5. 数据冗余 OK，**写多份是常态**

二级索引**唯一**适合的场景：
- 查询频率低（运维查询，不是业务高频）
- 索引列高基数（每个值对应少量 row）
- **绝对不要**用二级索引做主要查询

#### 验证

新设计上线前，每个业务查询走 `EXPLAIN`（CQL 5.0+ 支持）：

```sql
EXPLAIN SELECT * FROM orders_by_user WHERE user_id = '...';
```

确认每个查询都是 partition key 等值或 clustering key 范围，没有 `ALLOW FILTERING`（filtering 几乎等于全表扫描，禁用）。

> **口诀**：Cassandra 建模 = **Query-First + 反范式 + 一查询一张表**。SQL 思维做 Cassandra = 性能灾难。

#### 延伸：Materialized View 的坑

Cassandra 提供 `CREATE MATERIALIZED VIEW`（物化视图），看起来能自动维护"按另一个 key 查的表"：

```sql
CREATE MATERIALIZED VIEW orders_by_id AS
    SELECT * FROM orders_by_user
    WHERE order_id IS NOT NULL AND user_id IS NOT NULL
    PRIMARY KEY (order_id, user_id);
```

但 MV 在 3.x 和 4.x 都有已知 bug（一致性问题、数据丢失风险），社区**官方建议不要在生产用**（CASSANDRA-13826）。生产里**老老实实在应用层写多张表**更安全。

5.0 的 SAI（Storage-Attached Indexing）部分缓解了这个问题，但 MV 本身仍标记 experimental。

---

### 案例 ⑩ 节点扩容时数据流式失败 —— "bootstrap 跑了 12 小时还没完"

#### 现象

集群从 10 节点扩到 12 节点，加第一个新节点时 bootstrap 跑了 12 小时进度还在 60%。期间业务读偶发 `UnavailableException`。

#### 根因

`nodetool netstats` 看 streaming 状态：

```
Mode: NORMAL
Bootstrap c5dac460-...
    /10.0.0.5
        Receiving 142 files, 67.3 GB total
            ks/tbl-Data.db 1234567890/2345678901 bytes(52%) ★慢
```

机理：
1. bootstrap 期间新节点要从所有相关 replica **流式接收**该节点应该负责的 token 范围数据
2. 默认 `stream_throughput_outbound_megabits_per_sec = 200`（25 MB/s）→ 100GB 数据 ≥ 1 小时
3. 但 source 节点同时还要处理业务流量 → 实际带宽更低
4. 12 小时仍未完成的常见原因：磁盘 IO 饱和、网络抖动、source 节点 GC 暂停

更糟的是 `streaming_socket_timeout_in_ms`（默认 24 小时）一旦超时，bootstrap 失败必须从头来。

#### 修复

**临时**（救火）：

```bash
# 调高 streaming 限速（GB 网络可上 800Mbps）
nodetool setstreamthroughput 800

# 调高 inter-DC streaming（如果跨 DC）
nodetool setinterdcstreamthroughput 400
```

**预防**：
- 大集群扩容用 **Zero-Copy Streaming**（4.0+ 默认开启），SSTable 直接整文件传输不解析重写，10x+ 加速
- 一次只加一个节点（社区铁律），同时 bootstrap 多个节点会数据漂移
- 业务低峰期扩容
- 监控 `pending streams` + `failed streams`

```yaml
# cassandra.yaml
stream_entire_sstables: true                 # zero-copy streaming，4.0 默认 true
stream_throughput_outbound_megabits_per_sec: 800
inter_dc_stream_throughput_outbound_megabits_per_sec: 400
```

#### 验证

`nodetool netstats` 看 streaming 进度，bootstrap 完成后 `nodetool status` 显示新节点 `UN`（Up Normal），`Owns` 列显示其负责的 token 比例。

> **口诀**：扩容用 zero-copy streaming + 调高限速 + 一次一个节点。监控 streaming 进度，超时前介入而不是等失败重来。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 数据建模层
- [ ] 每张表对应明确的查询模式（Query Per Table）
- [ ] partition key 高基数 + 访问均衡（无热点）
- [ ] 单 partition 预估大小 < 100 MB / 10 万 row
- [ ] 数据冗余写多张表（不依赖 JOIN）
- [ ] 二级索引仅在低频运维场景使用，禁止主链路依赖
- [ ] CQL 中无 `ALLOW FILTERING`（除非已知数据量极小）

### 一致性层
- [ ] 默认 `LOCAL_QUORUM`（写 + 读）
- [ ] 跨 DC 用 `NetworkTopologyStrategy`，不用 `SimpleStrategy`
- [ ] Driver 配置 `DCAwareRoundRobinPolicy(local_dc='xxx')`
- [ ] 关键写入显式标 CL，避免依赖框架默认
- [ ] 永远不用 `CL=ANY`

### Compaction 层
- [ ] 写多读少 → STCS；读多写少 → LCS；时序 → TWCS
- [ ] `compaction_throughput_mb_per_sec >= 64`（NVMe 可上 256）
- [ ] `concurrent_compactors` 至少 4
- [ ] 监控 `pending compactions`，> 100 持续 10 分钟告警
- [ ] 磁盘留 50% 空闲（STCS 必需）

### Repair 层
- [ ] 部署 Cassandra Reaper 自动调度 repair
- [ ] 每个 keyspace 在 `gc_grace_seconds`（默认 10 天）内必须 repair 一次
- [ ] subrange repair 拆段执行，单次 < 1 小时
- [ ] 监控 repair 完成率，失败立即告警

### JVM 层
- [ ] Heap < 32 GB（保留压缩指针）
- [ ] G1GC + `MaxGCPauseMillis=300`（堆 > 16GB 时）
- [ ] CMS 仍可用于堆 ≤ 16GB
- [ ] GC 日志开启，集中收集分析
- [ ] Full GC 频率 < 1 次/天

### 写入层
- [ ] 跨 partition 批量写用 driver concurrent API，不用 BATCH
- [ ] BATCH 仅用于"同 partition 多 row"或"跨 partition 必须原子"
- [ ] CommitLog 在独立磁盘（IO 隔离）
- [ ] `commitlog_sync = periodic`（默认）—— 接受 10s 数据丢失换吞吐

### 监控层
- [ ] `nodetool tpstats` 各 stage pending 监控
- [ ] tombstone：`tombstone_warn_threshold = 1000`，超过查日志
- [ ] hot partition：`nodetool toppartitions` 定期跑
- [ ] 节点存活率（每 DC 都要 100%）
- [ ] 跨 DC 复制延迟
- [ ] history hint 数（`HintsService.totalHintsInProgress`）

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### ScyllaDB 的 C++ 重写
- **Seastar 框架**：每核一个 reactor，shared-nothing 架构，无锁
- 用户态 TCP / DPDK 零拷贝 IO
- 与 Cassandra wire 兼容 → driver / 工具链可直接复用
- 何时该迁：新项目、性能敏感、能接受相对小的社区

### Cassandra 5.0 的新特性
- **Storage-Attached Indexing (SAI)**：替代糟糕的旧二级索引，性能可用
- **Vector Search**：原生向量索引（AI 时代必备）
- **Trie-based memtables / SSTable**：压缩率提升、读性能提升
- 升级前必须看 release note 的 breaking changes

### Cassandra on K8s（K8ssandra）
- **K8ssandra Operator**：把 Cassandra 集群当成 CRD 管理
- StatefulSet + 持久卷（PVC）
- 滚动升级 / 自动 repair / 自动备份
- 注意：本地盘 vs 网络盘的延迟差异 10x+

### DataStax Astra（云服务）
- 全托管 Cassandra（serverless 选项）
- 按 token 计费，省运维但价格不便宜
- 适合中小团队，没专职 DBA

### 与 DynamoDB 的对比
- **DynamoDB**：AWS 全托管，partition 数自动管理；按吞吐量计费
- **Cassandra**：自托管，可控但要运维；硬件成本可控
- 选择维度：是否上 AWS、对一致性级别的灵活度需求、运维团队规模

### Time-Series 场景的更优选
- **TimescaleDB**：PostgreSQL 扩展，SQL + 时序优化
- **InfluxDB**：原生时序，但分布式版本商业化
- **Cassandra + TWCS**：海量低成本，但 SQL 能力差

### Cassandra Reaper / ScyllaDB Manager 的运维对比
- Reaper：开源，UI 友好，社区维护
- ScyllaDB Manager：官方，集成度高，更稳定
- 运维 100+ 节点集群必须有自动化工具

---

## 7. 引用

- Apache Cassandra 4.1 / 5.0 源码：[apache/cassandra](https://github.com/apache/cassandra)
- *Cassandra: The Definitive Guide* (3rd ed.) —— Jeff Carpenter / Eben Hewitt
- O'Neil et al., *The Log-Structured Merge-Tree (LSM-Tree)*（1996）—— LSM 原始论文
- DeCandia et al., *Dynamo: Amazon's Highly Available Key-value Store*（SOSP 2007）—— Cassandra 设计原型
- Lakshman / Malik, *Cassandra - A Decentralized Structured Storage System*（2010）
- ScyllaDB 架构白皮书：[scylladb.com/architecture](https://www.scylladb.com/product/technology/)
- DataStax 官方文档：[Tunable Consistency](https://docs.datastax.com/en/cassandra-oss/3.0/cassandra/dml/dmlConfigConsistency.html)
- The Last Pickle 博客（已被 DataStax 收购）：Cassandra 运维最佳实践集合

---

> 配套：[Redis 内核与一致性](/article/db/redis-internals) | [Elasticsearch 深入](/article/db/elasticsearch) | [共识算法](/article/distributed/consensus)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
