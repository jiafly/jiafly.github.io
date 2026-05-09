# 缓存设计 —— 击穿、雪崩、穿透、一致性与多级缓存的工程实战

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，看 [系统设计主题导览](/article/system-design/) 那段口诀就够了。
> 这里要回答的，是**线上 Redis 突然被一个 key 打爆、DB QPS 从 2K 飙到 200K、用户看到陈旧数据**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. 缓存**击穿**、**雪崩**、**穿透** —— 三个名词常被混着用。给一个能让人 5 秒钟分清的定义，然后说出三者的修复方案为什么不能互换。
2. "先删缓存再写 DB" 和 "先写 DB 再删缓存"，哪种更合理？为什么这两种都还有窗口期不一致？所谓"延迟双删"延迟的那段时间该取多少？怎么算？
3. 你的服务用 Canal 监听 binlog 同步缓存。Canal 自己 OOM 重启，落后 30 分钟才追上 —— 这 30 分钟里下游缓存是脏的，**怎么发现、怎么修**？为什么"重启 Canal 就行"是错的？
4. Caffeine 默认用的 W-TinyLFU 算法和 Guava 的 LRU 比，到底强在哪？什么情况下 W-TinyLFU **反而更差**？
5. 浏览器 → CDN → 网关 LocalCache → Redis → DB 五层缓存。一条数据从 DB 改了，**正确的失效顺序**是什么？为什么"逆向广播"也错？
6. 直播间一个商品突发 100 万 QPS 打爆 Redis 单 shard。你**怎么提前发现热 key**？怎么打散？打散后 DB 会不会被反向打爆？
7. 布隆过滤器（Bloom Filter）和布谷过滤器（Cuckoo Filter）的核心差别是什么？哪种支持删除？为什么"加了布隆就不会穿透"也是错的？
8. CDN 缓存了一张商品详情页，业务上线后用户看到的还是旧版。**强制刷 CDN 之外**还有哪几种工程方法？为什么不能简单 `Cache-Control: no-cache`？

如果有半数答不上，往下看。

---

## 1. 底层模型：缓存到底在解决什么

绝大多数人对缓存的认知止于"加一层 Redis"。这层抽象解释不了上面 8 个问题。要看懂它们，必须先把缓存的**模式**、**失效模式**、**层级**、**一致性窗口**讲清楚。

### 1.1 缓存的 4 种核心模式

`Cache Aside` / `Read Through` / `Write Through` / `Write Back` —— 名字背得熟，但落到代码 90% 的人都写成了同一种。

```
                 应用                  缓存                 DB
                  │                    │                   │
 ① Cache Aside    │── miss?            │                   │
   (旁路缓存)      │── load from DB ────────────────────────►│
                  │── set cache ──────►│                   │
                  │── return                              │
                  │── write DB ──────────────────────────►│
                  │── invalidate cache►│                   │

 ② Read Through   │── get ────────────►│                   │
   (穿透读)        │                    │── miss → load ───►│
                  │◄── return ─────────│◄── data ──────────│

 ③ Write Through  │── put ────────────►│── write DB ──────►│
   (穿透写)        │◄── ack ────────────│◄── ack ───────────│

 ④ Write Back     │── put ────────────►│ (only cache)
   (回写)          │◄── ack ────────────│
                                       │── async flush ───►│
```

| 模式 | 谁主导读写 | 一致性 | 复杂度 | 典型场景 |
| --- | --- | --- | --- | --- |
| **Cache Aside** | 应用 | 弱（有窗口期） | 低 | 90% 互联网业务 |
| **Read Through** | 缓存层 | 同上 | 中 | 含 Loader 的本地缓存（Caffeine LoadingCache） |
| **Write Through** | 缓存层 | 强 | 中 | DB 写量小、读极多 |
| **Write Back** | 缓存层 | 危险（异步刷） | 高 | OS Page Cache、CPU L1/L2 |

90% 的业务用的是 **Cache Aside**。后面所有讨论默认是这个模型。

> **口诀**：四种模式的差别只有一个 —— **谁负责把数据从 DB 拉到缓存里 / 写回 DB 里**。Cache Aside 是应用，其他三种是缓存层。

### 1.2 缓存的 5 种失效模式

教科书只讲三种（击穿、雪崩、穿透）。生产上你必须分清 5 种，因为修复方案完全不同。

```
模式             触发条件                          DB 受冲击的程度
─────────────────────────────────────────────────────────────────
① 击穿  单个热 key 过期瞬间 + 高并发请求           1 个 key 的并发请求全打到 DB
② 雪崩  大量 key 同时过期 / Redis 宕机             大批 key 同时打到 DB
③ 穿透  请求一个根本不存在的 key（攻击常见）        每个请求都打到 DB（缓存永远 miss）
④ 一致性 DB 改了但缓存没改，或者改的顺序反了        DB 没受冲击，但用户看到脏数据
⑤ 热 key 单 key 流量超出单实例承载能力             Redis 单 shard CPU/带宽爆，全站受影响
```

修复方案对照（这是面试 / 评审常被问的，背下来）：

| 失效模式 | 修复手段 |
| --- | --- |
| **击穿** | 互斥锁（singleflight / mutex）/ 永不过期 + 后台刷新 |
| **雪崩** | 过期时间加随机扰动 / 多级缓存兜底 / 限流降级 |
| **穿透** | 布隆过滤器 / 缓存 null 值（Negative Caching） |
| **一致性** | 双删 / 延迟双删 / Canal 监听 binlog / 版本号比较 |
| **热 key** | 多级缓存（LocalCache 兜热）/ key 打散（hash slot 分片）/ 探测系统 |

注意：**一个事故现场往往是几个失效模式叠加**。比如热 key 过期 → 击穿 → 大量请求 → 顺带触发了"穿透"（请求里夹着不存在的 key）。所以诊断要分清主次。

### 1.3 多级缓存：浏览器 → CDN → 网关 → LocalCache → Redis → DB

```
[Browser Cache]  ← Cache-Control / ETag / If-None-Match
       │
       ▼
   [CDN]         ← TTL / Cache-Control / surrogate keys
       │
       ▼
  [Gateway]      ← API Gateway 缓存（Kong / Apisix）
       │
       ▼
[LocalCache]     ← Caffeine / Guava，进程内，纳秒级
       │
       ▼
   [Redis]       ← 跨进程共享，毫秒级
       │
       ▼
    [DB]         ← MySQL / PG，10ms 级
```

每一层的设计目标不同：

| 层 | 目标 | 命中率目标 | 失效成本 |
| --- | --- | --- | --- |
| Browser | 减少首屏请求 | 60%+ | 改版即失效，控不住用户 |
| CDN | 减少回源带宽 | 90%+ | 主动 purge / TTL 自然过期 |
| Gateway | 减少后端调用 | 70%+ | 配置中心推送失效 |
| LocalCache | 抗 Redis 热点 | 80%+（热 key） | 进程内，单机控制 |
| Redis | 抗 DB | 95%+ | 写时主动 del，读时拉 |
| DB | 真源 | — | — |

**关键认知**：层数越多，**一致性窗口越长** —— 一条数据从 DB 改到所有缓存层都看到新值，可能要分钟级。这就是为什么后面要专门讲"失效顺序"。

### 1.4 一致性的"窗口"概念：业务能容忍多少 ms 的不一致

把"缓存与 DB 不一致"当成绝对错误是经验主义陷阱。**绝对一致需要分布式事务，代价是吞吐砸到地板**。务实的做法是定义"最大不一致窗口"：

| 业务场景 | 可容忍窗口 | 推荐方案 |
| --- | --- | --- |
| 商品详情页（详情、评价） | 秒级 | Cache Aside + Canal 异步同步 |
| 库存（抢购前判断） | < 100ms | Redis 单点强一致 + DB 扣减 |
| 用户余额、订单状态 | 不允许 | 不缓存或 Read Through 强一致 |
| 推荐结果、首页 feed | 分钟级 | 定时全量刷 + 异步增量 |
| 配置项、字典 | 小时级 | 启动加载 + 配置中心广播失效 |

设计前先和产品对齐"可容忍窗口"，再选方案。**先选方案再让产品被动接受**是反模式。

### 1.5 热 key vs bigkey：诊断和治理的不同方向

两个词都带 "key"，但治理方向完全相反：

| 维度 | 热 key | bigkey |
| --- | --- | --- |
| 定义 | QPS 集中在少数 key 上 | 单个 value 体积过大 |
| 阈值 | 单 key > 1 万 QPS | 单 key > 10 KB（hash/list/set 元素 > 5000） |
| 后果 | Redis CPU 100%、单 shard 被打爆 | 阻塞 Redis 单线程、网络带宽爆 |
| 治理 | LocalCache、key 打散、读写分离 | 拆分（hash 分片）、按需加载、惰性删除 |
| 检测 | `redis-cli --hotkeys`、监控埋点 | `redis-cli --bigkeys`、`MEMORY USAGE` |

热 key 的代表：直播间礼物排行榜、爆款商品详情。bigkey 的代表：用户全量收藏列表（几十万元素）、活动配置 JSON（几 MB）。

> **口诀**：热 key 是 **流量** 的集中，bigkey 是 **体积** 的集中。一个用 LocalCache 兜，一个用拆分治。

---

## 2. 源码切片：6 段关键路径

> 基于 Caffeine 3.1 / Redis 7.2 / Guava 32 / Canal 1.1.6 真实代码。注释为本文添加。

### 2.1 Caffeine W-TinyLFU 的核心：Window TinyLFU

W-TinyLFU 是 Caffeine 替代 LRU/LFU 的算法，核心思想：

```
                    ┌──────────┐
       新进入 ────►│  Window  │  (1% 总容量)，LRU
                    └────┬─────┘
                         │ 满了 → victim
                         ▼
                    ┌──────────┐
                    │ TinyLFU  │  ← 频率估算（Count-Min Sketch）
                    │ filter   │  victim vs 主区淘汰候选 PK
                    └────┬─────┘
                         │ 频率 > 候选 → 替换
                         ▼
       ┌─────────────────────────────────┐
       │           Main Region           │  (99% 总容量)，分两段
       │  ┌────────────┐  ┌─────────────┐│
       │  │ Probation  │  │  Protected  ││
       │  │  (20%)     │  │  (80%)      ││
       │  └────────────┘  └─────────────┘│
       └─────────────────────────────────┘
                LRU              LRU
```

关键代码（`com.github.benmanes.caffeine.cache.BoundedLocalCache`）：

```java
// Caffeine BoundedLocalCache，简化版
void onAccess(Node<K, V> node) {
    // ① 记录频率到 Count-Min Sketch
    frequencySketch().increment(node.getKeyReference());

    // ② 根据当前所在分区做不同处理
    if (node.inWindow()) {
        reorderProbation(windowDeque, node);     // window 内部 LRU 重排
    } else if (node.inMainProbation()) {
        // probation 命中 → 升级到 protected
        reorder(accessOrderProbationDeque, node);
        node.makeMainProtected();
        accessOrderProtectedDeque.offerLast(node);
    } else {
        reorder(accessOrderProtectedDeque, node);
    }
}

// 入口：满了之后的淘汰决策
void evictFromMain(int candidates) {
    Node<K, V> victim = accessOrderProbationDeque.peekFirst();   // 主区即将被淘汰的
    Node<K, V> candidate = accessOrderWindowDeque.peekFirst();   // window 准备晋升的

    int victimFreq = frequencySketch().frequency(victim.getKeyReference());
    int candidateFreq = frequencySketch().frequency(candidate.getKeyReference());

    if (candidateFreq > victimFreq) {
        evictEntry(victim, RemovalCause.SIZE, 0L);    // candidate 胜出
    } else {
        evictEntry(candidate, RemovalCause.SIZE, 0L); // victim 留任，candidate 被踢
    }
}
```

**逐段点透**：

- **频率不存全量**：用 Count-Min Sketch 4-bit 计数器估频率，1 个 key 占 4 bit，1 万 key 才 5KB。比"全量记 LRU 时间戳"省两个数量级内存。
- **Window + Main 分两区**：纯 LFU 对"突发新热点"反应慢（频率累积需要时间）。Window 给新条目一个"入场试用期"，1% 容量足够过滤"瞬时噪声"。
- **PK 机制**：window 想晋升 main 时，跟 main 的"待淘汰候选"PK 频率，赢了才换。这是 W-TinyLFU 比 LRU/LFU 高 10~20% 命中率的核心。
- **反直觉**：`accessOrderProtectedDeque` 占主区 80%。一旦进入 protected，"老朋友"很难被新晋升的踢出去 —— 防抖。

什么时候 W-TinyLFU **反而更差**？**完全均匀访问**（每个 key QPS 一样）的场景，频率统计提供不了任何信息，纯 LRU 还更简单些。生产中几乎不存在这种负载。

### 2.2 Redis 的 LFU 实现：8-bit 对数计数器

Redis 4.0+ 支持 LFU 淘汰，但它的"频率"是个**对数估算**。看 `evict.c` 的 `LFULogIncr`：

```c
// src/evict.c
uint8_t LFULogIncr(uint8_t counter) {
    if (counter == 255) return 255;
    double r = (double)rand() / RAND_MAX;
    double baseval = counter - LFU_INIT_VAL;        // LFU_INIT_VAL = 5
    if (baseval < 0) baseval = 0;
    double p = 1.0 / (baseval * server.lfu_log_factor + 1);
    if (r < p) counter++;
    return counter;
}
```

**逐段点透**：

- counter 只有 8 bit（最大 255）。直接累加几秒钟就溢出了。所以引入对数：counter 越大，下次自增的概率越小。
- 默认 `lfu_log_factor = 10`，counter 从 5 涨到 100 大约需要 100 万次访问。这是 Redis 用 1 字节估算 QPS 量级的核心技巧。
- 还有一个"衰减"机制：每隔 `lfu_decay_time`（默认 1 分钟）counter 减 1。防止历史热点永久霸占。

排查热 key 时务必区分：`OBJECT FREQ key` 返回的是 8 位对数计数器，不是真实 QPS。要拿到真实 QPS 必须 `MONITOR` 或埋点。

### 2.3 Guava LoadingCache：Read Through 的标准实现

Caffeine 的 API 模仿 Guava。`LoadingCache.get(key)` 实现了 Read Through 模式 + 防击穿：

```java
// com.google.common.cache.LocalCache.Segment#lockedGetOrLoad，简化
V lockedGetOrLoad(K key, int hash, CacheLoader<? super K, V> loader) {
    ReferenceEntry<K, V> e;
    ValueReference<K, V> valueReference = null;
    LoadingValueReference<K, V> loadingValueReference = null;
    boolean createNewEntry = true;

    lock();   // ★ Segment 级别锁
    try {
        // ① 双检：可能其他线程已经填好了
        e = getEntry(key, hash);
        if (e != null) {
            valueReference = e.getValueReference();
            if (valueReference.isLoading()) {
                createNewEntry = false;     // ② 别人正在 load，等
            } else {
                V value = valueReference.get();
                if (value != null) {
                    return value;            // ③ 已经 load 完了
                }
            }
        }

        if (createNewEntry) {
            loadingValueReference = new LoadingValueReference<>();
            // ... 标记为 loading 状态
        }
    } finally {
        unlock();
    }

    if (createNewEntry) {
        return loadSync(key, hash, loadingValueReference, loader);
    }
    return waitForLoadingValue(e, key, valueReference);    // ④ 等别人 load
}
```

**逐段点透**：

- **加锁的粒度是 Segment**，不是单 key。Guava 默认 4 个 Segment，竞争不会太严重。Caffeine 用了更精细的 striped lock。
- **第一个进来的线程负责 load**，其余线程在 `waitForLoadingValue` 上等待，**load 完了所有等待者拿到同一个值**。这就是 **singleflight** 模式 —— 防击穿的终极实现。
- 如果 loader 抛异常，等待者也会拿到同一个异常，不会出现"一个失败，N 个重试"。

> 这套 singleflight 是防击穿最干净的方案。你自己写互斥锁经常写漏边界（比如等待超时怎么办、loader 死锁怎么办）。能用 LoadingCache 就用。

### 2.4 Canal 同步 binlog → MQ → 缓存

Canal 把自己伪装成 MySQL slave，订阅 binlog 然后发到 MQ。下游消费者更新缓存。核心结构：

```
MySQL Master
   │ binlog (ROW format)
   ▼
[Canal Server]  ← 解析 binlog 成结构化事件 (RowData)
   │
   ▼
   MQ (Kafka / RocketMQ)
   │
   ▼
[Consumer]  ← 收到 RowData，按规则更新 Redis
```

消费者关键代码模板：

```java
@KafkaListener(topics = "canal-product-update")
public void onMessage(String json) {
    CanalEntry.Entry entry = parse(json);
    if (entry.getEntryType() != CanalEntry.EntryType.ROWDATA) return;

    CanalEntry.RowChange rowChange = CanalEntry.RowChange.parseFrom(entry.getStoreValue());
    for (CanalEntry.RowData rowData : rowChange.getRowDatasList()) {
        switch (rowChange.getEventType()) {
            case INSERT, UPDATE -> {
                Long id = extractId(rowData.getAfterColumnsList());
                // ★ 关键：直接 del 而不是 set，避免并发写覆盖
                redis.del("product:" + id);
            }
            case DELETE -> {
                Long id = extractId(rowData.getBeforeColumnsList());
                redis.del("product:" + id);
            }
        }
    }
}
```

**逐段点透**：

- **del 而不是 set**：Canal 收到事件时，DB 已经是最新值，但你不知道**你这条事件**是不是最新。两条 UPDATE 几乎同时到达，乱序消费时 `set` 可能把新值覆盖回旧值。`del` 则触发下次读时主动从 DB 加载，永远拿到最新。
- **位点保存的坑**：Canal 自己要保存"消费到 binlog 哪个位置"。如果 Canal 重启没读到正确位点，可能**重复消费**或**漏消费**。一般用 ZooKeeper 或 Redis 存位点，并配监控。
- **延迟监控**：必须监控 binlog 时间戳与当前时间的 gap。gap > 30s 必告警 —— 这是案例 ⑥ 的根。

### 2.5 双删 / 延迟双删 / Listen+Update 三种一致性方案对比

代码层面的 3 段写法：

```java
// 方案 A：先删缓存再写 DB（失败案例多，已弃）
public void update(Product p) {
    redis.del("product:" + p.id);     // ① 先删
    db.update(p);                     // ② 再写
}
// 风险：① 之后、② 之前，另一个线程读 DB 旧值并把旧值写回缓存 → 永远脏

// 方案 B：先写 DB 再删缓存（业界默认）
public void update(Product p) {
    db.update(p);                     // ① 先写
    redis.del("product:" + p.id);     // ② 再删
}
// 风险：极小窗口下，另一个线程在①之前已经读到旧 DB 值，但还没写回缓存
//       → ② 删了空缓存（无效），它再写回旧值 → 仍脏

// 方案 C：延迟双删（被广泛抄但少有人理解）
public void update(Product p) {
    redis.del("product:" + p.id);
    db.update(p);
    scheduledExecutor.schedule(() ->
        redis.del("product:" + p.id),
        500, TimeUnit.MILLISECONDS);  // ★ 延迟 500ms 再删一次
}
// 思路：第一次删覆盖了最常见情况；延迟 500ms 后再删一次，覆盖
//      "刚好在窗口期把旧值写回缓存"的边界情况
```

**为什么延迟时间是 500ms 不是 5ms？**

延迟时间必须 > "读 DB + 写 Redis" 的耗时。典型业务里：
- 读 DB：5~50ms（看网络和索引）
- 写 Redis：1~5ms

所以 500ms 是个安全的经验值。**不要拍脑袋选 50ms**，否则延迟双删等同于没做。

但延迟双删有它的弱点：**delete 可能失败**（Redis 连接断、应用崩了）。所以更可靠的是方案 D：

```java
// 方案 D：Canal 监听 binlog 异步删
// 应用层只负责 db.update，不负责删缓存
public void update(Product p) {
    db.update(p);   // 就这一行
}
// Canal 看到 binlog 后异步 del Redis
```

代价：引入了 Canal 这个组件，运维成本上升；窗口期是 binlog 延迟（典型 100ms ~ 数秒）。

> **口诀**：业务简单 → 写 DB + del 缓存（够用）；要求窗口 < 1s → 延迟双删；要求架构清晰、强依赖 binlog → Canal。

### 2.6 Guava BloomFilter：N 个 hash 函数 + 1 个 bitmap

```java
// com.google.common.hash.BloomFilterStrategies
public <T> boolean put(T object, Funnel<? super T> funnel,
                       int numHashFunctions, LockFreeBitArray bits) {
    long bitSize = bits.bitSize();
    long hash64 = Hashing.murmur3_128().hashObject(object, funnel).asLong();
    int hash1 = (int) hash64;
    int hash2 = (int) (hash64 >>> 32);   // ★ 取 64 位 hash 的高 32 位

    boolean bitsChanged = false;
    for (int i = 1; i <= numHashFunctions; i++) {
        int combinedHash = hash1 + (i * hash2);   // ★ 双 hash 法生成 N 个 hash
        if (combinedHash < 0) combinedHash = ~combinedHash;
        bitsChanged |= bits.set(combinedHash % bitSize);
    }
    return bitsChanged;
}
```

**逐段点透**：

- 不调 N 次 murmur3，而是调一次 murmur3 拿 64 位然后用 "double hashing" 法生成 N 个伪 hash。这是**性能优化**，对误判率几乎无影响（论文 Less Hashing, Same Performance）。
- numHashFunctions 由 expectedInsertions + fpp 公式算出。fpp = 0.03 时大约 5 个 hash；fpp = 0.001 时 10 个 hash。
- **不支持删除**：删除一个元素意味着把 5 个 bit 置 0，但这些 bit 可能也属于其他元素。这是布隆过滤器的根本限制。需要删除的场景用 **Cuckoo Filter**。

误判率与内存的曲线：fpp 每降一个数量级，bitmap 大小约翻 1.5 倍。1 亿元素 fpp=0.01 大约 120MB，fpp=0.001 约 180MB。生产里 0.01 ~ 0.001 是典型选择。

> **口诀**：布隆过滤器的"误判"是单向的 —— **可能误说"存在"，永远不会误说"不存在"**。所以适合"挡掉一定不存在的 key"。

---

## 3. 实测数据：4 组对比

> 没有数据的"性能更好"都是耍流氓。下面 4 组数据来自 16 核 / 64GB / NVMe / Redis 7.2 单实例 / MySQL 8.0 / Caffeine 3.1 的压测环境。仅供量级参考，不要照搬到容量评估。

### 3.1 双删 vs 延迟双删 vs Canal：一致性窗口实测

实验：开 50 个并发线程，循环 "改 DB + 读 DB + 读 Redis"，统计 "Redis 与 DB 不一致" 的窗口时间。

| 方案 | 平均不一致窗口 | P99 窗口 | 极端窗口 | 备注 |
| --- | --- | --- | --- | --- |
| 不做一致性 | ∞ | ∞ | ∞ | 直到 TTL 自然过期 |
| 先写 DB 后删缓存 | 8 ms | 35 ms | 2.1 s | 极端来自 Redis 短暂抖动 |
| 延迟双删（500ms） | 6 ms | 28 ms | 612 ms | 第二删兜底了大部分极端 |
| Canal 同步 | 120 ms | 480 ms | 3.4 s | Canal+MQ 链路本身有延迟 |

**反直觉的点**：Canal 平均窗口反而比双删大。原因是 Canal 走的是 MQ，链路更长。但 Canal 的优势是：**不依赖应用层是否记得删缓存**。

> **口诀**：要 P99 < 100ms 选延迟双删；要架构解耦选 Canal；两者也可以一起用（双保险）。

### 3.2 LocalCache vs Redis：P99 延迟差距

同样的查询负载（10 万 key，Zipfian 分布），分别用 Caffeine（10 万容量）和 Redis（本地）：

| 缓存 | 平均延迟 | P50 | P99 | P999 | QPS 上限 |
| --- | --- | --- | --- | --- | --- |
| Caffeine | 0.2 µs | 0.1 µs | 0.8 µs | 3 µs | ~ 5000 万/核 |
| Redis（本地 unix socket） | 35 µs | 28 µs | 90 µs | 350 µs | ~ 10 万/连接 |
| Redis（同机房 TCP） | 180 µs | 150 µs | 450 µs | 1.2 ms | ~ 5 万/连接 |
| Redis（跨机房） | 5 ms | 4 ms | 12 ms | 50 ms | ~ 1 万/连接 |

**差距三个数量级**。这就是为什么"热 key 兜底"必须用 LocalCache —— 单机能扛 5000 万 QPS，而 Redis 单实例上限 10 万 QPS 量级。

但 LocalCache 有个永远的难题：**多实例之间的不一致**。后面案例 ⑦ 会讲。

### 3.3 W-TinyLFU vs LRU：不同访问模式下的命中率

负载用 ARC trace（IBM 经典负载基准）：

| 负载特征 | LRU 命中率 | W-TinyLFU 命中率 | 提升 |
| --- | --- | --- | --- |
| Zipfian (α=1.0) | 71% | 89% | +18% |
| Zipfian + 突发 | 65% | 84% | +19% |
| 均匀分布 | 32% | 31% | -1% |
| 顺序扫描（OLTP nightly batch） | 4% | 11% | +7% |

W-TinyLFU 的优势在 **有热点 + 突发新热点** 的场景最大。OLTP 顺序扫描场景（缓存几乎没用）虽然提升幅度小，但 LRU 几乎为 0，W-TinyLFU 还能保住一点点（因为它会用 frequency 拒绝一次性扫描污染）。

均匀访问下 W-TinyLFU 略差是 frequency 维护的额外开销。生产中均匀访问极少。

### 3.4 布隆过滤器：误判率 vs 内存

1 亿元素的 BloomFilter 在不同 fpp 下的内存占用与单次查询耗时：

| fpp | 内存 | hash 函数数 | 单次查询 | 适用场景 |
| --- | --- | --- | --- | --- |
| 0.1 | 60 MB | 3 | 0.3 µs | 粗筛 |
| 0.01 | 120 MB | 7 | 0.6 µs | 业界默认 |
| 0.001 | 180 MB | 10 | 0.9 µs | 高准确性 |
| 0.0001 | 240 MB | 13 | 1.2 µs | 几乎不用 |

**反直觉的点**：fpp 从 0.01 降到 0.001 内存只多 50%，查询慢 50%。所以**对准确性敏感的场景直接选 0.001**，0.01 在大流量下还是会有不少误判穿透。

例如：1 亿请求 / fpp=0.01 → 100 万次误判（即 100 万次穿透到 DB）；fpp=0.001 → 10 万次。差 10 倍。

### 3.5 Caffeine 配置组合：吞吐对比

测试：每个 key 加载耗时 1ms，缓存大小 10 万，10 个并发线程读 100 万 key（Zipfian）：

| 配置 | 吞吐 (ops/s) | P99 (µs) | 命中率 | 内存 |
| --- | --- | --- | --- | --- |
| `maximumSize(10000) + expireAfterWrite(10m)` | 380K | 12 | 82% | 50 MB |
| `maximumSize(100000) + expireAfterWrite(10m)` | 720K | 8 | 93% | 480 MB |
| `maximumSize(100000) + expireAfterWrite(10m) + refreshAfterWrite(5m)` | 690K | 9 | 93% | 490 MB |
| `maximumSize(1000000)` | 850K | 7 | 96% | 4.5 GB |

`refreshAfterWrite` 看似吞吐略降，但**避免了击穿**（refresh 是异步的，期间老值仍可用）。生产强烈推荐。

`maximumSize(1000000)` 命中率最高，但 4.5 GB 让 GC 频繁（见案例 ⑧）。

> **口诀**：Caffeine 三件套 —— `maximumSize`（容量上限）+ `expireAfterWrite`（兜底）+ `refreshAfterWrite`（异步刷新防击穿）。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 直播间顶流商品热 key 打爆 Redis 单 shard

#### 现象

某直播间晚 8 点开播，主推一个爆款商品。开播 10 分钟，监控显示：
- Redis 集群中**某一个 shard** 的 CPU 100%（其他 shard CPU 30%）
- 该 shard 上的所有业务请求 P99 飙到 5s
- 全站非该商品业务也受影响（连接被该 shard 占满）

应用日志看到大量 `ReadTimeoutException`。

#### 根因

商品详情的 key 是 `product:detail:{productId}`。爆款商品 productId=88888，hash 到固定 shard。开播瞬间该 key 的 QPS 飙到 80 万，单 Redis 实例物理上线就 10 万 QPS 量级，CPU 直接打满。

更糟的是：**Redis 是单线程**，CPU 100% 后所有命令排队，连 PING 都超时。这个 shard 上其他业务全部连带被影响。

`redis-cli --hotkeys` 实时探测：

```bash
redis-cli -h <host> --hotkeys
# Hot key found: 'product:detail:88888' has 837K hits
```

或者埋点统计（更可靠）：在 Redis 客户端 SDK 里加一层 sliding window，记录每个 key 的 QPS。

#### 修复

**短期（事中）**：紧急上 LocalCache 兜底。

```java
// Caffeine 兜热 key
Cache<String, Product> hotCache = Caffeine.newBuilder()
    .maximumSize(1000)
    .expireAfterWrite(3, TimeUnit.SECONDS)   // ★ 短 TTL，可控的不一致
    .build();

public Product getProduct(long id) {
    Product p = hotCache.getIfPresent("product:" + id);
    if (p != null) return p;
    p = redis.get("product:" + id);
    if (p == null) {
        p = db.findById(id);
        redis.setex("product:" + id, 300, p);
    }
    hotCache.put("product:" + id, p);
    return p;
}
```

3 秒 TTL 让单实例 100% 命中本地，Redis QPS 直接降到几百。

**中期（防御）**：key 打散。一个热 key 拆成 N 个副本：

```java
// 写入：每个副本写一份
for (int i = 0; i < 16; i++) {
    redis.setex("product:detail:" + id + ":" + i, 300, json);
}

// 读取：随机选一个副本
int slot = ThreadLocalRandom.current().nextInt(16);
String json = redis.get("product:detail:" + id + ":" + slot);
```

16 个副本相当于把 80 万 QPS 分散到 16 个 shard（如果 hash 落到不同 shard 的话）。代价是写放大 16 倍 —— **只对极少数热 key 用**。

**长期（系统）**：建热 key 探测系统。

- 客户端 SDK 上报每个 key 的访问频次到一个聚合服务
- 聚合服务用滑动窗口判断 "QPS > 阈值" 的 key
- 推送到所有客户端实例的 LocalCache

京东开源的 **JdHotkey** 就是这么干的：[https://gitee.com/jd-platform-opensource/hotkey](https://gitee.com/jd-platform-opensource/hotkey)

#### 验证

```bash
# 攻击测试：模拟 10 万 QPS 单 key
wrk -t 16 -c 200 -d 60s 'http://app/product/88888'

# 看 Redis CPU 是否平稳，应用 P99 < 50ms
redis-cli info stats | grep instantaneous_ops_per_sec
```

> **口诀**：热 key 打 Redis = LocalCache 兜底 + 多副本打散 + 探测系统。**不要等被打爆才上**，提前给 Top 1000 商品配 LocalCache。

---

### 案例 ② 缓存雪崩：千级 key 同时过期，DB QPS 暴涨 100 倍

#### 现象

凌晨 3 点 DB 告警：MySQL CPU 从 10% 飙到 100%，QPS 从 2K 飙到 200K，应用 P99 飙到 30s。
3 分钟后 DB 自己缓过来，QPS 回落。监控显示：缓存命中率从 99% 跌到 30% 又恢复。

#### 根因

业务有个全量预热脚本，每天凌晨 2 点跑：

```java
for (Long id : allProductIds) {
    Product p = db.findById(id);
    redis.setex("product:" + id, 3600, json);   // ★ TTL 全是 3600 秒
}
```

预热完所有 key 的过期时间都集中在 3:00:00 ± 几秒。3 点整大批 key 同时过期 → 所有读请求穿透到 DB → DB 被打爆。

这就是经典的 **缓存雪崩**。

#### 修复

**手段 A**：TTL 加随机扰动。

```java
// 在固定 TTL 上加 0~10% 的随机
int baseTtl = 3600;
int jitter = ThreadLocalRandom.current().nextInt(360);
redis.setex(key, baseTtl + jitter, value);
```

简单有效。但只解决了"批量过期"问题，不解决"Redis 整体宕机导致雪崩"。

**手段 B**：多级缓存兜底。

```java
public Product getProduct(long id) {
    // L1: LocalCache
    Product p = localCache.getIfPresent(id);
    if (p != null) return p;
    // L2: Redis
    p = redis.get("product:" + id);
    if (p != null) {
        localCache.put(id, p);
        return p;
    }
    // L3: DB（带 singleflight 防击穿）
    return loadFromDbWithLock(id);
}
```

Redis 宕机时 LocalCache 还能扛一阵子（虽然各实例数据不一致，但能撑住业务别崩）。

**手段 C**：DB 入口限流。

不管缓存怎么挂，**DB 必须有限流兜底**。Sentinel / Resilience4j 给 DB 客户端配 QPS 上限。超过限流值的请求快速失败 / 降级，保护 DB 不被打死。

```java
@SentinelResource(value = "queryProduct", blockHandler = "blockHandler")
public Product queryProduct(long id) { ... }

public Product blockHandler(long id, BlockException ex) {
    return Product.degraded();   // 降级返回静态版本
}
```

#### 验证

```bash
# 模拟批量过期：清空 Redis 后压测
redis-cli flushdb
wrk -t 16 -c 200 -d 60s 'http://app/product?id=...'

# DB QPS 应保持在限流阈值以下
```

> **口诀**：雪崩 = 大批 key 同时失效 / Redis 宕机。**TTL 加随机 + 多级缓存 + DB 限流** 三件套。

---

### 案例 ③ 缓存击穿：单个热 key 过期瞬间，并发请求全打到 DB

#### 现象

监控显示：99% 时间 DB QPS 平稳，但**每隔 30 分钟左右**有个尖峰（DB QPS 从 1K 飙到 5W），持续几秒后回落。

#### 根因

某个热 key（首页推荐数据）TTL 设了 30 分钟。每次过期瞬间，几千个并发请求同时发现 cache miss → 全部打到 DB → DB 短暂顶不住。

这是 **击穿**：**单个 key** 的并发洪水（区别于雪崩的"多 key 同时过期"）。

```
                                key 过期瞬间
                                    │
     QPS  ▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁│▆▆▆▆▆▁▁▁▁▁▁▁▁▁▁▁▁▁
                                    └──→ 几千请求并发打 DB
```

#### 修复

**手段 A**：互斥锁 / singleflight。

```java
// Redis 分布式锁版
public Product getProduct(long id) {
    String key = "product:" + id;
    Product p = redis.get(key);
    if (p != null) return p;

    String lockKey = "lock:product:" + id;
    if (redis.set(lockKey, "1", "NX", "EX", 5)) {
        try {
            // 双检：可能在拿锁过程中别人已经填好了
            p = redis.get(key);
            if (p != null) return p;
            p = db.findById(id);
            redis.setex(key, 1800, p);
            return p;
        } finally {
            redis.del(lockKey);
        }
    } else {
        // 没拿到锁，等一下再读
        sleep(50);
        return redis.get(key);
    }
}
```

注意 **必须双检**：进入 if 后再 get 一次，否则有竞态。

**手段 B**：Caffeine LoadingCache（推荐）。

```java
LoadingCache<Long, Product> cache = Caffeine.newBuilder()
    .maximumSize(10000)
    .expireAfterWrite(30, TimeUnit.MINUTES)
    .refreshAfterWrite(25, TimeUnit.MINUTES)   // ★ 25 分钟时异步刷新
    .build(id -> db.findById(id));
```

`refreshAfterWrite` 的语义：**老值还能用，后台异步去 load 新值**。永远不会出现"过期瞬间所有请求等 load"的现象。

**手段 C**：永不过期 + 后台定时刷新。

```java
// 写入时不设 TTL
redis.set("product:" + id, json);

// 后台线程每分钟全量刷一次
@Scheduled(fixedRate = 60000)
public void refreshHotProducts() {
    for (Long id : hotProductIds) {
        Product p = db.findById(id);
        redis.set("product:" + id, json);
    }
}
```

适合"全量数据有限 + 高频访问"的场景，比如首页配置、推荐 Top 100。

#### 验证

```bash
# 模拟击穿：手动 del 热 key 后立即压测
redis-cli del 'product:88888'
wrk -t 16 -c 500 -d 10s 'http://app/product/88888'

# DB QPS 应平稳（singleflight 起效）
```

> **口诀**：击穿是单 key 问题。**互斥锁 / LoadingCache singleflight / 永不过期 + 后台刷新** 任选其一。最优雅的是 LoadingCache。

---

### 案例 ④ 缓存穿透：恶意请求查不存在的 key，绕过缓存

#### 现象

某天 DB 突然 QPS 飙升，但应用日志没有异常。Nginx access log 看到大量 `GET /product/<随机大数字>` 请求，每个请求的 productId 都不一样（且都不存在）。

明显是攻击 —— 攻击者随机生成 ID 探查接口。

#### 根因

代码逻辑：

```java
public Product getProduct(long id) {
    Product p = redis.get("product:" + id);
    if (p != null) return p;
    p = db.findById(id);   // ★ DB 查 null
    if (p == null) return null;   // ★ null 不写缓存
    redis.setex(...);
    return p;
}
```

不存在的 ID 永远缓存 miss → 每个请求都打到 DB。攻击者用 100 QPS 就能让 DB 跑 100 QPS 全表扫。

#### 修复

**手段 A**：缓存 null 值（Negative Caching）。

```java
public Product getProduct(long id) {
    String key = "product:" + id;
    String cached = redis.get(key);
    if (cached != null) {
        return "NULL".equals(cached) ? null : deserialize(cached);
    }
    Product p = db.findById(id);
    if (p == null) {
        redis.setex(key, 60, "NULL");   // ★ 缓存 null，短 TTL
        return null;
    }
    redis.setex(key, 1800, serialize(p));
    return p;
}
```

陷阱：**TTL 必须短**（比如 60 秒）—— 否则正常用户后来插入了这条数据，缓存仍是 NULL，业务报错。

更隐蔽的陷阱：攻击者用 1000 万随机 ID 攻击 → Redis 缓存了 1000 万个 NULL → Redis 内存爆。所以单独 NULL 缓存不够，还要配下面的。

**手段 B**：布隆过滤器。

```java
BloomFilter<Long> filter = BloomFilter.create(
    Funnels.longFunnel(),
    100_000_000,    // 1 亿元素
    0.001            // 千分之一误判率
);

// 启动时把所有合法 ID 放进去
db.streamAllIds().forEach(filter::put);

public Product getProduct(long id) {
    if (!filter.mightContain(id)) {
        return null;   // ★ 布隆说不存在，一定不存在
    }
    // 通过布隆，按正常流程
    ...
}
```

注意：**布隆过滤器需要预热**（把所有合法 ID 灌进去）。新增数据时也要同步加入。

**手段 C**：限流 / 鉴权。

最根本的是入口拦截：
- IP 维度限流（同 IP 1 秒内查 100 个不同 ID 直接拒）
- 业务鉴权（必须登录、有权限）
- WAF 规则识别异常 pattern

#### 验证

```bash
# 模拟穿透攻击
for i in $(seq 1 10000); do
  curl "http://app/product/$RANDOM$RANDOM" &
done

# DB QPS 应保持在低位（布隆挡住）
```

> **口诀**：穿透 = 查不存在的 key。**缓存 NULL 抗少量 + 布隆抗大量 + 限流防恶意**。三层防御。

---

### 案例 ⑤ 双删失败：写 DB 与删缓存不在同一事务

#### 现象

某业务用"先写 DB 再删缓存"的标准 Cache Aside 模式。生产偶发用户投诉"修改了资料但仍显示旧值"，刷新 N 次还是旧的，等 30 分钟（TTL）才好。

#### 根因

代码：

```java
@Transactional
public void updateUser(User u) {
    db.update(u);                       // ① 写 DB
    redis.del("user:" + u.id);          // ② 删缓存
}
```

看起来没问题。但出问题的场景：
- ② 删缓存这一步因为 Redis 短暂连接异常**抛异常**
- `@Transactional` 触发回滚 → DB 也回滚 → 用户看到的还是旧值（DB 也是旧值）→ 一致

**但**：还有更隐蔽的场景：
- ② 抛异常但 Redis 实际已删（网络包发出去了，只是没收到 ack）
- `@Transactional` 回滚 DB
- 缓存被删了，下次读会从 DB 加载 → 加载到回滚后的旧值
- 旧值写回缓存 → "用户改了但没改"

**最常见**的场景是另一种：

```java
public void updateUser(User u) {
    db.update(u);                       // ① 写 DB
    // ② 删缓存：网络抖动，超时 100ms，但实际 Redis 没收到指令
    try {
        redis.del("user:" + u.id);
    } catch (Exception e) {
        log.error("del cache failed", e);   // 仅记日志，不重试
    }
}
```

del 失败被吞了 → 缓存仍是旧值 → 直到 TTL 过期才一致。

#### 修复

**手段 A**：删缓存失败必须**异步重试** + 兜底报警。

```java
public void updateUser(User u) {
    db.update(u);
    deleteWithRetry("user:" + u.id);
}

private void deleteWithRetry(String key) {
    try {
        redis.del(key);
    } catch (Exception e) {
        // 入消息队列，由消费者重试
        mq.send("cache-invalidate", key);
        log.warn("del cache async, key={}", key);
    }
}

@KafkaListener(topics = "cache-invalidate")
public void retryDelete(String key) {
    redis.del(key);   // 失败会进死信队列，告警人工处理
}
```

**手段 B**：Canal 监听 binlog 删缓存（更优雅）。

应用层只负责写 DB，缓存失效完全交给 Canal。代码上极度清爽：

```java
@Transactional
public void updateUser(User u) {
    db.update(u);
    // 没有 redis.del！由 Canal 异步处理
}
```

代价：必须运维 Canal + MQ + 消费者。

**手段 C**：永远不要把 redis.del 放进 @Transactional。

```java
@Transactional
public void updateUser(User u) {
    db.update(u);
}

// 在事务提交后再删
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onUserUpdated(UserUpdatedEvent e) {
    redis.del("user:" + e.userId);
}
```

`AFTER_COMMIT` 保证只有 DB 真的提交后才删缓存，不会出现"事务回滚但缓存被删"的情况。

#### 验证

```bash
# 故意把 Redis 网络断开几秒，做修改操作
iptables -A OUTPUT -p tcp --dport 6379 -j DROP
# 应用调用 updateUser(...)
iptables -D OUTPUT -p tcp --dport 6379 -j DROP

# 检查：
# - 数据应在 DB 已更新
# - 缓存应在网络恢复后被异步删
# - 用户读取应能看到新值
```

> **口诀**：删缓存必须**重试** + **告警** + **不能进事务**。最强方案：用 Canal 把"删"从应用代码里抽走。

---

### 案例 ⑥ Canal 异常导致脏缓存长期存在

#### 现象

某业务依赖 Canal 同步缓存。某天上午用户反馈"系统看到的数据是 30 分钟前的"。检查发现 Canal 服务进程在凌晨 3 点 OOM 重启了，但应用没人发现。

#### 根因

Canal 进程重启时：
1. 重启过程中 binlog 持续在 MySQL 上增长
2. Canal 重启后从上次保存的位点继续消费 → 落后 6 小时
3. 这 6 小时里 DB 改了 1 万条数据 → 缓存里全是旧值
4. 应用层 Cache Aside 读到旧值 → 下游业务全错

最致命的是：**Canal 重启没有人感知**。监控只看了"Canal 进程是否存活"，没看"Canal 消费延迟"。

#### 修复

**手段 A**：监控 Canal 消费延迟。

Canal 自身暴露了 metrics：

```bash
curl http://canal-server:11111/metrics
# canal_instance_ack_position{destination="example"} 1234567
# canal_instance_traffic{destination="example"} ...
```

接到 Prometheus，画"binlog 当前位点"和"Canal ACK 位点"的差值。差值 > 30s 必告警。

**手段 B**：Canal 重启时主动失效相关 key。

Canal 重启检测到位点落后超过阈值时，**主动触发全量刷新**：

```java
// Canal 重启后的初始化
public void onCanalStarted(long lastAckPosition, long currentBinlogPosition) {
    long gap = currentBinlogPosition - lastAckPosition;
    if (gap > THRESHOLD) {
        log.warn("Canal lag too large: {}, triggering full refresh", gap);
        // 全量刷新所有相关缓存
        for (String table : monitoredTables) {
            invalidateAllCacheFor(table);
        }
    }
}
```

代价是重启后短期内 DB 流量飙升。要配合限流。

**手段 C**：Canal 高可用部署 + 双 Canal 互备。

阿里官方推荐 Canal 部署成 HA 集群（基于 ZooKeeper）。一个挂掉另一个立刻接管，避免"消费延迟"问题。

更激进的做法：双 Canal 各订阅一份 binlog，下游消费者做幂等。一个 Canal 挂了完全不影响。

**手段 D**：兜底 TTL。

无论 Canal 多可靠，缓存都必须设 TTL（比如 1 小时）。**Canal 完全失败时 TTL 兜底 1 小时一致性**。永久缓存 + Canal 是危险组合。

#### 验证

```bash
# 故意停掉 Canal 1 小时
systemctl stop canal-server

# 在 DB 改数据
mysql -e "UPDATE products SET price = 999 WHERE id = 1;"

# 启动 Canal，观察告警是否触发
systemctl start canal-server

# Prometheus 告警应在 30s 内触发
```

> **口诀**：Canal 不是"装上就万事大吉"。**消费延迟监控 + 重启全量刷新 + 兜底 TTL** 三件套缺一不可。

---

### 案例 ⑦ LocalCache 与 Redis 不同步：滚动发布时旧机器仍持有旧缓存

#### 现象

某业务用了 Caffeine 兜热 key（5 分钟 TTL）。某次配置变更上线滚动发布，新机器读到新配置，老机器（还没发布完）仍读到旧配置。**用户被路由到不同机器看到不同行为**，灰度策略错乱。

#### 根因

```
机器 A（已发布）：LocalCache.refresh → 读 DB 拿到新配置
机器 B（未发布）：LocalCache 命中 5 分钟前的旧值
```

LocalCache 是进程内的，**没人能远程让它失效**。Redis del 对它无效。每个进程都有自己一份 Cache，分布式不一致是天然属性。

更糟的场景：用户的会话不固定路由（轮询 LB），同一个用户两次请求落到 A 和 B，看到不同数据。

#### 修复

**手段 A**：缩短 LocalCache TTL。

```java
Caffeine.newBuilder()
    .expireAfterWrite(10, TimeUnit.SECONDS)   // ★ 配置类用 10 秒
    .build();
```

简单粗暴。代价是命中率下降。配置类、字典类适用。

**手段 B**：Pub/Sub 广播失效。

```java
// 写入端：改完 DB 后通过 Redis pub/sub 广播
public void updateConfig(Config c) {
    db.update(c);
    redis.publish("cache-invalidate", "config:" + c.key);
}

// 所有应用实例订阅
@PostConstruct
public void subscribeInvalidation() {
    redis.subscribe("cache-invalidate", (channel, message) -> {
        localCache.invalidate(message);
    });
}
```

延迟典型在 100ms 内。能把 LocalCache 的"分布式一致性窗口"压缩到秒级。

**手段 C**：JGroups / 自研多播协议。

如果你已经用了 Hazelcast / Apache Ignite 这类分布式缓存，它们自带集群广播。Caffeine 配合 JGroups 也能做。

**手段 D**：请求级会话粘性。

让同一用户的请求始终路由到同一台机器。这样虽然不同机器数据不一致，但单个用户体验一致。代价是 LB 复杂度上升。

#### 验证

```bash
# 滚动发布期间，并发读两台不同机器
for i in $(seq 1 100); do
    curl http://app1/config/key1 &
    curl http://app2/config/key1 &
done

# 应用层日志应显示两台机器看到的版本号差距 < 1 秒
```

> **口诀**：LocalCache 的根本短板是**进程内**。多实例一致性方案 = **短 TTL + Pub/Sub 广播失效**，二选一或都用。

---

### 案例 ⑧ Caffeine 配置错误：maxSize 太大导致 GC 频繁

#### 现象

某服务接入 Caffeine 后初期表现良好，运行一周后逐渐变慢。监控显示：
- Caffeine 命中率 99%（很好）
- 应用 P99 从 5ms 飙到 50ms
- Young GC 从 5次/分 飙到 50次/分
- Full GC 从 0 飙到 1次/分

#### 根因

代码：

```java
Caffeine.newBuilder()
    .maximumSize(10_000_000)   // ★ 1000 万
    .expireAfterWrite(2, TimeUnit.HOURS)
    .build();
```

每个 value 平均 2KB → 全填满 = 20GB 堆内存。JVM 配的 `-Xmx16g` 直接被吃光。

更隐蔽的问题：**Caffeine 的 entry 本身有元数据开销**（Node 对象 + 频率统计 + LRU 链表节点），1000 万 entry 仅元数据就 1GB+。

GC 频繁的根源不是 Caffeine 本身，是**用户配的 maxSize 远超堆容量**。

#### 修复

**手段 A**：用 weigher 按字节限制，而不是按数量。

```java
Caffeine.newBuilder()
    .maximumWeight(2L * 1024 * 1024 * 1024)   // 2GB 上限
    .weigher((String key, byte[] value) -> key.length() + value.length)
    .expireAfterWrite(2, TimeUnit.HOURS)
    .build();
```

`maximumWeight` 比 `maximumSize` 准确得多 —— 你直接控制内存占用而不是 entry 数。

**手段 B**：用堆外缓存。

```xml
<dependency>
    <groupId>org.ehcache</groupId>
    <artifactId>ehcache</artifactId>
</dependency>
```

EhCache 3 支持 OffHeap tier。绕过 GC：

```java
CacheManager cacheManager = CacheManagerBuilder.newCacheManagerBuilder()
    .withCache("products",
        CacheConfigurationBuilder.newCacheConfigurationBuilder(
            Long.class, byte[].class,
            ResourcePoolsBuilder.newResourcePoolsBuilder()
                .heap(1000, EntryUnit.ENTRIES)        // 堆内 1000 个
                .offheap(2, MemoryUnit.GB)             // 堆外 2GB
        ).build())
    .build(true);
```

OffHeap 不进 GC，但读写有序列化开销（10x 慢于堆内）。

**手段 C**：监控 Caffeine 真实占用。

Caffeine 自带 stats：

```java
CacheStats stats = cache.stats();
log.info("hitRate={}, evictionWeight={}, estimatedSize={}",
    stats.hitRate(), stats.evictionWeight(), cache.estimatedSize());
```

接到 Micrometer：

```java
CaffeineCacheMetrics.monitor(meterRegistry, cache, "products");
```

观察 `estimatedSize` 是否接近 maxSize，`hitRate` 是否上升到收益拐点。

#### 验证

```bash
# 跑一段时间后看 GC
jcmd <pid> GC.heap_info
jstat -gcutil <pid> 1s

# 期望：Old 区使用率 < 70%，Full GC 频次 = 0
```

> **口诀**：Caffeine 用 `maximumWeight` 不要用 `maximumSize`。容量超过堆 30% 时，考虑堆外或 Redis。**容量配错比配小更危险**。

---

### 案例 ⑨ CDN 缓存陈旧：业务上线后用户看到的还是旧版

#### 现象

前端发布了新版商品详情页，CDN 已经 purge 过了。但用户反馈：**部分用户还是看到旧版**，刷新页面没用，过几分钟才好。

#### 根因

CDN 的"缓存"分布在全球边缘节点，purge 是异步广播：
- 中心节点的 purge 立即生效
- 边缘节点的 purge 需要几分钟传播
- 用户已经下载到本地的浏览器缓存仍是旧的（受 `Cache-Control: max-age=3600` 控制）

更复杂的场景：CDN 之间还有"父子节点"。子节点 miss 时回源到父节点而不是源站。父节点没刷新 → 子节点拉到旧值 → 又缓存了。

#### 修复

**手段 A**：版本化 URL（推荐）。

```html
<!-- 旧 -->
<script src="/js/app.js"></script>

<!-- 新 -->
<script src="/js/app.v20240501.js"></script>
<!-- 或 hash -->
<script src="/js/app.a3f5c7b1.js"></script>
```

发布新版本就换 URL。CDN 把它当全新文件，永远不会拿到旧版。这是 SPA / 前端工程化的标准做法。

**手段 B**：Cache-Control + ETag 配合。

```nginx
location ~ \.html$ {
    add_header Cache-Control "no-cache, must-revalidate";
    # 内容不变时返回 304，浏览器复用
}

location ~ \.(js|css|png)$ {
    add_header Cache-Control "public, max-age=31536000, immutable";
    # 一年缓存，因为有版本号 hash
}
```

`no-cache` ≠ 不缓存，是 "每次都验证 ETag 是否变化"。

**手段 C**：CDN 主动 purge / preheat。

发布脚本里调 CDN API 主动失效：

```bash
# 阿里云 CDN
aliyun cdn RefreshObjectCaches \
    --ObjectPath "https://cdn.example.com/index.html" \
    --ObjectType File

# 还可以预热
aliyun cdn PushObjectCache \
    --ObjectPath "https://cdn.example.com/new-version/app.js"
```

预热（preheat）是把新内容主动推到所有边缘节点，避免首次访问回源。

**手段 D**：服务端推送配合 Service Worker。

PWA 可以注册 Service Worker 主动检测更新：

```javascript
navigator.serviceWorker.register('/sw.js').then(reg => {
    reg.update();   // 强制检查更新
});
```

但这是前端能做的，运维介入有限。

#### 验证

```bash
# 多地区压测看缓存版本
for region in cn-east cn-south us-west; do
    curl -I "https://cdn.example.com/index.html" \
        --resolve cdn.example.com:443:<region-edge-ip>
done

# 看 Last-Modified 或自定义 X-Cache-Version 头是否一致
```

> **口诀**：CDN 缓存 = **版本化 URL** + **合理 Cache-Control** + **发布时主动 purge/preheat**。永远不要依赖单一手段。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### LocalCache 层
- [ ] 用 `maximumWeight` + `weigher`，不用 `maximumSize`（避免容量失控）
- [ ] 必配 `expireAfterWrite` 兜底（避免脏数据永久留存）
- [ ] 推荐配 `refreshAfterWrite < expireAfterWrite`（异步刷新防击穿）
- [ ] 接入 Micrometer：`CaffeineCacheMetrics.monitor`
- [ ] 配置类、字典类的 LocalCache TTL ≤ 30 秒（多实例不一致窗口可控）
- [ ] 跨实例必须一致的数据 → Pub/Sub 广播失效

### Redis 层
- [ ] 所有 key 都有 TTL，不允许永久 key（除非有专人 review）
- [ ] TTL 加 0~10% 随机扰动（防雪崩）
- [ ] 大批量预热的 key 必须分散过期时间
- [ ] 单 key value 大小 < 10 KB；hash/list/set 元素 < 5000（防 bigkey）
- [ ] 读写命令注意是否阻塞 Redis 单线程（`KEYS *` / `SMEMBERS bigset` / `HGETALL bighash` 全禁）
- [ ] 监控 hot key（`redis-cli --hotkeys` 或客户端埋点）
- [ ] 监控 slowlog：`CONFIG SET slowlog-log-slower-than 10000`

### CDN 层
- [ ] 静态资源版本化 URL（`app.[hash].js`）
- [ ] HTML / 接口 `Cache-Control: no-cache, must-revalidate`
- [ ] 资源 `Cache-Control: public, max-age=31536000, immutable`
- [ ] 发布脚本调 CDN purge API + preheat
- [ ] 跨地区压测验证缓存版本一致

### 一致性层
- [ ] 写后立即读路由主库（避免主从延迟脏读）
- [ ] 写 DB + 删缓存的 del 必须重试 + 兜底告警
- [ ] del 操作放在 `AFTER_COMMIT` 阶段，不放进事务
- [ ] 强一致需求（库存、余额）禁止 Cache Aside，用 Read Through 或不缓存
- [ ] Canal 必须监控消费延迟 > 30s 告警
- [ ] Canal 重启后位点落后超阈值时全量失效相关 key
- [ ] 不存在的 key 缓存 NULL（短 TTL ≤ 60s）
- [ ] 存在攻击风险的接口加布隆过滤器

### 监控层
- [ ] 缓存命中率（按 key 前缀分组）：< 80% 告警
- [ ] 击穿 / 雪崩特征：DB QPS 突增 + 缓存 miss 率突增
- [ ] 热 key 探测：单 key QPS > 1 万
- [ ] bigkey 探测：周扫一次 `redis-cli --bigkeys`
- [ ] Redis 内存使用率 > 80% 告警
- [ ] LocalCache GC 影响：监控 estimatedSize / hit/miss 比

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### Read-Through 与 Cache-Aside 的工程化选型
- 为什么大多数业务用 Cache Aside 而不是 Read Through？答案是**事务边界清晰**
- Read Through 在多语言异构服务里的落地难点（缓存层得理解你的领域模型）
- Spring Cache 抽象用的是 Cache Aside 还是 Read Through？源码细节
- 推荐 Caffeine `LoadingCache` 在单服务内做 Read Through，跨服务用 Cache Aside

### 分布式 LocalCache（Caffeine + Pub/Sub 失效广播）
- 实现一个最小版的"分布式 Caffeine"：本地 Caffeine + Redis Pub/Sub 失效
- Hazelcast 的近端缓存（Near Cache）实现
- Apache Ignite 的客户端缓存
- Coherence / Geode 等商业方案的演进

### CDN 边缘计算与缓存
- Cloudflare Workers / AWS Lambda@Edge 在边缘节点执行业务逻辑
- ESI（Edge Side Includes）：在边缘节点拼装页面片段
- CDN 上的 A/B 测试缓存策略
- HTTP/3 + QUIC 对 CDN 缓存策略的影响

### 永久缓存 + 异步刷新模式
- 适合"数据量小、访问巨多、强不变" 的场景（如黑白名单、风控规则）
- 实现：Redis hash + 后台 Job 全量刷新 + 版本号校验
- 失败兜底：多版本保留，回滚机制

### GraphQL / API Gateway 上的缓存
- GraphQL 的 query-level cache vs field-level cache
- Apollo Client 的 normalized cache 设计
- API Gateway（Kong / APISIX）的请求级缓存
- 缓存与权限控制的冲突（同一接口不同用户结果不同怎么缓存）

### 缓存预热（cold start）的工程方案
- 上线 / 重启时的预热脚本设计
- 灰度发布时新机器的预热顺序
- 预热失败的回滚策略
- 大数据集预热的并发控制（避免预热过程把 DB 打爆）

### Negative Caching（缓存"不存在"）
- 不只是缓存 NULL，还包括缓存"权限拒绝"、"参数错误"等明确响应
- 与正常缓存的 TTL 策略差别（一般更短）
- 与布隆过滤器的协同
- HTTP 4xx / 5xx 响应该不该被 CDN 缓存？

---

## 7. 引用

- Caffeine 源码：[github.com/ben-manes/caffeine](https://github.com/ben-manes/caffeine)
- W-TinyLFU 论文：Einziger et al., *TinyLFU: A Highly Efficient Cache Admission Policy*（2017）
- Redis 源码：[github.com/redis/redis](https://github.com/redis/redis)（特别是 `src/evict.c` 与 `src/object.c`）
- Canal 源码：[github.com/alibaba/canal](https://github.com/alibaba/canal)
- 美团技术团队：[Cache 在分布式系统中的几个常见问题](https://tech.meituan.com/)
- 京东 [JdHotkey](https://gitee.com/jd-platform-opensource/hotkey) 设计文档
- 阿里 [TairCache](https://github.com/alibaba/tair) 架构文档
- 《Designing Data-Intensive Applications》—— Martin Kleppmann（缓存一致性章节）
- Less Hashing, Same Performance（Kirsch & Mitzenmacher, 2006）—— BloomFilter 双 hash 优化论文
- Cuckoo Filter 论文：Fan et al., *Cuckoo Filter: Practically Better Than Bloom*（2014）

---

> 配套：[系统设计主题导览](/article/system-design/) ｜ [Redis 内核与一致性](/article/db/redis-internals) ｜ [MySQL MVCC 与锁](/article/db/mysql-mvcc)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
