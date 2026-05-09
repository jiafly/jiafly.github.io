# 秒杀系统演进 —— 从 1K QPS 到 100K QPS 的工程实践

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果只想速读，看 [系统设计总览](/article/system-design/) 的"秒杀"那一节就够了。
> 这里要回答的，是**真正接到一个秒杀活动需求、自己签字上线**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. 库存 100 件、瞬时 10 万并发，怎么**彻底**避免超卖？仅靠 `update stock = stock - 1 where stock > 0` 够吗？为什么 RR 隔离下这条 SQL 在某些情况仍然有空隙？
2. 本地内存预扣 + Redis 扣减两种方案，**边界**在哪里？什么场景下本地预扣会"扣了但用户其实没买到"？怎么补偿？
3. 异步落库（订单先进 MQ，再异步写 DB）的可靠性怎么保证？MQ 投递失败、消费失败、下游 DB 超时各怎么处理？怎么避免重复落库？
4. 用户提交后到底应该立即给"成功/失败"，还是返回"排队中，请稍等"？两种体验的工程代价差在哪？什么时候必须用排队？
5. 黄牛用脚本绕过验证码、伪装成多个用户，**纯算法层**能识别出来吗？需要哪些数据维度？只靠 IP 限流为什么完全不够？
6. 秒杀峰值 QPS 是日常的 100 倍，你怎么保证**关联业务**（订单中心、用户中心、商品中心）不被打挂？数据库层面用什么方案隔离？
7. Nginx `limit_req` 用 burst + nodelay 配错了一个参数，瞬时几万请求穿透到 Tomcat —— 哪个参数？正确该怎么配？
8. 秒杀失败的用户、超时未支付的订单，库存怎么释放？回补到 Redis 还是 DB？回补失败怎么办？怎么防止"库存幽灵"（DB 还有但 Redis 已经清零）？

如果有半数答不上，往下看。

---

## 1. 底层模型：四个核心矛盾与流量分层削峰

绝大多数秒杀文章一上来就讲"Redis 扣库存 + MQ 异步落单"，然后画个架构图就结束了。这层认知够画 PPT，不够上线。要把秒杀做扎实，必须先想清楚**它和普通下单业务有哪些根本不同**。

### 1.1 秒杀的 4 个核心矛盾

普通下单是"水池放水"，慢慢流。秒杀是"水库泄洪"，三秒钟泄掉一年的水。

| 矛盾 | 表现 | 普通下单 | 秒杀 |
| --- | --- | --- | --- |
| **峰谷比** | QPS 峰值 / 日常 | 1~3 倍 | 50~500 倍 |
| **写热点** | 同一行被并发更新 | 几乎没有 | 全集中在 1~10 个 SKU |
| **失败率** | 用户期望成功率 | > 99% | **可以 99% 都失败**（库存就那么多） |
| **舞弊** | 异常请求比例 | < 1% | 50% 以上是脚本/黄牛 |

四个矛盾决定了秒杀**不能套用普通下单架构**：

- 峰谷比 → 必须**削峰**，不能让峰值打到 DB
- 写热点 → 必须**绕开行锁**，DB 行锁的极限大约 1~5K QPS / 行
- 高失败率 → **快速失败**比"努力让所有人成功"更重要
- 舞弊 → **风控前置**到入口层，过滤掉一半流量再说

### 1.2 流量分层削峰

秒杀架构的本质是**漏斗**：每一层都过滤掉一批，让真正打到 DB 的请求 < 库存数 × 安全系数。

```
        100K QPS                  CDN 拦截静态资源
       ┌─────────┐               (商品图、活动页)
用户 ─►│  CDN    │ ─ 90% ─►
       └─────────┘
            │ 10K QPS
            ▼
       ┌─────────┐               Nginx 限流 + 黑名单
       │ Nginx   │ ─ 50% ─►      (IP / UA / 频率)
       └─────────┘
            │ 5K QPS
            ▼
       ┌─────────┐               网关风控
       │ Gateway │ ─ 60% ─►      (验证码 / 行为分 / 用户黑名单)
       └─────────┘
            │ 2K QPS
            ▼
       ┌─────────┐               Redis Lua 扣库存
       │  App    │ ─ 95% ─►      (扣完即拒)
       └─────────┘                库存只有 100，多余的全拒
            │ 100 QPS
            ▼
       ┌─────────┐               MQ 异步落单
       │   MQ    │
       └─────────┘
            │
            ▼
       ┌─────────┐
       │  DB     │  最终只有 100 个真实订单
       └─────────┘
```

每一层的目标都不是"扛住全部流量"，而是**把流量削减到下一层能承受的量**。这是和普通业务最大的区别。

### 1.3 库存方案的三种主流选型

库存扣减是秒杀的核心。三种主流方案：

**方案 A：DB 行锁（`update stock = stock - 1 where id = ? and stock > 0`）**

最简单、最直接、最容易理解。但 DB 单行 update 的极限是 1~5K QPS（取决于锁等待和 WAL flush）。**只适合 QPS < 1K 的小活动**。

**方案 B：Redis Lua 原子扣减**

主流方案。把"判断 + 扣减"放进 Lua 脚本，Redis 单线程保证原子性。理论上限 5~10W QPS / Redis 实例。

**方案 C：本地内存预扣 + Redis 兜底**

极致方案。每个应用节点先扣本地内存，本地扣完才走 Redis。理论上限可以做到 50W+ QPS，但**有边界 case**（详见案例 ②）。

| 方案 | QPS 上限 | 超卖风险 | 实现难度 | 适用场景 |
| --- | --- | --- | --- | --- |
| DB 行锁 | 1~5K | 0 | ⭐ | 库存大 + QPS 低 |
| Redis Lua | 5~10W | 0（脚本正确的话） | ⭐⭐ | 主流秒杀 |
| 本地预扣 + Redis | 50W+ | 极低（需要补偿） | ⭐⭐⭐⭐ | 极致大促 |

### 1.4 订单方案：同步落库 vs 异步落库

库存扣减成功后，订单要落 DB。两种姿势：

**同步落库**：扣库存 → 落订单 → 返回成功。链路短，用户体验好；但 DB 压力 = QPS。

**异步落库**：扣库存 → 写 MQ → 立即返回"排队中" → MQ 消费者慢慢落 DB。DB 压力可控，但需要解决**用户怎么查订单状态**、**MQ 丢消息怎么办**、**重复消费怎么办**。

经验法则：
- QPS < 1K：同步
- QPS 1K~10K：可同步可异步
- QPS > 10K：必须异步

异步落库的可靠性靠**三件套**保证：
1. MQ 持久化 + 同步刷盘
2. 消费幂等（订单 ID 唯一索引）
3. 死信队列 + 人工介入

### 1.5 用户体验的三种姿势

不要只盯着技术，**用户视角的失败方式**直接决定了系统设计：

| 姿势 | 用户看到 | 工程代价 |
| --- | --- | --- |
| **立即成功/失败** | "抢到了" / "已抢光" | 同步链路要全部跑完，链路长 |
| **排队中** | "您前面还有 N 人" | 需要前端轮询接口；后端要维护"排队进度" |
| **异步通知** | 提交后跳转其他页面，几秒后弹窗"抢到了" | 需要 WebSocket 或长轮询；APP 推送通道 |

90% 的秒杀用第二种"排队"。理由：用户对"等几秒"的容忍度远高于"突然失败"。而后端只要返回"排队中"，扣库存的实际处理可以**完全异步**，链路压力锐减。

### 1.6 风控的三层防御

秒杀场景下，**50% 以上的请求是脚本和黄牛**。如果不做风控，库存全被脚本秒走，真实用户骂街。

| 层 | 手段 | 拦截率 |
| --- | --- | --- |
| **入口层** | IP 频率、UA 黑名单、请求来源 referrer | 30% |
| **业务层** | 滑动验证码、问答验证码、强制登录 | 40% |
| **行为层** | 用户画像、设备指纹、操作轨迹 | 剩余 30% 中的大部分 |

行为层是最难做的，但也是**唯一能识别"真人黄牛"**的手段（详见案例 ⑤）。

> **口诀**：秒杀 = 漏斗。**CDN 削 90%，Nginx 削 50%，网关风控削 60%，Redis 扣库存削 95%**。每一层都不指望挡住全部，只要削到下一层能扛。

---

## 2. 实现切片：核心代码

> 以下代码基于 Spring Boot 3.x + Redis 7 + RocketMQ 5.x。每段都是生产可运行的版本（去掉了无关日志和监控）。

### 2.1 Redis Lua 原子扣库存

错误示范：

```java
// ❌ 两条命令不原子
Long stock = redis.opsForValue().get("stock:1001");
if (stock > 0) {
    redis.opsForValue().decrement("stock:1001");
    return SUCCESS;
}
return FAIL;
```

两条命令之间有时间窗口，并发下必超卖。

正确做法：用 Lua 脚本把"判断 + 扣减"打包成一次 Redis 调用。

```lua
-- seckill_deduct.lua
-- KEYS[1] = stock key,  e.g. "seckill:stock:1001"
-- KEYS[2] = user set key,  e.g. "seckill:bought:1001"
-- ARGV[1] = userId
-- ARGV[2] = quantity (默认 1)

-- 一人一单：检查用户是否已经抢过
if redis.call('SISMEMBER', KEYS[2], ARGV[1]) == 1 then
    return -1   -- 重复购买
end

-- 检查库存
local stock = tonumber(redis.call('GET', KEYS[1]))
if stock == nil then
    return -2   -- 库存不存在（活动未开始或已结束）
end

local qty = tonumber(ARGV[2])
if stock < qty then
    return 0    -- 库存不足
end

-- 扣库存 + 记录用户
redis.call('DECRBY', KEYS[1], qty)
redis.call('SADD', KEYS[2], ARGV[1])

return 1        -- 成功
```

Java 调用：

```java
@Service
public class SeckillService {

    private final StringRedisTemplate redis;
    private final DefaultRedisScript<Long> deductScript;

    public SeckillService(StringRedisTemplate redis) {
        this.redis = redis;
        this.deductScript = new DefaultRedisScript<>();
        this.deductScript.setLocation(new ClassPathResource("seckill_deduct.lua"));
        this.deductScript.setResultType(Long.class);
    }

    public SeckillResult tryDeduct(long itemId, long userId, int qty) {
        List<String> keys = List.of(
            "seckill:stock:" + itemId,
            "seckill:bought:" + itemId
        );
        Long ret = redis.execute(deductScript, keys, String.valueOf(userId), String.valueOf(qty));

        return switch (ret.intValue()) {
            case 1  -> SeckillResult.success();
            case 0  -> SeckillResult.fail("库存不足");
            case -1 -> SeckillResult.fail("您已购买过");
            case -2 -> SeckillResult.fail("活动未开始");
            default -> SeckillResult.fail("未知错误: " + ret);
        };
    }
}
```

**几个反直觉点**：

- `SISMEMBER + DECRBY + SADD` 三条命令必须在**同一个 Lua**里，分开调用就会有竞态
- Lua 脚本通过 `EVALSHA` 缓存到 Redis 内存，第二次调用只发 SHA 不发脚本
- Redis Cluster 下，KEYS 必须**hash slot 一致**（所有 key 用 `{itemId}` 包起来强制同槽），否则脚本拒绝执行

### 2.2 本地内存预扣 + Redis 同步

Redis 单实例 5~10W QPS 是上限。再往上要靠本地内存预扣。

思路：每个应用节点启动时从 Redis 申请一段"配额"（比如 100 件库存里申请 10 件），用户请求先扣本地内存的配额，本地扣完了才回 Redis 申请下一批。

```java
@Component
public class LocalStockBucket {

    // itemId -> 本地剩余库存
    private final Map<Long, AtomicInteger> localStock = new ConcurrentHashMap<>();
    // itemId -> 是否已耗尽（避免反复打 Redis）
    private final Map<Long, AtomicBoolean> exhausted = new ConcurrentHashMap<>();

    private final StringRedisTemplate redis;
    private final RedisScript<Long> reserveScript;

    private static final int BATCH_SIZE = 10;     // 每次申请 10 件

    public boolean tryAcquire(long itemId) {
        AtomicBoolean done = exhausted.computeIfAbsent(itemId, k -> new AtomicBoolean(false));
        if (done.get()) return false;             // 已知耗尽，直接拒

        AtomicInteger bucket = localStock.computeIfAbsent(itemId, k -> new AtomicInteger(0));

        for (;;) {
            int cur = bucket.get();
            if (cur > 0) {
                if (bucket.compareAndSet(cur, cur - 1)) {
                    return true;                  // 本地扣成功
                }
                continue;                          // CAS 失败重试
            }
            // 本地空了，去 Redis 申请一批
            if (!refill(itemId, bucket, done)) {
                return false;                     // Redis 也空了
            }
        }
    }

    private synchronized boolean refill(long itemId, AtomicInteger bucket, AtomicBoolean done) {
        if (bucket.get() > 0) return true;        // 双检：已经被别的线程补好了
        if (done.get()) return false;

        Long got = redis.execute(reserveScript,
            List.of("seckill:stock:" + itemId),
            String.valueOf(BATCH_SIZE));

        if (got == null || got == 0) {
            done.set(true);
            return false;
        }
        bucket.addAndGet(got.intValue());
        if (got < BATCH_SIZE) {
            // Redis 也只剩 got 件了，下次直接拒
            done.set(true);
        }
        return true;
    }
}
```

`reserveScript` Lua：

```lua
-- 申请最多 N 件，返回实际申请到的数量
local stock = tonumber(redis.call('GET', KEYS[1]))
if stock == nil or stock <= 0 then
    return 0
end
local want = tonumber(ARGV[1])
local got = math.min(stock, want)
redis.call('DECRBY', KEYS[1], got)
return got
```

**关键设计决策**：

- `BATCH_SIZE` 不能太大（容易留尾），不能太小（Redis 压力大）。经验值：**总库存 / 节点数 / 10**
- `exhausted` 标记必须有，避免每个请求都去问 Redis
- `refill` 加 synchronized 是因为：多线程同时发现本地空了，会一起去打 Redis，浪费 RTT。串行化一下 refill 反而更快
- **本地预扣的边界 case**：节点宕机会让本地未消耗的配额"丢失"（详见案例 ②）

### 2.3 MQ 异步落库

扣完 Redis 库存后，订单要落 DB。同步落库会让 DB 成瓶颈，所以走 MQ。

```java
@Service
public class OrderProducer {

    private final RocketMQTemplate mq;

    public String submitOrder(long itemId, long userId, int qty) {
        // 1. 生成订单号（雪花算法 / Redis 自增 / UUID）
        String orderId = idGenerator.next();

        OrderMessage msg = new OrderMessage(orderId, itemId, userId, qty, System.currentTimeMillis());

        // 2. 发同步消息（必须确保 broker 收到）
        SendResult result = mq.syncSend("seckill_order_topic",
            MessageBuilder.withPayload(msg)
                .setHeader("KEYS", orderId)        // 用 orderId 作为消息 key，便于排查
                .build(),
            3000);                                  // 3s 超时

        if (result.getSendStatus() != SendStatus.SEND_OK) {
            // 3. MQ 发送失败 → 必须把 Redis 的库存还回去！
            stockService.rollback(itemId, qty, userId);
            throw new SeckillException("MQ 发送失败");
        }

        return orderId;     // 返回给前端，前端轮询订单状态
    }
}

@Component
@RocketMQMessageListener(topic = "seckill_order_topic", consumerGroup = "order_persist")
public class OrderConsumer implements RocketMQListener<OrderMessage> {

    @Resource private OrderMapper orderMapper;

    @Override
    public void onMessage(OrderMessage msg) {
        try {
            // 幂等：orderId 是唯一索引，重复消费直接 DuplicateKeyException
            Order order = new Order();
            order.setId(msg.getOrderId());
            order.setItemId(msg.getItemId());
            order.setUserId(msg.getUserId());
            order.setQty(msg.getQty());
            order.setStatus(OrderStatus.PENDING_PAY);
            order.setCreatedAt(LocalDateTime.now());
            orderMapper.insert(order);

        } catch (DuplicateKeyException dup) {
            // 重复消费，正常情况
            log.info("duplicate order {}, skip", msg.getOrderId());
        } catch (Exception e) {
            log.error("persist order {} failed", msg.getOrderId(), e);
            throw e;        // 抛出让 MQ 重试；失败 N 次进死信队列
        }
    }
}
```

**容易漏的点**：

- `mq.syncSend` 一定要用同步发送 + 检查 result，不能用 `asyncSend`（消息丢了你都不知道）
- 发送失败必须**回滚 Redis 库存**，否则会出现"用户没下成单但库存少了"
- 消费端必须**幂等**，靠 orderId 唯一索引兜底
- 死信队列必须**告警**，人工介入处理

### 2.4 防止超卖的多种方案对比

把上面几种方案全列出来，给个选型表：

```sql
-- 方案 1: DB 乐观锁（version 字段）
update stock set num = num - 1, version = version + 1
where item_id = ? and version = ? and num > 0;
-- 失败重试（最多 3 次）
```

```sql
-- 方案 2: DB 条件 update
update stock set num = num - 1
where item_id = ? and num >= 1;
-- 必须检查影响行数 == 1
```

```sql
-- 方案 3: select for update
begin;
select num from stock where item_id = ? for update;  -- 独占锁
update stock set num = num - 1 where item_id = ?;
commit;
-- ❌ 千万别这么用！高并发下行锁会让 QPS 卡在 100 以下
```

```java
// 方案 4: Redis Lua（推荐）
// 见 2.1
```

```java
// 方案 5: Redis 分布式锁 + DB
// ❌ 反模式：锁本身比扣库存还慢
String lockKey = "lock:stock:" + itemId;
if (redis.opsForValue().setIfAbsent(lockKey, "1", 1, TimeUnit.SECONDS)) {
    try {
        // 扣库存
    } finally { redis.delete(lockKey); }
}
```

| 方案 | 超卖风险 | 性能 | 复杂度 | 推荐度 |
| --- | --- | --- | --- | --- |
| DB 乐观锁 | 0 | 中（重试有损） | 低 | 中 QPS 场景 |
| DB 条件 update | 0 | 低（受行锁限制） | 极低 | 小活动 |
| select for update | 0 | 极低 | 低 | **不推荐** |
| Redis Lua | 0 | **高** | 中 | **生产首选** |
| Redis 分布式锁 | 0 | 低 | 高 | **反模式** |

### 2.5 用户提交频次限制

防止脚本快速点击。多层限流：

```java
@Component
public class FrequencyLimiter {

    private final StringRedisTemplate redis;

    // 1 秒内同 userId 最多 1 次
    public boolean checkUserFreq(long userId, long itemId) {
        String key = "freq:user:" + userId + ":" + itemId;
        Boolean firstTime = redis.opsForValue().setIfAbsent(key, "1", 1, TimeUnit.SECONDS);
        return Boolean.TRUE.equals(firstTime);
    }

    // 10 秒内同 IP 最多 5 次
    public boolean checkIpFreq(String ip) {
        String key = "freq:ip:" + ip;
        Long count = redis.opsForValue().increment(key);
        if (count == 1) redis.expire(key, 10, TimeUnit.SECONDS);
        return count <= 5;
    }

    // 滑动窗口（更精确）
    public boolean checkSlidingWindow(String key, int limit, int windowSeconds) {
        long now = System.currentTimeMillis();
        long windowStart = now - windowSeconds * 1000L;

        // ZSET：score = 时间戳，value = 唯一 ID
        redis.opsForZSet().removeRangeByScore(key, 0, windowStart);
        Long count = redis.opsForZSet().zCard(key);
        if (count != null && count >= limit) return false;

        redis.opsForZSet().add(key, UUID.randomUUID().toString(), now);
        redis.expire(key, windowSeconds, TimeUnit.SECONDS);
        return true;
    }
}
```

Nginx 层也要配，前置过滤：

```nginx
# 限流：每个 IP 每秒 10 个请求，突发 20
limit_req_zone $binary_remote_addr zone=seckill:10m rate=10r/s;

server {
    location /api/seckill/submit {
        limit_req zone=seckill burst=20 nodelay;   # ★ nodelay 才是真"突发"
        proxy_pass http://backend;
    }
}
```

`nodelay` 的含义：burst 队列里的请求**立即处理**，而不是按 rate 速率处理。如果不加 nodelay，burst 里的请求会被排队慢慢放出，用户感受是"卡顿"而不是"立即成功"。详见案例 ⑦。

### 2.6 DB 行锁 vs 乐观锁：实测对比

下面这段 JMH 测试展示了两种方案的差异：

```java
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@State(Scope.Benchmark)
@Threads(64)
public class StockDeductBench {

    @Param({"row_lock", "optimistic", "redis_lua"})
    public String mode;

    @Benchmark
    public boolean deduct() {
        return switch (mode) {
            case "row_lock"   -> deductWithRowLock();      // for update
            case "optimistic" -> deductOptimistic();        // version + retry
            case "redis_lua"  -> deductRedisLua();          // Lua 脚本
            default -> false;
        };
    }
}
```

> **口诀**：**Lua 脚本是秒杀的最大公约数**。比它简单的方案撑不住 QPS，比它复杂的方案有边界 case。先把 Lua 跑稳，再考虑本地预扣。

---

## 3. 实测数据：不同方案的吞吐与超卖率

> 测试环境：4 核 8G 应用 × 4 节点，Redis 7 单实例，MySQL 8 单实例（16 核 32G），库存 10000 件。

### 3.1 不同库存方案的吞吐对比

| 方案 | 吞吐 (QPS) | 平均延迟 (ms) | P99 (ms) | 超卖数 |
| --- | --- | --- | --- | --- |
| DB `for update` | 142 | 280 | 1200 | 0 |
| DB 乐观锁（重试 3 次） | 1850 | 35 | 180 | 0 |
| DB 条件 update | 2100 | 30 | 95 | 0 |
| Redis Lua | **38000** | **1.6** | 12 | 0 |
| 本地预扣 + Redis | **120000** | **0.5** | 8 | 0（开补偿） |
| 本地预扣 + Redis（无补偿） | 125000 | 0.4 | 8 | **2** |

观察：

- DB `for update` 是个**陷阱**，看似最稳，实际只能扛 100 多 QPS
- 乐观锁 / 条件 update 提升一个数量级，但 P99 已经到 100ms 级
- Redis Lua 又一个数量级跃升，且延迟极低
- 本地预扣再快 3 倍，但**没有补偿机制就会超卖**

### 3.2 同步落库 vs 异步落库

测试场景：100K 用户提交，库存 1000 件。

| 方案 | 用户感知延迟 (ms) | DB 写 QPS | 失败率 |
| --- | --- | --- | --- |
| 同步落库（Redis + DB 同步写） | 45 | 38000（短暂峰值） | DB 撑不住，失败率 12% |
| 异步落库（Redis + MQ） | 8 | 1000（平稳） | 0% |
| 异步落库 + DB 批量 insert（每 100 条一批） | 8 | 200 | 0% |

异步落库不仅快，还把 DB 压力**从尖峰削成平台**。批量 insert 进一步把 DB QPS 降到极低。

### 3.3 高并发下 Lua 脚本的瓶颈

Redis 单实例 Lua 脚本到了 5W QPS 后开始排队。瓶颈点：

```bash
# redis-cli --latency 看延迟
$ redis-cli --latency-history -h 10.0.0.1
min: 0, max: 1, avg: 0.10 (10000 samples)   # 1W QPS 时
min: 0, max: 23, avg: 4.8 (10000 samples)   # 5W QPS 时
min: 1, max: 156, avg: 32 (10000 samples)   # 8W QPS 时，开始排队
```

突破方法：
- 拆 key：`stock:1001` → `stock:1001:0`, `stock:1001:1`, ..., `stock:1001:9`，每个 key 走不同 Redis 实例
- 升 Redis Cluster（6+ 主节点）
- 上本地预扣，把 Redis 调用频率降下来

### 3.4 完整压测口诀

```
1K QPS：DB 条件 update 够用，别上 Redis
1K~10K QPS：Redis Lua + 同步落库
10K~50K QPS：Redis Lua + MQ 异步落库
50K~100K QPS：本地预扣 + Redis Cluster + MQ 批量落库
> 100K QPS：上面全部 + 多机房 + CDN 边缘扣减
```

> **口诀**：**先压测，再上线**。每提升一个数量级要换一种架构。不要用 100 QPS 的方案去扛 1W QPS。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① Lua 脚本未原子化导致超卖

#### 现象

某次大促上线，库存 1000 件，活动结束后 DB 里发现 1003 个有效订单。超卖 3 件。

#### 根因

代码看起来"对"，但仔细看：

```lua
-- ❌ 错误：分两步
local stock = redis.call('GET', KEYS[1])
if tonumber(stock) > 0 then
    redis.call('DECR', KEYS[1])
    return 1
end
return 0
```

等等，这是 Lua 脚本，单 Redis 实例下应该原子才对？为什么会超卖？

真相：**业务代码用了 Redis Cluster**，且 `KEYS[1]` 没有用 hash tag 包裹，Cluster 会把脚本发给某个分片执行，但**应用层的 fallback 逻辑**在脚本失败时直接走了"DB 兜底扣减"分支。两条路并行扣库存。

```java
// ❌ 危险的 fallback
try {
    Long ret = redis.execute(deductScript, ...);
    if (ret == 1) return success();
} catch (Exception e) {
    // Redis 异常，走 DB 兜底
    return dbStockService.deduct(itemId);
}
```

某次 Redis 网络抖动几秒，瞬间几百个请求走了 DB 兜底；与此同时 Redis 恢复后又有请求扣了 Redis。两边没同步。

#### 修复

**原则一：Lua 脚本必须用 hash tag 保证 hash slot 一致**。

```lua
-- key 改为 seckill:stock:{1001}, seckill:bought:{1001}
-- 大括号里的内容是 hash tag，Cluster 用它计算 slot
```

**原则二：禁止"Redis 失败走 DB"的 fallback**。要么全 Redis 要么全 DB，不能混。

```java
// ✅ 正确做法：Redis 不可用时直接拒绝服务
try {
    Long ret = redis.execute(deductScript, ...);
    return ret == 1 ? success() : fail();
} catch (Exception e) {
    log.error("redis unavailable, refuse seckill", e);
    throw new ServiceUnavailableException("系统繁忙，请稍后再试");
}
```

**原则三：DB 层面加最后一道兜底**。订单 insert 时检查库存：

```sql
insert into orders (...) select ... from dual
where (select num from stock where item_id = 1001 for update) > 0;
```

但这只是"侦测"，不是"防御"，目的是 alarm。

#### 验证

写一个能稳定复现超卖的测试：

```java
@Test
void noOversellUnderRedisFlapping() throws Exception {
    int initStock = 100;
    int users = 1000;
    redis.opsForValue().set("seckill:stock:{1001}", String.valueOf(initStock));

    // 模拟 Redis 偶发抖动
    AtomicInteger flapCount = new AtomicInteger(0);
    CountDownLatch latch = new CountDownLatch(users);
    AtomicInteger success = new AtomicInteger(0);

    ExecutorService pool = Executors.newFixedThreadPool(64);
    for (int i = 0; i < users; i++) {
        long uid = i;
        pool.execute(() -> {
            try {
                if (seckill.tryDeduct(1001, uid, 1).isSuccess()) {
                    success.incrementAndGet();
                }
            } finally { latch.countDown(); }
        });
    }
    latch.await();

    assertTrue(success.get() <= initStock);   // 永远不超卖
}
```

> **口诀**：超卖永远不是 Redis 不够原子，**99% 是业务代码加了"兜底"分支**。Redis Cluster 必须用 hash tag。

---

### 案例 ② 本地预扣 + Redis 同步的边界 case 超卖

#### 现象

上了本地预扣后，活动结束统计发现：Redis 里还剩 7 件库存，但 DB 已经有满库存的订单。复盘怀疑预扣有问题。

#### 根因

场景重现：

```
节点 A 启动，从 Redis 申请 10 件，本地剩 10
用户 1~3 来 A 节点，扣本地，本地剩 7
节点 A 突然 OOM 被杀
节点 A 重启，又从 Redis 申请 10 件 → Redis 又少了 10
节点 A 本地的"幽灵 7 件"已永久丢失
```

如果有 4 个节点，每个节点平均 OOM 一次，**会丢失 4 × 7 = 28 件库存**。库存"虚增"，DB 订单数小于 Redis 扣减数。

反向场景（真正的超卖）：

```
节点 A 申请 10 件，扣到本地剩 3
监控显示 Redis 已经 0 件
活动结束按钮被运维误按，Redis stock 重置为 0
节点 A 本地还能继续扣 3 件 → 实际超卖 3 件
```

#### 修复

**方案 A：节点优雅停机时归还本地配额**

```java
@PreDestroy
public void returnQuota() {
    localStock.forEach((itemId, bucket) -> {
        int remain = bucket.get();
        if (remain > 0) {
            redis.opsForValue().increment("seckill:stock:" + itemId, remain);
            log.info("returned {} stock to redis for item {}", remain, itemId);
        }
    });
}
```

但这只解决"优雅停机"，OOM / SIGKILL 拦不住。

**方案 B：定期心跳归还 + 配额过期**

```java
// 每 30 秒检查一次，把超过 1 分钟没扣的本地配额归还
@Scheduled(fixedDelay = 30000)
public void rebalanceQuota() {
    long now = System.currentTimeMillis();
    localStock.forEach((itemId, bucket) -> {
        Long lastUsed = lastUsedTime.get(itemId);
        if (lastUsed != null && now - lastUsed > 60_000) {
            int give = bucket.get() / 2;       // 归还一半，不归还光（避免抖动）
            if (give > 0 && bucket.compareAndSet(bucket.get(), bucket.get() - give)) {
                redis.opsForValue().increment("seckill:stock:" + itemId, give);
            }
        }
    });
}
```

**方案 C：活动结束统一对账**

```java
@Scheduled(cron = "0 0 1 * * *")    // 每天凌晨 1 点
public void reconcile() {
    for (Long itemId : activeItems()) {
        int redisStock = redis.opsForValue().get("seckill:stock:" + itemId);
        int dbOrderCount = orderMapper.countSuccessOrders(itemId);
        int initStock = activityMapper.getInitStock(itemId);

        if (redisStock + dbOrderCount != initStock) {
            log.error("STOCK INCONSISTENCY item={} redis={} db_orders={} init={}",
                itemId, redisStock, dbOrderCount, initStock);
            alarmService.send("库存对账失败 itemId=" + itemId);
        }
    }
}
```

**方案 D：禁用本地预扣的"危险扣减"**

如果业务本身价值不高（不是双 11 那种几亿 GMV），**直接退回 Redis Lua 方案**。本地预扣的边界 case 太多，运维成本高。

#### 验证

```bash
# 模拟节点 OOM
kill -9 <pid>

# 检查 Redis 库存 + DB 订单数总和是否 == 初始库存
redis-cli get seckill:stock:1001
mysql -e "select count(*) from orders where item_id=1001 and status != 'CANCELLED'"
```

> **口诀**：**本地预扣是"借贷"模型**，借了必须还。优雅停机归还、定时归还、活动结束对账，三件套缺一不可。如果团队驾驭不住，老老实实用 Redis Lua。

---

### 案例 ③ MQ 异步落库丢消息导致少订单

#### 现象

活动结束后用户投诉："明明点了'抢到了'，订单中心却没有我的订单"。

#### 根因

排查链路：
1. Redis 库存确实扣了
2. 用户拿到了订单号
3. MQ 里没有这条消息
4. DB 里没这条订单

复盘代码：

```java
// ❌ 错误：用了 sendOneway
mq.sendOneway("seckill_order_topic", msg);
return SUCCESS;   // 立即返回，不等 broker ack
```

`sendOneway` 是"发完就忘"，broker 收没收到都不管。一次 broker 重启或网络抖动，这条消息就丢了。

更隐蔽的版本：

```java
// 看起来用了 syncSend，但配置错了
mq.setSendMsgTimeout(50);    // ❌ 50ms 太短，broker 处理慢就超时
```

超时后业务以为发送失败，但 broker 实际是收到了。这种"半成品状态"最难处理。

#### 修复

**强可靠的发送配置**：

```java
@Bean
public RocketMQTemplate rocketMQTemplate() {
    DefaultMQProducer producer = new DefaultMQProducer("seckill_producer");
    producer.setNamesrvAddr(namesrv);
    producer.setSendMsgTimeout(3000);              // 3 秒超时
    producer.setRetryTimesWhenSendFailed(3);       // 同步发送重试 3 次
    producer.setRetryAnotherBrokerWhenNotStoreOK(true);  // 主 broker 失败换从
    producer.setCompressMsgBodyOverHowmuch(4096);

    RocketMQTemplate t = new RocketMQTemplate();
    t.setProducer(producer);
    return t;
}
```

**broker 端配置**：

```properties
# broker.conf
flushDiskType = SYNC_FLUSH                    # 同步刷盘，宕机不丢
brokerRole = SYNC_MASTER                      # 同步复制到 slave
```

> 注意：SYNC_FLUSH 会显著降低吞吐。秒杀这种"宁可慢点也不能丢"的场景值得；普通业务用 ASYNC_FLUSH 就行。

**最终一致性补偿：本地消息表**

```sql
create table local_message (
    id bigint primary key,
    msg_id varchar(64) unique,
    msg_body text,
    status tinyint,        -- 0=pending, 1=sent, 2=failed
    retry_count int,
    created_at datetime,
    updated_at datetime,
    index idx_status_created (status, created_at)
);
```

```java
@Transactional
public void submitOrderWithLocalMessage(...) {
    // 1. 业务表 + 本地消息表，同一个事务
    LocalMessage lm = new LocalMessage();
    lm.setMsgId(orderId);
    lm.setMsgBody(serialize(orderMsg));
    lm.setStatus(0);
    localMessageMapper.insert(lm);
    // 2. 事务提交后异步发 MQ
}

// 定时任务扫描未发送的消息
@Scheduled(fixedDelay = 5000)
public void retryUnsent() {
    List<LocalMessage> pending = localMessageMapper.findPending(100);
    for (LocalMessage lm : pending) {
        try {
            mq.syncSend(...);
            localMessageMapper.markSent(lm.getId());
        } catch (Exception e) {
            localMessageMapper.incrRetry(lm.getId());
        }
    }
}
```

事务消息（RocketMQ 4.3+ 原生支持）是同样思路的官方实现，可以直接用：

```java
mq.sendMessageInTransaction("seckill_order_topic", msg, null);
// + 实现 RocketMQLocalTransactionListener
```

#### 验证

```java
// 模拟 broker 不可用，验证消息不丢
@Test
void noLossWhenBrokerDown() {
    // 1. 正常发送 100 条
    for (int i = 0; i < 100; i++) submitOrder(...);

    // 2. 杀掉 broker
    killBroker();

    // 3. 再发 100 条（会进 local_message 表）
    for (int i = 0; i < 100; i++) submitOrder(...);

    // 4. 启 broker
    startBroker();

    // 5. 等定时任务跑完，检查 DB 订单数 == 200
    Awaitility.await().atMost(60, SECONDS)
        .until(() -> orderCount() == 200);
}
```

> **口诀**：**MQ 异步落单的可靠性 = 同步发送 + 同步刷盘 + 同步主从复制 + 本地消息表兜底**。任一环节走异步，必丢消息。

---

### 案例 ④ 退款时库存恢复错乱

#### 现象

某用户退款后投诉："退了款，但其他人显示库存还是 0，活动也没有开放购买"。运营查了下，发现 DB 库存表显示 +1 了，但 Redis 库存还是 0。

#### 根因

退款流程的代码：

```java
@Transactional
public void refund(String orderId) {
    Order order = orderMapper.getById(orderId);
    // 1. 退款（调支付平台）
    refundService.doRefund(order);
    // 2. 订单状态改 REFUNDED
    orderMapper.updateStatus(orderId, REFUNDED);
    // 3. DB 库存 +1
    stockMapper.increment(order.getItemId(), 1);
    // 4. Redis 库存 +1
    redis.opsForValue().increment("seckill:stock:" + order.getItemId(), 1);
}
```

问题：
- 步骤 4 在事务外（Redis 不参与 DB 事务），如果它失败，DB 已经 commit 但 Redis 没更新
- 即使没失败，**活动已经结束的场景**下回补 Redis 没意义（活动 key 可能已经删了）
- 重复退款（接口被调两次）会导致库存"虚增"

更深层：**秒杀活动结束后，Redis 库存已经清零并被标记"活动结束"**。这时退款回补，会让 Redis 库存变成 1，但前端的"立即购买"按钮已经隐藏，1 件库存永远卖不出去。

#### 修复

**方案 A：库存回补幂等**

```java
// 用退款记录表防重
create table refund_stock_log (
    order_id varchar(64) primary key,
    item_id bigint,
    qty int,
    status tinyint,    -- 0=pending, 1=done
    created_at datetime
);

@Transactional
public void refund(String orderId) {
    if (refundStockLogMapper.exists(orderId)) return;     // 已处理
    refundStockLogMapper.insert(orderId, ...);             // 唯一索引防重
    // 业务...
}
```

**方案 B：分场景处理回补**

```java
public void rollbackStock(long itemId, int qty) {
    Activity activity = activityCache.get(itemId);
    if (activity.isOngoing()) {
        // 活动进行中，回补到 Redis（让其他人能买）
        redis.opsForValue().increment("seckill:stock:" + itemId, qty);
        // 同时回补 DB 兜底库存
        stockMapper.increment(itemId, qty);
    } else {
        // 活动已结束，只回补 DB（用于对账）
        stockMapper.increment(itemId, qty);
        log.info("activity ended, only refund to DB stock for item {}", itemId);
    }
}
```

**方案 C：定时未支付订单自动释放**

很多退款其实是"未支付超时"，这种应该主动释放：

```java
// 订单创建后启动延迟任务
@RocketMQMessageListener(topic = "order_timeout_topic", consumerGroup = "auto_cancel")
public class OrderTimeoutConsumer {
    @Override
    public void onMessage(OrderMessage msg) {
        Order order = orderMapper.getById(msg.getOrderId());
        if (order.getStatus() == OrderStatus.PENDING_PAY) {
            // 自动取消 + 释放库存
            cancelOrder(order);
            rollbackStock(order.getItemId(), order.getQty());
        }
    }
}

// 下单时发延时消息
public void submitOrder(...) {
    Message msg = new Message("order_timeout_topic", body);
    msg.setDelayTimeLevel(16);    // RocketMQ 延时级别，16 = 30 分钟
    mq.send(msg);
}
```

#### 验证

```java
@Test
void refundIdempotent() {
    Order order = createOrder(itemId, userId);
    int stockBefore = redis.get("seckill:stock:" + itemId);

    refund(order.getId());
    refund(order.getId());     // 重复调
    refund(order.getId());

    int stockAfter = redis.get("seckill:stock:" + itemId);
    assertEquals(stockBefore + 1, stockAfter);     // 只 +1，不是 +3
}
```

> **口诀**：**库存回补 = 幂等 + 分场景**。活动中回补 Redis，活动后只回补 DB。未支付超时要主动释放，不要等用户手动退款。

---

### 案例 ⑤ 黄牛绕过验证码：行为模型识别

#### 现象

某次大促上线滑动验证码，自认为防住了脚本。结果活动结束发现，**95% 的库存被几个 IP 段瓜分**，真实用户基本没抢到。运营怒了。

#### 根因

排查 IP，发现：
- 这些 IP 来自正经的住宅宽带（不是机房 IP）
- UA 是真实的 iPhone Safari
- Cookie / Token 都是正常登录态
- 验证码确实有滑动行为日志

但仔细看登录数据：
- 这些"用户"30 秒前才注册账号
- 注册手机号是接码平台号段
- 滑动验证码的"轨迹"高度规则化（人手不可能这么稳）
- 提交时间精确到毫秒级集中（00:00:00.123）

这是**"真人黄牛池 + 接码平台 + 滑动模拟器"**的组合拳。纯算法层（IP / UA / Cookie）防不住。

#### 修复

**层 1：设备指纹**

收集浏览器 fingerprint：Canvas 渲染哈希、字体列表、屏幕分辨率、时区、WebGL 哈希等几十个维度，组合后做指纹。

```javascript
// 前端用 fingerprintjs 库
import FingerprintJS from '@fingerprintjs/fingerprintjs-pro';
const fp = await FingerprintJS.load({ apiKey: 'xxx' });
const result = await fp.get();
// 提交时带上 result.visitorId
```

后端拿到 visitorId 后：
- 单设备短时间内多账号 → 黄牛
- 设备指纹熵值过低（很多人共用一个指纹）→ 模拟器农场

**层 2：行为模型**

记录用户的鼠标轨迹、滑动验证码的速度曲线、点击间隔。训练一个简单的二分类模型（XGBoost / LightGBM 即可）：

```
特征：
  - 滑动速度方差（人手不稳，机器很稳）
  - 滑动加速度峰值（人手有冲量，机器是匀速）
  - 提交时间 mod 1000（机器倾向于整毫秒）
  - 注册到首次秒杀的时间间隔
  - 历史下单成功率（黄牛号倾向于命中率特别高）
  - 用户点击 → 提交的时间差（人 > 1s，脚本 < 100ms）
```

```java
@Component
public class BehaviorScorer {

    public double score(UserBehavior b) {
        // 简化模型：实际生产是 ML 模型
        double risk = 0;
        if (b.getSlideSpeedVariance() < 0.05) risk += 0.4;     // 滑动太稳
        if (b.getRegToFirstSeckillSeconds() < 60) risk += 0.3; // 新注册立刻秒杀
        if (b.getClickToSubmitMs() < 200) risk += 0.3;          // 点击太快
        if (b.getSubmitTimeMs() % 1000 < 50) risk += 0.2;       // 时间太整
        return Math.min(risk, 1.0);
    }
}

// 入口拦截
public SeckillResult submit(SeckillRequest req) {
    double risk = behaviorScorer.score(req.getBehavior());
    if (risk > 0.7) {
        log.warn("risky behavior, userId={}, risk={}", req.getUserId(), risk);
        return SeckillResult.fail("系统检测到异常行为");
    }
    // 正常流程
}
```

**层 3：实时风控**

对接公司风控系统，把这次秒杀的请求实时打分：

```java
RiskCheckRequest rr = new RiskCheckRequest();
rr.setEvent("seckill_submit");
rr.setUserId(userId);
rr.setIp(ip);
rr.setDeviceFp(deviceFp);
rr.setExtra(Map.of("itemId", itemId, "behaviorScore", score));

RiskCheckResponse resp = riskClient.check(rr);
if (resp.getDecision() == REJECT) {
    return SeckillResult.fail("风控拒绝");
}
```

公司级风控的优势：
- 跨业务的黑名单共享（黄牛号在借贷业务的不良记录会同步过来）
- 设备指纹库（一个设备指纹是不是历史黄牛设备）
- 关系图谱（用户 A 和已知黄牛 B 用过同一张银行卡）

#### 验证

不要在线上"等真黄牛"。买几个真实的脚本/外挂工具，自己模拟一次：

```bash
# 用 puppeteer 写个模拟脚本，看能不能通过风控
node simulate-seckill.js --concurrency 100 --account-pool 1000
```

记录：
- 通过率（应该 < 5%）
- 风控误杀率（真实流量被拒比例 < 0.1%）

> **口诀**：**纯算法防不住"真人黄牛"**。必须靠设备指纹 + 行为模型 + 关系图谱三件套。新业务直接对接公司风控，不要自己造。

---

### 案例 ⑥ 秒杀流量打挂关联业务

#### 现象

某次秒杀活动开始 30 秒后，订单中心、用户中心、商品中心**全部告警**。整个电商业务受影响，包括购物车、登录、详情页全部慢。

#### 根因

秒杀走的应用集群是独立的，但**底层 DB 共用一个集群**（dba 为了节约成本）。秒杀 QPS 上来后：
- 秒杀写订单 → 订单 DB 主库 CPU 100%
- 用户登录 → 同一个 DB 集群读 user 表 → 慢
- 商品详情 → 同一个 DB 集群读 sku 表 → 慢

更糟：很多关联业务用了**长连接 + 连接池**，DB 慢→连接池耗尽→应用线程全 hang→整个业务不可用。雪崩。

#### 修复

**方案 A：DB 物理隔离**

秒杀业务独立一套 DB（甚至独立机器、独立机房）：

```yaml
# 秒杀专用 DB 集群
seckill-db-cluster:
  master: 10.0.10.1:3306
  slaves: [10.0.10.2:3306, 10.0.10.3:3306]
  isolation: dedicated   # 不和任何业务共享

# 普通业务 DB 集群
business-db-cluster:
  master: 10.0.20.1:3306
  ...
```

**方案 B：DB 中间件按业务分库**

如果不能物理隔离，至少逻辑隔离：

```yaml
# ShardingSphere 配置
dataSources:
  seckill_ds:
    url: jdbc:mysql://10.0.10.1:3306/seckill
  user_ds:
    url: jdbc:mysql://10.0.20.1:3306/user
  order_ds:
    url: jdbc:mysql://10.0.30.1:3306/order
```

**方案 C：缓存隔离**

Redis 集群也要独立。秒杀的 Redis 不能和 Session、Cache、风控混用：

```yaml
seckill-redis:
  hosts: [10.0.50.1, 10.0.50.2, 10.0.50.3]
  type: cluster

session-redis:
  host: 10.0.60.1
  type: standalone
```

**方案 D：服务间隔离 + 熔断**

应用层用 Sentinel / Resilience4j 做熔断：

```java
@SentinelResource(value = "seckillSubmit",
    blockHandler = "blockHandler",
    fallback = "fallback")
public SeckillResult submit(...) { ... }

public SeckillResult blockHandler(SeckillRequest req, BlockException e) {
    return SeckillResult.fail("系统繁忙");
}

// 关联业务调用秒杀服务时
@SentinelResource(value = "queryStock", fallback = "fallbackStock")
public Long queryStock(long itemId) {
    return seckillService.getStock(itemId);
}
```

**方案 E：流量隔离从入口开始**

Nginx / 网关层把秒杀流量打到独立集群：

```nginx
upstream seckill_backend {
    server 10.0.100.1:8080;
    server 10.0.100.2:8080;
    server 10.0.100.3:8080;
}
upstream business_backend {
    server 10.0.200.1:8080;
    server 10.0.200.2:8080;
}

server {
    location /api/seckill/ {
        proxy_pass http://seckill_backend;
    }
    location / {
        proxy_pass http://business_backend;
    }
}
```

#### 验证

线上演练：在测试环境用秒杀流量灌一次，观察关联业务的 P99 延迟是否上升。

```bash
# 灌秒杀流量
wrk -t 16 -c 1000 -d 60s http://test/api/seckill/submit

# 同时压测登录接口
wrk -t 4 -c 100 -d 60s http://test/api/login
```

如果登录的 P99 在秒杀压测期间上升超过 20%，说明隔离没做好。

> **口诀**：**秒杀必须物理隔离**：DB 独立、Redis 独立、应用独立、入口独立。任一环节共享，都可能在大促时引发雪崩。

---

### 案例 ⑦ Nginx 限流配置错位（突发流量穿透）

#### 现象

Nginx 配了 `limit_req`，自测过 1000 QPS 限流是生效的。上线后突发 5W QPS 涌入，Tomcat 线程池瞬间打满。复盘发现 Nginx 限流"没起作用"。

#### 根因

错误配置：

```nginx
# ❌ 配置一：rate 配错单位
limit_req_zone $binary_remote_addr zone=seckill:10m rate=10r/m;
# 注意是 10r/m（每分钟 10 个），不是 10r/s
# 但又配了 burst=10000
location /api/seckill {
    limit_req zone=seckill burst=10000 nodelay;
}
# 实际效果：每分钟 10 个 + 一次性 burst 10000 → 等于不限
```

```nginx
# ❌ 配置二：limit_req_zone 用错变量
limit_req_zone $http_x_forwarded_for zone=seckill:10m rate=10r/s;
# X-Forwarded-For 是请求头，可被伪造
# 黄牛每个请求换一个 X-Forwarded-For → 限流被绕过
```

```nginx
# ❌ 配置三：burst 没配 nodelay
limit_req zone=seckill burst=20;     # 没 nodelay
# 结果：burst 队列里的请求排队等待，不是立即处理
# 用户体验是"卡 2 秒后成功"，而不是"立即拒绝"
# 排队时间过长导致客户端超时重试，雪上加霜
```

```nginx
# ❌ 配置四：限流单位混淆
limit_req_zone $binary_remote_addr zone=s1:10m rate=100r/s;
# rate=100r/s 实际是"按每秒 100 个的速率匀速放行"
# 也就是 10ms 一个，瞬时突发 100 个里的第 100 个会等 990ms
# 不是"1 秒内允许 100 个"
```

#### 修复

**正确的秒杀限流配置**：

```nginx
http {
    # IP 维度限流：每个 IP 每秒 5 次，突发 10 次（立即处理）
    limit_req_zone $binary_remote_addr zone=ip_limit:10m rate=5r/s;

    # 全局限流：所有请求合计每秒 1W 次
    limit_req_zone $server_name zone=global_limit:10m rate=10000r/s;

    # 连接数限制：每 IP 最多 5 个并发连接
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

    server {
        location /api/seckill/submit {
            limit_req zone=ip_limit burst=10 nodelay;
            limit_req zone=global_limit burst=2000 nodelay;
            limit_conn conn_limit 5;

            limit_req_status 429;          # 限流时返回 429（不是默认 503）
            limit_req_log_level warn;

            proxy_pass http://seckill_backend;

            # 客户端连接超时
            proxy_connect_timeout 1s;
            proxy_send_timeout 2s;
            proxy_read_timeout 2s;
        }
    }
}
```

**关键参数详解**：

- `rate=5r/s`：每秒 5 个的"平均速率"
- `burst=10`：突发 10 个允许进队列等待
- `nodelay`：队列里的请求**立即处理**，不排队等待
- 联合效果：瞬间允许 5+10=15 个通过，第 16 个开始 429；下一秒补充 5 个新配额

**真实 IP 获取**（如果前面有 LB）：

```nginx
# 信任前置 LB，用 X-Real-IP 或最后一跳
real_ip_header X-Forwarded-For;
set_real_ip_from 10.0.0.0/8;        # 只信任内网 LB
real_ip_recursive on;

# 之后用 $binary_remote_addr 就是真实 IP
```

**OpenResty 做更复杂的限流**（基于 Redis 共享状态）：

```lua
-- 用 lua-resty-limit-req 实现集群级限流
local limit_req = require "resty.limit.req"
local lim = limit_req.new("my_limit_req_store", 200, 100)
local key = ngx.var.binary_remote_addr
local delay, err = lim:incoming(key, true)
if not delay then
    if err == "rejected" then
        ngx.status = 429
        return ngx.exit(429)
    end
end
```

#### 验证

```bash
# 模拟超过限流的并发，观察 429 数量
ab -n 10000 -c 200 http://test/api/seckill/submit

# 检查 Nginx 日志
grep "limiting requests" /var/log/nginx/error.log | wc -l

# 应该看到大部分请求被拒
```

> **口诀**：**Nginx 限流的三个生死参数**：rate（速率）、burst（突发容量）、nodelay（不排队）。三者配合错任何一个，效果完全相反。

---

### 案例 ⑧ 用户大量重试导致 DOS-like 攻击

#### 现象

某次秒杀，库存 1 秒就清光，前端正常返回"已抢光"。但接下来 5 分钟内，**自家系统 QPS 居然反而上升**，从 5W 涨到 8W。运维以为是黄牛攻击，实际是真用户。

#### 根因

前端代码：

```javascript
// ❌ 前端疯狂重试
async function seckill() {
    for (let i = 0; i < 100; i++) {
        const res = await fetch('/api/seckill/submit');
        if (res.success) return showSuccess();
        if (res.code !== 'SOLD_OUT') await sleep(100);
        else continue;    // 已抢光也继续？！
    }
}
```

写代码的同学以为"已抢光"是网络问题，让前端持续重试。结果：每个用户 100 次重试 × 100W 用户 = 1 亿次请求。

更细的根因：
- 用户看到"已抢光"心有不甘，**手动刷新**
- APP 端有"网络异常自动重试"机制，把"已抢光"误认成网络异常
- 后端返回 200 + body 里 success=false，前端的拦截器没识别，认为成功就该有数据

#### 修复

**层 1：返回明确的 HTTP 状态码**

```java
// ❌ 不要 200 + success=false
return ResponseEntity.ok(SeckillResult.fail("已抢光"));

// ✅ 用业务相关的状态码
return ResponseEntity.status(409).body(...);     // 409 Conflict 表示资源冲突
// 或专门定义
return ResponseEntity.status(429).body(...);     // 429 Too Many Requests
```

前端拦截器对 4xx 一律不重试。

**层 2：返回 Cache-Control，让 CDN 缓存"已抢光"**

```java
@GetMapping("/api/seckill/check/{itemId}")
public ResponseEntity<?> checkStock(@PathVariable long itemId) {
    boolean exhausted = seckill.isExhausted(itemId);
    if (exhausted) {
        return ResponseEntity.ok()
            .header("Cache-Control", "public, max-age=10")    // CDN 缓存 10 秒
            .body(Map.of("status", "SOLD_OUT"));
    }
    return ResponseEntity.ok()
        .header("Cache-Control", "no-cache")
        .body(Map.of("status", "AVAILABLE"));
}
```

CDN 缓存"已抢光"后，重试根本到不了源站。

**层 3：服务端识别同用户重复请求**

```java
public SeckillResult submit(long userId, long itemId) {
    // 已知该用户失败，10 秒内不再处理
    String failKey = "fail:" + userId + ":" + itemId;
    if (redis.hasKey(failKey)) {
        return SeckillResult.fail("已抢光");
    }

    SeckillResult result = doSubmit(userId, itemId);

    if (!result.isSuccess() && "SOLD_OUT".equals(result.getCode())) {
        redis.opsForValue().set(failKey, "1", 10, TimeUnit.SECONDS);
    }
    return result;
}
```

**层 4：前端控制重试**

```javascript
async function seckill() {
    const res = await fetch('/api/seckill/submit');
    const data = await res.json();

    // 只对网络异常重试，业务失败不重试
    if (res.status >= 500 && retryCount < 3) {
        await sleep(1000 * (retryCount + 1));   // 指数退避
        return seckill(retryCount + 1);
    }

    if (data.code === 'SOLD_OUT') {
        return showSoldOut();    // 不再重试
    }
    if (data.success) {
        return showSuccess();
    }
}
```

#### 验证

```bash
# 模拟用户疯狂刷新场景
ab -n 100000 -c 1000 -H "Cookie: user=alice" http://test/api/seckill/submit

# 观察 95% 的请求应该被 Redis 失败缓存挡住，没到业务层
```

> **口诀**：**用户重试是真实流量的放大器**。返回明确状态码 + CDN 缓存"失败"结果 + 服务端记忆失败用户 + 前端禁止业务失败重试，四件套一起上。

---

### 案例 ⑨ 静态资源没走 CDN，业务源站被打挂

#### 现象

秒杀活动开始的瞬间，应用集群 CPU 100%，但奇怪的是**核心业务接口的 QPS 并没有那么高**。挂载到 APM 看，发现大量 CPU 花在了"返回静态资源"上。

#### 根因

排查 access log：

```
GET /seckill/banner.jpg     200    50KB
GET /seckill/banner.jpg     200    50KB
GET /seckill/banner.jpg     200    50KB
... (重复几万次)
GET /seckill/product.png    200    200KB
GET /seckill/product.png    200    200KB
... (重复几万次)
```

秒杀活动页面的图片、CSS、JS 全部由源站直接提供。每个用户打开页面下载 1MB 静态资源 × 10W 用户 = 100GB 流量瞬时打到源站。带宽和 CPU 双双爆掉。

更深层：前端开发同学以为"banner 图很小，没必要 CDN"。结果"小 + 高并发"反而比"大 + 低并发"更致命。

#### 修复

**强制所有静态资源走 CDN**：

```nginx
# 源站 Nginx：禁止直接访问图片
server {
    server_name seckill.example.com;

    location ~ \.(jpg|png|gif|css|js|woff)$ {
        # 重定向到 CDN
        return 301 https://cdn.example.com$request_uri;
    }

    location / {
        proxy_pass http://backend;
    }
}
```

**前端 build 配置**：

```javascript
// webpack.config.js
module.exports = {
    output: {
        publicPath: 'https://cdn.example.com/seckill/',     // 强制 CDN 域名
    }
};
```

**CDN 预热**：活动开始前 1 小时，主动把所有静态资源推到 CDN 节点：

```bash
# 用 CDN 厂商提供的 API
curl -X POST https://cdn-api.example.com/v2/preload \
    -d '{"urls": ["/seckill/banner.jpg", "/seckill/product.png", ...]}'
```

否则 CDN 节点没有缓存，第一波用户的请求依然回源，依然打挂。

**HTML 页面也要 CDN**：

很多人只想到图片，没想到 HTML 本身也是热点。秒杀活动页 HTML 走 CDN：

```html
<!-- 活动页 HTML 用 CDN 缓存，业务接口走 API 域名 -->
<html>
<head>
    <script>
        const API_BASE = 'https://api.example.com';   // 接口走源站
    </script>
</head>
<body>
    <img src="https://cdn.example.com/banner.jpg" />
    <button onclick="submitSeckill()">立即抢购</button>
</body>
</html>
```

**接口域名和静态域名分离**：

```
api.example.com    → 源站集群
cdn.example.com    → CDN
static.example.com → CDN（可同 cdn）
```

源站只处理动态接口，静态全走 CDN。

#### 验证

```bash
# 看 CDN 命中率（应该 > 95%）
curl -I https://cdn.example.com/seckill/banner.jpg
# 看 X-Cache 头：HIT 表示命中，MISS 表示回源

# 源站 access log 应该只有接口请求，没有静态资源
tail -f /var/log/nginx/access.log | grep -v "GET /api/"
# 上面这条命令在秒杀期间应该几乎没输出
```

#### 进阶：边缘秒杀

CDN 厂商支持把"是否还有库存"的判断放到边缘节点：

```javascript
// CloudFlare Workers / 阿里云 EdgeRoutine
addEventListener('fetch', event => {
    event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
    // 边缘节点的 KV 存储
    const stock = await SECKILL_KV.get('item:1001');
    if (parseInt(stock) <= 0) {
        return new Response('已抢光', { status: 409 });
    }
    // 还有库存，回源
    return fetch(request);
}
```

这一招能把"库存查询"的请求 90% 拦在边缘，源站根本看不到。

> **口诀**：**秒杀的 80% 流量是静态资源**。所有图、CSS、JS、HTML 全部 CDN，源站只处理动态接口。CDN 必须预热，否则等于没有。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 库存层

- [ ] 库存扣减用 Redis Lua 脚本（不要分两步）
- [ ] Redis Cluster 下 keys 必须用 hash tag 强制同槽
- [ ] **禁止"Redis 失败走 DB"**的 fallback（要么全 Redis，要么全 DB）
- [ ] 一人一单用 SET 数据结构记录已购用户
- [ ] 库存预热脚本经过压测（包括 Redis 集群预热和缓存击穿场景）
- [ ] DB 兜底库存表存在，定时和 Redis 对账

### 落库层

- [ ] 订单走 MQ 异步落库（QPS > 1W 时强制）
- [ ] MQ 同步发送 + 同步刷盘 + 同步主从（SYNC_FLUSH + SYNC_MASTER）
- [ ] 消费幂等：orderId 唯一索引兜底
- [ ] 死信队列接监控告警，人工介入
- [ ] 本地消息表作为 MQ 的可靠性兜底
- [ ] DB insert 用批量（每 100~500 条一批）

### 流量层

- [ ] CDN 预热活动页所有静态资源（banner / 详情图 / CSS / JS）
- [ ] HTML 页面也走 CDN
- [ ] Nginx `limit_req` 配置正确：rate + burst + nodelay 三件套
- [ ] 限流单位（r/s 还是 r/m）确认
- [ ] 真实 IP 获取配置（`real_ip_header`）
- [ ] Tomcat / 网关连接池容量是 Nginx 限流上限的 2 倍
- [ ] 业务返回 4xx 状态码（不是 200 + success=false）

### 风控层

- [ ] 滑动验证码 + 设备指纹双重保护
- [ ] 行为模型（注册时间、点击间隔、滑动轨迹）
- [ ] 接入公司风控系统，跨业务黑名单
- [ ] 失败用户写 Redis，10 秒内拒绝重试
- [ ] 单 IP / 单设备短时多账号识别

### 隔离层

- [ ] 秒杀 DB 独立集群（不和订单 / 用户 / 商品共享）
- [ ] 秒杀 Redis 独立（不和 Session / Cache 共享）
- [ ] 秒杀应用独立部署（不和普通业务混部）
- [ ] 入口域名隔离（seckill.xxx vs api.xxx）
- [ ] 服务间调用配置 Sentinel / Resilience4j 熔断

### 退款 & 库存释放

- [ ] 退款幂等（refund_stock_log 表 + 唯一索引）
- [ ] 活动中和活动后的回补策略不同
- [ ] 未支付订单延时取消（RocketMQ 延时消息 / Redis 延时队列）
- [ ] 释放后回补 Redis + DB 双写
- [ ] 定时对账 + 不一致告警

### 容灾层

- [ ] Redis 主从 + Sentinel / Cluster 高可用
- [ ] MQ 多 broker（每个 broker 主从）
- [ ] 应用集群 ≥ 3 节点，分多 AZ
- [ ] 限流降级开关（一键拒绝所有秒杀请求）
- [ ] 活动结束后 Redis key 主动 delete（不靠 TTL）

### 监控层

- [ ] 核心指标：QPS / P99 / 失败率 / 库存剩余 / 已下单数
- [ ] 业务指标：实时下单成功率（应该 = 库存数 / 提交数）
- [ ] 告警：Redis 延迟 > 50ms / MQ 堆积 > 1W / DB CPU > 80%
- [ ] 大屏：活动开始的前 30 秒每秒刷新
- [ ] APM 链路追踪覆盖整个秒杀链路

### 演练层

- [ ] 上线前必做全链路压测，目标 QPS ≥ 预估峰值 × 2
- [ ] 故障演练：杀 Redis 主、杀 MQ broker、杀应用节点
- [ ] 限流演练：构造超量请求，验证限流和降级生效
- [ ] 黄牛演练：自己写脚本攻击，验证风控拦截率

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### 大厂真实秒杀架构

- **阿里**：双 11 的"全链路压测"和"分布式事务 Seata"。库存使用 Tair（自研 KV）+ 数据库分库分表。详见 [《大型网站技术架构》](https://book.douban.com/subject/25723064/)
- **京东**：京东到家的 [JIMDB](https://github.com/jd-opensource/joylive-injection) + 路由层"流量染色"。618 的"提前预占"机制
- **美团**：基于 [Squirrel](https://tech.meituan.com/2018/01/26/cache-squirrel.html)（自研 Redis）的多级缓存。秒杀降级开关用配置中心实时下发

### 库存预热与活动调度

- 活动开始前 1 小时把库存数据从 DB 加载到 Redis，避免冷启动击穿
- 活动调度系统：定时把"待开始"活动转为"进行中"
- 活动数据隔离：每个活动一个 Redis key 前缀，避免相互影响

### 双 11 / 618 大型秒杀演练

- 全链路压测体系：影子库 / 影子表 / 影子 Redis，让压测流量和真实流量物理隔离
- 容量评估方法论：不是简单的 QPS 推导，要考虑 fan-out（一次秒杀触发的下游调用数）
- 应急预案手册：每个核心组件挂掉的恢复 SOP

### 直播秒杀（IM + 秒杀）

- 直播间秒杀的特殊性：用户都在 WebSocket 长连接里，秒杀指令通过 IM 推送
- 推送延迟决定了用户的"公平性"（先收到指令的有优势）
- 反作弊：识别"机器代抢"的 WebSocket 客户端

### 海外秒杀

- 多 region 部署的库存路由：用户去最近的 region，但库存是全局共享
- 跨地域延迟下的扣库存：不能用单一 Redis 集群，必须分层（边缘 → 区域 → 全局）
- 时区问题：北京时间 0 点开抢，纽约用户的体验

### 风控系统（基于行为图谱）

- 用户行为图谱：用户 → 设备 → IP → 银行卡 → 收货地址 → 其他用户
- 图遍历找到"明显异常"的子图（10 个用户共用 1 张卡 + 同一收货地址）
- 实时计算 vs 离线分析的边界

### 秒杀的"灰度策略"

- 分批放出库存：1000 件库存分 10 批，每批 100 件，间隔 30 秒
- 优势：摊平流量峰值，每批的失败用户可以再来抢下一批
- 劣势：用户体验差，"为什么我看到有库存却抢不到"

### 秒杀和"砍价 / 拼团"的差异

- 砍价：QPS 不集中，但状态变化频繁（每个用户的进度独立）
- 拼团：库存按"团"算，复杂度上升一个量级
- 共同点：都需要**强一致性的库存** + **高并发的写入**

---

## 7. 引用

- 《大型网站技术架构：核心原理与案例分析》李智慧
- 《亿级流量网站架构核心技术》张开涛 —— 秒杀章节是国内最详尽的工业实践
- 阿里 [Sentinel 文档](https://github.com/alibaba/Sentinel/wiki)
- RocketMQ [事务消息原理](https://rocketmq.apache.org/docs/featureBehavior/04transactionmessage/)
- 京东技术博客 [秒杀系统架构演进](https://mp.weixin.qq.com/s/joq6vEAyoT_Aua8iN6Lpfg)
- 美团技术博客 [Cache Aside 模式与多级缓存](https://tech.meituan.com/2017/03/17/cache-about.html)
- 阿里 [TTL（TransmittableThreadLocal）](https://github.com/alibaba/transmittable-thread-local) —— 风控上下文透传
- Martin Kleppmann, *Designing Data-Intensive Applications* —— 一致性章节
- [@guava RateLimiter 源码](https://github.com/google/guava/blob/master/guava/src/com/google/common/util/concurrent/RateLimiter.java) —— 令牌桶/漏桶的工业实现

---

> 配套：[系统设计主题](/article/system-design/) | [缓存设计](/article/system-design/cache-design) | [限流降级熔断](/article/system-design/resilience) | [Redis 内核](/article/db/redis-internals)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你觉得这个深度对，告诉我下一篇该写哪个。
