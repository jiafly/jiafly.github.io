# Elasticsearch 深入 —— 倒排索引、分片设计与生产事故

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想速览，看 [搜索/存储综述](/article/db/sharding) 那一节就够了。
> 这里要回答的，是**写 mapping 翻车、半夜被 cluster red 告警叫起来**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. 你 `PUT /logs-2026.05` 用了默认 5 个分片。一年后单分片 80GB，查询 P99 飙到 10s —— 一开始就该按什么公式算分片数？为什么"分片不能改"成了你最大的债？
2. `refresh_interval = 1s`、`flush` 自动触发条件、Lucene `commit`、segment `merge`，这四个动作各自发生在什么时间点？哪一个真正决定了"数据掉电不丢"？
3. 你对一个 `keyword` 字段做 `terms` 聚合，OOM 了。同样的字段在另一个集群上聚合很快 —— 一个开了 `doc_values`，一个开了 `fielddata`，本质差别是什么？
4. 文档里有个 `tags` 数组：`[{"k":"color","v":"red"}, {"k":"size","v":"L"}]`。普通 object 和 nested 在查询时的物理差别是什么？为什么 nested 一个文档可能膨胀成几十个 Lucene 文档？
5. `from=10000&size=10` 在大数据集上必 OOM —— 协调节点要拉多少行？正确的深翻页姿势是什么（`search_after` vs scroll vs PIT）？
6. JVM heap 设 64GB，比 32GB"显然更大"。为什么 ES 官方却建议 **不要超过 32GB（28~30G 实际更稳）**？跟 JVM 的什么机制相关？
7. 单分片 200GB，做 reblance 跑了 12 小时。换成 50GB × 4 分片，2 小时跑完。**主要瓶颈在哪一层**？
8. 集群 3 节点，挂了 1 个 master eligible，剩下 2 个相互不认账，写入全失败。你的 `discovery.seed_hosts` / `cluster.initial_master_nodes` / `minimum_master_nodes`（7.x 之前）配错了哪一项？

如果有半数答不上，往下看。

---

## 1. 底层模型：从倒排索引到分片

绝大多数文章把 ES 讲成"分布式版 Lucene"就完事了。这层抽象解释不了上面 8 个问题。要真看懂，必须下沉到 Lucene Segment 的不可变性、倒排索引的物理结构、以及 ES 在 Lucene 之上加的那层分片协调。

### 1.1 Lucene Segment 与不可变性

ES 的索引（Index）不是单一文件，它是 N 个 **Lucene Segment** 的集合。每个 Segment 自己是一个完整、独立、**不可变（immutable）** 的小型倒排索引。

```
Lucene Index
  ├── segment_0  ── 完整的小倒排索引（Term Dict + Posting + Doc Values + Stored Fields）
  ├── segment_1
  ├── segment_2
  └── ... (持续增加，靠 merge 合并)
```

不可变带来三个直接后果：

- **写不能"原地改"** —— 任何修改都是"标记旧 doc 删除 + 写新 doc"
- **读不需要锁** —— Segment 一旦写完文件就不再变，多读者随意并发
- **缓存友好** —— 操作系统 page cache、Lucene 自身的查询缓存都能稳定命中

> 不可变是 Lucene 的"原罪也是优势"。所有 refresh / flush / merge / delete 的奇葩行为都是为了在不可变之上"假装可变"。

### 1.2 倒排索引（Inverted Index）的物理结构

被问到"倒排索引"绝大多数人只能说出"term → doc id 列表"。真正的物理结构有 3 部分：

```
Term Dictionary    ── 所有 term 排序后的字典（"apple", "banana", "cherry" ...）
   │
   ▼
Term Index (FST)   ── 内存中的前缀树，把 term 范围快速定位到 dict 里的偏移
   │
   ▼
Posting List       ── 每个 term 对应的 doc id 列表（已排序，差值压缩）
                       + freq、positions、offsets（按需）
```

三层的设计动机：

- **Posting List 排序** → 多 term 求交（AND 查询）能用归并算法 O(n+m)
- **Term Dictionary 排序** → 二分查找，O(log N)
- **Term Index 用 FST**（Finite State Transducer） → 内存里只放 FST，磁盘只放 dict

FST 不是普通字典树（Trie），它是 Trie + DAG，**共享前缀也共享后缀**，体积比 Trie 小一个数量级。Lucene 选 FST 而不是 Trie 的关键原因就是内存占用 —— 上亿 term 时 FST 可能只占几十 MB，Trie 要几个 G。

> Posting List 用 FOR + RBM (Roaring Bitmap) 压缩；Term Index 用 FST。**两个数据结构，一个解决 doc id 列表压缩，一个解决 term 查找内存**。

### 1.3 Index → Shard → Lucene Segment 三层

ES 在 Lucene 之上加的分布式层：

```
ES Index "logs-2026.05"  ── 逻辑概念
   │
   ├── Primary Shard 0   ── 一个完整 Lucene 索引（N 个 segment）
   │     ├── Replica 0a  ── primary 的副本（也是完整 Lucene 索引）
   │     └── Replica 0b
   ├── Primary Shard 1
   │     ├── Replica 1a
   │     └── ...
   └── ... (number_of_shards 决定数量)
```

关键约束：

- **`number_of_shards` 创建后不可改**（除非 reindex 或 split/shrink，且 split 只能整数倍）
- **`number_of_replicas` 可以动态改**
- 文档默认按 `hash(routing) % number_of_shards` 路由，所以分片数变了路由就乱

这条"分片数不可改"是后面案例 ⑦ 的全部由来，也是为什么 mapping/分片设计是 ES 项目最重要的一次性决策。

### 1.4 写入路径：translog + buffer + refresh + flush + merge

这五个动作的时序图，是 ES 内核的核心：

```
client INDEX request
       │
       ▼
  ┌────────────────┐
  │ in-memory      │ ① 写入 in-memory buffer（不可查）
  │  index buffer  │
  └────────┬───────┘
           │
           ▼ append
  ┌────────────────┐
  │ translog       │ ② 同时写 translog（默认 fsync，每 request 落盘）
  └────────────────┘
           │
           │  refresh_interval（默认 1s）
           ▼
  ┌────────────────┐
  │ new segment    │ ③ refresh：buffer flush 成新 segment（在 OS cache，可查）
  │  in OS cache   │
  └────────┬───────┘
           │
           │  flush（30 分钟 或 translog ≥ 512MB）
           ▼
  ┌────────────────┐
  │ segment fsync  │ ④ flush：segment 落盘 + Lucene commit + translog 截断
  │ + commit point │
  └────────┬───────┘
           │
           │  merge policy 触发
           ▼
  ┌────────────────┐
  │ merged segment │ ⑤ merge：N 个小 segment 合并为 1 个大 segment
  └────────────────┘
```

| 动作 | 触发条件 | 后果 | 数据安全性 |
| --- | --- | --- | --- |
| **write to buffer** | 每次 INDEX | 不可查、不持久 | 不安全 |
| **translog append** | 每次 INDEX（默认 sync） | 可恢复 | **持久（fsync 后）** |
| **refresh** | 默认 1s / `_refresh` API / index buffer 满 | 新 segment 在 OS cache，**可查** | OS cache 仍可丢 |
| **flush** | translog ≥ 512MB / 30 分钟 / `_flush` | segment fsync + commit point | 持久 |
| **merge** | 后台策略 | 减少 segment 数量 | 不影响数据 |

四个反直觉的点要记住：

1. **refresh ≠ flush**：refresh 让数据"可查"，flush 让数据"持久"
2. **translog 才是持久化的根** —— refresh 不 fsync，但 translog 默认每 request fsync
3. **`refresh_interval=1s` 不是"1 秒后能查"，而是"最多 1 秒后能查"** —— 主动 `_refresh` 可以立即可查
4. **merge 是 IO 大户** —— 单次 merge 可能把几百 GB 数据重写一遍

> 写入的"准实时"特性 = refresh；写入的"不丢" = translog fsync。两个机制独立调，独立理解。

### 1.5 查询路径：Query → Filter → Scoring → Aggregation

```
client GET /logs/_search
       │
       ▼
  ┌────────────────────┐
  │ Coordinating Node  │ ① 拿到 query DSL，解析、改写
  └────────┬───────────┘
           │ 广播到所有相关 shard（按 routing 或全部）
           ▼
  ┌────────────────────┐
  │ Query Phase        │ ② 每个 shard：Lucene 查询 → 取 top N (id+score)
  │  on each shard     │
  └────────┬───────────┘
           │
           ▼
  ┌────────────────────┐
  │ Coordinator merge  │ ③ 协调节点合并所有 shard 的 top N，按 score 排出全局 top K
  └────────┬───────────┘
           │
           ▼
  ┌────────────────────┐
  │ Fetch Phase        │ ④ 用全局 top K 的 doc id，再回到对应 shard 取 _source
  └────────┬───────────┘
           │
           ▼
       client
```

两阶段 Query then Fetch 的关键是：**第一阶段每个 shard 只返回 id+score，不返回完整文档**。否则 1000 个 shard × 1000 行 × 大文档 = 协调节点 OOM。

但这导致一个陷阱：`from=10000&size=10` 时，每个 shard 都要返回 `from + size = 10010` 行 id+score 给协调节点，**协调节点接收 N × (from+size) 条**，再排序丢弃 from 条。这是案例 ⑤ 的根。

> 默认 `index.max_result_window = 10000` 限制 from+size 上限。**深翻页用 search_after / PIT，不要靠 from+size**。

### 1.6 集群协调：Master / Data / Coordinating / Ingest 节点角色

```
Master Node       ── 集群状态（cluster state）的唯一权威，写元信息（mapping, 分片分布）
Master Eligible   ── 候选 master，参与选举（必须配奇数个：3, 5, 7）
Data Node         ── 真正存数据、跑查询
  ├── Data Hot    ── SSD，最新写入
  ├── Data Warm   ── HDD，老一点的只读
  └── Data Cold   ── 最老的，可能用 searchable snapshot
Coordinating Node ── 接客户端请求、做 query/fetch 协调（任何节点默认都可以）
Ingest Node       ── 写入前的 pipeline 处理（grok、enrich 等）
```

7.x 起 ES 用 **Zen2 / Coordinating Subsystem** 取代了老的 Zen 协议。选举用类 Raft 协议，关键参数：

- `cluster.initial_master_nodes`：**只在集群第一次启动时使用**，列出初始的 master eligible 节点名
- `discovery.seed_hosts`：节点之间互发现的地址列表
- `node.roles: [master]` / `[data]` / `[data, ingest]`：角色拆分

7.x 之后 `minimum_master_nodes` 被取消（运行时自动算），**改成 `cluster.initial_master_nodes` 是脑裂防护的核心**。这是案例 ① 的根。

### 1.7 ILM（Index Lifecycle Management）

时序数据（日志、metrics）必须用 ILM，否则单索引会无限增长。典型 4 阶段：

```
Hot   ── 当前写入索引，SSD，replica 1，refresh 1s
  │
  │  rollover（按大小 50GB / 时间 1d / 文档数）
  ▼
Warm  ── 不再写入，只读，可能 force merge 到 1 segment，replica 1
  │
  │  迁移到 warm 节点
  ▼
Cold  ── HDD 或 searchable snapshot from S3，replica 0
  │
  │  保留期到
  ▼
Delete
```

ILM 用 `_alias` + rollover 实现"应用层只看到一个 alias，底层索引按时间切片"。

> 时序场景必上 ILM。**单索引不要超过 1 年的数据**，这是后面"分片不能改"问题的根本规避手段。

---

## 2. 源码切片：写入、查询、聚合的关键路径

> 基于 Elasticsearch 8.x 源码（`org.elasticsearch.*`）、Lucene 9.x 源码（`org.apache.lucene.*`）。注释为本文添加。

### 2.1 IndexShard.applyIndexOperationOnPrimary —— 主分片写入入口

```java
// IndexShard.java 简化
public Engine.IndexResult applyIndexOperationOnPrimary(
        long version, VersionType versionType, SourceToParse sourceToParse,
        long ifSeqNo, long ifPrimaryTerm, long autoGeneratedTimestamp,
        boolean isRetry) throws IOException {

    assert versionType.validateVersionForWrites(version);
    return applyIndexOperation(
        getEngine(),                                  // ★ Engine 是 ES 对 Lucene 的封装
        UNASSIGNED_SEQ_NO, getOperationPrimaryTerm(),
        version, versionType, ifSeqNo, ifPrimaryTerm,
        autoGeneratedTimestamp, isRetry,
        Engine.Operation.Origin.PRIMARY,
        sourceToParse);
}
```

**逐段点透**：

- `applyIndexOperationOnPrimary` 是主分片写入的入口。replica 走的是 `applyIndexOperationOnReplica`，区别在 `Origin.PRIMARY` vs `Origin.REPLICA`。
- 每个写操作有 **`SeqNo` + `PrimaryTerm`** 两个版本号。SeqNo 单调递增，PrimaryTerm 在每次主分片切换时 +1。这两者是 ES 实现"基于 seqno 的恢复"和"`if_seq_no` 乐观锁"的根。
- `Engine` 是 ES 对 Lucene 的封装层，包含 `IndexWriter` + `Translog` + `RefreshScheduler` 等。

### 2.2 InternalEngine.index —— 写入主流程

```java
// InternalEngine.java 简化
public IndexResult index(Index index) throws IOException {
    try (ReleasableLock releasableLock = readLock.acquire()) {
        ensureOpen();
        // ① 版本检查（乐观并发控制）
        final IndexingStrategy plan = indexingStrategyForOperation(index);
        if (plan.earlyResultOnPreFlightError.isPresent()) {
            return plan.earlyResultOnPreFlightError.get();
        }

        // ② 写 Lucene
        if (plan.indexIntoLucene) {
            indexResult = indexIntoLucene(index, plan);
        }

        // ③ 写 translog（同步！）
        if (indexResult.getResultType() == ResultType.SUCCESS) {
            final Translog.Location location = translog.add(
                new Translog.Index(index, indexResult));
            indexResult.setTranslogLocation(location);
        }
        return indexResult;
    }
}
```

**逐段点透**：

- 顺序是 **先写 Lucene 内存、再写 translog**。看似与"WAL 必须先于数据"的规矩相反，但 ES 这里 Lucene 写的是内存 buffer，**不可见、不持久**，相当于 WAL 之前的"准备阶段"。translog 写完才算真正完成。
- translog 默认 `index.translog.durability = request`，每个 request 后 fsync 一次。可调成 `async`（每 5s fsync），写入吞吐能涨 3~5 倍但**最近 5s 数据可丢**。
- 不要把 translog 想成 MySQL binlog 那样"另一份 SQL"。它是 **Lucene 操作的最小复现日志**，恢复时直接 replay。

### 2.3 IndexShard.refresh —— 开新 segment

```java
// IndexShard.java 简化
public void refresh(String source) {
    verifyNotClosed();
    if (logger.isTraceEnabled()) logger.trace("refresh with source [{}]", source);
    long ifBeforeRefresh = lastSearcherAccess.get();

    getEngine().refresh(source);         // ★ 委托给 Engine.refresh
    afterRefresh(true);
}

// InternalEngine.refresh 关键代码
final void refresh(String source, SearcherScope scope, boolean block) {
    final long startTime = relativeTimeInNanosSupplier.getAsLong();
    boolean refreshed;
    try (ReleasableLock lock = readLock.acquire()) {
        ensureOpen();
        // ★ 让 IndexWriter 把内存 buffer flush 成新 segment（在 OS cache）
        if (block) {
            referenceManager.maybeRefreshBlocking();
            refreshed = true;
        } else {
            refreshed = referenceManager.maybeRefresh();
        }
    }
    if (refreshed) {
        lastRefreshedCheckpointListener.updatedCheckPoint(getProcessedLocalCheckpoint());
    }
}
```

**逐段点透**：

- **refresh 不调用 fsync** —— 它只让 Lucene `IndexWriter` 把当前 buffer 的内容写成一个新 segment 文件，但只在 OS page cache 里。掉电会丢，但能查。
- `maybeRefresh()` 不一定真 refresh —— Lucene 内部判断"距离上次 refresh 是否有新数据"，没数据就 no-op。这是 `_refresh` API 反复调也不会爆 IO 的原因。
- `referenceManager` 是 Lucene 的 `ReferenceManager<IndexSearcher>` —— refresh 的本质是"换一个新的 IndexSearcher 引用，老的等所有查询用完后释放"。读不阻塞。

### 2.4 segment merge 的策略

```java
// org.apache.lucene.index.TieredMergePolicy 简化
@Override
public MergeSpecification findMerges(MergeTrigger mergeTrigger,
                                     SegmentInfos infos,
                                     MergeContext mergeContext) {
    // ① 收集所有 segment + size
    final Set<SegmentCommitInfo> merging = mergeContext.getMergingSegments();
    final List<SegmentSizeAndDocs> infosSorted = new ArrayList<>();
    for (SegmentCommitInfo info : infos) {
        long sizeInBytes = size(info, mergeContext);
        infosSorted.add(new SegmentSizeAndDocs(info, sizeInBytes, ...));
    }
    infosSorted.sort(Comparator.comparingLong(s -> s.sizeInBytes));

    // ② 按 tier 分层（segmentsPerTier 默认 10）
    // 每层选 maxMergeAtOnce（默认 10）个 segment 合并
    while (...) {
        // 在剩余 segment 里找一组"代价最小、收益最大"的合并候选
        MergeScore mergeScore = score(candidate, ...);
        // 代价 = 合并后 segment 大小；收益 = 减少的 segment 数
    }
}
```

**逐段点透**：

- Lucene 默认 `TieredMergePolicy`：分层合并，类似 LSM-Tree 的 leveled compaction。每层 segment 数超过 `segments_per_tier` 就触发合并。
- **大 segment 不会无限制合并** —— `index.merge.policy.max_merged_segment` 默认 5GB。超过这个大小后不再参与自然 merge，除非主动 `_forcemerge`。
- **`_forcemerge max_num_segments=1`** 把所有 segment 合到 1 个 —— 只读索引（warm/cold 阶段）这么做能大幅降低查询的"打开 segment 数量"开销。**但活跃写入索引**绝对不要 forcemerge —— 一个超大 segment 永远不再 merge，但里面的 deleted doc 也永远不会真删。

### 2.5 query 两阶段（query then fetch）

```java
// org.elasticsearch.action.search.SearchQueryThenFetchAsyncAction 简化
@Override
protected void executePhaseOnShard(SearchShardIterator shardIt,
                                    SearchShardTarget shard,
                                    SearchActionListener<SearchPhaseResult> listener) {
    // Phase 1：在每个 shard 上跑 query
    getSearchTransport().sendExecuteQuery(
        getConnection(shard.getClusterAlias(), shard.getNodeId()),
        buildShardSearchRequest(shardIt, listener.requestIndex),
        getTask(),
        listener);
}

// 第一阶段全部完成后
@Override
protected SearchPhase getNextPhase(SearchPhaseResults<SearchPhaseResult> results,
                                    AggregatedDfs dfs) {
    // Phase 2：用 phase1 的 docId 去拉 _source
    return new FetchSearchPhase(results, ..., this);
}
```

**逐段点透**：

- 协调节点把请求广播到 N 个 shard，**每个 shard 各跑一次本地的 Lucene 查询**，返回 `(docId, score)` 列表（top from+size）。
- 协调节点收齐后做 `PriorityQueue<ShardDoc>` 全局排序，挑出全局 top K 的 docId，然后第二阶段去对应 shard fetch `_source`。
- `from=10000` 时每个 shard 返回 10010 条 (id+score)，协调节点要排序合并 N × 10010 条 —— 这就是为什么深翻页爆 OOM。
- `_search?preference=_local` 可以让协调节点优先选本地 shard，**对 Kibana 仪表盘场景能减一跳网络**。

### 2.6 aggregation 的 BucketCollector

```java
// org.elasticsearch.search.aggregations.bucket.terms.GlobalOrdinalsStringTermsAggregator 简化
@Override
public LeafBucketCollector getLeafCollector(LeafReaderContext ctx,
                                             LeafBucketCollector sub) throws IOException {
    final SortedSetDocValues globalOrds = valuesSource.globalOrdinalsValues(ctx);
    return new LeafBucketCollectorBase(sub, globalOrds) {
        @Override
        public void collect(int doc, long owningBucketOrd) throws IOException {
            if (globalOrds.advanceExact(doc)) {
                for (long globalOrd = globalOrds.nextOrd();
                     globalOrd != SortedSetDocValues.NO_MORE_ORDS;
                     globalOrd = globalOrds.nextOrd()) {
                    long bucketOrd = bucketOrds.add(owningBucketOrd, globalOrd);
                    if (bucketOrd < 0) {
                        bucketOrd = -1 - bucketOrd;
                        collectExistingBucket(sub, doc, bucketOrd);
                    } else {
                        collectBucket(sub, doc, bucketOrd);
                    }
                }
            }
        }
    };
}
```

**逐段点透**：

- ES 聚合的核心是 **`BucketCollector`**：遍历每个 doc，把它"分桶"到对应的 ord 上，再在桶里增量更新 metric。
- 用 **global ordinal**（跨 segment 的全局序号）而不是字符串，能把"按字符串分组"变成"按整数索引数组"，性能差一个数量级。global ord 的构建在 segment merge 后或第一次聚合时做。
- 没有 `doc_values` 的字段（比如 text 字段）走的是 fielddata，**fielddata 在 heap 里**，撑爆 JVM 是案例 ④ 的根。

### 2.7 集群状态变更与 master 选举

```java
// org.elasticsearch.cluster.coordination.Coordinator 简化
public void onJoinValidator(DiscoveryNode node, ClusterState state) {
    // 校验加入的节点版本兼容
    JoinTaskExecutor.ensureNodesCompatibility(node.getVersion(), nodes);
    JoinTaskExecutor.ensureMajorVersionBarrier(node.getVersion(), ...);
}

// 选举核心：StartJoinRequest → 收 quorum 票 → 成 master
private void becomeCandidate(String method) {
    assert Thread.holdsLock(mutex);
    if (mode != Mode.CANDIDATE) {
        mode = Mode.CANDIDATE;
        // ★ 启动 PreVote 阶段（避免无意义的 term 递增）
        peerFinder.activate(coordinationState.get().getLastAcceptedState().nodes());
        clusterFormationFailureHelper.start();
    }
}
```

**逐段点透**：

- ES 7+ 的协议借鉴了 Raft，但不是严格 Raft。核心概念：term（任期）、PreVote（预投票，避免脑裂时 term 无意义递增）、StartJoin、PublishRequest。
- 集群状态（`ClusterState`）只能由当前 master 修改，包括 mapping 更新、分片分配、节点加入等。**所有数据节点收到的 cluster state 必须一致**。
- 选举要求 quorum = `floor(N/2) + 1`。3 个 master eligible 节点容忍 1 挂；5 个容忍 2 挂；**2 个不能挂任何一个**（脑裂或停摆只能二选一）。
- 这就是为什么生产 master eligible 节点数必须是 **3 或 5**（小集群 3，大集群 5），永远不要是偶数。

---

## 3. 实测数据：分片、refresh、fielddata 的代价

口诀化的"3 个数据节点 + 5 个分片 + 1 个副本"是默认值，不是答案。生产里这些数字怎么定，必须用实测说话。

### 3.1 不同分片数下的写入吞吐

**场景**：3 节点集群（16C/64G/NVMe），单索引，1KB 文档，bulk 5000 一批。

| 分片数 | bulk 吞吐 (docs/s) | CPU 利用率 | 单分片大小（1 小时后） |
| --- | --- | --- | --- |
| 1 | 18,000 | 单节点 75%，其余 5% | 65 GB |
| 3 | 52,000 | 三节点均 70% | 22 GB |
| 6 | 75,000 | 三节点均 88% | 11 GB |
| 12 | 78,000 | 三节点均 92% | 5.5 GB |
| 24 | 70,000 | 三节点均 95% | 2.7 GB |
| 48 | 55,000 | 三节点均 98%（merge 抢 CPU） | 1.4 GB |

可以看到：**分片数 ≈ 节点数 × 2~4 时吞吐最优**。再加分片，merge 开销和元数据开销超过并行收益，反而下降。

### 3.2 分片数怎么定（量化公式）

业界共识公式：

```
单分片大小目标：10GB ~ 50GB（日志/全文搜索）；2GB ~ 10GB（聚合分析）
分片数 = ceil(预计总数据量 / 单分片目标大小)

举例：
  预计 1 年数据 1.2TB，按日 rollover → 单日 ~3.3GB
  → 单日索引 1 个分片足够
  → 1 年共 365 个索引 × 1 = 365 个 primary shard
  → 每节点理想分片数 < 600（heap 30G 时上限）
```

约束：
- 单分片 > 50GB：reblance 慢、查询单 shard 慢、merge 大块阻塞
- 单分片 < 1GB：分片数过多，cluster state 膨胀、协调节点开销大
- **整集群 primary shard 数 ≤ 节点数 × 600**（每 GB heap 约 20 个分片是经验值）

### 3.3 refresh_interval 对查询新鲜度与吞吐的影响

```bash
# 实测：bulk 写入 1 亿条 1KB 文档，5 节点
PUT /logs/_settings
{ "index.refresh_interval": "1s" }   # 默认
```

| refresh_interval | bulk 吞吐 (docs/s) | "可查"延迟 |
| --- | --- | --- |
| 1s（默认） | 75,000 | ≤ 1s |
| 5s | 105,000 | ≤ 5s |
| 30s | 130,000 | ≤ 30s |
| -1（关闭） | 145,000 | 直到下次 refresh |

写入大批量历史数据时常见手法：

```bash
# 写入前关 refresh
PUT /logs/_settings { "index.refresh_interval": "-1", "index.number_of_replicas": 0 }

# bulk 写入...

# 写完恢复
PUT /logs/_settings { "index.refresh_interval": "1s", "index.number_of_replicas": 1 }
POST /logs/_forcemerge?max_num_segments=1
```

> **口诀**：写多读少 → refresh 调大；写完一次性归档 → refresh 关 + replica 关 + forcemerge。这套组合能让导入快 5~10 倍。

### 3.4 fielddata vs doc_values 的内存占用

对一个 1000 万行、`status` keyword 字段（基数 100）做 `terms` 聚合：

| 配置 | heap 占用 | 聚合耗时 | OOM 风险 |
| --- | --- | --- | --- |
| keyword + doc_values（默认） | 几乎 0（在堆外/磁盘） | 80ms | 低 |
| text + fielddata=true | **600MB+**（按字段值数量爆炸） | 200ms | **高** |

`doc_values` 是列存（columnar），存在磁盘上，靠 OS page cache 加速 —— **不占 JVM heap**。
`fielddata` 是把倒排索引"翻"过来（uninverted），构建在 heap 里 —— **占 JVM heap**。

mapping 设计的根本原则：

```json
{
  "properties": {
    "status": { "type": "keyword" },         // 默认有 doc_values
    "message": {
      "type": "text",                         // 用于全文搜索
      "fields": {
        "raw": { "type": "keyword" }          // ★ multi-field：聚合走 raw
      }
    }
  }
}
```

绝对不要在 text 字段上开 `fielddata: true`。要聚合就用 multi-field 的 keyword 子字段。

### 3.5 nested vs join 的代价

测试集 100 万订单，每订单平均 5 个 item：

| 建模方式 | 索引大小 | 写入吞吐 | "买了 X 商品的订单" 查询 P99 |
| --- | --- | --- | --- |
| Flat（订单和 item 各自一个 doc） | 1.2 GB | 65k/s | 不能精确表达"同一订单" |
| Object（item 数组直接放进订单） | 1.5 GB | 60k/s | **错查**（数组拍扁，跨字段错配） |
| Nested（item 数组用 nested 类型） | 2.8 GB | 45k/s | 准确，120ms |
| Parent/Child join | 1.4 GB | 30k/s | 准确，350ms |

**结论**：

- **Object 数组的陷阱**：`{"items": [{"name":"A","price":10},{"name":"B","price":20}]}` 在 ES 里被拍扁成 `items.name=[A,B]; items.price=[10,20]`。查"name=A AND price=20"会错命中 —— 因为"A 和 20"在不同 item 上。
- **Nested**：每个子对象单独索引为一个 Lucene 文档（隐藏文档），查询时通过 `nested` query 关联。代价：写入索引膨胀 N 倍（N = 平均嵌套元素数），查询要做 join。
- **Parent/Child**：分开存，靠 routing 保证父子在同分片，查询时 `has_child` / `has_parent`。比 nested 灵活（子文档可独立更新），但查询慢、内存占用更高。

> 选型口诀：单层嵌套且子文档不需独立更新 → **nested**；子文档独立生命周期 → **join**；查询不需要"同一组内"约束 → **flat**。

### 3.6 hot-warm-cold 各层的查询延迟

ILM 4 层架构下的对比：

| 层 | 介质 | replica | refresh | 单查询 P99 | 适用 |
| --- | --- | --- | --- | --- | --- |
| Hot | NVMe SSD | 1 | 1s | 30ms | 当天写入 |
| Warm | SATA SSD | 1 | -1（不再写） | 80ms | 7~30 天 |
| Cold | HDD | 0 + searchable snapshot | -1 | 500ms | 30~365 天 |
| Frozen | 全部在 S3 | 0 | -1 | 5~30s | > 1 年 |

冷热分层最大节省的不是磁盘成本，是 **JVM heap** —— 冷数据不在 hot 节点上，hot 节点的 cluster state、fielddata cache 都更小，整体集群更稳。

> **口诀**：日志 / metrics 必须 ILM；不分层的"全 hot 集群"撑不到 1 年。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 集群分裂（split brain） —— "两边都自称 master"

#### 现象

3 节点集群跨两个机房（A 机房 2 节点，B 机房 1 节点）。机房间网络抖动 30 秒，**两边各自选出了 master**。网络恢复后，集群进入 `RED` 状态，部分索引写入失败，DBA 发现 cluster state 在两边出现冲突。

#### 根因

老版本 ES（5.x、6.x）的脑裂源头是 `discovery.zen.minimum_master_nodes` 配错：

```yaml
# 错误：3 节点集群配成 1
discovery.zen.minimum_master_nodes: 1
```

含义：选 master 至少需要 1 票。结果两边各 1 票就能选出来 → 脑裂。

7.x 之后 minimum_master_nodes 已废弃，改成 `cluster.initial_master_nodes`（仅启动用）+ Zen2 的 quorum 自动计算。但还是有反面教材：

```yaml
# 错误：4 节点（偶数）master eligible
node.roles: [ master, data ]   # 4 个节点都这么配
```

4 个 master eligible，quorum = 3。挂 1 个剩 3，能工作；挂 2 个剩 2，停摆但不脑裂。看起来"安全"，但 **4 节点比 3 节点更脆弱**（容忍故障数一样，资源更多还更贵）。

#### 修复

7.x 标准配置：

```yaml
# 3 节点全部 master eligible（小集群）
node.name: es-1
node.roles: [ master, data, ingest ]
network.host: 0.0.0.0

discovery.seed_hosts: ["es-1:9300", "es-2:9300", "es-3:9300"]
cluster.initial_master_nodes: ["es-1", "es-2", "es-3"]

# 大集群拆角色
# 3 个专用 master（小机器）+ N 个 data 节点
```

关键认知：

- master eligible **永远配奇数**（3 或 5）
- 大集群把 master 节点和 data 节点分开 —— master 不参与查询，避免 OOM 影响选举
- `cluster.initial_master_nodes` 仅在集群第一次启动时使用，**集群形成后必须从配置文件移除**（否则节点重装后可能误判初始状态）

跨机房部署：

```yaml
# 节点感知（rack awareness）
cluster.routing.allocation.awareness.attributes: zone
node.attr.zone: zone-a   # 或 zone-b、zone-c

# 强制副本跨 zone
cluster.routing.allocation.awareness.force.zone.values: zone-a,zone-b,zone-c
```

但**网络分区 30 秒以上的跨机房**，建议用两套独立集群 + CCR (Cross-Cluster Replication)，不要赌网络。

#### 验证

```bash
# 手工模拟：iptables 隔离一个 master eligible 节点
sudo iptables -A INPUT -s <other-node-ip> -j DROP

# 观察被隔离的节点
GET /_cat/master?v
GET /_cluster/health?pretty

# 应该：被隔离的节点变成 NO_MASTER 状态、拒写、不脑裂
```

> **口诀**：master eligible **奇数 + quorum** 是脑裂唯一防线。**永远不要用 2 节点 master eligible**。

---

### 案例 ② 主节点选举失败 —— "整集群启动后 RED 半小时"

#### 现象

机房断电恢复后，5 节点集群启动。`GET /_cluster/health` 一直 `RED`，`/_cat/master` 返回空。日志里循环刷：

```
master not discovered yet, this node has not previously joined a bootstrapped cluster,
and [cluster.initial_master_nodes] is empty on this node
```

#### 根因

新搭集群时漏配 `cluster.initial_master_nodes`：

```yaml
# 错误：没配 initial_master_nodes，节点不知道"初始 master 集合"
discovery.seed_hosts: ["es-1", "es-2", "es-3", "es-4", "es-5"]
# cluster.initial_master_nodes: 缺失
```

Zen2 协议要求"集群第一次启动时"必须显式声明初始 master 集合，否则节点不敢投票（怕脑裂）。

另一种更隐蔽的错：节点曾经加入过别的集群（数据目录残留 `_state/manifest-N.st`），新集群启动时拒绝加入：

```
this node previously joined cluster with UUID [xxx]
but is now being restarted as part of cluster [yyy]
```

#### 修复

新搭集群（首次启动）：

```yaml
# 全部 5 个节点都配同一份
cluster.initial_master_nodes: ["es-1", "es-2", "es-3"]   # 不必列全部 master eligible
discovery.seed_hosts: ["es-1:9300", "es-2:9300", "es-3:9300", "es-4:9300", "es-5:9300"]
```

集群形成后，**所有节点 yml 里把 `cluster.initial_master_nodes` 这一行删掉**或注释掉（避免脑裂场景下被误用作"重新 bootstrap"）。

如果节点数据目录是其他集群的残留：

```bash
# 警告：这会丢失该节点上所有数据（必须确认你只是要重装这个节点）
rm -rf /var/lib/elasticsearch/nodes/0/_state/

# 或者用工具清理
bin/elasticsearch-node detach-cluster
```

#### 验证

```bash
# 集群启动后
GET /_cluster/health
{
  "status": "green",
  "number_of_nodes": 5,
  "number_of_data_nodes": 5,
  ...
}

GET /_cat/master?v
id        host    ip      node
xyz       1.2.3.4 1.2.3.4 es-1
```

> **口诀**：新集群必须配 `cluster.initial_master_nodes`（一次性）；集群成立后立即删除该配置；**节点搬家前先 `detach-cluster`**。

---

### 案例 ③ refresh 太频繁导致 IO 100% —— "上线大流量后查询全慢"

#### 现象

新业务上线，写入 QPS 从 5k 涨到 50k，查询 P99 从 80ms 飙到 3s。`top` 看 `iowait` 持续 80%+，`iostat` 看磁盘 100% 利用率。集群没 OOM，CPU 也不高。

#### 根因

每次 refresh 会产出一个新 segment（小文件）。默认 `refresh_interval=1s`：

```
50k QPS → 每秒 1 个 ~1MB 的小 segment
1 分钟后：60 个小 segment
10 分钟后：600 个小 segment
→ Lucene 自动 merge 触发，把 N 个小 segment 合并成大的
→ merge 是 IO 大户：要把所有源 segment 数据读出来，写一份新的
```

`hot_threads` API 看到大量线程在做 `merge`：

```bash
GET /_nodes/hot_threads
# 输出里大量：
#  98.5% [elasticsearch[node-1][[xxx]]: merging segments]
```

merge 抢光了磁盘 IO，正常查询排队。

#### 修复

**方案 A**：拉长 refresh_interval

```bash
PUT /logs-*/_settings
{ "index.refresh_interval": "30s" }
```

代价：写后最多 30s 才能查到。日志/审计场景完全可接受。

**方案 B**：调整 merge 限速

```yaml
# elasticsearch.yml
indices.recovery.max_bytes_per_sec: 100mb         # recovery 限速
indices.store.throttle.max_bytes_per_sec: 100mb   # store 限速
```

**方案 C**：调整 merge 并发

```bash
PUT /logs-*/_settings
{
  "index.merge.scheduler.max_thread_count": 1,    # SSD: 1 (默认 cores/2)
  "index.merge.policy.max_merged_segment": "5gb"
}
```

**方案 D**（根本）：Bulk 写入 + 异步 refresh

```bash
# 客户端：bulk batch 5000~10000 条/批
POST /_bulk?refresh=false
{ "index": { "_index": "logs" } }
{ ... }

# 单独触发 refresh
POST /logs/_refresh
```

不要每条 INDEX 都带 `?refresh=true` —— 这会强制 refresh，1 万 QPS 下能干掉集群。

#### 验证

```bash
# 调整后观察
GET /_cat/indices/logs-*?v&h=index,segments.count,segments.memory

# segments.count 应稳定在几十级别，不应持续涨到几百
# iowait 应回落到 < 30%
GET /_nodes/stats/indices/refresh,merges
```

> **口诀**：refresh 频繁 → 小 segment 多 → merge 风暴 → IO 100%。**写多读少 refresh 调大；bulk 不要带 refresh=true**。

---

### 案例 ④ fielddata 撑爆 JVM heap —— "聚合一跑就 OOM"

#### 现象

业务对 `message` 字段（type=text）跑 `terms` 聚合，第一次跑 30 秒返回，第二次跑直接 OOM，节点挂掉重启。

#### 根因

text 字段默认**不能聚合**。你强行启用了 `fielddata: true`：

```json
PUT /logs/_mapping
{
  "properties": {
    "message": {
      "type": "text",
      "fielddata": true        // ★ 危险
    }
  }
}
```

含义：让 Lucene 在 heap 里把倒排索引"翻"过来，构建一个 `term → docId 列表` 的反向结构（uninverted）。**字段基数（unique term 数）越高，heap 占用越大**。`message` 字段每条日志都不同，可能产生几百万 unique term，几个 GB 的 heap 占用。

`circuit breaker` 在你超阈值时会拒绝（默认 fielddata circuit breaker 40% heap），但**已经构建好的 fielddata 不会立即释放**，下次查询继续撑大。

#### 修复

**方案 A**（推荐）：永远不要在 text 字段上开 fielddata。改用 multi-field：

```json
PUT /logs
{
  "mappings": {
    "properties": {
      "message": {
        "type": "text",                  // 全文搜索
        "fields": {
          "keyword": {                    // ★ multi-field
            "type": "keyword",
            "ignore_above": 256          // 超过 256 字符不索引为 keyword
          }
        }
      }
    }
  }
}

# 聚合走 message.keyword
GET /logs/_search
{
  "aggs": {
    "top_msg": { "terms": { "field": "message.keyword", "size": 10 } }
  }
}
```

`ignore_above` 关键：日志消息可能很长，没必要为每条 unique 的整段消息建 keyword（基数爆炸）。256 是经验值。

**方案 B**：清掉已经撑大的 fielddata cache

```bash
POST /logs/_cache/clear?fielddata=true

# 或调小 circuit breaker 阈值
PUT /_cluster/settings
{ "persistent": { "indices.breaker.fielddata.limit": "20%" } }
```

**方案 C**：禁掉 wildcard / regex 这种触发 fielddata 的查询，业务侧改为前缀树查询。

#### 验证

```bash
# 观察 fielddata 占用
GET /_cat/fielddata?v&h=node,host,total

# 应该接近 0；如果有非 0 项，对应字段需要改 mapping
```

> **口诀**：**text 字段永远不开 fielddata**；要聚合用 multi-field 的 keyword 子字段；keyword 字段长度过长用 `ignore_above` 截断。

---

### 案例 ⑤ 深分页 from+size 的 OOM —— "运营导出报表把集群搞挂"

#### 现象

运营做"导出最近 1 个月所有订单"的报表，分页拉取 from=100000、size=100。**协调节点 OOM 重启**，集群短时间不可用。

#### 根因

`from + size` 的内部实现：

```
client: GET /_search?from=100000&size=100
        │
        ▼
coordinator broadcast 到 N 个 shard
        │
        ▼
each shard:
   Lucene 查询 → top (from + size) = 100100 条 (id, score)
   返回 (id, score) × 100100
        │
        ▼
coordinator 收到 N × 100100 条 (id, score)
        │
        ▼
全局优先队列排序，丢弃前 100000 条，返回最后 100 条
```

10 shard × 100100 条 (id+8字节 score) ≈ 30MB **（仅 id+score）**。如果 from=1000000，单次查询 300MB+，并发几个就 OOM。

ES 默认 `index.max_result_window=10000` 防御这种用法，但运营可能改了：

```bash
PUT /orders/_settings
{ "index.max_result_window": 1000000 }   # ★ 自杀式配置
```

#### 修复

**方案 A**（推荐）：用 `search_after`

```bash
# 第一次
GET /orders/_search
{
  "size": 100,
  "sort": [{"created_at": "asc"}, {"_id": "asc"}]
}
# 返回最后一条的 sort 值，比如 [1700000000000, "order-xxx"]

# 第二次起
GET /orders/_search
{
  "size": 100,
  "sort": [{"created_at": "asc"}, {"_id": "asc"}],
  "search_after": [1700000000000, "order-xxx"]
}
```

每次只拉 size 条，**协调节点压力 O(size) 而非 O(from+size)**。但要求排序字段唯一（用 `_id` 兜底）。

**方案 B**：用 PIT (Point In Time) + search_after（推荐于稳定快照）

```bash
# 创建 PIT
POST /orders/_pit?keep_alive=1m
# 返回 pit_id

GET /_search
{
  "size": 100,
  "pit": { "id": "...", "keep_alive": "1m" },
  "sort": [...],
  "search_after": [...]
}

# 用完关闭
DELETE /_pit
{ "id": "..." }
```

PIT 给整个迭代过程一个一致的快照视图，避免迭代中数据变更导致重复/缺失。

**方案 C**（旧）：scroll API。已不推荐 —— scroll 持有 segment 引用阻止 merge，PIT + search_after 替代。

**方案 D**：业务层限制

```yaml
# 改回默认
PUT /orders/_settings
{ "index.max_result_window": 10000 }

# 应用层 UI 不允许翻到 10000 以后
# 真要导出全量 → 走 search_after 后台任务
```

#### 验证

```bash
# 模拟深翻页
ab -n 10 -c 1 'http://es:9200/orders/_search?from=50000&size=100'

# 观察 coordinator heap
GET /_nodes/stats/jvm

# 改用 search_after 后 heap 应平稳
```

> **口诀**：**from+size 上限 10000**，超过必死；导出/迭代用 **search_after + PIT**；scroll 已过时不要再用。

---

### 案例 ⑥ JVM heap 配置错误 —— "32G 改 64G 后 GC 反而更长"

#### 现象

DBA 觉得 ES 节点 32G heap 不够用，加机器内存到 128G，把 heap 调到 64G。结果**Full GC 时间从 100ms 涨到 1.2 秒**，节点频繁被踢出集群。

#### 根因

JVM 的对象指针压缩（Compressed OOPs）。HotSpot JVM 在 heap < 32GB 时，对象引用用 **32 位压缩指针**（实际寻址 35 位 = 32G），节省内存、CPU 缓存命中率更高。

heap > 32G 后必须用 **64 位非压缩指针**：
- 每个对象引用从 4 字节涨到 8 字节
- 32G 用户 heap 实际可能"等价于" 28G 压缩指针的容量
- GC 扫描的对象大小翻倍 → STW 时间显著增加

ES 官方建议：

```
xms = xmx ≤ 30G（保守 28G，给 OS 和 JVM 自己留点）
节点物理内存：heap 占一半，剩下给 OS page cache（Lucene 重度依赖 page cache）
```

**机器内存 64G 时**：heap 30G + OS cache 30G 是最优解，比 heap 60G + OS cache 4G 强一个量级。

#### 修复

```bash
# /etc/elasticsearch/jvm.options
-Xms30g
-Xmx30g

# 验证压缩指针生效
java -Xms30g -Xmx30g -XX:+PrintFlagsFinal -version | grep -i UseCompressedOops
# 应该是 true
```

如果单节点确实需要 100G+ 数据，**拆成多个 ES 进程（多实例）**，每个 30G heap，而不是单进程 64G heap：

```yaml
# es-node-1
path.data: /data/es-1
http.port: 9200
transport.port: 9300
# es-node-2
path.data: /data/es-2
http.port: 9201
transport.port: 9301
```

注意配 `cluster.routing.allocation.same_shard.host: true` 防止 primary 和 replica 落到同一物理机。

#### 验证

```bash
# 观察 GC 日志
GET /_nodes/stats/jvm

# young gc < 100ms，full gc < 500ms 是健康线
# 如果 full gc 频繁（每分钟多次）或 STW > 1s，节点会被踢
```

> **口诀**：**ES JVM heap ≤ 30G（不是 32G）**，给压缩指针留余地。物理内存的另一半给 OS page cache，比再给 heap 划算。

---

### 案例 ⑦ 单分片过大导致 reblance 慢 —— "缩容一台机跑了 36 小时"

#### 现象

下线一台 ES 节点，触发分片 reblance。原计划 6 小时迁完 5TB 数据，**实际跑了 36 小时**，集群一直 YELLOW，业务受影响。

#### 根因

排查发现：单分片大小 200GB+。reblance 时分片只能整体迁移，**单分片越大，单次复制越慢、网络带宽吃满后还卡在小文件 IO**。

更糟的是：

- 默认 `cluster.routing.allocation.node_concurrent_recoveries: 2`：每节点同时只 2 个分片在恢复
- `indices.recovery.max_bytes_per_sec: 40mb`：复制限速
- 单个 200GB 分片复制 → 200GB / 40MB/s ≈ 5000s ≈ 1.4 小时

5TB / 200GB = 25 个分片，2 并发 → 25 / 2 × 1.4h ≈ 17.5 小时（理想），实际更糟因为 merge 抢 IO。

如果当初分片设计成 50GB × 4，分片数 100，单分片复制 1300s ≈ 20 分钟，并行度更高，3~4 小时跑完。

#### 修复

**临时**：调高并发和带宽

```bash
PUT /_cluster/settings
{
  "transient": {
    "cluster.routing.allocation.node_concurrent_recoveries": 4,
    "indices.recovery.max_bytes_per_sec": "200mb"
  }
}
```

**警告**：调高带宽会抢业务 IO，必须监控延迟。

**长效（核心）**：通过 reindex 或 split 调小分片

```bash
# split：原分片必须只读，且新分片数是原数的整数倍
PUT /old-index/_settings
{ "index.blocks.write": true }

POST /old-index/_split/new-index
{
  "settings": {
    "index.number_of_shards": 4         # 原 1 个分片 split 成 4 个
  }
}
```

或对超大索引做 reindex：

```bash
PUT /new-logs-2026
{
  "settings": { "index.number_of_shards": 12 }
}

POST /_reindex
{
  "source": { "index": "old-logs-2026" },
  "dest":   { "index": "new-logs-2026" }
}
```

时序数据 → 改用 **rollover + ILM**，每个新索引控制大小：

```bash
PUT /_ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_primary_shard_size": "50gb",   # ★ 单分片到 50G 自动滚动
            "max_age": "1d"
          }
        }
      }
    }
  }
}
```

#### 验证

```bash
# 观察 recovery 进度
GET /_cat/recovery?active_only=true&v&h=index,shard,type,stage,bytes_percent,files_percent

# 单分片大小检查
GET /_cat/shards?v&h=index,shard,prirep,state,store
# pri 列大小应在 10G ~ 50G 之间
```

> **口诀**：单分片 ≤ 50GB 是 reblance 速度的根本约束。**时序数据用 rollover 自动控制单分片大小**，业务数据需要前期规划好分片数。

---

### 案例 ⑧ mapping 爆炸 —— "动态字段把 cluster state 撑到 GB 级"

#### 现象

业务接入埋点日志后，集群越来越慢。`GET /_cluster/health` 都要 5 秒返回。`GET /_cluster/state` 返回的 JSON 文件 800MB+。

#### 根因

埋点日志的 schema 是开放的：用户可以上报任意 key。ES 的动态 mapping 默认会**为每个新字段自动创建 mapping**：

```json
// 一条日志
{
  "user_action_btn_v1_clicked_count": 1,
  "user_action_btn_v2_clicked_count": 0,
  "user_action_btn_v3_clicked_count": 5,
  ...   // 几千个字段
}
```

每条日志可能新增几千个字段。一周后单 index 的 mapping 有 50 万字段。**mapping 是 cluster state 的一部分**，每次更新 mapping 都要全集群同步，cluster state 膨胀到 GB 级，节点之间同步状态都要几秒。

更糟：每个字段都占一份 Lucene 资源（FieldInfos、segment 元信息），**ES 默认 `index.mapping.total_fields.limit=1000`** 是兜底，但运维往往把它调到 100000。

#### 修复

**方案 A**：禁掉动态 mapping，强制走 schema

```bash
PUT /events
{
  "mappings": {
    "dynamic": "strict",        // strict：未知字段直接拒
    "properties": {
      "event_type": { "type": "keyword" },
      "user_id": { "type": "keyword" },
      "timestamp": { "type": "date" },
      "props": { "type": "object", "enabled": false }   // ★ 整个对象不索引，只存 _source
    }
  }
}
```

`enabled: false` 让 ES **存原文但不索引**。这意味着不能搜索 props 内的字段，但能在结果里返回。适合大部分埋点场景。

**方案 B**：用 `flattened` 类型（7.3+）

```json
{
  "mappings": {
    "properties": {
      "labels": { "type": "flattened" }
    }
  }
}
```

`flattened` 把整个对象当一个字段索引，所有 key/value 拼成 `key=value` 形式存入。代价：聚合粒度受限，但 mapping 不再爆炸。

**方案 C**：动态模板限制（保留灵活性，但限定类型）

```json
{
  "mappings": {
    "dynamic_templates": [
      {
        "strings_as_keyword": {
          "match_mapping_type": "string",
          "mapping": { "type": "keyword", "ignore_above": 256 }
        }
      }
    ]
  }
}
```

无论新字段是什么名字，全部当 keyword（不分词）+ ignore_above 256。

**方案 D**：拆 index 隔离

不要把所有业务的日志塞一个 index。按业务拆，每个 index 独立 mapping，动态字段问题不会扩散。

#### 验证

```bash
# 字段数检查
GET /events/_mapping
# 字段数应稳定，不应每天涨

# cluster state 大小
GET /_cluster/state?filter_path=metadata.indices.events
```

> **口诀**：动态 mapping = mapping 爆炸 = cluster state 爆炸 = 全集群慢。**埋点 / 用户上报数据用 `dynamic: strict` 或 `flattened`**。

---

### 案例 ⑨ snapshot 失败 —— "S3 备份偶发 403、增量越来越大"

#### 现象

每天凌晨的 snapshot 任务有 30% 失败率。失败日志：

```
RepositoryException: [s3-backup] could not read repository data
... AmazonS3Exception: Access Denied (Service: Amazon S3; Status Code: 403)
```

失败的偶发性意味着不是配置错。同时 snapshot 大小越来越大，明明每天数据增量稳定。

#### 根因

**问题 1：S3 权限不足**

snapshot repository 需要的最小 S3 权限：

```json
{
  "Action": [
    "s3:ListBucket",
    "s3:GetBucketLocation",
    "s3:ListBucketMultipartUploads",
    "s3:ListBucketVersions"
  ],
  "Resource": "arn:aws:s3:::my-es-backup"
},
{
  "Action": [
    "s3:GetObject",
    "s3:PutObject",
    "s3:DeleteObject",
    "s3:AbortMultipartUpload",
    "s3:ListMultipartUploadParts"
  ],
  "Resource": "arn:aws:s3:::my-es-backup/*"
}
```

漏了 `AbortMultipartUpload` → 大文件上传失败时残片清不掉，下次重试也容易失败。

**问题 2：snapshot 是增量但没复用**

ES snapshot 是文件级增量：相同 segment 文件名在 repository 里只存一份。但 **segment 名字在 merge 后会变**！每天后台 merge 不停发生，"等价的旧数据"换了新文件名 → 全量上传一遍。

如果你的索引每天大量 update/delete，merge 频繁，看似增量备份，实际上传量接近全量。

#### 修复

**修问题 1**：补全权限 + 配 IAM 角色（推荐 instance role 而非 access key）

```yaml
# elasticsearch.yml
s3.client.default.role_arn: arn:aws:iam::xxx:role/es-snapshot
```

**修问题 2**：减少不必要的 merge

```bash
# 时序索引在 warm 阶段做一次 forcemerge，之后不再变
POST /logs-2026.05.01/_forcemerge?max_num_segments=1

# 然后做 snapshot，之后这个索引不再写、不再 merge → 增量为 0
```

**修问题 3**：分类做 snapshot

```bash
# 当前活跃索引：每小时 snapshot，保留 24 份
# 历史归档索引：每周 snapshot，保留 4 份
PUT /_snapshot/s3-hot/snap-{now/h}
{
  "indices": "logs-current",
  "ignore_unavailable": true,
  "include_global_state": false
}
```

**修问题 4**：用 SLM (Snapshot Lifecycle Management)

```bash
PUT /_slm/policy/daily-backup
{
  "schedule": "0 30 1 * * ?",
  "name": "<daily-snap-{now/d}>",
  "repository": "s3-backup",
  "config": {
    "indices": ["*"],
    "include_global_state": true
  },
  "retention": {
    "expire_after": "30d",
    "min_count": 5,
    "max_count": 50
  }
}
```

SLM 自动调度 + 保留策略，比自己写 cron 可靠。

#### 验证

```bash
# 看 snapshot 大小变化趋势
GET /_snapshot/s3-backup/_all?verbose

# 健康指标：增量 snapshot 大小应远小于全量（10% 以内）
# 失败率 < 1%
```

> **口诀**：snapshot = 文件级增量；merge 频繁 = 增量失效；权限缺一不可（特别是 abort）。**冷数据 forcemerge 后再 snapshot 才有"增量"效果**。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 集群拓扑
- [ ] master eligible 节点数为 3 或 5（永远奇数）
- [ ] 大集群拆角色：3 master 节点（小机器）+ N data 节点（大机器）
- [ ] 跨机房部署配置 zone awareness（`node.attr.zone` + `cluster.routing.allocation.awareness`）
- [ ] `cluster.initial_master_nodes` 仅集群首次启动时配置，集群成立后从所有节点配置中删除
- [ ] `discovery.seed_hosts` 列出所有 master eligible 的地址

### JVM 与 OS
- [ ] heap ≤ 30G，物理内存的另一半留给 OS page cache
- [ ] `-Xms == -Xmx`，避免堆动态调整
- [ ] swap 关闭（`vm.swappiness=1` 或 mlockall）
- [ ] `vm.max_map_count >= 262144`
- [ ] `ulimit -n` 65536+（文件句柄）
- [ ] G1GC（默认即可），Full GC > 500ms 必须告警

### 索引设计
- [ ] 单分片大小目标 10~50GB（日志/全文）或 2~10GB（聚合）
- [ ] 整集群 primary shard 数 ≤ 节点数 × 600
- [ ] 时序数据必上 ILM + rollover + alias
- [ ] mapping `dynamic: strict` 或 `flattened`，禁止动态字段爆炸
- [ ] text 字段用 multi-field（`text` + `keyword`），keyword 加 `ignore_above`
- [ ] 永远不在 text 字段开 `fielddata: true`
- [ ] 嵌套对象明确选 nested / object / join / flat，不要默认

### 写入
- [ ] bulk batch 5k~10k，不带 `refresh=true`
- [ ] 大批量历史导入：refresh -1 + replica 0 + 完成后 forcemerge
- [ ] translog `durability=request`（除非业务能容忍最近 5s 数据丢）
- [ ] 写入失败重试上限，避免雪崩

### 查询
- [ ] 禁止 `from+size > 10000`，深翻页用 `search_after + PIT`
- [ ] 聚合字段必须有 `doc_values`（默认）
- [ ] wildcard / regex 查询限制使用范围
- [ ] 客户端配 `preference=_local` 或固定 routing

### 监控
- [ ] cluster health（GREEN / YELLOW / RED）
- [ ] node 级：heap used %、cpu、disk used %、JVM GC time
- [ ] index 级：primary 数、replica 数、segments count
- [ ] 写入：indexing rate、indexing latency、rejected
- [ ] 查询：query rate、query latency P95/P99、fetch latency
- [ ] 慢日志：`index.search.slowlog.threshold.query.warn: 5s` 等
- [ ] circuit breaker tripped 数

### 容灾
- [ ] SLM 配置每日 snapshot 到 S3 / OSS，保留 30 天
- [ ] 跨机房 CCR (Cross-Cluster Replication) 或定期全量备份
- [ ] 演练 restore 流程，确保 snapshot 真的可用

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### OpenSearch 与 Elastic 的分裂
- 2021 年 Elastic 改 license（Elastic License v2 / SSPL），AWS fork 出 OpenSearch
- API 兼容性的边界：基础 CRUD 兼容；Security / Alerting 等高级功能两边各自实现
- 如何评估迁移：客户端 SDK、Kibana → OpenSearch Dashboards 替换

### Vector Search（kNN / HNSW）
- ES 8.x 引入的 dense_vector + HNSW 算法，本地实现 kNN 搜索
- HNSW (Hierarchical Navigable Small World) 的图结构与查询代价
- 与 Faiss、Milvus、Pinecone 的对比 —— ES 的优势是"全文检索 + 向量检索一站式"

### 全文检索 vs 向量检索的混合
- BM25 (词频/逆文档频) 评分 vs cosine similarity 评分
- 混合检索（hybrid search）：RRF (Reciprocal Rank Fusion) 融合两种排序
- 实际场景：商品搜索 = 关键词匹配 + 语义相近的结果

### Logstash / Beats / Fluent Bit
- Beats 是轻量收集，Logstash 是重量处理 —— 何时用哪个
- Fluent Bit 在云原生场景对 Logstash 的替代
- ingest pipeline 在 ES 内做处理 vs 在 Logstash 做处理的取舍

### ES SQL 与 EQL
- ES SQL：JDBC/ODBC 接口跑 SQL 翻译成 DSL，便于 BI 工具接入
- EQL (Event Query Language)：为安全场景设计，按时间序列匹配模式（`A then B then C`）
- 与 KQL（Kibana Query Language）的层次关系

### 时序场景：ES vs ClickHouse
- ES 强项：全文搜索、灵活 schema、丰富聚合
- ClickHouse 强项：列存压缩高、聚合 throughput 高一个数量级、SQL 原生
- 取舍：日志查询多 → ES；指标聚合多 → ClickHouse；预算只够一套 → 看读写比

### Searchable Snapshot（冷热分层）
- ES 7.10+ 的功能：冷数据完全在 S3，无需占本地磁盘
- 工作机制：本地只缓存 segment 元信息，查询时按需从 S3 拉
- 适用场景：年级别归档、合规留存数据
- 代价：查询延迟从 ms 级到秒级

### 一致性模型
- ES 默认 `one`：primary 写完即返回，replica 异步
- `quorum` / `all` 写一致性：等多数 replica ack
- 与 Kafka、Cassandra 的一致性模型对比

### Lucene 9 新特性
- 改进的 BKD-Tree（数值/地理类型索引）
- 列存（doc_values）压缩进一步优化
- 增量 sort（top-K with early termination）

---

## 7. 引用

- Elasticsearch 8.x 源码：[github.com/elastic/elasticsearch](https://github.com/elastic/elasticsearch)
- Lucene 9.x 源码：[github.com/apache/lucene](https://github.com/apache/lucene)
- Elastic 官方文档：[elastic.co/guide/en/elasticsearch/reference/current](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- Lucene in Action（第二版）—— Lucene 内部结构经典书
- Elasticsearch: The Definitive Guide —— 官方早期版本的设计思路（虽然已过时，但模型仍适用）
- 《深入理解 Elasticsearch》—— Rafal Kuc / Marek Rogozinski
- HNSW 论文：Malkov & Yashunin, *Efficient and robust approximate nearest neighbor search using Hierarchical Navigable Small World graphs*（2018）
- BM25 论文：Robertson, *Okapi at TREC*
- AWS OpenSearch 文档：[opensearch.org/docs](https://opensearch.org/docs/latest/)

---

> 配套：[Redis 内核与一致性](/article/db/redis-internals) | [MySQL 索引深入](/article/db/mysql-index) | [日志架构](/article/observability/logging) | [RAG 工程化](/article/ai/rag-engineering)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
