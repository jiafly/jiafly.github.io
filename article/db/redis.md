# Redis 核心（速记）

> Redis 面试三大支柱：**数据结构、持久化、高可用**，加上无处不在的"缓存三连"——穿透/击穿/雪崩。本文一篇打包。
>
> 单线程内核、Lua 原子性、AOF rewrite OOM、主从切换丢数据窗口、Cluster 重 slot 风暴等 9 个生产事故 → **[Redis 内核与一致性（深文）](/article/db/redis-internals)**

---

## 1. 五种基础数据结构

| 类型 | 底层 | 典型用途 |
| --- | --- | --- |
| **String** | SDS（动态字符串） | 计数、Token、对象序列化 |
| **List** | quicklist（链表 + ziplist） | 消息队列、最近访问 |
| **Hash** | hashtable / listpack | 对象字段存储 |
| **Set** | hashtable / intset | 标签、共同好友 |
| **Sorted Set / ZSet** | skiplist + hashtable | 排行榜、延迟队列 |

### 1.1 进阶结构
- **Bitmap**：位图，签到、UV；
- **HyperLogLog**：基数估计，12KB 估计 2^64 个元素，误差 0.81%；
- **GEO**：地理坐标，附近的人；
- **Stream**：消息流（5.0+），支持消费组；
- **BloomFilter**（RedisBloom 模块）：解决缓存穿透。

### 1.2 ZSet 为什么用跳表？
- 增删查均 O(log n)，与红黑树同；
- 区间查询天然有序；
- **实现简单**（链表 + 多级索引），并发场景下更易做无锁；
- Antirez 原话："skiplist 内存可控、调试简单"。

---

## 2. 持久化

### 2.1 RDB
- **快照**：fork 子进程把当前数据集 dump 成 .rdb；
- 优点：体积小、恢复快；
- 缺点：可能丢数据（按时间点快照）。

```bash
# 配置
save 900 1     # 900 秒至少 1 次写则保存
save 300 10
save 60 10000
```

### 2.2 AOF（Append Only File）
- **追加日志**：每条写命令写到 .aof；
- 三种 fsync 策略：
  - `always`：每次都刷盘，最安全也最慢；
  - **`everysec`**（默认）：每秒刷盘，最多丢 1 秒；
  - `no`：交给 OS。

#### AOF 重写
- 文件膨胀后通过 `BGREWRITEAOF` 压缩；
- fork 子进程，用当前数据集生成最小命令序列。

### 2.3 混合持久化（4.0+）
- AOF 文件 = RDB 头部 + 增量 AOF；
- 兼顾恢复速度（RDB）和数据安全（AOF）；
- `aof-use-rdb-preamble yes`（默认开）。

---

## 3. 过期策略与淘汰策略

### 3.1 过期 key 的处理
- **惰性删除**：访问时才检查；
- **定期删除**：每 100ms 抽 20 个带 TTL 的 key 检查；
- 两者结合，仍可能堆积 → 内存淘汰。

### 3.2 八种淘汰策略

```
volatile-*：仅在带 TTL 的 key 中淘汰
allkeys-*： 在所有 key 中淘汰

* = lru / lfu / random / ttl(仅 volatile-) / noeviction
```

| 策略 | 含义 |
| --- | --- |
| `noeviction`（默认） | 写直接报错 |
| `allkeys-lru` | 最近最少用（**最常用**） |
| `allkeys-lfu` | 4.0+，最不常用 |
| `volatile-lru` | 带 TTL 的 LRU |
| `volatile-ttl` | 优先淘汰快过期的 |

> 缓存场景**默认改成 `allkeys-lru`**。

### 3.3 LRU 实现
Redis 不维护严格 LRU 链表，而是采样：每个 key 记录最近访问时间，淘汰时从池子里随机抽 N 个，挑最旧的。

---

## 4. 缓存三连（必背）

### 4.1 缓存穿透
**问题**：查询数据库都不存在的数据，每次都打到 DB。

**解法**：
- 空值缓存（短 TTL，1~5 分钟）；
- **布隆过滤器**（前置）：判断 key 是否可能存在；
- 接口校验（参数合法性）。

### 4.2 缓存击穿
**问题**：某个**热点 key 过期瞬间**，并发请求全打到 DB。

**解法**：
- 互斥锁：缓存失效时只允许一个线程查 DB，其他等待；
- 热点 key 永不过期 + 异步刷新；
- **逻辑过期**：value 中存逻辑过期时间，过期后异步更新。

```java
// 双重检查 + 互斥锁伪码
String v = redis.get(k);
if (v == null) {
    if (lock.tryLock()) {
        try {
            v = redis.get(k);
            if (v == null) {
                v = db.load(k);
                redis.setex(k, 10*60, v);
            }
        } finally { lock.unlock(); }
    } else {
        Thread.sleep(50); return get(k);    // 自旋等待
    }
}
```

### 4.3 缓存雪崩
**问题**：大量 key **同时过期** 或 Redis 宕机，DB 被打挂。

**解法**：
- 过期时间加随机值（避免同时失效）；
- Redis 集群 + 多级缓存（本地 Caffeine + Redis）；
- 限流降级（Sentinel / Hystrix）；
- 提前预热。

---

## 5. 缓存与数据库一致性

### 5.1 主流方案

| 方案 | 优缺点 |
| --- | --- |
| **先更新 DB，再删缓存**（推荐） | 简单、最终一致，可能小窗口不一致 |
| 先删缓存，再更新 DB | 并发下旧值可能被回写到缓存 |
| 双删 | 删除 → 更新 → 延迟 N 秒再删一次 |
| **订阅 binlog**（Canal） | 强一致、解耦，但架构复杂 |

### 5.2 为什么是"删"不是"改"？
- 删简单且幂等；
- 改容易出现"中间状态"和"覆盖问题"；
- 冷数据可能根本不需要更新缓存。

### 5.3 极端场景的最终一致兜底
- 缓存设较短 TTL；
- MQ 异步补偿；
- 监控不一致率（采样比对）。

---

## 6. 高可用架构

### 6.1 主从复制
- **全量**：第一次 / RDB；
- **增量**：repl_backlog 环形缓冲。

### 6.2 哨兵（Sentinel）
- 多个 Sentinel 通过 Raft 选主；
- 监控、通知、自动故障转移；
- 客户端连接哨兵拿主地址。

### 6.3 Cluster 集群（生产首选）
- **16384 个槽**，按 `CRC16(key) mod 16384` 分配；
- 客户端直连任意节点，槽不在本机会返回 `MOVED` 重定向；
- 每个主有从，主挂自动切；
- 不支持跨槽事务，跨槽用 `{tag}` 强制同槽。

---

## 7. 分布式锁

### 7.1 单节点 SETNX
```bash
SET key value NX PX 30000
```
- `NX`：不存在才设；
- `PX`：毫秒级 TTL；
- 释放：用 Lua 校验 value 后 DEL，避免误删别人的锁。

```lua
if redis.call("get", KEYS[1]) == ARGV[1] then
  return redis.call("del", KEYS[1])
else
  return 0
end
```

### 7.2 Redisson
- 基于 SETNX + Lua + 看门狗（自动续期）；
- 支持可重入、读写锁、信号量、公平锁；
- **生产推荐**。

### 7.3 RedLock
- Redis 之父 Antirez 提出，跨多个独立 Redis 节点加锁；
- 必须半数+1 节点加锁成功才算成功；
- 复杂、有争议（Martin Kleppmann 撰文质疑），常规场景不必上。

---

## 8. 性能优化与排查

### 8.1 慢日志
```bash
CONFIG SET slowlog-log-slower-than 10000   # 微秒
SLOWLOG GET 20
```

### 8.2 大 key 问题
- 单 key value > 10KB 或 List/Hash 元素数 > 5000；
- 影响：网络阻塞、迁移慢、扩容失败；
- 排查：`redis-cli --bigkeys`、`MEMORY USAGE key`；
- 解决：**拆分成多个 key**、用 `HSCAN/SSCAN` 分批操作。

### 8.3 热 key
- 单 key QPS 占整体 > 10%；
- 解决：**本地缓存** + Redis、key 多副本（key:1 ~ key:N）。

### 8.4 危险命令
| 命令 | 危害 |
| --- | --- |
| `KEYS *` | 阻塞主线程，**严禁线上使用** |
| `FLUSHALL/DB` | 清库 |
| `SAVE` | 同步 RDB，阻塞 |
| `DEBUG SLEEP` | 阻塞 |

> 用 `SCAN` 替代 `KEYS`，`UNLINK` 替代 `DEL` 大 key。

---

## 9. Redis 6 / 7 关键新特性

- 6.0 多线程（仅网络 IO，命令仍单线程）；
- 6.0 ACL；
- 6.2 客户端缓存；
- 7.0 Function（替代 Lua 脚本）；
- 7.0 Sharded Pub/Sub；
- 7.2 Active-Active CRDT。

---

## 10. 高频面试题速记

1. **Redis 为什么快？** 内存 + 单线程（避免锁）+ IO 多路复用 + 高效数据结构。
2. **ZSet 底层？为什么不用红黑树？** skiplist + hashtable；范围查询友好、并发实现简单。
3. **缓存穿透/击穿/雪崩区别与方案？** 见第 4 节。
4. **缓存一致性？** 先更新 DB 再删缓存 + 短 TTL 兜底；强一致用 binlog。
5. **持久化机制？** RDB 快照 + AOF 日志 + 混合持久化。
6. **Cluster 槽怎么分？** CRC16(key) mod 16384。
7. **过期 key 处理？** 惰性 + 定期 + 内存淘汰。
8. **分布式锁？** SETNX + Lua 解锁；推荐 Redisson 看门狗。
9. **大 key/热 key 怎么治？** 拆分 + 本地缓存 + 多副本。
10. **Redis 6.0 多线程？** 仅网络 IO 多线程，命令执行仍单线程。

---

> 上一篇：[MySQL 核心](/article/db/mysql) ｜ 下一站：[AI 工程](/article/ai/llm-app)
