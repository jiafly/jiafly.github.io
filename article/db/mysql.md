# MySQL 核心

> MySQL 面试三大支柱：**索引、事务、锁**。本文按这条主线展开，附带常用排查命令。

---

## 1. 存储引擎（八股一句话）

| 引擎 | 特征 |
| --- | --- |
| **InnoDB**（默认） | 支持事务、行锁、MVCC、外键，B+ 树聚簇索引 |
| MyISAM | 表锁、不支持事务，已不推荐 |
| Memory | 全内存，重启丢数据 |

> 现代项目几乎只有 InnoDB 一个选项。

---

## 2. 索引

### 2.1 B+ 树为什么是默认结构？
- 节点存指针，**叶子节点串成有序链表** → 区间查询 O(log n + k)；
- 树高低（一般 3~4 层即可承载千万行），磁盘 IO 少；
- 与 B 树相比，非叶子不存数据 → 单节点能放更多 key → 树更矮。

### 2.2 聚簇索引 vs 二级索引（必懂）

```
主键索引（聚簇索引）           二级索引
   │                             │
   ▼                             ▼
[B+ 树]                        [B+ 树]
叶子 = 整行数据                 叶子 = 主键
                                ↓
                           回表查聚簇索引
```

- **InnoDB 主键索引就是数据本身**（聚簇）；
- 二级索引叶子存的是**主键**，不是行地址；
- 二级索引查询命中后，需要回表（除非走"覆盖索引"）。

### 2.3 覆盖索引
查询的列**全部在索引中**，不需要回表：
```sql
-- 索引：(name, age)
SELECT name, age FROM users WHERE name = 'tom';   -- 覆盖索引
SELECT name, age, city FROM users WHERE name='tom';  -- 仍需回表
```

### 2.4 最左前缀
联合索引 `(a, b, c)` 可命中：
- `WHERE a = ?`
- `WHERE a = ? AND b = ?`
- `WHERE a = ? AND b = ? AND c = ?`
- `WHERE a = ? AND c = ?`（c 走不到，但 a 命中）

不能命中：
- `WHERE b = ?` ❌
- `WHERE b = ? AND c = ?` ❌

### 2.5 索引失效场景（高频）
1. 函数运算：`WHERE YEAR(create_time) = 2024` → 改成范围；
2. 隐式类型转换：`WHERE phone = 13800001234`（phone 是 varchar）；
3. `LIKE '%abc'`（前导通配符）；
4. `OR` 两边一个无索引；
5. `!=` / `NOT IN`（部分场景）；
6. 联合索引未走最左；
7. 数据分布极不均匀，优化器认为全表扫更快。

### 2.6 EXPLAIN 关键列
```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 1;
```
| 列 | 含义 |
| --- | --- |
| **type** | 访问类型，从好到差：`system > const > eq_ref > ref > range > index > ALL` |
| **key** | 实际用的索引（NULL = 没走索引） |
| **rows** | 预计扫描行数 |
| **Extra** | `Using index`（覆盖）、`Using where`、`Using filesort`（坏）、`Using temporary`（坏） |

---

## 3. 事务与隔离级别（速记）

> 写代码遇到的怪现象、生产死锁、长事务事故全在 → **[MySQL MVCC 与锁深文](/article/db/mysql-mvcc)**

### 3.1 ACID
- A 原子性：undo log 实现回滚；
- C 一致性：业务规则；
- I 隔离性：锁 + MVCC；
- D 持久性：redo log 实现 crash safe。

### 3.2 四种隔离级别

| 级别 | 脏读 | 不可重复读 | 幻读 |
| --- | --- | --- | --- |
| READ UNCOMMITTED | ✓ | ✓ | ✓ |
| READ COMMITTED | ✗ | ✓ | ✓ |
| **REPEATABLE READ**（MySQL 默认） | ✗ | ✗ | **MVCC + Next-Key Lock 近似阻止** |
| SERIALIZABLE | ✗ | ✗ | ✗ |

> ⚠️ 常见误区："MySQL RR 完全防幻读"是错的。RR 只对一致性读 (SELECT) 防幻读，UPDATE/DELETE 仍是当前读 → 仍能看到新行。
> → 复现序列与机理见 [深文 §4 案例 ③](/article/db/mysql-mvcc#案例--rr-下也会幻读--重复扫描看到新行)

### 3.3 MVCC 速记
- **隐藏列**：`DB_TRX_ID`（最后修改的事务）+ `DB_ROLL_PTR`（指向 undo 链）
- **可见性判定**：基于 ReadView（活跃事务列表 + min/max trx_id）
- **RC vs RR 的唯一差别**：RC 每次 SELECT 重建 ReadView；RR 整事务复用
- **一致性读 vs 当前读**：普通 SELECT 走 ReadView；FOR UPDATE / UPDATE / DELETE 直接读最新行

→ 源码切片、可见性算法证明、长事务杀 history list 的事故复盘 见 [深文 §1-2](/article/db/mysql-mvcc#1-底层模型隐藏列undo-log-链readview)

### 3.4 redo log / undo log / binlog

| 日志 | 作用 | 层 |
| --- | --- | --- |
| **redo log** | 物理日志，crash 后重做 | InnoDB |
| **undo log** | 逻辑日志，回滚 + MVCC 历史版本 | InnoDB |
| **binlog** | 主从复制 + 时间点恢复 | Server |

#### 两阶段提交（必懂）
为了 redo log 与 binlog 一致：
```
1. 写 redo log（prepare）
2. 写 binlog
3. 写 redo log（commit）
```
任意阶段崩溃都能恢复一致状态。

---

## 4. 锁（速记）

### 4.1 锁的分类
- 全局锁：`FLUSH TABLES WITH READ LOCK`（备份）
- 表锁：`LOCK TABLES`（少用）；MDL 元数据锁（DDL 时）
- 行锁（InnoDB）：
  - **Record Lock**：锁单条索引记录
  - **Gap Lock**：锁记录间的"间隙"（防插入，仅 RR）
  - **Next-Key Lock** = Record + Gap：RR 下范围扫描的默认形式
  - **Insert Intention Lock**：INSERT 在等 Gap Lock 时拿的"意向锁"
- 意向锁（IS / IX）：表级，给"快速判断表上有没有行锁"用

### 4.2 共享锁 / 排他锁
```sql
SELECT ... LOCK IN SHARE MODE;   -- S 锁
SELECT ... FOR UPDATE;            -- X 锁
```

### 4.3 死锁速排查
```sql
SHOW ENGINE INNODB STATUS\G                    -- 看 LATEST DETECTED DEADLOCK
SELECT * FROM performance_schema.data_locks;   -- 实时锁状态
SELECT * FROM information_schema.innodb_lock_waits;
```
预防铁律：统一加锁顺序、控制事务长度、监控 `innodb_lock_wait_timeout`。

→ 二级索引死锁现场、INSERT 等什么锁、Online DDL 撞 MDL 的全套案例 见 [深文 §4 案例 ④⑤⑧](/article/db/mysql-mvcc#案例--二级索引--间隙锁的死锁现场)

---

## 5. 主从复制

### 5.1 流程
```
Master           Slave
  │                │
binlog ─────► IO 线程 ─► relay log ─► SQL 线程 ─► 重放
```

### 5.2 三种复制模式
| 模式 | 一致性 | 性能 |
| --- | --- | --- |
| 异步（默认） | 弱 | 高 |
| 半同步 | 中 | 中 |
| 全同步 | 强 | 低 |

### 5.3 主从延迟原因
- 从库单 SQL 线程，主库高并发写；
- 大事务、大表 DDL；
- 网络抖动；
- 从库做查询、IO 抢占。

> 解决：MySQL 5.7+ 支持并行复制（`slave_parallel_workers`），按库/事务并行。

---

## 6. 慢 SQL 排查方法论

```bash
# 开启慢日志
set global slow_query_log = 1;
set global long_query_time = 1;

# 分析
mysqldumpslow -t 10 -s t /path/slow.log

# 高级
pt-query-digest /path/slow.log
```

排查清单：
1. `EXPLAIN` 看执行计划；
2. 看是否走索引、type 等级；
3. 是否回表、是否覆盖；
4. 单表数据量是否过大（单表 > 2000 万考虑分表）；
5. 锁等待 → `SHOW ENGINE INNODB STATUS`；
6. 临时表、文件排序 → 加合适联合索引；
7. **小表驱动大表**；
8. 子查询改 `JOIN`。

---

## 7. 分库分表

### 7.1 何时分？
- 单表数据 > 2000 万 / 单库容量 > 1TB；
- QPS 写超过单库瓶颈（一般万级）；
- 历史数据归档不够。

### 7.2 维度
- **垂直拆分**：按业务模块拆库、按字段拆表（冷热分离）；
- **水平拆分**：按规则（hash、range、一致性 hash）打散到多库多表。

### 7.3 主流方案
- **ShardingSphere-JDBC**：客户端分片，无代理；
- **Vitess**、**MyCAT**：中间件代理；
- **TDDL / Apollo / 自研**。

### 7.4 分布式 ID
- **雪花 Snowflake**（41 位时间戳 + 10 位机器 + 12 位序列）；
- **美团 Leaf**：号段 + 雪花混合；
- **百度 UidGenerator**；
- 业界趋势：**BIGINT 自增 + 业务前缀**（简单且单调）。

---

## 8. 高频面试题速记

1. **InnoDB 为什么用 B+ 树？** 矮胖 + 范围查询友好 + 叶子链表。
2. **聚簇 vs 二级？** 聚簇叶子是数据；二级叶子是主键 → 回表。
3. **覆盖索引是什么？** SELECT 的列在索引中，无需回表。
4. **MySQL RR 怎么解决幻读？** Next-Key Lock 锁住范围。
5. **MVCC 原理？** trx_id + undo log + ReadView。
6. **redo / undo / binlog 区别？** 见第 3.4 节。
7. **两阶段提交？** prepare → binlog → commit，确保一致。
8. **索引失效场景？** 见 2.5。
9. **慢 SQL 排查思路？** 见第 6 节。
10. **什么时候分库分表？** 数据量 + QPS + 业务隔离。

---

> 下一篇：[Redis 核心](/article/db/redis)
