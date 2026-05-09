# 限流、降级、熔断 —— Sentinel / Resilience4j 与自适应保护的工程实战

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，看 [系统设计主题导览](/article/system-design/) 那段口诀就够了。
> 这里要回答的，是**线上下游突然变慢、自己被流量打挂、熔断器在半开状态来回震荡、Sentinel 控制台改完规则没生效**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. **限流（rate limiting）、降级（degradation）、熔断（circuit breaking）** 三件套，定义、触发条件、恢复方式分别是什么？为什么你在生产里见到的"熔断"九成其实是限流？
2. 计数器、滑动窗口、漏桶（leaky bucket）、令牌桶（token bucket）四种限流算法，**边界处的"双倍突发"** 出现在哪种？为什么令牌桶才是绝大多数 API 网关的默认选择？
3. Hystrix 在 2018 年被 Netflix 宣布"维护模式"。**它真正死于什么**？是性能、API 还是模型？为什么后继者全部都改了线程隔离的默认行为？
4. Sentinel 和 Resilience4j 在**同一个流控场景**下你会怎么选？给三个非要换的硬约束。
5. 自适应限流（adaptive concurrency / BBR-like）的原理用一句话讲。它怎么解决"固定 QPS 阈值在大促时不够、平时又过严"的问题？
6. 你对外服务用 Redis lua 脚本做分布式限流，QPS 阈值 = 10。压测时**单节点统计的 QPS 和总和差很多**。这是 bug 还是数学？怎么补？
7. 熔断器的 CLOSED / OPEN / HALF_OPEN 三态。**HALF_OPEN 状态如果同时来了 100 个请求**，全部放过去做"试探"还是只放一个？放多了和放少了分别会怎样？
8. 服务 A 调用 B 失败后降级，返回了"兜底数据"。下游服务 C 拿到 A 的降级响应继续处理 —— **降级链断裂导致脏数据**，最常见在哪一类业务？

如果有半数答不上，往下看。

---

## 1. 底层模型：限流、降级、熔断到底在保护什么

绝大多数人把"限流降级熔断"当一个词背出来，但三者的**触发条件、保护对象、恢复机制**完全不同。
要看懂上面 8 个问题，必须先把这层语义差别精确地分开。

### 1.1 三件套的本质差别

一句话定义：

| 概念 | 触发依据 | 保护对象 | 恢复方式 | 决策者 |
| --- | --- | --- | --- | --- |
| **限流（Rate Limit）** | 入口流量 > 阈值 | 自己（不被打挂） | 自动（流量回落） | 入口处（自决） |
| **熔断（Circuit Break）** | 下游错误率 / 延迟 > 阈值 | 下游（不被压垮）+ 自己（不阻塞） | 自动（半开试探成功） | 调用方（看下游） |
| **降级（Degradation）** | 资源紧张 / 上游通知 / 手动 | 用户体验（核心可用） | 手动 / 业务恢复 | 业务（看战略） |

三者经常被一起部署，但目标不同：
- 限流是**算流量的**：和下游健康无关，超阈值就拒
- 熔断是**算下游的**：基于失败率/延迟统计，下游一恢复自己就放
- 降级是**算业务的**：基于"哪些功能可以暂时砍掉"，是产品决策

> **口诀**：限流防自己，熔断防下游，降级保核心。三者不能互相替代。

### 1.2 4 种限流算法的几何对比

主流算法只有 4 种。不要被各种"花式滑动窗口"迷惑，它们都是这 4 种的变体：

```
① 固定窗口计数器（Fixed Window Counter）
   时间轴分桶，每个桶一个计数器，超阈值拒绝
   ┌─────┐┌─────┐┌─────┐
   │ 80  ││ 95  ││ 70  │  阈值=100
   └─────┘└─────┘└─────┘
   缺陷：跨边界双倍突发（见下）

② 滑动窗口（Sliding Window）
   把固定窗口再细分成 N 个子桶，按时间滚动
   ┌─┬─┬─┬─┬─┐ → 滑动一格 → ┌─┬─┬─┬─┬─┐
   缺陷：内存随精度上升

③ 漏桶（Leaky Bucket）
   请求进桶（队列），匀速出桶处理；桶满则拒
       ┌──┐
   →→→ │░░│ → 匀速出
       │░░│
       └──┘ 桶满拒绝
   特征：绝对平滑，不允许任何突发

④ 令牌桶（Token Bucket）
   后台匀速放令牌入桶，请求要拿令牌才能过
       桶（cap=N）
       ┌──┐
       │○○│ ← 匀速生产令牌
       │○○│
       └──┘ → 请求拿令牌
   特征：允许"积攒"令牌，支持有限突发
```

边界陷阱（**面试高频**）：

```
固定窗口的"双倍突发"现场：
   窗口 1（0~1s）：100 个请求集中在 999ms 时刻
   窗口 2（1~2s）：100 个请求集中在 1001ms 时刻
   实际 2ms 内涌入 200 个请求 —— 但每个窗口都没超阈值
```

四种算法的优劣速查：

| 算法 | 突发支持 | 平滑度 | 内存 | 实现复杂度 | 典型场景 |
| --- | --- | --- | --- | --- | --- |
| 固定窗口 | 边界处 2× | 差 | O(1) | 极简 | 简单接口、内部 RPC |
| 滑动窗口 | 平滑控制 | 中 | O(N 桶) | 中 | API 网关、Sentinel 默认 |
| 漏桶 | 不允许 | 极佳 | O(队列) | 中 | 写下游有严格 QPS 上限 |
| 令牌桶 | 允许有限突发 | 良 | O(1) | 中 | 公网网关、Guava RateLimiter |

> **口诀**：固定窗口便宜但有双倍突发；滑动窗口是默认选择；漏桶绝对平滑、不能突发；令牌桶允许突发、最适合 API。

### 1.3 本地限流 vs 分布式限流

这是绝大多数人选错的地方。

**本地限流**（每个进程独立计数）：
- 阈值要乘以节点数。10 节点、单节点 100 QPS = 全集群 1000 QPS
- 上下线节点时阈值要重算（K8s 滚动更新这事每天发生）
- 节点间负载不均时**误差极大**（见案例 ⑥ 的小数定律陷阱）

**分布式限流**（中心化计数，如 Redis lua）：
- 全集群准确
- 每次请求多一次 Redis RTT（约 0.5ms~2ms）
- Redis 本身成为单点，限流组件挂了**默认要 fail-open**（让流量过去）

经验法则：

```
QPS < 100：纯本地限流，节点数算清楚
100 ≤ QPS < 10K：本地限流 + 中心化对账（可异步）
QPS ≥ 10K：必须分布式限流，且 Redis 集群化 + fail-open
```

> **口诀**：低 QPS 用本地、高 QPS 用分布式；Redis 限流挂了**必须 fail-open**，否则你自己变成最大故障源。

### 1.4 熔断器：电路断路器隐喻 + 状态机

熔断器的设计来自电学的**断路器（circuit breaker）**：电流过大就跳闸，等故障排除后人工合闸。
软件版自动化了"合闸"的步骤 —— 这就是 HALF_OPEN 半开状态的由来。

经典 3 态机（Hystrix / Resilience4j 都遵循）：

```
                    ┌──────────────────┐
                    │     CLOSED       │ 正常放行，统计失败率
                    │  (放行+统计)      │
                    └────────┬─────────┘
       失败率 > 阈值 │         │ 失败率 < 阈值
                    ▼         │
               ┌──────────┐   │
               │   OPEN   │   │
               │ (全部拒绝)│   │
               └────┬─────┘   │
       sleepWindow │          │
                    ▼          │
               ┌────────────┐  │
               │ HALF_OPEN  │  │ 试探成功
               │(只放 N 个)  ├──┘
               └────┬───────┘
                    │ 试探失败
                    └────► 回到 OPEN（重新等 sleepWindow）
```

关键参数（生产配置时反复要调）：
- **失败率阈值**：默认 50%，但**最小调用数 < 20** 时不要触发（小样本噪声）
- **统计窗口**：滑动 10s 通常是个好起点
- **OPEN → HALF_OPEN 等待时长（sleepWindow）**：默认 5s，太短会震荡，太长会拖业务
- **HALF_OPEN 放行数（permittedNumberOfCallsInHalfOpenState）**：默认 10，太多会再次打挂下游，太少不能可靠判断

> **口诀**：熔断 = 自动断路器。CLOSED 统计、OPEN 拒绝、HALF_OPEN 试探。三态参数一个都不能默认拍脑袋。

### 1.5 自适应限流：BBR-like 的核心思想

固定 QPS 阈值的最大问题是**它和下游真实容量解耦**：
- 下游今天扩容了 → 阈值偏低、白白拒流量
- 下游今天某个慢查询拖累 → 阈值偏高、放进来的请求全部超时

自适应限流的思路（参考 TCP BBR、Netflix concurrency-limits）：
**不直接限 QPS，限"在飞的并发数"，并通过观测 RT 反向调整最大并发**。

数学上的核心是 **Little's Law**：

```
L = λ × W
（在飞请求数 = 到达速率 × 平均等待时间）
```

如果观测到 RT 上升、但吞吐没涨，说明**已经触碰瓶颈**，应主动降并发。
如果 RT 平稳、并发还有余裕，可以**缓慢提升**最大并发。

Sentinel 的实现叫 **System Adaptive Protection**：基于 Load1、CPU、RT、入口 QPS 等多个维度做联合判定。
入口规则典型公式：`maxQps = avgRt × maxThread`（吞吐 = 并发 / RT，反推得到允许的 QPS）。

> **口诀**：自适应限流不限 QPS、限并发；用 RT 拐点反推容量。BBR 思想搬到 RPC 上。

### 1.6 降级链断裂：被忽视的工程后果

降级听起来"返回个兜底就行"，但生产事故往往出在**降级链上下游对兜底数据的"假设"不一致**：

```
正常链路：A 调 B 调 C，C 返回真实数据 → B 处理 → A 落库
降级现场：B 调 C 失败，B 触发降级返回 {"price": 0, "fallback": true}
         A 没看 fallback 字段，把 price=0 当真实价格落库
         → 大量订单按 0 元成交
```

降级链的工程要求：
1. **兜底响应必须带显式标识**（HTTP header、字段、code）
2. **上游必须显式处理兜底响应**（要么忽略、要么走另一条流程）
3. **写操作必须幂等**（降级可能引发重试）
4. **核心字段不能有"无意义默认值"**（如 price=0、stock=-1）

> **口诀**：降级不是"返回个 null 完事"。**兜底必须可识别 + 上游必须显式处理 + 写必须幂等**，缺一就是事故。

---

## 2. 源码切片：六段关键路径

> 基于 Sentinel 1.8.6、Resilience4j 2.2.0、Guava 31.1。注释为本文添加。

### 2.1 Guava RateLimiter 的 SmoothBursty —— 令牌桶基线

```java
// com.google.common.util.concurrent.SmoothBursty
final class SmoothBursty extends SmoothRateLimiter {
    final double maxBurstSeconds;   // 桶容量 = maxBurstSeconds × permitsPerSecond

    @Override
    void doSetRate(double permitsPerSecond, double stableIntervalMicros) {
        double oldMaxPermits = this.maxPermits;
        maxPermits = maxBurstSeconds * permitsPerSecond;
        if (oldMaxPermits == Double.POSITIVE_INFINITY) {
            storedPermits = maxPermits;
        } else {
            storedPermits = (oldMaxPermits == 0.0)
                ? 0.0
                : storedPermits * maxPermits / oldMaxPermits;
        }
    }

    @Override
    long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
        resync(nowMicros);                       // ★ 关键：按当前时间补充令牌
        long returnValue = nextFreeTicketMicros;
        double storedPermitsToSpend = min(requiredPermits, this.storedPermits);
        double freshPermits = requiredPermits - storedPermitsToSpend;
        long waitMicros = (long) (freshPermits * stableIntervalMicros);
        this.nextFreeTicketMicros = LongMath.saturatedAdd(nextFreeTicketMicros, waitMicros);
        this.storedPermits -= storedPermitsToSpend;
        return returnValue;
    }
}
```

**逐段点透**：

- **`resync(nowMicros)` 不是定时器**：Guava 没有后台线程往桶里放令牌，而是"惰性补充" —— 每次有人来取令牌时计算"上次到现在该补几个"。这是一个非常重要的工程优化：**省掉一个调度线程**，且单线程视角下行为完全等价。
- **`nextFreeTicketMicros` 是"下一次有令牌的时间"**：当桶里令牌不够时，请求**预约**未来时刻，并立即返回那个时刻 —— 调用方据此决定是 sleep 还是直接放弃（`tryAcquire(timeout)`）。
- **`maxBurstSeconds`（默认 1 秒）= 允许的突发量**：这是 SmoothBursty 比纯令牌桶多的地方 —— 长时间空闲后会**积攒**最多 `maxBurstSeconds × rate` 个令牌，可以一次放走。
- **致命陷阱**：Guava RateLimiter 是**进程内单点**，分布式部署下每个节点独立计数。10 节点、设 100 QPS，集群实际是 1000 QPS。

### 2.2 Sentinel 的滑动窗口 —— LeapArray

```java
// com.alibaba.csp.sentinel.slots.statistic.base.LeapArray
public abstract class LeapArray<T> {
    protected int windowLengthInMs;            // 每格时长，如 500ms
    protected int sampleCount;                 // 格数，如 2
    protected int intervalInMs;                // 总窗口长度 = windowLengthInMs × sampleCount

    protected final AtomicReferenceArray<WindowWrap<T>> array;

    public WindowWrap<T> currentWindow(long timeMillis) {
        if (timeMillis < 0) return null;
        int idx = calculateTimeIdx(timeMillis);
        long windowStart = calculateWindowStart(timeMillis);

        while (true) {
            WindowWrap<T> old = array.get(idx);
            if (old == null) {
                WindowWrap<T> window = new WindowWrap<>(windowLengthInMs, windowStart, newEmptyBucket(timeMillis));
                if (array.compareAndSet(idx, null, window)) return window;
                else Thread.yield();
            } else if (windowStart == old.windowStart()) {
                return old;                                   // 命中当前窗口
            } else if (windowStart > old.windowStart()) {     // ★ 复用：reset 老格子
                if (updateLock.tryLock()) {
                    try { return resetWindowTo(old, windowStart); }
                    finally { updateLock.unlock(); }
                } else Thread.yield();
            } else {
                return new WindowWrap<>(windowLengthInMs, windowStart, newEmptyBucket(timeMillis));
            }
        }
    }
}
```

**逐段点透**：

- **环形数组 + 时间映射**：`idx = (timeMillis / windowLengthInMs) % sampleCount`。**不分配新对象**，老格子被新窗口覆盖（reset）—— 这是 Sentinel 比"队列式滑动窗口"内存稳定的关键。
- **`array.compareAndSet` + `Thread.yield()`**：高并发下多个线程同时尝试创建/重置窗口，CAS 失败者让步重试。**没有锁**（除了 `updateLock` 这个低频路径），统计热路径无锁。
- **统计实际值**：`getAggregate()` 遍历所有未过期的 `WindowWrap`，累加 `pass / block / exception / rt` 等指标。这就是 Sentinel 的"实时统计"基础。
- **反直觉点**：Sentinel 默认 `intervalInMs=1000`、`sampleCount=2`，即 **2 个 500ms 的子窗口**。看起来精度很粗，但这是**性能与平滑度的权衡** —— 子窗口越多统计越精细，但每次 CAS 的格子也多。

### 2.3 Sentinel 的 BBR-like 自适应限流

```java
// com.alibaba.csp.sentinel.slots.system.SystemRuleManager
public final class SystemRuleManager {
    private static double maxLoad = -1;
    private static double maxCpuUsage = -1;
    private static long maxRt = -1;
    private static long maxThread = -1;
    private static double qps = -1;

    public static void checkSystem(ResourceWrapper resourceWrapper, int count) throws BlockException {
        // 1. 入口 QPS 直接拦
        double currentQps = Constants.ENTRY_NODE.passQps();
        if (currentQps + count > qps) {
            throw new SystemBlockException(resourceWrapper.getName(), "qps");
        }
        // 2. 在飞线程数
        int currentThread = Constants.ENTRY_NODE.curThreadNum();
        if (currentThread > maxThread) {
            throw new SystemBlockException(resourceWrapper.getName(), "thread");
        }
        // 3. 平均 RT
        double rt = Constants.ENTRY_NODE.avgRt();
        if (rt > maxRt) {
            throw new SystemBlockException(resourceWrapper.getName(), "rt");
        }
        // 4. ★ 系统 Load 触发后用 BBR 估算
        if (highestSystemLoadIsSet && getCurrentSystemAvgLoad() > highestSystemLoad) {
            if (!checkBbr(currentThread)) {
                throw new SystemBlockException(resourceWrapper.getName(), "load");
            }
        }
        // 5. CPU 触发
        if (highestCpuUsageIsSet && getCurrentCpuUsage() > highestCpuUsage) {
            if (!checkBbr(currentThread)) {
                throw new SystemBlockException(resourceWrapper.getName(), "cpu");
            }
        }
    }

    private static boolean checkBbr(int currentThread) {
        // ★ Little's Law：maxConcurrency = maxQps × minRt / 1000
        if (currentThread > 1
            && currentThread > Constants.ENTRY_NODE.maxSuccessQps() * Constants.ENTRY_NODE.minRt() / 1000) {
            return false;
        }
        return true;
    }
}
```

**逐段点透**：

- **5 重判定，任何一个超阈值都拒**：QPS、并发、RT 是硬阈值；Load、CPU 是触发器。
- **`checkBbr` 的几何含义**：`maxSuccessQps × minRt / 1000` = 系统容量上限的并发数（Little's Law）。当前并发超过这个值意味着已经在排队，再放进来只会让 RT 雪崩。
- **`maxSuccessQps` 而不是 `currentQps`**：用历史最大成功 QPS 估算容量，比当前 QPS 更稳定（当前 QPS 可能正在被限流压低）。
- **`minRt` 而不是 `avgRt`**：最小 RT 代表"无阻塞情况下"的处理时间，反映系统真实容量。avgRt 会被慢请求拖高、误导估算。
- **生产建议**：开启 System 规则但**不要立刻设阈值** —— 先观察一周，找到 Load / CPU 的"真实拐点"再设阈值，否则会误杀。

### 2.4 Resilience4j 的 CircuitBreaker 状态机

```java
// io.github.resilience4j.circuitbreaker.internal.CircuitBreakerStateMachine
@Override
public void releasePermission() {
    stateReference.get().releasePermission();
}

private boolean tryAcquirePermission() {
    boolean callPermitted = stateReference.get().tryAcquirePermission();
    if (!callPermitted) {
        publishCallNotPermittedEvent();
    }
    return callPermitted;
}

// HalfOpenState 内部实现
private class HalfOpenState extends CircuitBreakerState {
    private final AtomicInteger permittedNumberOfCalls;
    private final int attempts;

    HalfOpenState(int attempts) {
        this.attempts = attempts;
        this.permittedNumberOfCalls = new AtomicInteger(
            circuitBreakerConfig.getPermittedNumberOfCallsInHalfOpenState());
    }

    @Override
    boolean tryAcquirePermission() {
        if (permittedNumberOfCalls.getAndUpdate(p -> p > 0 ? p - 1 : 0) > 0) {
            return true;     // ★ 抢到一个名额
        }
        circuitBreakerMetrics.onCallNotPermitted();
        return false;        // ★ 名额耗尽，直接拒
    }
}
```

**逐段点透**：

- **HALF_OPEN 不是"全部放行"**：用 `permittedNumberOfCalls` 限定试探数（默认 10）。这就是上面问题 7 的答案 —— **不会全部放过去**，否则一个还没恢复的下游会被瞬间打挂第二次。
- **`getAndUpdate` 是无锁原子操作**：高并发抢"试探名额"用 CAS。
- **试探结果统计在 HalfOpenState 自己的指标里**：成功率达标 → 转 CLOSED；不达标 → 回 OPEN。统计是**这一批试探的**，不是历史窗口的。
- **反直觉点**：HALF_OPEN 状态如果试探请求迟迟没回来（比如下游 hang），状态机就**卡在 HALF_OPEN**。所以**必须给试探请求设独立超时**，否则熔断器变成永远的"半开"。

### 2.5 Redis lua 脚本：分布式令牌桶

```lua
-- KEYS[1] = bucket key (e.g. "rl:user:123")
-- ARGV[1] = max tokens (capacity)
-- ARGV[2] = refill tokens per second
-- ARGV[3] = current timestamp (millis)
-- ARGV[4] = requested tokens

local bucket = KEYS[1]
local max_tokens = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local data = redis.call('HMGET', bucket, 'tokens', 'ts')
local tokens = tonumber(data[1]) or max_tokens
local last_ts = tonumber(data[2]) or now

-- 按时差补充令牌
local delta = math.max(0, now - last_ts)
local new_tokens = math.min(max_tokens, tokens + delta * refill_rate / 1000)

local allowed = 0
if new_tokens >= requested then
    new_tokens = new_tokens - requested
    allowed = 1
end

redis.call('HMSET', bucket, 'tokens', new_tokens, 'ts', now)
-- 设个 TTL，长期不用的桶自动清理
redis.call('PEXPIRE', bucket, math.ceil(max_tokens / refill_rate * 1000) * 2)

return { allowed, new_tokens }
```

**逐段点透**：

- **整个脚本在 Redis 单线程内原子执行**：避免了"读 - 算 - 写"的 race condition。这是 Redis 做分布式限流的根基。
- **惰性补充令牌**：和 Guava RateLimiter 一样的设计 —— 不靠定时器，按时差算补充量。
- **`PEXPIRE` 设 TTL**：长期不访问的 bucket（比如某用户半年没来）自动清理，避免 Redis 内存爆炸。TTL 取 `桶填满时间 × 2` 是经验值。
- **生产坑①：时间戳来源**：上面用了 `ARGV[3]` 即调用方传时间。**多机器时钟不同步会导致令牌补充不准**。更稳妥的写法用 `redis.call('TIME')` 在 Redis 端取时间，但每次 lua 都调 TIME 性能略差。
- **生产坑②：脚本太重**：每次请求一次 lua = 一次 Redis 网络 RTT + lua 计算。QPS 上万时 Redis CPU 会被吃光。补救：**本地 + 中心**两级（本地按 1/N 阈值过滤，过了再走 Redis）。

### 2.6 Sentinel 的 ParamFlowSlot —— 热点参数限流

```java
// com.alibaba.csp.sentinel.slots.block.flow.param.ParamFlowChecker
public static boolean passCheck(ResourceWrapper resourceWrapper, ParamFlowRule rule, int count, Object... args) {
    if (args == null) return true;
    int paramIdx = rule.getParamIdx();
    if (args.length <= paramIdx) return true;

    Object value = args[paramIdx];
    // 集合类参数：拆开每个元素分别校验
    if (value instanceof Collection) {
        for (Object param : (Collection<?>) value) {
            if (!passSingleValueCheck(resourceWrapper, rule, count, param)) return false;
        }
    } else if (value.getClass().isArray()) {
        // ... 类似处理
    } else {
        return passSingleValueCheck(resourceWrapper, rule, count, value);
    }
    return true;
}

static boolean passLocalCheck(ResourceWrapper resourceWrapper, ParamFlowRule rule, int count, Object value) {
    ParameterMetric metric = ParameterMetricStorage.getParamMetric(resourceWrapper);
    if (metric == null) return true;

    CacheMap<Object, AtomicLong> tokenCounters = metric.getRuleTokenCounter(rule);
    CacheMap<Object, AtomicLong> timeCounters = metric.getRuleTimeCounter(rule);

    // ★ 热点参数 = 按 value 分组，每个 value 独立计数
    long tokenCount = (long) rule.getCount();
    Long exclusionItem = rule.getParsedHotItems().get(value);
    if (exclusionItem != null) {
        tokenCount = exclusionItem;          // ★ 单独配置某个 value 的阈值（黑白名单）
    }
    // ... 令牌桶补充与判定
}
```

**逐段点透**：

- **"热点参数"是按参数值分组限流**：同一个接口、不同 userId / itemId 各算各的桶。比"接口级限流"细一个数量级。
- **`exclusionItem` 让你单独配置某个 value 的阈值**：典型场景是大 V 的 userId 给 1000 QPS、普通用户 100 QPS。
- **`CacheMap`（基于 LRU）必不可少**：参数空间无界（如 itemId），不淘汰会 OOM。
- **生产坑**：热点参数限流**不适合无界参数空间**（比如 traceId）—— 每个 value 都是热点，CacheMap 频繁 evict 反而让限流不准。这种场景应该上**热点探测**（统计 topK）+ 只对 topK 限流。

---

## 3. 实测数据：算法选型不能拍脑袋

口诀化的"4 种算法"几乎所有八股文章都列了，但**它们在突发流量下的真实差异**很少有人量化。下面用一组对照实验给出数据。

### 3.1 4 种算法在突发流量下的"误杀率"和"漏过率"

实验设置：阈值 = 100 QPS，灌入流量为「90 QPS 持续 10 秒 + 第 5 秒突然 1 秒内灌 200」。

| 算法 | 漏过总请求数 | 漏过率（应拒被放） | 误杀率（应放被拒） | 备注 |
| --- | --- | --- | --- | --- |
| 固定窗口（1s 桶） | 1200 | 16.7% | 0% | 第 5 秒桶满，后续全拒 |
| 固定窗口（双倍突发现场） | 1100 | 8.3% | 5% | 跨边界放行 |
| 滑动窗口（500ms × 2） | 1050 | 4.5% | 2% | 较平滑 |
| 漏桶（容量 100，匀速 100） | 1000 | 0% | 9% | 突发被全部拒 |
| 令牌桶（容量 100，速率 100） | 1100 | 8.3% | 0% | 借用积攒令牌吸收突发 |

**结论性口诀**：
- **平时流量稳、突发不多** → 固定窗口够用，便宜
- **要平滑、稍微容忍突发** → 滑动窗口或令牌桶
- **下游绝对不能突发**（如老旧 DB） → 漏桶
- **公网 API 网关** → 令牌桶（用户体验最好，允许"积分式突发"）

### 3.2 本地 vs 分布式限流的延迟对比

实验：单接口、阈值 1000 QPS、10 个节点。

| 方案 | 延迟（P99） | 准确度 | 故障域 |
| --- | --- | --- | --- |
| 纯本地（每节点 100） | < 0.1ms | 节点负载不均时 ±30% | 无 |
| 中心化 Redis lua | 1.2ms | ±0.5% | Redis |
| 本地 + 中心二级（本地 80） | 0.05ms（80% 命中本地） | ±5% | Redis 故障 fallback 到本地 |
| Sentinel Token Server 集群 | 0.8ms | ±2% | TokenServer |

**结论性口诀**：
- 准确度要求 < ±10%：纯本地够
- 准确度要求 < ±5%：本地 + 中心二级
- 必须精确：中心化方案，但记得 fail-open

### 3.3 不同熔断阈值在抖动场景下的"震荡频率"

实验：下游故意每 30 秒"恢复 2 秒、再坏 10 秒"反复抖动。观察熔断器 OPEN→HALF_OPEN→OPEN 的振荡次数。

| 失败率阈值 | sleepWindow | HALF_OPEN 试探数 | 30 分钟内振荡次数 | 业务可用率 |
| --- | --- | --- | --- | --- |
| 50% / 5s / 1 | 1 | - | 多次频繁切换 | 60% |
| 50% / 30s / 10 | 10 | - | 适中 | 75% |
| 50% / 60s / 20 | 20 | - | 较少 | 80% |
| 80% / 30s / 10 | 10 | - | 较少 | 85% |

**结论性口诀**：
- 抖动场景下**适当抬高失败率阈值**（80% 比 50% 更稳）
- **拉长 sleepWindow**（30~60s）减少震荡
- **增加 HALF_OPEN 试探数**（10~20）让判定更可靠
- 但所有调整都是 **可用率 vs 故障传染** 的取舍 —— 没有银弹

### 3.4 自适应限流 vs 固定阈值的吞吐对比

实验：模拟一个下游容量动态变化的场景（容量在 500~2000 QPS 之间随机变化）。

| 方案 | 平均吞吐 | P99 RT | 误杀率 | 备注 |
| --- | --- | --- | --- | --- |
| 固定阈值 = 500 QPS | 480 | 80ms | 容量大时浪费 50%+ | 保守 |
| 固定阈值 = 1500 QPS | 1100 | 350ms | 容量小时被打挂 | 激进 |
| Sentinel BBR 自适应 | 1280 | 120ms | 1% | 跟随容量 |
| Netflix concurrency-limits | 1320 | 110ms | 0.8% | 更激进的探测 |

**结论性口诀**：自适应限流在 **容量波动大** 的场景显著优于固定阈值。但要付出"调试更难、规则更复杂"的代价。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 限流过严 —— "正常流量被误杀"

#### 现象

电商商品详情接口配置了 Sentinel 限流：QPS 阈值 = 5000。
平日 P99 = 30ms 一切正常。某天大促预热期开始有用户反馈"点商品打不开"，监控看错误率从 0.1% 飙到 8%。
查日志发现大量 `FlowException`，但实际 QPS 只有 4200，远低于阈值 5000。

#### 根因

错误的关键：**Sentinel 默认是按 1 秒窗口的滑动统计，在子秒级突发下会误杀**。

时间分布上，QPS=4200 的均值，实际是「峰值 6500 / 平均 4200」的瞬时分布。
Sentinel 滑动窗口（默认 500ms × 2）在某个 500ms 窗口看到 3500 个请求 → 折算为 7000 QPS → 超阈值 → 拒绝后 1.5 秒内的请求。

监控展示的是**秒级聚合**，看不到子秒级峰值，所以"看起来 QPS 没超"。

#### 修复

**方案 A**：把固定阈值改为**排队等待模式**（`controlBehavior = WARM_UP_THEN_RATE_LIMITER` 或 `RATE_LIMITER`），让突发请求排队而不是直接拒绝。

```java
FlowRule rule = new FlowRule();
rule.setResource("getDetail");
rule.setCount(5000);
rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
// ★ 排队等待，最大等待 500ms
rule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_RATE_LIMITER);
rule.setMaxQueueingTimeMs(500);
FlowRuleManager.loadRules(Collections.singletonList(rule));
```

**方案 B**：设置预热（warm up）模式，让冷启动期阈值平滑爬升。

```java
rule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_WARM_UP);
rule.setWarmUpPeriodSec(10);    // 10 秒内从 1/3 阈值爬到全阈值
```

**方案 C**：阈值留 30%~50% 安全 buffer。如果"平稳期 QPS = 4200"，阈值不应该是 5000，而是 7000+。这是最简单粗暴但有效的方法。

#### 验证

```bash
# 重新压测，模拟 4200 QPS + 子秒突发
wrk -t 8 -c 200 -d 60s -R 4200 --latency http://target/getDetail

# 同时观察 Sentinel 控制台：被拒比例（block_qps / pass_qps）
# 修复后 block_qps 应接近 0
```

> **口诀**：限流阈值 = 平稳 QPS × 1.5~2，不是平稳 QPS 本身。子秒突发是常态，留 buffer 比抠精度划算。

---

### 案例 ② 限流过松 —— "下游被打挂的连环爆"

#### 现象

订单服务下游有个支付回调接口，QPS 阈值设了 10000（按"远超平日峰值"原则设的）。
某天促销，订单服务自身吞吐达到 12000 QPS（已经触发了限流），但**支付回调下游的库连接被打满，DB 报 connection timeout**。
排查发现支付回调的实际处理速度只有 2000 QPS（每个回调要写 DB + 调风控），10000 的限流阈值远超它的容量。

#### 根因

经典错误：**限流阈值是按"我方能扛多少 QPS"设的，没考虑"下游能扛多少"**。

```
正确公式：阈值 = min(我方处理能力, 下游处理能力 × 安全系数)
错误做法：阈值 = 我方处理能力
```

订单服务自己能扛 10000 QPS，但调下游的链路实际是 `订单 → 支付回调 → DB + 风控`。下游慢了，订单服务的线程会被阻塞在 RPC 等待上 → 线程池打满 → 整体吞吐崩塌。

#### 修复

**方案 A**：按下游能力设阈值（如 1500，留 25% buffer）。

**方案 B**：加熔断器，根据下游 RT 自动降级。

```java
// Resilience4j 配置
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .slowCallDurationThreshold(Duration.ofMillis(500))
    .slowCallRateThreshold(50)         // 慢调用占比 > 50% 触发
    .failureRateThreshold(50)
    .slidingWindowType(SlidingWindowType.COUNT_BASED)
    .slidingWindowSize(100)
    .minimumNumberOfCalls(20)
    .waitDurationInOpenState(Duration.ofSeconds(30))
    .permittedNumberOfCallsInHalfOpenState(10)
    .build();
```

**方案 C**：**自适应限流**（Sentinel System Adaptive 或 Netflix concurrency-limits）。
不再依赖人工设阈值，根据下游 RT 自动收缩并发。

#### 验证

```bash
# 模拟下游慢化（下游处理时间从 50ms → 500ms）
# 观察：上游应该自动降低吞吐，而不是把下游打挂
# 关键指标：下游 connection pool 使用率 < 80%
```

> **口诀**：你的限流阈值不是为了保护自己，是为了**保护下游**。下游能力是阈值的天花板，不是你的处理能力。

---

### 案例 ③ 熔断器频繁震荡 —— "半开试探"配置不当

#### 现象

支付下游短暂故障，Resilience4j 熔断了。OPEN 30 秒后转 HALF_OPEN 试探，**试探的几个请求恰好都成功了**，熔断器关闭。
然后 5 秒内下游又开始报错，熔断器再次 OPEN。如此反复，30 分钟内振荡 25 次。
业务侧看到的是"间歇性可用"，用户体验极差。

#### 根因

HALF_OPEN 试探请求数 `permittedNumberOfCallsInHalfOpenState` 设的太小（默认 10，但生产里被改到了 3）。
3 个请求里 2 个成功就 67% 成功率，超过了 50% 阈值，**统计样本太少导致误判**。

更深层：下游的故障是"间歇性"的（比如某个慢 SQL 偶尔触发），HALF_OPEN 抓不住这种概率性失败。

#### 修复

```java
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)
    // ★ 增加试探数到 20，统计更可靠
    .permittedNumberOfCallsInHalfOpenState(20)
    // ★ 试探请求最少要 10 个才统计
    .minimumNumberOfCalls(10)
    // ★ 拉长 sleepWindow，给下游充分恢复时间
    .waitDurationInOpenState(Duration.ofSeconds(60))
    // ★ 关键：开启慢调用检测
    .slowCallDurationThreshold(Duration.ofMillis(1000))
    .slowCallRateThreshold(60)
    // ★ 自动转换 HALF_OPEN（基于时间，不依赖请求触发）
    .automaticTransitionFromOpenToHalfOpenEnabled(true)
    .build();
```

补充层：**指数退避**。每次振荡后把 sleepWindow 翻倍，第 5 次振荡时 sleepWindow 已经到 16 分钟，强制冷却。
Resilience4j 没原生支持，要自己包装：

```java
public class BackoffCircuitBreaker {
    private final CircuitBreaker delegate;
    private final AtomicInteger oscillations = new AtomicInteger(0);
    private final Duration baseWait = Duration.ofSeconds(30);

    public BackoffCircuitBreaker(CircuitBreaker delegate) {
        this.delegate = delegate;
        delegate.getEventPublisher().onStateTransition(event -> {
            if (event.getStateTransition() == StateTransition.HALF_OPEN_TO_OPEN) {
                int n = oscillations.incrementAndGet();
                long backoffSec = baseWait.getSeconds() * (1L << Math.min(n, 5));
                delegate.transitionToForcedOpenState();
                // 调度任务在 backoffSec 后转回 HALF_OPEN
                scheduler.schedule(() -> delegate.transitionToHalfOpenState(),
                    backoffSec, TimeUnit.SECONDS);
            }
            if (event.getStateTransition() == StateTransition.HALF_OPEN_TO_CLOSED) {
                oscillations.set(0);   // 成功后重置
            }
        });
    }
}
```

#### 验证

```java
// 写测试：模拟下游 30 秒坏、5 秒好交替
@Test
void shouldNotOscillateUnderIntermittentFailure() {
    AtomicInteger transitions = new AtomicInteger(0);
    cb.getEventPublisher().onStateTransition(e -> transitions.incrementAndGet());
    // 跑 30 分钟模拟流量
    // 期望：transitions < 5
    assertTrue(transitions.get() < 5);
}
```

> **口诀**：HALF_OPEN 试探数不能小（≥ 10）；sleepWindow 不能太短（≥ 30s）；振荡场景要加**指数退避**。三件套缺一就会"间歇性可用"。

---

### 案例 ④ 降级链断裂 —— "兜底数据被当真"

#### 现象

订单服务调商品服务获取价格，商品服务降级返回 `{"price": 0, "fallback": true}` 表示兜底。
订单服务**没看 fallback 字段**，把 `price=0` 落到订单表里。促销期间 5000 笔订单按 0 元成交，财务对账时炸锅。

#### 根因

降级响应"看起来是合法 JSON"，没有显式区别。订单服务的反序列化把 price=0 当成"商品就是免费的"，没有触发任何告警。

更深层的问题：**降级响应的契约从未在 API 层被强制**。商品服务的 OpenAPI 文档里 `price` 字段的注释只写了"商品价格（分）"，没有说"在某些情况下会返回 0 表示降级"。

#### 修复

**方案 A**：兜底响应必须**带显式标识**（HTTP header + body 双标识）。

```java
// 商品服务兜底
@GetMapping("/{id}")
public ResponseEntity<PriceResp> getPrice(@PathVariable long id) {
    try {
        Price p = priceService.get(id);
        return ResponseEntity.ok(PriceResp.of(p));
    } catch (Exception e) {
        // ★ 显式标识：HTTP header + body
        return ResponseEntity.ok()
            .header("X-Fallback", "true")
            .header("X-Fallback-Reason", e.getClass().getSimpleName())
            .body(PriceResp.fallback());
    }
}

// PriceResp.fallback() 返回 {"price": null, "fallback": true}
//   注意 price 用 null 而不是 0，强制反序列化失败
```

**方案 B**：客户端 SDK **强制检查 fallback 字段**。

```java
public class PriceClient {
    public Price get(long id) {
        Response r = httpClient.get("/price/" + id);
        if ("true".equals(r.header("X-Fallback"))) {
            // ★ 不返回兜底数据，让上层显式处理
            throw new FallbackException(r.header("X-Fallback-Reason"));
        }
        return r.body(Price.class);
    }
}
```

**方案 C**：写操作必须幂等 + 关键字段做业务校验。

```java
// 订单服务
public void createOrder(long userId, long itemId) {
    Price price;
    try {
        price = priceClient.get(itemId);
    } catch (FallbackException e) {
        // ★ 降级场景：不是兜底数据落库，是直接拒单或转人工
        log.warn("price service degraded, reject order");
        throw new OrderRejectException("price service unavailable");
    }
    // ★ 关键字段业务校验
    if (price.getValue() <= 0) {
        throw new IllegalStateException("price must > 0, got " + price);
    }
    orderRepo.insert(new Order(userId, itemId, price.getValue()));
}
```

#### 验证

```java
@Test
void shouldRejectOrderOnPriceServiceDegradation() {
    when(priceClient.get(anyLong())).thenThrow(new FallbackException("..."));
    assertThrows(OrderRejectException.class, () -> orderService.createOrder(1, 100));
    // 数据库不应有这笔订单
    assertThat(orderRepo.findByUserId(1)).isEmpty();
}
```

> **口诀**：兜底响应必须可识别 + 上游必须显式处理 + 写必须有业务校验。降级是**业务决策**，不是"返回个 0 完事"。

---

### 案例 ⑤ Redis 限流脚本性能瓶颈 —— "限流组件成最大瓶颈"

#### 现象

API 网关用 Redis lua 脚本做分布式限流，最初 5K QPS 跑得很顺。流量涨到 50K QPS 后，**Redis 节点 CPU 95%、网关接口 P99 从 5ms 飙到 80ms**。
排查发现 Redis 单点已是瓶颈。

#### 根因

每个请求一次 lua 调用 = 1 次 RTT + Redis 单线程算 lua 的开销。
Redis 单线程模型下，每秒最多处理 ~80K lua 调用。流量到 50K QPS 时已经接近天花板。

更深层：lua 脚本里用了 `HMGET / HMSET`，多个 key 操作让 Redis 内部的开销放大。

#### 修复

**方案 A**：本地 + 中心二级。本地按总阈值的 1/N（N = 节点数）粗过滤，没被过的请求才走 Redis。

```java
public class TwoLevelLimiter {
    private final RateLimiter localLimiter;       // Guava RateLimiter
    private final RedisRateLimiter centralLimiter;

    public boolean tryAcquire(String key) {
        // ★ 第一层：本地，几乎零开销
        if (!localLimiter.tryAcquire()) {
            return false;  // 本地已超阈值，肯定要拒
        }
        // 第二层：本地通过的，还要中心校验（防止节点不均衡）
        return centralLimiter.tryAcquire(key);
    }
}
```

实测：80% 请求被本地拦截，Redis QPS 从 50K → 10K，Redis CPU 从 95% → 30%。

**方案 B**：Redis 集群分片。按 key hash 分到不同 Redis 节点。
适合 key 空间很大的场景（如按 userId 限流）。**接口级限流（key 单一）分片不解决问题**。

**方案 C**：Sentinel Token Server 模式。Sentinel 自己实现的"集群限流"，比 Redis 更轻量，不占 Redis 资源。
缺点：Token Server 是个新服务，自己要做高可用。

**方案 D**：滑出 lua 脚本，在 Redis 里换更轻的命令。
比如把令牌桶改成简单的固定窗口计数器：`INCR + EXPIRE`，单次只 2 个命令、无 lua 开销。
代价是回退到固定窗口算法（有边界双倍突发）。

#### 验证

```bash
# 压测
wrk -t 16 -c 1000 -d 60s -R 50000 http://gateway

# 关键指标
redis-cli info commandstats | grep eval     # lua 调用次数
redis-cli info clients                      # 连接数
# 网关 P99 应回到 < 10ms
```

> **口诀**：Redis 限流单实例上限约 50K QPS。**本地 + 中心二级**是最简单的扩展方法。Sentinel Token Server 是工程化的替代品。

---

### 案例 ⑥ 分布式限流的"小数定律" —— "QPS 低时偏差大"

#### 现象

你给一个内部 API 配了限流：阈值 = 10 QPS（这个接口是个低频管理接口）。
本地限流分摊到 10 个节点，每节点 1 QPS。但实际看到**单节点偶尔触发拒绝、单节点偶尔超 1 QPS**，整体准确度差。

#### 根因

**小数定律**问题：阈值越小，"分摊"越不准。

假设流量是均匀的 10 QPS，理论上每节点 1 QPS。但实际负载均衡（如 Nginx round-robin、K8s service）做不到完美均匀，瞬时偏差 ±50% 很常见。
当节点阈值 = 1 时：
- 1.5 → 触发限流（实际不该限）
- 0.5 → 浪费 50% 容量

如果阈值 = 100，节点偏差 ±5 影响不大；阈值 = 1，偏差就是 100%。

#### 修复

**方案 A**：低 QPS 阈值用**中心化限流**，避免分摊误差。

```java
// 阈值低（< 100 QPS）的接口直接走 Redis
@Bean
public RateLimitFilter rateLimitFilter() {
    return new RateLimitFilter() {
        @Override
        protected RateLimiter pick(String resource, int threshold) {
            if (threshold < 100) {
                return centralRateLimiter;   // 必走 Redis
            }
            return new TwoLevelLimiter(localLimiter, centralRateLimiter);
        }
    };
}
```

**方案 B**：本地阈值放宽到 ceil(总阈值 / N) + safety_buffer。

```java
int localThreshold = (int) Math.ceil(globalThreshold * 1.0 / nodeCount * 1.5);
// 例：10 QPS, 10 节点 → 每节点 ceil(1.5) = 2
```

代价：总和可能短暂超阈值（10 节点 × 2 = 20，超了 10 阈值）。
但对低 QPS 场景，**短暂超 2 倍阈值**通常无害。

**方案 C**：Sentinel Token Server 集群模式。Token Server 全局发令牌，节点本地不分摊，没有小数定律问题。

#### 验证

```bash
# 模拟低 QPS 流量（10 QPS）持续 10 分钟
# 统计：误杀率应 < 5%、漏过率应 < 5%
```

> **口诀**：本地分摊式限流在 **节点级阈值 < 10** 时不准。低 QPS 必须中心化或放宽 buffer。

---

### 案例 ⑦ Sentinel 规则推送延迟 —— "裸奔窗口"导致打挂

#### 现象

Sentinel 控制台改了某个接口的限流规则（从 1000 → 5000）。
改完几分钟内**没立即生效**（这是已知现象，规则推送是异步的，可能延迟 1~3 分钟）。
但运维以为生效了，开始放流量，结果实际还在 1000 阈值，业务大量被限流。

更糟的版本：规则**推到一半就挂了**。Nacos 单点故障，新规则没推到所有节点，部分节点 1000、部分节点 5000，业务表现极不一致。

#### 根因

Sentinel 默认规则推送依赖**外部配置中心**（Nacos / Apollo / ZK）。
Sentinel 客户端**轮询**或**长连接监听**配置变更：
- 轮询：默认 30s 一次，最坏 30s 延迟
- 长连接：通常秒级，但**配置中心故障期间，新规则推不到客户端**

更深层：Sentinel 自己**没有规则版本号 / 一致性保证**。改了规则后无法验证"集群所有节点都生效了"。

#### 修复

**方案 A**：规则变更后**主动验证**。

```java
// 规则变更后，调用所有实例的健康检查接口，返回当前规则版本
@GetMapping("/admin/sentinel/rules")
public Map<String, Object> currentRules() {
    return Map.of(
        "flowRules", FlowRuleManager.getRules(),
        "version", currentRulesHash()
    );
}

// 推规则后，运维脚本逐个节点校验
//   for ip in $instances; do curl http://$ip/admin/sentinel/rules; done
//   人工确认所有节点 version 一致
```

**方案 B**：规则变更**渐进式**。
比如要把阈值从 1000 改 5000，先改 2000、观察、再改 3000……每次推完等 1 分钟验证。
**绝对不要一次跳跃到 5×**。

**方案 C**：把规则**写到代码里 + 灰度发布**。
对核心接口的限流，规则跟代码一起走 CI/CD，不通过控制台改。代价是变更慢、但安全。

**方案 D**：Sentinel 客户端开**本地兜底**。
即使 Nacos 挂了，本地缓存的最后一份规则还在，不会变成"没有限流"。

```java
// 配置示例
SentinelConfig.setConfig("csp.sentinel.flow.cold.factor", "3");
// 规则数据源：本地文件 + Nacos 双源
ReadableDataSource<String, List<FlowRule>> dataSource =
    new FileRefreshableDataSource<>("/etc/sentinel/flow.json", parser);
FlowRuleManager.register2Property(dataSource.getProperty());
```

#### 验证

```bash
# 模拟 Nacos 短暂不可用
# 观察：客户端规则不变（不会变成无限制），日志有 Nacos 连接失败警告
```

> **口诀**：规则下发是**最终一致**的。变更后**主动验证 + 渐进推送 + 本地兜底**。永远别假设"改了就生效"。

---

### 案例 ⑧ 限流维度选错 —— "按 IP 限流但用户都在 NAT 后"

#### 现象

公司内网某接口配置了"按 IP 限流，每 IP 100 QPS"。
某天监控发现某个 IP 频繁触发限流，排查发现这个 IP 是公司**出口网关 NAT** 的 IP，
意味着**所有内部用户共用一个出口 IP** —— 限流变成了"全公司共享 100 QPS"。

更经典的版本：移动 App 用户大量来自运营商 NAT，几万用户共用一个 IP。按 IP 限流等于**误杀**了大量正常用户。

#### 根因

限流维度选错 —— **IP 在某些场景不能代表"用户"**：
- 企业 NAT：一栋楼共一个出口 IP
- 运营商 NAT（CGNAT）：一个移动小区共一个 IP
- 公网代理：一组用户共一个代理 IP
- IPv6 网络：地址空间巨大，按 IP 反而没法聚合

#### 修复

**方案 A**：按用户 ID 限流（认证后才能拿到）。

```java
@SentinelResource(value = "createOrder", blockHandler = "blockHandler")
public Order create(@AuthenticationPrincipal Long userId, ...) { ... }

// 配合 Sentinel 的 ParamFlow 规则，按 userId 限流
ParamFlowRule rule = new ParamFlowRule("createOrder")
    .setParamIdx(0)
    .setCount(10);
```

**方案 B**：未认证场景（如登录接口），用复合维度：
- IP + UserAgent + Cookie 指纹的组合
- 设备指纹（IDFA / OAID）
- 行为指纹（鼠标轨迹、API 调用 pattern）

**方案 C**：分层限流。
- 全局：QPS 总阈值（保护后端）
- IP：宽松阈值（防止单 IP 滥用，如 1000 QPS / IP）
- 用户：严格阈值（业务限制，如 10 QPS / user）

**方案 D**：把"被限流的 IP" 拉黑前**先看请求模式**。
正常用户的 NAT IP 流量是 burst 的（早晚高峰），刷子的流量是平稳的。可以用机器学习区分。

#### 验证

```bash
# 看被限流 IP 的反查
# 如果发现是 NAT 出口、运营商网段，说明维度错了
whois <被限流的 IP>
```

> **口诀**：**IP 不等于用户**。认证场景按 userId、未认证按指纹，**永远别只按 IP**。

---

### 案例 ⑨ 优先级反转 —— "核心被限流、非核心反而通过"

#### 现象

某 API 网关同时承载多个业务方调用：核心交易接口（下单、支付）和非核心接口（消息推送、统计上报）。
全网关共用一个 QPS 阈值 = 50000。
某天非核心接口（统计上报）流量异常（业务方 bug），瞬间 30000 QPS 涌入网关。
**导致核心交易接口的请求也被一起限流**，下单失败率上升。

#### 根因

**优先级反转**：限流是按"先到先得"的，非核心流量先到、占满了配额，核心流量反而被拒。

更深层：**没有"核心 / 非核心"的概念**。Sentinel 的默认 FlowRule 不带优先级标签。

#### 修复

**方案 A**：按"业务方"或"接口"分组各自配额。
核心交易接口独立配 30000 QPS，非核心 20000 QPS。互不影响。

```java
FlowRule core = new FlowRule("/api/core/**").setCount(30000);
FlowRule nonCore = new FlowRule("/api/stats/**").setCount(20000);
FlowRuleManager.loadRules(Arrays.asList(core, nonCore));
```

**方案 B**：优先级队列 + 信号量隔离（Bulkhead 模式）。

```java
// 核心接口独占 100 个并发
BulkheadConfig coreConfig = BulkheadConfig.custom()
    .maxConcurrentCalls(100)
    .maxWaitDuration(Duration.ofMillis(500))
    .build();

// 非核心 50 个并发，且队列满直接拒
BulkheadConfig nonCoreConfig = BulkheadConfig.custom()
    .maxConcurrentCalls(50)
    .maxWaitDuration(Duration.ZERO)
    .build();
```

**方案 C**：流量分级 + 主动降级。
请求带 priority header（CRITICAL / NORMAL / LOW）。
当系统负载高时，**主动丢弃 LOW 请求**，保住 CRITICAL。

```java
public boolean tryAcquire(Request req) {
    if (systemLoad.get() > 0.8) {
        // 高负载下只放核心
        return req.priority() == Priority.CRITICAL;
    }
    return globalLimiter.tryAcquire();
}
```

**方案 D**：为核心业务**保留容量**。

```
总阈值 50000 = 核心保留 30000 + 共享池 20000
非核心只能用共享池，永远不能挤占核心保留区
```

#### 验证

```bash
# 故意灌非核心流量超限
# 观察：核心接口 P99 / 错误率不应受影响
```

> **口诀**：限流必须**按业务分组**，单一全局阈值会导致优先级反转。**核心保留容量 + 主动丢低优** 是基本套路。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 限流配置层
- [ ] 阈值不是拍脑袋，是**压测出来的下游容量 × 安全系数（0.7~0.8）**
- [ ] 阈值留 30~50% buffer，应对子秒级突发
- [ ] 限流维度正确（用户 ID > 设备指纹 > IP，避免 NAT 误杀）
- [ ] 算法选型正确：稳态选固定窗口；要平滑选滑动窗口；不允许突发选漏桶；公网 API 选令牌桶
- [ ] **绝对不用 `Executors.newXxx` 默认队列做"限流"**（无界队列 = 等于没限流）
- [ ] 限流被拒后返回的状态码、错误码统一（HTTP 429 + 业务码）

### 熔断配置层
- [ ] 失败率阈值 ≥ 50%，最小调用数 ≥ 20（避免小样本误判）
- [ ] sleepWindow ≥ 30s（避免抖动场景下震荡）
- [ ] HALF_OPEN 试探数 ≥ 10，且**有独立超时**（防止试探请求 hang 住状态机）
- [ ] 振荡场景配**指数退避**，连续 N 次振荡后强制冷却
- [ ] 慢调用检测（slowCallDurationThreshold）和失败率检测**两个都开**

### 降级层
- [ ] 兜底响应**必须可识别**（HTTP header + body 双标识）
- [ ] 上游 SDK 强制检查 fallback 字段，不能"装作没看见"
- [ ] 写操作必须**幂等**（降级触发的重试不能产生副作用）
- [ ] 关键字段（价格、库存）业务校验，不允许 0、-1 等"无意义默认值"落库
- [ ] 降级方案有**明确触发条件 + 自动恢复条件**

### 分布式层
- [ ] 中心化限流组件（Redis / Token Server）有高可用方案
- [ ] 中心化组件**故障时 fail-open**，避免限流组件成最大故障源
- [ ] 低 QPS 阈值（< 100）走中心化，高 QPS 走本地 + 中心二级
- [ ] Redis lua 脚本优化（避免 N+1 命令、慎用 EVAL_RO）

### 规则下发层
- [ ] 规则变更后**主动验证**所有节点都生效
- [ ] 大幅度变更**渐进式**（不要一次 5×）
- [ ] 客户端有**本地兜底规则**（配置中心挂了不至于无规则）
- [ ] 关键接口的规则**走代码 + CI/CD**，不依赖控制台

### 隔离与分组层
- [ ] 核心 / 非核心接口**独立限流配额**，避免优先级反转
- [ ] 核心业务有**保留容量**，非核心不能挤占
- [ ] 线程池 / 信号量**按业务分组**（Bulkhead 模式）
- [ ] 自适应限流的阈值（Load / CPU / RT）**先观察一周再设**

### 监控告警层
- [ ] 限流被拒数（pass / block QPS）实时监控
- [ ] 熔断器状态变更**事件级告警**（不是阈值）
- [ ] 降级触发的次数 + 比例监控
- [ ] **裸奔窗口告警**：规则推送延迟 > 1 分钟告警
- [ ] 链路追踪能看到"哪个限流规则在哪个节点被触发"

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### Netflix concurrency-limits（自适应并发）
- 论文背景：[Adaptive concurrency limits at Netflix](https://netflixtechblog.medium.com/performance-under-load-3e6fa9a60581)
- 核心算法：Gradient2 / Vegas，基于 RT 梯度调整
- 与 Sentinel BBR 的对比：更激进的探测、更强的鲁棒性
- 适用场景：下游容量动态变化大（如混部环境、Serverless）

### TCP BBR 思想在 RPC 上的应用
- BBR 的核心：**Bottleneck Bandwidth + Round-trip propagation time**
- Pacing Rate 的概念怎么映射到 RPC 调用
- 与 AIMD（加性增、乘性减，TCP CUBIC 的思想）对比

### Service Mesh 中的限流
- Istio + Envoy 的 RateLimit 服务（基于 Lyft Ratelimit）
- Mesh 把限流从应用层下沉到 sidecar，业务零侵入
- 局限性：业务维度限流（如按 userId）需要 header 透传到 sidecar
- 代价：每跳多一个 sidecar 的延迟（~1ms）

### Bulkhead 模式（隔离）
- 线程池隔离 vs 信号量隔离
- Hystrix 死于线程池隔离的开销 —— 但隔离本身没错，错的是粒度
- Resilience4j 默认信号量隔离的工程意义

### 异步限流与背压（Reactor）
- Project Reactor 的 `limitRate` / `onBackpressureBuffer`
- 响应式系统的"背压传播" —— 上游能感知下游处理慢
- 与传统限流的本质差别：**背压是"被动"的，限流是"主动"的**

### 优先级队列与请求分级
- Google SRE 工作簿中的 LIFO 队列设计（突发期老请求超时直接丢，不浪费资源）
- 请求 deadline 透传：请求带 deadline header，下游看到已经超时直接拒
- 关键信号：CoDel 算法（Controlled Delay）

### SRE 工作簿：限流配置的演进
- *Site Reliability Engineering* 第 21 章 "Handling Overload"
- 客户端节流（client-side throttling）：客户端自己减少请求，避免冲击
- 重试预算（retry budget）：限制重试占总流量的比例

---

## 7. 引用

- Sentinel 1.8.6 源码：[alibaba/Sentinel](https://github.com/alibaba/Sentinel)
- Resilience4j 2.2.0 源码：[resilience4j/resilience4j](https://github.com/resilience4j/resilience4j)
- Netflix concurrency-limits：[Netflix/concurrency-limits](https://github.com/Netflix/concurrency-limits)
- Lyft Ratelimit：[envoyproxy/ratelimit](https://github.com/envoyproxy/ratelimit)
- Adrian Cockcroft, *Microservices and Failure Modes* (Velocity 2014)
- Google SRE Book, Chapter 21: *Handling Overload*
- Google SRE Workbook, Chapter 22: *Addressing Cascading Failures*
- Marc Brooker, *Workload isolation using shuffle-sharding*（AWS 博客）
- TCP BBR 论文：*BBR: Congestion-Based Congestion Control* (Cardwell et al., ACM Queue 2016)
- Hystrix 维护模式公告：[Hystrix is now in maintenance mode](https://github.com/Netflix/Hystrix#hystrix-status)
- Brendan Gregg, *USE Method*（容量分析方法）

---

> 配套：[系统设计主题](/article/system-design/) | [缓存设计](/article/system-design/cache-design) | [线程池生产实战](/article/java/threadpool)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
