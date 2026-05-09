# MySQL 索引深入 —— B+ 树物理结构、索引下推与优化器陷阱

> 这是按 [桃花岛深度模板](/STYLE) 写的第三篇深文。如果你只想刷面试，去 [MySQL 速记](/article/db/mysql) 那一节够了。
> 这里要回答的，是**写 SQL 一查就慢、上线发现优化器选错索引**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. 一张 5000 万行、单行 1KB 的表，B+ 树聚簇索引为什么 **3~4 层** 就能装下？给出页大小、扇出、层数的算式。
2. 索引下推（Index Condition Pushdown，ICP）解决了什么问题？没有 ICP 的时候，server 层和存储引擎层是怎么"扯皮"的？
3. 覆盖索引为什么避免回表？"二级索引叶子节点存的就是主键 + 索引列"这句话精确吗？
4. 联合索引 `(a, b, c)`，给出**至少两个**会让"最左前缀"失效的反直觉 SQL（即看起来用了 a 但其实不走索引）。
5. 一个列基数 100 万、表 1 亿行，选择性 = 1%。这种情况建索引值不值？给一个量化的判断标准。
6. 同一条 SQL，今天走 idx_a、明天走 idx_b，业务报"突然变慢"。优化器为什么会"变心"？怎么稳定住？
7. `FORCE INDEX(idx_x)` 看起来很爽，它的真实代价是什么？什么时候会**反噬**？
8. MRR（Multi-Range Read）解决什么问题？为什么默认是关的（在某些 case 下）？

如果有半数答不上，往下看。

---

## 1. 底层模型：B+ 树、页、ICP、代价模型

绝大多数文章把 MySQL 索引讲成"B+ 树 + 最左前缀"，然后画一个示意图就结束了。这层认知不够诊断生产问题。要看懂上面 8 个问题，必须下沉到 **页 (page) 这一物理单元** 和 **优化器代价模型** 这两个核心。

### 1.1 B+ 树为什么 3~4 层就够装千万行

InnoDB 索引的物理基本单位是**页 (page)**，默认 16KB。这个数字几乎决定了一切。

```
一个 InnoDB 索引页 = 16KB

非叶子节点：存 (索引列值, 子页号)
  - 索引列值：以 BIGINT 为例，8 字节
  - 子页号：6 字节（InnoDB 页号 = 4 字节空间号 + 4 字节页内偏移，但页指针是 6 字节）
  - 一对 ≈ 14 字节
  - 16384 / 14 ≈ 1170 个分支（"扇出"）

叶子节点：存完整行
  - 一行 1KB → 一页约 16 行
```

层数推算：

| 层数 | 总行数 |
| --- | --- |
| 1 层（只有根） | 16 |
| 2 层（根 → 叶） | 1170 × 16 ≈ 1.87 万 |
| 3 层（根 → 中间 → 叶） | 1170 × 1170 × 16 ≈ 2190 万 |
| 4 层 | 1170 × 1170 × 1170 × 16 ≈ 256 亿 |

**结论：3 层 B+ 树能装 2 千万行，4 层装下 200 亿**。一次主键等值查询最多 3~4 次磁盘 IO（实际上根和中间层几乎常驻 buffer pool，只有叶子需要真实 IO）。

这就是"为什么不用红黑树"——红黑树高度 O(log₂ N)，5000 万行需要 26 层，每层一次 IO，撑不住。

### 1.2 聚簇索引 vs 二级索引

InnoDB **数据本身就是按主键排好序存的**，主键索引的叶子节点 = 完整行数据。这叫**聚簇索引 (clustered index)**。

二级索引（也叫辅助索引）的叶子节点 **不存完整行**，只存：

```
二级索引叶子节点 = (索引列值, 主键值)
```

所以普通查询走二级索引时：

```
1. 先在二级索引 B+ 树里查到主键值
2. 拿主键值再去聚簇索引 B+ 树查完整行  ← 这一步叫"回表"(table lookup)
```

回表是二级索引性能的天敌。一次 `SELECT * WHERE name = 'tom'`：
- 二级索引扫描：3~4 IO
- 每个匹配项回表：3~4 IO
- 匹配 100 行 → 总共 300+ IO

**这是为什么覆盖索引那么重要**——后面 1.4 详谈。

### 1.3 索引页的结构

打开一个 InnoDB 索引页，它不是简单"一堆记录"，而是有严格的物理布局：

```
┌─────────────────────────────────────────┐
│ File Header (38 字节)                   │ 页号、上下页指针、checksum
├─────────────────────────────────────────┤
│ Page Header (56 字节)                   │ 记录数、free 空间、最后插入位置
├─────────────────────────────────────────┤
│ Infimum + Supremum (26 字节)            │ ★ 虚拟的"最小"和"最大"记录
├─────────────────────────────────────────┤
│ User Records (按插入顺序堆放)           │ 实际数据
│   ↓ 但通过链表按主键有序串起来          │
├─────────────────────────────────────────┤
│ Free Space                              │
├─────────────────────────────────────────┤
│ Page Directory (slot 数组)              │ ★ 二分查找的"目录"
├─────────────────────────────────────────┤
│ File Trailer (8 字节)                   │ checksum 副本
└─────────────────────────────────────────┘
```

几个反直觉的点：

**Infimum 和 Supremum 是虚拟记录**，物理存在但没有实际数据。Infimum 比页内任何记录都小，Supremum 比任何都大。它们是链表的起点和终点哨兵。

**User Records 在物理上是按插入顺序堆放的**（heap），但通过 `next_record` 字段串成按主键有序的单向链表。所以"页内顺序" ≠ "物理顺序"。

**Page Directory 是稀疏目录**：每 4~8 条记录抽一个 slot，slot 数组按主键有序。页内查找的算法是 **先在 slot 数组上二分定位到一个区间（O(log slot_count)），再在区间内顺着链表扫一段（O(几条记录)）**。

> 这是一个"二分 + 顺序扫描"的混合体。纯链表 O(n) 太慢，纯二分要求物理有序成本太高，slot + 链表是性价比最高的折中。

### 1.4 联合索引的字典序排序

联合索引 `(a, b, c)` 在 B+ 树里**按 (a, b, c) 字典序排序**：

```
(1, 'a', 100)
(1, 'a', 200)
(1, 'b', 50)
(1, 'b', 150)
(2, 'a', 10)
(2, 'c', 20)
```

这就是"最左前缀原则"的物理基础：
- `WHERE a = 1` → B+ 树连续区间，可用
- `WHERE a = 1 AND b = 'a'` → 更窄的连续区间，可用
- `WHERE a = 1 AND b = 'a' AND c > 100` → 还能用，但 c 部分是范围
- `WHERE b = 'a'` → 不连续！跨多个 a 值都有 b='a'，**不可用**
- `WHERE a = 1 AND c = 100` → a 部分可用、c 部分不可用，**c 在叶子层过滤**（这就是 ICP 的舞台）

**覆盖索引 (covering index)** 的精确定义：**所查询的所有列都在该索引的叶子节点里**。对于二级索引 `(a, b, c)`，叶子节点存的是 `(a, b, c, 主键)`。所以：

```sql
SELECT a, b, c FROM t WHERE a = 1;        -- ✅ 覆盖
SELECT id, a, b FROM t WHERE a = 1;       -- ✅ 覆盖（主键也在叶子里）
SELECT * FROM t WHERE a = 1;              -- ❌ 要回表（叶子里没其他列）
SELECT a, b, d FROM t WHERE a = 1;        -- ❌ d 不在索引里
```

`EXPLAIN` 看 Extra 列：`Using index` 就是覆盖索引（不回表）。

### 1.5 索引下推（ICP）

ICP（Index Condition Pushdown）是 MySQL 5.6 引入的优化。理解它必须先理解**没有 ICP 时的尴尬**。

假设有联合索引 `(name, age)`，SQL：

```sql
SELECT * FROM employee WHERE name LIKE '张%' AND age = 30;
```

**没有 ICP 时**（5.6 之前的逻辑）：

```
1. 存储引擎：扫描二级索引，name LIKE '张%' 是范围条件 → 找到所有 name 以 '张' 开头的索引项
   假设有 1 万个 → 全部回表
2. 回表：拿主键去聚簇索引读 1 万行完整数据传给 server 层
3. server 层：对这 1 万行做 age = 30 的过滤，假设只剩 100 行
```

代价：1 万次回表，但只需要 100 行。**回表浪费了 99%**。

**有了 ICP**（5.6+）：

```
1. 存储引擎：扫描二级索引时，发现 (name, age) 都在索引里
   边扫描边判断 age = 30，**只把 age = 30 的索引项**拿去回表
   1 万 → 100
2. 回表：100 次
```

回表次数从 1 万降到 100，**提升 100 倍**。

`EXPLAIN` 看 Extra 列：`Using index condition` 就是 ICP 生效。

ICP 的限制：
- 只能用在**索引列上**的条件（不在索引里的列没办法在引擎层过滤）
- InnoDB 和 MyISAM 都支持，NDB 不支持
- 触发器、子查询里的列不支持下推

### 1.6 优化器的代价模型

优化器选索引不是"随机"的，它有一个明确的代价公式（MySQL 8.0）：

```
total_cost = io_cost + cpu_cost

io_cost  ≈ 需要读取的页数 × IO_BLOCK_READ_COST
           （随机 IO 1.0，顺序 IO 0.25 左右；可通过 mysql.engine_cost 配）

cpu_cost ≈ 处理的行数 × ROW_EVALUATE_COST
           （默认 0.1，可通过 mysql.server_cost 配）
```

代价模型的核心数据来源：**统计信息**。

```sql
SHOW INDEX FROM t;
-- Cardinality 列就是该索引的"基数估计"
```

```sql
SELECT * FROM mysql.innodb_index_stats WHERE table_name = 't';
-- 更详细的统计：每个索引前缀的 distinct 值估计
```

InnoDB 通过**采样若干叶子页**估算基数（`innodb_stats_persistent_sample_pages`，默认 20）。表很大、数据分布偏斜时，**估算可能严重失真** —— 这就是案例 ⑥ 的根。

代价模型是优化器决策的"上帝视角"，但上帝看的是估算值，**估算错了就选错索引**。这是 SQL 优化最深的坑之一。

> **小结**：B+ 树解决"找一行"的物理问题；ICP 解决"扫一段"的回表问题；代价模型解决"用哪个索引"的策略问题。三者中**代价模型最不稳定**——它依赖统计信息，这是大多数"优化器选错索引"案例的根。

---

## 2. 源码切片：InnoDB 索引查询关键路径

> 基于 MySQL 8.0 InnoDB 源码。注释为本文添加。

### 2.1 row_search_mvcc：索引查询的入口

```cpp
// storage/innobase/row/row0sel.cc 简化
dberr_t row_search_mvcc(
    byte *buf,                       // 输出缓冲：放查到的行
    page_cur_mode_t mode,            // PAGE_CUR_GE / PAGE_CUR_LE / PAGE_CUR_G / ...
    row_prebuilt_t *prebuilt,        // 预构建的查询上下文（包含 search tuple）
    ulint match_mode,                // 0 / ROW_SEL_EXACT / ROW_SEL_EXACT_PREFIX
    const ulint direction) {         // 0=新查询  ROW_SEL_NEXT/PREV=继续上一次

    dict_index_t *index = prebuilt->index;
    btr_pcur_t *pcur = prebuilt->pcur;
    ReadView *trx_view = nullptr;

    if (prebuilt->select_lock_type == LOCK_NONE
        && trx->isolation_level != TRX_ISO_READ_UNCOMMITTED) {
        trx_assign_read_view(trx);   // ① 一致性读：分配 ReadView
        trx_view = trx->read_view;
    }

    // ② B+ 树下降到目标位置
    btr_pcur_open_with_no_init(index, search_tuple, mode,
                               BTR_SEARCH_LEAF, pcur, ...);

    // ③ 如果走了二级索引且需要回表，准备 secondary→clustered 的转换
    rec_t *rec = btr_pcur_get_rec(pcur);
    if (!index->is_clustered() && prebuilt->need_to_access_clustered) {
        rec = row_sel_get_clust_rec_for_mysql(prebuilt, index, rec, ...);
    }

    // ④ ICP 检查：在引擎层做条件过滤
    if (prebuilt->idx_cond) {
        ICP_RESULT result = innobase_index_cond(prebuilt);
        if (result == ICP_NO_MATCH) {
            goto next_rec;           // 跳过，不回表
        }
    }

    // ⑤ 拷贝行到 mysql 格式输出
    row_sel_store_mysql_rec(buf, prebuilt, rec, ...);
    return DB_SUCCESS;
}
```

**逐段点透**：

- **②`btr_pcur_open_with_no_init`** 是 B+ 树下降入口，会一路从根走到叶。
- **③`need_to_access_clustered`** 决定要不要回表。如果是覆盖索引，这里 = false，省掉一次 B+ 树下降。
- **④ ICP 在第 ④ 步**——**在回表之前**。这是 ICP 的本质：把过滤条件从 server 层推到引擎层、从"回表后"推到"回表前"。
- **⑤** 是格式转换：InnoDB 内部行格式（`rec_t`）和 MySQL 上层用的格式（`mysql.row`）不同，需要拷贝。

### 2.2 page_cur_search：页内二分查找

```cpp
// storage/innobase/page/page0cur.cc 简化
void page_cur_search_with_match(
    const buf_block_t *block,
    const dict_index_t *index,
    const dtuple_t *tuple,           // 要找的 key
    page_cur_mode_t mode,
    ulint *iup_matched_fields,       // 输入/输出：上界已匹配字段数
    ulint *ilow_matched_fields,
    page_cur_t *cursor) {

    page_t *page = block->frame;
    ulint low = 0;
    ulint up  = page_dir_get_n_slots(page) - 1;   // ★ 在 slot 数组上二分

    // ① slot 数组二分
    while (up - low > 1) {
        ulint mid = (low + up) / 2;
        page_dir_slot_t *slot = page_dir_get_nth_slot(page, mid);
        rec_t *mid_rec = page_dir_slot_get_rec(slot);

        int cmp = cmp_dtuple_rec_with_match(tuple, mid_rec, ...);
        if (cmp > 0)       low = mid;
        else if (cmp < 0)  up  = mid;
        else { up = low = mid; break; }
    }

    // ② 在二分锁定的小区间内顺序扫
    rec_t *rec = page_dir_slot_get_rec(page_dir_get_nth_slot(page, low));
    while (rec != page_dir_slot_get_rec(page_dir_get_nth_slot(page, up))) {
        int cmp = cmp_dtuple_rec_with_match(tuple, rec, ...);
        if (cmp <= 0) break;
        rec = page_rec_get_next(rec);   // 顺着 next_record 链表走
    }

    cursor->rec = rec;
}
```

**逐段点透**：

- **slot 数组的妙处**：页内 100 多条记录，slot 大约 25 个左右。先在 25 个上二分（5 次比较），再在 4~8 条记录的小区间内顺序扫，比直接对 100 条二分**少一次缓存未命中**（链表节点散在页内各处，slot 数组连续紧凑）。
- **`cmp_dtuple_rec_with_match`** 比较时记录"已匹配多少字段"，这个数后面会作为联合索引的"前缀匹配长度"传出去。
- **整个函数没有锁**，因为页已经被上层 `mtr_t`（mini-transaction）latch 住了。

### 2.3 btr_cur_search_to_nth_level：B+ 树下降

```cpp
// storage/innobase/btr/btr0cur.cc 大幅简化
void btr_cur_search_to_nth_level(
    dict_index_t *index,
    ulint level,                     // 0 = 叶子层
    const dtuple_t *tuple,
    page_cur_mode_t mode,
    ulint latch_mode,                // BTR_SEARCH_LEAF / BTR_MODIFY_LEAF / BTR_INSERT
    btr_cur_t *cursor,
    mtr_t *mtr) {

    page_id_t page_id(space_id, dict_index_get_page(index));   // ① 从根开始
    ulint height = ULINT_UNDEFINED;

    for (;;) {
        buf_block_t *block = buf_page_get_gen(page_id, ...);   // ② 拿页（命中 buffer pool 或读盘）
        page_t *page = buf_block_get_frame(block);

        if (height == ULINT_UNDEFINED) {
            height = btr_page_get_level(page);                 // 第一次进来，记下树高
        }

        page_cur_search_with_match(block, index, tuple, mode,
                                   &up_match, &low_match,
                                   page_cursor);              // ③ 页内查找

        if (height == 0) break;       // ④ 到叶子层，结束

        // ⑤ 非叶子层：拿出指向下一层的页号，继续
        rec_t *node_ptr = page_cur_get_rec(page_cursor);
        page_id.set_page_no(btr_node_ptr_get_child_page_no(node_ptr, ...));
        height--;
    }

    cursor->page_cur = *page_cursor;
}
```

**逐段点透**：

- **②`buf_page_get_gen`** 是 buffer pool 的入口。命中（hit）时直接返回内存指针，未命中（miss）时同步从磁盘读。**生产里 IO 几乎都发生在叶子层**，因为根和中间层热度极高，常驻 LRU 头部。
- **③ 页内查找的结果是一个 cursor**（指向某条记录或某个 slot），下一层用这个 cursor 指向的"子页号"继续。
- **④ 树高的循环**：因为 InnoDB 索引高度 3~4，这个循环最多跑 3~4 次，每次一个页 IO（最差情况）。
- **`latch_mode`** 决定了是 SX latch（搜索）还是 X latch（修改），影响 SMO（split/merge operation）的并发度。这就是 InnoDB 的 **B-link tree-like** 优化的入口。

### 2.4 ICP：row_search_idx_cond_check

```cpp
// storage/innobase/row/row0sel.cc 简化
ICP_RESULT row_search_idx_cond_check(
    byte *mysql_rec,                 // 已转好格式的行（只填了索引覆盖的列）
    row_prebuilt_t *prebuilt,
    const rec_t *rec,
    const ulint *offsets) {

    if (!prebuilt->idx_cond) {
        return ICP_MATCH;            // 没设 ICP 条件，直接通过
    }

    // ① 把当前 rec 里的索引列拷贝到 mysql_rec
    if (!row_sel_store_mysql_rec(mysql_rec, prebuilt, rec, nullptr,
                                 prebuilt->index, offsets, ...)) {
        return ICP_NO_MATCH;
    }

    // ② 调 server 层注册的"判断函数"
    ICP_RESULT result = innobase_index_cond(prebuilt);

    switch (result) {
    case ICP_MATCH:
    case ICP_OUT_OF_RANGE:           // ★ 比 mysql 写入的范围都大，可以提前结束扫描
        return result;
    case ICP_NO_MATCH:
        return ICP_NO_MATCH;         // 不匹配，跳过这行（不回表）
    }
}
```

**逐段点透**：

- **`innobase_index_cond`** 是个回调函数指针，由 server 层注册到 prebuilt。它本质是把"上层的过滤条件"做成 lambda 传到引擎层。
- **`ICP_OUT_OF_RANGE` 是隐藏彩蛋**：比如 `WHERE a = 1 AND b > 100`，扫到 `b = 200, 300, ...` 都在范围内，但扫到下一个 `a = 2` 时直接 OUT_OF_RANGE，**整个查询结束**，不必继续扫。这是 ICP 带来的"提前终止"能力。
- 如果 ICP 返回 NO_MATCH，**主表完全不读**——这是 ICP 节省 IO 的本质。

### 2.5 优化器代价：best_access_path

```cpp
// sql/sql_planner.cc 简化
void Optimize_table_order::best_access_path(JOIN_TAB *s, ...) {
    double best_read_cost = DBL_MAX;
    Key_use *best_key = nullptr;

    for (Key_use *keyuse = s->keyuse; keyuse; keyuse++) {
        KEY *key = &table->key_info[keyuse->key];

        // ① 估算：用这个索引，要扫多少行
        ha_rows records = estimate_records(s, keyuse, ...);

        // ② 估算：要读多少页 → IO 成本
        double io_cost = records * cost_model->page_read_cost(...);

        // ③ CPU 成本：评估这些行的代价
        double cpu_cost = records * ROW_EVALUATE_COST;

        double total = io_cost + cpu_cost;

        if (total < best_read_cost) {
            best_read_cost = total;
            best_key = keyuse;
        }
    }

    // ④ 跟全表扫比一比
    double table_scan_cost = table->file->table_scan_cost().total_cost();
    if (best_read_cost > table_scan_cost) {
        best_key = nullptr;          // ★ 索引扫还不如全表扫
    }

    s->position->key = best_key;
}
```

**逐段点透**：

- **`estimate_records`** 是整个优化器最容易出错的函数。它依赖 `index->stat_n_diff_key_vals` 和 `range_optimizer` 的估计。**统计信息陈旧 → 估错 → 选错索引**。
- **④ 跟全表扫比** 是优化器经常"出乎意料"选全表扫的根因。如果索引扫预估代价 > 全表扫，就走全表扫。某些 LIMIT 查询里这个比较经常出 bug。
- **`cost_model->page_read_cost`** 区分顺序 IO 和随机 IO。机械盘时代这个比例 1:10，SSD 时代实际接近 1:1，但 MySQL 的默认参数还停留在机械盘时代——这是为什么 SSD 部署后某些查询要手动调 `engine_cost`。

### 2.6 MRR：批量回表的优化

```cpp
// storage/innobase/handler/ha_innodb.cc 简化
int ha_innobase::multi_range_read_init(
    RANGE_SEQ_IF *seq, void *seq_init_param,
    uint n_ranges, uint mode, ...) {

    if (mode & HA_MRR_USE_DEFAULT_IMPL) {
        return handler::multi_range_read_init(seq, ..., mode, ...);
    }

    // ① 收集所有要查的范围（来自 IN / OR / 二级索引扫描）
    DsMrr_impl mrr_iter;
    mrr_iter.dsmrr_init(this, seq, seq_init_param, n_ranges, mode);

    // ② 批量从二级索引拿到所有匹配的主键
    std::vector<key_id_t> keys;
    while (mrr_iter.next_key(&key)) {
        keys.push_back(key);
    }

    // ③ ★ 把主键按聚簇索引顺序排序
    std::sort(keys.begin(), keys.end());

    // ④ 顺序扫聚簇索引取行
    for (auto k : keys) {
        clust_lookup(k);             // 这一系列查找因为有序，对 buffer pool 友好
    }
    return 0;
}
```

**逐段点透**：

- **③ 排序是 MRR 的本质**：二级索引扫描出来的主键值是**按二级索引顺序**的，不一定按主键顺序。直接逐个回表 → 主键扫描跳来跳去 → 随机 IO。先排序再回表 → 主键访问近似顺序 → 顺序 IO。
- 机械盘上 MRR 收益巨大（顺序 IO vs 随机 IO 差 50 倍）；SSD 上收益小很多（差 2~5 倍），所以 SSD 时代很多 case MRR 反而被关掉。
- 控制参数：`SET optimizer_switch='mrr=on,mrr_cost_based=on'`。`cost_based` 决定优化器自己判断要不要用 MRR。

> **小结**：从入口 `row_search_mvcc` → 到 B+ 树下降 → 到页内 slot 二分 → 到 ICP 在引擎层过滤 → 到 MRR 批量回表，整条链路的每个环节都有可观测、可调优的点。看懂这条链路是排查"索引为什么慢"的基础。

---

## 3. 实测数据：用 sysbench / EXPLAIN 验证

### 3.1 单列 / 联合 / 覆盖索引的成本对比

测试表：

```sql
CREATE TABLE bench (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT,
  status TINYINT,
  amount DECIMAL(10,2),
  created_at DATETIME,
  remark VARCHAR(200),
  KEY idx_uid (user_id),
  KEY idx_uid_status (user_id, status),
  KEY idx_uid_status_amt (user_id, status, amount)
) ENGINE=InnoDB;

-- 灌 1000 万行（user_id 分布 0~10000，每个 user 平均 1000 行）
```

跑同一个查询的不同变体：

```sql
-- 查询 A：单列索引
SELECT id, status, amount FROM bench WHERE user_id = 5000;

-- 查询 B：联合索引
SELECT id, status, amount FROM bench WHERE user_id = 5000 AND status = 1;

-- 查询 C：覆盖索引（idx_uid_status_amt 覆盖了 user_id, status, amount，主键 id 自带）
SELECT id, status, amount FROM bench WHERE user_id = 5000 AND status = 1;
-- ★ 优化器会自动选 idx_uid_status_amt 因为覆盖
```

`EXPLAIN ANALYZE` 在 MySQL 8.0 上的典型输出（M2 Mac、buffer pool 充足）：

| 查询 | 选用索引 | 扫描行数 | Extra | 实测耗时 |
| --- | --- | --- | --- | --- |
| A `idx_uid` | `idx_uid` | 1000 | NULL（要回表） | **18 ms** |
| B `idx_uid_status` | `idx_uid_status` | 100 | NULL（要回表） | 2.1 ms |
| C 覆盖 | `idx_uid_status_amt` | 100 | **Using index** | **0.6 ms** |

**结论**：
- 单列索引 → 联合索引：扫描行数从 1000 → 100，**10x 提升**
- 联合索引 → 覆盖索引：消除回表，**3.5x 提升**

### 3.2 ICP 启用前后对比

测试 LIKE 前缀 + 联合索引：

```sql
SET optimizer_switch = 'index_condition_pushdown=off';
EXPLAIN ANALYZE SELECT * FROM bench WHERE user_id BETWEEN 1 AND 100 AND status = 1;
-- Extra: Using where    扫描 ~10万行  耗时 320ms

SET optimizer_switch = 'index_condition_pushdown=on';
EXPLAIN ANALYZE SELECT * FROM bench WHERE user_id BETWEEN 1 AND 100 AND status = 1;
-- Extra: Using index condition    回表 ~1万行  耗时 35ms
```

ICP 把回表从 10 万降到 1 万，**~10x 提升**。这只是单表，多表 join 时收益更大。

### 3.3 不同选择性下的索引效果

往 1000 万行表上加一列 `gender TINYINT`（0/1 各半），建索引 `KEY idx_g (gender)`：

```sql
EXPLAIN SELECT * FROM bench WHERE gender = 1;
```

可能的输出：

```
type: ALL                     ← 全表扫
key:  NULL
rows: 9876543
Extra: Using where
```

**优化器拒绝用这个索引**。原因：

```
选择性 = distinct(gender) / total = 2 / 10,000,000 ≈ 0
代价 = 索引扫一半 + 回表一半 ≈ 5,000,000 × (1 + 4 IO/行)
     ≈ 远大于全表扫的 5,000,000 IO
```

**经验法则**：
- 选择性 > 30% → 索引几乎无用，不要建
- 选择性 5%~30% → 视情况，覆盖索引才划算
- 选择性 < 5% → 一般有用
- 选择性 < 0.1%（如订单号） → 必建

### 3.4 INDEX SKIP SCAN 的边界

MySQL 8.0 引入的"跳跃扫描"：联合索引 `(a, b)`，缺第一列 a 时仍能用，前提是 a 的基数极低。

```sql
CREATE TABLE t (a TINYINT, b INT, KEY idx_ab (a, b));
-- a 只有 0/1 两个值；b 范围 1~100万
INSERT INTO t SELECT FLOOR(RAND()*2), FLOOR(RAND()*1000000) FROM ...;

EXPLAIN SELECT * FROM t WHERE b = 50000;
-- Extra: Using index for skip scan    ← ★ 用了 idx_ab
```

原理：优化器把查询拆成 `a IN (0, 1) AND b = 50000`，相当于自动加上了"前缀枚举"。

**适用场景极窄**：a 必须基数很小（< 100），否则枚举开销 > 全索引扫。

```sql
EXPLAIN SELECT * FROM t WHERE b = 50000;
-- 如果 a 基数大，会退化成 type: ALL 或 type: index
```

### 3.5 EXPLAIN 关键列详解

| 列 | 含义 | 优化目标 |
| --- | --- | --- |
| **type** | 访问类型 | 至少要到 `range`，最好 `ref` 或 `eq_ref` |
| **possible_keys** | 可能用的索引 | 不为空 |
| **key** | 实际选的索引 | 非 NULL |
| **key_len** | 用了索引的多少字节 | 越接近预期越好（联合索引部分用到时这里会反应） |
| **rows** | 估算扫描行数 | 越小越好；和实际差太远要 `ANALYZE TABLE` |
| **Extra** | 额外信息 | 见下表 |

type 从好到坏：

```
const > eq_ref > ref > range > index > ALL
```

Extra 常见值与含义：

| Extra | 含义 | 好/坏 |
| --- | --- | --- |
| **Using index** | 覆盖索引，不回表 | 好 |
| **Using index condition** | ICP 生效 | 好 |
| **Using where** | server 层做了过滤（说明引擎层没拦截到位） | 中 |
| **Using filesort** | 排序用了临时空间 | 差 |
| **Using temporary** | 用了临时表 | 差 |
| **Using join buffer (Block Nested Loop)** | 没走索引的 join | 很差 |

> **口诀**：`type ≥ range && rows < 1万 && Extra 含 'Using index'` 是大部分查询的合格线。否则需要重看索引设计。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 隐式类型转换 —— "phone 是 varchar，传 long 上去就全表扫"

#### 现象

DBA 抓到一条慢查询：

```sql
SELECT * FROM user WHERE phone = 13800138000;
-- 5 秒才出结果
```

但 phone 列上明明有索引。

#### 根因

```sql
SHOW CREATE TABLE user\G
-- phone VARCHAR(20) NOT NULL,
-- KEY idx_phone (phone)
```

phone 是 varchar，传入是数字（应用层用 long 拼 SQL）。MySQL 的"类型转换"优先级：**字符串 → 数字**（IEEE 754 规则）。

所以这条 SQL 等价于：

```sql
SELECT * FROM user WHERE CAST(phone AS DOUBLE) = 13800138000;
```

`CAST(phone AS DOUBLE)` 是函数运算 → **索引失效**（B+ 树是按原始 varchar 字典序排，函数转换后顺序乱了）。

`EXPLAIN` 验证：

```
type: ALL
key:  NULL
Extra: Using where
```

#### 修复

参数对齐类型：

```sql
SELECT * FROM user WHERE phone = '13800138000';   -- ★ 字符串
```

应用层（MyBatis 例）：

```xml
<!-- ❌ 错：${} 直接拼，丢类型 -->
SELECT * FROM user WHERE phone = ${phone}

<!-- ✅ 对：#{} 预编译，自动用对应类型 -->
SELECT * FROM user WHERE phone = #{phone}
```

Java 端的 phone 也应该是 String，不要用 long（手机号本身就不是"数字"，开头不能为 0 等业务约束）。

#### 验证

```sql
EXPLAIN SELECT * FROM user WHERE phone = '13800138000';
-- type: ref, key: idx_phone, rows: 1
```

> **口诀**：**列是什么类型，参数就传什么类型**。MyBatis 用 `#{}`，原生 JDBC 用 PreparedStatement，避免拼字符串。

---

### 案例 ② LIKE '%xxx' —— "前导通配符索引必败"

#### 现象

```sql
SELECT * FROM article WHERE title LIKE '%MySQL%';
-- 全表扫 50 万行，3 秒
```

#### 根因

B+ 树的有序性是从**字符串开头**算起的。`LIKE 'MySQL%'` 能利用，因为它退化成范围扫描 `[MySQL, MySQM)`；但 `LIKE '%MySQL%'` 缺少左前缀，B+ 树**没有**任何顺序可用 → 全表扫。

#### 修复

**方案 A**：反转列。如果是查"以 xxx 结尾"，可以加一个 `title_reversed` 列存反转字符串，建索引，查询时也反转。

```sql
ALTER TABLE article ADD COLUMN title_reversed VARCHAR(500), ADD KEY idx_tr (title_reversed);
UPDATE article SET title_reversed = REVERSE(title);
SELECT * FROM article WHERE title_reversed LIKE REVERSE('%MySQL') ;
-- 等价 WHERE title LIKE '%MySQL'，但走索引
```

**方案 B**：全文索引（FULLTEXT）。

```sql
ALTER TABLE article ADD FULLTEXT INDEX ft_title (title) WITH PARSER ngram;
SELECT * FROM article WHERE MATCH(title) AGAINST('+MySQL' IN BOOLEAN MODE);
```

中文必须用 ngram 分词器，且 SET `ngram_token_size=2`。

**方案 C**：换专业搜索引擎（ES / Manticore）。LIKE '%xxx%' 在大数据量下永远是 MySQL 的痛点。

#### 验证

```sql
EXPLAIN SELECT * FROM article WHERE title LIKE 'MySQL%';   -- range
EXPLAIN SELECT * FROM article WHERE title LIKE '%MySQL%';  -- ALL
```

> **口诀**：**LIKE 后通配符**（`'xxx%'`）能用索引；**前通配符**（`'%xxx'`）必失效。模糊搜索别硬怼 MySQL。

---

### 案例 ③ 函数运算导致索引失效 —— "YEAR(create_time)"

#### 现象

报表 SQL：

```sql
SELECT * FROM `order` WHERE YEAR(create_time) = 2024;
-- 5 秒，全表扫
```

create_time 上有索引 idx_ct。

#### 根因

`YEAR(create_time)` 把列扔进函数 → MySQL 必须对每一行计算 YEAR(...) 再比较 → B+ 树按 create_time 原值排序，但 YEAR 之后的值"散在原值的多处"，B+ 树无法二分定位 → 全表扫 + 计算。

同类陷阱：

```sql
WHERE DATE(create_time) = '2024-01-01'
WHERE SUBSTRING(name, 1, 3) = 'abc'
WHERE id + 1 = 100              -- 算术也算"函数"
WHERE LOWER(email) = 'tom@x.com'
```

#### 修复

**方案 A**（首选）：把函数移到右边。

```sql
SELECT * FROM `order`
WHERE create_time >= '2024-01-01' AND create_time < '2025-01-01';
```

这是个"按 create_time 范围"的查询，**完美利用 idx_ct**。

类似地：

```sql
WHERE create_time >= '2024-01-01' AND create_time < '2024-01-02'   -- 替代 DATE(...)
WHERE id = 99                                                       -- 替代 id + 1 = 100
WHERE email = LOWER('Tom@X.com')                                    -- 假设入参可标准化
```

**方案 B**：MySQL 8.0 函数索引 / 表达式索引。

```sql
ALTER TABLE `order` ADD KEY idx_year ((YEAR(create_time)));
-- 现在 WHERE YEAR(create_time) = 2024 能走 idx_year
```

代价：每次 INSERT/UPDATE 要计算函数值并维护这个索引；存储多一份。

**方案 C**：业务侧把 year 单独存一列。

```sql
ALTER TABLE `order` ADD COLUMN order_year SMALLINT GENERATED ALWAYS AS (YEAR(create_time)) STORED;
ALTER TABLE `order` ADD KEY idx_oy (order_year);
```

`STORED` 列等价于"自动维护的物化列"。

#### 验证

```sql
EXPLAIN SELECT * FROM `order` WHERE create_time >= '2024-01-01' AND create_time < '2025-01-01';
-- type: range, key: idx_ct
```

> **口诀**：**列上不要套函数**。能改成范围条件就改，改不了就上函数索引。

---

### 案例 ④ OR 一边没索引 —— "全表扫"

#### 现象

```sql
SELECT * FROM user WHERE name = 'tom' OR mobile = '13800138000';
-- 慢
```

name 和 mobile 都有单列索引。

#### 根因

OR 的优化器逻辑：**只要任意一边没索引，整条 OR 退化成全表扫**。

如果两边都有索引，MySQL 会尝试 **index_merge**：

```
EXPLAIN SELECT * FROM user WHERE name = 'tom' OR mobile = '13800138000';
-- type: index_merge
-- Extra: Using union(idx_name, idx_mobile); Using where
```

但 index_merge 对结果做去重和合并，开销不小，常常不如 UNION ALL 快。

#### 修复

**方案 A**：拆 UNION ALL。

```sql
SELECT * FROM user WHERE name = 'tom'
UNION ALL
SELECT * FROM user WHERE mobile = '13800138000' AND name <> 'tom';
```

`AND name <> 'tom'` 是去重，避免双方都匹配的行重复出现。

**方案 B**：`UNION`（自动去重，但要加 sort，慢）。

**方案 C**：业务上避免 OR 结构。

```sql
-- 把"按 name 或 mobile 查"拆成两个接口，让前端按实际场景选一个
```

**方案 D**：数据冗余。建一个 `search_key` 列，把 name 和 mobile 都丢进去，建联合索引。但这是反范式，慎用。

#### 验证

```sql
EXPLAIN SELECT * FROM user WHERE name = 'tom'
UNION ALL
SELECT * FROM user WHERE mobile = '13800138000' AND name <> 'tom';
-- 两个 SELECT 各自 type: ref，远好于 index_merge
```

> **口诀**：**OR 是优化器的天敌**。能拆 UNION ALL 就拆；index_merge 要看 EXPLAIN 验证，不要想当然。

---

### 案例 ⑤ 联合索引顺序写错 —— "最左前缀失效"

#### 现象

线上业务反馈某个分页接口慢。SQL：

```sql
SELECT * FROM `order` WHERE status = 1 AND user_id = 5000 ORDER BY create_time DESC LIMIT 20;
```

表上有索引 `KEY idx_uid_status_ct (user_id, status, create_time)`。`EXPLAIN`：

```
type: ALL
key: NULL
```

奇怪，明明 user_id 和 status 都在索引最左两列里。

#### 根因（多个）

**根因 1**：常常是另一个误以为"等价"的索引被误用了。仔细看 SHOW CREATE TABLE，结果有：

```sql
KEY idx_status_uid (status, user_id)      -- ★ 最左是 status
KEY idx_uid_status_ct (user_id, status, create_time)
```

优化器评估两个索引代价时，估错了。**真正的问题不是"最左前缀失效"，而是"选错索引"**。

**根因 2**（更隐蔽）：以为联合索引顺序"无所谓"，实际上：

```sql
KEY idx_status_uid (status, user_id)
SELECT ... WHERE user_id = 5000;          -- 不能用此索引，最左是 status
```

如果只有 `idx_status_uid`，缺 `WHERE status` 时这个索引完全用不上。

**根因 3**：等值 + 范围混合时，列的顺序决定能用多少：

```sql
KEY idx_a_b_c (a, b, c)
WHERE a = 1 AND b > 10 AND c = 5;
-- 用 a + b（范围之前的等值），c 在 ICP 里过滤，不走索引 seek
```

如果改成 `(a, c, b)`，则 a + c 都能用 seek，b 范围在 ICP 过滤，效果更好。

#### 修复

```sql
-- 删掉错的索引，加上对的
ALTER TABLE `order` DROP INDEX idx_status_uid;
-- 排序是 create_time DESC，把它放联合索引最后能用上 ORDER BY 优化
ALTER TABLE `order` ADD KEY idx_uid_status_ct (user_id, status, create_time);
```

设计联合索引列序的口诀：
1. 等值放前 (`=` 和 `IN`)
2. 范围放中（一个范围条件之后的列只能 ICP，不能 seek）
3. ORDER BY 放后（且方向一致 ASC/DESC）

#### 验证

```sql
EXPLAIN SELECT * FROM `order` WHERE status = 1 AND user_id = 5000 ORDER BY create_time DESC LIMIT 20;
-- type: ref, key: idx_uid_status_ct, key_len = ...,  Extra: Backward index scan
```

`Backward index scan` 是 8.0 引入的"反向走索引"，可以避免 filesort。

> **口诀**：联合索引列序 = **等值优先 → 范围 → 排序**。一个范围条件之后的列基本都只能 ICP，不能 seek。

---

### 案例 ⑥ 优化器选错索引 —— "今天走 A，明天走 B"

#### 现象

同一条 SQL：

```sql
SELECT * FROM `order` WHERE user_id = 5000 AND status = 1 ORDER BY id DESC LIMIT 20;
```

昨天用 `idx_uid_status` 跑得很快（5ms），今天突然走了 `PRIMARY` 反向扫描，耗时变 5 秒。

```sql
EXPLAIN SELECT ...;
-- 昨天：key = idx_uid_status, rows = 100
-- 今天：key = PRIMARY,        rows = 50  ← 估算严重错误
```

#### 根因

优化器的"反向扫描 + LIMIT" 判断逻辑：

```
如果 ORDER BY id DESC LIMIT 20，优化器会想：
   "我反向扫主键，扫到 20 行符合 user_id=5000 AND status=1 就停"
   "估计要扫 50 行就够了"

但如果实际数据里 user_id=5000 AND status=1 的记录非常稀疏，
反向扫主键可能要扫几百万行才凑够 20 行。
```

为什么"昨天行今天不行"？因为：
1. **统计信息更新**了（`innodb_stats_auto_recalc=ON` 默认开），新的统计让优化器换了主意
2. **数据分布变了**（业务跑了一段时间，user_id=5000 的最新订单都在很后面）
3. **buffer pool 状态变了**（重启后冷启动）

#### 修复

**方案 A**：FORCE INDEX。简单粗暴。

```sql
SELECT * FROM `order` FORCE INDEX (idx_uid_status)
WHERE user_id = 5000 AND status = 1 ORDER BY id DESC LIMIT 20;
```

代价：永远不会用其他索引。**未来加了更优的索引也不会用**——这是 FORCE INDEX 最大的反噬。

**方案 B**：扩展索引让它更"诱人"。

```sql
-- 把 id 放进联合索引
ALTER TABLE `order` ADD KEY idx_uid_status_id (user_id, status, id);
```

现在优化器用 `idx_uid_status_id` 时，能直接拿到按 id 排序的结果（联合索引最后一列默认升序，但 8.0 也支持 DESC 索引），不需要回到主键反向扫。

**方案 C**：改写查询，用子查询锁定主键再 join。

```sql
SELECT o.* FROM `order` o
JOIN (
    SELECT id FROM `order`
    WHERE user_id = 5000 AND status = 1
    ORDER BY id DESC LIMIT 20
) t ON o.id = t.id;
```

子查询走 `idx_uid_status`（覆盖到 id），外层用主键 join。`STRAIGHT_JOIN` 可强制 join 顺序。

**方案 D**：定期 ANALYZE TABLE，让统计信息稳定。

```sql
ANALYZE TABLE `order`;
```

并把 `innodb_stats_persistent_sample_pages` 调高（默认 20，调到 200~2000 让采样更准）。

```ini
innodb_stats_persistent = ON
innodb_stats_persistent_sample_pages = 200
```

#### 验证

```sql
-- 用 EXPLAIN FORMAT=JSON 看代价细节
EXPLAIN FORMAT=JSON
SELECT * FROM `order` WHERE user_id = 5000 AND status = 1 ORDER BY id DESC LIMIT 20\G

-- 看 cost_info 字段，对比加 FORCE INDEX 前后的 query_cost
```

> **口诀**：**优化器变心 = 统计信息变了**。FORCE INDEX 能解一时之急，但是**短期方案**。长期靠扩展索引、ANALYZE TABLE、子查询改写。

---

### 案例 ⑦ 索引选择性太低 —— "建了反而更慢"

#### 现象

`gender` 列上建了索引 `idx_g`，发现某些查询 EXPLAIN 居然显示 `key: idx_g`，但跑出来比全表扫还慢。

```sql
SELECT * FROM user WHERE gender = 1 AND register_year = 2024;
-- 用了 idx_g，扫 500 万行，耗时 8 秒
-- 全表扫 1000 万行，耗时 5 秒
```

#### 根因

二级索引扫 500 万 + 回表 500 万次（随机 IO） = 灾难。全表扫顺序读，吞吐高得多。

但优化器为什么选了 idx_g？两种可能：
1. **统计信息严重失真**（采样太少 + 数据偏斜）
2. 业务用了 `USE INDEX` / `FORCE INDEX` 强制

#### 修复

**方案 A**：删掉低选择性索引。

```sql
ALTER TABLE user DROP INDEX idx_g;
```

经验法则（再强调）：**选择性 > 30% 直接不建索引**。

**方案 B**：改成联合索引，让它"有用起来"。

```sql
-- 单独 gender 没用，但 (gender, register_year) 联合可能有用
ALTER TABLE user ADD KEY idx_g_y (gender, register_year);
SELECT * FROM user WHERE gender = 1 AND register_year = 2024;
-- 二级索引扫 50 万（gender=1 中 register_year=2024 的部分），回表 50 万
-- 比全表扫还是慢，但比单 idx_g 好
```

**方案 C**：改成覆盖索引，避免回表代价。

```sql
ALTER TABLE user ADD KEY idx_g_y_n (gender, register_year, name);
SELECT name FROM user WHERE gender = 1 AND register_year = 2024;
-- Using index，飞快
```

**方案 D**：用 IGNORE INDEX 强制忽略错误索引。

```sql
SELECT * FROM user IGNORE INDEX (idx_g) WHERE gender = 1 AND register_year = 2024;
```

#### 验证

`EXPLAIN ANALYZE` 对比"用 / 不用 idx_g"两种执行计划的真实耗时。

> **口诀**：**选择性低的列单建索引是负收益**。要么联合，要么覆盖，要么删掉。

---

### 案例 ⑧ 大表加索引锁表 —— "用 gh-ost / pt-osc"

#### 现象

DBA 在生产对一张 5000 万行的 `orders` 表加索引：

```sql
ALTER TABLE orders ADD INDEX idx_uid (user_id);
```

跑了 30 分钟还没完，业务大量超时告警，被迫 KILL。

#### 根因

虽然 MySQL 5.6+ 的 InnoDB 支持 **Online DDL**（号称"不阻塞读写"），但：

1. **首尾仍要 MDL 写锁**（见 MVCC 文章案例 ⑧）
2. **中间过程虽然不阻塞 DML，但要扫全表 + 写新索引**：5000 万行的 IO + CPU 都在 DDL 上，业务 QPS 受冲击
3. **从库延迟**：DDL 在 binlog 里是个大事件，从库 SQL 线程会卡很久

#### 修复

工业标准：**gh-ost** 或 **pt-online-schema-change**。

**gh-ost 原理**（GitHub 出品）：
1. 创建一张 `_orders_gho`（影子表）
2. 在影子表上跑 ALTER（瞬间完成，因为是空表）
3. 启动一个伪从库，订阅 binlog，把主表的增量改动应用到影子表
4. 同时分批把主表存量数据 chunk 拷到影子表
5. 全部追平后做"瞬间 RENAME"切换：`RENAME TABLE orders TO _orders_old, _orders_gho TO orders`

```bash
gh-ost \
  --user=admin --password=*** \
  --host=db.example.com --database=app --table=orders \
  --alter="ADD INDEX idx_uid (user_id)" \
  --max-load=Threads_running=50 \
  --critical-load=Threads_running=200 \
  --chunk-size=2000 \
  --execute
```

关键参数：
- `--max-load`：检测到主库压力高时**自动暂停**
- `--throttle-flag-file`：提前创建这个文件可以暂停拷贝
- `--cut-over=default`：切换时尝试零阻塞 RENAME

**pt-online-schema-change**（Percona）：
- 用 trigger 同步增量改动（gh-ost 用 binlog，互斥的设计取舍）
- 触发器对源表的写有性能影响
- 配置简单一些

```bash
pt-online-schema-change \
  D=app,t=orders \
  --alter="ADD INDEX idx_uid (user_id)" \
  --execute
```

**生产选哪个**：
- 主从架构、源表写量不大 → pt-osc（trigger 简单可靠）
- 写量很大 / 想要更细的限流 → gh-ost
- 云环境（RDS / Aurora）→ 看云厂商支持哪个

#### 验证

```bash
# 跑过程中观察主从延迟、QPS、CPU
SHOW SLAVE STATUS\G
# 切换瞬间业务可能有 < 1s 的小抖动，可接受

# 检查新索引
SHOW INDEX FROM orders;
```

> **口诀**：**> 1000 万行的表加索引一律用 gh-ost / pt-osc**。直接 ALTER 是给自己挖坑。

---

### 案例 ⑨ 重复索引浪费空间 —— "idx_a 和 idx_a_b 同时存在"

#### 现象

DBA 巡检发现一张表有这样的索引：

```sql
SHOW INDEX FROM user;
-- KEY idx_uid (user_id)
-- KEY idx_uid_status (user_id, status)
-- KEY idx_uid_status_ct (user_id, status, create_time)
```

三个索引的最左列都是 user_id。

#### 根因

`idx_uid` 是 `idx_uid_status` 的**真子集**：任何能用 `idx_uid` 的查询，`idx_uid_status` 都能用，且代价相同（最左前缀等价）。

`idx_uid_status` 又是 `idx_uid_status_ct` 的真子集。

所以 `idx_uid` 和 `idx_uid_status` **完全冗余**。代价：
- 占额外存储空间（一个二级索引大概是表的 5%~30%）
- 每次 INSERT/UPDATE/DELETE 都要维护多一个 B+ 树
- 优化器的索引选择空间变大，更容易选错

#### 修复

```sql
ALTER TABLE user DROP INDEX idx_uid;
ALTER TABLE user DROP INDEX idx_uid_status;
-- 只保留 idx_uid_status_ct
```

但要先验证业务上没有 `FORCE INDEX(idx_uid)` 这种硬编码。

如何系统性发现冗余索引？

```sql
-- MySQL 8.0：sys schema 提供视图
SELECT * FROM sys.schema_redundant_indexes;
```

输出：

```
+--------------+---------------+------------------+
| table_schema | table_name    | redundant_index_name |
+--------------+---------------+------------------+
| app          | user          | idx_uid          |
| app          | user          | idx_uid_status   |
+--------------+---------------+------------------+
```

直接给出可以删的索引名。

**MySQL 8.0 的更稳健做法：INVISIBLE INDEX**。

```sql
ALTER TABLE user ALTER INDEX idx_uid INVISIBLE;
-- 优化器不再考虑这个索引；但索引仍在维护
-- 观察一两周，业务正常 → DROP；出问题 → ALTER ... VISIBLE 立即恢复
ALTER TABLE user DROP INDEX idx_uid;
```

这是删索引的"灰度发布"。

#### 验证

```sql
-- 删后看其他业务 SQL 是否还能正常走索引
EXPLAIN SELECT * FROM user WHERE user_id = 5000;
-- key: idx_uid_status_ct，正常
```

> **口诀**：**联合索引 (a, b, c) 已经覆盖了 (a) 和 (a, b)**。删冗余前用 INVISIBLE 灰度。`sys.schema_redundant_indexes` 是免费的体检报告。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 设计层

- [ ] 表必须有主键，且主键尽量短（推荐 BIGINT 自增；不要用 UUID 字符串作主键）
- [ ] 联合索引列序遵循 **等值优先 → 范围 → 排序**
- [ ] 单字段选择性 > 30% 不建索引
- [ ] 大字段（TEXT/BLOB/VARCHAR(500+)）禁止建索引；非要建用前缀索引（`KEY (name(20))`）
- [ ] 高频查询尽量做覆盖索引，避免回表
- [ ] 同一张表索引数量 ≤ 5（含主键），多的检查冗余

### SQL 层

- [ ] 列上不套函数 / 算术（`YEAR(t)`、`t + 1`、`LOWER(c)` 都是禁区）
- [ ] 参数类型与列类型严格一致（避免隐式转换）
- [ ] LIKE 不能以 `%` 开头
- [ ] OR 两边都有索引；否则拆 UNION ALL
- [ ] IN 列表长度 ≤ 1000（更长改 join 临时表）
- [ ] 避免 `SELECT *`，按需取列以利覆盖索引
- [ ] LIMIT 大偏移用"延迟关联"：先按主键过滤再 join

### 优化器层

- [ ] EXPLAIN 必看：type ≥ range，rows < 1 万，Extra 含 'Using index' 或 'Using index condition'
- [ ] 复杂 SQL 用 `EXPLAIN FORMAT=JSON` 看 cost_info
- [ ] 慎用 FORCE INDEX，永远先尝试 ANALYZE / 改写 / 加新索引
- [ ] `innodb_stats_persistent_sample_pages` 设为 200+ 让统计信息更准
- [ ] `innodb_stats_auto_recalc=ON`，但大表手动 ANALYZE 更稳

### 上线层

- [ ] 大表（> 1000 万行）DDL 一律用 gh-ost / pt-osc
- [ ] 索引变更先在 staging 跑 EXPLAIN 对比
- [ ] 删索引前用 INVISIBLE 灰度 1~2 周
- [ ] 定期跑 `sys.schema_redundant_indexes` 体检
- [ ] 监控 `Innodb_rows_read / Innodb_rows_returned` 比例（> 100 是警报）
- [ ] 慢查询日志开启，`long_query_time = 1`
- [ ] `slow_query_log_file` 定期 pt-query-digest 分析

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### 自适应哈希索引（AHI）

InnoDB 在监测到某索引被频繁等值查询时，会自动在内存里建一个哈希表，把查询从 O(log n) 降到 O(1)。

- 参数：`innodb_adaptive_hash_index = ON`（默认开）
- 副作用：写多场景下 AHI 维护开销 > 收益，可关
- 观察：`SHOW ENGINE INNODB STATUS` 的 `INSERT BUFFER AND ADAPTIVE HASH INDEX` 段

### 全文索引（FULLTEXT）

InnoDB FULLTEXT 用倒排索引（inverted index），底层是 6 张辅助表。

- 中文必须用 ngram 分词器
- `SET ngram_token_size = 2`
- 不适合超大量级（> 千万），仍推荐 ES

### 空间索引（R-tree）

`KEY idx_loc (location) USING RTREE`，专门给地理位置数据用。MyISAM 历史更长，InnoDB 5.7+ 也支持。

- 适合"查矩形范围内的点"这类查询
- 不能做精确点查询，只能做空间关系（CONTAINS / WITHIN / INTERSECTS）

### PostgreSQL 的 GIN / GiST 对比

PG 的索引生态比 MySQL 丰富：
- **GIN (Generalized Inverted Index)**：倒排索引，适合 JSON 字段、全文、数组
- **GiST (Generalized Search Tree)**：通用搜索树，适合空间数据、范围
- **BRIN (Block Range Index)**：超大表的"块级索引"，超省空间

如果业务有大量 JSONB 查询、全文搜索，PG 的 GIN 比 MySQL 全文索引强很多。

### 列式索引（ColumnStore）

行式存储不擅长聚合分析。MariaDB ColumnStore、ClickHouse、Doris 这类列存引擎用专门的"列式索引"（min/max 索引、bloom filter、bitmap）。

如果你的"MySQL 慢"其实是 OLAP 场景，应该把分析查询拆出去到列式数据库。

### 函数索引 / 表达式索引（MySQL 8.0+）

```sql
ALTER TABLE t ADD KEY idx_year ((YEAR(t.created_at)));
```

底层是隐藏的虚拟列 + 索引。可以解 case ③ 的 YEAR() 问题。代价：写入要计算 + 维护。

### 不可见索引（INVISIBLE INDEX）

```sql
ALTER TABLE t ALTER INDEX idx_x INVISIBLE;
```

优化器不再考虑这个索引，但索引仍在维护。**用法**：删索引前的灰度验证、A/B 测试索引效果、临时关掉某个让优化器选错的索引。

### TiDB / 分布式数据库的索引

- TiDB 全局索引 vs 分区索引
- 分布式 join 下索引选择策略
- HTAP 场景下的列存副本

---

## 7. 引用

- MySQL 8.0 InnoDB 源码：[storage/innobase/](https://github.com/mysql/mysql-server/tree/8.0/storage/innobase)
- 《MySQL 实战 45 讲》—— 林晓斌（极客时间）—— 索引部分讲得最透
- 《高性能 MySQL（第 4 版）》—— Schwartz / Zaitsev
- [MySQL 8.0 Reference Manual: Optimization](https://dev.mysql.com/doc/refman/8.0/en/optimization.html)
- gh-ost 设计文档：[GitHub gh-ost](https://github.com/github/gh-ost)
- Percona pt-online-schema-change：[Percona Toolkit](https://www.percona.com/doc/percona-toolkit/3.0/pt-online-schema-change.html)
- 《Database System Concepts》（Silberschatz）—— 第 14 章 Indexing 是经典理论参考

---

> 配套：[MySQL 速记](/article/db/mysql) ｜ [MVCC 与锁深入](/article/db/mysql-mvcc) ｜ [MySQL 分库分表](/article/db/sharding)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。下一篇候选告诉我。
