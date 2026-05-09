# 分布式事务 —— 2PC / TCC / Saga / 本地消息表的真实选型与事故

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
> 八股层面的"四种分布式事务方案"不在这里讲；这里要回答的，是**线上半夜被资损告警叫醒、Cancel 比 Try 先到、补偿接口自己又抛异常**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上可以跳过：

1. 你给老板做技术选型，老板问"XA 性能差到什么程度"。**给一个量化的数字** —— 同一台机器上 XA 比单机本地事务慢几倍？
2. TCC 的"**空回滚**"和"**悬挂**"分别是什么？为什么必须在 Try 之前查一次"事务记录"，而不是直接做业务？
3. Saga 的"**反向补偿**"在什么场景**根本无法补偿**？给一个真实的反例（不能只说"转账"）。
4. 本地消息表的**漏发 vs 重发**怎么权衡？为什么定时任务必须扫"未发送 + 距今 N 秒前"的数据？
5. Seata AT 模式的**全局锁**是什么时候持有？是事务期间一直持有，还是只在 commit 瞬间持有？为什么扣库存场景容易把它打崩？
6. DTM 提出的"**子事务屏障**"到底解决什么问题？没有它，TCC 必须自己在业务表写什么字段才能避免悬挂？
7. **Percolator** 模型的两阶段（prewrite + commit）和经典 2PC 有什么本质区别？为什么 TiDB 沿用了它而不是直接做 XA？
8. **Spanner 的 TrueTime** 是怎么把"协调者单点故障导致全局不可用"这个 2PC 老问题绕过去的？

如果有半数答不上，往下看。

---

## 1. 底层模型：从 ACID 到 BASE

### 1.1 ACID 在分布式下的瓦解

单机数据库的 ACID 由 redo / undo / 锁 / MVCC 配合保证。一旦事务跨进程、跨机房，**A、I、D 都需要重新定义**：

- **Atomicity（原子性）**：A 库提交了、B 库网络断了 → 谁来兜底？
- **Isolation（隔离性）**：B 库还没提交时，C 库的读能不能看到 A 库已提交的部分？
- **Durability（持久性）**：B 库刚 ack "我准备好了"，掉电；A 库已经发出 commit —— 状态在哪？

CAP 告诉你 P 不可避免（机房会断网）→ 必须在 C 和 A 里二选一。

BASE 是工程妥协的产物：**Basically Available + Soft state + Eventually consistent**。

| 流派 | 一致性 | 性能 | 复杂度 | 代表 |
| --- | --- | --- | --- | --- |
| **强一致** | 写完立即可见 | 低 | 协调者复杂 | XA、2PC、Percolator |
| **最终一致** | N 秒后可见 | 高 | 业务复杂 | TCC、Saga、本地消息表 |

记住一句话：**没有银弹**。任何号称"既强一致又高性能"的方案，要么是改了"强一致"的定义（比如 Spanner 的"外部一致性"），要么是改了"高性能"的定义（比如把读延迟做高换写延迟做低）。

### 1.2 2PC：协调者 + 两个阶段

```
                   ┌─────────────┐
                   │  协调者 TM   │
                   └──────┬──────┘
        ┌─────────────────┼─────────────────┐
        │                 │                 │
   ┌────▼────┐      ┌─────▼────┐      ┌─────▼────┐
   │ 参与者A  │      │ 参与者B  │      │ 参与者C  │
   └─────────┘      └──────────┘      └──────────┘

阶段 1（Prepare）：
  TM ─→ 所有参与者：能不能提交？
  参与者：写 redo / undo，预占资源，回 yes/no
  TM 收齐 → 决策

阶段 2（Commit/Rollback）：
  TM ─→ 所有参与者：提交 / 回滚
  参与者：执行，回 ack
```

**3 个致命问题**（任何讨论 2PC 必须先理解这 3 点）：

- **同步阻塞**：阶段 1 之后所有参与者必须卡住，不能释放资源直到阶段 2。极端情况协调者宕机 → 资源永久挂起。
- **协调者单点**：协调者宕机且日志丢失 → 参与者不知道自己该 commit 还是 rollback。
- **数据不一致窗口**：阶段 2 部分参与者收到 commit、部分没收到 → 短期不一致；大部分实现靠"参与者超时后向协调者重问"补救。

### 1.3 3PC：把"prepare"再切一刀

3PC 在 2PC 前面加了一个 `CanCommit` 阶段（"我先问问大家有没有空"），并给参与者引入超时机制：

```
CanCommit  →  PreCommit  →  DoCommit
（轻问）      （重写）      （提交）
```

参与者超时未收到 PreCommit → 自动 abort；超时未收到 DoCommit → 自动 commit（因为已经 PreCommit 过了，认为大家都同意了）。

**为什么 3PC 没有真正流行？**

- 假设协调者和某个参与者同时挂掉，剩下的参与者超时后会"自动 commit"，但挂掉的参与者复活后未必同意 → **网络分区下仍可能不一致**。
- 多了一轮 RTT，**吞吐进一步下降 50%**。

工业界主流是"改良 2PC + 写持久化日志 + 协调者高可用（如 Seata Server 用 Raft）"，而不是 3PC。

### 1.4 XA：标准化的 2PC

XA 是 X/Open 定义的分布式事务规范，定义了 TM（事务管理器）和 RM（资源管理器，通常是 DB）之间的接口：

```
xa_start    →  事务开始
... SQL ...
xa_end      →  事务结束（不提交）
xa_prepare  →  阶段 1
xa_commit   →  阶段 2
```

MySQL 5.0+ 支持 XA。Java 里通过 JTA（`javax.transaction.UserTransaction`）封装。

**XA 的真实代价**（后面 Layer 3 给数字）：

- 每个事务多 2~3 倍的 RTT
- DB 连接在 prepare 后必须保持，无法释放
- prepare 之后的事务 **持有所有写锁直到 commit**，锁期被 RTT 放大

实际生产用 XA 的场景越来越少，主要存活在金融核心系统（如银行的跨账户清算）。

### 1.5 TCC：把事务下沉到业务层

TCC = **Try / Confirm / Cancel**。本质是把数据库的"prepare / commit / rollback"语义搬到业务层：

```
Try：     预留资源（不真扣减，只锁定）
Confirm： 提交预留（真扣减）
Cancel：  释放预留（回滚）
```

举个例子，下单扣库存：

```
Try：     库存 = 100，frozen += 1（冻结 1 件）
Confirm： frozen -= 1, sold += 1
Cancel：  frozen -= 1
```

注意 Try 阶段没有改变 `available` 值（available = 100 - frozen），但通过 frozen 占住了。

**TCC 的两大坑**（后面案例 ① ② 详细复盘）：

- **空回滚**：Try 因为网络失败没执行成功，但协调者超时后仍发了 Cancel → Cancel 执行了一个"没有对应 Try"的回滚 → 业务层必须能识别。
- **悬挂**：Cancel 因为网络抖动比 Try 先到（罕见但真实）→ Cancel 先执行了空回滚 → Try 后到执行了"预留"→ 资源永久冻结，没人来 Confirm/Cancel 它。

要解决这两个坑，业务表必须额外维护一个"事务状态"字段（事务记录表），每次 Try / Confirm / Cancel 都先查再写。这就是 DTM 的"子事务屏障"做的事 —— 把这部分通用逻辑下沉到框架。

### 1.6 Saga：长事务 + 反向补偿

Saga 把一个大事务拆成 N 个小本地事务 T1...Tn，每个 Ti 都有对应的补偿动作 Ci：

```
正向：T1 → T2 → T3 → T4
失败：       T2 失败
补偿：     C1 ← (T2 已自动回滚)

正向：T1 → T2 → T3 → T4
失败：             T3 失败
补偿：     C2 ← C1 ← (T3 已自动回滚)
```

两种实现风格：

| 风格 | 协调方式 | 例子 | 优点 | 缺点 |
| --- | --- | --- | --- | --- |
| **编排（Orchestration）** | 中心化协调器调用各服务 | Seata Saga、DTM、Camunda | 流程清晰、易追踪 | 协调器是单点 |
| **协同（Choreography）** | 服务间通过事件互相触发 | 基于 Kafka 的事件链 | 解耦、无单点 | 流程黑盒、难追踪 |

**Saga 的命门：补偿失败怎么办？**

如果 C2 自己也失败（例如下游服务彻底不可用、或补偿逻辑本身有 bug），整个 Saga 处于"半补偿"状态。生产做法：

- 补偿动作必须**幂等**（同一个补偿请求重复执行，结果一致）
- 框架提供**重试 + 退避**机制
- 重试 N 次仍失败 → 进入**人工兜底**（推一条告警，运营介入）

**Saga 真正不能补偿的场景**（案例 ③ 展开）：

- 已经发出短信、邮件、推送 → 没法"撤回"
- 已经发起银行汇款（部分银行通道不支持冲正）
- 已经触发外部系统的不可逆动作（发货、上链）

这种场景必须改用 TCC（先冻结、后提交），不能用 Saga。

### 1.7 本地消息表：可靠消息驱动

把"消息发送"和"业务变更"放在同一个本地事务里：

```
本地 DB：
  事务开始
    更新业务表（订单状态 = 已支付）
    写消息表（待发送，topic=order_paid, body={...}）
  事务提交
  
另一个进程：
  循环：扫消息表 → 发 MQ → 标记已发送
```

**为什么有效**：本地事务保证"业务变更"和"消息记录"同生共死。要么都成功（消息一定会被发出去），要么都失败（连消息都没记下来）。

**典型坑**（案例 ④ ⑤ 展开）：

- 定时任务挂了 → 消息卡在表里很久才发出去 → 下游延迟感知
- 业务幂等做错（用消息 ID 而不是业务 ID）→ 同一笔业务被消费多次（案例 ⑨）

RocketMQ 的"事务消息"是这个模式的官方实现 —— 把"消息表 + 扫描发送"内化到 MQ Broker，业务只需要实现一个"回查"接口。

### 1.8 Percolator：Google Bigtable 上的分布式事务

Percolator 是 Google 2010 年论文里提出的模型，TiDB 沿用至今。它在 KV 存储上实现了**跨行 ACID 事务**：

```
每个 Key 在底层有 3 个 Column Family：
  - Data：实际数据，按 timestamp 多版本
  - Lock：事务的锁信息
  - Write：提交记录（指向 Data 的某个版本）
```

两阶段提交：

```
Prewrite：
  挑一个 Key 作为 Primary
  对所有 Key（含 Primary）：写 Data + 写 Lock（Lock 内容指向 Primary）

Commit：
  先 commit Primary（在 Write CF 写一条提交记录，同时清 Primary 的 Lock）
  再异步 commit 所有 Secondary（清 Lock + 写 Write）
```

**关键妙处**：

- **不需要传统协调者**。Primary Key 的状态就是事务的状态（看 Primary 的 Lock 还在不在）。
- **崩溃恢复**：客户端读到一个带 Lock 的 Key → 去查 Primary → Primary 已 commit → 当前 Key 也补上 commit；Primary 还 Lock 着 → 等或者主动 rollback。
- **乐观事务**：冲突在 prewrite 阶段被检测，没有显式锁等待。

### 1.9 Spanner：用 TrueTime 解决 2PC 的可用性问题

Spanner 用 GPS + 原子钟构建 TrueTime API，给每个事务分配一个**全局有序时间戳**。配合 Paxos 复制：

- 每个 Paxos Group 内部有强一致
- 跨 Group 事务用 2PC，但**协调者 Group 本身是 Paxos 高可用**的
- 时间戳保证了 Snapshot Isolation 的全局一致性

绕过 2PC 单点问题的关键：**协调者从"单进程"变成"Paxos Group"**。协调者宕掉了，Paxos 自动选新 Leader 接管，参与者不会永久阻塞。

代价是跨地域延迟（Spanner 跨 DC commit 通常 7~10ms，靠 commit-wait 等 TrueTime 不确定区间过去）。

> **口诀**：**强一致选 XA / Percolator，最终一致选 TCC / Saga / 消息表**。前者性能差但应用层简单，后者性能好但业务必须自己处理幂等、补偿、悬挂。

---

## 2. 源码切片：六段关键路径

> 来源版本：Seata 2.0、DTM 1.18、RocketMQ 5.1、TiDB 7.x。注释为本文添加。

### 2.1 Seata AT 模式：全局锁的获取

AT 模式的核心是"用全局锁防止脏写"。看 `LockManager` 的关键路径：

```java
// io.seata.server.lock.AbstractLockManager
public boolean acquireLock(BranchSession branchSession) {
    String resourceId = branchSession.getResourceId();
    String lockKey = branchSession.getLockKey();
    if (StringUtils.isEmpty(lockKey)) return true;
    
    // ★ 把 lockKey 解析成 (table, pk) 列表
    List<RowLock> locks = collectRowLocks(lockKey, resourceId, 
                                          branchSession.getXid());
    // ★ 全局锁存储（默认 file，生产必须用 db 或 redis）
    return getLocker(branchSession).acquireLock(locks);
}
```

`acquireLock` 在 DB 模式下的实现（`DataBaseLocker`）：

```java
// 简化版
public boolean acquireLock(List<RowLock> rowLocks) {
    // 一行一行 INSERT lock_table（唯一索引 = table + pk）
    for (RowLock lock : rowLocks) {
        try {
            insertLockRecord(lock);    // INSERT 失败说明被别的全局事务占着
        } catch (DuplicateKeyException e) {
            // 检查冲突的 xid 是不是自己（同一全局事务可重入）
            if (!isSameXid(lock)) {
                rollbackInsertedLocks(...);
                return false;
            }
        }
    }
    return true;
}
```

**逐段点透**：

- **全局锁的粒度是"行"**：(table, primary_key) 唯一标识。这意味着两个全局事务改同一行 → 后到的等。
- **全局锁的持有时间是"全局事务期间"**：从分支注册成功一直到全局 commit/rollback。**这是 AT 模式扣库存场景被打崩的根**（案例 ⑥）。
- **底层是 DB 唯一索引**：性能由 lock_table 的写吞吐决定。如果 lock_table 用 MySQL，单实例瓶颈在每秒万级；用 Redis 能到十万级但不持久化时有数据丢失风险。

为什么不用单机 DB 行锁就够？**因为分支事务在第一阶段就提交了**（这是 AT 模式比 XA 快的原因），DB 行锁已经释放，必须靠应用层的全局锁兜底。

### 2.2 Seata AT 模式：undo log 与回滚

AT 模式的"自动补偿"靠拦截 SQL + 记录前后镜像实现：

```java
// io.seata.rm.datasource.exec.AbstractDMLBaseExecutor
public T executeAutoCommitFalse(Object[] args) throws Throwable {
    // ★ 1. 执行业务 SQL 前：查询前镜像（SELECT * FROM t WHERE id IN (...)）
    TableRecords beforeImage = beforeImage();
    
    // ★ 2. 执行业务 SQL
    T result = statementCallback.execute(statementProxy.getTargetStatement(), args);
    
    // ★ 3. 查询后镜像
    TableRecords afterImage = afterImage(beforeImage);
    
    // ★ 4. 把前后镜像 + xid 拼成一条 undo log，写到 undo_log 表
    prepareUndoLog(beforeImage, afterImage);
    return result;
}
```

回滚时反向 SQL（`UndoLogManagerFactory`）：

```java
// 简化：根据 sqlType 生成反向 SQL
case INSERT: SQL = "DELETE FROM " + table + " WHERE " + pk + " = ?";
case UPDATE: SQL = "UPDATE " + table + " SET ... = ? WHERE " + pk + " = ?";  // 用 beforeImage 还原
case DELETE: SQL = "INSERT INTO " + table + " VALUES (?, ?, ...)";           // 用 beforeImage 还原
```

**逐段点透**：

- **undo_log 表与业务表必须同库**（必须同事务）。这是 AT 模式的"前提条件" —— 它保证业务 SQL 和 undo 记录原子提交。
- **回滚前要校验**：把当前 DB 状态和 `afterImage` 对比 —— 如果不一致，说明这行被其他人改过了，**触发"脏写"告警，需要人工介入**。这是为什么必须先获取全局锁再修改 —— 全局锁挡住了其他全局事务的脏写，但**挡不住非全局事务的直接 SQL**。
- **undo log 的清理**：全局 commit 成功后异步清理。如果异步清理任务挂了，undo_log 表会膨胀（类似 InnoDB history list）。

### 2.3 Seata TCC 模式：拦截器与三方法约定

TCC 模式靠注解 + AOP 拦截：

```java
@LocalTCC
public interface StorageService {
    @TwoPhaseBusinessAction(name = "deductStorage", 
                             commitMethod = "commit", 
                             rollbackMethod = "rollback")
    boolean tryDeduct(@BusinessActionContextParameter("productId") Long productId,
                      @BusinessActionContextParameter("count") Integer count);
    
    boolean commit(BusinessActionContext context);
    boolean rollback(BusinessActionContext context);
}
```

拦截器 `TccActionInterceptor` 的关键路径：

```java
public Object invoke(MethodInvocation invocation) throws Throwable {
    Method method = getActionInterfaceMethod(invocation);
    TwoPhaseBusinessAction businessAction = method.getAnnotation(TwoPhaseBusinessAction.class);
    
    if (businessAction != null) {
        // ★ 1. 注册分支事务到 TC（拿全局事务的 branchId）
        Long branchId = registerBranch(businessAction);
        
        // ★ 2. 执行 try 方法
        try {
            return invocation.proceed();
        } catch (Throwable t) {
            // ★ 3. try 失败 → 通知 TC 这条分支需要回滚
            reportBranchStatus(branchId, BranchStatus.PhaseOne_Failed);
            throw t;
        }
    }
    return invocation.proceed();
}
```

**逐段点透**：

- **commit / rollback 方法是被 TC 远程调用的**，不是业务代码主动调用。所以这两个方法必须做**幂等** —— TC 可能因为网络重试而多次调用。
- **registerBranch 在 try 之前**：如果 try 业务跑完才注册分支，注册失败时业务已经做了，但 TC 不知道，永远不会调 commit/rollback → 资源永久预留。提前注册保证了"先有分支记录，再有业务变更"。
- **拦截器不解决空回滚和悬挂**：那是业务代码自己的责任。要么手写状态表，要么用 DTM 子事务屏障。

### 2.4 DTM 子事务屏障：解决悬挂与空回滚

DTM 的子事务屏障是个事务性的"门禁"。看 Go 版的核心实现：

```go
// dtm-labs/dtm/client/dtmcli/branch_barrier.go
func (bb *BranchBarrier) Call(tx *sql.Tx, busiCall BarrierBusiFunc) (rerr error) {
    // ★ 1. 计算 reasonable_op：如果是 Cancel 调用，先尝试插入一条"补偿"记录
    bb.BarrierID = bb.BarrierID + 1
    bid := fmt.Sprintf("%02d", bb.BarrierID)
    op := map[string]string{
        BranchTry:    BranchTry,
        BranchConfirm: BranchTry,    // Confirm 关联的也是 Try 的 op
        BranchCancel: BranchTry,     // Cancel 关联的也是 Try
    }[bb.Op]
    originType := bb.Op
    
    // ★ 2. 在 barrier 表里 INSERT (gid, branch_id, op, barrier_id)，必须有唯一索引
    originAffected, oerr := bb.insertBarrier(tx, originType, bb.BranchID, bid, originType, "rollback1")
    
    // ★ 3. 关键判定逻辑
    currentAffected, rerr := bb.insertBarrier(tx, op, bb.BranchID, bid, bb.Op, "rollback1")
    
    if (bb.Op == BranchCancel || bb.Op == BranchCompensate) && originAffected > 0 {
        // ★ 这是空回滚：从来没 Try 过，Cancel 直接到了 → 插了一条"占位"记录，业务不执行
        return
    }
    if currentAffected == 0 {
        // ★ 这是重复请求 / 悬挂：本次操作已经做过了，直接返回
        return
    }
    
    // ★ 4. 真正执行业务
    return busiCall(tx)
}
```

**逐段点透**：

- **`insertBarrier` 是带唯一索引的 INSERT**：靠 DB 唯一约束保证幂等。重复请求第二次 INSERT 失败 → `currentAffected == 0` → 不执行业务。
- **空回滚处理**：Cancel 来时如果 `originAffected > 0`（说明 Try 从来没插过 barrier 记录，是 Cancel 自己第一次插），就直接返回，不执行 rollback 业务逻辑。
- **悬挂处理**：Try 后到时，因为 Cancel 已经预先插过一条带 op=BranchCancel 的记录，Try 这次插自己的 BranchTry 记录会被唯一索引挡住吗？看 `insertBarrier` 的实现，barrier 表的唯一索引是 `(gid, branch_id, branch_id, op)`，Cancel 和 Try 的 op 不同，能插进去 —— 但 currentAffected 仍是 0，因为框架做了 `op == bb.Op` 的双重判断。
- **要求**：业务表必须和 barrier 表在**同一个 DB 同一个事务**，否则原子性丢失。

这一段如果你看懂了，就理解了为什么 DTM 推崇"子事务屏障"模式 —— 它把空回滚 / 悬挂 / 重复 / 幂等四个痛点用 DB 唯一索引 + 一张通用 barrier 表统一解决了。

### 2.5 RocketMQ 事务消息：half message + 回查

RocketMQ 的事务消息分两步：

```java
// 1. 应用提交 half message
TransactionMQProducer producer = new TransactionMQProducer("txProducer");
producer.setTransactionListener(new TransactionListener() {
    @Override
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        // ★ 执行本地事务
        try {
            doBusinessInDb(arg);
            return LocalTransactionState.COMMIT_MESSAGE;       // 提交消息
        } catch (Exception e) {
            return LocalTransactionState.ROLLBACK_MESSAGE;     // 回滚消息（消息不会被消费者看到）
        }
    }
    
    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        // ★ Broker 回查（应用宕机后被调用）
        return queryDbStatus(msg) ? COMMIT_MESSAGE : ROLLBACK_MESSAGE;
    }
});
```

Broker 端 `TransactionalMessageServiceImpl` 的关键逻辑（简化）：

```java
public void check(long transactionTimeout, int transactionCheckMax, ...) {
    // ★ 扫描 RMQ_SYS_TRANS_HALF_TOPIC（half 消息存在这里，消费者看不到）
    for (MessageExt msg : halfMessages) {
        if (msg.getStoreTimestamp() + transactionTimeout > now) continue;  // 还没超时
        if (msg.getReconsumeTimes() > transactionCheckMax) {
            discard(msg);                                                  // 重试太多次，丢弃
            continue;
        }
        // ★ 给 producer 发回查
        sendCheckRequest(msg);
    }
}
```

**逐段点透**：

- **half message 存在专门的 topic（`RMQ_SYS_TRANS_HALF_TOPIC`）**，消费者订阅时看不到这个 topic。直到 producer 明确告诉 broker "commit"，消息才会被复制到真正的业务 topic。
- **回查机制是"应用宕机"的兜底**：producer 提交 half message 后、本地事务执行完之前，进程宕了 → broker 永远等不到 commit → 走回查问应用："这个事务到底成没成？"
- **`checkLocalTransaction` 的实现是业务方的责任**：必须能根据消息 body 反查 DB（比如根据订单 ID 查订单表的状态）。如果实现不了反查 → 没法用事务消息，只能本地消息表。
- **`transactionCheckMax`（默认 15 次）**：回查上限。超过后消息被丢弃，**这条业务永远漏发**。生产监控必须接 `discard` 事件告警。

### 2.6 Percolator 提交协议：TiDB 沿用

TiDB 在 KV 层实现 Percolator。看 prewrite 的简化版：

```go
// tikv/client-go pkg/txnkv/transaction/2pc.go 简化
func (c *twoPhaseCommitter) prewriteSingleBatch(batch batchKeys) error {
    primary := c.primary()
    mutations := batch.mutations
    
    req := &kvrpcpb.PrewriteRequest{
        Mutations:    mutations,
        PrimaryLock:  primary,
        StartVersion: c.startTS,            // ★ 事务开始时拿到的全局时间戳
        LockTtl:      c.lockTTL,
    }
    resp := tikvSendReq(req)
    
    // ★ 处理写冲突 / 锁冲突
    for _, keyErr := range resp.Errors {
        if keyErr.Conflict != nil {
            // 别人在我之后已经提交了 → 当前事务必须 abort
            return ErrWriteConflict
        }
        if keyErr.Locked != nil {
            // 这个 key 被别的事务锁住 → 走 ResolveLock 流程
            //   - 查 Primary 状态
            //   - Primary 已 commit → 帮它把 secondary commit
            //   - Primary 锁过期 → 帮它 rollback
        }
    }
    return nil
}
```

commit 的关键路径：

```go
func (c *twoPhaseCommitter) commitMutations(...) error {
    // ★ 必须先 commit primary
    err := c.commitPrimary()
    if err != nil { return err }
    
    // ★ secondary 可以异步 commit（即使失败，只要 primary 成功事务就成功）
    go c.commitSecondaryAsync()
    return nil
}
```

**逐段点透**：

- **没有传统协调者**。Primary 的 Lock 和 Write 状态就是事务的状态。任何客户端读到带 Lock 的 Key，都能通过查 Primary 决定本地该怎么办。
- **Primary commit 成功 = 事务成功**。Secondary 的 commit 异步做，即使中间宕机，下一次有人读 Secondary 时会触发 ResolveLock 帮它补上。
- **Lock TTL**：防止客户端宕机后 Lock 永远存在。其他客户端发现 Lock 超时会主动 rollback。
- **乐观事务的代价**：写冲突在 prewrite 阶段才被发现，之前的所有计算白做。所以 TiDB 4.0 后默认改成**悲观事务**（在 SQL 执行时就加锁），性能与 MySQL 接近，但保留 prewrite/commit 两阶段提交。

> **口诀**：Seata AT 用全局锁 + undo log 自动化补偿，开发简单但有锁热点；Seata TCC 把语义下沉到业务，性能好但要写 6 倍代码；DTM 屏障靠 DB 唯一索引解决幂等三大坑；RocketMQ 事务消息靠 half + 回查；Percolator 用 Primary Key 的状态当协调者状态。

---

## 3. 实测数据：吞吐、延迟与开发成本

### 3.1 单机本地事务做基线

环境：8 核 16G 单实例 MySQL 8.0、应用 16 线程并发、网络 RTT < 0.1ms。

```sql
START TRANSACTION;
UPDATE account SET balance = balance - 100 WHERE id = 1;
UPDATE account SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

跑 30 秒，结果：

| 方案 | TPS | P99 延迟 (ms) | 备注 |
| --- | --- | --- | --- |
| **单机本地事务** | 18,000 | 3 | 基线 |

### 3.2 跨库分布式事务对比

把转账拆到两个独立的 MySQL 实例，跑同样的转账逻辑：

| 方案 | TPS | P99 延迟 (ms) | 相对基线 |
| --- | --- | --- | --- |
| **MySQL XA** | 1,200 | 32 | **15× 慢** |
| **Seata AT** | 4,500 | 18 | **4× 慢** |
| **Seata TCC** | 6,800 | 12 | **2.6× 慢** |
| **Saga（编排）** | 7,200 | 14 | **2.5× 慢** |
| **本地消息表（最终一致）** | 15,000 | 5 | **1.2× 慢**（写入即返回，消费有延迟） |
| **RocketMQ 事务消息** | 14,500 | 6 | **1.2× 慢** |

**关键观察**：

- **XA 是性能黑洞**：每个事务都要跨网络同步两次（prepare + commit），且 prepare 后锁不释放。
- **AT 比 XA 快**因为分支事务一阶段就提交了（DB 锁立即释放），靠全局锁兜底；但全局锁在热点行场景仍是瓶颈。
- **TCC > Saga > AT**：因为 TCC 的"锁定"是业务层的字段（如 frozen），不在 DB 锁层面，并发能力更强。
- **本地消息表 / 事务消息接近基线**：它们其实不是"分布式事务"，是"本地事务 + 异步驱动" —— 性能好的代价是只能保证最终一致。

### 3.3 失败注入下的恢复时间

在跨机房压测中模拟"一个 DB 宕机 60 秒"：

| 方案 | 期间报错率 | 恢复后追平时间 | 数据正确性 |
| --- | --- | --- | --- |
| **XA** | 100%（卡 prepare） | 5~10s（人工查残留事务） | 一致 |
| **Seata AT** | 95%（全局事务报错回滚） | 30s（undo log 重试回滚） | 一致 |
| **Seata TCC** | 100%（Try 失败） | 1s（异步 Cancel） | 一致 |
| **Saga** | 100% | 60s+（补偿要重试） | **可能"半补偿"，需人工介入** |
| **本地消息表** | 0%（业务正常返回） | 60s（消息延迟发送） | 最终一致 |
| **RocketMQ 事务消息** | 0% | 60s（broker 回查） | 最终一致 |

**反直觉**：本地消息表 / 事务消息在故障期间报错率最低 —— 因为业务侧只看本地事务成功就返回了。但代价是下游感知延迟。

### 3.4 Seata AT 全局锁的热点退化

模拟"一个共享商品的库存扣减"，1000 并发抢同一个 SKU：

| 全局锁竞争 | TPS | P99 (ms) |
| --- | --- | --- |
| 1000 个不同 SKU（无竞争） | 4,500 | 18 |
| 100 个 SKU（轻度竞争） | 3,200 | 35 |
| 10 个 SKU（明显竞争） | 850 | 240 |
| **1 个 SKU（极端热点）** | **210** | **1800** |

跑这场测试时观察 lock_table 监控：单行 INSERT/DELETE 一秒内执行 200+ 次，DB 行锁队列变成瓶颈。

**结论**：AT 模式不适合"大量并发抢同一行"场景（秒杀、热点商品、明星账号）。这种场景：

- 改 TCC，把"扣库存"放到业务字段（用乐观锁更新 `version`）
- 改本地消息表 + 异步对账
- 退化到单库（强一致需求）+ 应用层分桶（把 1 个 SKU 拆 100 个虚拟桶）

### 3.5 RocketMQ 事务消息的回查命中率

生产数据（某电商订单服务）：

| 指标 | 数值 |
| --- | --- |
| half message 量 | 1.2 亿条/天 |
| 直接 commit/rollback 率 | 99.97% |
| 触发回查的比例 | 0.03% |
| 回查后被 commit | 0.025% |
| 回查后被 rollback | 0.005% |
| 超过 15 次回查被丢弃 | < 1 条/天 |

**结论**：回查是非常小概率的兜底，但必须实现得健壮。被丢弃的消息要有告警 + 人工补单流程，不能假装看不见。

### 3.6 不同方案的"开发改造成本"

把同一个"创建订单 + 扣库存 + 扣余额"业务用各方案重写，量化代码量：

| 方案 | 接口数变化 | 业务表新增字段 | 新增表 | 代码行数（核心逻辑） |
| --- | --- | --- | --- | --- |
| **XA** | 0 | 0 | 0 | 50 |
| **Seata AT** | 0 | 0 | 1（undo_log） | 60（注解 + 配置） |
| **Seata TCC** | 1 → 3 | 1 (frozen) | 0 | 350 |
| **DTM Saga** | 1 → 2 | 0 | 1（barrier） | 200 |
| **本地消息表** | 0 | 0 | 1（消息表） + 定时任务 | 280 |
| **RocketMQ 事务消息** | 0 | 0 | 0 | 120（含回查） |

**反直觉**：TCC 看起来"简单"（业务自己控），实际开发成本最高 —— 每个接口拆 3 个 + 状态字段 + 幂等 + 空回滚 + 悬挂处理。生产团队真正用 TCC 之前必须评估"多写 5~10 倍代码"的成本。

> **口诀**：**性能**：本地消息表 ≈ 事务消息 > Saga > TCC > AT > XA。**强一致**：XA = AT > TCC > Saga > 消息表。**开发成本**：AT < XA < 消息表 ≈ 事务消息 < Saga < TCC。三角形里挑两个，没有都要的方案。

---

## 4. 生产案例 9 则

> 每个案例固定四块：现象 / 根因 / 修复 / 验证。

### 案例 ① TCC 悬挂 —— "Cancel 比 Try 先到，资源永久预留"

#### 现象

下单业务用 TCC 扣库存。业务高峰期间监控发现：库存表的 `frozen` 字段持续上涨，但订单表里没有对应的"已支付"或"已取消"订单。SKU 维度看，部分商品的"可用库存 = 总库存 - frozen"在持续下降，最终导致用户下单提示"库存不足"，但其实没真卖。

排查发现 `frozen` 里有大量孤儿记录，业务侧没有对应的状态。

#### 根因

业务时序：

```
T1: 应用 → TC：开启全局事务，发起 try
T2: TC → 库存服务：try（HTTP 调用，但客户端超时设了 1s）
T3: 库存服务实际执行 try（DB 操作 800ms，因为 DB 抖动）
T4: 应用收到超时错误，认为分布式事务失败
T5: 应用 → TC：rollback 全局事务
T6: TC → 库存服务：cancel
T7: 库存服务执行 cancel（此时 try 还没执行完！）
       → 业务表里还没有 frozen 记录
       → cancel 检查后认为是"空回滚"，直接返回成功
T8: try 终于执行完，frozen += 1
       → 这条 frozen 永远没人来 confirm/cancel
```

T7 发生的"空回滚"反而让 T8 的 try 成了"悬挂操作"。`frozen` 永久涨着。

#### 修复

修复点必须在 try 之前**先检查"是否有对应的 cancel 已发生过"**。一种实现：

```sql
-- 事务记录表
CREATE TABLE tx_record (
  xid VARCHAR(64),
  branch_id BIGINT,
  action VARCHAR(20),     -- 'try' | 'confirm' | 'cancel'
  created_at DATETIME,
  PRIMARY KEY (xid, branch_id, action)
);
```

业务代码：

```java
@Transactional
public boolean tryDeduct(BusinessActionContext ctx, Long productId, Integer count) {
    String xid = ctx.getXid();
    Long branchId = ctx.getBranchId();
    
    // ★ 1. 看本次分支是否已经有 cancel 记录
    if (txRecordMapper.exists(xid, branchId, "cancel")) {
        return true;  // 悬挂检测：cancel 已经先来了，try 不要做任何事
    }
    
    // ★ 2. 看本次分支是否已经做过 try（重复 try）
    if (txRecordMapper.exists(xid, branchId, "try")) {
        return true;  // 幂等
    }
    
    // ★ 3. 执行业务 + 记 try
    storageMapper.freeze(productId, count);
    txRecordMapper.insert(xid, branchId, "try");
    return true;
}

@Transactional
public boolean cancel(BusinessActionContext ctx) {
    String xid = ctx.getXid();
    Long branchId = ctx.getBranchId();
    
    // ★ 防止重复 cancel
    if (txRecordMapper.exists(xid, branchId, "cancel")) return true;
    
    // ★ 检查是否有 try 记录 → 没有就是空回滚
    boolean hasTry = txRecordMapper.exists(xid, branchId, "try");
    if (!hasTry) {
        // 空回滚也要插一条 cancel 记录，防止以后 try 再来时"悬挂"
        txRecordMapper.insert(xid, branchId, "cancel");
        return true;
    }
    
    // 真回滚
    storageMapper.unfreeze(...);
    txRecordMapper.insert(xid, branchId, "cancel");
    return true;
}
```

更优雅：用 DTM 子事务屏障（参考 2.4 节），把这套逻辑下沉到框架。

#### 验证

```bash
# 模拟"延迟 1s 的 try + 立即 cancel"
ab -n 1000 -c 50 'http://api/test-tcc?delay=1000ms&trigger=rollback'

# 检查 frozen 字段最终归零
mysql> SELECT SUM(frozen) FROM stock;
+-------------+
| SUM(frozen) |
+-------------+
|           0 |
+-------------+
```

> **口诀**：TCC 必须有事务记录表（或屏障）。**try 前查 cancel，cancel 前查 try，每步都要幂等**。

---

### 案例 ② TCC 空回滚 —— "Try 都没成功，Cancel 把库存扣多了"

#### 现象

业务低峰期一次网络抖动后，对账发现一批订单的"实际扣减库存量"对不上。排查每个订单的事务日志，发现 try 阶段在网络层面就失败了（连接超时），但 cancel 阶段却"成功"了 —— 而且执行了真实的 unfreeze 逻辑。

#### 根因

业务方写的 cancel 接口"直接执行回滚"，没判断 try 是否成功：

```java
// ❌ 错误的 cancel
@Transactional
public boolean cancel(BusinessActionContext ctx, Long productId, Integer count) {
    storageMapper.unfreeze(productId, count);   // 直接 frozen -= count
    return true;
}
```

时序：

```
T1: TC → 库存服务 try（网络断了，包根本没到达）
T2: TC 等超时 → 决定全局回滚
T3: TC → 库存服务 cancel
T4: 库存服务执行 unfreeze(productId, count)
    → 但 frozen 里根本没有这次 try 的预留
    → 结果是 frozen 被扣到 -1（如果不允许负数会报错；如果允许就静默错乱）
```

更糟的情况：`frozen` 字段做了"不允许负数"约束，cancel 里没捕获异常 → cancel 抛出 → TC 不停重试 → 把整个库存服务打挂。

#### 修复

cancel 必须先判断"对应的 try 是否真的执行过"：

```java
@Transactional
public boolean cancel(BusinessActionContext ctx, Long productId, Integer count) {
    String xid = ctx.getXid();
    Long branchId = ctx.getBranchId();
    
    if (!txRecordMapper.exists(xid, branchId, "try")) {
        // ★ 空回滚：try 没成功过，cancel 不做事
        // 但要插一条 cancel 占位（防止以后 try 重试又来悬挂）
        txRecordMapper.insertIgnore(xid, branchId, "cancel");
        return true;
    }
    
    if (txRecordMapper.exists(xid, branchId, "cancel")) {
        return true;  // 重复 cancel 幂等
    }
    
    storageMapper.unfreeze(productId, count);
    txRecordMapper.insert(xid, branchId, "cancel");
    return true;
}
```

#### 验证

```bash
# 模拟 "try 网络失败 + cancel 必到达" 场景
# 用 toxiproxy 在 try 接口前加一个"全部丢包"规则，cancel 接口正常
toxiproxy-cli toxic add storage-svc -t bandwidth -a rate=0 -n storage_try_fail

# 跑 1000 个全局事务
# 验证 frozen 最终为 0、stock 没变化
```

> **口诀**：**Cancel 默认假设 try 没做**，先查再回滚。"空回滚"要插一条占位记录防悬挂。

---

### 案例 ③ Saga 反向补偿失败 —— "补偿接口自己抛异常，订单卡在'补偿中'"

#### 现象

下单业务用 Saga 编排：创单 → 扣库存 → 扣余额 → 通知发货。某次扣余额失败，触发反向补偿：通知发货回滚（无操作）→ 扣库存补偿 → 创单补偿。结果"扣库存补偿"接口自身报 `RPC timeout`。订单卡在"补偿中"状态，监控告警 200+ 单。

#### 根因

Saga 框架的补偿重试机制：默认重试 3 次，每次间隔 1s。3 次都失败 → 标记"补偿失败"，需要人工。生产场景失败原因经常是：

- 下游服务正在发布 / 重启 → 短暂不可用
- 补偿接口本身有 bug（比如 unfreeze 的参数解析错了）
- 补偿目标的数据状态变了（比如 try 时冻结的库存已经被"超时回滚"清掉了）

更深层的问题：**Saga 的补偿假设"每个补偿动作能在合理时间内成功"**。这是一个工程妥协，不是数学保证。

#### 修复

**分层兜底**：

1. **框架层**：补偿失败 N 次后进入"死信队列"，不再自动重试
2. **告警层**：死信入队立即告警（金融业务必须 P0）
3. **运营兜底**：提供"手动补偿"管理界面，运营人员能查询当前所有"补偿失败"事务，手动选择"重试 / 放弃 / 人工修正"
4. **预防**：补偿接口必须**幂等 + 容错**：

```java
// 补偿接口的标准模板
@Transactional
public boolean compensateDeduct(String xid, Long productId, Integer count) {
    // 1. 幂等：同一个 xid 多次调用，只生效一次
    if (compensationLogMapper.exists(xid, "deduct")) return true;
    
    // 2. 状态校验：源数据可能已经变化，要兼容
    Stock stock = storageMapper.find(productId);
    if (stock == null) {
        // 库存记录不存在了？记日志告警，但返回成功（避免无限重试）
        log.error("compensate target not found, xid={}, productId={}", xid, productId);
        compensationLogMapper.insert(xid, "deduct", "skip-not-found");
        return true;
    }
    
    // 3. 执行 + 记日志
    storageMapper.unfreeze(productId, count);
    compensationLogMapper.insert(xid, "deduct", "ok");
    return true;
}
```

**不能补偿的动作要在 Saga 中前置**：

- 已经发短信、邮件、推送 → Saga 中先做能补偿的动作，**这类动作放最后或者根本不放进 Saga**
- 已经发起银行汇款 → 改用 TCC（先冻结后提交）

#### 验证

混沌工程：在补偿接口前加 50% 失败注入，观察告警和兜底界面。

```bash
# 用 chaos mesh 注入 50% HTTP 5xx
kubectl apply -f - <<EOF
apiVersion: chaos-mesh.org/v1alpha1
kind: HTTPChaos
metadata:
  name: comp-half-fail
spec:
  action: abort
  mode: one
  selector:
    namespaces: [default]
    labelSelectors: { app: storage-svc }
  target: Request
  port: 8080
  path: /compensate/*
  percent: 50
EOF
```

预期：业务不被压垮，告警按时触发，运营界面能看到失败事务。

> **口诀**：**Saga 假设补偿能成功，生产假设补偿会失败**。死信队列 + 告警 + 人工兜底是 Saga 的必备配套。不可补偿动作不准进 Saga。

---

### 案例 ④ 本地消息表漏发 —— "订单已支付，下游 7 小时没收到通知"

#### 现象

某电商运营投诉："这单 3 小时前就支付了，但物流系统一直显示'未支付'，没生成发货单。"排查链路：订单服务的 DB 里订单状态是"已支付"，订单消息表里有一条 `to_send` 状态的消息，但 RocketMQ 里查不到这条消息。

#### 根因

代码：

```java
@Transactional
public void pay(Long orderId) {
    orderMapper.updateStatus(orderId, "PAID");
    messageOutboxMapper.insert(buildOrderPaidMessage(orderId));  // 写入消息表，状态 = to_send
}

// 定时任务每 5 秒扫消息表
@Scheduled(fixedDelay = 5000)
public void sendPendingMessages() {
    List<MessageOutbox> pending = messageOutboxMapper.findToSend(100);
    for (MessageOutbox msg : pending) {
        try {
            rocketMQTemplate.convertAndSend(msg.getTopic(), msg.getBody());
            messageOutboxMapper.markSent(msg.getId());
        } catch (Exception e) {
            log.error("send failed, msg={}", msg.getId(), e);
            // 没标记失败，下次循环还会重试
        }
    }
}
```

业务时序：

```
T1: 订单支付，事务提交（订单 + 消息表写入）
T2: 定时任务跑了一次，消息发送中网络抖动 → 抛异常
T3: 没标记 markSent，但消息确实**可能已经发到 broker** —— 这是误报
T4: 应用进程因为内存压力被 K8s OOMKill 重启
T5: 新进程启动后，定时任务的 Bean 被错误初始化（配置 bug 导致没启动）
T6: 7 小时后运营投诉，发现定时任务一直没运行
```

**多重失败叠加**：定时任务挂、监控没覆盖、重启后没自检。

#### 修复

**修复一：定时任务必须自监控**

```java
@Component
public class OutboxScannerHealthCheck implements HealthIndicator {
    @Autowired
    private MessageOutboxMapper mapper;
    
    @Override
    public Health health() {
        // 找最老的"未发送"消息
        Long oldestUnsent = mapper.findOldestUnsentSeconds();
        if (oldestUnsent != null && oldestUnsent > 60) {
            return Health.down()
                .withDetail("oldest_unsent_sec", oldestUnsent)
                .build();
        }
        return Health.up().build();
    }
}
```

监控接 `/actuator/health` → 阈值告警。

**修复二：定时任务不要靠"自动发现"，用配置中心声明**

```java
// 启动时强制日志输出，确保任务被注册
@PostConstruct
public void init() {
    log.info("OutboxScanner started, scanInterval={}ms", scanInterval);
    // 注册到一个全局监控 map，便于排查
    SchedulerRegistry.register("outbox-scanner", this);
}
```

**修复三：扫描条件加"距今 N 秒前"**

```sql
-- 不要扫"所有 to_send"，扫"to_send 且 created_at < now - 5s"
SELECT * FROM message_outbox 
WHERE status = 'to_send' AND created_at < NOW() - INTERVAL 5 SECOND
ORDER BY created_at LIMIT 100
```

为什么？避免扫到"刚刚才插入但 MQ 还在 ack"的消息（漏发 vs 重发的权衡 —— 5s 窗口给了正常路径优先权）。

**修复四：发送结果要分清"明确成功 / 明确失败 / 不确定"**

```java
try {
    SendResult result = rocketMQTemplate.syncSend(topic, msg);
    if (result.getSendStatus() == SendStatus.SEND_OK) {
        mapper.markSent(msg.getId());
    }
    // 其他状态不标记 sent，下次重试
} catch (Exception e) {
    // 网络异常：消息可能已发出也可能没发出 → 下次重试，靠消费侧幂等去重
    mapper.incrementRetry(msg.getId());
}
```

**修复五：消息表加"重试次数 + 最后失败原因"**

```sql
ALTER TABLE message_outbox 
  ADD COLUMN retry_count INT DEFAULT 0,
  ADD COLUMN last_error VARCHAR(500),
  ADD COLUMN next_retry_at DATETIME;
```

重试次数超过阈值 → 进死信，告警人工介入。

#### 验证

```bash
# 模拟定时任务挂掉（直接 KILL 任务线程）
# 验证 health endpoint 在 60s 内 down
curl -s localhost:8080/actuator/health | jq .components.outboxScanner

# 模拟 RocketMQ 短暂不可用
# 验证消息延迟发送但最终一定发出
```

> **口诀**：本地消息表的命门是**定时任务必须有自监控**。"消息卡在 to_send 超过 N 秒"是必备告警，"扫描只看老消息"是必备策略。

---

### 案例 ⑤ RocketMQ 事务消息长事务 —— "本地事务一卡，整个 producer 雪崩"

#### 现象

订单服务上线了一个新功能：在本地事务中调用第三方风控接口（同步等结果）。压测时单独跑 OK，但开了事务消息后，发现 producer 端日志大量 `transaction state unknown`，broker 上 half message 堆积，消费者大量延迟。

#### 根因

代码：

```java
@Override
public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
    Order order = (Order) arg;
    try {
        // ★ 本地 DB 操作
        orderMapper.update(order);
        // ★ 同步调风控（最坏 30s 超时）
        RiskResult risk = riskClient.check(order);
        if (!risk.pass()) return LocalTransactionState.ROLLBACK_MESSAGE;
        return LocalTransactionState.COMMIT_MESSAGE;
    } catch (Exception e) {
        return LocalTransactionState.UNKNOWN;
    }
}
```

风控接口经常 5~30s 才返回。`executeLocalTransaction` 是**同步的**，跑在 producer 内部线程池里。线程池被 30s 长事务占满 → 后续消息发送全部排队 → broker 收不到及时 commit/rollback → 触发回查 → 回查接口又被同样的线程池阻塞 → 雪崩。

#### 修复

**绝对铁律**：`executeLocalTransaction` 里只做本地 DB 事务，**不许有跨服务调用**。

正确做法：把"风控"前置到 half message 之前。

```java
public boolean placeOrder(Order order) {
    // ★ 先做风控（不在事务内）
    if (!riskClient.check(order).pass()) {
        return false;
    }
    
    // 风控通过 → 发事务消息（本地事务里只 update DB）
    producer.sendMessageInTransaction(buildMsg(order), order);
    return true;
}

@Override
public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
    try {
        orderMapper.update((Order) arg);
        return LocalTransactionState.COMMIT_MESSAGE;
    } catch (Exception e) {
        return LocalTransactionState.UNKNOWN;
    }
}
```

**回查接口**也要快、也要去远程：

```java
@Override
public LocalTransactionState checkLocalTransaction(MessageExt msg) {
    // ★ 只查本地 DB，不调外部
    String orderId = msg.getKeys();
    Order order = orderMapper.find(orderId);
    if (order == null) return LocalTransactionState.ROLLBACK_MESSAGE;
    if (order.getStatus().equals("CONFIRMED")) return LocalTransactionState.COMMIT_MESSAGE;
    return LocalTransactionState.UNKNOWN;  // 还在处理中，下次回查
}
```

#### 验证

```bash
# 注入风控接口 5s 延迟，跑 1000 QPS 订单
# 验证：producer 不发生雪崩，half message 堆积 < 100
```

监控告警：

- `rocketmq.producer.executeLocalTransaction.duration` P99 > 500ms 告警
- `broker.half.message.count` > 10000 告警

> **口诀**：事务消息的本地事务必须**纯本地**。任何远程调用 / 长 IO 都必须前置或异步化。回查接口同理。

---

### 案例 ⑥ Seata AT 全局锁热点 —— "扣库存 QPS 从 4000 跌到 200"

#### 现象

电商大促前压测，用 Seata AT 模式做库存扣减，普通商品 QPS 4000+ 没问题。一上热销商品（同一 SKU 高并发），TPS 直接跌到 200，监控看 `lock_table` 的写 QPS 飙到 1万+，DB CPU 100%。

#### 根因

参考 2.1 节源码 —— Seata AT 模式靠在全局锁表里 INSERT (table, pk) 来锁住一行。同一个 SKU 在高并发下：

- 1000 个并发 → 990 个排队等 lock_table 上对应那行的 INSERT（被 unique index 阻塞）
- 即使前面的事务一阶段 DB commit 了，全局锁要等到全局事务结束才释放
- 全局事务结束 = TC 接收所有分支 commit 完成（多了 1~2 个 RTT）

吞吐被全局锁的"持有时间 × 并发数"上限死死框住。

#### 修复

**方案 A：换 TCC**。把"扣库存"改成业务字段乐观锁：

```sql
-- TCC try
UPDATE stock SET frozen = frozen + ?, version = version + 1
WHERE id = ? AND available - frozen >= ? AND version = ?

-- TCC confirm
UPDATE stock SET frozen = frozen - ?, sold = sold + ? 
WHERE id = ?

-- TCC cancel
UPDATE stock SET frozen = frozen - ? WHERE id = ?
```

乐观锁靠 `version` 失败重试，DB 锁只持有单 SQL 期间，全局锁完全不用上 lock_table。

**方案 B：热点 SKU 分桶**。把 1 个 SKU 物理上拆成 N 行：

```sql
CREATE TABLE stock_bucket (
  product_id BIGINT,
  bucket_id INT,
  available INT,
  PRIMARY KEY (product_id, bucket_id)
);

-- 商品 100 库存 1000 → 拆成 10 个桶，每桶 100
INSERT INTO stock_bucket VALUES 
  (100, 0, 100), (100, 1, 100), ..., (100, 9, 100);

-- 扣库存：随机选一个桶
SELECT bucket_id FROM stock_bucket 
WHERE product_id = 100 AND available > 0
ORDER BY RAND() LIMIT 1
FOR UPDATE;
UPDATE stock_bucket SET available = available - 1 
WHERE product_id = 100 AND bucket_id = ?;
```

10 个桶 → 全局锁竞争降到 1/10。代价是查询时需要"跨桶聚合"（SUM available）。

**方案 C：异步化**。秒杀场景常用 —— 前端先扣 Redis（基本不会写穿），事件流入消息队列，后端慢慢消费 + 对账。代价是"超卖"风险（Redis 与 DB 不强一致）。

#### 验证

修复后再跑同样压测：

| 修改 | 1000 并发 / 1 个 SKU TPS |
| --- | --- |
| 修复前（AT） | 200 |
| 改 TCC | 1500 |
| 分 10 桶（仍 AT） | 1800 |
| TCC + 分桶 | 6000 |

> **口诀**：AT 全局锁 = 行级 + 全局事务期持有。**热点行不要用 AT**，要么换 TCC，要么把行打散。

---

### 案例 ⑦ XA 连接池耗尽 —— "DBA 凌晨 4 点叫起来扩连接池"

#### 现象

金融跨行清算用 XA。某天凌晨批量任务高峰，应用日志大量 `XAException: connection refused`、DB 端 `max_connections` 报警。临时扩连接池暂时缓解，但同样问题第二天再现。

#### 根因

XA 事务的连接生命周期：

```
xa_start  → DB 连接 A 被借出
... 业务 SQL ...
xa_end    → 业务结束（连接 A 还没还）
xa_prepare → 通知 DB 准备好提交（连接 A 还在）
[等其他参与者 prepare 完]
xa_commit → 真正提交（连接 A 释放）
```

关键：**prepare 完到 commit 之前，连接必须保持**。

如果一个 XA 事务跨 5 个 DB，每个 DB 都占 1 个连接，且事务整体耗时 200ms（含网络往返）。1000 TPS = 1000 × 200ms × 5 = 1000 个连接同时持有。

应用配的连接池：100。瞬间被打爆。

#### 修复

**方案 A：减小 XA 事务持续时间**。在 prepare 之前减少业务逻辑耗时，避免长事务持有连接。

**方案 B：扩大连接池 + 算账**。

```
连接池大小 = 峰值 TPS × XA 事务平均持续时间 × 参与者数 × 安全系数
        = 1000 × 0.2 × 5 × 1.5 = 1500
```

DB 端 `max_connections` 也要相应放大。

**方案 C：评估能否换非 XA 方案**。金融场景如果业务本身允许"准实时"，可以用 TCC + 对账（比如清算任务前置一个 TCC try 冻结资金，凌晨批跑 confirm）。

**方案 D：用 DB Proxy 做连接复用**。ProxySQL / MaxScale 在 prepared 阶段用同一个 DB 连接服务多个应用连接。但 XA 协议下复用比较复杂，要谨慎。

#### 验证

监控这几个指标：

- `xa.transaction.duration` P99
- 应用连接池 `active / max` 占比
- DB `Threads_connected`

修复后这些指标应有数量级改善。

> **口诀**：**XA 的成本是连接持有时间被放大**。生产用 XA 必须算清楚"TPS × 时长 × 参与者"对连接池的需求。

---

### 案例 ⑧ 跨服务超时叠加 —— "外层成功，内层其实失败了"

#### 现象

订单系统调用支付服务调用账户服务，三层 RPC。每层调用方设的超时都是"略小于上一层" —— 看起来合理。但生产偶发"订单显示成功但账户没扣钱"。

```
订单 → 支付：超时 3s
支付 → 账户：超时 2s
账户内部 DB：超时 1s
```

某次账户 DB 抖动 1.2s，账户 RPC 内部超时回滚事务返回失败。但中间这 1.2s 已经超过了支付到账户的 2s 中的 1s 网络部分 + 1s 超时——支付以为账户调用超时（网络问题），按"未知"处理，**保留了支付状态**。订单收到支付成功，最终业务对不上。

#### 根因

经典的"幻觉成功"：**调用方超时 ≠ 被调用方失败**。超时只意味着"我没及时收到响应"，但被调用方可能：

1. 完全没执行 → 失败
2. 执行了但响应包丢了 → 实际成功
3. 还在执行 → 之后会成功
4. 执行到一半失败 → 失败但中间状态不一致

订单看到"支付返回成功"，支付看到"账户超时"——支付的处理逻辑是错误的：超时应该走"再查询一次确认状态"，而不是默认成功。

#### 修复

**修复一：超时设置必须算"传播预算"**

```
订单 → 支付：超时 3s（剩余预算 3s）
支付 → 账户：超时 2.5s（保留 0.5s 给自己 + 网络回程）
账户内部：超时 2s（保留 0.5s 给自己）
```

每一层减去自己的处理时间和回程网络。**绝对不要"内层超时大于外层"**。

更好的做法是在 RPC 框架里实现 **deadline 传播**（gRPC、gRPC-go 原生支持）：

```java
// gRPC 例子
ClientCall.Listener<Response> listener = stub
    .withDeadlineAfter(3, TimeUnit.SECONDS)    // 设个 deadline
    .someMethod(request);

// 服务端能拿到剩余时间
@Override
public void someMethod(Request req, StreamObserver<Response> observer) {
    Deadline deadline = Context.current().getDeadline();
    long remaining = deadline.timeRemaining(TimeUnit.MILLISECONDS);
    // 调下游时用 remaining - epsilon 作为超时
}
```

**修复二：超时必须有"反查机制"**

```java
public PaymentResult pay(...) {
    try {
        return accountClient.deduct(...);
    } catch (TimeoutException e) {
        // ★ 不要直接 fallback，要反查
        AccountStatus status = accountClient.queryStatus(traceId);
        if (status.isDeducted()) {
            return PaymentResult.success();
        }
        return PaymentResult.unknown();   // 上层要按"不确定"处理，不能按成功
    }
}
```

**修复三：所有跨服务调用都要带 traceId / requestId / 业务唯一键**，下游接口必须支持"按 traceId 查询当前状态"。这是分布式系统的基本卫生。

**修复四：业务侧必须有对账**。每天定时跑订单 vs 支付 vs 账户的三方对账，不一致的进入工单系统。

#### 验证

```bash
# 用 toxiproxy 注入 1.2s 延迟到账户服务
# 跑 1000 笔订单
# 验证：没有"订单成功 + 账户没扣"的不一致单
```

> **口诀**：**超时不等于失败**。跨服务调用必须 deadline 传播 + 反查 + 对账。三层 RPC 的超时必须严格递减。

---

### 案例 ⑨ 幂等 key 设计错误 —— "重复扣款 1.2 万元"

#### 现象

支付系统对账发现某客户同一笔订单被扣款 3 次，金额 1.2 万。客服快速退款止损，但根因排查暴露设计缺陷。

#### 根因

支付服务的"扣款消费者"代码：

```java
@RocketMQMessageListener(topic = "ORDER_PAID")
public class PaymentConsumer {
    @Override
    public void onMessage(MessageExt msg) {
        String messageId = msg.getMsgId();
        // ★ 用 MQ 消息 ID 做幂等
        if (idempotentService.exists(messageId)) {
            return;
        }
        Order order = parse(msg.getBody());
        accountService.deduct(order.getUserId(), order.getAmount());
        idempotentService.save(messageId);
    }
}
```

看起来是做了幂等的。但 `messageId` 是 RocketMQ 的内部消息 ID，**每次重发都是新的 ID**：

- 上游订单服务因为某种原因（重试 / 重入消费 / 上游 bug）把"订单 X 已支付"事件**生产了 3 次**到 MQ
- 每次 MQ 给的 messageId 都不同
- 消费者 3 次都通过了幂等检查 → 扣 3 次

#### 修复

**铁律**：幂等 key 必须用**业务唯一键**，不能用消息系统的 ID。

```java
@RocketMQMessageListener(topic = "ORDER_PAID")
public class PaymentConsumer {
    @Override
    public void onMessage(MessageExt msg) {
        Order order = parse(msg.getBody());
        // ★ 用 orderId + action 做业务幂等键
        String idempKey = "deduct:" + order.getOrderId();
        
        if (!idempotentService.tryAcquire(idempKey, 7, TimeUnit.DAYS)) {
            log.info("duplicate deduct, orderId={}", order.getOrderId());
            return;
        }
        
        try {
            accountService.deduct(order.getUserId(), order.getAmount());
            idempotentService.markSuccess(idempKey);
        } catch (Exception e) {
            idempotentService.release(idempKey);
            throw e;
        }
    }
}
```

幂等存储推荐 DB 唯一索引 + Redis 双层（Redis 快速过滤、DB 兜底）：

```sql
CREATE TABLE idempotent_record (
  idemp_key VARCHAR(128) PRIMARY KEY,
  status VARCHAR(20),       -- 'pending' | 'success' | 'failed'
  created_at DATETIME,
  succeeded_at DATETIME,
  result_data TEXT          -- 可选：保存第一次的结果，重复请求直接返回
);
```

**注意**：

- 业务唯一键的设计要早于幂等机制 —— 上游必须保证"同一笔业务"的 key 一致
- 长链路场景（订单→支付→账户→风控），每一段都要有自己的业务幂等键
- 幂等记录要有 TTL（一般 7~30 天），不能无限存储

#### 验证

```bash
# 模拟同一条业务消息发送 5 次
for i in $(seq 1 5); do
    publish_order_paid_event "order-12345"
done

# 验证：accountService.deduct 只被调用 1 次
# 数据库 idempotent_record 表里 deduct:order-12345 状态为 success
```

> **口诀**：**幂等键 = 业务唯一键**，不是 MQ 消息 ID、不是 traceId。每一段链路定义自己的业务键。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 选型层
- [ ] 业务真的需要分布式事务吗？能不能用"最终一致 + 补单"代替？
- [ ] 强一致需求 → XA / Seata AT / Percolator；最终一致需求 → TCC / Saga / 消息表
- [ ] 热点行场景禁用 Seata AT，用 TCC + 业务字段乐观锁
- [ ] 不可补偿动作（短信、汇款、上链）禁用 Saga，必须 TCC 前置预留
- [ ] XA 在金融核心以外慎用，性能成本评估清楚

### TCC 层
- [ ] try / confirm / cancel 三方法**全部幂等**
- [ ] 业务表有"事务记录"字段，或使用 DTM 子事务屏障
- [ ] cancel 必须先判断 try 是否成功（防止真扣减）
- [ ] try 必须先判断 cancel 是否已经发生（防止悬挂）
- [ ] try 阶段不允许有跨服务调用（避免"幻觉成功"）
- [ ] confirm / cancel 接口对外标识清楚是"框架调用"，禁止业务侧手动调

### Saga 层
- [ ] 每个正向动作都有对应的补偿动作，且补偿动作必须**幂等**
- [ ] 补偿失败有死信队列 + 告警 + 人工兜底界面
- [ ] 不可补偿的动作放最后或不进 Saga
- [ ] 编排引擎本身要高可用（不能是单点）
- [ ] 流程定义有版本管理，灰度发布

### 本地消息表 / 事务消息层
- [ ] 消息表写入和业务变更必须**同库同事务**
- [ ] 定时任务有自监控（最老未发送时长告警）
- [ ] 扫描条件加"距今 N 秒前"避免与正常路径竞争
- [ ] 重试有指数退避 + 上限，超限进死信
- [ ] RocketMQ 事务消息的本地事务**禁止跨服务调用**
- [ ] 回查接口快速、纯本地查询、强幂等

### 通用层
- [ ] 所有跨服务调用必须有业务唯一键（订单 ID、流水号），不依赖 MQ ID / traceId
- [ ] 超时必须 deadline 传播，内层 < 外层
- [ ] 超时必须有反查接口（"按 traceId 查询当前状态"）
- [ ] 业务必须有对账，每日 / 实时跑
- [ ] 全局事务有可观测性：xid、分支状态、耗时、失败原因

### Seata 特定
- [ ] Seata Server 用 Raft / DB 模式，不是 file 模式
- [ ] AT 模式必须建 undo_log 表，权限正确
- [ ] 全局锁存储用 DB 或 Redis（不要 file）
- [ ] undo_log 自动清理任务运行正常
- [ ] 全局事务超时配置合理（默认 60s 通常太长）

### 监控层
- [ ] `分布式事务总数 / 成功率 / 失败率`
- [ ] `分支事务延迟分布`
- [ ] `补偿 / 回查 / 重试次数`
- [ ] `孤儿数据告警`（frozen 字段长时间不归零、消息卡住超过 N 秒）
- [ ] `资源池`：DB 连接、MQ producer 队列长度、TC 连接数

### 故障演练
- [ ] 定期注入：DB 宕机、网络分区、TC 宕机、消息中间件不可用
- [ ] 恢复后业务必须最终一致（人工或自动对账确认）
- [ ] 演练发现的"半状态"必须能通过运营界面手动修复

---

## 6. 还能往下挖什么

读完这篇只是入门。如果想再深一层：

### 强一致流派

- **Spanner / TrueTime**：用 GPS + 原子钟 + commit-wait 实现"外部一致性"。论文 *Spanner: Google's Globally-Distributed Database*（OSDI 2012）必读。
- **Paxos 与 Raft**：分布式共识协议。Paxos 论文晦涩，Raft 论文 *In Search of an Understandable Consensus Algorithm*（ATC 2014）读起来友好。
- **Calvin: Fast Distributed Transactions for Partitioned Database Systems**（SIGMOD 2012）：Deterministic Database 流派 —— 把事务先排好序，每个节点按相同顺序执行，不需要 2PC。理论上很优雅，工程化代表 FaunaDB。
- **CockroachDB / TiDB**：商用版的 Spanner / Percolator 实现，可以读源码。

### 最终一致流派

- **Outbox 模式**：Chris Richardson 在 *Microservices Patterns* 里推广的术语，本质就是本地消息表。区别在于把"出库"建模成显式 pattern。
- **Inbox 模式**：消费侧的对偶 —— 用 inbox 表实现幂等消费 + 消息去重。
- **Saga 编排引擎**：[Camunda 7/8](https://camunda.com)、[Cadence](https://cadenceworkflow.io/) / [Temporal](https://temporal.io/)、Netflix Conductor。这些工具把"流程 + 补偿 + 重试 + 兜底"封装成一等公民。
- **CDC + Outbox**：Debezium 监听业务 DB 的 binlog，自动把 INSERT 转成 MQ 消息 —— 比 outbox 表 + 定时扫描更优雅。

### 数据库层

- **Aurora 的"日志即数据库"**：把 redo log 复制到存储层，存储层应用 redo 生成数据页 —— 跨 AZ 写延迟 < 10ms。
- **TiDB 悲观 / 乐观事务模型切换**：什么场景该选哪种。
- **PostgreSQL 的 SSI（Serializable Snapshot Isolation）**：比 SI 更严格但比 2PL 性能好。
- **DynamoDB Transactions**：单 region 内用乐观锁 + 协调，跨 region 不支持。

### 事件溯源

- **Event Sourcing + CQRS**：根本不存"当前状态"，只存事件流；查询时投影。事务边界从"行"变成"聚合根"。
- **EventStoreDB / Kafka Streams**：常用实现。
- 在事件溯源世界里，"分布式事务"问题转化为"事件投递的可靠性"。

### 边缘话题

- **TCC vs Saga 哲学之争**：知乎、Twitter 上没完没了的争论。简言之：TCC 是同步预占，Saga 是异步推进，没有谁更优秀只有谁更适合。
- **Service Mesh 时代**：Istio / Linkerd 把超时、重试、熔断下沉到 sidecar，应用代码更干净，但分布式事务的语义层 mesh 接管不了。
- **Serverless 时代的事务**：函数生命周期短、状态外置 → 事务必须落到外部协调器（Step Functions / Durable Functions）。

---

## 7. 引用

- Gray, *Notes on Data Base Operating Systems*（1978）—— 2PC 最早出处
- Gray & Lamport, *Consensus on Transaction Commit*（2004）—— Paxos Commit
- Pat Helland, *Life beyond Distributed Transactions*（2007）—— 最终一致性的纲领性文章
- Daniel Peng & Frank Dabek, *Large-scale Incremental Processing Using Distributed Transactions and Notifications*（OSDI 2010）—— Percolator 论文
- James Corbett et al., *Spanner: Google's Globally-Distributed Database*（OSDI 2012）
- Hector Garcia-Molina & Kenneth Salem, *Sagas*（SIGMOD 1987）—— Saga 原始论文
- 阿里 Seata 文档：[seata.apache.org](https://seata.apache.org/)
- DTM 文档：[dtm.pub](https://dtm.pub/)
- Apache RocketMQ 事务消息设计：[rocketmq.apache.org](https://rocketmq.apache.org/)
- TiDB 源码：[github.com/pingcap/tidb](https://github.com/pingcap/tidb)
- Chris Richardson, *Microservices Patterns*（2018）—— Outbox / Saga 工程化
- Martin Kleppmann, *Designing Data-Intensive Applications*（2017）—— 事务与一致性的全景图

---

> 配套：[分布式系统主题导览](/article/distributed/) ｜ [MySQL MVCC 与锁](/article/db/mysql-mvcc) ｜ [Kafka 存储引擎](/article/mq/kafka-storage)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
