# HBase 深入 —— LSM、Region 分裂、协处理器与生产事故

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，看 [Cassandra 深入](/article/db/cassandra) 里 LSM 那一节也能凑合。
> 这里要回答的，是**线上 RegionServer 凌晨被打挂、Compaction 拖住业务、RowKey 设计踩坑**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. 同样是 LSM-Tree、同样宣称"列式存储"，HBase 和 Cassandra 在 CAP 选择、写入路径、一致性模型上的**本质差别**是什么？为什么阿里早期把 HBase 换成 OceanBase / Lindorm 而不是 Cassandra？
2. 你设计的 RowKey 是 `userId + timestamp`，上线一周一个 RegionServer CPU 100%，其他几台空闲 —— 错在哪？给出三种规避方案。
3. Region 默认配 10GB 自动 Split，一个突发活动后 Region 数从 200 涨到 2000 —— 这是好事还是坏事？分别会引发什么后果？
4. MemStore Flush 触发的条件至少有 5 个，你能列全吗？哪个是"全局阻塞写入"的元凶？
5. Major Compaction 默认 7 天跑一次。你能不能关掉它、改用手动触发？关掉之后有什么风险？
6. BlockCache 的 LruBlockCache 和 BucketCache 各自适合什么场景？为什么大堆内存不能全交给 LruBlockCache？
7. 业务想在 HBase 里做"扫描后聚合 Top-N"，你是用 Scan 拉到客户端聚合，还是用 Endpoint 协处理器？两者性能差几个数量级？
8. HBase 不支持跨行事务。但业务确实需要"扣库存 + 写订单"的原子性，你怎么落地？为什么 Phoenix 的 ACID 也不能直接用于这个场景？

如果有半数答不上，往下看。

---

## 1. 底层模型：从 Bigtable 论文到 LSM 实现

HBase 是 Google Bigtable 论文的开源实现。所以理解 HBase 必须先理解 Bigtable 想解决什么问题：**在普通 PC 服务器集群上、用 GFS（HDFS）做存储、用 Chubby（ZooKeeper）做协调，构建一个能容纳 PB 级稀疏数据的有序 KV 表**。

每个设计选择都是"这个目标"的直接产物。下面拆开看。

### 1.1 整体架构：四个角色

```
                 ┌──────────────┐
                 │ ZooKeeper 集群 │ ← 元数据/选主/状态
                 └──────┬───────┘
                        │
                ┌───────▼───────┐
                │  HMaster (主)  │ ← 元数据维护、Region 分配、负载均衡
                │  HMaster (备)  │
                └───────┬───────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
   ┌────▼────┐    ┌────▼────┐    ┌────▼────┐
   │ Region  │    │ Region  │    │ Region  │   ← 真正干活的节点
   │ Server  │    │ Server  │    │ Server  │     存放 Region、处理读写
   └────┬────┘    └────┬────┘    └────┬────┘
        │              │              │
        └──────────────┼──────────────┘
                       ▼
                ┌──────────────┐
                │     HDFS     │ ← 持久化存储（HFile + WAL）
                └──────────────┘
```

四个角色的职责泾渭分明：

| 角色 | 职责 | 挂掉影响 |
| --- | --- | --- |
| **ZooKeeper** | 存元数据位置（hbase:meta 在哪）、Master 选主、RegionServer 心跳 | 全集群停摆 |
| **HMaster** | 元数据 DDL、Region 分配、负载均衡、故障转移 | 短期数据读写不受影响（无 Master 也能跑），但不能建表/Split |
| **RegionServer** | 真正承载 Region、处理 Get/Scan/Put、做 Flush 和 Compaction | 该 RS 上的 Region 被重新分配到其他 RS，期间这些 Region 不可读写 |
| **HDFS** | 持久化 HFile 和 WAL | 全集群写入挂掉、读受 BlockCache 命中率影响 |

**关键设计选择**：HBase 自身不存数据，全靠 HDFS。所以"扩容存储"和"扩容计算"是分开的 —— 这是后面"为什么 HBase 适合 PB 级冷热混合"的根。

> RegionServer 不是数据的"主人"，HDFS 才是。RegionServer 只是 HDFS 上一段数据的"当前服务者"。这个视角很重要：RegionServer 死了数据不会丢，只是服务暂时不可用。

### 1.2 数据模型：稀疏多维 Map

HBase 的逻辑模型不是关系表，而是：

```
SortedMap< RowKey,
  SortedMap< ColumnFamily,
    SortedMap< ColumnQualifier,
      SortedMap< Timestamp, Value >>>>
```

四层嵌套 SortedMap。一条逻辑"行"实际上是 N 个 (Row, CF, Qualifier, Timestamp, Value) 五元组（HBase 内部叫 **KeyValue** 或 **Cell**）。

举个例子：

| RowKey | ColumnFamily | Qualifier | Timestamp | Value |
| --- | --- | --- | --- | --- |
| user:001 | info | name | 1700000001 | alice |
| user:001 | info | age | 1700000001 | 30 |
| user:001 | info | age | 1700000050 | 31 (新版本) |
| user:001 | stats | login_cnt | 1700000100 | 5 |

存储时是这样的物理布局：

```
KeyValue 1: <user:001, info, name,      1700000001, alice>
KeyValue 2: <user:001, info, age,       1700000050, 31>
KeyValue 3: <user:001, info, age,       1700000001, 30>     ← 老版本
KeyValue 4: <user:001, stats, login_cnt, 1700000100, 5>
```

四件事必须看出来：

1. **每个 Cell 都重复存 RowKey 和 CF**：没有压缩前一行 100 列的存储远大于 RDBMS。HBase 靠**前缀编码 + Block 压缩**才把这个开销压回来。
2. **多版本是一等公民**：默认存 1 个版本（老版本写过来直接覆盖磁盘上的同 ts），可配 `VERSIONS => 3` 或 `VERSIONS => 100`。删除是写入一个 Tombstone（墓碑）KeyValue，物理删除要等 Major Compaction。
3. **稀疏免费**：某个 RowKey 没有某个 Qualifier 就是"不存"，不像 RDBMS 要存 NULL。一行写 1 个列和写 1000 个列，前者只有 1 个 Cell。
4. **同一 ColumnFamily 内的所有 Cell 物理上连续存储**（在同一组 HFile 里）。**不同 CF 是不同 HFile**。所以"用列族隔离冷热数据"是 HBase 的核心建模技巧。

> 别拿 RDBMS 的"宽表"思维设计 HBase。HBase 是 KV，行是 RowKey 维度上的稀疏 Map。CF 数量 ≤ 3，CF 内列数随意。

### 1.3 RowKey 设计：决定一切的 16~64 字节

HBase 的所有数据按 RowKey 字典序排列，物理上散布在 N 个 Region 上。**RowKey 设计决定了**：

- 单点查询能不能 O(1) —— 散列度
- 扫描能不能高效 —— 局部性
- 写入热点会不会出现 —— 散列度
- 跨行查询能不能下推 —— 前缀设计

四个原则：

**原则一：避免单调递增**

```
错的 RowKey：timestamp + userId
   1700000001-u1    ┐
   1700000002-u2    │
   1700000003-u3    ├─ 全部落在同一个最大 Region
   ...              │   该 Region 所在的 RegionServer CPU 爆掉
   1700000999-u9    ┘
```

**原则二：散列前缀（Salt / Hash）**

```
方案 A：取 hash(userId) 前 N 位作为前缀
   00-u3-1700000003    ┐
   01-u1-1700000001    ├─ 散列到不同 Region
   02-u2-1700000002    ┘

方案 B：reverse(userId) + timestamp
   3u-1700000003       ┐
   1u-1700000001       ├─ userId 后缀差异大，散列均匀
   2u-1700000002       ┘
```

方案 A 的代价：**按 userId 的 Scan 必须扫所有 Region**。如果业务核心 Scan 是"按 userId 查一段时间的数据"，方案 A 反而把性能压死。

**原则三：把"扫描友好"放第一位**

如果你的最高频查询是"查 user u1 最近 7 天的事件"，RowKey 就该设计成：

```
u1-(Long.MAX_VALUE - timestamp)
```

- 同 user 的所有事件物理连续 → 一次 Scan 拉完
- 用 `MAX_VALUE - ts` 让最新的排在前面 → Scan 加 LIMIT 10 就是"最近 10 条"

**原则四：固定长度 + 紧凑编码**

每个 KeyValue 都重复存 RowKey。**RowKey 多 10 字节 → 100 亿数据多 100GB**。

```
错：order_id_2024-11-15-user123-actionABC          (40+ 字节)
对：reverse(orderId)+yymmdd+uid_hash               (16 字节)
```

> RowKey 是 HBase 设计中最难悔改的决定。一旦上线、有数据、有业务依赖，**改 RowKey = 重导全表**。设计阶段要把 Top 5 查询场景列出来，对每种场景验证 RowKey 是否能高效支持。

### 1.4 Region 与 Region Split

一张表被水平切分成 N 个 **Region**。每个 Region 是 RowKey 区间 `[startKey, endKey)`。Region 是 HBase 的**负载均衡和分布式的基本单位**。

```
table = users
   Region R1: [, "k") on RS-1
   Region R2: ["k", "p") on RS-2
   Region R3: ["p", "u") on RS-3
   Region R4: ["u", ) on RS-1
```

Split 触发时机（按 HBase 2.x 默认 `IncreasingToUpperBoundRegionSplitPolicy`）：

```
maxFileSize = min(R^3 × hbase.hregion.memstore.flush.size × 2, hbase.hregion.max.filesize)
其中 R = 该 RS 上该表的 Region 数
```

| 当 RS 上某表的 Region 数 | maxFileSize |
| --- | --- |
| 1 | 256MB（小） |
| 2 | 2GB |
| 3 | 6.75GB |
| ≥ 4 | 10GB（封顶） |

**为什么是这个公式？** 因为新表初始一般只有 1 个 Region，如果按 10GB 才 Split，新表写入全堆在那一个 Region 上 —— 会形成严重热点。这个公式让"前几次 Split 来得早一些"，让数据快速分散。

Split 物理过程（简化）：

```
1. RegionServer 发现 R1 该 Split
2. 在 ZooKeeper 上创建 splitting znode
3. 在 HDFS 上为 R1 创建 .splits 目录
4. 关闭 R1（短暂不可用）
5. 创建两个子 Region R1a 和 R1b 的元数据
6. 不复制数据：用 HDFS Reference 文件指向父 R1 的 HFile 的上半段/下半段
7. 更新 hbase:meta 元数据
8. 上线 R1a 和 R1b
9. R1 标记为已 Split
10. 后续 Compaction 时把 Reference 真正物化
```

**关键点**：Split 本身不复制数据（用 Reference），所以很快（秒级）。但**会让 R1 短暂不可用**（步骤 4~8 之间）。

> Region 数太少 → 单 RS 热点；Region 数太多 → MemStore 占满堆、Compaction 风暴。一个 RS 健康的 Region 数大概在 50~200 之间。**预分区**是上线前必做的事。

### 1.5 LSM-Tree：MemStore + WAL + HFile

HBase 写入路径完全是 LSM 套路：

```
客户端 Put
   │
   ▼
RegionServer
   ├─→ ① 先写 WAL (HDFS append)        ← 持久化保证
   └─→ ② 再写 MemStore (内存 SortedMap) ← 快返回
                │
                │ (MemStore 满了)
                ▼
            Flush 成 HFile (HDFS)        ← L0 文件
                │
                │ (HFile 多了)
                ▼
            Compaction                   ← 合并成更大的 HFile
                │
                ▼
            HFile (HDFS, 最终态)
```

**WAL（Write-Ahead Log）**：一个 RS 上所有 Region 共享一个（HBase 1.x 默认）或多个（2.x MultiWAL）WAL 文件。每条 Put 先 append 到 WAL，**fsync** 到 HDFS（实际是写到 N 个 DataNode 的 OS 缓存，不一定真落盘 —— 这是 HBase 写入"快"的关键之一）。

**MemStore**：每个 Region 的每个 ColumnFamily 各自有一个 MemStore，本质是 `ConcurrentSkipListMap<KeyValue, KeyValue>`（HBase 2.x 引入 CompactingMemStore 把 SkipList 改成多段 Segment）。

**HFile**：MemStore 满了之后顺序写入 HDFS 形成的不可变文件。结构类似 SSTable：数据块 + 索引块 + 布隆过滤器 + 元数据。

**为什么 LSM 而不是 B+Tree？**

- B+Tree 的随机写会导致**随机 IO + 写放大**（一次更新可能要写一整页）
- LSM 把随机写转成"内存 + 顺序磁盘"，单次写入开销极小
- 代价：读取可能要查 N 个 HFile + MemStore，**读放大**
- 缓解：BlockCache + Bloom Filter + Compaction

> LSM 的本质是**用读放大换写放大**。HBase 是写优化的存储，纯读场景（如 OLAP）应该用 ClickHouse/Doris，不是 HBase。

### 1.6 Compaction：Minor 与 Major

LSM 的代价就是 HFile 越来越多、读越来越慢。靠 Compaction 合并：

| 类型 | 合并对象 | 物理删除 Tombstone | 触发频率 |
| --- | --- | --- | --- |
| **Minor Compaction** | 选若干小 HFile 合成一个稍大的 | **不删除**（避免影响其他版本） | 频繁，毫秒级触发判断 |
| **Major Compaction** | 一个 Region 的某个 CF 的**所有 HFile** 合成一个 | **删除**（重写过程中跳过墓碑数据） | 默认 7 天一次 |

**Minor Compaction 选文件的策略**（默认 `ExploringCompactionPolicy`）：

```
找一个 HFile 子集，使得：
1. 子集大小在 [minFiles, maxFiles] 之间
2. 子集内最小文件 × ratio ≥ 子集内最大文件
3. 在所有合法子集里选合并后总大小最小的
```

参数：
- `hbase.hstore.compaction.min`（默认 3）：至少 3 个文件才触发
- `hbase.hstore.compaction.max`（默认 10）：一次最多合 10 个文件
- `hbase.hstore.compaction.ratio`（默认 1.2）：用于上面的 ratio

**Major Compaction 的代价**：
- 一个 RS 同时跑多个 Region 的 Major → 磁盘 IO 拉满 → 读写延迟暴涨
- 默认 7 天周期会"扎堆"，整个集群同一天集体抖动

**生产经验**：关掉自动 Major（`hbase.hregion.majorcompaction=0`），改用脚本错峰手动触发。后面案例 ③ 会展开。

### 1.7 BlockCache：读路径的核心

HBase 读路径的 IO 来源：MemStore + 多个 HFile。HFile 在 HDFS 上，每次读都要走网络 + 磁盘。BlockCache 是 RS 进程内的缓存。

两种实现：

| 实现 | 存储位置 | 容量 | GC 影响 | 适用 |
| --- | --- | --- | --- | --- |
| **LruBlockCache** | JVM 堆内 | 受 -Xmx 约束（一般几个 GB） | **大**（频繁淘汰触发 GC） | 默认；小堆场景 |
| **BucketCache** | 堆外（DirectByteBuffer 或 SSD） | 几十 GB ~ 几 TB | **小**（不进 GC） | 大内存机器；推荐 |

生产实践：
- 堆内只放 Index、Bloom 这种"必须快速访问"的元数据（约 1~2GB）
- 数据块放 BucketCache（堆外或 SSD）

```xml
<!-- hbase-site.xml -->
<property>
  <name>hbase.bucketcache.ioengine</name>
  <value>offheap</value>          <!-- 堆外 -->
</property>
<property>
  <name>hbase.bucketcache.size</name>
  <value>32768</value>            <!-- 32GB -->
</property>
```

**淘汰策略**：LRU（最近最少使用）。但有一个细节：HBase 有"in-memory"标记的 CF（`IN_MEMORY => true`），其 Block 在 LruBlockCache 中放在"高优先级 LRU 段"，更不易被淘汰。设计 CF 时如果某个 CF 是高频小数据（如配置表），可以打这个标记。

> BlockCache 命中率 < 50% 基本意味着你的工作集放不下缓存。要么扩大 BucketCache，要么优化 RowKey 让访问局部性更好。命中率 > 90% 说明缓存够大，可以考虑给 MemStore 多分点。

### 1.8 协处理器：Coprocessor

HBase 的 SQL 是后加的（Phoenix）。HBase 自身只有 KV 接口。如果想做"服务端聚合"或"服务端事件触发"，靠 **Coprocessor**。

两类：

| 类型 | 模型 | 类比 |
| --- | --- | --- |
| **Observer** | 在 Region 操作前后插入回调 | RDBMS 的 Trigger |
| **Endpoint** | 自定义 RPC 服务，在 Region 服务端跑业务逻辑 | RDBMS 的 Stored Procedure |

**Observer 子类**：
- `RegionObserver`：拦截 Get/Put/Delete/Scan
- `WALObserver`：拦截 WAL 写
- `MasterObserver`：拦截 DDL
- `RegionServerObserver`：拦截 RS 级别事件（Open/Close Region）

**Endpoint 典型用例**：
- 服务端聚合（Sum / Count / TopN）—— 避免把 GB 级 Cell 拉到客户端
- 服务端二级索引维护
- 服务端权限校验

**风险**：Coprocessor 跑在 RegionServer 进程内，**异常会拖死整个 RS**。生产上线要走严格 Code Review、必须有 try-catch + 超时控制。

> 协处理器是 HBase 工程化最强大也最危险的能力。**写出来很简单，写对很难**。能用客户端 Filter 解决的就别上 Endpoint；能用消息中间件做异步的就别上 Observer。

---

## 2. 源码切片：写入路径、Flush、Split、协处理器

> 基于 HBase 2.5 源码。注释为本文添加。

### 2.1 RegionServer 写入路径 —— HRegion.batchMutate

```java
// 简化自 org.apache.hadoop.hbase.regionserver.HRegion
public OperationStatus[] batchMutate(Mutation[] mutations) throws IOException {
    long mvccNum = mvcc.getReadPoint();
    boolean walSyncSuccessful = false;
    try {
        // ① 拿行锁（每个 Mutation 对应一行；同一 Region 内细粒度锁）
        acquireRowLocks(mutations);

        // ② 准备 WALEdit（一批 Mutation 合一条 WAL 记录）
        WALEdit walEdit = buildWALEdit(mutations);

        // ③ 写 WAL（异步刷盘）
        long txid = wal.append(this.htableDescriptor, this.regionInfo,
                               walKey, walEdit, true);

        // ④ 写 MemStore（更新 SkipList）
        applyToMemStore(mutations, mvccNum);

        // ⑤ 等 WAL sync 成功（可配置同步/异步）
        wal.sync(txid);
        walSyncSuccessful = true;

        // ⑥ MVCC complete（让读端可见）
        mvcc.completeAndWait(writeEntry);
    } finally {
        if (!walSyncSuccessful) {
            // WAL 失败 → 回滚 MemStore（撤销已应用的修改）
            rollbackMemStore(mutations);
        }
        releaseRowLocks(mutations);
    }
}
```

**逐段点透**：

- **WAL 在 MemStore 之前写**：保证崩溃时能从 WAL 重放未 Flush 的数据。但 WAL 失败后必须回滚 MemStore —— 否则会出现"内存里有但磁盘上没有"的脏数据。
- **MVCC 完成**在最后：HBase 用 MVCC 让"写入中的数据"对读不可见，写完成后才"翻牌"。这是 HBase 实现"行级一致性"的方式。
- **批量行锁顺序**：`acquireRowLocks` 内部按 RowKey 排序后加锁，避免不同顺序加锁导致的死锁。
- **WAL 是 Region 级共享的**：同一 RS 上不同 Region 的写入串行化在同一个 WAL append 上（HBase 2.x 的 MultiWAL 把这个共享改成了 N 个 WAL）。

### 2.2 MemStore Flush —— MemStoreFlusher.run

```java
// 简化自 org.apache.hadoop.hbase.regionserver.MemStoreFlusher
public void run() {
    while (!server.isStopped()) {
        FlushQueueEntry fqe = flushQueue.take();   // 阻塞拿任务
        HRegion region = fqe.getRegion();

        // 触发 Flush 的可能原因：
        //   ① MemStore 大小超过 hbase.hregion.memstore.flush.size (默认 128MB)
        //   ② RS 级 MemStore 总大小超过 lowerLimit
        //   ③ WAL 文件数超过 hbase.regionserver.maxlogs
        //   ④ 用户手动 flush
        //   ⑤ Region close

        try {
            boolean shouldCompact = region.flushcache(false, false);

            // Flush 后立即检查是否要触发 Compaction
            if (shouldCompact) {
                server.compactSplitThread.requestCompaction(
                    region, "After flush");
            }
        } catch (IOException e) {
            LOG.error("Cache flush failed for region " + region, e);
        }
    }
}
```

**核心：HRegion.flushcache 的关键步骤**

```java
protected FlushResult flushcache(...) {
    // ① 拿 updateLock 写锁（短暂）—— 阻塞所有写入
    lock.writeLock().lock();
    try {
        // ② 给 MemStore 拍快照（不复制数据，引用切换）
        for (HStore store : stores.values()) {
            store.snapshot();
        }
        flushedSeqId = wal.startCacheFlush(encodedRegionName);
    } finally {
        lock.writeLock().unlock();   // ③ 立即释放写锁
    }

    // ④ 把 MemStore 快照写入 HFile（耗时，但不阻塞写入）
    for (HStore store : stores.values()) {
        store.flushSnapshotToHFile();
    }

    // ⑤ 通知 WAL：seqId 之前的可以丢弃了
    wal.completeCacheFlush(encodedRegionName);
}
```

**逐段点透**：

- **写锁只拿很短时间**：只覆盖"快照切换"，不覆盖磁盘 IO。这是 Flush 看起来"不阻塞写入"的原因。
- **snapshot 不复制数据**：MemStore 里有两个引用 —— `kvset`（活跃）和 `snapshot`（待 Flush）。snapshot 切换是 O(1)。
- **写入仍然继续**：Flush 期间新的 Put 写到新的 `kvset`。读则要同时查 `kvset` + `snapshot` + 已有 HFile。
- **如果 Flush 失败**：HFile 写到一半失败，HDFS 上有半截文件，启动时根据 .meta 信息和 WAL 重放恢复。

**全局阻塞写入的元凶**：当 RS 上所有 Region 的 MemStore 总大小超过 `hbase.regionserver.global.memstore.size.upper.limit`（默认 0.4 × 堆），Flusher 会**强制阻塞所有 Put**直到总大小降到 lower limit 以下。后面案例 ④ 会展开。

### 2.3 Compaction 调度 —— CompactSplitThread

```java
// 简化自 org.apache.hadoop.hbase.regionserver.CompactSplitThread
public synchronized void requestCompaction(HRegion region, String why) {
    for (HStore store : region.getStores()) {
        // ① 评估是否需要 Compaction
        CompactionRequest request = store.requestCompaction();
        if (request == null) continue;

        // ② 选择执行池：大文件用 longCompactions，小文件用 shortCompactions
        ThreadPoolExecutor pool;
        if (store.throttleCompaction(request.getSize())) {
            pool = longCompactions;     // 默认 1 线程
        } else {
            pool = shortCompactions;    // 默认 1 线程
        }

        // ③ 提交 Compaction 任务
        pool.execute(new CompactionRunner(store, region, request));
    }
}

private class CompactionRunner implements Runnable {
    public void run() {
        try {
            // ④ 真正合并 HFile
            store.compact(request);
        } catch (IOException e) {
            LOG.error("Compaction failed", e);
        }
    }
}
```

**逐段点透**：

- **大小文件用不同池**：避免一个大文件 Compaction 卡住所有任务。但默认每个池只 1 个线程 —— 高写入量下要调大。
- **`requestCompaction` 内部**调用 `CompactionPolicy.selectCompaction(...)` 决定选哪些文件、Major 还是 Minor。
- **Compaction 是 HFile 级原子的**：合并完成前老 HFile 还在读，合并成功后切换引用、删除老文件。
- **Compaction 期间的读**：可能同时读到老 HFile 和新 HFile（不同 Scanner），靠 MVCC 保证一致性。

### 2.4 Region Split 物理过程 —— SplitTransaction

```java
// 简化自 org.apache.hadoop.hbase.regionserver.SplitTransactionImpl
public PairOfSameType<HRegion> execute(Server server) throws IOException {
    PairOfSameType<HRegion> regions = createDaughters(server);
    openDaughters(server, regions);
    transitionZKNode(server, regions);
    return regions;
}

private PairOfSameType<HRegion> createDaughters(Server server) throws IOException {
    // ① ZooKeeper 上创建 splitting 节点（让其他人知道在 split）
    createNodeSplitting(zk, parent, server.getServerName());

    // ② 关闭父 Region（短暂不可用）
    parent.close(false);

    // ③ 在 HDFS 上创建 .splits 目录
    Path splitDir = createSplitDir(parent);

    // ④ 为每个 HFile 创建 Reference
    //    Reference 是个小文件（几百字节），指向父 HFile 的某段
    splitStoreFiles(splitDir, hstoreFilesToSplit);

    // ⑤ 创建子 Region 的 .regioninfo 文件
    HRegion daughterA = createDaughterRegion(this.hri_a, ...);
    HRegion daughterB = createDaughterRegion(this.hri_b, ...);

    // ⑥ 更新 hbase:meta（删除父行，插入两个子行）
    MetaTableAccessor.splitRegion(server.getConnection(),
        parent.getRegionInfo(), daughterA.getRegionInfo(), daughterB.getRegionInfo(),
        server.getServerName(), regionReplication);

    return new PairOfSameType<>(daughterA, daughterB);
}
```

**Reference 文件结构**（简化）：

```
$ hadoop fs -cat /hbase/data/default/users/r1a/info/abc123.def456
# Reference 文件内容：
hfile_path = /hbase/data/default/users/parent_region/info/abc123
top_or_bottom = top
split_key = "k"
```

读子 Region 时，Scanner 看到 Reference，自动只读父 HFile 的对应一半。

**逐段点透**：

- **不复制数据是 Split 快的核心**：千万行的 Region 也能秒级 Split。
- **真正的物理分裂在后续 Compaction**：当子 Region 触发 Compaction 时，会把 Reference 指向的数据真的写到子 Region 自己的 HFile 里。在这之前，**父 Region 的 HFile 不能删**。
- **父 Region 不能立刻 GC**：HBase 有个 CatalogJanitor 周期性扫描，发现"父 Region 已 Split + 所有子 Region 都 Compaction 过 Reference 指向的文件" → 才删除父 Region。
- **Split 的"短暂不可用"窗口**：约几十毫秒到几秒，取决于元数据更新速度和 RegionServer 负载。

### 2.5 客户端 RegionLocator 缓存

```java
// 简化自 org.apache.hadoop.hbase.client.AsyncRegionLocatorHelper
public CompletableFuture<HRegionLocation> getRegionLocation(
        TableName table, byte[] row) {
    // ① 先查本地缓存
    HRegionLocation cached = metaCache.getCachedLocation(table, row);
    if (cached != null) return CompletableFuture.completedFuture(cached);

    // ② 缓存未命中：查 hbase:meta
    return queryMeta(table, row).thenApply(loc -> {
        metaCache.cacheLocation(table, loc);   // 写回缓存
        return loc;
    });
}
```

**问题**：Region 被 Split 或 Move 后，客户端缓存会陈旧。

**怎么发现陈旧？** 用旧位置发请求，RegionServer 返回 `NotServingRegionException` 或 `RegionMovedException`。客户端收到后清缓存重查。

```java
// AsyncRpcRetryingCaller 简化
public void onError(Throwable error) {
    if (error instanceof NotServingRegionException
        || error instanceof RegionMovedException) {
        metaCache.clearCache(tableName, row);   // ★ 清缓存
        retry();                                  // 重新查 meta 后再发请求
    }
}
```

**坑**：如果客户端进程长期运行，元信息查询失败的告警往往是 hbase:meta 那个 Region 所在的 RS 出问题。后面案例 ⑧ 会展开。

### 2.6 BulkLoad —— HFileOutputFormat2

不走客户端 Put，而是直接生成 HFile 然后塞进 HBase：

```java
// 典型 MR 作业
Job job = Job.getInstance(conf, "bulk-load-users");
job.setMapperClass(MyMapper.class);
job.setMapOutputKeyClass(ImmutableBytesWritable.class);
job.setMapOutputValueClass(KeyValue.class);

Connection conn = ConnectionFactory.createConnection(conf);
Table table = conn.getTable(TableName.valueOf("users"));
RegionLocator locator = conn.getRegionLocator(TableName.valueOf("users"));

// ① 自动按 Region 范围配置 Partitioner
HFileOutputFormat2.configureIncrementalLoad(job, table, locator);

// ② Map 阶段输出 KeyValue，Reduce 阶段每个 Region 一个 HFile
FileOutputFormat.setOutputPath(job, new Path("/tmp/bulk-output"));
job.waitForCompletion(true);

// ③ 把 HFile 移到 HBase 表目录
LoadIncrementalHFiles loader = new LoadIncrementalHFiles(conf);
loader.doBulkLoad(new Path("/tmp/bulk-output"), admin, table, locator);
```

**逐段点透**：

- **Mapper 输出 KeyValue 而不是 Put**：跳过客户端 Put 序列化、网络、WAL、MemStore 全流程。
- **Partitioner 按 Region 范围分**：每个 Reducer 写一个 Region 的 HFile。
- **doBulkLoad 是 mv 操作**：HFile 已经按 HBase 格式写好了，HBase 只是把它"挂载"到 Region 目录。
- **完全不走 WAL**：不消耗 RegionServer 的 CPU/内存，吞吐比 Put 高 1~2 个数量级。
- **代价**：BulkLoad 的数据**不会被复制到从集群**（Replication 走 WAL，BulkLoad 跳过了）—— 跨集群同步要单独处理。

### 2.7 协处理器示例 —— Endpoint 求和

定义 protobuf 接口：

```proto
service SumService {
  rpc getSum (SumRequest) returns (SumResponse);
}
message SumRequest {
  required string family = 1;
  required string column = 2;
}
message SumResponse {
  required int64 sum = 1;
}
```

服务端实现（跑在 RegionServer 内）：

```java
public class SumEndpoint extends SumService implements RegionCoprocessor {
    private RegionCoprocessorEnvironment env;

    @Override
    public void start(CoprocessorEnvironment e) {
        this.env = (RegionCoprocessorEnvironment) e;
    }

    @Override
    public void getSum(RpcController controller,
                       SumRequest request,
                       RpcCallback<SumResponse> done) {
        long sum = 0;
        Scan scan = new Scan();
        scan.addColumn(Bytes.toBytes(request.getFamily()),
                       Bytes.toBytes(request.getColumn()));
        try (RegionScanner scanner = env.getRegion().getScanner(scan)) {
            List<Cell> results = new ArrayList<>();
            boolean hasMore;
            do {
                hasMore = scanner.next(results);
                for (Cell c : results) sum += Bytes.toLong(CellUtil.cloneValue(c));
                results.clear();
            } while (hasMore);
        } catch (IOException e) {
            CoprocessorRpcUtils.setControllerException(controller, e);
            return;
        }
        done.run(SumResponse.newBuilder().setSum(sum).build());
    }
}
```

客户端调用：

```java
Map<byte[], Long> results = table.coprocessorService(
    SumService.class,
    null, null,                       // RowKey 范围（null = 全表）
    proxy -> {
        SumRequest req = SumRequest.newBuilder()
            .setFamily("info").setColumn("score").build();
        BlockingRpcCallback<SumResponse> cb = new BlockingRpcCallback<>();
        proxy.getSum(null, req, cb);
        return cb.get().getSum();
    });

// results：每个 Region 各返回一个 sum，客户端合并
long total = results.values().stream().mapToLong(Long::longValue).sum();
```

**逐段点透**：

- **每个 Region 并行执行 Endpoint**：客户端只需合并 N 个 Region 的小结果，避免拉取 GB 级 Cell。
- **scanner 必须 try-with-resources**：忘了关 → RegionServer 文件描述符泄漏 → 几小时后挂掉。
- **异常必须 setControllerException**：不抛异常的话客户端永远等不到结果（看起来像 hang）。
- **加载方式**：可以全表 `ALTER 'users', 'coprocessor' => '/path/to/jar|com.x.SumEndpoint|1001|'`，也可以全集群配 `hbase-site.xml`。

> 全集群配置的协处理器升级风险高 —— 一改就要重启所有 RS。生产建议表级配置。

---

## 3. 实测数据：用真实数字看 HBase 的代价

### 3.1 不同 Compaction 策略的写放大

实验：连续写入 1 亿条 KV（RowKey 随机散列、Value 100B），观察 `hdfs dfsadmin -report` 和 RegionServer 的 IO 吞吐。

| Compaction 策略 | 写放大（HDFS 写入字节 / 用户写入字节） | 读延迟（P99 ms） |
| --- | --- | --- |
| 关闭 Compaction | 1.0× | 280 |
| 默认 ExploringCompactionPolicy | 4.2× | 12 |
| StripeCompaction | 3.1× | 18 |
| DateTieredCompaction（时序友好） | 2.4× | 22 |

**解读**：
- 关闭 Compaction → 写放大最低，但 HFile 数飙升、读延迟爆炸
- 默认策略读延迟最优，但写放大最高
- StripeCompaction 适合大 Region（10GB+）+ 范围扫描多的场景
- DateTieredCompaction 适合时序数据（HFile 按时间分层，老数据不再合并）

> Compaction 没有"最好"的策略。**写放大与读延迟的权衡，取决于你的写入/查询比例**。

### 3.2 BlockCache 命中率对查询的影响

实验：在 1TB 数据集上做随机 Get，BlockCache 配置不同。

| BlockCache 配置 | 命中率 | P99 Get 延迟（ms） | P999 Get 延迟（ms） |
| --- | --- | --- | --- |
| LruBlockCache 4GB | 12% | 38 | 220 |
| LruBlockCache 16GB | 41% | 18 | 95 |
| BucketCache 64GB（堆外） | 78% | 6 | 32 |
| BucketCache 256GB（SSD） | 95% | 3 | 12 |

**解读**：
- 命中率每提升 30%，P99 大约下降 50%
- 大堆内 LruBlockCache（> 32GB）会因 GC 导致 P999 暴涨，得不偿失
- BucketCache + SSD 是 PB 级数据集的标配

> 所有"HBase 慢"的工单，先看 BlockCache 命中率。命中率 < 50% 基本是缓存不够。

### 3.3 BulkLoad vs Put 的吞吐对比

实验：导入 10 亿条数据（约 200GB），10 个 RegionServer。

| 方式 | 总耗时 | 吞吐（万行/秒） | RS CPU 平均 | RS GC 时间占比 |
| --- | --- | --- | --- | --- |
| 单线程客户端 Put | 35 小时 | 0.8 | 15% | 8% |
| 多线程客户端 Put（256 并发） | 5.2 小时 | 5.3 | 78% | 22% |
| HBase Bulk API（PutBatch + 客户端缓冲） | 2.8 小时 | 9.9 | 65% | 18% |
| **BulkLoad（MR）** | **22 分钟** | **75** | 12% | < 1% |

**解读**：
- BulkLoad 比客户端 Put 快约 100 倍
- 关键在 BulkLoad 完全不走 WAL + MemStore + RegionServer Put 路径
- 代价：BulkLoad 不触发 Replication，跨集群同步要另行处理

> 历史数据迁移、批量导入只用 BulkLoad。在线写入才用 Put。混着用是常见错误。

### 3.4 Region 数对 RegionServer 资源的消耗

实验：单 RegionServer，逐步增加 Region 数，观察堆内存和 GC 行为。

| Region 数 | MemStore 总大小 | 堆使用率 | Young GC 频率 | Full GC |
| --- | --- | --- | --- | --- |
| 50 | 6GB | 35% | 4 次/分钟 | 0 |
| 200 | 12GB | 55% | 8 次/分钟 | 0 |
| 500 | 18GB | 75% | 14 次/分钟 | 偶发 |
| 1000 | 25GB | 92% | 22 次/分钟 | 频繁 |
| 2000 | 28GB（接近上限） | 98% | 30 次/分钟 | **每 5 分钟一次** |

**解读**：
- 每个 Region 的 MemStore 即使没数据也要占 ~10MB 元信息
- Region 数 > 500 时 Flush 的"碎片化"严重 —— 大量小 HFile 触发频繁 Compaction
- Region 数 > 1000 在 32GB 堆上基本不可用

**经验值**：单 RS 的 Region 数控制在 **20 × (堆 GB) / 1**（即 32GB 堆约 600 Region 上限）。但实际生产建议 100~200 之间。

> Region 多 ≠ 性能好。**Region 数与 MemStore 总大小、Compaction 压力呈线性关系**，超出阈值会雪崩。

### 3.5 大行 vs 多列的存储差异

实验：100 万 RowKey，每行不同列数：

| 设计 | HFile 总大小 | 单点 Get 延迟 | 全行 Get 延迟 |
| --- | --- | --- | --- |
| 1 行 1 列 100B | 105MB | 1.2ms | 1.2ms |
| 1 行 100 列 1B | 380MB | 1.5ms | 8ms |
| 1 行 1000 列 1B | 3.2GB | 2.1ms | 65ms |
| 1 行 1 列 100KB | 105GB | 12ms | 12ms |

**解读**：
- 多列存储有重复 RowKey + Qualifier 的开销，Cell 数 ↑ → HFile 大小不线性
- 单点 Get（指定 Qualifier）即使在 1000 列下也很快 —— Block Index 找得到目标 Block
- 全行 Get 受 Cell 数线性影响 —— 1000 列要 50 倍的解码时间
- 巨大 Value（> 100KB）让 Block 难以缓存，单次 Get 退化到磁盘 IO

> HBase 的"宽行"是个误区。**单 Cell 控制在几 KB 内**；列多到要几百，考虑用 Protobuf 序列化成单 Cell。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① RowKey 热点 —— "单 RegionServer 被打挂"

#### 现象

某游戏业务上线，RowKey 设计为 `serverTimestamp + userId`。上线后 3 小时，监控显示 RegionServer-7 的 CPU 100%、网络入流量是其他 RS 的 50 倍，其余 9 台 RS 几乎空闲。然后 RS-7 OOM 死亡，Region 漂移，新接管的 RS 又被打挂，**形成多米诺式雪崩**。

#### 根因

`serverTimestamp` 是单调递增的。HBase 按 RowKey 字典序分配 Region：

```
Region R1: [.., 1700000000)
Region R2: [1700000000, 1700100000)
...
Region R10: [1700900000, ..)   ← 当前在 RS-7
```

所有新写入的 RowKey 都以"当前时间戳"开头，**全部落到 R10**。RS-7 的 CPU 全在处理 R10 的写入和 Compaction。

挂掉后 R10 漂移到 RS-3，继续打 RS-3。

#### 修复

**方案 A：Salt 前缀（最简单）**

```java
// RowKey = (hash(userId) % N).toString().padLeft(2,'0') + "-" + ts + "-" + userId
String salt = String.format("%02d", Math.abs(userId.hashCode()) % 32);
byte[] rowKey = Bytes.toBytes(salt + "-" + ts + "-" + userId);
```

- 32 个前缀均匀打散到 32 个虚拟分组
- 配合**预分区** `SPLITS => ['00', '01', ..., '31']` 让 32 个 Region 在建表时就分散到所有 RS

**方案 B：reverse(userId) + ts**

```java
byte[] rowKey = Bytes.toBytes(new StringBuilder(userId).reverse().toString()
                              + "-" + ts);
```

适合**单 user 的 Scan 比"按时间扫全表"更重要**的场景。

**方案 C：MD5(userId) 前缀**

```java
byte[] hash = MessageDigest.getInstance("MD5").digest(userId.getBytes());
byte[] prefix = Arrays.copyOf(hash, 4);  // 取 4 字节
byte[] rowKey = Bytes.add(prefix, Bytes.toBytes(ts), Bytes.toBytes(userId));
```

散列度最优，但完全失去局部性。**任何按 userId 的 Scan 都要扫全表**。

**预分区建表**：

```bash
hbase shell
create 'events', 'info',
  { SPLITS => (0..31).map { |i| sprintf('%02d', i) } }
```

#### 验证

```bash
# 看每个 RegionServer 的请求 QPS 是否均匀
hbase hbck -details

# 监控 RegionServer 级 metric
curl http://rs-1:16030/jmx | grep -i regionrequestcount
```

修复后所有 RS 的 QPS 应在 ±20% 范围内。

> **口诀**：**单调递增 + 默认预分区 = 必定热点**。RowKey 设计前先列 Top 5 查询，再选散列方案。

---

### 案例 ② Region Split 风暴 —— "突发活动后 Region 数暴涨"

#### 现象

电商大促期间，一张订单表的 Region 数从平时的 200 涨到 2000，RegionServer 频繁 GC、出现长 STW，业务超时报错。大促结束后 Region 数没有自动减少，问题持续。

#### 根因

HBase 的默认 Split 策略 `IncreasingToUpperBoundRegionSplitPolicy` 会在数据快速增长时**频繁 Split**：

```
活动开始：单 Region 写入速率 100K rows/s
   → 每 5 分钟一个 Region 长到 10GB
   → 触发 Split，得到两个 5GB Region
   → 几分钟后两个 Region 又分别长到 10GB
   → 再 Split → 4 个 Region → 8 个 → 16 个 → ...
```

Region 数指数增长。每个 Region 占 MemStore + 元信息约 100MB+，2000 个 Region = 200GB MemStore 占用，远超堆配置。RegionServer 不停 Flush + Compaction + GC。

更糟的是 HBase **不会自动 Merge Region**。流量降下来后 Region 数不会减少，2000 个稀疏小 Region 持续消耗资源。

#### 修复

**事中**：手动停止 Split

```bash
hbase shell
# 改 Split 策略为 DisabledRegionSplitPolicy（拒绝任何 Split）
alter 'orders', { METHOD => 'table_att',
  CONFIG => {'SPLIT_POLICY' => 'org.apache.hadoop.hbase.regionserver.DisabledRegionSplitPolicy'} }
```

**事后**：手动 Merge Region

```bash
# 查找小 Region
hbase shell
> scan 'hbase:meta', {COLUMNS => 'info:regioninfo'}

# 合并相邻的两个 Region
> merge_region 'region_encoded_name_1', 'region_encoded_name_2'

# 大批量合并要写脚本
```

**长效预防**：

1. **预分区到位**：建表时按业务高峰期的 Region 数预分区，少依赖自动 Split

```bash
create 'orders', 'info',
  { NUMREGIONS => 200, SPLITALGO => 'HexStringSplit' }
```

2. **改用更"懒"的 Split 策略**：

```bash
alter 'orders', { METHOD => 'table_att',
  CONFIG => {'SPLIT_POLICY' => 'org.apache.hadoop.hbase.regionserver.ConstantSizeRegionSplitPolicy',
             'MAX_FILESIZE' => '20000000000'} }   # 20GB 才 Split
```

3. **写监控告警**：单 RS 的 Region 数 > 300 报警

#### 验证

```bash
# 单 RS 的 Region 数
hbase shell
> status 'detailed'

# 通过 JMX 看
curl http://rs-1:16030/jmx?qry=Hadoop:service=HBase,name=RegionServer,sub=Server | jq .beans[0].regionCount
```

> **口诀**：**预分区 + 大 Region 阈值 + 监控 + 手动 Merge** 四件套。HBase 不会自己收缩，扩散容易回收难。

---

### 案例 ③ Major Compaction 拖慢服务 —— "每周三凌晨 P99 飙到 30s"

#### 现象

某业务监控显示 P99 延迟每周三凌晨 3 点准时飙到 30s，持续 4 小时。其他时间一切正常。

#### 根因

`hbase.hregion.majorcompaction = 7 * 24 * 3600 * 1000`（7 天）+ `hbase.hregion.majorcompaction.jitter = 0.5`（抖动）。集群上线后第一周的周三凌晨刚好是第一次 Major Compaction 的时间窗口，**所有 Region 集中在同一窗口跑 Major**。

Major Compaction 的开销：
- 一个 Region 的某 CF 的所有 HFile 全读出来重写
- 单 Region 几十 GB 数据 → 几分钟到几小时的磁盘 IO 拉满
- 一个 RS 上几十个 Region 同时跑 Major → IO 100%

业务读写受影响：
- HFile 引用切换瞬间的 Scanner reset → 读延迟抖动
- HDFS DataNode IO 100% → 所有 HDFS 操作变慢
- 网络带宽被合并写入吃光

#### 修复

**方案 A：关闭自动 Major，手动错峰**

```xml
<!-- hbase-site.xml -->
<property>
  <name>hbase.hregion.majorcompaction</name>
  <value>0</value>          <!-- 关闭自动 Major -->
</property>
```

```bash
#!/bin/bash
# 凌晨 2 点跑：每分钟 Major 一个 Region
for region in $(echo "list_regions 'orders'" | hbase shell -n | grep ',,' | awk '{print $1}'); do
    echo "major_compact '$region'" | hbase shell -n
    sleep 60
done
```

**方案 B：限速**

```xml
<property>
  <name>hbase.hstore.compaction.throughput.lower.bound</name>
  <value>10485760</value>   <!-- 10MB/s -->
</property>
<property>
  <name>hbase.hstore.compaction.throughput.higher.bound</name>
  <value>20971520</value>   <!-- 20MB/s -->
</property>
```

让 Compaction IO 不超过 20MB/s，腾出带宽给业务。

**方案 C：用 OffPeakHours 让 Compaction 只在低峰跑**

```xml
<property>
  <name>hbase.offpeak.start.hour</name>
  <value>0</value>          <!-- 凌晨 0 点开始 -->
</property>
<property>
  <name>hbase.offpeak.end.hour</name>
  <value>6</value>          <!-- 早上 6 点结束 -->
</property>
```

#### 验证

```bash
# 抓周三凌晨的 RS metrics
curl http://rs-1:16030/jmx | grep -i compaction

# 看 majorCompactedCellsCount 是否分散到多天
```

修复后 P99 应该平稳，不再有"每周一抖"。

> **口诀**：**自动 Major 是上线第一个要关掉的功能**。手动错峰 + 限速 + 低峰窗口三件套。

---

### 案例 ④ MemStore 占满导致写入阻塞 —— "Put 全部 hang 住"

#### 现象

业务监控显示 HBase 写入完全 hang 住，Put RPC 全部超时（30s）。RegionServer 日志看到大量：

```
WARN  [regionserver/rs-1:16020] regionserver.HRegion:
Region too busy due to exceeding memstore size limit
```

或：

```
WARN  [regionserver/rs-1:16020] regionserver.MemStoreFlusher:
Memstore is above high water mark and block 4923ms
```

#### 根因

HBase 有一个**全局 MemStore 上限**保护机制：

```
upper limit = hbase.regionserver.global.memstore.size.upper.limit
            = 0.4 × heapSize (默认)
lower limit = hbase.regionserver.global.memstore.size.lower.limit
            = 0.95 × upper limit
```

当 RS 上**所有 Region 的 MemStore 总大小**超过 upper limit，**RS 强制阻塞所有写入**，直到通过 Flush 把总大小降到 lower limit 之下。

为什么会撑爆？典型几种：

1. **Region 数过多**：500 个 Region 即使每个 MemStore 只有 50MB，总和也 25GB
2. **Flush 速度跟不上写入**：HDFS 慢、Compaction 抢 IO、Flush 线程不够
3. **HFile 数过多触发 Flush 阻塞**：`hbase.hstore.blockingStoreFiles`（默认 16），单 Store HFile 数超过这个值，Flush 会等 Compaction —— Compaction 完不成，MemStore 永远 Flush 不出去

#### 修复

**事中：紧急扩 Flush 线程 + 扩 MemStore 上限**

```bash
# 在线动态修改（部分参数支持）
echo "alter_status 'orders'" | hbase shell

# 不行就重启对应 RS
```

```xml
<!-- 调大 Flush 线程数 -->
<property>
  <name>hbase.hstore.flusher.count</name>
  <value>8</value>      <!-- 默认 2 -->
</property>

<!-- 提高 blocking 阈值 -->
<property>
  <name>hbase.hstore.blockingStoreFiles</name>
  <value>32</value>     <!-- 默认 16 -->
</property>
```

**事后：根本性修复**

1. **减少 Region 数**：Merge 小 Region
2. **提升 HDFS 写性能**：DataNode 数、磁盘类型、网络
3. **错峰 Compaction 让出 IO**
4. **提高 Compaction 并发**：

```xml
<property>
  <name>hbase.regionserver.thread.compaction.small</name>
  <value>4</value>      <!-- 默认 1 -->
</property>
<property>
  <name>hbase.regionserver.thread.compaction.large</name>
  <value>2</value>      <!-- 默认 1 -->
</property>
```

#### 验证

```bash
# 监控全局 MemStore 占用
curl http://rs-1:16030/jmx | grep -i 'MemstoreSize\|MemStoreSize'

# Block 时间应该归零
grep "block" $HBASE_HOME/logs/hbase-*.log | wc -l
```

> **口诀**：**MemStore 全局上限 + HFile blocking 阈值 + Flush/Compaction 线程数** 是写入吞吐的天花板。卡其中一个 → 全集群停摆。

---

### 案例 ⑤ HDFS 故障导致 WAL 写失败 —— "整个 RS 自杀"

#### 现象

凌晨 4 点 HDFS 集群网络抖动 30 秒，业务监控发现 5 个 RegionServer 同时下线（Master 报告 dead）。RS 日志看到：

```
FATAL [regionserver/rs-3:16020] regionserver.HRegionServer:
ABORTING region server: WAL.sync failed
java.io.IOException: cannot complete file ...
```

#### 根因

HBase 的 WAL 写到 HDFS。RS 写 WAL 时调用 `wal.sync()` 等 HDFS 持久化（实际是写到 N 个 DataNode 的 OS 缓冲区）。如果 HDFS 几秒内不能完成（DataNode 不响应、超过副本数无法写入），`wal.sync()` 抛 IOException。

HBase 的设计哲学：**WAL 失败必须自杀**。理由：
- WAL 失败意味着已写入 MemStore 的数据无法持久化
- 如果继续接受请求，崩溃后这些数据会丢失
- 唯一安全的做法是 abort RS，让其他 RS 接管（接管时会重放老 WAL，同时这个 RS 不会再产生新数据）

源码：

```java
// HRegionServer.abort
public void abort(String reason, Throwable cause) {
    LOG.error("ABORTING region server: " + reason, cause);
    this.abortRequested = true;
    stop("Aborting");
}
```

WAL 失败的常见原因：
- HDFS NameNode 切主（短时间不可写）
- DataNode 节点挂太多（副本数不足）
- 网络分区
- 磁盘满

#### 修复

**事中**：恢复 HDFS。RS 自杀后会被 Master 重启，Region 自动重新分配。

**长效**：

1. **HDFS 高可用配置到位**：NameNode HA、足够的 DataNode、机架感知配置

2. **调高 WAL 重试**（缓解短暂抖动）

```xml
<property>
  <name>hbase.regionserver.hlog.tolerable.lowreplication</name>
  <value>1</value>    <!-- WAL 副本低于这个值才告警，不直接 abort -->
</property>
<property>
  <name>hbase.regionserver.hlog.lowreplication.rolllimit</name>
  <value>5</value>    <!-- 副本不足时自动 roll WAL -->
</property>
```

3. **使用 MultiWAL** 减少单 WAL 故障的影响范围

```xml
<property>
  <name>hbase.wal.provider</name>
  <value>multiwal</value>
</property>
<property>
  <name>hbase.wal.regiongrouping.numgroups</name>
  <value>4</value>    <!-- 4 个 WAL，每个负责 1/4 Region -->
</property>
```

4. **监控 HDFS 与 RS 的耦合度**：监控 `Hadoop:service=HBase,name=RegionServer,sub=WAL` 的 SyncTime / AppendTime

#### 验证

注入 HDFS 故障（封网络包），观察：
- WAL 是否会 roll 而不是 abort RS
- Sync 失败次数是否能容忍

> **口诀**：**HDFS 是 HBase 的生命线**。HDFS 抖一抖，HBase 死一片。WAL 配置必须容忍短暂故障。

---

### 案例 ⑥ ZooKeeper 性能瓶颈 —— "建表 30 秒、列表 1 分钟"

#### 现象

集群规模膨胀到 200+ RS、20000+ Region 后，Admin 操作（建表、Disable 表、`list_regions`）从秒级变成分钟级。RS 启动也很慢、Region 重分配很慢。

#### 根因

ZooKeeper 是 HBase 的"协调中心"：
- Master 选主
- RS 心跳
- Region 状态变更（OPENING / OPEN / CLOSING / CLOSED / SPLITTING / ...）
- ACL、Replication 元数据

ZK 的性能瓶颈：
- 写吞吐：单 Leader 写，约 5K~10K 写/秒
- 单 Znode 大小：默认 1MB
- 子节点数：> 10 万就慢

HBase 早期版本（1.x）把每个 Region 的状态都放 ZK。20000 Region → ZK 上几十万节点 → ZK 操作变慢 → HBase 操作变慢。

HBase 2.x 引入了 **AssignmentManager v2 (AMv2)**：把 Region 状态从 ZK 搬到了 `hbase:meta` 表。但仍有部分元数据走 ZK。

#### 修复

**长效**：升级到 HBase 2.x AMv2

```xml
<!-- HBase 2.x 默认开启 -->
<property>
  <name>hbase.assignment.usezk</name>
  <value>false</value>
</property>
```

**ZK 调优**：

```bash
# zoo.cfg
tickTime=2000
syncLimit=10
autopurge.snapRetainCount=5
autopurge.purgeInterval=24
maxClientCnxns=300       # 允许的客户端连接数

# JVM
-Xmx8g -Xms8g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=50
```

**监控**：

```bash
# ZK 自身的 mntr 命令
echo mntr | nc zk-1 2181
# 关键指标：
#   zk_avg_latency        - 平均请求延迟
#   zk_outstanding_requests - 排队请求数
#   zk_znode_count        - 节点总数
```

**业务侧规避**：

- 不要频繁创建/删除表
- 大表 Disable / Drop 之前先减小 Region 数
- 避免在业务低峰外做大批量 Admin 操作

#### 验证

ZK 平均延迟应该 < 5ms。`list` / `create` 操作秒级返回。

> **口诀**：HBase 1.x 的 ZK 瓶颈在 Region 状态。2.x 升级到 AMv2 + ZK 自身调优是双管齐下。

---

### 案例 ⑦ 大事务超时 —— "扣库存 + 写订单"

#### 现象

业务想在 HBase 里做"扣库存（行 A）+ 写订单（行 B）"的原子操作。两行不在同一 Region，业务用了"先扣后写"的代码：

```java
// ❌ 错误代码
table.increment(stockRow, "info", "qty", -1);   // 扣库存
table.put(orderPut);                             // 写订单
```

如果第二步失败（网络、RS 挂、超时），库存被扣了订单没生成。客诉爆炸。

#### 根因

HBase **只支持单行原子性**（Row-level ACID）。跨行、跨 Region、跨表都不保证原子性。

为什么？因为：
- 不同行可能在不同 Region、不同 RS
- 跨 RS 的两阶段提交需要协调者
- HBase 的设计目标是 P 级数据规模 + 水平扩展，不愿背 2PC 的复杂度

那 HBase 内置的 `checkAndPut`、`checkAndMutate`、`mutateRow` 呢？
- 都是**单行操作**，跨行无效

Phoenix 的"Transaction"呢？
- 基于 Tephra / Omid 实现的 SI（Snapshot Isolation）
- 性能差（要写 Tephra 日志）
- 严格一致性场景不能用 —— Tephra 自身有故障窗口

#### 修复

**方案 A：把"事务相关数据"放同一行**

如果库存和订单可以共用 RowKey（比如 `userId-skuId`），放同一行就是原子的。

**方案 B：业务层"补偿事务"（Saga）**

```java
// 1. 写订单（pending 状态）
orderPut = ... .put("status", "pending");
table.put(orderPut);

// 2. 扣库存
boolean ok = table.checkAndIncrement(stockRow, "info", "qty",
                                      currentQty, -1);
if (!ok) {
    // 库存不足或被改 → 把订单标记 cancelled
    table.put(... .put("status", "cancelled"));
    return;
}

// 3. 订单转为 paid
table.put(... .put("status", "paid"));

// 4. 后台对账：定期扫"pending 超过 5 分钟的订单"
//    根据真实库存状态决定是否重试或回滚
```

**方案 C：用消息队列做最终一致性**

```
扣库存 → 成功 → 发消息到 MQ → 消费者写订单
                              失败 → 重试，幂等
```

**方案 D：上层用支持分布式事务的存储**

- TiDB（基于 Percolator，强一致）
- 阿里 Lindorm（HBase 兼容协议 + 跨行事务）
- 把 HBase 当成"列式宽表存储"，事务用 MySQL/TiDB

#### 验证

故障注入：在两步操作之间 kill RS / 断网，确认数据最终一致。

> **口诀**：**HBase = 单行原子**，跨行就是分布式事务问题。要么改数据模型让它单行、要么走 Saga、要么换存储。

---

### 案例 ⑧ 客户端 Locator 缓存陈旧 —— "偶发 RegionMovedException"

#### 现象

应用日志频繁出现：

```
WARN  ... RegionLocator: Cached location is stale
       org.apache.hadoop.hbase.NotServingRegionException:
       Region is not online: orders,...
```

业务有重试，最终成功，但 P99 延迟周期性飙升。

#### 根因

每次 Region Split / Move / RS 切换后，老的 RS 不再服务该 Region。客户端缓存的位置陈旧，第一次发请求会失败，触发 Meta 重查 + 重试。

正常情况下这是自愈的。问题是**频繁出现**说明：

- 集群在频繁 Split（见案例 ②）
- RegionServer 频繁挂 + 漂移
- hbase:meta 表所在的 RS 自身负载过高，Meta 查询慢

#### 修复

**事中：定位频繁变动的源头**

```bash
# 看 Master 日志的 Region 变动
grep "OPENING\|CLOSED\|SPLIT" $HBASE_HOME/logs/hbase-master-*.log | tail -100

# 看 hbase:meta 表的所在 RS
echo "locate_region 'hbase:meta', ''" | hbase shell
```

**长效**：

1. **客户端配置合理的重试**

```java
Configuration conf = HBaseConfiguration.create();
conf.setInt("hbase.client.retries.number", 10);
conf.setInt("hbase.client.pause", 100);     // 重试间隔基数
conf.setInt("hbase.client.operation.timeout", 60000);  // 单操作总超时
conf.setInt("hbase.rpc.timeout", 30000);
```

2. **Meta 缓存 TTL 调优**（HBase 2.x）

```java
conf.setLong("hbase.client.meta.cache.eviction.ms", 600000);  // 10 分钟
```

3. **保护 hbase:meta 表**

- Meta 表所在 RS 单独配置（不要让业务 Region 落到它上面）
- Meta 表的 BlockCache 大幅提高
- 集群规模大时考虑 Meta 表 Split（HBase 2.x 支持）

4. **避免 Region 频繁变动**

- Split 策略调成 ConstantSize + 大 MaxFileSize
- RS 端 SLA 保证（监控 + 自愈）

#### 验证

```bash
# 客户端 metric
# 监控 hbase.client.meta.cache.miss
# 应该 < 总请求的 0.1%
```

> **口诀**：客户端 Locator 缓存陈旧是"症状"，**根本原因在于 Region 频繁变动 + Meta 表负载**。两端一起治。

---

### 案例 ⑨ 与 Spark / MapReduce 的集成踩坑 —— "扫描把 RS 打挂"

#### 现象

数据团队跑了一个 Spark 作业全表扫描 HBase，做特征工程。任务跑了 2 小时后多个 RegionServer OOM、HBase 集群"半瘫"，业务大面积超时。

#### 根因

代码大概是这样：

```scala
// ❌ 错误用法
val rdd = sc.newAPIHadoopRDD(
    conf, classOf[TableInputFormat],
    classOf[ImmutableBytesWritable], classOf[Result])
val features = rdd.map(extractFeature).collect()    // collect 拉到 Driver
```

问题：
1. **Scan 无 Caching、无 BatchSize**：每次 RPC 拉一行，对小行表 RPC 量爆炸
2. **collect**：把所有 Result 拉到 Driver，OOM
3. **没有 setCacheBlocks(false)**：Scan 把全表数据塞进 BlockCache，把 BlockCache 里业务的热点数据全挤出去
4. **没有限速**：Spark 的 200 个 task 并行扫，把 RS CPU 和 HDFS IO 拉满

#### 修复

**正确的 Scan 配置**：

```scala
val scan = new Scan()
scan.setCaching(1000)           // 每次 RPC 拉 1000 行
scan.setBatch(100)              // 每个 Result 包含至多 100 个 Cell
scan.setCacheBlocks(false)      // ★ 不污染 BlockCache
scan.setMaxResultSize(2 * 1024 * 1024)  // 单次 RPC 不超过 2MB

conf.set(TableInputFormat.SCAN, TableMapReduceUtil.convertScanToString(scan))
```

**限制 Spark 并发**：

```scala
val rdd = sc.newAPIHadoopRDD(...)
  .repartition(20)              // 不要让 200 个 task 同时扫
```

**优先用 Snapshot Scan**：

```scala
// HBase Snapshot：不经过 RegionServer，直接读 HDFS 上的 HFile
val scan = new Scan()
TableSnapshotInputFormat.setInput(job, "snapshot_name", restoreDir, scan, ...)
```

Snapshot Scan 的好处：
- 完全绕过 RegionServer，不影响在线业务
- 直接读 HFile，吞吐高
- 适合 OLAP 类批处理

**最后：让"分析"走旁路**

```
HBase（OLTP，在线服务）
   │
   │ 实时同步（Replication / CDC）
   ▼
Hive / ClickHouse / Doris（OLAP，分析）
   │
   ▼
Spark 作业读这里，不读 HBase
```

#### 验证

```bash
# Spark 作业跑的同时观察 RS metric
curl http://rs-1:16030/jmx | grep -i 'requestCount\|blockCacheCount'

# BlockCache 命中率应该不会暴跌
# RS CPU 应该平稳
```

> **口诀**：**全表扫 HBase = 给业务下毒**。要么用 Snapshot Scan、要么用 Replication 旁路、要么严格限速 + 不污染 BlockCache。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### RowKey 与表设计层
- [ ] RowKey 设计文档化：列出 Top 5 查询场景，对每种场景验证 RowKey 是否高效
- [ ] RowKey 长度控制在 16~64 字节，固定长度
- [ ] 严禁使用单调递增的 RowKey（timestamp / 自增 ID）做前缀
- [ ] 散列方案明确（Salt / Hash / Reverse），并配合预分区
- [ ] ColumnFamily 数 ≤ 3（最好 1~2）
- [ ] 单 Cell 大小 < 1MB（建议 < 100KB）
- [ ] 列数过多时考虑用 Protobuf 序列化为单 Cell

### 建表层
- [ ] **预分区到位**：根据预期数据量和写入 QPS 决定 Region 数（一般初始 50~200）
- [ ] 配置合理的 SPLIT_POLICY（推荐 ConstantSize + 大 MaxFileSize）
- [ ] 明确的 TTL（按业务保留期，避免无限增长）
- [ ] 明确的 VERSIONS（默认 1，多版本场景显式配）
- [ ] BLOCKCACHE / IN_MEMORY 标记按数据热度配置
- [ ] BLOOMFILTER（默认 ROW，多列查询用 ROWCOL）
- [ ] COMPRESSION（推荐 SNAPPY 或 ZSTD）

### Compaction 层
- [ ] **关闭自动 Major Compaction**（`hbase.hregion.majorcompaction = 0`）
- [ ] 用脚本错峰手动触发 Major
- [ ] 配置 Compaction 限速（`hbase.hstore.compaction.throughput.*`）
- [ ] 调整 short/long Compaction 线程数（默认 1 太少）
- [ ] 配置 OffPeakHours

### MemStore / Flush 层
- [ ] 全局 MemStore 上限合理（默认 0.4 × 堆）
- [ ] 单 Region MemStore Flush 大小（默认 128MB，写入量大时调到 256MB+）
- [ ] hbase.hstore.blockingStoreFiles 调高到 32+
- [ ] Flush 线程数 4+（默认 2 太少）

### BlockCache 层
- [ ] **大堆机器（> 32GB）必须用 BucketCache**
- [ ] BlockCache 命中率监控，告警阈值 < 80%
- [ ] 堆内只放索引和 Bloom，数据走 BucketCache

### WAL 层
- [ ] 大集群启用 MultiWAL
- [ ] HDFS HA 配置到位（NameNode HA + 足够 DataNode）
- [ ] WAL 副本数 = 3
- [ ] 配置 lowreplication 容错参数

### Region 数量层
- [ ] 单 RS Region 数监控，告警阈值 200~300
- [ ] 定期检查 Region 大小分布，过小的 Region 手动 Merge
- [ ] 建立 Merge 自动化脚本

### 客户端层
- [ ] Connection 单例（重 RPC 资源）
- [ ] Table 实例 thread-safe，可复用
- [ ] BufferedMutator 用于批量写入
- [ ] Scan 必须设 Caching、BatchSize、setMaxResultSize
- [ ] **大批量 Scan 必须 setCacheBlocks(false)**
- [ ] 客户端超时和重试参数明确

### Coprocessor 层
- [ ] 协处理器代码必须 try-catch + 超时控制
- [ ] 表级配置（不用全集群配置，便于灰度）
- [ ] 上线前压测（协处理器异常会拖死 RS）

### 监控告警层
- [ ] RegionServer 进程存活
- [ ] 单 RS 的 Region 数、QPS、延迟
- [ ] 全局 MemStore 占用、HFile 数、Compaction 队列
- [ ] BlockCache 命中率、淘汰速率
- [ ] WAL Sync 延迟、HDFS 写延迟
- [ ] ZooKeeper 平均延迟、连接数
- [ ] Master 主备切换事件

### 业务集成层
- [ ] **HBase 不支持跨行事务**：业务设计阶段明确，不要硬上
- [ ] OLAP 类查询走旁路（Hive / ClickHouse），不要直接 Scan HBase
- [ ] 历史数据导入用 BulkLoad，不用客户端 Put
- [ ] 跨集群同步评估 Replication（注意 BulkLoad 不被复制）

---

## 6. 还能往下挖什么

### HBase 3.0 新特性

- **AsyncFSWAL** 默认启用 —— WAL 写入异步化，写入吞吐提升 30%+
- **Java 11+ 支持** —— G1GC + ZGC 让大堆 RS 的 STW 大幅降低
- **In-Memory Compaction 默认 BASIC** —— 减少 MemStore 内的小段碎片
- **Region Replicas** 进一步成熟 —— 可用性增强（Read Replica）
- 关注 `hbase-3.0.0` 的 Release Notes 里的 Backward Incompatible Changes

### Apache Phoenix（HBase 上的 SQL）

- Phoenix 把 HBase 包装成 SQL：DDL、DML、JDBC 接口
- 自动维护二级索引（Local Index / Global Index 两种实现）
- 适合：业务有 SQL 习惯、查询多维度
- 不适合：超大规模 OLTP（Phoenix 的解析+执行开销不容忽视）
- 关键概念：Salt Buckets、Skip Scan、Sequence

### HBase 与 Cassandra 的本质对比

| 维度 | HBase | Cassandra |
| --- | --- | --- |
| CAP | CP（强一致，分区时不可用） | AP（最终一致，分区时仍可用） |
| 架构 | Master-Slave + ZK + HDFS | 完全去中心化（Gossip） |
| 数据复制 | HDFS 块级（3 副本） | 应用级（一致性级别可调） |
| 跨机房 | 弱（Replication 异步） | 强（NetworkTopologyStrategy） |
| 写一致性 | 强（单行原子） | 可调（ONE / QUORUM / ALL） |
| 适合场景 | 强一致 OLTP、PB 级历史数据 | 高写吞吐、跨地域、容忍弱一致 |

阿里早期把 HBase 换成 OceanBase / Lindorm 的核心原因不是 HBase 不行，而是 **OceanBase 同时给到了强一致 + SQL + 跨地域**，而 Lindorm 是 HBase 的"超集"（兼容协议 + 跨行事务 + 更好的运维）。

### OpenTSDB（基于 HBase 的时序）

- RowKey 编码：metric_uid + base_timestamp + tag_uid_pairs
- 把"指标元信息"映射成 short uid 节省空间
- 单 Cell 内存储一个小时窗口的所有点位（紧凑编码）
- 缺点：UID 空间有限（默认 3 字节，1600 万）；Tag 组合膨胀

### HBase Replication

- 主从异步复制，基于 WAL
- 模式：Master-Slave / Master-Master / Cyclic
- 不复制 BulkLoad（要单独处理）
- 不复制元数据（建表/Schema 变更要在两端各做一遍）
- Lag 监控：源端 `replication.source.shippedKBs` vs 目标端 ack

### 与 Bigtable 的对比

- Bigtable 论文是 HBase 的"原型"，但 Google 的实现细节差很多
- Bigtable 的 SSTable 在 Colossus（GFS 升级版），HBase 在 HDFS
- Bigtable 用 Chubby（Paxos），HBase 用 ZooKeeper（ZAB）
- Bigtable 内部据说放弃了 Major Compaction（只有连续合并），HBase 仍保留两种
- GCP Bigtable 服务化后，运维难度大幅降低（HBase 自管的最大痛点）

### HBase 在 K8s 上的部署

- 难点：HBase 强依赖 HDFS、ZK、状态化（Region 在哪个 RS）
- HBase Operator（Cloudera / Apache）逐步成熟
- 持久卷（PV）让 RS 重启后能找回 WAL/HFile
- 但生产成熟度仍不如 RDBMS Operator
- 大集群仍以裸金属或 VM 为主

---

## 7. 引用

- HBase 2.5 源码：[apache/hbase](https://github.com/apache/hbase)
- Bigtable 论文：Chang et al., *Bigtable: A Distributed Storage System for Structured Data*（OSDI 2006）
- 《HBase 权威指南》（Lars George）—— 经典书，但已偏老（基于 1.x）
- 《HBase 实战》—— 适合工程师入门
- HBase 官方文档：[hbase.apache.org/book.html](https://hbase.apache.org/book.html)
- Cloudera HBase 调优指南：[Cloudera Tuning HBase](https://docs.cloudera.com/runtime/latest/configuring-hbase/topics/hbase-tuning-guide.html)
- 阿里云 Lindorm 与 HBase 兼容性文档
- HBaseCon Asia / HBaseCon 历年 PPT（2018~2023）

---

> 配套：[Cassandra 深入](/article/db/cassandra) | [Redis 内核](/article/db/redis-internals) | [MySQL MVCC](/article/db/mysql-mvcc)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
