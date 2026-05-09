# MySQL MVCC 与锁 —— 从可见性算法到死锁复盘

> 这是按 [桃花岛深度模板](/STYLE) 写的第二篇深文。如果你只想快速复习面试，去 [MySQL 速记](/article/db/mysql) 那一节够了。
> 这里要回答的，是**写 SQL 遇到怪现象、半夜被死锁告警叫起来**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过：

1. RR 隔离级别下，事务 A 第一次 SELECT 看到 100 行，事务 B 在中间提交了一条 INSERT，事务 A 第二次 SELECT 还是 100 行 —— 这叫**幻读被阻止了**。但 RR 真的能完全阻止幻读吗？给一个能在 RR 下复现幻读的 SQL 序列。
2. `SELECT * FROM t WHERE id = 1` 和 `SELECT * FROM t WHERE id = 1 FOR UPDATE`，**前者是不是不加锁**？两者读到的数据可能不同吗？
3. RC 和 RR 都用 MVCC，**唯一的差别**是什么？用一句话回答。
4. `UPDATE t SET v = v + 1 WHERE id = 1` 在 RR 下，能不能读到其他事务**已提交的**最新 v 值？
5. 二级索引上的 `WHERE name = 'tom'`，在 RR 下用 `FOR UPDATE`，会锁住哪些范围？给出 Next-Key Lock 的精确边界。
6. 有人说"长事务会让 SELECT 变慢"，**机理**是什么？哪个内部数据结构在膨胀？怎么观测？
7. `INSERT INTO t (id, name) VALUES (5, 'x')` 在 id=5 已被另一个未提交事务锁住时，会等什么锁？为什么这个锁的名字这么奇怪？
8. 主从复制下，主库一个事务在 RR 下看到的快照，从库 SQL 线程回放时**看到的是同一个快照吗**？binlog 是 ROW 和 STATEMENT 时分别有什么差异？

如果有半数答不上，往下看。

---

## 1. 底层模型：隐藏列、undo log 链、ReadView

绝大多数文章把 MVCC 讲成"读不阻塞写、写不阻塞读"，然后画一个 undo log 链就结束了。这层认知不够诊断生产问题。要看懂上面 8 个问题，必须下沉到 InnoDB 的三个核心数据结构。

### 1.1 InnoDB 行格式中的三个隐藏列

每一行 InnoDB 数据除了用户列，还藏着 3 列：

| 隐藏列 | 长度 | 含义 |
| --- | --- | --- |
| `DB_ROW_ID` | 6 字节 | 没有主键时 InnoDB 自己生成的行 ID（聚簇索引隐式主键） |
| **`DB_TRX_ID`** | 6 字节 | **最近修改这行的事务 ID** |
| **`DB_ROLL_PTR`** | 7 字节 | **指向 undo log 的指针**，串成版本链 |

只有 `DB_TRX_ID` 和 `DB_ROLL_PTR` 与 MVCC 直接相关。它们是 MVCC 整套机制的物理基础。

### 1.2 undo log 版本链

行的"历史版本"不会原地保留，而是放在 undo log 里。每次更新一行，InnoDB 做的事是：

```
修改前：[id=1, v=10, trx=100, roll_ptr=NULL]

事务 200 把 v 改成 11：
   1. 把"原行"完整拷贝到 undo log，得到一个 undo 记录 U1
   2. 修改主表行：[id=1, v=11, trx=200, roll_ptr=→U1]
   3. U1 自己也带 trx 和 roll_ptr，指向更老的版本

事务 300 把 v 改成 12：
   1. 拷贝当前主表行（v=11, trx=200, →U1）到 undo log，得到 U2
   2. 修改主表行：[id=1, v=12, trx=300, roll_ptr=→U2]
```

形成一条版本链：

```
主表 [v=12, trx=300] → U2 [v=11, trx=200] → U1 [v=10, trx=100] → ∅
```

读老版本 = 沿 `roll_ptr` 回溯，直到找到一个"对当前事务可见"的版本。

**两类 undo log**（生产排查必须区分）：
- **insert undo**：事务提交后立即可丢弃，因为 INSERT 的"老版本"对其他事务无意义
- **update undo**：必须保留到**没有任何活跃事务可能读到这个老版本**为止

后者是后面"长事务把 history list 撑爆"的根。

### 1.3 ReadView：可见性判定的快照

ReadView 是事务"在某个瞬间看到的世界快照"。它不是数据快照（数据还在主表+undo 链里），而是**当时活跃事务的列表**。

```cpp
// InnoDB 8.0 read0read.h 关键字段
class ReadView {
  trx_id_t  m_low_limit_id;     // 当前最大已分配 trx_id + 1（"未来"的边界）
  trx_id_t  m_up_limit_id;      // 活跃事务列表里最小的 trx_id
  trx_id_t  m_creator_trx_id;   // 创建这个 ReadView 的事务自己
  ids_t     m_ids;              // 创建瞬间所有"活跃"事务 ID 的列表（已排序）
};
```

可见性判定（伪码）：

```
判定 row 的 trx_id 对当前 ReadView 是否可见：

if trx_id == creator_trx_id:        # 自己改的，看得到
    return VISIBLE
if trx_id < up_limit_id:            # 比"最小活跃"还小，说明已提交，看得到
    return VISIBLE
if trx_id >= low_limit_id:          # 比"最大已分配"还大，说明是未来的事务，看不到
    return INVISIBLE
if trx_id ∈ m_ids:                  # 在活跃列表里，说明 ReadView 创建时它还没提交，看不到
    return INVISIBLE
else:                               # 在 (up_limit, low_limit) 区间但不在活跃列表
    return VISIBLE                  # 说明 ReadView 创建时它已提交
```

**RC 与 RR 的唯一差别**就在这里：
- **RC**：每个 SELECT 语句都新建一个 ReadView
- **RR**：事务首次 SELECT 创建 ReadView，整个事务复用

这两条规则解释了 90% 的 MVCC 行为差异。

### 1.4 一致性读 vs 当前读

| 类型 | 何时使用 | 数据来源 |
| --- | --- | --- |
| **一致性读 (snapshot read)** | 普通 `SELECT` | 走 ReadView + undo 链 |
| **当前读 (current read)** | `SELECT ... FOR UPDATE` / `LOCK IN SHARE MODE` / 所有 `UPDATE` / `DELETE` / `INSERT` | **直接读主表最新版本**，不走 ReadView |

这个区分是后面"案例 ② SELECT 与 FOR UPDATE 读到不同数据"的根。

**特别注意**：`UPDATE t SET v = v + 1 WHERE id = 1` 是当前读 + 写。它读到的 `v` 是**其他事务最新已提交的值**，不是事务 A 自己 ReadView 里那个旧值。这是为什么 RR 不能完全避免"丢失更新"。

### 1.5 锁的类型（一张图看清）

```
          锁
          │
   ┌──────┼──────┐
   表锁    行锁    意向锁
            │      ├── IS（意向共享）
   ┌────────┼─────┐└── IX（意向排他）
 Record  Gap   Next-Key
  Lock   Lock    Lock
                 = Record + Gap

   插入意向锁 (Insert Intention Lock)：特殊的 Gap Lock，
   表示"我打算往这个 gap 里插入"，互不冲突，但被普通 Gap Lock 阻塞
```

| 锁名 | 锁的内容 | 何时出现 |
| --- | --- | --- |
| **Record Lock** | 单条索引记录 | 主键唯一查询的等值锁 |
| **Gap Lock** | 索引记录之间的"间隙" | RR 下的范围扫描，防止幻读 |
| **Next-Key Lock** | Record + 它前面的 Gap | RR 下默认的行锁形式 |
| **Insert Intention** | 插入位置的"意向" | INSERT 等待 Gap Lock 释放时 |

为什么需要插入意向锁？因为多个事务想往同一个 gap 插不同的数据时，**它们之间不应该互斥**。但它们都被一个 SELECT FOR UPDATE 持有的 Gap Lock 阻塞。

---

## 2. 源码切片：InnoDB 关键路径

> 基于 MySQL 8.0 InnoDB 源码。注释为本文添加。

### 2.1 ReadView 的可见性判定

```cpp
// storage/innobase/include/read0types.h
bool ReadView::changes_visible(trx_id_t id, const table_name_t &name) const {
    ut_ad(id > 0);

    if (id < m_up_limit_id || id == m_creator_trx_id) {
        return (true);                           // ① 老于最小活跃 或 自己改的 → 可见
    }

    check_trx_id_sanity(id, name);

    if (id >= m_low_limit_id) {
        return (false);                          // ② 比"最大已分配"还新 → 不可见
    }
    if (m_ids.empty()) {
        return (true);                           // ③ 没有活跃事务 → 可见
    }

    const ids_t::value_type *p = m_ids.data();
    return (!std::binary_search(p, p + m_ids.size(), id));
                                                 // ④ 二分查找：不在活跃列表 → 可见
}
```

**逐段点透**：

- **① 的妙处**：`id < m_up_limit_id` 是绝大多数读的快路径 —— 只读正常已提交数据时几乎所有 trx_id 都比 up_limit 小，直接返回 true。
- **③ 的细节**：m_ids 只在 ReadView 创建时填充一次。即使后来又有事务开启，**对这个 ReadView 不可见**（因为它的 low_limit 已经定死）。这正是 RR "可重复读" 的实现。
- **④ 用二分**：m_ids 创建时已排序。生产中大事务场景活跃列表可能上万，二分查找让单次可见性判定 O(log n)。
- **整个函数没有锁**：ReadView 是事务私有的，无并发，无锁查询。

### 2.2 沿 undo log 链回溯老版本

```cpp
// storage/innobase/row/row0vers.cc 简化
bool row_vers_build_for_consistent_read(
    const rec_t *rec,           // 主表行
    mtr_t *mtr,
    dict_index_t *index,
    ulint **offsets,
    ReadView *view,             // 当前事务的 ReadView
    rec_t **old_vers) {         // 输出：可见的老版本

    trx_id_t trx_id = row_get_rec_trx_id(rec, index, *offsets);

    if (view->changes_visible(trx_id, index->table->name)) {
        return DB_SUCCESS;       // 当前版本就可见，直接用
    }

    // 不可见 → 沿 roll_ptr 回溯
    rec_t *prev_version = nullptr;
    while (true) {
        roll_ptr_t roll_ptr = row_get_rec_roll_ptr(rec, index, *offsets);
        if (roll_ptr == 0) {
            *old_vers = nullptr;        // 回到链尾还不可见 → 这一行对当前事务不存在
            return DB_SUCCESS;
        }

        // 从 undo log 取出上一版本
        trx_undo_prev_version_build(rec, mtr, index, *offsets,
                                    roll_ptr, &prev_version);
        if (!prev_version) {
            *old_vers = nullptr;        // undo 已被清理（purge 过了）
            return DB_SUCCESS;
        }

        trx_id_t prev_trx_id = row_get_rec_trx_id(prev_version, index, *offsets);
        if (view->changes_visible(prev_trx_id, index->table->name)) {
            *old_vers = prev_version;   // 找到一个可见的版本
            return DB_SUCCESS;
        }
        rec = prev_version;             // 继续往老版本走
    }
}
```

**逐段点透**：

- **`trx_undo_prev_version_build`** 是 MVCC 的"读放大"来源：每次回溯都要解析一段 undo log。**版本链越长，SELECT 越慢**。这就是案例 ① 的根。
- **回到链尾还不可见** 意味着：这行是在 ReadView 创建之后才被某个事务 INSERT 的 → 对当前事务"不存在"。这是 MVCC 防幻读的一半（另一半是 Gap Lock）。
- **`!prev_version`**：undo log 可能被 **purge 线程**清理。purge 必须等到"没有任何活跃 ReadView 可能用到这个老版本"才能进行 —— 这是长事务阻止 purge 的机理。

### 2.3 Next-Key Lock 的加锁路径

```cpp
// storage/innobase/lock/lock0lock.cc 简化版
dberr_t lock_clust_rec_read_check_and_lock(
    const buf_block_t *block,
    const rec_t *rec,
    dict_index_t *index,
    const ulint *offsets,
    select_mode sel_mode,
    lock_mode mode,                  // LOCK_S 或 LOCK_X
    ulint gap_mode,                  // LOCK_ORDINARY / LOCK_GAP / LOCK_REC_NOT_GAP / LOCK_INSERT_INTENTION
    que_thr_t *thr) {

    if (gap_mode == LOCK_REC_NOT_GAP) {
        // 仅锁记录，不锁 gap（RC 模式或唯一索引等值查询）
        type_mode = LOCK_REC | mode | LOCK_REC_NOT_GAP;
    } else if (gap_mode == LOCK_GAP) {
        type_mode = LOCK_REC | mode | LOCK_GAP;
    } else {
        type_mode = LOCK_REC | mode;     // 默认：Next-Key Lock
    }

    return lock_rec_lock(true, sel_mode, type_mode, block, heap_no, index, thr);
}
```

**关键决策点：什么时候用 LOCK_REC_NOT_GAP（仅记录锁，不上 gap）？**

```cpp
// 主键/唯一索引的等值查询，且记录存在 → 只需要 Record Lock
if (index->is_clustered() && srch_mode == PAGE_CUR_GE
    && unique_search && record_found) {
    gap_mode = LOCK_REC_NOT_GAP;
}
```

这是**"主键等值 SELECT FOR UPDATE 不上 gap 锁"** 的源码依据。如果是范围查询、非唯一索引查询、或主键查询但行不存在，则上 Next-Key。

---

## 3. 实测数据：用 sysbench 验证 MVCC 的代价

### 3.1 长事务下 SELECT 的退化

**实验**：开一个长事务（不提交），同时另一个事务对同一行高频 UPDATE。然后在第三个会话里测 SELECT 的耗时。

```sql
-- Session A：长事务，开着不提交
START TRANSACTION;
SELECT * FROM bench WHERE id = 1;     -- 触发 ReadView 创建
-- 不提交，挂起

-- Session B：高频更新
UPDATE bench SET v = v + 1 WHERE id = 1;   -- 跑 100 万次

-- Session C：测 SELECT 耗时
SELECT BENCHMARK(10000, (SELECT v FROM bench WHERE id = 1));
```

典型结果（MySQL 8.0、单行 100 万次 UPDATE 后）：

| Session A 状态 | Session C 单次 SELECT 耗时 |
| --- | --- |
| 不开长事务 | ~5 µs |
| 开长事务 100 万 UPDATE | **~12 ms** |

**慢了约 2400 倍**。原因：Session C 沿 undo 链要回溯 100 万次才能找到对它可见的版本。

### 3.2 长事务把 history list 撑爆

```sql
-- 观察 history list length（HLL）
SHOW ENGINE INNODB STATUS\G
```

```
History list length 23
```

正常生产应该 < 1000。如果你看到 100 万 +，说明有长事务阻止了 purge。

监控告警：`innodb_history_list_length > 100000` 持续 5 分钟。

### 3.3 RR 与 RC 在不同负载下的吞吐对比

sysbench `oltp_read_write` 在 16 核 / 64GB / NVMe 机器上：

| 隔离级别 | TPS（4 线程） | TPS（64 线程） | TPS（256 线程） |
| --- | --- | --- | --- |
| **READ COMMITTED** | 5,200 | 38,000 | **42,000** |
| **REPEATABLE READ** | 5,100 | 36,000 | 35,000 |

高并发下 RC 更快。原因：
- RR 用 Next-Key Lock，锁范围更大 → 锁等待更多
- RR 的 ReadView 整事务复用 → undo 链积累得更快 → SELECT 退化

> **口诀**：**云原生场景 + 应用层幂等做得好 → RC 是更好的默认**。AWS Aurora、阿里云 PolarDB 默认都建议 RC。但 MySQL 自身默认仍是 RR，且业务依赖 RR 行为，切换前必须评估。

---

## 4. 生产案例 8 则

### 案例 ① 长事务把 history list 撑爆 —— "主从延迟从 0 飙到 8 小时"

#### 现象
凌晨 2 点，DBA 告警：主从延迟从 0 飙到 8 小时还在涨。`SHOW SLAVE STATUS` 看 `Seconds_Behind_Master` 持续增长。CPU、IO 正常，binlog 也在写。

#### 根因
登上主库 `SHOW ENGINE INNODB STATUS`：

```
TRANSACTIONS
------------
Trx id counter 18439020384
Purge done for trx's n:o < 18439015100 undo n:o < 0
History list length 4823521        ← ★
```

History list length 480 万。正常应该 < 1000。

继续看活跃事务：

```
---TRANSACTION 18438900000, ACTIVE 18234 sec
mysql tables in use 1, locked 0
MySQL thread id 12345, OS thread handle 0x7f..., query id 9999 idle
```

**ACTIVE 18234 秒** —— 一个事务开了 5 个小时还没提交。

机理串起来：
1. 长事务持有一个 ReadView，purge 不能清理它可能读到的 undo
2. 主库高频更新，每次更新都加一条 undo
3. undo 链 + history list 持续膨胀
4. 从库 SQL 线程回放主库的 binlog 时，每条 UPDATE 也要走相同的 undo 链
5. 从库回放速度跟不上 → 延迟剧增

#### 修复
**临时**：找到长事务杀掉。

```sql
-- 找正在跑的事务及其持续时间
SELECT trx_id, trx_started,
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS seconds,
       trx_mysql_thread_id, trx_query
FROM information_schema.innodb_trx
ORDER BY trx_started;

-- 杀掉
KILL 12345;
```

**长效**：
1. 监控 `innodb_history_list_length`，> 100000 告警
2. 业务排查所有 `START TRANSACTION` 必须有对应的 `COMMIT/ROLLBACK`，且事务体内不能有阻塞操作（远程调用、长 IO）
3. SQL 客户端开 `autocommit=1`，避免开了事务忘提交
4. 应用框架配 `transactionTimeout=30s`，超时自动回滚

#### 验证
```sql
-- 改完后观察 HLL 必须能下降到 < 1000
SHOW ENGINE INNODB STATUS\G
-- 主从延迟回到 0
SHOW SLAVE STATUS\G
```

> **口诀**：长事务 = ReadView 不释放 = purge 不能动 = history list 膨胀 = 全库 SELECT 变慢 + 主从延迟。**事务体里禁止远程调用**。

---

### 案例 ② SELECT 与 FOR UPDATE 读到不同数据 —— "幻觉里的两份现实"

#### 现象
RR 隔离下，事务 A 同一查询，先 `SELECT` 看到一份数据，紧接着 `SELECT ... FOR UPDATE` 看到另一份数据。

```sql
-- Session A
START TRANSACTION;
SELECT v FROM t WHERE id = 1;                      -- 输出：v = 10

-- Session B 提交：UPDATE t SET v = 100 WHERE id = 1;

-- Session A
SELECT v FROM t WHERE id = 1;                      -- 输出：v = 10 （RR 重复读）
SELECT v FROM t WHERE id = 1 FOR UPDATE;           -- 输出：v = 100 ★
```

#### 根因
- **普通 SELECT** 是一致性读（snapshot read），走 ReadView
- **SELECT FOR UPDATE** 是当前读（current read），**绕过 ReadView 直接读主表最新行**

ReadView 在事务 A 第一次 SELECT 时就创建了，记录了"看不到 Session B 的事务"。但 FOR UPDATE 不看 ReadView。

更危险的实践 —— "丢失更新"：

```sql
-- 错的写法
START TRANSACTION;
SELECT v FROM t WHERE id = 1;          -- 读到 v = 10
-- 应用层算 newV = 10 * 2 = 20
UPDATE t SET v = 20 WHERE id = 1;      -- ★ 丢失：如果 B 在中间更新了 v，B 的更新被覆盖
COMMIT;
```

UPDATE 是当前读，它会读到 B 已提交的 v=100，然后用应用层算的 20 覆盖掉。**RR 没救**。

#### 修复
策略 A：**乐观锁**（推荐）

```sql
SELECT v, version FROM t WHERE id = 1;
-- 应用层算 newV
UPDATE t SET v = newV, version = version + 1
WHERE id = 1 AND version = ?;          -- ★ 如果别人改过，version 不匹配，影响行 = 0
```

应用层判断 affectedRows == 0 → 重试。

策略 B：**SELECT FOR UPDATE 锁住**

```sql
START TRANSACTION;
SELECT v FROM t WHERE id = 1 FOR UPDATE;   -- ★ 加 X 锁，B 等
-- 算 newV
UPDATE t SET v = newV WHERE id = 1;
COMMIT;
```

注意：FOR UPDATE 加 X 锁会让其他事务等待。在主键等值查询且行存在时只是 Record Lock，影响小；范围查询会上 Next-Key Lock，影响大。

策略 C：**原子操作改写**

```sql
UPDATE t SET v = v * 2 WHERE id = 1;       -- ★ 在 DB 端原子完成
```

#### 验证
两个会话并发跑 1000 次 SELECT-计算-UPDATE，最终值应该等于 1000 × 初始计算结果。错误写法会丢更新，正确写法不会。

> **口诀**：**RR 只对一致性读 (SELECT) 重复读；UPDATE / DELETE / SELECT FOR UPDATE 永远读最新**。读后改场景必须用乐观锁或 FOR UPDATE，不要靠 RR。

---

### 案例 ③ RR 下也会幻读 —— "重复扫描看到新行"

#### 现象
有人说"MySQL RR 不会幻读"。但下面这个序列在 RR 下能稳定复现幻读：

```sql
-- Session A
START TRANSACTION;
SELECT * FROM t WHERE id > 10;                     -- 输出：[]

-- Session B 提交：INSERT INTO t (id) VALUES (15);

-- Session A
SELECT * FROM t WHERE id > 10;                     -- 输出：[]（RR 一致性读，看不到 B）
UPDATE t SET v = 1 WHERE id > 10;                  -- ★ 影响行 = 1，居然改到了 B 的行
SELECT * FROM t WHERE id > 10;                     -- 输出：[id=15, v=1]   ← 幻读！
```

#### 根因
"RR 阻止幻读"是不严谨的说法。MySQL 的 RR 用两个机制配合：
- **MVCC（一致性读）** 阻止"普通 SELECT 看到新插入的行"
- **Next-Key Lock（当前读 + 显式锁）** 阻止"加锁查询时被插入新行"

但**普通 SELECT 不上锁，UPDATE 是当前读**。普通 SELECT 看不到 B 的行，但 UPDATE 看得到、改得到，**改完之后那行就成了"自己改的行"，对自己可见**。

ANSI SQL 的 SERIALIZABLE 才严格不允许幻读。MySQL RR 给的是"近似不幻读"。

#### 修复
方案 A：用 SERIALIZABLE。代价：所有 SELECT 自动加 LOCK IN SHARE MODE，并发暴跌。

方案 B：业务上把这种"先查再批量改"的逻辑加显式锁：

```sql
SELECT * FROM t WHERE id > 10 FOR UPDATE;          -- ★ Next-Key Lock 锁住 (10, +∞)
-- B 的 INSERT 会等
UPDATE t SET v = 1 WHERE id > 10;
```

#### 验证
跑上面三步序列，确认锁加上后 B 的 INSERT 会阻塞。

> **口诀**：MySQL RR 的"防幻读"只对一致性读有效。**写场景要防幻读必须显式 FOR UPDATE 或换 SERIALIZABLE**。

---

### 案例 ④ 二级索引 + 间隙锁的死锁现场

#### 现象
两个事务并发执行一个简单的"先 DELETE 再 INSERT"逻辑（典型的"upsert 替换"模式），偶发死锁。

```sql
-- 表结构
CREATE TABLE t (
  id INT PRIMARY KEY,
  name VARCHAR(50),
  KEY idx_name (name)
);

-- 数据
INSERT INTO t VALUES (10, 'a'), (20, 'c'), (30, 'e');
```

```sql
-- Session A                        Session B
START TRANSACTION;                  START TRANSACTION;
DELETE FROM t WHERE name = 'b';     DELETE FROM t WHERE name = 'd';
INSERT INTO t VALUES (15, 'b');     INSERT INTO t VALUES (25, 'd');
                                    -- ★ 死锁！
```

#### 根因
RR 下 DELETE 走二级索引 `idx_name`，找不到记录时仍要加 Next-Key Lock，锁的范围是**包含等值点的整个 gap**：

- A `DELETE WHERE name='b'` → `idx_name` 上锁范围 `('a', 'c']`
- B `DELETE WHERE name='d'` → `idx_name` 上锁范围 `('c', 'e']`

到这里没冲突。然后两边都 INSERT：

- A `INSERT (15, 'b')` → 需要在 `idx_name` 的 `('a', 'c']` 加插入意向锁 —— 它**自己持有这把 gap 锁**，自己跟自己不冲突，应该能成功？

实际上 InnoDB 的实现里，DELETE 加的是 **Next-Key Lock**，INSERT 等的是 **Insert Intention Lock**。Insert Intention Lock 在 gap 已经被**别的事务**的 Next-Key 持有时会等待。

但奇怪的是 A INSERT 'b' 应该不与 B 持有的 `('c', 'e']` 冲突啊？真正死锁的现场更复杂 —— `SHOW ENGINE INNODB STATUS` 看 LATEST DETECTED DEADLOCK 段：

```
*** (1) TRANSACTION:
  ... waiting for lock on idx_name (gap before 'c'), insert intention
*** (2) TRANSACTION:
  ... waiting for lock on idx_name (gap before 'e'), insert intention

*** WE ROLL BACK TRANSACTION (2)
```

实际原因是：你的 SQL 在不同行号下两边的等值点穿插，**等值不存在的 DELETE 在 RR 下加的 Next-Key 范围比你以为的大**，加上插入位置的间隙重叠时就死锁。

更普遍的死锁规律：

```
事务 1: 先锁 A，再请求 B
事务 2: 先锁 B，再请求 A
→ 死锁
```

DELETE 走二级索引时锁的是二级索引上的 gap；INSERT 时按聚簇索引位置插入，又在二级索引上加 Insert Intention Lock。**两个索引上的锁顺序不一致**就容易死锁。

#### 修复
方案 A（推荐）：把 "DELETE + INSERT" 改成 `INSERT ... ON DUPLICATE KEY UPDATE` 或 `REPLACE INTO`。

```sql
INSERT INTO t (id, name) VALUES (15, 'b')
ON DUPLICATE KEY UPDATE name = VALUES(name);
```

它在 InnoDB 里是单条原子操作，加锁顺序固定。

方案 B：业务侧统一加锁顺序。比如永远按 id 升序处理。

方案 C：用 RC 隔离级别。RC 不上 Gap Lock（除非外键检查），死锁概率大幅下降，代价是要业务层处理"不可重复读 / 幻读"。

#### 验证
跑两个并发的 "DELETE 不存在的行 + INSERT" 几千次，修复前死锁率高，修复后归零。

> **口诀**：**RR 下 DELETE 找不到行也加 Next-Key Lock**。涉及二级索引的 DELETE+INSERT 模式高危。能用 INSERT ... ON DUPLICATE 就别拆。

---

### 案例 ⑤ INSERT 在等什么 —— "插入意向锁的真相"

#### 现象
监控显示某条 `INSERT INTO orders` 偶发等锁 30s 超时。错误日志：

```
Lock wait timeout exceeded; try restarting transaction
```

但表上没有显式的 SELECT FOR UPDATE，谁锁住了？

#### 根因
查询活跃锁：

```sql
SELECT * FROM performance_schema.data_locks;
```

```
+-----------+------------+-------------+-----------+-----------+----------+--------+
| ENGINE    | OBJECT     | INDEX       | LOCK_TYPE | LOCK_MODE | STATUS   | DATA   |
+-----------+------------+-------------+-----------+-----------+----------+--------+
| INNODB    | orders     | PRIMARY     | RECORD    | X,GAP     | GRANTED  | 100    |
| INNODB    | orders     | PRIMARY     | RECORD    | X,INSERT_INTENTION | WAITING | 100 |
+-----------+------------+-------------+-----------+-----------+----------+--------+
```

第一条：某个事务对 PRIMARY 索引、id=100 之前的 gap，持有 X+GAP 锁
第二条：你的 INSERT 在等同一位置的 INSERT_INTENTION 锁

机理：
- Insert Intention Lock 的语义是"我想在这个 gap 插入"
- 它和**其他 Insert Intention 兼容**（多个事务可以同时往一个 gap 插不同行）
- 但**和 Gap Lock 互斥**（Gap Lock 持有者不希望任何插入进来）
- 持有 Gap Lock 的另一个事务通常是某个 `SELECT WHERE id > 50 FOR UPDATE` 或 `UPDATE WHERE id > 50`

找出谁持有：

```sql
SELECT
    r.trx_id AS waiting_trx,
    r.trx_query AS waiting_query,
    b.trx_id AS blocking_trx,
    b.trx_query AS blocking_query,
    b.trx_started AS blocker_started
FROM information_schema.innodb_lock_waits w
JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id
JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id;
```

#### 修复
- 找出持有 Gap Lock 的长事务，缩短它
- 评估能不能换 RC（RC 下普通 UPDATE 不上 Gap Lock）
- 大批量更新拆小事务，避免一个事务持有大范围 Gap Lock 太久

#### 验证
监控 `innodb_lock_wait_current_count` + `innodb_lock_timeouts`，修复后应大幅下降。

> **口诀**：INSERT 等的锁叫 **Insert Intention Lock**。它被 Gap Lock 阻塞，但不被同类阻塞。看到 INSERT 慢，去查谁的 SELECT/UPDATE 持有了 Gap Lock。

---

### 案例 ⑥ 主从一致性 —— "主库读到、从库读不到"

#### 现象
应用读写分离：写主库、读从库。某用户下单后立即查订单详情，**偶发查不到**，刷新一次又有了。

#### 根因
异步复制（默认）下：
1. 主库 `INSERT order` 提交
2. 主库返回客户端"成功"
3. 应用立即去**从库**查这个订单
4. 此时主库的 binlog 还没传到从库或从库还没回放完

**MVCC 在这里的作用**：从库的 SQL 回放线程是有自己的 ReadView 的，但你的查询走的是从库的另一个连接，看不到尚未回放完的事务。

binlog 格式的影响：
- **STATEMENT**：从库执行的是 SQL 字符串。如果 SQL 含 `NOW()`、`UUID()` 等非确定函数，主从可能不一致
- **ROW**（推荐）：从库执行的是逐行变更，主从数据严格一致，但 binlog 体积大
- **MIXED**：默认 STATEMENT，遇到非确定操作切 ROW

但格式只影响**一致性正确性**，不影响**延迟**。延迟是物理传输+回放速度问题。

#### 修复
方案 A：**同 session 写后立即读 → 路由到主库**。常见做法：
- ShardingSphere 的 `HintManager.setWriteRouteOnly()`
- MyBatis 拦截器：检测是否在事务中，事务中所有读走主库
- 应用层标记："写后 N 秒内的同 session 读走主库"

方案 B：**半同步复制**。主库等至少一个从库 ACK 后再返回成功。延迟换一致性。

```ini
# 主库
plugin_load_add = "rpl_semi_sync_master=semisync_master.so"
rpl_semi_sync_master_enabled = ON
rpl_semi_sync_master_timeout = 1000

# 从库
plugin_load_add = "rpl_semi_sync_slave=semisync_slave.so"
rpl_semi_sync_slave_enabled = ON
```

方案 C：**MGR / Group Replication / 云原生 Aurora 系列**。底层共识协议保证 quorum 读到的最新数据。

#### 验证
压测"写后立即读"场景的成功率，应 100%。

> **口诀**：异步复制下"写主读从"必有窗口期不一致。**强一致需求 → 写后路由主库 / 半同步 / 共识协议**。

---

### 案例 ⑦ 大事务回滚 —— "几个 G 的回滚跑了一夜"

#### 现象
DBA 误执行 `UPDATE bigtable SET status = 1`（全表 5000 万行），跑了 30 分钟发现错了，`KILL` 掉。**结果服务器 CPU 持续高位 5 个小时**，大量"卡顿"告警。

#### 根因
ROLLBACK 不是免费的。InnoDB 必须沿着 undo log 链**逐行撤销**已经做过的修改。
- 已修改 5000 万行 → 必须读 5000 万条 undo log → 逐行写回老版本
- 同时还要回滚之前为该事务记的所有锁
- 回滚期间持有大量行锁、Gap 锁，其他事务被阻塞

更糟：
- 回滚期间**不响应 KILL**
- binlog 已经写了的部分要"反向"清掉（实际是写一条 `ROLLBACK` 标记，但中间步骤要清掉 binlog cache）

观察回滚进度：

```sql
SELECT trx_id, trx_state, trx_rows_modified, trx_rows_locked
FROM information_schema.innodb_trx
WHERE trx_state = 'ROLLING BACK';
```

看 `trx_rows_modified` 是否在下降（如果还在增就说明前进缓慢）。

#### 修复

预防：
1. **永远不要直接跑全表 UPDATE / DELETE**。强制加 LIMIT，分批跑：
   ```sql
   -- 一次只改 10000 行，跑循环
   UPDATE bigtable SET status = 1 WHERE status != 1 LIMIT 10000;
   ```
2. 工具：`pt-archiver`、Gh-ost、自研分批工具
3. 客户端配 `safe_updates = 1`，没 WHERE 的 UPDATE/DELETE 直接报错

事中：
- 不要再 KILL，等回滚跑完更快
- 真要 KILL 后立即重启 MySQL：从崩溃恢复反而比"前台回滚"快（崩溃恢复用并行 rollback）
- 重启前确认 binlog 已 sync，避免数据丢失

#### 验证
分批后单批执行时间应 < 1s，整批可控、可中断、可恢复。

> **口诀**：**事务大小决定回滚代价**。生产 UPDATE/DELETE 必须分批，单批 < 万行。

---

### 案例 ⑧ Online DDL 与长事务的冲突

#### 现象
执行 `ALTER TABLE ADD INDEX` 加索引（号称 Online DDL，不阻塞读写），结果**业务全卡住**，监控显示大量"等元数据锁"。

```sql
SHOW PROCESSLIST;
+--------+------+--------------------+
| Id     | ...  | State              |
+--------+------+--------------------+
| 1234   | ...  | Waiting for table metadata lock |
| 1235   | ...  | Waiting for table metadata lock |
| ...                                |
+--------+------+--------------------+
```

#### 根因
所谓"Online DDL"，**首尾两个瞬间仍要拿 MDL（Metadata Lock）写锁**：
1. 开始：拿 MDL 写锁，准备元信息
2. 中间：长时间的索引构建可以并发读写
3. 结束：再拿 MDL 写锁，提交元信息

MDL 写锁需要等待**所有持有该表 MDL 读锁的事务结束**。如果有长事务正持有该表的 MDL 读锁（比如一个跑了 1 小时的 SELECT），DDL 就卡在第一步。

更危险：DDL 在等 MDL 写锁时，**后续所有该表的查询（无论读写）都被它阻塞**。一个 DDL 卡住 → 全表停摆。

#### 修复
方案 A：DDL 前先杀长事务

```sql
SELECT trx_id, trx_started, trx_mysql_thread_id, trx_query
FROM information_schema.innodb_trx
WHERE trx_started < NOW() - INTERVAL 30 SECOND;
-- 杀掉持续时间长的
KILL <thread_id>;
```

方案 B：DDL 加 LOCK_WAIT_TIMEOUT，避免无限等

```sql
SET SESSION lock_wait_timeout = 30;
ALTER TABLE t ADD INDEX idx_name (name);
-- 拿不到 MDL 就快速失败，不阻塞业务
```

方案 C：在线 DDL 工具

- **gh-ost**（GitHub 出品）：通过 binlog 解析复制方式做 DDL，不需要长时间持有 MDL
- **pt-online-schema-change**：用触发器同步增量

```bash
gh-ost --user=root --password=*** \
  --host=localhost --database=db --table=t \
  --alter="ADD INDEX idx_name (name)" \
  --execute
```

#### 验证
DDL 期间业务 QPS 平稳，无 MDL 等待告警。

> **口诀**：Online DDL 不是"无锁"，是"中间过程不锁"。**首尾必拿 MDL 写锁，长事务会卡住整个 DDL 进而卡住全表**。

---

## 5. 上线检查清单

### 隔离级别
- [ ] 业务的实际隔离级别是否与你以为的一致（`SELECT @@transaction_isolation`）
- [ ] 评估能否用 RC（云原生场景推荐 RC + 应用层乐观锁）
- [ ] 切换隔离级别前必须做"读后改"逻辑的全量回归

### 事务体内
- [ ] 事务体内**禁止远程调用 / 长 IO**
- [ ] 事务超时（`innodb_lock_wait_timeout` + 应用层超时）必须配
- [ ] `autocommit = 1`，避免开了事务忘提交
- [ ] 大批量 UPDATE/DELETE 分批，每批 < 1 万行
- [ ] 读后改用乐观锁（version 列）或 FOR UPDATE 显式锁

### 监控
- [ ] `innodb_history_list_length` 监控，> 100000 告警
- [ ] `innodb_lock_wait_current_count` 监控
- [ ] `innodb_row_lock_time_avg` 监控
- [ ] 主从延迟 `Seconds_Behind_Master`
- [ ] 慢查询日志开启，`long_query_time = 1`
- [ ] 死锁日志开启 `innodb_print_all_deadlocks = ON`

### 锁
- [ ] 索引设计避免大范围 Next-Key Lock
- [ ] 高频写表评估 RC（减少 Gap Lock）
- [ ] DELETE+INSERT 模式改 `INSERT ... ON DUPLICATE`
- [ ] 业务侧统一加锁顺序，预防死锁

### 复制
- [ ] binlog_format = ROW（强一致首选）
- [ ] 写后立即读路由主库（应用层框架支持）
- [ ] 关键场景考虑半同步复制
- [ ] 从库 `slave_parallel_workers > 1`（并行复制减延迟）

### DDL
- [ ] 大表 DDL 用 gh-ost / pt-osc，不直接 ALTER
- [ ] DDL 前检查并杀掉长事务
- [ ] DDL 时设 `lock_wait_timeout` 防止卡死全表

---

## 6. 还能往下挖什么

### 跨数据库视野
- **PostgreSQL** 的 MVCC：不删老版本，靠 vacuum 清理（"膨胀"问题不同）
- **Oracle** 的 MVCC：用 SCN（System Change Number）+ undo segment
- **Snapshot Isolation 论文**（Berenson 1995）：MVCC 隔离的理论基础
- **Serializable Snapshot Isolation (SSI)**：PostgreSQL 9.1+，比传统 SI 更严格

### TiDB / 分布式数据库
- **Percolator 模型**（Google 2010）：分布式 MVCC + 乐观事务
- TiDB 的悲观/乐观事务模型切换
- 分布式事务的两阶段提交在 MVCC 下的可见性挑战

### 共识协议与 MVCC 结合
- **MGR (MySQL Group Replication)**：多主复制下的冲突检测与可见性
- **Aurora**：日志即数据库，MVCC 下沉到存储层
- **Spanner / TrueTime**：全球时钟下的 Snapshot Isolation

### 读流量分摊
- 读写分离的"读后立即读"问题的工业方案
- ProxySQL / ShardingSphere / Vitess 的会话粘性
- 缓存与 MVCC 的一致性（Cache Aside / 双删 / Canal 同步）

### InnoDB 内部
- **purge thread** 的具体调度（`innodb_purge_threads`、`innodb_purge_batch_size`）
- **change buffer** 与二级索引插入的延迟合并
- **doublewrite buffer** 与 redo log 的协作（防止 partial page write）

---

## 7. 引用

- MySQL 8.0 InnoDB 源码：[storage/innobase/](https://github.com/mysql/mysql-server/tree/8.0/storage/innobase)
- Berenson et al., *A Critique of ANSI SQL Isolation Levels*（1995）—— SI 论文
- 《MySQL 实战 45 讲》—— 林晓斌（极客时间）
- 《高性能 MySQL（第 4 版）》—— Schwartz / Zaitsev
- [How MySQL InnoDB Cluster handles failures](https://dev.mysql.com/doc/refman/8.0/en/mysql-innodb-cluster-introduction.html)
- 阿里云 RDS [MVCC 最佳实践](https://help.aliyun.com/document_detail/417082.html)

---

> 上一篇综述：[MySQL 速记](/article/db/mysql) ｜ 配套：[Redis 核心](/article/db/redis)
> 这是按 [桃花岛深度模板](/STYLE) 写的第二篇深文。下一篇候选：[synchronized 内幕](/article/java/concurrent#2-synchronized-与-lock) / [AQS 源码深读](/article/java/concurrent#3-aqs必背) / [RAG 工程化](/article/ai/rag) —— 想看哪个告诉我。
