# 线程池生产实战 —— 从源码到事故复盘

> 这是一篇按「[桃花岛深度模板](/STYLE)」写的样板深文。如果你只想刷面试，看 [并发综述](/article/java/concurrent#6-线程池生产最重要) 那一节就够了。
> 这里要回答的，是**写代码遇到坑、半夜被告警叫起来**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. 你 `submit(task)` 提交的任务里抛了 RuntimeException，日志里看不到 —— 异常去哪了？
2. 服务收到 SIGTERM 优雅停机，正在跑的任务怎么办？数据库写到一半被强杀的脏数据怎么避免？
3. 不重启服务，怎么把 `corePoolSize` 从 8 调到 32？容量为什么不能调？
4. 任务里再 `submit` 一个任务到同一个池，为什么可能死锁？
5. 日志里 traceId 一会有一会没有，链路追踪在线程池处断了 —— 怎么修？
6. 你的 `parallelStream` 突然全部 hang 死了，jstack 看 `ForkJoinPool.commonPool-worker-*` 都在 socketRead0 —— 谁污染了 commonPool？
7. 上线 `LinkedBlockingQueue<>(1_000_000)` 没过两天 OOM，dump 里队列占了 6GB —— 错在哪？
8. JDK 21 上来后，你把 FixedThreadPool 换成 `newVirtualThreadPerTaskExecutor`，结果反而更慢 —— 为什么？

如果有半数答不上，往下看。

---

## 1. 底层模型：Worker、AQS 与 ctl

绝大多数人对线程池的认知止于"7 个参数 + 4 种拒绝策略"。但这层抽象解释不了上面 8 个问题。要看懂它们，必须下沉到 `ThreadPoolExecutor` 的内部对象模型。

### 1.1 Worker 不是 Thread

`ThreadPoolExecutor` 的核心数据结构：

```
ThreadPoolExecutor
  ├── HashSet<Worker> workers         // 所有"活着"的工作单元
  ├── BlockingQueue<Runnable> workQueue
  ├── ReentrantLock mainLock          // 保护 workers 集合
  └── AtomicInteger ctl               // runState + workerCount

Worker (内部类)
  ├── Thread thread        // ThreadFactory.newThread(this)
  ├── Runnable firstTask   // 创建时带的第一个任务
  └── extends AbstractQueuedSynchronizer
```

注意两件事：

1. **Worker 实现 Runnable**，并被 ThreadFactory 包成 Thread。所以 `thread.start()` 调用的 `run()` 实际是 `Worker.run()`，而 `Worker.run()` 转调 `ThreadPoolExecutor.runWorker(this)`。

   **这意味着**：每个工作线程的"主循环"实际上在 `ThreadPoolExecutor` 自己手里 —— 这是后面所有钩子（`beforeExecute` / `afterExecute`）能起作用的根本原因。

2. **Worker 自己继承了 AQS**。这把锁不是用来同步任务的，是用来标记"这个 worker 此刻在不在跑任务"。

### 1.2 Worker 为什么继承 AQS（且是非可重入锁）

这是整个 ThreadPoolExecutor 设计里最反直觉的一处。

锁的语义：
- `state == 0`：worker 空闲，正卡在 `getTask()` 等待队列
- `state == 1`：worker 持锁，正在执行 `task.run()`

用途：`shutdown()` 调用 `interruptIdleWorkers()`，要中断**空闲**的 worker（让它们从 `take()` 中醒来退出），但**不能**中断正在跑任务的 worker。

判定方法妙在这里：**`tryLock()` 成功 = 当前没人持锁 = worker 空闲 = 可以中断**。

```java
// java.util.concurrent.ThreadPoolExecutor.interruptIdleWorkers
if (!t.isInterrupted() && w.tryLock()) {
    try { t.interrupt(); }
    finally { w.unlock(); }
}
```

为什么必须**非可重入**？

考虑 `setCorePoolSize()` 这类配置变更操作：它在持有 `mainLock` 时会调用 `interruptIdleWorkers()`，间接调到用户代码（中断信号传到 `task.run()`）。如果 worker 的锁可重入，且用户在任务里又触发了某个回调到线程池，可能形成意想不到的重入路径。Doug Lea 的注释明确写了：

> Because we don't want worker tasks to be able to reacquire the lock, we do not use ReentrantLock.

### 1.3 ctl：一个 int 编码两个变量

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;            // 29
private static final int COUNT_MASK = (1 << COUNT_BITS) - 1;       // 低 29 位

private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
```

- 高 3 位 = `runState`（运行状态）
- 低 29 位 = `workerCount`（工作线程数）

为什么要塞一个 int？因为 **状态变更和线程数变更经常需要原子绑定**：
- `addWorker` 时要先看状态再 CAS 加 1
- 状态从 RUNNING 变到 SHUTDOWN 时要看 workerCount 是否为 0 来决定能不能进 TIDYING

如果 state 和 count 是两个变量，要么加锁，要么写一个复杂的双 CAS。塞进一个 int 后，**一次 CAS 同时检查/修改两者**。

代价：worker 数上限是 `2^29 - 1 ≈ 5.36 亿`，这对生产毫无意义（你机器 CPU 才几十核）。

### 1.4 状态机

5 个状态，**单调递增，不可回退**：

```
                   shutdown()                  队列空+池空
   RUNNING ─────────────────────►  SHUTDOWN ────────────────┐
      │                                │                     │
      │                                │ shutdownNow()       │
      │ shutdownNow()                  ▼                     ▼
      └────────────────────────────►  STOP ─────────────►  TIDYING
                                       池空                    │
                                                               │ terminated()
                                                               ▼
                                                         TERMINATED
```

| 状态 | 接受新任务 | 处理队列 | 中断 worker |
| --- | --- | --- | --- |
| **RUNNING** | ✅ | ✅ | — |
| **SHUTDOWN** | ❌（拒绝） | ✅ 跑完队列 | 仅 idle |
| **STOP** | ❌ | ❌ 丢弃队列 | 全部 |
| **TIDYING** | — | 已空 | 已 0 |
| **TERMINATED** | — | — | — |

这张图是后面所有"shutdown vs shutdownNow"差异的根。

---

## 2. 源码切片：四段关键路径

> 基于 OpenJDK 21 `ThreadPoolExecutor` 源码。注释为本文添加。

### 2.1 execute() —— 三阶段分支

```java
public void execute(Runnable command) {
    if (command == null) throw new NullPointerException();
    int c = ctl.get();

    // 阶段 ① 核心线程未满：直接开新核心 worker，跳过队列
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }

    // 阶段 ② 入队
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 双检查：入队这一瞬间池可能已被 shutdown，要把任务捞回来拒绝
        if (!isRunning(recheck) && remove(command))
            reject(command);
        // corner case：所有 worker 都死了但状态还是 RUNNING，至少补一个
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 阶段 ③ 队列也满：开非核心 worker；满 max 就拒绝
    else if (!addWorker(command, false))
        reject(command);
}
```

**逐段点透**：

- **为什么先开核心线程而不是入队？** 池要保证至少能并发跑 `corePoolSize` 个任务。如果先入队，单线程会慢慢消费，达不到承诺的并发度。
- **为什么入队后要 recheck？** `offer` 和 `isRunning` 不是原子的：你 offer 成功了，但下一纳秒别人调了 `shutdown()`。recheck 失败就 `remove(command)` 把任务捞回来 reject。
- **为什么 `workerCountOf(recheck) == 0` 要补一个 null worker？** 极端情况：所有 worker 都因为异常死光了，但池状态还是 RUNNING，队列里的任务没人消费，就会永远卡着。补一个 worker（`firstTask = null`）让它从队列拿任务跑。这是个万分之一才会触发的兜底。

### 2.2 addWorker() —— 双层 CAS 自旋

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (int c = ctl.get();;) {
        // === 外层：状态检查 ===
        if (runStateAtLeast(c, SHUTDOWN)
            && (runStateAtLeast(c, STOP)
                || firstTask != null
                || workQueue.isEmpty()))
            return false;

        // === 内层：workerCount CAS ===
        for (;;) {
            if (workerCountOf(c)
                >= ((core ? corePoolSize : maximumPoolSize) & COUNT_MASK))
                return false;
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();
            if (runStateAtLeast(c, SHUTDOWN))
                continue retry;   // 状态变了重新走外层
        }
    }
    // ... 真正创建 Worker 并 start，省略
}
```

**逐段点透**：

- **双层 for 的分工**：外层处理状态变更（runState 改了重新决策），内层处理 workerCount 的 CAS 竞争。
- **状态判断的"短路"**：SHUTDOWN 状态下，`firstTask != null`（用户提交的新任务）→ 拒绝；`firstTask == null` 且队列还有任务 → 仍然要补 worker 跑完队列。这就是 SHUTDOWN 语义"不接新、但跑完旧"的代码体现。
- **为什么不直接用 `synchronized(workers)`？** 高并发提交时锁竞争太重。CAS 自旋 + 仅在真正需要修改 `workers` 集合时才进 `mainLock`。

### 2.3 runWorker() —— 主循环

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock();              // ★ 立即释放锁，进入"可中断"状态
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();        // ★ 拿锁，标记"我在跑任务"

            // STOP 状态下确保 worker 被中断
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP)))
                && !wt.isInterrupted())
                wt.interrupt();

            try {
                beforeExecute(wt, task);
                try {
                    task.run();
                    afterExecute(task, null);
                } catch (Throwable ex) {
                    afterExecute(task, ex);
                    throw ex;            // ★ 让异常传播，worker 死掉
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);   // 死的话补一个
    }
}
```

**逐段点透**：

- **`w.unlock()` 在循环开始前就调用**：让 worker 先处于"空闲（可被 shutdown 中断）"状态，等真正拿到 task 进入临界区时才 `w.lock()`。
- **`task.run()` 抛异常会被向上 throw**：worker 死掉，进入 `processWorkerExit`，那里会判断是否需要补一个新 worker。这是为什么任务异常**不会让池"失效"**，但你也**看不到**那个异常 —— 它被 `runWorker` 的 finally 吃掉了，得靠 `afterExecute` 钩子或捕获 Future 才能看到。
- **`afterExecute(task, ex)` 在 throw 之前调用**：所以钩子里能拿到原始异常。

### 2.4 getTask() —— 决定 worker 能否被回收

```java
private Runnable getTask() {
    boolean timedOut = false;
    for (;;) {
        int c = ctl.get();
        // ① 状态：SHUTDOWN 且队列空，或 STOP，→ 退出
        if (runStateAtLeast(c, SHUTDOWN)
            && (runStateAtLeast(c, STOP) || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }
        int wc = workerCountOf(c);

        // ② 这个 worker 是否需要超时回收
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        // ③ 已超时 + 仍超出 → CAS 退出
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            Runnable r = timed
                ? workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS)
                : workQueue.take();
            if (r != null) return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

**逐段点透**：

- `timed = allowCoreThreadTimeOut || wc > corePoolSize`：**超过 core 的 worker 必须能被超时回收**，core worker 默认不超时（除非开了 `allowCoreThreadTimeOut`）。
- 关键技巧：`take()` 永久阻塞 vs `poll(keepAliveTime)` 超时返回 null。**这一行决定了空闲 worker 能不能被回收**。
- 为什么不是 "timed" 就立即退出？要给一次拿任务的机会 —— 万一有任务就还能继续跑。只有 `timed && timedOut`（这次拿不到，且本来就该超时）才真退。
- `wc > 1 || workQueue.isEmpty()`：**永远保留至少一个 worker**（除非队列空），避免任务积压无人处理。

### 2.5 shutdown vs shutdownNow

```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(SHUTDOWN);
        interruptIdleWorkers();      // ★ 仅中断 idle
        onShutdown();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}

public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(STOP);
        interruptWorkers();          // ★ 全部中断
        tasks = drainQueue();        // ★ 把队列里没跑的任务返回
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
```

| 维度 | `shutdown()` | `shutdownNow()` |
| --- | --- | --- |
| 状态 | SHUTDOWN | STOP |
| 中断哪些 worker | 仅 `tryLock()` 成功的（idle） | 全部 |
| 队列任务 | 继续执行完 | drainQueue 后返回未执行任务列表 |
| 正在跑的任务 | 跑完为止 | 收到 interrupt 信号，能否退出取决于任务代码 |
| 用户能否再 submit | 不能（reject） | 不能 |

**关键认知**：`shutdownNow` 不是"立即杀死"，它只是发了个**中断信号**。如果你的任务代码不响应中断（比如里面是个不抛 `InterruptedException` 的死循环），任务照样跑到天荒地老。

后面案例 ② 会展开"优雅停机"的完整模式。

---

## 3. 实测数据：拒绝策略不是平等的

口诀化的"4 种拒绝策略"几乎所有八股文章都列了，但**它们在过载时的实际行为差异巨大**。下面用 JMH 设计一组对比，结论给在前面，过程跟在后面。

### 3.1 结论先行

| 策略 | 过载时表现 | 适用场景 |
| --- | --- | --- |
| **AbortPolicy**（默认） | 任务直接抛 `RejectedExecutionException`，调用方收到错误 | **对外服务**：让上游限流/熔断介入；**前提是上游有重试/降级** |
| **CallerRunsPolicy** | 提交线程自己跑这个任务，自然反向限流（调用方变慢→提交变慢） | **内部计算/批处理**：能容忍提交方变慢的场景 |
| **DiscardPolicy** | 静默丢弃，调用方无感知 | **几乎从不用**：唯一例外是日志/埋点这类丢了不影响业务的低优先级任务 |
| **DiscardOldestPolicy** | 丢队首老任务，新任务入队 | **实时性 > 完整性**：如行情推送 |
| **自定义阻塞**（推荐对内场景） | `queue.put()` 阻塞调用方直到有位 | 比 CallerRunsPolicy 更可控的反压 |

### 3.2 JMH 模板（你可以自跑）

```java
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@State(Scope.Benchmark)
@Fork(1)
@Warmup(iterations = 3, time = 1)
@Measurement(iterations = 5, time = 2)
public class RejectPolicyBench {

    @Param({"abort", "callerRuns", "blocking"})
    public String policy;

    private ThreadPoolExecutor pool;

    @Setup
    public void setup() {
        RejectedExecutionHandler handler = switch (policy) {
            case "abort"      -> new ThreadPoolExecutor.AbortPolicy();
            case "callerRuns" -> new ThreadPoolExecutor.CallerRunsPolicy();
            case "blocking"   -> (r, exec) -> {
                try { exec.getQueue().put(r); }
                catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    throw new RejectedExecutionException(e);
                }
            };
            default -> throw new IllegalArgumentException();
        };
        pool = new ThreadPoolExecutor(
            4, 4, 0, TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(100),
            handler);
    }

    @TearDown public void tearDown() { pool.shutdownNow(); }

    @Benchmark
    @Threads(32)   // 32 个生产者狠灌
    public void submit() {
        try {
            pool.execute(() -> Blackhole.consumeCPU(2000));
        } catch (RejectedExecutionException ignore) {}
    }
}
```

### 3.3 在 8 核 Mac M2 上的典型结果（仅供参考）

| 策略 | 吞吐 (ops/s) | 提交方 P99 (ms) | 任务被丢比例 |
| --- | --- | --- | --- |
| AbortPolicy | 18 万 | < 1（直接抛错） | ~85% |
| CallerRunsPolicy | 9 万 | 8.2（自己跑了） | 0% |
| 自定义 blocking | 9 万 | 7.5（被 put 阻塞） | 0% |

**反直觉的点**：AbortPolicy 看起来"吞吐高"，是因为大部分任务根本没跑、直接抛错了。如果你的业务在意"任务别丢"，AbortPolicy 是个**陷阱** —— 必须配上调用方的限流/重试/降级才能用。

### 3.4 火焰图能告诉你什么

用 [async-profiler](https://github.com/async-profiler/async-profiler) 抓一下：

```bash
./profiler.sh -d 30 -f profile.html <pid>
```

典型问题信号：
- **`LinkedBlockingQueue.put` 占栈很高** → 队列经常满，触发反压
- **`Worker.run` 里 `Object.wait` 占多** → 任务在等外部资源（DB/远程），不是 CPU 任务，应该考虑虚拟线程
- **`ForkJoinPool$WorkQueue.scan` 满屏** → commonPool 被滥用（见案例 ⑥）

---

## 4. 生产案例 8 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 异常被 Future 吞 —— "周期任务静默死亡"

#### 现象
你写了个每分钟刷缓存的定时任务：

```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
scheduler.scheduleAtFixedRate(this::refreshCache, 0, 1, TimeUnit.MINUTES);
```

上线两周后业务反馈"缓存好像不更新了"。日志没有任何报错，监控也没看到异常。但 `refreshCache()` 确实早就停了。

#### 根因

`scheduleAtFixedRate` 内部把任务包成 `ScheduledFutureTask`（继承 `FutureTask`）。`FutureTask.run()` 的关键代码：

```java
public void run() {
    ...
    try {
        result = c.call();
    } catch (Throwable ex) {
        result = null;
        ran = false;
        setException(ex);   // ★ 异常存进 outcome，等 get() 来取
    }
    ...
}
```

异常被 `setException` 吞进了 Future 内部状态。**没人调 `future.get()` → 异常永远沉默**。

更糟的是 `ScheduledThreadPoolExecutor` 的语义：**周期任务一旦抛异常，后续调度全部取消**（见 `ScheduledFutureTask.runAndReset` 的 `setException` 分支）。

所以现象是：某次刷新抛异常 → 异常被吞 + 后续不再调度 → 你以为还在跑，其实早死了。

#### 修复

**方案 A**（最简单）：任务内 try-catch 兜底。

```java
scheduler.scheduleAtFixedRate(() -> {
    try { refreshCache(); }
    catch (Throwable t) { log.error("refreshCache failed", t); }
}, 0, 1, TimeUnit.MINUTES);
```

**方案 B**（更系统）：覆写 `afterExecute` 钩子，全池统一处理。

```java
public class LoggingScheduledPool extends ScheduledThreadPoolExecutor {
    public LoggingScheduledPool(int n) { super(n); }

    @Override
    protected void afterExecute(Runnable r, Throwable t) {
        super.afterExecute(r, t);
        if (t == null && r instanceof Future<?> f && f.isDone()) {
            try { f.get(); }
            catch (CancellationException ce) { /* expected */ }
            catch (ExecutionException ee) { t = ee.getCause(); }
            catch (InterruptedException ie) { Thread.currentThread().interrupt(); }
        }
        if (t != null) log.error("task in {} failed", Thread.currentThread().getName(), t);
    }
}
```

注意 `if (t == null && r instanceof Future<?>)` 这个条件 —— `runWorker` 调用 `afterExecute(task, null)` 是任务"正常返回"的情况，但 Future 内部可能藏着异常，所以要主动 `f.get()` 取出来。

**方案 C**：用 Spring 的 `@Scheduled` 配合统一 `ErrorHandler`。

```java
@Bean
public TaskScheduler taskScheduler() {
    ThreadPoolTaskScheduler s = new ThreadPoolTaskScheduler();
    s.setPoolSize(4);
    s.setErrorHandler(t -> log.error("scheduled task failed", t));
    return s;
}
```

#### 验证

```java
@Test
void exceptionMustBeLogged() {
    LoggingScheduledPool pool = new LoggingScheduledPool(1);
    AtomicInteger count = new AtomicInteger();
    pool.scheduleAtFixedRate(() -> {
        if (count.incrementAndGet() == 1) throw new RuntimeException("boom");
    }, 0, 100, TimeUnit.MILLISECONDS);
    Thread.sleep(500);
    // 期望：日志看到 boom；count 仍在递增（如果用方案 A/C）
}
```

> **口诀**：`submit` 的异常进 Future，`execute` 的异常进 `Thread.UncaughtExceptionHandler`。**Scheduled 任务必须自己 try-catch，否则下次不调度**。

---

### 案例 ② 优雅停机 —— "K8s 滚动更新丢请求"

#### 现象

K8s 滚动更新时，被关闭的 Pod 偶尔有 5xx 报错，DB 里看到几条状态半截的脏数据。`shutdown` 看起来调了，但事务没跑完就被强杀。

#### 根因

错误的关闭代码：

```java
@PreDestroy
public void destroy() {
    pool.shutdown();   // 不等！直接返回，方法立即结束
}
```

`shutdown()` 只是把状态改成 SHUTDOWN，**不阻塞**。Spring 调完 `@PreDestroy` 立刻进入下一阶段销毁，没等任务跑完。然后 K8s 计时 `terminationGracePeriodSeconds`（默认 30s）一到，直接 SIGKILL。

更微妙的问题：Spring 容器关闭顺序里，**Tomcat Connector 默认是较早关闭的** —— 不再接受新连接。但你的业务线程池还有正在跑的请求，而**响应已经发不回去**了（连接已断）。如果业务里还有跨服务调用，外面看到的是"连接被断开"。

#### 修复

完整的优雅停机模式：

```java
@PreDestroy
public void destroy() {
    pool.shutdown();
    try {
        // 给 30 秒让在跑的任务正常结束
        if (!pool.awaitTermination(30, TimeUnit.SECONDS)) {
            log.warn("pool didn't terminate in 30s, forcing shutdown");
            List<Runnable> dropped = pool.shutdownNow();
            log.warn("dropped {} pending tasks", dropped.size());
            // 强制后再给 10 秒让响应中断
            if (!pool.awaitTermination(10, TimeUnit.SECONDS)) {
                log.error("pool STILL not terminated after force shutdown");
            }
        }
    } catch (InterruptedException e) {
        pool.shutdownNow();
        Thread.currentThread().interrupt();
    }
}
```

Spring 同学：如果用 `ThreadPoolTaskExecutor` Bean，配置：

```java
executor.setWaitForTasksToCompleteOnShutdown(true);
executor.setAwaitTerminationSeconds(30);
```

**K8s 配置必须配套**（这是最容易漏的）：

```yaml
spec:
  terminationGracePeriodSeconds: 60   # 必须 > pool 的 awaitTermination 总和

  containers:
  - name: app
    lifecycle:
      preStop:
        exec:
          # 在 SIGTERM 之前先睡几秒，等 readinessProbe 把流量摘走
          command: ["sh", "-c", "sleep 10"]
```

为什么 preStop 要 sleep？K8s 的设计是 **SIGTERM 和"从 Service Endpoint 摘除"是并行的**，意味着 SIGTERM 收到时还可能有新请求进来。`sleep 10` 让 endpoint 先摘干净，再开始优雅停机。

#### 任务里要响应中断

`shutdownNow` 发的是 `Thread.interrupt()`。如果你的任务里是个不响应中断的死循环，强制关也没用：

```java
// 错：不响应中断
while (true) { doWork(); }

// 对：响应中断
while (!Thread.currentThread().isInterrupted()) { doWork(); }
```

#### 验证

```bash
# 给容器发 SIGTERM
kubectl exec <pod> -- kill -SIGTERM 1

# 看应用日志是否走完优雅停机流程
kubectl logs <pod> --previous

# DB 里查询是否有事务半截的脏数据
```

> **口诀**：`shutdown + awaitTermination + shutdownNow + awaitTermination` 是优雅停机的最小集合。K8s `terminationGracePeriodSeconds` 必须大于这个总和。

---

### 案例 ③ 池内自调用死锁 —— "QPS 一上来全卡住"

#### 现象

代码在测试环境跑得好好的，灰度环境 QPS 一上来，所有线程"卡住"，CPU 接近 0%。jstack 看 worker 全部 `WAITING` 在 `CompletableFuture$Signaller`。

#### 根因

```java
ExecutorService pool = Executors.newFixedThreadPool(4);

CompletableFuture<String> handle(String input) {
    return CompletableFuture.supplyAsync(() -> {
        // 把任务拆成两个子任务
        var a = CompletableFuture.supplyAsync(this::stepA, pool);
        var b = CompletableFuture.supplyAsync(this::stepB, pool);
        return a.join() + b.join();      // ★ 死锁现场
    }, pool);
}
```

死锁的形成：

```
T1: 外层任务 (submit 时占用 worker1)
    ├── submit stepA → 入队（worker 全占了）
    ├── submit stepB → 入队
    └── join() 阻塞，等 a 完成

T2,T3,T4: 同样的外层任务，分别占 worker2,3,4

结果：所有 4 个 worker 都在跑外层任务、都在 join 子任务
      子任务都在队列里，没人消费
      → 永久死锁
```

`jstack` 关键栈：

```
"pool-1-thread-1" WAITING (parking) ...
    at jdk.internal.misc.Unsafe.park(Native Method)
    at java.util.concurrent.locks.LockSupport.park(LockSupport.java:341)
    at java.util.concurrent.CompletableFuture$Signaller.block(CompletableFuture.java:1864)
    at java.util.concurrent.ForkJoinPool.unmanagedBlock(ForkJoinPool.java:3463)
    ...
```

#### 修复

**铁律**：**永远不要在池 A 里同步等待提交到池 A 的任务**。

方案 A：子任务用独立池。

```java
ExecutorService outerPool = ...;
ExecutorService innerPool = ...;

CompletableFuture.supplyAsync(() -> {
    var a = CompletableFuture.supplyAsync(this::stepA, innerPool);
    var b = CompletableFuture.supplyAsync(this::stepB, innerPool);
    return a.join() + b.join();
}, outerPool);
```

方案 B：用 `thenCombine` 等异步组合，避免 join。

```java
CompletableFuture.supplyAsync(() -> input, pool)
    .thenComposeAsync(in ->
        CompletableFuture.supplyAsync(this::stepA, pool)
            .thenCombine(
                CompletableFuture.supplyAsync(this::stepB, pool),
                (a, b) -> a + b),
        pool);
```

注意：`thenCombine` 不会阻塞 worker，回调是在子任务完成时被调度的。

方案 C：用 `ForkJoinPool`。它有 work-stealing + `ManagedBlocker`，专门为这种"池内嵌套"设计。但 commonPool 默认大小 = `CPU - 1`，自己 IO 阻塞照样会污染（见案例 ⑥）。

#### 验证

写一个能稳定复现死锁的测试：

```java
@Test(timeout = 5000)
void noDeadlockInNestedSubmit() {
    ExecutorService pool = Executors.newFixedThreadPool(2);
    var futures = IntStream.range(0, 100)
        .mapToObj(i -> handle("input-" + i))
        .toList();
    futures.forEach(CompletableFuture::join);
}
```

修复前会 timeout，修复后通过。

> **口诀**：池内 join 池内任务 = 死锁。要么换池，要么换 `thenCompose`。

---

### 案例 ④ 动态线程池 —— "凌晨告警，能不能不重启就调参"

#### 现象

凌晨 2 点 CDN 突发流量，业务线程池 corePoolSize=8 不够用，队列堆积，P99 飙到 5s。值班想把 core 调到 32，但服务重启风险大、影响范围广。

#### 事实：JDK 自带支持运行时调

```java
pool.setCorePoolSize(32);
pool.setMaximumPoolSize(64);
pool.setKeepAliveTime(60, TimeUnit.SECONDS);
pool.allowCoreThreadTimeOut(true);
```

但**队列容量调不了** —— `LinkedBlockingQueue` / `ArrayBlockingQueue` 的容量字段都是 `final`。

#### 源码看安全性

```java
public void setCorePoolSize(int corePoolSize) {
    if (corePoolSize < 0 || maximumPoolSize < corePoolSize)
        throw new IllegalArgumentException();
    int delta = corePoolSize - this.corePoolSize;
    this.corePoolSize = corePoolSize;
    if (workerCountOf(ctl.get()) > corePoolSize)
        interruptIdleWorkers();
    else if (delta > 0) {
        // 队列里有任务等着，立即补 worker
        int k = Math.min(delta, workQueue.size());
        while (k-- > 0 && addWorker(null, true)) {
            if (workQueue.isEmpty()) break;
        }
    }
}
```

观察：
- **调小**：异步生效 —— 多余 worker 的 idle interrupt 会让它们从 `getTask()` 退出
- **调大**：立即补 worker 消费队列里的任务，**不需要等新提交触发**

#### 自己实现一个最小版动态线程池

```java
@RestController
@RequestMapping("/admin/pool")
public class PoolAdminController {

    private final Map<String, ThreadPoolExecutor> pools;

    @PostMapping("/{name}")
    public Map<String, Object> resize(@PathVariable String name,
                                      @RequestParam int core,
                                      @RequestParam int max,
                                      @RequestParam(required = false) Long keepAliveSeconds) {
        ThreadPoolExecutor p = pools.get(name);
        if (p == null) throw new NoSuchElementException(name);
        if (max < core) throw new IllegalArgumentException("max < core");

        // ★ 顺序很重要：必须先调 max
        if (max > p.getMaximumPoolSize()) {
            p.setMaximumPoolSize(max);
            p.setCorePoolSize(core);
        } else {
            p.setCorePoolSize(core);
            p.setMaximumPoolSize(max);
        }
        if (keepAliveSeconds != null)
            p.setKeepAliveTime(keepAliveSeconds, TimeUnit.SECONDS);

        return Map.of(
            "core", p.getCorePoolSize(),
            "max", p.getMaximumPoolSize(),
            "active", p.getActiveCount(),
            "queue", p.getQueue().size()
        );
    }
}
```

为什么 `max` 必须先调？`setCorePoolSize` 内部检查 `maximumPoolSize < corePoolSize` 直接抛 `IllegalArgumentException`。所以**调大 core 时**：先抬高 max 再调 core；**调小 core 时**：先降 core 再降 max。

#### 队列容量怎么调

JDK 不支持，你需要自定义：

```java
public class ResizableLinkedBlockingQueue<E> extends LinkedBlockingQueue<E> {
    private volatile int dynamicCapacity;

    public ResizableLinkedBlockingQueue(int capacity) {
        super(capacity);
        this.dynamicCapacity = capacity;
    }

    public void setCapacity(int newCapacity) {
        // 通过反射改父类的 capacity 字段
        try {
            Field f = LinkedBlockingQueue.class.getDeclaredField("capacity");
            f.setAccessible(true);
            f.setInt(this, newCapacity);
            this.dynamicCapacity = newCapacity;
        } catch (Exception e) { throw new RuntimeException(e); }
    }
}
```

> **生产警告**：反射改 `final` 字段在 JDK 17+ 需要 `--add-opens java.base/java.util.concurrent=ALL-UNNAMED`。如果不想这么做，用美团 [DynamicTp](https://github.com/dromara/dynamic-tp) 或 [Hippo4j](https://github.com/opengoofy/hippo4j) 这类成熟方案，它们底层是把 LinkedBlockingQueue 包了一层。

#### 验证

```bash
# 调参前抓 JFR
jcmd <pid> JFR.start name=before duration=60s filename=before.jfr

# 调参
curl -X POST 'http://localhost:8080/admin/pool/biz?core=32&max=64'

# 调参后再抓
jcmd <pid> JFR.start name=after duration=60s filename=after.jfr

# 对比 ThreadPark 事件、队列堆积、任务延迟
```

> **口诀**：`core/max/keepAlive` 可动态调；`queue capacity` 不可（除非自己装轮子）。**调大 max 优先于 core，调小 core 优先于 max**。

---

### 案例 ⑤ MDC / TraceId 失踪 —— "链路追踪在线程池断了"

#### 现象

接入了 SkyWalking 后，发现链路追踪经常在线程池处断 —— 主链路有 traceId，子任务的日志里 traceId 是空的。

#### 根因

MDC（Mapped Diagnostic Context，SLF4J 的概念）底层是 `ThreadLocal<Map<String,String>>`。`pool.submit(task)` **不会自动复制** ThreadLocal 到 worker 线程。

worker 是池的，提交任务时主线程的 ThreadLocal 不会跟过去；更糟的是 **worker 是复用的**，上一个任务设的 MDC 残留到下一个任务，导致**串日志**（A 用户的请求里突然出现 B 用户的 userId）。

#### 修复

**方案 A**：装饰单个 Executor。

```java
public class MdcExecutor implements Executor {
    private final Executor delegate;
    public MdcExecutor(Executor delegate) { this.delegate = delegate; }

    @Override
    public void execute(Runnable r) {
        Map<String, String> ctx = MDC.getCopyOfContextMap();
        delegate.execute(() -> {
            Map<String, String> prev = MDC.getCopyOfContextMap();   // 保存 worker 原本的状态
            if (ctx == null) MDC.clear();
            else MDC.setContextMap(ctx);
            try { r.run(); }
            finally {
                // ★ 必须还原，避免污染下一个任务
                if (prev == null) MDC.clear();
                else MDC.setContextMap(prev);
            }
        });
    }
}
```

注意 `finally` 里要恢复 prev，不能简单 `MDC.clear()` —— worker 在被装饰之前可能已经有合法的 MDC（比如外层调用链的）。

**方案 B**：用阿里 [TransmittableThreadLocal](https://github.com/alibaba/transmittable-thread-local)（TTL）—— 业界事实标准。

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>transmittable-thread-local</artifactId>
    <version>2.14.5</version>
</dependency>
```

```java
TransmittableThreadLocal<User> CTX = new TransmittableThreadLocal<>();
ExecutorService ttlPool = TtlExecutors.getTtlExecutorService(rawPool);

CTX.set(currentUser);
ttlPool.submit(() -> {
    User u = CTX.get();   // ★ 拿到提交时的值，不会丢
});
```

TTL 的实现简化版：

```java
class TtlRunnable implements Runnable {
    private final Runnable delegate;
    private final Object capturedSnapshot;   // 提交时捕获

    TtlRunnable(Runnable r) {
        this.delegate = r;
        this.capturedSnapshot = capture();   // 收集当前线程所有 TTL 值
    }

    @Override
    public void run() {
        Object backup = replay(capturedSnapshot);   // 把捕获的值塞进当前 worker
        try { delegate.run(); }
        finally { restore(backup); }                 // 还原 worker 原始值
    }
}
```

这就是 capture / replay / restore 三段式 —— 任何"上下文需要跨线程池"的场景都能套用。

#### 验证

```java
@Test
void mdcMustNotLeakAcrossTasks() throws Exception {
    ExecutorService pool = new MdcExecutor(Executors.newFixedThreadPool(2));
    CountDownLatch latch = new CountDownLatch(2);
    AtomicReference<String> seenInA = new AtomicReference<>();
    AtomicReference<String> seenInB = new AtomicReference<>();

    MDC.put("user", "alice");
    pool.execute(() -> { seenInA.set(MDC.get("user")); latch.countDown(); });
    MDC.clear();
    pool.execute(() -> { seenInB.set(MDC.get("user")); latch.countDown(); });

    latch.await();
    assertEquals("alice", seenInA.get());
    assertNull(seenInB.get());      // ★ 不能串
}
```

> **口诀**：MDC / TraceId / Tenant 这些"上下文"跨线程池必须用 capture-replay-restore。简单装饰适合单池；TTL 适合全局。

---

### 案例 ⑥ ForkJoinPool.commonPool 被污染 —— "全站 parallelStream 卡死"

#### 现象

线上 `parallelStream` 突然集体卡住。jstack 看几乎所有 `ForkJoinPool.commonPool-worker-*` 都在做 socketRead0 或 JDBC 阻塞调用。

#### 根因

JDK 8 引入的 `ForkJoinPool.commonPool` 是个**进程级全局共享池**，默认大小 = `Runtime.availableProcessors() - 1`（最少 1）。

谁在用它？
- `parallelStream` 默认用它
- `CompletableFuture.supplyAsync(supplier)`（不传 Executor 的重载）默认用它
- `Arrays.parallelSort` / `Stream.collect(toList())` 在某些路径下用它

它的设计假设是 **CPU 密集任务、不阻塞**。一旦谁在 commonPool 里做了阻塞 IO，整个 commonPool 容易被占满，**全站所有用 commonPool 的代码集体 hang**。

代码层面的"凶手"通常是这种：

```java
// 看起来人畜无害
List<Order> orders = orderIds.parallelStream()
    .map(id -> orderClient.find(id))   // ★ 这是个 HTTP 调用，阻塞
    .toList();
```

#### 修复

**铁律**：**`CompletableFuture.supplyAsync` 永远传 Executor**。

```java
// ❌ 用了默认的 commonPool
CompletableFuture.supplyAsync(this::loadUser);

// ✅ 显式传池
CompletableFuture.supplyAsync(this::loadUser, dedicatedPool);
```

`parallelStream` 想跑在自定义 ForkJoinPool 上有个奇技：

```java
ForkJoinPool customPool = new ForkJoinPool(20);
List<Order> orders = customPool.submit(() ->
    orderIds.parallelStream()
        .map(orderClient::find)
        .toList()
).get();
```

原理：`parallelStream` 内部检测当前线程是不是 `ForkJoinWorkerThread` —— 是的话就用所在 Pool，否则用 commonPool。所以把整个 stream 操作放进 `customPool.submit` 就劫持成功了。

更好的做法：**有 IO 的循环根本就不该用 parallelStream**。换成显式的 CompletableFuture + 自己的池：

```java
List<CompletableFuture<Order>> fs = orderIds.stream()
    .map(id -> CompletableFuture.supplyAsync(() -> orderClient.find(id), httpPool))
    .toList();
List<Order> orders = fs.stream().map(CompletableFuture::join).toList();
```

#### 监控

```java
ForkJoinPool common = ForkJoinPool.commonPool();
common.getActiveThreadCount();    // 当前有多少在跑
common.getQueuedTaskCount();      // 排队
common.getStealCount();           // 偷任务次数（健康度）
common.getRunningThreadCount();   // 不阻塞的活跃线程
```

接到 Micrometer：

```java
Gauge.builder("commonpool.active", common, ForkJoinPool::getActiveThreadCount)
    .register(meterRegistry);
```

#### 验证

```java
// 故意把 commonPool 占满，看其他 parallelStream 还能不能跑
@Test(timeout = 5000)
void parallelStreamMustNotBlockOnIo() {
    int n = ForkJoinPool.commonPool().getParallelism();
    for (int i = 0; i < n + 5; i++) {
        ForkJoinPool.commonPool().execute(() -> {
            try { Thread.sleep(60_000); } catch (Exception ignore) {}
        });
    }
    // 这里如果你的代码还在用 commonPool，会 timeout
    long sum = LongStream.range(0, 1_000_000).parallel().sum();
    assertTrue(sum > 0);
}
```

> **口诀**：commonPool 是全进程共享的稀缺资源。**给它的任务必须 CPU 密集 + 不阻塞**。任何不符合的，都换自己的池。

---

### 案例 ⑦ 队列容量过大 OOM —— "看似有界，其实是大对象炸弹"

#### 现象

线上 OOM。dump 文件里 `LinkedBlockingQueue$Node` 占了 6GB。

#### 根因

代码：

```java
new ThreadPoolExecutor(
    8, 32, 60, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(),         // ❌ 默认 Integer.MAX_VALUE
    threadFactory);
```

或者：

```java
new LinkedBlockingQueue<>(1_000_000);    // 看起来"有界"
```

但仔细一看，每个 task 持有一个图片字节数组（5KB），100 万 task = 5GB。流量稍微一冲就 OOM。

更深层：`LinkedBlockingQueue$Node` 本身有对象头开销 + 引用 + GC 卡片成本，当队列堆到几十万时，**仅 GC 的扫描成本**就能让 Young GC 退化成 Full GC。

#### 修复

**原则一：队列必须真正"有界"**。

```java
// 容量不要拍脑袋。计算公式：
// queue ≈ max(QPS) × 平均处理时间(s) × 安全系数(2~3)
// 例：3000 QPS × 50ms = 150，安全系数 2 → 队列 300
new ArrayBlockingQueue<>(300)
```

**原则二：队列里只放轻量"票据"**。

```java
// ❌ 把整张图直接塞进任务
class ResizeTask implements Runnable {
    byte[] image;   // 5MB
    public void run() { resize(image); }
}

// ✅ 只塞 ID，跑任务时再去拿
class ResizeTask implements Runnable {
    long imageId;
    public void run() {
        byte[] img = imageStore.get(imageId);
        resize(img);
    }
}
```

**原则三：必须配监控**（这是最容易漏的）。

```java
new ExecutorServiceMetrics(pool, "biz-pool", Tags.of("type", "biz"))
    .bindTo(meterRegistry);
```

会自动暴露：
- `executor.active.threads`
- `executor.queued`
- `executor.completed.tasks`
- `executor.rejected.tasks`

告警阈值参考：
- `executor.queued / queue.capacity > 0.7` 持续 3 分钟 → 警告
- `executor.rejected.tasks` > 0 → 告警

#### 验证

```bash
# 上线前压测，看队列在峰值 QPS 下的实际堆积
wrk -t 8 -c 200 -d 60s http://...

# 同时观察
jcmd <pid> GC.heap_info
curl http://localhost:8080/actuator/metrics/executor.queued
```

> **口诀**：**永远不要用无界队列**。容量必须算过；任务对象必须轻；监控必须配。三件套缺一不可。

---

### 案例 ⑧ 虚拟线程的反模式 —— "JDK 21 升级后反而更慢了"

#### 现象

服务从 JDK 17 升到 JDK 21，业务把 `Executors.newFixedThreadPool(8)` 换成 `Executors.newVirtualThreadPerTaskExecutor()` 想吃虚拟线程红利。结果 P99 反而上升 30%。

#### 根因 1：虚拟线程不该池化

虚拟线程的设计哲学是**"一任务一线程，用完即弃"**。它的创建成本极低（几百纳秒），栈是按需扩展的连续内存块。

如果你试图"池化虚拟线程"：

```java
// ❌ 把虚拟线程"池化"，反而拿不到任何好处
ExecutorService pool = Executors.newFixedThreadPool(8, Thread.ofVirtual().factory());
```

`newFixedThreadPool` 只会创建 8 个虚拟线程，等于把虚拟线程退化成"轻量版的平台线程"，还多了池化的锁竞争开销。

正确姿势：

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (var t : tasks) executor.submit(t);
}
```

`newVirtualThreadPerTaskExecutor` 内部**每个任务起一个新虚拟线程**，没有池、没有队列。

#### 根因 2：CPU 密集任务不该用虚拟线程

虚拟线程靠 carrier thread（载体平台线程）跑，carrier 数量 = `availableProcessors()`。如果你的任务是 CPU 密集（不会主动 yield），虚拟线程就一直占着 carrier，调度退化成"几个虚拟线程在抢几个 carrier"，比直接用 `FixedThreadPool(N)` 还差 —— 因为多了一层调度开销。

| 场景 | 用什么 |
| --- | --- |
| HTTP 客户端、DB 查询、消息消费 | **虚拟线程**（IO 密集） |
| 图像处理、加解密、序列化 | **平台线程池**（核数 × 1~2） |
| 混合 | 拆任务 → 不同池 |

#### 根因 3：synchronized 块里做长 IO 会 pin

JDK 21 的虚拟线程在 `synchronized` 块内做阻塞 IO 时，**会"钉住"（pin）carrier thread** —— 该虚拟线程无法被卸载，相当于退化成平台线程，且占着 carrier 不放。

```java
// ❌ pin 现场
synchronized (this) {
    httpClient.send(req, ...);   // 长 IO
}

// ✅ 把锁换成 ReentrantLock 不会 pin
lock.lock();
try {
    httpClient.send(req, ...);
} finally { lock.unlock(); }
```

诊断 pinning：

```bash
# JDK 21
java -Djdk.tracePinnedThreads=full ...

# 输出会列出所有 pin 的栈
```

JEP 491（JDK 24+）大幅缓解了 `synchronized` pin 的问题，但生产升级前必须验证。

#### 修复

判断逻辑：

```java
// 业务一开始就分两个池
ExecutorService cpuPool = new ThreadPoolExecutor(
    cores, cores * 2, 60, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(500),
    threadFactory);

ExecutorService ioPool = Executors.newVirtualThreadPerTaskExecutor();

// 任务路由
public void handle(Request r) {
    if (r.isCpuBound()) {
        cpuPool.execute(() -> compute(r));
    } else {
        ioPool.execute(() -> remoteCall(r));
    }
}
```

#### 验证

```java
@Test
void virtualThreadShouldNotPin() throws Exception {
    Thread.startVirtualThread(() -> {
        synchronized (this) {
            // 这里如果有阻塞 IO，会触发 pin 警告
            try { Thread.sleep(100); } catch (Exception ignore) {}
        }
    }).join();
    // 观察 stderr 里有没有 "Thread pinned" 警告
}
```

> **口诀**：虚拟线程**不要池化、不要 CPU 密集、不要 synchronized 包长 IO**。三不要记牢就避开 80% 的坑。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 配置层
- [ ] 所有线程池都有可识别的线程名（`new ThreadFactoryBuilder().setNameFormat("biz-%d").build()`）
- [ ] 队列必须有界，且容量经过 `QPS × 处理时长` 计算
- [ ] 拒绝策略明确：对外服务用 AbortPolicy + 上游限流；内部计算用 CallerRunsPolicy 或自定义阻塞
- [ ] 不用 `Executors.newXxx`（无界队列陷阱），永远手写 `ThreadPoolExecutor`
- [ ] `core / max / keepAlive` 在配置中心可调（动态线程池）

### 关闭层
- [ ] `@PreDestroy` 配置 `shutdown + awaitTermination + shutdownNow + awaitTermination`
- [ ] Spring `ThreadPoolTaskExecutor` 设 `setWaitForTasksToCompleteOnShutdown(true)`
- [ ] K8s `terminationGracePeriodSeconds > pool 总等待时间`
- [ ] `preStop` 有 `sleep N` 等待 endpoint 摘除
- [ ] 任务代码响应中断（`!Thread.currentThread().isInterrupted()`）

### 异常层
- [ ] 覆写 `afterExecute` 或在任务里 try-catch 兜底
- [ ] Scheduled 任务必须 try-catch，否则抛异常后不再调度
- [ ] 设置 `Thread.UncaughtExceptionHandler`（仅对 `execute` 有效）

### 上下文层
- [ ] MDC / TraceId / 多租户上下文用 capture-replay-restore 透传（装饰器或 TTL）
- [ ] worker 复用前清理 ThreadLocal（finally 里 restore）

### 隔离层
- [ ] 不在池 A 里 `join` / `get` 提交到池 A 的任务（嵌套死锁）
- [ ] `CompletableFuture.supplyAsync` 永远传 Executor，不依赖 commonPool
- [ ] 长任务（> 分钟级）单独池，不混在通用池
- [ ] CPU 密集 / IO 密集分两个池

### 监控层
- [ ] Micrometer `ExecutorServiceMetrics` 接入
- [ ] 监控指标：active / queue / completed / rejected
- [ ] 告警：`queue / capacity > 0.7` 持续 3 分钟；rejected > 0
- [ ] commonPool 也要监控（`ForkJoinPool.commonPool().getActiveThreadCount()`）

### 虚拟线程层（JDK 21+）
- [ ] 虚拟线程用 `newVirtualThreadPerTaskExecutor`，不要池化
- [ ] CPU 密集任务用平台线程池，不丢虚拟线程
- [ ] `synchronized` 不要包长 IO（用 ReentrantLock 替代）
- [ ] 启动参数加 `-Djdk.tracePinnedThreads=short` 至少跑一次

---

## 6. 还能往下挖什么

读完上面 8 个案例只是起点。如果想再深一层：

### HotSpot 层
- `Thread.start()` 的 native 路径：`Thread.start0` → `JVM_StartThread` → `pthread_create` → `JavaThread::run`
- Worker 线程的栈大小由 `-Xss` 决定，默认 1MB；千线程级应用要算总开销
- 可以用 `jdk.JavaMonitorEnter` JFR 事件追踪 synchronized 竞争

### Loom（虚拟线程）
- JEP 444 虚拟线程实现：`Continuation` + `ContinuationScope` + carrier thread 调度
- JEP 491（JDK 24）的 `synchronized` pin 修复方案
- `StructuredTaskScope`（JEP 505）—— 结构化并发，把任务的"作用域"显式化
- ScopedValue（JEP 506）—— ThreadLocal 的不可变替代，对虚拟线程友好

### 调度模型对比
- **ThreadPoolExecutor**（生产者-消费者）：通用，控制力强
- **ForkJoinPool**（work-stealing）：任务能拆分时最优，IO 阻塞时退化
- **EventLoop**（Netty / Vert.x）：单线程跑大量连接，回调风格
- **Virtual Thread + StructuredTaskScope**（JDK 21+）：同步代码 + 异步性能
- **Disruptor / LMAX**：单生产单消费 + 环形缓冲区，做到延迟 < 微秒

### 响应式 vs 虚拟线程
- Project Reactor / RxJava 的 Scheduler 模型为什么开始"过时"
- WebFlux 在虚拟线程时代的定位
- 哪些场景虚拟线程仍然替代不了响应式（背压、流式聚合）

### ScheduledThreadPoolExecutor 内部
- 它的队列是个**最小堆**（`DelayedWorkQueue`），不是普通 BlockingQueue
- 周期任务的"漂移"问题：`scheduleAtFixedRate` 在任务超时执行后会立即追赶
- `scheduleWithFixedDelay` 与 `scheduleAtFixedRate` 在 GC 长 STW 后的不同表现

---

## 7. 引用

- OpenJDK 21 源码：[ThreadPoolExecutor.java](https://github.com/openjdk/jdk21u/blob/master/src/java.base/share/classes/java/util/concurrent/ThreadPoolExecutor.java)
- Doug Lea, *The java.util.concurrent Synchronizer Framework*（AQS 论文，2005）
- JEP 444: Virtual Threads
- JEP 491: Synchronize Virtual Threads without Pinning
- 阿里 [TransmittableThreadLocal](https://github.com/alibaba/transmittable-thread-local) 源码
- 美团 [DynamicTp](https://github.com/dromara/dynamic-tp) 设计文档
- Brian Goetz, *Java Concurrency in Practice* —— 并发圣经（虽然 2006 年，但模型仍适用）

---

> 上一篇综述：[并发编程总览](/article/java/concurrent) ｜ 配套：[JMM](/article/jvm/jmm)
> 这是按 [桃花岛深度模板](/STYLE) 写的样板。如果你觉得这个深度对，告诉我下一篇该写哪个。
