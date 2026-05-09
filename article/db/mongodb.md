# MongoDB 深入 —— 文档模型、副本集、分片与生产事故

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。MongoDB 在中文社区的口碑像过山车 —— 早年被吹成"NoSQL 银弹"，后来被骂"丢数据"，4.0+ 之后又卷土重来。
> 这里要回答的，是**写文档模型遇到坑、半夜被 Primary 切换告警叫起来**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. MongoDB 4.0 才支持多文档事务。在那之前，"原子性"只到**单文档**这一层 —— 那为什么 `$inc` 改两个嵌入字段是原子的，但跨文档不行？嵌入设计能不能完全替代事务？
2. WiredTiger 是个 B-tree 引擎，但它和 InnoDB 的 B+tree 有几个反直觉的差别：**append-only**、**没有 doublewrite**、**Leaf 节点压缩**。这些设计能扛住断电吗？
3. 副本集（Replica Set）有 Primary / Secondary / Arbiter 三种角色。**Arbiter 永远不要用** —— 为什么？官方文档自己都开始劝退它，根本原因是什么？
4. 5 种读偏好（Read Preference）：`primary` / `primaryPreferred` / `secondary` / `secondaryPreferred` / `nearest`。哪一种**最容易让你读到 5 分钟前的数据**？
5. 写关注（Write Concern）的 `w: 1` 和 `w: majority`，吞吐能差几倍？什么场景下 `w: 1` 会**真的丢数据**？
6. 分片集群里 Shard Key 一旦选定**很难改**（4.4 之前根本不能改）。`{userId: 1}` 和 `{userId: "hashed"}` 的差别不只是"分布均匀"那么简单 —— 范围查询会怎么不一样？
7. 一个 Aggregation Pipeline 跑到一半 OOM，错误是 `Exceeded memory limit for $group` —— 默认 100MB 是哪个参数？怎么让它 spill 到磁盘？
8. MongoDB 的 16MB 单文档上限是怎么来的？踩到怎么破？为什么 GridFS 不是一个好答案？

如果有半数答不上，往下看。

---

## 1. 底层模型：文档、WiredTiger、副本集与分片

绝大多数中文 MongoDB 教程止于"insert / find / update"和"它是 NoSQL"。但这层认知解释不了上面 8 个问题。要看懂它们，必须把 MongoDB 拆成三层来看：**文档模型层、存储引擎层、分布式协议层**。

### 1.1 文档模型：BSON、嵌入、引用

MongoDB 的文档不是"JSON 直接存"，是 **BSON（Binary JSON）**：

```javascript
// 你写的 JSON
{ "name": "alice", "age": 30, "tags": ["a", "b"] }

// BSON 编码后大致结构
\x40\x00\x00\x00          // 文档总长度（小端 int32）
\x02 name\x00 \x06\x00\x00\x00 alice\x00   // type=02(string), key, len, value
\x10 age\x00  \x1e\x00\x00\x00              // type=10(int32), key, value
\x04 tags\x00 ...                            // type=04(array)
\x00                       // 文档结束符
```

为什么不直接用 JSON？三个原因：
- **类型更丰富**：JSON 只有 string/number/bool/null/array/object，BSON 加了 `Date`、`ObjectId`、`Decimal128`、`BinData`、`Timestamp` 等
- **有长度前缀**：跳过文档不需要解析 —— 这是 mongod 内部能高效"扫文档但不解析"的关键
- **二进制更小、更快**：尤其是数字类型，JSON 用字符串表示数字（"123456"），BSON 用 4/8 字节定长

**ObjectId 的设计值得拆开看**：

```
ObjectId = 12 bytes
  ┌────────────────┬──────────┬──────────┬──────────┐
  │ 4B 时间戳(s)    │ 5B 随机   │ 3B 自增   │          │
  │ epoch since 1970│ per-process│per-process│          │
  └────────────────┴──────────┴──────────┴──────────┘
```

- 单调递增（粗略）→ 主键聚簇时新插入大致顺序写
- 内嵌时间 → `_id.getTimestamp()` 拿创建时间，不需要额外字段
- 不需要中心化生成 → 客户端直接生成，写入零往返

> **口诀**：BSON 不是"二进制 JSON"，是为了让 mongod 不解析就能扫描；ObjectId 是为了让客户端无中心地生成主键。

### 1.2 嵌入 vs 引用：建模的核心抉择

文档模型给你一个新自由度：**字段可以是嵌套对象/数组**。这把双刃剑：

```javascript
// 嵌入（embedded）
{
  _id: 1,
  name: "alice",
  orders: [
    { orderId: "A1", amount: 100 },
    { orderId: "A2", amount: 200 }
  ]
}

// 引用（referenced，类似关系库的外键）
// users 集合
{ _id: 1, name: "alice" }
// orders 集合
{ _id: "A1", userId: 1, amount: 100 }
{ _id: "A2", userId: 1, amount: 200 }
```

| 维度 | 嵌入 | 引用 |
| --- | --- | --- |
| 读 | 一次查询拿完整对象 | 需要 `$lookup` 或两次查询 |
| 写 | 单文档原子更新 | 跨文档需要事务（4.0+） |
| 大小 | 受 16MB 限制 | 无 |
| 一致性 | 强 | 弱（除非用事务） |
| 适合场景 | 强从属、一起读取（订单+订单项） | 多对多、独立生命周期 |

**铁律**：嵌入数组的元素数量必须**有上界**。`comments`、`logs`、`events` 这种**理论上无界**的子集合，永远用引用。原因：超过 16MB 就插不进去；即使不到，读 50MB 文档去改 1 个字段也是巨大的写放大。

### 1.3 WiredTiger：B-tree + MVCC + 压缩

MongoDB 3.2 之后默认存储引擎是 WiredTiger。它和 InnoDB 都是 B-tree 家族，但设计差别很大：

```
              WiredTiger 一个 collection
                       │
                  ┌────┴────┐
              内存 cache    磁盘 .wt file
              (eviction)        │
                            ┌───┴───┐
                          B-tree   journal
                          (页面)   (WAL)
                            │
                       ┌────┴────┐
                    Internal   Leaf
                    (路由)     (数据 + KEYS 压缩)
```

几个反直觉的设计：

- **页面是变长的**：默认 32KB，但被压缩后可能只有 8KB。InnoDB 是定长 16KB，WiredTiger 用变长换压缩比。
- **没有 doublewrite buffer**：InnoDB 怕"页面写一半"（partial page write），用 doublewrite 区先写一遍。WiredTiger 用 **checkpoint + journal** 替代：checkpoint 写盘后才更新 metadata，崩溃时直接回到上一个 checkpoint + 重放 journal。
- **MVCC 在内存中**：每个页面在内存里挂一条修改链（update list），事务读取时按时间戳查可见版本。这点像 PostgreSQL 的元组级 MVCC，不像 InnoDB 的"主表 + undo 链"。
- **append-only 写入**：被修改的页面在内存中标记 dirty，evict 时整页写到**新位置**，旧位置作为 reusable space 回收。这就是 WiredTiger 的 LSM-flavor 部分。

```javascript
// 看你的 WiredTiger 配置
db.serverStatus().wiredTiger.cache
// 关键字段：
// "bytes currently in the cache": 12000000000     // 当前用了多少
// "maximum bytes configured":     16000000000     // 上限（默认 RAM 一半）
// "tracked dirty bytes in the cache": 800000000   // dirty 数据量
// "pages evicted because they exceeded the in-memory maximum count":  // evict 大页次数
```

> **口诀**：WiredTiger 是个"内存优先 + checkpoint 持久化"的 B-tree。**Cache 命中率 < 80% 就要警惕**，因为 evict 出去的都是可能再读回来的。

### 1.4 副本集：Primary / Secondary / Arbiter + Oplog

副本集（Replica Set）是 MongoDB 高可用的基本单位。最小配置：

```
        ┌───── Primary ──────┐
        │   接受所有写        │
        │   生成 oplog       │
        └────────┬──────────┘
                 │ 复制 oplog（pull 模式）
        ┌────────┼─────────┐
        ▼                   ▼
   Secondary 1         Secondary 2
   (回放 oplog)        (回放 oplog)
```

几个核心概念：

- **Oplog**：Primary 上的一个特殊 capped collection（`local.oplog.rs`），记录所有写操作。Secondary 通过 long-poll 拉取并回放。
- **Heartbeat**：节点间每 2 秒互相 ping，10 秒没回应就认为对方挂了，触发选举。
- **选举**：基于 Raft（MongoDB 3.2+ 用的是改良 Raft，叫 PV1）。需要**多数派**（majority）投票才能选出 Primary。
- **Term**：每次成功选举 term + 1。Secondary 拿到带更高 term 的 oplog 就承认新 Primary。

为什么 Arbiter（仲裁者）**永远不要用**？

```
错误配置：1 Primary + 1 Secondary + 1 Arbiter
- 看起来 3 节点能选举
- 但只有 2 份数据副本
- Primary 挂了 → Secondary 升主，但它可能没有 Primary 最后的写
- 因为 w:majority = 2，写到 1P+1S 就算成功，Arbiter 不复制数据

风险：
- 数据安全性 = 2 副本（不是 3）
- w:majority 失败时 Primary 自降级（4.4+），写入完全停止
- "PSS（3 数据节点）" 的所有缺点 + "PSA" 的额外脆弱
```

**正确做法**：永远 PSS（Primary + 2 Secondary），不要 PSA（Primary + Secondary + Arbiter）。如果是因为成本，那就接受单实例，至少老老实实知道自己没高可用。

> **口诀**：副本集最小是 3 数据节点。Arbiter 是个历史遗留陷阱，2024 年后官方文档已经在劝退。

### 1.5 分片集群：Mongos + Config Server + Shard

副本集解决"高可用 + 读扩展"，但单副本集的写吞吐是上限（Primary 单点）。要水平扩展写，必须**分片**（Sharded Cluster）：

```
                ┌──────────────────────────┐
                │  应用 (driver)            │
                └──────────┬───────────────┘
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
          mongos 1                  mongos 2          ← 路由层（无状态）
              │                         │
        ┌─────┴─────────────────────────┴──────┐
        │           查 chunk 路由表              │
        ▼                                      ▼
   Config Server (副本集)            ┌────────┴────────┐
   存 chunk → shard 的映射            ▼                 ▼
                                Shard 1 RS       Shard 2 RS
                                (P + 2S)         (P + 2S)
                                chunk[a-m)        chunk[m-z)
```

四个组件：
- **mongos**：无状态路由器，应用连它。本身不存数据，只把请求路由到对应 shard
- **Config Server**：存元数据（哪个 chunk 在哪个 shard 上）。本身也是副本集，3 节点
- **Shard**：每个 shard 是一个独立的副本集，存实际数据
- **Balancer**（运行在 Config Server 的 Primary 上）：监测 chunk 分布，自动迁移让分布均匀

**Shard Key** 是分片的灵魂。它决定了：
- 数据如何切分成 chunk
- 查询能否被路由到单 shard（targeted）还是必须广播到所有 shard（scatter-gather）
- 写入是否会形成**热点**

| Shard Key 类型 | 写分布 | 范围查询 | 典型场景 |
| --- | --- | --- | --- |
| **Ranged**（如 `{userId: 1}`） | 可能不均（用户 ID 单调时全压最后一个 chunk） | 高效（连续 chunk） | 时序、ID 单调时危险 |
| **Hashed**（如 `{userId: "hashed"}`） | 极均 | 低效（broadcast） | 单点查询多、范围少 |
| **Compound**（如 `{tenantId: 1, ts: 1}`） | 多租户均 + 租户内有序 | 高效 | SaaS 场景常用 |

后面案例 ④ 会展开"选错 Shard Key"的代价。

### 1.6 读偏好（Read Preference）

驱动层的概念，控制 SDK 把读请求发给哪个节点：

| 模式 | 行为 | 一致性 | 延迟 | 何时用 |
| --- | --- | --- | --- | --- |
| `primary` | 必须读 Primary | 最强（看到所有已 ack 的写） | 取决于 Primary 负载 | 默认。事务、强一致读 |
| `primaryPreferred` | 优先 Primary，挂了找 Secondary | 大多数时强 | 同上 | 故障容忍但偏好强一致 |
| `secondary` | 必须读 Secondary | 弱（可能滞后秒级） | 通常更低 | 报表、分析、能容忍滞后 |
| `secondaryPreferred` | 优先 Secondary，没 Sec 就 Primary | 弱 | 低 | 分摊读压力 |
| `nearest` | 网络延迟最低的（不论角色） | 弱 | 最低 | 多地域，对延迟敏感 |

`secondary` 模式下，"可能滞后秒级"这个表述有误导性 —— **理论上限是无限**。如果 Secondary 出了故障一直没追上，应用代码读它就一直读到老数据。需要**配合 `maxStalenessSeconds`** 限制最大可接受的滞后。

```javascript
// driver 配置
db.collection.find({}, {
  readPreference: { mode: "secondary", maxStalenessSeconds: 90 }
})
// 滞后超过 90 秒的 Secondary 不会被选中
```

> **口诀**：`secondary` 不是"读更快"的免费午餐，是**用一致性换吞吐**。配 `maxStalenessSeconds` 是底线。

### 1.7 写关注（Write Concern）和读关注（Read Concern）

写关注控制"写算成功"的条件：

```javascript
db.users.insertOne({ name: "alice" }, {
  writeConcern: { w: "majority", j: true, wtimeout: 5000 }
})
```

| 选项 | 含义 |
| --- | --- |
| `w: 0` | 不等任何确认（fire-and-forget），最快但写丢了你都不知道 |
| `w: 1` | Primary 收到就返回（默认）。如果 Primary 还没复制就挂了，rollback 时丢 |
| `w: "majority"` | 多数派节点确认（3 节点中 2 个） |
| `w: N` | 指定 N 个节点确认 |
| `j: true` | 等 journal 落盘（崩溃后能恢复） |
| `wtimeout` | 等待超时时间（毫秒） |

**`w: 1` 真的会丢数据**：场景是 Primary 写了本地、还没复制给 Secondary 就 crash。新 Primary 选出来后，老 Primary 起来发现自己有新 Primary 没有的数据，会做 **rollback**（数据被写到 `rollback/` 目录，需要人工恢复）。

读关注控制"读到的数据"的可见性：

| Read Concern | 含义 |
| --- | --- |
| `local`（默认） | 读本地最新（包括尚未复制到多数派的写） |
| `available` | 类似 local，但分片场景下不查 config 元数据，可能不一致 |
| `majority` | 只读已被多数派确认的写（需要 enableMajorityReadConcern，4.2+ 默认开） |
| `linearizable` | 线性一致读，必须从 Primary 读，需配 `w: majority` 才有意义 |
| `snapshot` | 事务内的一致快照读 |

**生产推荐组合**：
- 默认 `w: majority` + `j: true`（写）+ `local`（读）
- 强一致场景 `w: majority` + `readConcern: majority`
- 事务内 `readConcern: snapshot`

> **口诀**：`w: majority` 是写的底线，`maxStalenessSeconds` 是读的底线。两个不配，副本集就是个"看起来高可用"的玩具。

### 1.8 多文档事务（4.0+）

MongoDB 4.0 才有副本集事务，4.2 才有分片事务。语法和关系库类似：

```javascript
const session = db.getMongo().startSession();
session.startTransaction({
  readConcern: { level: "snapshot" },
  writeConcern: { w: "majority" }
});

try {
  session.getDatabase("app").accounts.updateOne(
    { _id: "alice" }, { $inc: { balance: -100 } });
  session.getDatabase("app").accounts.updateOne(
    { _id: "bob" }, { $inc: { balance: 100 } });
  session.commitTransaction();
} catch (e) {
  session.abortTransaction();
  throw e;
}
```

**事务的代价**（必须知道）：
- 默认 60 秒超时（`transactionLifetimeLimitSeconds`），超时强制 abort
- 跨分片事务用 **2PC**（两阶段提交），延迟显著
- 事务期间被修改的文档**不能被其他写读**（持锁），影响并发
- WiredTiger 的 cache pressure 会因为事务的"快照保留"而上升

实测对比（详见 §3.5）：单文档事务比无事务慢 30%~50%，跨分片事务慢 3~5 倍。

> **口诀**：能用嵌入避免事务就嵌入；能用单分片完成就别跨分片；非要用就把事务体压到 < 100ms。

---

## 2. 源码切片：WiredTiger、Oplog、选举、Mongos

> 基于 MongoDB 7.0 / WiredTiger 11.x 的源码片段。注释为本文添加。

### 2.1 WiredTiger 的 B-tree 写入

WiredTiger 的写入**不直接修改页面**，而是挂"修改链"：

```c
// wiredtiger/src/btree/bt_cursor.c 简化
int __wt_btcur_insert(WT_CURSOR_BTREE *cbt) {
    WT_PAGE *page;
    WT_UPDATE *upd;

    page = cbt->ref->page;

    // ① 构造 UPDATE 节点（heap 上一个小对象）
    WT_RET(__wt_update_alloc(session, &cbt->iface.value, &upd, &size, WT_UPDATE_STANDARD));
    upd->txnid = session->txn->id;          // 当前事务 ID
    upd->start_ts = session->txn->commit_timestamp;

    // ② 链到当前 key 的 update list 头
    WT_RET(__wt_row_modify(session, cbt, ..., upd, ...));
    // 内部把 upd->next 指向旧 head，CAS 替换 head 指针

    // ③ 标记页面 dirty（不立即落盘）
    __wt_page_modify_set(session, page);

    return 0;
}
```

**逐段点透**：

- **WT_UPDATE 是个链表节点**：每个 key 的修改都挂在 leaf 页面的 update list 上，按时间戳倒序。读取时根据事务的 read_timestamp 找第一个 visible 的版本。
- **CAS 替换链表头**：无锁并发写。这就是 WiredTiger 高并发写吞吐的关键 —— 不像 InnoDB 那样需要页锁。
- **页面只标记 dirty，不立即写盘**：写盘由 **eviction 线程**异步完成，触发条件是 cache 占用率超过阈值（默认 80% 软上限，95% 硬上限）。

反直觉点：**写入返回了不代表数据已经在 B-tree 节点里**。它只是挂在 update list 上。要等 eviction reconcile 时才整合进页面写盘。崩溃恢复靠 journal。

### 2.2 Oplog 的循环 capped collection

Oplog 是个固定大小的环形集合：

```javascript
// 看 oplog 配置
db.getReplicationInfo()
// {
//   "logSizeMB": 51200,                // 总大小（默认 5% 磁盘）
//   "usedMB": 21340,
//   "timeDiff": 86400,                 // 第一条到最后一条的时间差（秒）
//   "timeDiffHours": 24,
//   "tFirst": "Mon Jan 15 2024 00:00:00",
//   "tLast":  "Tue Jan 16 2024 00:00:00"
// }
```

oplog 的核心数据结构是 capped collection：

```cpp
// mongo/db/repl/oplog.cpp 简化
Status logOp(OperationContext* opCtx, MutableOplogEntry* entry) {
    auto oplogColl = autoColl.getCollection();

    // ① 分配下一个 OpTime（term + timestamp）
    auto opTime = LogicalClock::get(opCtx)->reserveTicks(1);
    entry->setOpTime(opTime);

    // ② 写入 oplog（capped collection 自动覆盖最老的）
    Status s = oplogColl->insertDocument(opCtx, entry->toBSON(), nullptr);

    // ③ 唤醒等待的 Secondary
    JournalListener::get(opCtx)->onDurable(opTime);
    return s;
}
```

**逐段点透**：

- **OpTime = (timestamp, term)**：timestamp 单调递增（毫秒级），term 是当前选举周期。Secondary 拉 oplog 时按 OpTime 比较，能精确排序。
- **capped collection 是个环**：写满后从头覆盖。这就是为什么 oplog "时间窗口"会变化 —— 高负载时 24 小时的 oplog 可能 1 小时就被覆盖了。
- **Secondary 拉 oplog 用 long-poll**：连一个 tailable cursor，Primary 有新写入就推送，没有就 hang 着。这比轮询省资源。

**生产坑**：如果 Secondary 因为故障下线很久，回来时发现自己的最后 OpTime 已经被 Primary 的 oplog **覆盖了**（没了），就进入 `RECOVERING` 状态，必须**全量同步**（initial sync），耗时数小时。这就是案例 ① 的根。

### 2.3 副本集的心跳与选举（PV1，Raft-like）

```cpp
// mongo/db/repl/topology_coordinator.cpp 简化
HeartbeatResponseAction TopologyCoordinator::processHeartbeatResponse(
    Date_t now, Milliseconds networkRoundTripTime,
    const HostAndPort& target, const StatusWith<ReplSetHeartbeatResponse>& hbResponse) {

    // ① 更新对方的最后心跳时间
    MemberData& memberData = _memberData[memberIndex];
    memberData.setLastHeartbeat(now);

    if (!hbResponse.isOK()) {
        // 网络异常 → 标记 down
        memberData.setDownValues(now, hbResponse.getStatus().reason());
        // ② 如果对方是 Primary 且我已经超时没收到它，触发选举
        if (memberData.getMemberId() == _currentPrimaryMemberId
            && now - memberData.getLastHeartbeat() > _electionTimeout) {
            return HeartbeatResponseAction::makeElectAction();
        }
        return HeartbeatResponseAction::makeNoAction();
    }

    auto hbReply = hbResponse.getValue();
    memberData.setUpValues(now, hbReply);

    // ③ 如果对方报告它是 Primary 且 term 比我高 → 承认它
    if (hbReply.hasState() && hbReply.getState() == MemberState::RS_PRIMARY) {
        if (hbReply.getTerm() > _term) {
            return HeartbeatResponseAction::makeStepDownSelfAction();
        }
    }
    return HeartbeatResponseAction::makeNoAction();
}
```

**逐段点透**：

- **心跳间隔默认 2 秒，选举超时默认 10 秒**。这是为了对短暂网络抖动有容忍度，但也意味着 Primary 真挂了至少 10 秒后才有新 Primary。
- **term 单调递增**：每次成功选举 term + 1。Secondary 拿到带更高 term 的写就承认新 Primary。
- **PV1 的 dry-run 投票**：发起选举前先做一次 "我能不能赢" 的预投票，避免不必要的 term 跳跃。这是 Raft 的优化。

**关键认知**：选举期间**集群没有 Primary**，所有写请求会被驱动 **buffer 30 秒**（`serverSelectionTimeoutMS` 默认值），等新 Primary 选出来。如果 30 秒还没选出来，写请求才报错。这是为什么 Primary 切换在应用层"通常感受不到"，但应用必须做好**重试幂等**。

### 2.4 Mongos 的查询路由

```cpp
// mongo/s/query/cluster_find.cpp 简化
std::vector<RemoteCursor> establishCursors(
    OperationContext* opCtx,
    std::shared_ptr<executor::TaskExecutor> executor,
    const NamespaceString& nss,
    const ReadPreferenceSetting& readPref,
    const std::vector<AsyncRequestsSender::Request>& remotes) {

    // ① 根据 shard key 范围算出涉及的 shards
    auto routingInfo = uassertStatusOK(
        Grid::get(opCtx)->catalogCache()->getCollectionRoutingInfo(opCtx, nss));

    std::set<ShardId> shardIds;
    if (query.canTargetSingleShard()) {
        // ② targeted query：只发给一个 shard
        shardIds = {routingInfo.shardKeyToShard(queryFilter)};
    } else {
        // ③ scatter-gather：发给所有 shard
        shardIds = routingInfo.getAllShardIds();
    }

    // ④ 并行向所有目标 shard 发请求
    AsyncRequestsSender ars(opCtx, executor, nss.db(), readPref, requests, ...);
    return ars.waitForResponses();
}
```

**逐段点透**：

- **`canTargetSingleShard`** 决定查询是 targeted 还是 scatter。条件：查询里包含 shard key 的等值条件（或前缀），且不是哈希分片范围查询。
- **scatter-gather 的代价**：N 个 shard 都要扫一遍，结果在 mongos 内存里合并。**P99 延迟 = max(各 shard 延迟)**，任何一个 shard 慢都拖累全局。
- **mongos 是无状态的**：它把路由表缓存在内存（`catalogCache`），balancer 移动 chunk 后会通过 versioning 机制让 mongos 失效缓存重新拉取。

**反直觉**：mongos **不会做大规模 sort/group**。`$sort` 跨 shard 只是把每个 shard 内排好的结果做归并；`$group` 跨 shard 必须每个 shard 先做局部 group，mongos 再合并。如果你的聚合不能下推到 shard，吞吐就上不去。

### 2.5 Aggregation Pipeline 的执行器

```cpp
// mongo/db/pipeline/document_source_group.cpp 简化
DocumentSource::GetNextResult DocumentSourceGroup::doGetNext() {
    if (!_initialized) {
        // ① 把所有上游文档读完，按 group key 哈希到 _groupsMap
        while (auto next = pSource->getNext()) {
            auto& accum = _groupsMap[groupKey(next)];
            for (size_t i = 0; i < _accumulators.size(); ++i) {
                accum[i]->process(/* ... */);
            }
            _memoryUsageBytes += sizeof(/* ... */);

            // ② 内存超限：spill 到磁盘 或 报错
            if (_memoryUsageBytes > _maxMemoryUsageBytes) {
                if (_allowDiskUse) {
                    spillToDisk();
                } else {
                    uasserted(ErrorCodes::QueryExceededMemoryLimitNoDiskUseAllowed,
                              "Exceeded memory limit for $group");
                }
            }
        }
        _initialized = true;
    }
    // ③ 一个一个 emit 结果
    if (_iter == _groupsMap.end()) return GetNextResult::makeEOF();
    auto result = makeOutput(_iter);
    ++_iter;
    return result;
}
```

**逐段点透**：

- **必须读完所有上游再开始 emit**：`$group` 是 **blocking** 阶段。这就是 streaming 阶段（如 `$match`、`$project`）和 blocking 阶段（`$group`、`$sort`）的根本差别。
- **默认 100MB 内存上限**：`internalDocumentSourceGroupMaxMemoryBytes`。超了报 `QueryExceededMemoryLimitNoDiskUseAllowed`。这是案例 ⑤ 的根。
- **`{ allowDiskUse: true }`** 解锁 spill：把部分 group 的中间结果写到 `_tmp/` 目录，归并阶段读回来。**慢 5~10 倍**，但能跑完。

`$sort` 也是同样的设计 —— 需要把所有数据吃进来才能排序，超过 100MB 就要 `allowDiskUse: true`。

### 2.6 Change Streams 的实现

Change Streams 是 3.6 引入的"订阅集合变更"能力。本质是封装好的 oplog tail：

```javascript
// 监听一个集合
const changeStream = db.orders.watch([
  { $match: { "fullDocument.status": "paid" } }
]);

while (!changeStream.isClosed()) {
  if (changeStream.hasNext()) {
    const change = changeStream.next();
    console.log(change);
    // {
    //   _id: { _data: "..." },         // resume token
    //   operationType: "insert",
    //   clusterTime: Timestamp(...),
    //   fullDocument: { ... },
    //   ns: { db: "app", coll: "orders" },
    //   documentKey: { _id: ... }
    // }
  }
}
```

源码层面，Change Streams 是构造一个特殊的 aggregation pipeline：

```cpp
// mongo/db/pipeline/document_source_change_stream.cpp 简化
std::list<intrusive_ptr<DocumentSource>> DocumentSourceChangeStream::createFromBson(...) {
    std::list<intrusive_ptr<DocumentSource>> stages;

    // ① 从 oplog 读
    stages.push_back(DocumentSourceOplogMatch::create(...));

    // ② 转换成 change event 格式
    stages.push_back(DocumentSourceChangeStreamTransform::create(...));

    // ③ 用户的 $match/$project 等附加在后面
    for (auto& userStage : userStages) {
        stages.push_back(userStage);
    }

    return stages;
}
```

**逐段点透**：

- **resume token** = oplog OpTime 的编码。客户端断开重连时把上次的 token 传回去，从那个位置继续。
- **副本集场景**：直接从 Secondary 的 oplog 读（如果 readPref 是 secondary）
- **分片场景**：mongos 同时连所有 shard 的 oplog，按 cluster time 全局排序后输出。**任何一个 shard 慢都拖累全局延迟**。

**反直觉**：Change Streams **必须开启 majority read concern**（默认就是）。否则可能读到老 Primary 上后来被回滚的写，给消费者一个"鬼影"事件。

### 2.7 Compound Index 与 Index Intersection

MongoDB 的索引和 MySQL B+tree 一样有 ESR 规则（Equality, Sort, Range）：

```javascript
// 集合
db.orders.createIndex({ status: 1, createdAt: -1, amount: 1 })

// 高效查询：等值 + 排序 + 范围
db.orders.find({ status: "paid" }).sort({ createdAt: -1 })

// 可以用索引但不那么高效：跳过中间字段
db.orders.find({ status: "paid", amount: { $gt: 100 } })
// → 用 (status) 部分索引，amount 部分变成索引内过滤
```

源码层面 `QueryPlanner::plan` 给候选索引打分，选最优：

```cpp
// mongo/db/query/planner_analysis.cpp 简化
double scorePlan(const QuerySolution& soln, const QueryPlannerParams& params) {
    double score = 0.0;

    // ① 用上索引 → +1
    if (hasIndexScan(soln)) score += 1.0;

    // ② 索引覆盖（不需要回表）→ 大加分
    if (isCovered(soln)) score += 0.5;

    // ③ 索引可以提供 sort 顺序 → 加分
    if (providesSort(soln)) score += 0.3;

    // ④ scan 的 keys 数量越少越好
    score += 1.0 / (1.0 + estimatedKeysExamined(soln));
    return score;
}
```

**Index Intersection**：当一个查询能用多个单字段索引时，MongoDB 可能同时扫多个索引取交集。但**生产中很少触发**，因为：

- 优化器更倾向单索引 + filter
- 多索引 intersection 的 RID 排序合并代价高
- 真正需要时不如建个 compound 索引

> **口诀**：MongoDB 的索引设计和 MySQL 一致 —— **Equality, Sort, Range 顺序**。不要指望 Index Intersection 救你，先建对的 compound 索引。

---

## 3. 实测数据：压缩、读偏好、分片粒度、索引、事务

> 测试环境：3 节点副本集（1P + 2S），每节点 8 vCPU / 32GB / NVMe SSD，MongoDB 7.0。客户端 Java driver 4.11，`w: majority`，连接池 100。

### 3.1 WiredTiger 不同压缩算法的对比

WiredTiger 有三种 block compressor：

```javascript
// 创建集合时指定压缩
db.createCollection("logs", {
  storageEngine: {
    wiredTiger: { configString: "block_compressor=zstd" }
  }
})
```

测试方法：写 1 亿条平均 1KB 的日志型文档（高重复字段）。

| 压缩算法 | 磁盘占用 | 写吞吐 (ops/s) | 读吞吐 (ops/s) | CPU |
| --- | --- | --- | --- | --- |
| `none` | 100 GB | 95,000 | 180,000 | 35% |
| `snappy`（默认） | 38 GB | 92,000 | 175,000 | 38% |
| `zlib` | 22 GB | 64,000 | 150,000 | 65% |
| `zstd`（4.2+） | 24 GB | 88,000 | 170,000 | 42% |

**结论**：
- `snappy` 是合理默认，CPU 几乎没成本
- `zstd` 是更优默认（4.2+），压缩比接近 zlib，CPU 接近 snappy
- `zlib` 仅在磁盘极贵时考虑，CPU 代价大

**索引也可以单独配压缩**（默认 prefix compression）：

```javascript
db.collection.createIndex({...}, {
  storageEngine: { wiredTiger: { configString: "prefix_compression=true" } }
})
```

> **口诀**：日志型 collection（写多读少、重复多）用 zstd 能省 70% 磁盘且几乎不损吞吐。

### 3.2 副本集 Read Preference 对延迟的影响

测试方法：单条点查 P99，对比 5 种读偏好。

| Read Preference | P50 (ms) | P99 (ms) | 一致性风险 |
| --- | --- | --- | --- |
| `primary` | 1.2 | 4.5 | 无 |
| `primaryPreferred` | 1.2 | 4.5 | Primary 故障时降级到 sec |
| `secondary` | 0.9 | 6.8 | 滞后秒级（高峰可达分钟） |
| `secondaryPreferred` | 0.9 | 6.8 | 同上 |
| `nearest` | 0.8 | 5.2 | 弱一致 |

**反直觉点**：
- `secondary` P99 反而比 primary 高，因为 Secondary 同时在做 oplog 回放（写放大），CPU 比 Primary 还高
- 只在 Primary 已经过载时，secondary 才能真的减压

**压力测试 Primary CPU 80% 时**：

| Read Preference | P50 (ms) | P99 (ms) |
| --- | --- | --- |
| `primary` | 8 | 45 |
| `secondaryPreferred` | 1.5 | 8 |

这种情况下 `secondaryPreferred` 显著优于 `primary`。

> **口诀**：`secondary` 不是默认更快。**只在 Primary 过载或地理就近场景下用**，且必须配 `maxStalenessSeconds`。

### 3.3 分片粒度（chunk size）对性能的影响

`chunkSize` 默认 128MB（5.0+，之前是 64MB）。它是 balancer 决定何时切分 chunk 的阈值。

| chunkSize | chunk 总数 | 单点查 P99 | balancer 迁移频率 | config 元数据大小 |
| --- | --- | --- | --- | --- |
| 32 MB | 100,000 | 5 ms | 高（频繁迁移） | 大 |
| 128 MB（默认） | 25,000 | 4 ms | 中 | 中 |
| 256 MB | 12,000 | 4 ms | 低 | 小 |
| 1 GB | 3,000 | 6 ms | 极低 | 极小 |

**反直觉**：chunk 太大不一定好。原因：
- chunk 是 balancer 迁移的最小单位。1GB 的 chunk 迁移一次要拷贝 1GB 数据，期间该范围被 read-only
- 单 chunk 内的"热点"无法通过迁移分散

通常**保持默认 128MB**就好。除非：
- 数据量极大（PB 级）→ 可以加大到 256MB-1GB 减少 chunk 数
- 写压力极不均匀 → 可以减小到 32-64MB 让 balancer 更细粒度

### 3.4 索引数量对写入性能的影响

每个索引都需要在写入时维护。测试一个 10 字段集合，从 0 到 10 个索引：

| 索引数 | 单条 insert 延迟 (μs) | 吞吐 (ops/s) |
| --- | --- | --- |
| 0 | 80 | 125,000 |
| 1 (主键 _id) | 95 | 105,000 |
| 3 | 140 | 71,000 |
| 5 | 195 | 51,000 |
| 10 | 380 | 26,000 |

**结论**：每加一个索引，写吞吐降 10%~20%。**生产 collection 索引数应控制在 5 个以内**。

排查无效索引：

```javascript
// 看每个索引的使用情况
db.orders.aggregate([{ $indexStats: {} }])
// 输出每个索引的 ops 计数
// {
//   name: "status_1_createdAt_-1",
//   key: {...},
//   accesses: { ops: 12345, since: Date(...) },
//   ...
// }
// ops 长期 0 的索引可以删
```

> **口诀**：索引不是越多越好，**> 5 个就需要严肃 review**。`$indexStats` 是发现僵尸索引的唯一靠谱办法。

### 3.5 多文档事务的吞吐损失

测试：单文档 update vs 双文档事务 update。

| 操作 | 吞吐 (ops/s) | P99 (ms) |
| --- | --- | --- |
| 无事务，单文档 `$inc` | 38,000 | 5 |
| 单文档放在事务内 | 26,000 | 8 |
| 跨文档（单分片）事务 | 18,000 | 14 |
| 跨分片事务（2PC） | 4,200 | 95 |

**结论矩阵**：
- 单文档放进事务也有 30% 性能损失
- 跨分片事务比无事务慢约 9 倍
- 事务体里有阻塞操作（远程调用）会让 cache pressure 飙升

**生产铁律**：
1. 优先嵌入设计避免跨文档操作
2. 必须事务时尽量保持单分片
3. 事务体内只做 DB 操作，禁止远程调用
4. 事务设超时 < 5s，避免长事务挤压 cache

> **口诀**：MongoDB 事务是"应急方案"不是"日常工具"。能不用就不用，必须用就压短。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。

### 案例 ① Oplog 太小 —— "从节点掉队后无法追上"

#### 现象

某次大批量数据迁移（一晚上往主库灌了 5 亿条数据），第二天发现一个 Secondary 处于 `RECOVERING` 状态，无法回到 `SECONDARY`。`db.printSecondaryReplicationInfo()` 显示该节点滞后 12 小时。

应用层告警：因为只剩 1 个可用 Secondary，`w: majority` 的写延迟从 5ms 飙到 50ms。

#### 根因

```javascript
// 主库上看 oplog 窗口
db.getReplicationInfo()
// {
//   logSizeMB: 51200,        // 50 GB
//   usedMB: 50800,
//   timeDiff: 3600,          // ★ 只够 1 小时
//   tFirst: ...
//   tLast:  ...
// }
```

oplog 是个环。Secondary 故障期间 Primary 持续写，新的 oplog **覆盖了** Secondary 还没来得及拉的旧 oplog。当 Secondary 恢复想从断点继续时，断点在 Primary 的 oplog 里**已经不存在了**。

MongoDB 此时进入 stale 状态：
- 可选方案 1：从其他 Secondary 拷贝当前最新数据（initial sync）
- 可选方案 2：手动 resync

但因为数据量大，initial sync 跑了 6 小时还在跑。期间集群只有 2 个数据节点，`w: majority` 风险升高。

#### 修复

**临时**：等 initial sync 跑完。

**长效**：oplog 必须足够大，能覆盖 **最长可能的 Secondary 故障窗口**。MongoDB 4.4+ 推荐：

```javascript
// 调整 oplog 大小（不需要重启，4.4+）
db.adminCommand({
  replSetResizeOplog: 1,
  size: 102400              // 100 GB（单位 MB）
})
```

公式：`oplogMB ≥ peakWriteMBps × 期望故障窗口秒 × 2`

例如：peak 写 50 MB/s，希望容忍 24 小时故障，则 oplog ≥ 50 × 86400 × 2 = 8.6 TB。

显然不可能这么大。所以**真正的解法**是：
1. oplog 配 24~48 小时窗口（典型 50~200 GB）
2. 监控 oplog 窗口（`replSetGetStatus.replicationProgress` + `getReplicationInfo`）
3. **Secondary 故障 4 小时内必须恢复**，否则进入运维流程

#### 验证

```javascript
// 监控 oplog window
db.adminCommand("replSetGetStatus").members.forEach(m => {
  print(m.name, "lag:", (Date.now() - m.optimeDate.getTime()) / 1000, "s");
})

// 告警阈值：任一 secondary lag > 600 秒
```

> **口诀**：oplog 是个环，覆盖了就完蛋。**oplog 窗口至少 24 小时**，监控覆盖时间而不是空间。

---

### 案例 ② Primary 选举失败 —— "网络分区导致集群只读"

#### 现象

机房 A、B 各两个节点，C 一个节点，配置 5 节点副本集。某次 A-B 之间光纤抖动 30 秒，集群 30 秒内**无法写入**，应用大面积超时。但单节点没挂，奇怪。

#### 根因

```
机房 A: Node1 (Primary), Node2
机房 B: Node3, Node4
机房 C: Node5

光纤抖动期间：
  A 看见: Node1, Node2, Node5  →  3 个，多数派 ✓
  B 看见: Node3, Node4, Node5  →  3 个，多数派 ✓
```

但 A、B 都看见了 Node5。Node5 同时收到两边的心跳，状态混乱。

实际机理：Node1（原 Primary）发现自己看不到 Node3、Node4，活跃成员只剩 3 个（自己 + Node2 + Node5）。理论上 majority = 3 还能维持。但 **Node5 收到来自 Node3、Node4 的更新心跳**，同时 Node3、Node4 也试图发起选举（它们看不到原 Primary）。

最坏情况：原 Primary 的 majority 心跳被部分阻塞，触发 step down；Node3、Node4 又凑不齐 majority。集群进入"无 Primary"状态 30 秒。

#### 修复

**架构层面**：

1. **跨机房副本集成员数应该是 2-2-1 或 1-1-1，不是 4-0-1**
   - 1-1-1：每个机房 1 个，任意 1 个机房挂仍有 majority
   - 2-2-1：两个机房各 2 个 + 仲裁机房 1 个（注意：是数据节点，不是 Arbiter）

2. **明确"主机房"**：用 priority 让某个机房的节点更倾向于当 Primary

```javascript
cfg = rs.conf();
cfg.members[0].priority = 10;     // 机房 A
cfg.members[1].priority = 10;
cfg.members[2].priority = 5;      // 机房 B
cfg.members[3].priority = 5;
cfg.members[4].priority = 1;      // 机房 C 备
rs.reconfig(cfg);
```

3. **缩短选举超时**（4.0+，谨慎）：

```javascript
cfg.settings.electionTimeoutMillis = 5000;  // 默认 10000
```

代价是网络抖动更容易触发选举。

**应用层面**：

```java
// 驱动设置合理的 retryWrites + serverSelectionTimeout
MongoClientSettings settings = MongoClientSettings.builder()
    .retryWrites(true)                                    // 默认 true，4.2+
    .applyToClusterSettings(b -> b
        .serverSelectionTimeout(15, TimeUnit.SECONDS))    // 选 Primary 等 15 秒
    .applyToSocketSettings(b -> b
        .connectTimeout(3, TimeUnit.SECONDS)
        .readTimeout(10, TimeUnit.SECONDS))
    .build();
```

`retryWrites: true` 让驱动在 Primary 切换时**自动重试一次**写操作。这能屏蔽掉选举的短暂窗口（前提是写操作幂等）。

#### 验证

混沌测试：

```bash
# 在生产前模拟分区
sudo iptables -A INPUT -s <node3-ip> -j DROP
sudo iptables -A INPUT -s <node4-ip> -j DROP

# 观察应用层延迟和错误率
# 期望：< 15 秒内自动切换，写超时不超过 retry 次数
```

> **口诀**：副本集成员的机房分布要对称。priority 决定"谁优先当 Primary"，retryWrites 屏蔽切换窗口。

---

### 案例 ③ 读 Secondary 数据陈旧 —— "用户刚改密码立刻登录失败"

#### 现象

用户在网页改了密码，立刻退出重登 → 提示密码错误。3 秒后再试就成功。

#### 根因

应用配置：
```java
// 全局默认读偏好为 secondaryPreferred 想分摊压力
.readPreference(ReadPreference.secondaryPreferred())
```

流程：
1. 用户改密码 → 写到 Primary（成功）
2. 复制到 Secondary 需要时间（毫秒级，但有时百毫秒）
3. 用户登录请求 → 路由到 Secondary，看到的是**老密码**
4. 比对失败 → 报错

**RYW（Read Your Writes）一致性**在 secondary 读模式下没有保证。

#### 修复

**方案 A**（最简单）：登录这种关键操作**强制 primary**。

```java
collection.withReadPreference(ReadPreference.primary())
    .find(Filters.eq("username", username));
```

**方案 B**（4.4+ 推荐）：使用 **causal consistency** session。

```javascript
// 同一个 session 内保证因果一致（自己的写自己读得到）
const session = db.getMongo().startSession({ causalConsistency: true });

session.getDatabase("app").users.updateOne({ _id: userId }, { $set: { pwd: hash } });

// 后面这条 read 即使路由到 Secondary，也保证看到上面的写
session.getDatabase("app").users.findOne({ _id: userId });
```

机制：session 记录最后一次写的 cluster time，后续读会等 Secondary 复制到这个时间点。

**方案 C**：所有"先写后读同一数据"的关键路径都走 primary。报表/列表用 secondary。

```java
// 在 DAO 层标记
@ReadFromPrimary
public User findByUsername(String username) { ... }
```

#### 验证

```java
@Test
void afterUpdateMustReadOwnWrite() {
    User u = userService.changePassword(userId, "newPwd");
    User read = userService.findById(userId);
    assertEquals("newPwd", read.getHashedPassword());
}
// 在 secondaryPreferred 配置下这个测试会偶发失败
// 修复后必须稳定通过
```

> **口诀**：**写后立即读同一数据 = 必须 primary 或 causal consistency**。secondary 只读那些"我不在意几秒前的版本"的数据。

---

### 案例 ④ Sharding 选错 Shard Key —— "热点单 shard，其他都是闲的"

#### 现象

电商系统按 `{ orderId: 1 }` 做 ranged sharding（orderId 是单调递增的雪花 ID）。上线 6 个月，看监控发现 shard0 写 IOPS 90%，shard1-3 都不到 10%。balancer 看起来在动 chunk，但永远追不上。

#### 根因

orderId 单调递增 → 新订单**永远落在最右边的 chunk** → 该 chunk 总是属于某一个 shard。balancer 不停切分新 chunk 然后迁移，但单调递增的写入永远在追新 chunk，**热点无法消除**。

```
chunk 范围：
  shard0: orderId [0, 1000)
  shard1: orderId [1000, 2000)
  shard2: orderId [2000, 3000)
  shard3: orderId [3000, ∞)    ← 所有新写都来这里
```

balancer 切分 [3000, ∞) 后：
```
  shard3: orderId [3000, 4000)
  shardN: orderId [4000, ∞)    ← 又变成热点
```

**hashed shard key 能避免这个问题**，但范围查询会变成 scatter-gather。

#### 修复

**方案 A**：改成 hashed shard key（4.4 之前需要重建集合，5.0+ 可在线 reshard）。

```javascript
// MongoDB 5.0+ 在线 reshard
db.adminCommand({
  reshardCollection: "shop.orders",
  key: { orderId: "hashed" }
})
// 内部新建影子集合，复制数据，期间双写。耗时按数据量算
```

**方案 B**（更优）：复合 shard key，把高基数字段前置打散。

```javascript
// 例：先按 storeId hash 打散，再按时间排序
db.adminCommand({
  shardCollection: "shop.orders",
  key: { storeId: "hashed", createdAt: 1 }
})
```

这样：
- 不同店铺的订单分散到不同 shard（写均匀）
- 同店铺的订单按时间聚集（"看店铺最近订单"高效）

**方案 C**（适合时序）：pre-split 已知的写热点。

```javascript
// 预先在 shard 上分好 chunk，避免初始热点
sh.shardCollection("metrics.events", { hour: "hashed" }, false, { numInitialChunks: 64 });
```

#### shard key 选择决策树

```
1. 主要查询模式包含某字段的等值或前缀？
   → 该字段必须在 shard key 里
2. 该字段基数足够高（百万级）？
   → 可以用 ranged
3. 该字段单调递增（ID、时间戳）？
   → 必须 hashed 或前面加打散字段
4. 跨字段查询多？
   → 用 compound
```

#### 验证

```javascript
// 检查 chunk 分布
db.orders.getShardDistribution()
// 输出应该接近 25% / 25% / 25% / 25%（4 shard）

// 检查写入热度
db.serverStatus().opLatencies
// 各 shard 的 writeLatencies.ops 应该接近
```

> **口诀**：shard key **一旦选定极难改**，宁可上线前多花一天设计也不要事后 reshard。**单调递增字段绝不能直接 ranged**。

---

### 案例 ⑤ Aggregation 内存溢出 —— "$group 报 Exceeded memory limit"

#### 现象

跑了几个月的报表 SQL，某天突然失败：

```
MongoServerError: Exceeded memory limit for $group, but didn't allow external sort.
Pass allowDiskUse:true to opt in.
```

数据量从去年的 500 万行涨到了 1500 万行。

#### 根因

```javascript
db.events.aggregate([
  { $match: { ts: { $gte: ISODate("2024-01-01") } } },
  { $group: {
      _id: { user: "$userId", action: "$action" },
      count: { $sum: 1 },
      lastSeen: { $max: "$ts" }
  }}
])
```

`$group` 在内存里维护一个 `Map<groupKey, accumulator>`。group key 的基数 = 用户数 × action 类型 = 5万 × 50 = 250 万。每个 accumulator 大约 100 字节，总共 250 MB —— 超过 100 MB 默认上限。

`internalDocumentSourceGroupMaxMemoryBytes` 默认 100MB，超过就报错。

#### 修复

**方案 A**（最快）：开 `allowDiskUse`。

```javascript
db.events.aggregate([
  { $match: { ts: { $gte: ISODate("2024-01-01") } } },
  { $group: { ... } }
], { allowDiskUse: true })
```

代价：spill 到 `_tmp/`，慢 5~10 倍。

**方案 B**（更好）：让 `$group` 能利用索引提前减小数据量，或用 `$bucket` 预聚合。

```javascript
db.events.aggregate([
  // 利用索引扫，按 day 桶预聚合
  { $match: { ts: { $gte: ISODate("2024-01-01") } } },
  { $group: {
      _id: {
        user: "$userId",
        action: "$action",
        day: { $dateTrunc: { date: "$ts", unit: "day" } }
      },
      count: { $sum: 1 }
  }},
  // 第二阶段不在 day 维度
  { $group: {
      _id: { user: "$_id.user", action: "$_id.action" },
      count: { $sum: "$count" }
  }}
], { allowDiskUse: true })
```

**方案 C**：用 `$out` 物化中间结果到一个新集合，分批跑。

```javascript
db.events.aggregate([
  { $match: { ts: { $gte: ISODate("2024-01-01"), $lt: ISODate("2024-02-01") } } },
  { $group: { ... } },
  { $out: "events_jan_summary" }
])
// 每月一个 summary 集合，最后再 merge
```

**方案 D**（生产推荐）：把"分析查询"挪到独立的 Secondary 节点，避免影响 Primary。

```javascript
cfg.members[2].tags = { use: "analytics" };
cfg.members[2].priority = 0;     // 永不当 Primary
cfg.members[2].hidden = true;    // 对客户端不可见
rs.reconfig(cfg);

// 应用层用 tag 路由
db.collection.find({...}).readPref("secondary", [{ use: "analytics" }])
```

#### 限制：`$lookup` 的内存上限

`$lookup` 也是个内存吃货。MongoDB 5.0+ 后可以用 `allowDiskUse`，但更好的做法是**避免大表 join**：把要 join 的字段冗余进文档（去规范化）。

#### 验证

```javascript
// explain 看哪一步用了多少内存
db.events.explain("executionStats").aggregate([...])
// 关注 stages 里每一步的 nReturned 和 executionTimeMillisEstimate
```

> **口诀**：`$group`/`$sort` 默认 100MB 内存上限。**线上必须开 `allowDiskUse`**，但更要从查询设计上避免触发 spill。

---

### 案例 ⑥ TTL 索引清理不及时 —— "集合膨胀 10 倍后磁盘告警"

#### 现象

某个 session 集合配了 TTL 索引（24 小时过期）。某天突然告警磁盘 80%，发现这个集合从预期的 2GB 涨到 30GB。

#### 根因

```javascript
db.sessions.createIndex({ expireAt: 1 }, { expireAfterSeconds: 0 })
```

TTL 索引由后台任务 **TTLMonitor** 每 60 秒扫一次。它的工作方式是：
1. 找索引里 `expireAt < now` 的文档
2. 一条一条删

问题：**TTLMonitor 是单线程的**。如果到期文档积累得太快（业务峰值期一秒新增 1 万条 session，每秒过期 5 千条），TTLMonitor 一次扫 60 秒生成的过期数据，自己删不完。

更糟：删除是 logical delete，要写 oplog；积累的"待删"文档让 B-tree 持续膨胀；磁盘文件**只增不减**（除非 compact）。

#### 修复

**方案 A**：调小 TTL 扫描间隔（4.4+）。

```javascript
db.adminCommand({ setParameter: 1, ttlMonitorSleepSecs: 30 })
```

**方案 B**：让 TTLMonitor 一次删更多（6.0+）。

```javascript
db.adminCommand({ setParameter: 1, ttlMonitorBatchDeletes: true })
db.adminCommand({ setParameter: 1, ttlIndexDeleteTargetTimeMS: 1000 })
// 每个集合一次循环最多花 1 秒，比"删完为止"安全
```

**方案 C**（结构化）：批量清理代替 TTL 索引。业务低峰期跑一个 job：

```javascript
// 不依赖 TTLMonitor，用业务 job 控制速率
const cutoff = new Date(Date.now() - 86400 * 1000);
let total = 0;
while (true) {
  const r = db.sessions.deleteMany(
    { expireAt: { $lt: cutoff } },
    { limit: 10000 }
  );
  total += r.deletedCount;
  if (r.deletedCount < 10000) break;
  sleep(100);   // 让出 IO
}
print("deleted:", total);
```

**方案 D**：磁盘已经膨胀的，跑 `compact`（注意会锁集合）。

```javascript
// 6.0+ 不再阻塞读写（使用 in-place compact）
db.runCommand({ compact: "sessions" })
```

5.0 及之前的版本 `compact` 会阻塞，需要在 Secondary 上一台一台跑（rolling compact）。

#### 监控

```javascript
db.serverStatus().metrics.ttl
// {
//   deletedDocuments: NumberLong("12000000"),
//   passes: NumberLong("8640")            // TTLMonitor 跑了多少次
// }
// passes 跟时间对得上（每分钟一次），但 deletedDocuments 增长慢 → 删不动
```

> **口诀**：TTL 索引是"够用"的方案不是"高吞吐"的方案。**每秒过期 > 5000 条时必须用业务 job 替代**。

---

### 案例 ⑦ 大文档（16MB 上限）拆分 —— "聊天记录涨到 17MB 写入失败"

#### 现象

用户消息 collection 设计成"每个 chat 一个文档，messages 数组嵌入所有消息"。某些活跃 chat 上线半年后涨到 16MB+，写入直接报错：

```
BSONObjectTooLarge: object size: 17234567 is larger than: 16793600
```

#### 根因

MongoDB 单文档硬上限 16MB（`BSONObjMaxUserSize`）。这个限制是早期为防止 mongod 内存爆炸 + 网络包过大设的，不可调。

GridFS 看起来是答案，但实际上：
- GridFS 把文件切成 chunk 存在两个 collection 里（fs.files + fs.chunks）
- 适合**整存整取的二进制大文件**（视频、图片）
- **不适合需要部分更新的结构化数据**（chat 消息）

#### 修复

**正确设计**：消息**不该嵌入**到 chat 文档里。

```javascript
// ❌ 错误：嵌入数组无界增长
{
  _id: "chat_1",
  members: [...],
  messages: [
    { _id: ..., from: ..., text: ..., ts: ... },
    ...10000 条
  ]
}

// ✅ 正确：分文档存储
// chats collection
{ _id: "chat_1", members: [...], lastMessageTs: ... }

// messages collection
{ _id: ..., chatId: "chat_1", from: ..., text: ..., ts: ... }
```

索引：
```javascript
db.messages.createIndex({ chatId: 1, ts: -1 })
// 拉某 chat 最近消息：
db.messages.find({ chatId: "chat_1" }).sort({ ts: -1 }).limit(50)
```

**已经踩坑的怎么迁移**：

```javascript
// 一次性迁移脚本
db.chats.find({ messages: { $exists: true } }).forEach(chat => {
  if (!chat.messages || chat.messages.length === 0) return;

  const docs = chat.messages.map(m => ({
    chatId: chat._id,
    from: m.from,
    text: m.text,
    ts: m.ts
  }));

  db.messages.insertMany(docs, { ordered: false });

  db.chats.updateOne(
    { _id: chat._id },
    {
      $unset: { messages: "" },
      $set: { lastMessageTs: chat.messages[chat.messages.length - 1].ts }
    }
  );
});
```

**防御性设计**：业务层就该有"嵌入数组上限"硬编码。

```java
public void appendMessage(String chatId, Message msg) {
  long count = messageRepository.countByChatId(chatId);
  if (count >= MAX_MESSAGES_PER_CHAT) {
    throw new IllegalStateException("chat full, archive first");
  }
  messageRepository.save(msg);
}
```

#### 验证

```javascript
// 找出当前最大的文档
db.messages.aggregate([
  { $project: { size: { $bsonSize: "$$ROOT" } } },
  { $sort: { size: -1 } },
  { $limit: 10 }
])
// 任何 size > 1MB 的文档都值得 review
```

> **口诀**：嵌入数组必须有上界。**16MB 不是建议，是硬墙**。GridFS 是给二进制大文件的，不是给"嵌入文档太大"的解药。

---

### 案例 ⑧ Schema 设计过度规范化 —— "查个用户主页要 8 次 round-trip"

#### 现象

新人按 SQL 思维设计了 MongoDB 模型：users / profiles / addresses / contacts / preferences / orders / order_items / payments 八张表。展示用户主页要：

```
1. db.users.findOne({_id: ...})
2. db.profiles.findOne({userId: ...})
3. db.addresses.find({userId: ...})
4. db.contacts.find({userId: ...})
5. db.preferences.findOne({userId: ...})
6. db.orders.find({userId: ...}).limit(10)
7. db.orderItems.find({orderId: {$in: [...]}})
8. db.payments.find({orderId: {$in: [...]}})
```

8 次往返，每次 2~5ms，主页打开要 30ms+。用 `$lookup` 后吞吐反而更差。

#### 根因

MongoDB 不是关系库的"无 schema 版本"。它的设计哲学是：**经常一起读的数据，存在一起**。

过度规范化（normalization）抹掉了文档模型的核心优势。`$lookup` 不是 SQL JOIN 的等价物 —— 它在分片场景下尤其昂贵（外表必须 unsharded 或本地）。

#### 修复

**重新建模**：

```javascript
// 用户文档（嵌入相对静态、一起展示的数据）
{
  _id: ObjectId(...),
  username: "alice",
  email: "alice@x.com",
  profile: {
    nickname: "Alice",
    avatar: "...",
    bio: "..."
  },
  addresses: [
    { type: "home", city: "Beijing", ... },
    { type: "work", city: "Shanghai", ... }
  ],
  preferences: { theme: "dark", lang: "zh" }
}
// 一次 findOne 拿全部主页核心信息

// 订单单独（独立生命周期，数量大）
{
  _id: ObjectId(...),
  userId: ObjectId(...),
  status: "paid",
  items: [
    { productId: ..., name: "...", price: 100, qty: 1 }    // 冗余 name
  ],
  payment: { method: "alipay", amount: 100, paidAt: ... }   // 嵌入支付信息
}
// 主页底部"最近订单"：单 collection 的 limit 10 即可
```

关键决策：
- **静态/一起读 → 嵌入**（profile、preferences、addresses）
- **数量有限的子集 → 嵌入数组**（一个订单的 items）
- **独立访问/数量大 → 独立 collection + 适度冗余**（orders）
- **多对多 → 引用 + 冗余关键字段**（订单里冗余 product name 避免每次 join）

#### 冗余字段的同步

冗余意味着写放大。需要一个机制保证一致：

**方案 A**：业务代码同步（最简单）。

```java
// 改商品名时同步所有订单的 item.name（异步任务）
public void updateProductName(String productId, String newName) {
  productRepo.update(productId, "name", newName);
  // 异步：updateMany orders.items.productId == productId
  asyncExec.submit(() -> {
    db.orders.updateMany(
      { "items.productId": productId },
      { $set: { "items.$[e].name": newName } },
      new UpdateOptions().arrayFilters(asList(eq("e.productId", productId)))
    );
  });
}
```

**方案 B**：用 Change Streams 监听源表，自动同步到目标表。

```javascript
db.products.watch().on("change", change => {
  if (change.operationType === "update" && change.updateDescription.updatedFields.name) {
    syncProductNameToOrders(change.documentKey._id, change.updateDescription.updatedFields.name);
  }
});
```

#### 验证

```javascript
// 用 explain 看主页查询的实际开销
db.users.find({...}).explain("executionStats")
// 期望：1 次 IXSCAN + 1 次 FETCH，executionTimeMillis < 5
```

> **口诀**：MongoDB 不是关系库。**冗余是优化，不是错误**。先想"用户怎么读"，再决定"怎么存"。

---

### 案例 ⑨ Spring Data MongoDB 的兼容性 —— "升级 5.0 后聚合查询全部报错"

#### 现象

把 MongoDB 服务端从 4.4 升到 5.0，没改任何代码。Spring Data MongoDB 3.4 的应用启动后，所有聚合查询报：

```
org.springframework.data.mongodb.UncategorizedMongoDbException:
  Command failed with error 168 (InvalidPipelineOperator):
  '$indexOfBytes' parameter 1 must evaluate to a string
```

#### 根因

MongoDB 5.0 对 `$indexOfBytes` 这类操作符的参数类型校验**变严格**了。Spring Data 在生成查询时把某些字段当成 String，但实际 BSON 类型是 BinData（旧数据），4.4 容忍，5.0 不容忍。

更深层：**Spring Data MongoDB 的版本兼容矩阵很重要，不是"老 driver 能连新 server"那么简单**。

| MongoDB Server | 推荐 Spring Data MongoDB | 推荐 Java Driver |
| --- | --- | --- |
| 4.4 | 3.x | 4.0+ |
| 5.0 | 3.4+ / 4.x | 4.3+ |
| 6.0 | 4.0+ | 4.7+ |
| 7.0 | 4.2+ | 4.10+ |

#### 修复

**升级 Spring Data MongoDB 版本**：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
<!-- 通过 Spring Boot 版本控制 -->
<!-- Spring Boot 3.1+ 带 Spring Data MongoDB 4.x -->
```

**清理脏数据**：找出所有 BinData 类型字段，转 String 或单独处理。

```javascript
db.collection.find({
  someField: { $type: "binData" }
}).count()
// 如果 > 0，先做数据类型修复迁移
```

**Spring Data 易踩的别的坑**：

1. **@Indexed 自动建索引在生产慢**：默认应用启动时同步建索引，大集合会卡几分钟。生产关掉：

```java
@Configuration
public class MongoConfig {
    @Bean
    public IndexResolver indexResolver(MongoMappingContext mappingContext) {
        return new MongoPersistentEntityIndexResolver(mappingContext);
    }

    // 关掉自动索引创建
    @Bean
    public MongoCustomConversions customConversions() {
        return new MongoCustomConversions(Collections.emptyList());
    }
}

# application.yml
spring:
  data:
    mongodb:
      auto-index-creation: false   # ★ 生产必关
```

索引必须用 migration 脚本管理，跟 DB 一起 review。

2. **`MongoTemplate.find` 默认 readPref 是 primary**：要全局改 secondary：

```java
@Bean
public MongoClientSettingsBuilderCustomizer customizer() {
    return builder -> builder.readPreference(ReadPreference.secondaryPreferred());
}
```

3. **`@DBRef` 几乎是个反模式**：每次访问都触发 lazy load 一次额外查询，N+1 问题严重。**永远用手动引用 + 业务层加载**。

#### 验证

```bash
# 升级前在测试环境跑全量回归
mvn test -Dspring.profiles.active=mongo5

# 看 mongod 日志的 slow query 和 error
tail -f /var/log/mongodb/mongod.log | grep -E '"errCode"|"durationMillis":[5-9][0-9]{3}'
```

> **口诀**：MongoDB 服务端升级前必须升级 Spring Data MongoDB 和 Java Driver。**`auto-index-creation` 生产永远关闭**。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### Schema 设计层
- [ ] 嵌入数组**有上界**（comments、events、logs 这种永远用引用，不嵌入）
- [ ] 静态字段（profile、preferences）嵌入主文档
- [ ] 关键字段适度冗余，避免大量 `$lookup`
- [ ] 单文档大小预估 < 1MB（极限 16MB 是硬墙）
- [ ] 不滥用 `@DBRef`（Spring Data）

### 副本集层
- [ ] **永远 PSS（3 数据节点），永远不用 Arbiter**
- [ ] oplog 大小 ≥ 24 小时窗口（监控覆盖时间，不是空间）
- [ ] 跨机房成员对称分布（1-1-1 或 2-2-1）
- [ ] 用 `priority` 明确"优先 Primary"
- [ ] `electionTimeoutMillis` 默认 10s，按业务延迟容忍度调整

### 分片层
- [ ] Shard Key 上线前 review，**避免单调递增字段直接 ranged**
- [ ] 时序场景用 `{ tenantId: "hashed", ts: 1 }` 这类复合 key
- [ ] chunkSize 保持默认 128MB，除非有理由调整
- [ ] balancer 监控告警（chunk 分布偏差 > 20%）

### 写关注 / 读关注层
- [ ] **生产默认 `w: majority` + `j: true`**
- [ ] 关键路径明确 `readConcern: majority` 或 `linearizable`
- [ ] secondary 读必须配 `maxStalenessSeconds`
- [ ] 写后读同一数据 → primary 或 causal consistency session

### 索引层
- [ ] 单 collection 索引数 ≤ 5
- [ ] 用 `$indexStats` 定期清理 0 ops 的僵尸索引
- [ ] Compound 索引按 ESR（Equality, Sort, Range）顺序
- [ ] **`auto-index-creation` 生产关闭**（Spring Data 默认开）
- [ ] 大 collection 建索引必须用 `{ background: true }`（4.2+ 默认是）

### 事务层
- [ ] 优先嵌入避免事务
- [ ] 必须事务时**单分片优先**
- [ ] 事务体内禁止远程调用
- [ ] 事务超时 < 5s（默认 60s 偏长）

### 查询层
- [ ] `$group`/`$sort` 必须 `allowDiskUse: true`
- [ ] 大数据量聚合考虑 `$out` 物化或挪到 hidden secondary
- [ ] 高频写场景禁止用 TTL 索引（每秒过期 > 5000 用业务 job）

### 客户端层
- [ ] `retryWrites: true`（4.2+ 默认）
- [ ] `serverSelectionTimeout` 配 15~30s
- [ ] 连接池大小 = peak QPS × 平均延迟（典型 100~500）
- [ ] 驱动版本与 server 版本兼容矩阵（见案例 ⑨）

### 监控层
- [ ] `replSetGetStatus` 每节点 lag 监控（> 10s 告警）
- [ ] oplog 窗口监控（< 6 小时告警）
- [ ] WiredTiger cache 命中率（< 80% 警告）
- [ ] 慢查询日志（默认 100ms，调到 50ms）
- [ ] balancer 状态、chunk 分布偏差

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### MongoDB Atlas（云服务）
- 自动化的备份、监控、扩展。生产中比自建可观成本可能更低
- Atlas Search（基于 Lucene）、Atlas Triggers（serverless 函数）
- 区别：Atlas 默认 `w: majority` + 严格 schema 校验，自建经常被忽略

### Time Series Collections（5.0+）
- 内部用列存优化时序数据，写吞吐和压缩比对普通 collection 提升 3~5 倍
- 限制：不能 sharding（5.0），6.0+ 才能；不能 update/delete（只能 TTL）
- 适合：监控指标、IoT 数据、日志

### Vector Search（6.0+）
- 把 embedding 向量存进文档，用 $vectorSearch 做近邻检索
- 内部基于 HNSW（Hierarchical Navigable Small World）算法
- 与 RAG 应用结合，可以替代 Pinecone / Qdrant 这类专用向量库
- 限制：维度 ≤ 4096，单 vector 索引只能用一个

### 与 PostgreSQL JSONB 的对比
- PG JSONB 也支持文档查询、GIN 索引，事务上甚至比 MongoDB 更成熟
- 但 PG 不擅长水平分片（Citus 是扩展），副本集成熟度也不如 MongoDB
- 选型：**事务复杂、单机够用 → PG；横向扩展、文档为主 → Mongo**

### Change Streams + 实时同步
- 用 Change Streams 把 MongoDB 数据流到 Elasticsearch / ClickHouse / Kafka
- Debezium MongoDB Connector：开箱即用的 CDC，比自己写 watch 健壮
- 注意：oplog 必须配大，Change Streams 断开时间不能超 oplog 窗口

### Field Level Encryption（4.2+）
- 客户端加密某些字段（比如手机号、身份证），server 看不到明文
- Queryable Encryption（6.0+）：加密字段也能查询（等值），密码学黑魔法
- 合规场景（GDPR、医疗）的硬需求

### MongoDB 7+ 新特性
- **TimeSeries 改进**：支持 sharding、二级索引
- **Cluster-to-cluster sync**：内置的双集群同步工具
- **Queryable Encryption GA**
- **改进的 `$lookup`**：在分片场景下性能优化
- **Compound wildcard indexes**：`{ "user.$**": 1 }` 形式的通配符索引

---

## 7. 引用

- MongoDB 7.0 源码：[github.com/mongodb/mongo](https://github.com/mongodb/mongo)
- WiredTiger 源码：[github.com/wiredtiger/wiredtiger](https://github.com/wiredtiger/wiredtiger)
- 官方手册：[docs.mongodb.com](https://docs.mongodb.com/manual/)
- *MongoDB: The Definitive Guide* (3rd Ed.) by Bradshaw, Brazil & Chodorow
- *Designing Data-Intensive Applications* by Martin Kleppmann —— 文档/列式/图存储的对比章节是必读
- MongoDB Server 设计文档：[github.com/mongodb/mongo/tree/master/src/mongo/db/repl](https://github.com/mongodb/mongo/tree/master/src/mongo/db/repl) 的 README
- Raft 论文：*In Search of an Understandable Consensus Algorithm* (Ongaro & Ousterhout, 2014)
- Aphyr 的 Jepsen 报告：[jepsen.io/analyses/mongodb-4.2.6](https://jepsen.io/analyses/mongodb-4.2.6) —— MongoDB 一致性的独立审计

---

> 配套：[MySQL MVCC 与锁](/article/db/mysql-mvcc) ｜ [Cassandra 深入](/article/db/cassandra) ｜ [Elasticsearch 深入](/article/db/elasticsearch)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你觉得这个深度对，告诉我下一篇该写哪个。
