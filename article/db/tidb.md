# TiDB 与 NewSQL 深入 —— 分布式事务、Multi-Raft 与 HTAP

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想快速复习 NewSQL 的"是什么"，看 [分库分表速记](/article/db/sharding) 那一节够了。
> 这里要回答的，是**业务从单机 MySQL 迁到 TiDB、半夜被热点 Region 告警叫起来**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. TiDB 号称"100% MySQL 兼容"。但你能立刻说出至少 5 处**不兼容**的地方吗？为什么自增 ID 在 TiDB 上不再连续？
2. TiDB 的乐观事务和悲观事务，**底层都是 Percolator 模型**，差别在哪一步？什么场景下乐观事务会"集体崩塌"？
3. TiKV 的 Multi-Raft 把数据切成 Region。一个 Region 默认 96MB，凭什么这个数？小了/大了各有什么坑？
4. TiFlash 是**Raft Learner**，意思是它参与同步但不投票。它怎么保证读到的列存数据"足够新"？读 TiFlash 时如何确认行存和列存数据一致？
5. PD（Placement Driver）是个**单点**吗？如果 PD Leader 挂了，集群还能跑多久？TSO（时间戳）是怎么从 PD 拿的，单点会不会成瓶颈？
6. Percolator 的两阶段：Prewrite 和 Commit。**Primary Key 在哪一步被特殊对待**？如果客户端在 Prewrite 完成后崩溃，数据怎么处理？
7. 给一个简单的 `SELECT name FROM users WHERE age > 30 ORDER BY id LIMIT 100` —— TiDB 怎么决定哪些算子下推到 TiKV / TiFlash，哪些在 TiDB Server 自己跑？什么是 Coprocessor？
8. 同样是 NewSQL，TiDB 选 Percolator + Raft，CockroachDB 选 Raft + 分布式 SSI，YugabyteDB 选 Raft + RocksDB 双层，OceanBase 走 Paxos + LSM 单机分区。**一句话总结这四家最大的设计差别**。

如果有半数答不上，往下看。

---

## 1. 底层模型：四组件、Region、Percolator

绝大多数文章把 TiDB 讲成"分布式 MySQL"，然后画一张架构图就结束了。这层认知不够诊断生产问题。要看懂上面 8 个问题，必须下沉到 TiDB 的四个核心组件、Region 模型、Percolator 事务模型。

### 1.1 四个组件 —— 各管各的事

```
┌──────────────────────────────────────────────────────────┐
│                     业务/应用层 (MySQL Client)            │
└──────────────────────────────────────────────────────────┘
                ▲ MySQL Protocol (4000)
                │
┌───────────────┴──────────────────────────────────────────┐
│  TiDB Server  (无状态 SQL 层)                             │
│   - SQL 解析 / 优化 / 执行                                 │
│   - 事务两阶段协调                                         │
│   - 不存数据，可水平扩展                                    │
└───────────────┬──────────────────────────────────────────┘
                │ KV / Coprocessor RPC (gRPC, 20160)
       ┌────────┼─────────┐
       ▼        ▼         ▼
┌──────────┐ ┌──────┐ ┌──────────────────┐
│   TiKV   │ │  PD  │ │     TiFlash      │
│ 行存集群  │ │ 元   │ │  列存集群         │
│ Raft Group│ │ 数据 │ │  Raft Learner    │
│ 多副本    │ │ 调度 │ │  (不投票)        │
└──────────┘ └──────┘ └──────────────────┘
```

| 组件 | 进程数典型值 | 角色 | 数据 | 扩展方式 |
| --- | --- | --- | --- | --- |
| **TiDB Server** | 2~10+ | SQL 层、事务协调 | 无状态 | 加机器即扩展 |
| **TiKV** | 3~100+ | 行存、Raft 复制 | 实际数据（行格式） | 加机器自动迁 Region |
| **PD** | 3 / 5 | 元数据、调度、TSO | Region 元信息 | Raft（奇数节点） |
| **TiFlash** | 0~多 | 列存、HTAP 加速 | 列格式副本 | 按表/库选择性同步 |

**反直觉的设计**：TiDB Server 完全无状态。它崩了，开新的就行；TiKV 才是数据本体。这个分层让 TiDB 的"SQL 层水平扩展"和"存储层水平扩展"可以独立发生。

### 1.2 Region —— 数据的物理切片

TiKV 把整个 KV 空间按 Key 排序，然后切成一段段，每段叫一个 **Region**：

```
全局 Key 空间（按字节序排）：
[''         , 'user_001'): Region 1
['user_001' , 'user_500'): Region 2
['user_500' , 'user_900'): Region 3
['user_900' , +∞       ): Region 4
```

每个 Region 默认 96MB（`region-split-size`），有 5 项关键属性：
- **范围 [start_key, end_key)**：左闭右开
- **3 副本**（默认）：分散在不同 TiKV 节点上
- **Raft Group**：3 副本就是一个 Raft Group，1 主 2 从
- **唯一 Region ID**：全局递增
- **版本号 (epoch)**：分裂/迁移时递增

为什么是 96MB？太小：Region 数量爆炸（百万级），PD 调度压力大、Raft 心跳多；太大：迁移成本高、热点难分散、副本 catch-up 慢。**96MB 是经验值**，PD 也允许调（但不要轻易动）。

> **口诀**：表→行→KV→Region→Raft Group→TiKV 节点。这条链路上每一层都有故障域，每一层都对应一种生产事故。

### 1.3 Multi-Raft —— 不是一个 Raft，是几万个

经典 Raft 是"整个集群一个 Raft Group"。这种设计在数据量大时性能极差 —— 所有写都要过同一个 Leader。

TiKV 的设计：**每个 Region 是一个独立的 Raft Group**。一个 100GB 表会切出约 1000 个 Region，对应 1000 个 Raft Group：

```
TiKV-Node-A:   [R1-Leader]  [R2-Follower] [R3-Follower] [R4-Leader] ...
TiKV-Node-B:   [R1-Follower][R2-Leader]   [R3-Follower] [R4-Follower] ...
TiKV-Node-C:   [R1-Follower][R2-Follower] [R3-Leader]   [R4-Follower] ...
```

观察：
- 每个 Region 的 Leader 可以在不同节点 → **写流量天然分散**
- PD 主动做 `balance-leader` 调度 → 让 Leader 数量在节点间均衡
- 一个节点挂了，那个节点上每个 Region 各自重新选主，**故障粒度是 Region 而非整个节点**

这是 TiKV 能做到"线性扩展写性能"的核心。

### 1.4 Region 分裂与合并

Region 不是固定的，会动态分裂/合并：

| 触发 | 阈值 | 动作 |
| --- | --- | --- |
| **Size Split** | Region > 144MB（`region-max-size`） | 按 key 中点分裂成两个 |
| **Keys Split** | key 数 > 1.44M（`region-max-keys`） | 同上 |
| **Load Split** | QPS > 阈值 + 流量倾斜 | 在热点位置分裂 |
| **Region Merge** | 相邻两个 Region 都 < 20MB | 合并成一个 |

分裂的关键路径：
1. Leader 检测到 size 超阈值
2. 选一个分裂点 key（默认按 key 中点）
3. 通过 Raft 提交一条 `Split` 日志
4. 所有副本 apply 后，原 Region 变成两个 Region，各自继续 Raft

**注意**：分裂后两个新 Region 仍在同一组节点上。需要等 PD 后续调度 `balance-region` 才会分散。这是热点 Region 即使分裂了**短期内仍可能压在同一节点**的根。

### 1.5 Percolator —— TiDB 事务的祖宗

TiDB 的分布式事务模型来自 Google Percolator 论文（2010）。核心思想：**用一张全局排序的 KV 表 + 两阶段提交，实现 SI（Snapshot Isolation）**。

每行数据在 TiKV 里实际上有三个 CF（Column Family）：

```
default_cf:   key + start_ts → value           (实际数据)
lock_cf:      key             → lock info       (Prewrite 时加的锁)
write_cf:     key + commit_ts → write info     (Commit 时加的指针，指向 default_cf)
```

写一条 `UPDATE t SET v = 100 WHERE id = 1` 的物理流程：

```
事务 start_ts = 50, commit_ts = 60

阶段 1 (Prewrite)：
  - 选第一个 key 作为 Primary Key (PK)
  - 给 PK 写 lock_cf:  {pk_lock = "PRIMARY", start_ts=50}
  - 给 default_cf 写: {key=id=1, ts=50, value=100}
  - 给其他 secondary keys 写 lock_cf: {lock = pointer→PK, start_ts=50}

阶段 2 (Commit)：
  - 先 commit Primary Key:
      write_cf 写 {key, commit_ts=60, info={start_ts=50, type=PUT}}
      lock_cf 删除 PK 的锁
  - PK commit 成功 → 整个事务"逻辑上"已提交
  - 异步 commit secondary keys（同样写 write_cf + 删 lock_cf）
```

**关键认知**：**Primary Key 的 commit 是事务的"原子点"**。
- PK commit 之前事务挂掉 → 事务未提交，所有 lock 视为"待回滚"
- PK commit 之后事务挂掉 → 事务已提交，残留的 secondary lock 视为"待 cleanup"

这就是为什么后面 TiKV 的 `BackoffAndMaybeCleanLocks` 路径能确定一个事务到底是应该 rollback 还是 commit —— **去看 PK 的 write_cf**。

### 1.6 乐观事务 vs 悲观事务

Percolator **原版只支持乐观事务**：所有锁在 Commit 时（Prewrite 阶段）才加。

```
乐观事务流程：
  1. begin → 拿 start_ts
  2. 业务读写（在 TiDB Server 内存里 buffer）
  3. commit → Prewrite 加锁 → 加锁失败立即报错
```

问题：高冲突下吞吐崩塌（见案例 ①）。

TiDB 4.0+ 增加了**悲观事务**：DML 时即加锁。

```
悲观事务流程：
  1. begin → 拿 start_ts
  2. UPDATE/SELECT FOR UPDATE → 立即向 TiKV 加 pessimistic lock
  3. commit → Prewrite（已经持锁了，Prewrite 把 pessimistic lock 升级成 prewrite lock）→ Commit
```

| 维度 | 乐观事务 | 悲观事务 |
| --- | --- | --- |
| 加锁时机 | Commit 时 | DML 时 |
| 冲突表现 | Commit 失败 + 客户端重试 | DML 时阻塞或快速失败 |
| 适用场景 | 冲突低、批处理 | 冲突高、迁移自 MySQL |
| 与 MySQL 行为 | 不一致（MySQL 用悲观） | **基本一致** |
| 性能（无冲突） | 略好 | 略差（多一次 RPC 加锁） |
| 性能（高冲突） | 灾难 | 平稳 |

TiDB 5.0 之后，**默认是悲观事务**。`tidb_txn_mode = 'pessimistic' / 'optimistic'` 可切。

### 1.7 TSO —— 全局时间戳

TiDB 用全序时间戳（Timestamp Oracle, TSO）来排所有事务的可见性顺序。**TSO 由 PD Leader 唯一分配**：

```
TSO 格式（int64）：
  | 高 46 位：物理时间 (ms)  | 低 18 位：逻辑计数器 |
```

**单点会不会成瓶颈**？看数字：每个事务**两次** TSO 申请（start_ts + commit_ts），单 PD 节点能提供大约 **百万级 TSO/s**。秘诀：
- TiDB Server 端**批量 (batch) 拉取 TSO**：一次 RPC 拿 N 个
- PD 内存里维护 `tso_counter`，分配只是 atomic add，无锁竞争

但 PD 单点仍是问题：**PD Leader 切换瞬间 TSO 不可用**，所有事务卡几百毫秒到几秒（见案例 ④）。

### 1.8 TiFlash —— 列存 + HTAP

TiFlash 不是"另一个数据库"，而是 **TiKV 的列存副本**。架构关系：

```
某个表的某个 Region：
  TiKV-Node-A:  Leader (行存)
  TiKV-Node-B:  Follower (行存)
  TiKV-Node-C:  Follower (行存)
  TiFlash-D:    Learner (列存) ← 不参与投票，只接收 Raft 日志
```

特点：
- **同一份数据**通过 Raft 同步，行存列存"逻辑一致"
- TiFlash 节点是 Raft Learner，**不影响 OLTP 写延迟**
- 查询时优化器自动选行存 / 列存（基于代价）

TiFlash 读一致性的保证：
- 收到查询时，TiFlash 知道当前事务的 `read_ts`
- 检查本地数据的 apply 进度，如果 apply_ts < read_ts → **触发 wait_index 等到追上**
- 如果落后太多 → 报错"region unavailable"，TiDB 回退到 TiKV 行存读

```sql
-- 强制走 TiFlash
SELECT /*+ READ_FROM_STORAGE(tiflash[t]) */ * FROM t WHERE ...;

-- 强制走 TiKV
SELECT /*+ READ_FROM_STORAGE(tikv[t]) */ * FROM t WHERE ...;
```

> **口诀**：TiFlash 是"行存 + 列存共用一份逻辑数据"。HTAP 不是"一个引擎搞定一切"，而是"两个引擎按 workload 自动切"。

### 1.9 PD —— 调度大脑

PD 的职责：
- **元数据**：哪个 Region 在哪些节点上、谁是 Leader
- **TSO 分配**：全局唯一时间戳
- **调度**：balance-region / balance-leader / hot-region-scheduler
- **副本管理**：副本数不足时补副本，节点失联时迁移

PD 自身用 etcd 风格的 Raft（基于 etcd 的 raft 库），3 节点或 5 节点。**只有 Leader 工作**，Follower 实时备份元信息。

调度的核心算子：
- **balance-region**：让每个节点的 Region 数量均衡
- **balance-leader**：让每个节点的 Leader 数量均衡
- **hot-region-scheduler**：检测到热点 Region 后主动迁移
- **evict-leader-scheduler**：把某节点的所有 Leader 赶走（运维用，比如要重启该节点）

> **口诀**：PD 是"集群医生"，不在数据路径上，但是稳定性命脉。**PD 一抖，全集群事务卡顿**。

---

## 2. 源码切片：七段关键路径

> 基于 TiDB 7.x / TiKV 7.x 源码。注释为本文添加。

### 2.1 TiDB SQL 优化器 —— 算子下推决策

```go
// pkg/planner/core/rule_predicate_push_down.go 简化
func (s *PPDSolver) optimize(ctx context.Context, lp LogicalPlan, opt *logicalOptimizeOp) (LogicalPlan, error) {
    _, p, err := lp.PredicatePushDown(nil, opt)
    return p, err
}

// 对每个算子：把能下推的过滤条件推到子节点
func (p *DataSource) PredicatePushDown(predicates []expression.Expression, opt *logicalOptimizeOp) ([]expression.Expression, LogicalPlan) {
    predicates = expression.PropagateConstant(p.SCtx(), predicates)
    // ★ 关键：把过滤条件标到 DataSource，后续生成 Coprocessor 请求时带上
    p.allConds = append(p.allConds, predicates...)
    p.pushedDownConds, predicates = expression.PushDownExprs(
        p.SCtx().GetExprCtx().GetEvalCtx(),
        predicates,
        p.SCtx().GetClient(),     // ★ Client 决定能下推到哪种 store (TiKV/TiFlash)
        kv.UnSpecified,
    )
    return predicates, p
}
```

**逐段点透**：

- **PushDownExprs 的关键参数 `Client`**：每种 store 类型支持的算子不同。TiKV 支持简单过滤、聚合、TopN；TiFlash 支持复杂表达式、Join。
- **`pushedDownConds` vs 剩余 `predicates`**：能下推的成 Coprocessor 请求；不能下推的留在 TiDB Server 内存里执行。
- 这就是 `EXPLAIN` 里看到 `cop[tikv]` / `cop[tiflash]` / `root` 三种执行位置的根。

下推失败的常见原因：
- 函数 TiKV / TiFlash 不支持（如某些字符串函数）
- 表达式涉及"不确定"调用（NOW()、UUID()）
- 数据类型转换不支持下推

```sql
-- 看下推情况
EXPLAIN SELECT count(*) FROM t WHERE substr(name, 1, 3) = 'abc';
-- 关注：filter 在 root 还是 cop[tikv]
```

### 2.2 TiKV Raft 选举 —— 不是教科书的 Raft

```rust
// components/raftstore/src/store/peer.rs 简化
impl Peer {
    fn check_election(&mut self) -> bool {
        // ① 选举超时（默认 election_timeout_ticks = 50, tick = 100ms = 5s）
        if self.election_elapsed >= self.election_timeout {
            // ② Pre-Vote：先问问大家"我能不能赢"，避免无谓打扰
            if !self.pre_vote_started {
                self.start_pre_vote();
                return true;
            }
            // ③ Pre-Vote 通过 → 真正发起 Vote
            self.start_election();
        }
        false
    }

    fn start_pre_vote(&mut self) {
        // 发 MsgPreVote，term 字段是 current_term + 1，但不真改 term
        let msg = MsgPreVote { term: self.term + 1, ... };
        for peer in self.peers() {
            self.send(peer, msg.clone());
        }
    }
}
```

**逐段点透**：

- **Pre-Vote 是 TiKV / etcd 的标配优化**，不在原 Raft 论文里。它解决的问题是：网络分区里的"少数派"节点不停发起 Vote 把整个集群的 term 往上推，导致不必要的重选举。Pre-Vote 不真改 term，只是问问"如果我发起选举，你们会投我吗"。
- **每个 Region 独立的 election_timeout**：成千上万个 Region 同时跑 election 心跳。TiKV 用 batch tick（合并心跳消息）避免 RPC 风暴。
- **Region 心跳 vs Store 心跳**：Region 心跳传给 PD 是为了让 PD 知道 Leader / size / 流量；Store 心跳传给 PD 是节点级别（容量、CPU）。

### 2.3 Region Split 的 Raft 提交

```rust
// components/raftstore/src/store/fsm/peer.rs 简化
fn on_split_region_check_tick(&mut self) {
    // ① 周期检查 Region 大小
    let approximate_size = self.fsm.peer.approximate_size;
    if approximate_size >= self.cfg.region_max_size {
        // ② 异步选分裂点（按 key 中点 / 流量热点）
        let task = SplitCheckTask::Split { region: self.region().clone() };
        self.sched_split_check(task);
    }
}

fn on_prepare_split_region(&mut self, split_keys: Vec<Vec<u8>>) {
    // ③ 把 split 操作包成一条 Raft 日志（AdminCmd::BatchSplit）
    let req = new_batch_split_request(split_keys);
    self.propose_raft_command(req, Callback::None);
    // ④ 这条日志被 commit 后，所有副本同步执行 split
}
```

**逐段点透**：

- **分裂是通过 Raft 提交的**：保证原子性。要么所有副本都看见新的两个 Region，要么都没看见。绝不会出现"主分裂了，从还没分"的状态。
- **分裂期间 Region 不可写**：从 propose split 到 apply split 完成的几十毫秒，所有写到该 Region 的请求会被 stalled。这是为什么大量并发 split 时**写延迟会有毛刺**。
- **split 后两个新 Region 仍在同一组节点上**：要等 PD 后续 `operator-balance-region` 才迁。**短时间内热点没有分散**。

### 2.4 Percolator Prewrite —— 选 PK + 加锁

```rust
// components/txn/src/commands/prewrite.rs 简化
impl Command for Prewrite {
    fn process_write(&self, snapshot: Snapshot, context: WriteContext) -> Result<WriteResult> {
        let mut txn = MvccTxn::new(self.start_ts, ...);
        let mut reader = SnapshotReader::new(self.start_ts, snapshot);

        for mutation in self.mutations {
            // ★ Primary Key：第一个 key（由 TiDB Server 选定）
            let is_pessimistic = self.is_pessimistic_lock.contains(&mutation.key);
            let prewrite_result = prewrite(
                &mut txn,
                &mut reader,
                mutation,
                &self.primary,            // ★ secondary lock 都指向 primary
                is_pessimistic,
                self.lock_ttl,
                ...
            )?;
        }

        // 把改动写入 lock_cf 和 default_cf
        let to_be_write = WriteData::from_modifies(txn.into_modifies());
        Ok(WriteResult { to_be_write, ... })
    }
}

fn prewrite(txn, reader, mutation, primary, ...) -> Result<...> {
    // ① 检查写写冲突：是否有更晚的提交版本
    let prev_write = reader.seek_write(&mutation.key, MAX_TS)?;
    if let Some((commit_ts, _)) = prev_write {
        if commit_ts > txn.start_ts {
            return Err(Error::WriteConflict { ... });   // ★ 乐观事务的"冲突崩塌"源头
        }
    }
    // ② 检查锁冲突：是否被另一个事务锁住
    if let Some(lock) = reader.load_lock(&mutation.key)? {
        if lock.ts != txn.start_ts {
            return Err(Error::KeyIsLocked(lock.into_lock_info()));
        }
    }
    // ③ 加自己的 lock
    txn.put_lock(mutation.key, Lock { primary, start_ts: txn.start_ts, ... });
    txn.put_value(mutation.key, mutation.value);
    Ok(())
}
```

**逐段点透**：

- **WriteConflict 的判定**：找当前 key 的最新已提交版本，若其 commit_ts 大于本事务的 start_ts，说明在我开始之后有人改过 → 写写冲突。乐观事务在 Prewrite 阶段才检查；悲观事务在加 pessimistic lock 时就检查。
- **锁全部指向 PK**：secondary key 的 lock 里只存"PK 在哪"。后续清理只需要看 PK 的状态。
- **lock_ttl**：锁有 TTL（默认 3 秒，根据事务大小动态调整）。TTL 过了别的事务可以来 cleanup。这是事务客户端崩了之后锁能自动释放的机制。

### 2.5 Percolator Commit —— PK 优先

```rust
// components/txn/src/commands/commit.rs 简化
impl Command for Commit {
    fn process_write(&self, snapshot: Snapshot, ...) -> Result<WriteResult> {
        let mut txn = MvccTxn::new(self.start_ts, ...);
        for key in &self.keys {
            // ① 找到 lock_cf 里我们 prewrite 时加的锁
            let lock = reader.load_lock(key)?;
            match lock {
                Some(lock) if lock.ts == self.start_ts => {
                    // ② 写 write_cf 标记 commit
                    txn.put_write(key, WriteType::Put, self.commit_ts, lock.short_value);
                    // ③ 删除 lock_cf
                    txn.delete_lock(key);
                }
                _ => {
                    // 锁不见了？可能已经被别的事务 cleanup 了
                    // 看 write_cf 是否已经有 commit 记录
                    let write = reader.get_txn_commit_record(key)?;
                    match write {
                        Some(WriteType::Rollback) => return Err(Error::TxnLockNotFound),
                        Some(_) => continue,    // 已提交，幂等
                        None => return Err(Error::TxnLockNotFound),
                    }
                }
            }
        }
        Ok(WriteResult { ... })
    }
}
```

**逐段点透**：

- **PK 必须先 commit**：TiDB Server 端的代码强制顺序 —— 先 commit PK，等返回成功，再异步 commit secondary。
- **PK commit 成功 = 事务已提交**：从这一刻起，无论 secondary commit 是否成功，事务"逻辑上"都成功。secondary 即使 RPC 失败、节点挂了，未来读到的事务会发现 PK 已 commit，自己帮忙完成 secondary commit（叫做 **resolve lock**）。
- **TxnLockNotFound**：锁被别人清掉了 —— 通常是因为事务超过 TTL，被读取者 cleanup 成了 Rollback。这时候本事务必须报错（不能再继续 commit，否则违反原子性）。

### 2.6 PD 调度器 —— balance-region

```go
// server/schedulers/balance_region.go 简化
func (s *balanceRegionScheduler) Schedule(cluster sche.SchedulerCluster, dryRun bool) ([]*operator.Operator, []plan.Plan) {
    stores := cluster.GetStores()
    // ① 按"region 数量 / 容量"排序，找最空和最满的两端
    sort.Slice(stores, func(i, j int) bool {
        return stores[i].RegionScore() > stores[j].RegionScore()
    })
    
    for _, source := range stores {                 // 从最满的开始
        for _, target := range stores[len(stores)-1:] {   // 找最空的
            if source.GetID() == target.GetID() { continue }
            
            // ② 在 source 上挑一个 Region 来迁
            region := s.pickRegionFromStore(cluster, source)
            if region == nil { continue }
            
            // ③ 评估：迁完之后两端是否更均衡？
            if !s.shouldBalance(source, target, region) { continue }
            
            // ④ 生成 Operator：在 target 上加副本 → 等 catch up → 删 source 的副本
            op := operator.CreateMovePeerOperator(...)
            return []*operator.Operator{op}, nil
        }
    }
    return nil, nil
}
```

**逐段点透**：

- **RegionScore**：考虑 Region 数 + 数据量 + 节点容量，综合打分。不是简单"按 Region 数均衡"。
- **MovePeer 是三步**：Add Learner → 等 Learner catch up → Remove Old Peer。期间副本数会临时超过配置的 N，等 catch up 完成才"减回去"。这是为什么 balance 期间集群对磁盘的瞬时占用会涨。
- **节流**：每个 store 同时进行的 operator 数有限（`store-balance-rate`），避免 balance 风暴拖垮集群。

### 2.7 TSO 批量分配

```go
// pkg/tso/global_allocator.go 简化
func (gta *GlobalTSOAllocator) GenerateTSO(ctx context.Context, count uint32) (pdpb.Timestamp, error) {
    // ① 拿当前物理时间
    now := time.Now()
    
    // ② 内存里的 logical counter
    gta.timestampOracle.tsoMux.Lock()
    defer gta.timestampOracle.tsoMux.Unlock()
    
    physical := gta.timestampOracle.tsoMux.physical
    if physical == typeutil.ZeroTime {
        return ts, errs.ErrGenerateTimestamp
    }
    
    // ③ logical 累加
    gta.timestampOracle.tsoMux.logical += int64(count)
    if gta.timestampOracle.tsoMux.logical >= maxLogical {
        // logical 用完，物理时间往前推一格
        physical = physical.Add(time.Millisecond)
        gta.timestampOracle.tsoMux.physical = physical
        gta.timestampOracle.tsoMux.logical = int64(count)
    }
    
    // ④ 持久化到 etcd（批量、异步）—— 防止 PD 重启后时间倒退
    if needPersist {
        gta.timestampOracle.SaveTimestamp(physical.Add(updateTimestampGuard))
    }
    
    return pdpb.Timestamp{Physical: physical.UnixNano() / 1e6, Logical: gta.timestampOracle.tsoMux.logical}, nil
}
```

**逐段点透**：

- **批量分配**：`count` 通常是几十到几百，TiDB Server 一次拿一批回去，本地分配给事务。这把 TSO 的 RPC 压力降低 N 倍。
- **logical 上限 (1<<18 = 262144)**：单个物理毫秒最多分配 26 万 TSO。超了 → 物理时间"借"一个毫秒往前。
- **持久化保护**：PD Leader 切换时，新 Leader 必须从 etcd 读到上一个 Leader 持久化的最大物理时间，**只能从那个时间往后分配**，绝不能回退。所以 PD 切换时会有几百毫秒"等持久化"的卡顿。

---

## 3. 实测数据：TiDB 不是万能的

### 3.1 OLTP 性能 vs MySQL 单机

sysbench `oltp_read_write` 测试，3 节点 TiDB（每节点 16C/64G/NVMe） vs 单机 MySQL 8.0（同规格）：

| 配置 | 4 线程 TPS | 64 线程 TPS | 256 线程 TPS | P99 (ms) |
| --- | --- | --- | --- | --- |
| **MySQL 8.0** | 4,800 | 28,000 | 32,000 | 18 |
| **TiDB 3 节点** | 3,200 | 22,000 | 38,000 | 35 |
| **TiDB 6 节点** | 3,200 | 26,000 | 65,000 | 28 |

观察：
- **低并发下 TiDB 比 MySQL 慢**：因为多了网络往返（TiDB Server → TiKV）。一个简单查询可能要 2~3 次 RPC。
- **高并发下 TiDB 反超**：单机 MySQL 受限于单机 IO 和 CPU；TiDB 可以横向扩。
- **延迟代价是固定的**：TiDB 的 P99 比 MySQL 高 50~100%，这是分布式的"网络税"。

> **口诀**：单机能跑得动的业务别迁 TiDB。**单机 MySQL 在数据量 < 1TB、QPS < 5万时几乎总是更快、更便宜、更省心**。TiDB 的甜区是"单机扛不住"。

### 3.2 TiKV 横向扩展的线性度

固定负载（256 线程 sysbench），增加 TiKV 节点数：

| TiKV 节点数 | TPS | 单节点 TPS | 线性度 |
| --- | --- | --- | --- |
| 3 | 38,000 | 12,667 | 100% (基准) |
| 6 | 65,000 | 10,833 | 86% |
| 12 | 110,000 | 9,167 | 72% |
| 24 | 175,000 | 7,292 | 58% |

线性度逐步下降。原因：
- TiDB Server 也成为瓶颈，需要同步加
- PD 调度压力增大
- 网络带宽跨机柜不足

实践：**3 → 6 节点扩容收益最大**，再往上要同步评估 SQL 层和 PD。

### 3.3 乐观 vs 悲观事务在不同冲突率下

构造 `UPDATE t SET v = v + 1 WHERE id = ?`，控制 id 的分布，模拟不同冲突率：

| 冲突率 | 乐观 TPS | 悲观 TPS | 乐观失败率 |
| --- | --- | --- | --- |
| 0%（每个 id 不同） | 28,000 | 25,000 | 0% |
| 5%（少量热点） | 22,000 | 23,000 | 5% |
| 30%（明显热点） | 8,000 | 18,000 | 32% |
| 80%（极度热点） | **300** | **6,000** | 85% |

**乐观事务在高冲突下吞吐崩塌**：原因在 Prewrite 阶段大量 WriteConflict，客户端反复重试，整个系统"做无用功"。

**悲观事务**在高冲突下虽然慢，但有锁排队 → 串行化执行 → 不会浪费 CPU。

> **口诀**：迁移自 MySQL → 一定要用悲观事务（默认）；纯批处理 + 无热点 → 可以用乐观事务。**敢用乐观事务的业务必须有"冲突重试到上限"的兜底**。

### 3.4 TiFlash HTAP 查询的延迟

某 5000 万行表，做 `SELECT count(*), sum(amt) FROM orders WHERE date BETWEEN ... GROUP BY region`：

| 引擎 | 耗时 |
| --- | --- |
| TiKV（行存）| 18.2 秒 |
| TiFlash（列存） | **0.42 秒** |
| TiFlash + MPP（多节点分布式） | **0.18 秒** |

列存的优势在 OLAP 查询上是数量级的。**MPP（Massively Parallel Processing）模式**进一步加速 —— Join、聚合在 TiFlash 节点之间分布式执行，不用全部拉回 TiDB Server。

### 3.5 跨地域部署的延迟代价

3 副本分布在 3 个机房（同城跨机房 RTT ~2ms）：

| 部署 | 写延迟 P99 | 读延迟 P99 |
| --- | --- | --- |
| 单机房 3 副本 | 8ms | 3ms |
| 同城三机房 3 副本 | **18ms** | 4ms |
| 两地三中心（5 副本） | **45ms** | 5ms |

写慢的原因：Raft 写要 majority quorum 确认（3 副本要 2 票，5 副本要 3 票）。跨机房的网络 RTT 直接进入写延迟。

**Follower Read** 可以缓解读延迟在跨地域下的问题，但只对一致性要求弱的场景（默认仍读 Leader）。

> **口诀**：**TiDB 多机房部署的写延迟下限 = max(机房 RTT) × 2**。跨城 30ms 网络的话，写就是几十 ms 起。这种场景必须先评估业务能不能接受。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 乐观事务高冲突崩塌 —— "灰度切流后 TPS 掉到 100"

#### 现象

业务从 MySQL 迁到 TiDB，灰度环境 100% 流量切过来，**TPS 从原来的 8000 掉到 100**。`tidb.log` 里大量：

```
[WARN] [conn.go:899] [PrewriteWriteConflict] [conn=12345] ...
[INFO] [retry.go:114] [RetryableWithBackoff] retries=10 reason="write conflict"
```

业务的关键 SQL：

```sql
UPDATE inventory SET stock = stock - 1 WHERE sku_id = ?;
```

灰度时 sku_id 集中在几个爆品上 → 极度冲突。

#### 根因

业务 TiDB 集群当时配置：

```sql
SET GLOBAL tidb_txn_mode = 'optimistic';
```

乐观事务的逻辑：业务发出 UPDATE → TiDB Server 在内存里 buffer → COMMIT 时才向 TiKV Prewrite。多个事务同时改同一 sku_id 时，**所有事务都通过了 buffer 阶段，全部进 Prewrite**。Prewrite 阶段 WriteConflict 检查（见 2.4）大量失败 → 客户端重试 → 又一波集体冲突 → 雪崩。

监控 `tikv_scheduler_command_duration_seconds_bucket{type="prewrite"}` P99 飙到 5s。

#### 修复

**立即**：切换到悲观事务（默认）。

```sql
SET GLOBAL tidb_txn_mode = 'pessimistic';
-- 已建立的连接需要重连或 SET SESSION
```

悲观事务在 UPDATE 时即向 TiKV 加 pessimistic lock，**冲突事务在 UPDATE 时排队等锁**，不会全部跑到 Prewrite 才发现冲突。

**长效**：业务侧分热点。

```sql
-- 把单个 sku 的库存拆成 N 行
CREATE TABLE inventory_sharded (
    sku_id BIGINT,
    shard_id INT,
    stock INT,
    PRIMARY KEY (sku_id, shard_id)
);
-- 减库存时随机选一个 shard
UPDATE inventory_sharded SET stock = stock - 1
WHERE sku_id = ? AND shard_id = FLOOR(RAND() * 16);
```

代价：查总库存要 SUM。

**进一步**：评估业务是否要 RC 隔离。TiDB 5.0+ 支持 `tidb_rc_read_check_ts`，RC 模式下读取性能更好（少一次 latch）。

#### 验证

```sql
-- 观察事务模式
SELECT @@tidb_txn_mode;

-- 观察冲突
SELECT * FROM information_schema.cluster_processlist WHERE STATE LIKE '%lock%';

-- Grafana 面板：tikv-details → Scheduler → Lock Wait 类型
```

> **口诀**：**乐观事务 + 高冲突 = 雪崩**。从 MySQL 迁来的业务永远先用悲观事务。乐观事务只给"明确无冲突"的场景。

---

### 案例 ② 单 Region 热点 —— "刚上线就被打挂一个 TiKV"

#### 现象

新业务上线，某张表用自增 ID 作主键。流量进来后，监控显示 **某个 TiKV 节点 CPU 100%**，其他节点 CPU 30%。所有写延迟 P99 从 5ms 飙到 500ms。

```
Grafana → tikv-details → Hot Region:
  Region 1023: 12,450 writes/s    ← 比第二名高 50 倍
  Region 1024: 250 writes/s
  ...
```

#### 根因

InnoDB 风格的自增主键 + TiKV 的"全局 key 排序"是地狱组合：

```
业务写入：
  id=1001, 1002, 1003, 1004 ... 全部写到末尾
  
TiKV 视角：
  所有新行都进同一个 Region（最后那个 Region）
  这个 Region 的 Leader 在某一个节点上
  → 单点写热点
```

PD 检测到 Region 大了会分裂（splittin），但分裂出的两个 Region **仍然在同一组节点上**，要等 `balance-region` 调度 → 短期内热点不会自然消除。

#### 修复

**方案 A**（推荐）：用 `AUTO_RANDOM` 替代 `AUTO_INCREMENT`。

```sql
-- ❌ 单点热点
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    ...
);

-- ✅ 散列分布
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_RANDOM(5),    -- 高 5 位随机
    ...
);
```

`AUTO_RANDOM(5)` 把 ID 的高 5 位变成随机，**新插入的行散布在不同 Region**。

**方案 B**：手动预切分 Region（pre-split）。

```sql
-- 建表时按 hash 切 16 个 Region
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_RANDOM(5),
    ...
) SHARD_ROW_ID_BITS = 4 PRE_SPLIT_REGIONS = 4;
-- PRE_SPLIT_REGIONS = 4 表示预先切 2^4 = 16 个 Region
```

**方案 C**：改业务键设计。把"递增的业务 ID"散列：

```sql
-- 不要用：CREATE TABLE log (ts TIMESTAMP, ... PRIMARY KEY(ts))
-- 改：把 ts 哈希成前缀
CREATE TABLE log (
    bucket TINYINT,
    ts TIMESTAMP,
    ...
    PRIMARY KEY (bucket, ts)
);
INSERT INTO log VALUES (CRC32(client_ip) % 16, NOW(), ...);
```

**临时缓解**：手动 split + scatter。

```sql
SPLIT TABLE orders BETWEEN (0) AND (9223372036854775807) REGIONS 16;
-- scatter 让 PD 把新 Region 分散到不同节点
```

#### 验证

```sql
-- 看每个 Region 的写入分布
SELECT REGION_ID, WRITTEN_BYTES, WRITTEN_KEYS
FROM INFORMATION_SCHEMA.TIKV_REGION_STATUS
WHERE TABLE_NAME = 'orders'
ORDER BY WRITTEN_BYTES DESC
LIMIT 10;

-- 修复前：TOP 1 的 WRITTEN_BYTES 远超其他
-- 修复后：分布均匀
```

> **口诀**：TiDB 的主键设计要"散"，不要"递增"。**`AUTO_RANDOM` + `PRE_SPLIT_REGIONS` 是新表的默认套路**。

---

### 案例 ③ TiKV 磁盘满集群不可写 —— "一个节点拖死整个集群"

#### 现象

凌晨 3 点告警：所有写请求超时。`tidb.log`：

```
[ERROR] [region_request.go:455] [send_request_failed] err="server is busy: store has no space left"
```

`pd-ctl store` 看：

```
{
  "store_id": 5,
  "available": "0 B",         ← ★ 满了
  "capacity": "1 TB",
  "state": "Up",
  "region_count": 12340
}
```

#### 根因

TiKV 的"安全水位"机制：磁盘空间不足时进入 **only-read** 模式，**该节点上的所有 Region 不再接受写入**。问题是该节点上有 12340 个 Region，每个 Region 的 3 副本里有 1 副本在这里，**其他副本能不能写还要看那 2 副本是否在 quorum 内**。

更糟：PD 的副本调度（`replica-schedule-limit`）默认很慢，从满节点迁出 Region 速度很慢，**满节点状态下整个集群处于半瘫痪**。

#### 修复

**紧急止损**：把满节点设为 offline，强制迁数据。

```bash
# 1. 给该节点打"驱逐 leader"标签，避免新流量
tiup ctl pd scheduler add evict-leader-scheduler 5

# 2. 设置该 store 为 offline，PD 会启动副本迁移
tiup ctl pd store delete 5

# 3. 查看进度
tiup ctl pd store 5
# 看 region_count 应持续下降到 0
```

注意 `store delete` 不是真删，是触发"优雅下线"。下线完成后 `store remove-tombstone` 清理元数据。

**短期**：扩容磁盘 / 加 TiKV 节点。

```bash
# 扩容磁盘后重启 TiKV，自动恢复
systemctl restart tikv@20160

# 或者加新节点
tiup cluster scale-out <cluster> scale-out.yaml
```

**根治**：监控 + 自动告警。

```yaml
# Prometheus 告警规则
- alert: TiKVDiskAlmostFull
  expr: (node_filesystem_avail_bytes{mountpoint=~"/data.*"} / node_filesystem_size_bytes) < 0.20
  for: 5m
  annotations:
    summary: "TiKV node {{ $labels.instance }} disk usage > 80%"
```

设三个阈值：
- **70%**：警告，准备扩容
- **80%**：紧急，立刻扩
- **85%**：TiKV 自动开始拒绝部分写

#### 验证

```bash
# 集群健康检查
tiup ctl pd cluster
tiup ctl pd store

# 写请求应恢复正常
mysql -h tidb -e "INSERT INTO test VALUES (1)"
```

> **口诀**：TiKV 一个节点磁盘满 = 整集群写入退化。**磁盘水位监控比 CPU 监控更要命**。预留 30% 空间给 compaction + 副本迁移。

---

### 案例 ④ PD Leader 切换 TSO 抖动 —— "全集群 5 秒卡顿"

#### 现象

监控看到每天凌晨 4 点准时有 3~10 秒的全集群事务卡顿，应用方反馈 P99 飙到 5 秒。但 TiKV / TiDB 节点指标都正常。

```
tidb.log:
[WARN] [tso_dispatcher.go:288] [getTSAsync] err="rpc error: code = Unavailable desc = transport is closing"
```

#### 根因

PD 的 etcd lease（默认 3s）+ 凌晨 4 点的运维定时任务（备份 / 巡检）导致 PD Leader 偶发 lease 过期 → 重新选主。

PD Leader 切换的过程：
1. 旧 Leader lease 失效
2. 其他 PD 节点发起 election
3. 新 Leader 选出（几百 ms）
4. 新 Leader 从 etcd 读取上次持久化的最大 TSO，**强制等过那个时间点才能开始分配**（防回退）
5. 新 Leader 接客

**全过程 1~5 秒**。期间所有 TiDB Server 拿不到 TSO → 所有事务卡住。

更糟的情况：业务在凌晨有定时跑批，跑批和 PD 切换撞上 → 跑批的事务集体超时回滚。

#### 修复

**减少 PD 切换频次**：
- PD 节点专机部署，避免和 TiKV 抢 IO
- 调大 `tso-save-interval`（默认 3s）→ 适当调到 10s 减少持久化频率，但切换后等待时间也会变长（trade-off）
- PD 节点之间网络要稳，最好同机房同 VLAN

**TiDB Server 端增加重试**：

```sql
-- 增大 TSO RPC 超时
SET GLOBAL tidb_pd_server_request_timeout = '5s';

-- 业务连接的事务重试
-- TiDB 4.0+ 自动重试 PD 不可用的事务（在事务幂等的前提下）
SET GLOBAL tidb_disable_txn_auto_retry = 0;   -- 注意：仅适合幂等业务
```

**业务侧**：跑批避开运维窗口；高优先级在线业务用更短的 SQL 超时 + 重试。

**TiDB 6.5+ 的优化**：`pd_async_tso` 配合 batch 进一步降低 PD 单点压力，PD 切换的影响已减小到秒级。

#### 验证

```bash
# 主动 transfer leader 测试
pd-ctl member leader transfer pd-2

# 同时观察 TiDB 端事务延迟
# 应该看到几百毫秒的尖峰，但不应该是几秒
```

> **口诀**：PD 是单点 TSO 中心。**PD 切换 = 全集群事务卡顿**。运维窗口和业务高峰必须错开。

---

### 案例 ⑤ TiFlash 同步延迟 —— "HTAP 查到三天前的数据"

#### 现象

BI 报表用 TiFlash 跑，业务方反馈 **"昨天的数据看不到"**。但同样的查询走 TiKV 是有的。

```sql
-- 走 TiFlash
SELECT /*+ READ_FROM_STORAGE(tiflash[orders]) */ MAX(created_at) FROM orders;
-- 返回：2024-01-10 00:00:00

-- 走 TiKV
SELECT /*+ READ_FROM_STORAGE(tikv[orders]) */ MAX(created_at) FROM orders;
-- 返回：2024-01-13 23:59:59
```

#### 根因

TiFlash 是 Raft Learner，**异步**接收 Raft 日志。如果某个 Region 的 TiFlash 副本同步严重落后：

1. TiFlash 节点资源紧张（CPU 满、磁盘满）
2. 网络拥塞
3. Region 数量过多，TiFlash 单实例承载不住

落后到一定程度会被 PD 标记为 unavailable，但**已过 wait_index 超时的 query 仍然会回退到 TiKV**。如果业务用了 `READ_FROM_STORAGE(tiflash)` hint 强制走 TiFlash，会超时报错；如果优化器自选 TiFlash 但落后了，**会自动 fallback 到 TiKV**。

更微妙：TiFlash 默认 `tiflash_max_threads` 在大查询下会"被自己卡住"——查询占满线程，apply Raft 日志的线程拿不到 CPU → 越查越落后。

#### 修复

**诊断**：

```sql
-- 看 TiFlash 副本同步状态
SELECT * FROM information_schema.tiflash_replica
WHERE TABLE_NAME = 'orders';

-- AVAILABLE = 0 表示副本不可用
-- PROGRESS < 1 表示同步还在进行
```

```sql
-- 看每个 TiFlash 节点的 region 同步延迟
SELECT * FROM information_schema.tiflash_segments;
```

**应对**：

- **加 TiFlash 节点**：分摊 Region 数
- **限制大查询并发**：用 `tiflash_max_threads` 或 Resource Group 隔离
- **关键报表加保护**：通过 `tiflash_replica_read = "closest_replicas"` 让 TiFlash 不从落后节点读
- **业务侧兜底**：报表 SQL 检查"最大时间戳"是否新于阈值，否则 fallback 到 TiKV

```sql
-- 设置：TiFlash 落后超过 N 秒就报错（默认 30s）
SET tiflash_replica_read = 'all_replicas';
```

**长效**：TiFlash 7.x 的 storage compute 分离架构（Serverless 模式）能把存算分开，缓解 TiFlash 节点的资源竞争。

#### 验证

```sql
-- 报表查询前先验证副本健康
SELECT MAX(created_at) FROM orders;   -- 看时间戳
SELECT progress, available FROM information_schema.tiflash_replica WHERE TABLE_NAME='orders';

-- progress = 1, available = 1 才放心
```

> **口诀**：TiFlash 是异步 Learner，**HTAP 不等于"实时一致"**。关键报表必须有"数据新鲜度"校验，否则等于在看历史快照。

---

### 案例 ⑥ 分布式 JOIN 内存放大 —— "OOM 把 TiDB Server 干掉"

#### 现象

分析师跑了一个看似无害的查询：

```sql
SELECT a.user_id, b.order_id, c.product_name
FROM users a
JOIN orders b ON a.user_id = b.user_id
JOIN products c ON b.product_id = c.product_id
WHERE a.region = 'CN';
```

`tidb.log`：

```
[FATAL] [tidb-server.go:412] runtime error: out of memory
```

TiDB Server 进程被 OOM Killer 干掉。监控看 RSS 暴涨到 32GB（机器 64GB）然后崩溃。

#### 根因

TiDB Server 是无状态 SQL 层，但 **JOIN / 聚合 / 排序的中间结果在 TiDB Server 内存里**。一个大 JOIN 可能：

1. 从 TiKV 拉 a 表（1000 万行）
2. 从 TiKV 拉 b 表（1 亿行）
3. 在内存做 hash join → hash table 占 GB 级
4. 加上 c 表更大

而且 TiDB 的 hash join 默认**两边都要加载到内存**（如果走 build / probe 模式，build 表必须能装下）。

数据放大的另一来源：**没有合适的下推**。看 `EXPLAIN`：

```
HashJoin   ← root（在 TiDB Server 跑）
├── TableReader   ← cop[tikv]
│   └── TableScan  table:a, range:[..]
└── TableReader   ← cop[tikv]
    └── TableScan  table:b, range:[..]
```

如果 b 表 100GB，全量被拉到 TiDB Server，必然 OOM。

#### 修复

**方案 A**：限制单 SQL 内存。

```sql
-- 单 SQL 超过 10GB 就 kill
SET GLOBAL tidb_mem_quota_query = 10737418240;

-- 行为：cancel / log / kill_session
SET GLOBAL tidb_mem_oom_action = 'CANCEL';
```

**方案 B**：开启磁盘溢出（spill）。

```sql
SET GLOBAL tidb_enable_tmp_storage_on_oom = ON;
SET GLOBAL tmp-storage-quota = 64 * 1024 * 1024 * 1024;   -- 64GB
```

开启后大 JOIN 的中间结果会溢出到本地磁盘，避免 OOM 但变慢。

**方案 C**：让 TiFlash + MPP 接走。

```sql
-- 强制走 TiFlash MPP
SET tidb_allow_mpp = ON;
SET tidb_enforce_mpp = ON;
SELECT /*+ READ_FROM_STORAGE(tiflash[a, b, c]) */ ...
```

MPP 模式下 JOIN 在 TiFlash 节点之间分布式执行，单节点内存压力小得多。

**方案 D**：业务 SQL 优化。
- 加合适的索引让 TableScan 变 IndexScan
- 用子查询提前过滤：`WHERE user_id IN (SELECT user_id FROM users WHERE region='CN')`
- 用 LIMIT + 分页

#### 验证

```sql
-- 看每个查询的内存使用
SELECT * FROM information_schema.processlist WHERE MEM > 1 * 1024 * 1024 * 1024;

-- 看历史 OOM
SELECT * FROM information_schema.cluster_slow_query
WHERE Mem_max > 1024*1024*1024
ORDER BY Time DESC LIMIT 20;
```

> **口诀**：TiDB Server 的内存是分析查询的天花板。**默认开启 `mem_quota_query` 和 spill**，给重型分析配 TiFlash MPP，不要让一个 SQL 干掉整台 TiDB Server。

---

### 案例 ⑦ Coprocessor 下推失效 —— "为什么这条简单 SQL 慢 100 倍"

#### 现象

```sql
-- A: 5ms
SELECT * FROM orders WHERE user_id = 12345 AND status = 1;

-- B: 5000ms
SELECT * FROM orders WHERE user_id = 12345 AND DATE(created_at) = '2024-01-13';
```

两条查询表面看差不多，B 多了一个 DATE 函数，慢 1000 倍。

#### 根因

`EXPLAIN` 看 B：

```
TableReader_7   1.00  root  data:Selection_6
└─Selection_6   1.00  cop[tikv]  ...
  └─TableScan_5 1000000.00 cop[tikv] table:orders, keep order:false
```

注意 `TableScan` 的估算行数 **1000000**！全表扫了 100 万行。

为什么？**`DATE(created_at) = '2024-01-13'` 这个表达式无法用索引**，且 `DATE()` 函数在 TiKV Coprocessor 里要执行，相当于把所有行拉过来过滤。

而 A 的 `user_id = 12345 AND status = 1` 走了 `idx_user_id` 索引，只读几行。

更糟的下推失败例子：
- `WHERE substr(name, 1, 3) = 'abc'`：函数无法走索引
- `WHERE id + 1 = 100`：表达式中包含列计算，无法用索引
- `WHERE LOWER(email) = 'xxx'`：函数应用在列上

最危险：**TiDB 不支持的某些函数会回到 root 层执行**，意味着 TiKV 把全部数据拉到 TiDB Server。

#### 修复

**方案 A**：把函数挪到值上。

```sql
-- ❌ 函数应用在列上
WHERE DATE(created_at) = '2024-01-13'

-- ✅ 函数应用在值上，列保持原样
WHERE created_at >= '2024-01-13' AND created_at < '2024-01-14'
```

后者可以走 `created_at` 上的索引。

**方案 B**：Generated Column + 索引。

```sql
ALTER TABLE orders ADD COLUMN created_date DATE GENERATED ALWAYS AS (DATE(created_at)) STORED;
ALTER TABLE orders ADD INDEX idx_created_date (created_date);

-- 然后查询自动走索引
WHERE created_date = '2024-01-13'
```

**方案 C**：用 `EXPLAIN` 主动审计。

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE DATE(created_at) = '2024-01-13';
-- 看 actRows 和 estRows，差距大说明优化器估错了
-- 看 cop_task / total_keys，total_keys 大说明全扫了
```

**方案 D**：定期 ANALYZE 收集统计信息。

```sql
ANALYZE TABLE orders;
-- 自动 analyze 默认开启，但大表可能要手动触发
```

#### 验证

```sql
-- 慢日志里筛出全表扫
SELECT Query, Total_keys, Process_keys
FROM information_schema.cluster_slow_query
WHERE Total_keys > 100000
ORDER BY Time DESC LIMIT 20;
```

> **口诀**：列上套函数 = 索引失效。`EXPLAIN` 是 TiDB 性能调优的第一工具，**没看 EXPLAIN 就上线的 SQL 都是定时炸弹**。

---

### 案例 ⑧ MySQL 兼容性陷阱 —— "本地能跑 TiDB 上炸"

#### 现象

某 ORM 生成的 SQL 在 MySQL 上正常，到 TiDB 上各种报错或行为不一致。下面汇总几类常见的兼容性陷阱：

#### 不兼容点 1：自增 ID 不连续

```sql
INSERT INTO t (name) VALUES ('a'), ('b'), ('c');
-- MySQL 8.0：id = 1, 2, 3
-- TiDB：id 可能是 1, 30001, 60001（每个 TiDB Server 一段范围）
```

**原因**：TiDB 的 AUTO_INCREMENT 是**每个 TiDB Server 缓存一段（默认 30000）本地分配**。重启或换 Server 时段会跳。

**修复**：
- 业务不要依赖自增 ID 连续
- 必须连续 → `SET tidb_allow_remove_auto_inc = 1` 用单调严格递增模式（性能差）
- 推荐用 `AUTO_RANDOM` 一开始就接受"无序但唯一"

#### 不兼容点 2：外键

TiDB 6.6+ 才支持外键。之前版本 `CREATE TABLE ... FOREIGN KEY` 是**静默忽略**的。

```sql
-- TiDB 6.5 及以前：FK 子句被解析但不生效
CREATE TABLE child (
    pid INT,
    FOREIGN KEY (pid) REFERENCES parent(id)
);
INSERT INTO child VALUES (999);   -- ❌ MySQL 报错；TiDB 静默成功
```

**修复**：升 6.6+，或业务层做引用完整性。

#### 不兼容点 3：特定函数 / 语法不支持

| 不支持 | 替代 |
| --- | --- |
| 存储过程 / 触发器 / 视图（部分） | 应用层实现 |
| `CREATE TEMPORARY TABLE`（5.3 之前） | 子查询 / WITH |
| 全文索引 FULLTEXT | 接 ES / Manticore |
| 空间索引 SPATIAL | 暂不支持 |
| `LOCK TABLES` | 用悲观事务 |
| 部分字符集（如 ucs2） | 用 utf8mb4 |

#### 不兼容点 4：DDL 行为

```sql
-- MySQL：ALGORITHM=INSTANT 几乎瞬时
ALTER TABLE t ADD COLUMN c INT;

-- TiDB：DDL 也很快但是异步执行
-- DDL 提交后立即返回，但需要等所有 TiDB Server 同步 schema
```

**修复**：DDL 后给一个等待窗口，或调用 `ADMIN SHOW DDL JOBS` 确认完成。

#### 不兼容点 5：临时表

TiDB 5.3 才支持 `CREATE GLOBAL TEMPORARY TABLE` 和 `CREATE TEMPORARY TABLE`，且行为与 MySQL 不完全一致。

#### 不兼容点 6：字符集 / Collation

```sql
CREATE TABLE t (name VARCHAR(50));
-- MySQL 默认 utf8mb4_general_ci
-- TiDB 默认 utf8mb4_bin（区分大小写）
INSERT INTO t VALUES ('Apple');
SELECT * FROM t WHERE name = 'apple';
-- MySQL：返回 Apple
-- TiDB：返回空
```

**修复**：建表显式指定 `COLLATE utf8mb4_general_ci`，或开 `new_collations_enabled_on_first_bootstrap`（仅初始化前生效）。

#### 不兼容点 7：MySQL 系统变量

很多 MySQL 系统变量 TiDB 不支持或语义不同：

```sql
SET sql_log_bin = OFF;   -- TiDB 接受语法但行为不同（TiDB 没有 binlog 传统）
SET autocommit = 0;       -- 兼容
```

#### 不兼容点 8：`SAVEPOINT`、嵌套事务

TiDB 4.0+ 才支持 SAVEPOINT，且不支持嵌套事务。

#### 修复（系统化）

1. 用 [TiDB Compatibility Tool](https://docs.pingcap.com/tidb/stable/) 跑 SQL 兼容性扫描
2. 应用层 DAO 封装，避免业务直接写 raw SQL
3. CI 中跑同一套测试用例：MySQL + TiDB 两套，对比结果

#### 验证

```sql
-- 跑业务的关键 DAO 测试集
-- 关注 INSERT 后的自增 ID 是否符合预期
-- 大小写敏感的查询行为
-- DDL 后立即查询的可见性
```

> **口诀**：TiDB 不是"MySQL 的多机版"，而是"绝大部分场景兼容 MySQL"。**关键边界必须用业务测试覆盖**，别信"100% 兼容"的宣传。

---

### 案例 ⑨ BR 备份冲击线上 —— "做了一次备份，业务挂了 30 分钟"

#### 现象

DBA 凌晨用 BR（Backup & Restore）做全量备份。备份开始后业务监控告警：P99 飙到 5 秒，部分请求超时。

```bash
br backup full --pd "10.0.0.1:2379" --storage "s3://bucket/backup-2024-01-13" --ratelimit 128
```

#### 根因

BR 的工作机制：通过 PD 拿到所有 Region → 让每个 TiKV 节点扫描自己的数据 → 写到目标存储。

冲击点：
1. **Region 扫描占用 CPU 和 IO**：TiKV 同时要服务在线流量，资源被抢
2. **网络上传带宽**：备份数据量大，挤占了内部 RPC 带宽
3. **TiKV 的 Coprocessor 队列**：备份请求和业务请求挤同一队列，业务排队
4. 默认 `--ratelimit` 不设上限就是"全速跑"

更隐蔽的：BR 的 backup task 也会触发 PD 的 scheduler 暂停某些操作（如 region merge），积累一段时间后释放，可能引发短时调度风暴。

#### 修复

**方案 A**：限速 + 错峰。

```bash
# 限速到每节点 50MB/s
br backup full \
  --pd "10.0.0.1:2379" \
  --storage "s3://bucket/backup" \
  --ratelimit 50 \
  --concurrency 4
```

**方案 B**：用 Resource Control（TiDB 7.0+）。

```sql
-- 给 BR 用户单独的 Resource Group，限制其 RU 消耗
CREATE RESOURCE GROUP rg_br RU_PER_SEC = 1000;
-- BR 连接时绑定该 Resource Group
```

**方案 C**：增量备份 + log backup。

```bash
# 全量周做一次
br backup full ...

# 日常增量（基于 CDC）
br log start --task-name=bk1 --pd ... --storage ...
```

增量备份对线上的冲击远小于全量。

**方案 D**：物理备份分担到从节点。

虽然 TiDB 没有传统主从概念，但可以临时加几个 TiKV 节点专门用于备份扫描（标签调度），扫完就缩容。

#### 验证

```bash
# 观察备份期间的业务影响
# Grafana → tidb-overview → query duration
# tikv-details → CPU / IO
# 确认 P99 没有飙升
```

> **口诀**：BR 备份对线上有冲击。**生产 BR 必须配 ratelimit 和 concurrency，重要业务用 Resource Group 隔离**。

---

## 5. 上线检查清单

打印贴墙 / 加进运维 checklist。

### 选型层
- [ ] 数据量 < 1TB、QPS < 5万 → 优先单机 MySQL，不要轻易上 TiDB
- [ ] 业务读写比、热点分布、事务大小三项都做过画像
- [ ] 评估 MySQL 兼容性边界（自增 / 外键 / 字符集 / 函数清单）
- [ ] 确认机房 RTT 与写延迟预算（多机房 RTT 直接进事务延迟）
- [ ] 确认 TiFlash 必要性（OLAP 比重 > 30% 才有意义）

### 架构层
- [ ] PD 至少 3 节点，专机部署，与 TiKV 隔离 IO
- [ ] TiKV 至少 3 节点，副本数 ≥ 3
- [ ] TiDB Server 至少 2 节点，前面挂 LB（HAProxy / NLB）
- [ ] 监控 + 告警接入 Prometheus + AlertManager
- [ ] BR 备份策略（全量 + 增量 + log backup）

### 表设计层
- [ ] 主键用 `AUTO_RANDOM` 而非 `AUTO_INCREMENT`
- [ ] 大表建表加 `PRE_SPLIT_REGIONS` 预切分
- [ ] 业务键避免单调递增（时间、序号）
- [ ] 字符集统一 utf8mb4，明确 collation
- [ ] 大字段（TEXT/BLOB）单独表，避免行变大触发 split

### SQL 层
- [ ] 关键 SQL 都过 EXPLAIN ANALYZE
- [ ] 列上不套函数；时间范围查询用 `>=` 配 `<`
- [ ] 慢查询日志开启，`tidb_slow_log_threshold = 300ms`
- [ ] 大 JOIN 评估走 TiFlash MPP
- [ ] 单 SQL 内存上限 `tidb_mem_quota_query = 1GB` + spill 开启

### 事务层
- [ ] **默认用悲观事务**（`tidb_txn_mode = 'pessimistic'`）
- [ ] 应用层避免长事务（事务体内禁止远程调用）
- [ ] 事务超时 `innodb_lock_wait_timeout` + 应用层超时配齐
- [ ] 业务幂等的 → 可用 `tidb_disable_txn_auto_retry = 0` 自动重试

### 容量层
- [ ] TiKV 磁盘水位监控 70/80/85% 三级告警
- [ ] PD 元数据 etcd 容量监控
- [ ] Region 数量监控（单节点 > 50000 个就要考虑拆 Region 或扩节点）
- [ ] 内存监控：TiKV `block-cache.capacity` 配置合理（一般机器内存的 30%）

### 备份层
- [ ] BR 配 `--ratelimit` 限速
- [ ] 用 Resource Control 隔离备份的 RU 消耗
- [ ] 定期演练恢复（备份不演练等于没备份）
- [ ] 跨地域副本（DR 需求）

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### CockroachDB 的对比
- **架构差别**：CockroachDB 是"对等架构"，每个节点既是 SQL 又是存储；TiDB 把 SQL 层和存储层物理分开
- **事务模型**：CockroachDB 用 **HLC（Hybrid Logical Clock） + 分布式 SSI**，不依赖中心 TSO
- **强项**：跨地域部署天然友好，无 PD 单点
- **弱项**：MySQL 兼容性差（兼容 PostgreSQL）；HTAP 能力弱
- 看 *CockroachDB SQL Layer* 论文（SIGMOD 2020）和源码 `pkg/kv/kvserver/`

### YugabyteDB 的对比
- **架构差别**：YSQL（PostgreSQL 兼容）+ YCQL（Cassandra 兼容）；存储层基于 RocksDB
- **事务模型**：基于 Spanner 的 **Hybrid Logical Clock + 2PC**
- **强项**：双 API、多模型；复用 PostgreSQL 的优化器
- **弱项**：生态相对小；中文社区少
- YugabyteDB 的两层架构（Query layer + DocDB）是个值得研究的设计

### OceanBase 的对比
- **架构差别**：单机分区 + 多副本 Paxos；不像 TiDB 把数据切成无数 Region
- **事务模型**：基于 Paxos 的强一致 + 多版本并发控制
- **强项**：金融级强一致；TPC-C 世界纪录；同城多活成熟
- **弱项**：单机表的瓶颈仍在；MySQL/Oracle 双兼容是优势也是复杂度
- 看蚂蚁的论文 *OceanBase: A 707 Million tpmC Distributed Relational Database System*（VLDB 2022）

### Spanner 的 TrueTime + Paxos
- **TrueTime**：Google 用 GPS + 原子钟提供 "[earliest, latest]" 区间时间戳
- 解决了"无中心 TSO 也能保证全序"的问题
- TiDB 也在探索 HLC 的方向（实验阶段）
- 看原论文 *Spanner: Google's Globally-Distributed Database*（2012）

### DynamoDB / FoundationDB
- **DynamoDB**：完全 NoSQL，无 SQL 层，按 partition key 路由
- **FoundationDB**：苹果在用，**有序 KV + 客户端事务**，可以在上面构建 SQL 层（Snowflake 元数据用它）
- 这两家代表了"非 NewSQL 但能解决类似规模问题"的另一条路

### HTAP 的演进
- **Snowflake**：完全云原生 OLAP，存算彻底分离，无 OLTP
- **Databricks (Photon)**：Lakehouse，OLAP 融合
- **TiDB Serverless / TiFlash 存算分离**：S3 + Compute Pod，按 query 计费
- HTAP 的下半场是 **"用云的弹性把行存和列存彻底解耦"**

### TiDB Serverless
- 多租户共享 TiKV / TiFlash，按 RU（Request Unit）计费
- Compute 层 Pod 化，弹性扩缩
- 适合 SaaS 业务和"用量不可预测"的场景
- 缺点：不能自定义参数、对 IO 抖动更敏感

### TiKV Raft 优化
- **Raft Engine**：替代 RocksDB 作为 Raft log 存储，吞吐提升 30%+
- **Multi-Raft 的 Hibernate**：闲置 Region 不发心跳
- **Joint Consensus**：Raft 配置变更的"两阶段"实现
- 看 PingCAP 在 SIGMOD 2020 的 *TiDB: A Raft-based HTAP Database* 论文

---

## 7. 引用

- TiDB / TiKV 源码：[https://github.com/pingcap/tidb](https://github.com/pingcap/tidb)，[https://github.com/tikv/tikv](https://github.com/tikv/tikv)
- TiDB 官方文档：[https://docs.pingcap.com/zh/tidb/stable](https://docs.pingcap.com/zh/tidb/stable)
- *TiDB: A Raft-based HTAP Database*（VLDB 2020）—— TiDB 官方论文
- *Large-scale Incremental Processing Using Distributed Transactions and Notifications*（OSDI 2010）—— Google Percolator 论文
- *Spanner: Google's Globally-Distributed Database*（2012）
- *CockroachDB SQL Layer*（SIGMOD 2020）
- *OceanBase: A 707 Million tpmC Distributed Relational Database System*（VLDB 2022）
- 《分布式系统原理与范型》—— Tanenbaum
- 《数据密集型应用系统设计》—— Martin Kleppmann（"DDIA"）

---

> 配套：[MySQL 分库分表](/article/db/sharding) | [MySQL MVCC 与锁](/article/db/mysql-mvcc) | [共识算法](/article/distributed/consensus) | [分布式事务](/article/distributed/dist-tx)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
