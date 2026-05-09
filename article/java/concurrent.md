# 并发编程总览

> 并发是 Java 后端的"硬通货"。本文按"线程基础 → 锁 → 并发工具 → 线程池 → 实战经验"的路径串起来，前置阅读 [JMM](/article/jvm/jmm)。

---

## 1. 线程基础

### 1.1 创建线程的 4 种方式
```java
// 1. 继承 Thread
new Thread(){ public void run(){...} }.start();

// 2. 实现 Runnable（推荐）
new Thread(() -> {...}).start();

// 3. Callable + FutureTask（带返回值）
FutureTask<String> task = new FutureTask<>(() -> "hello");
new Thread(task).start();
String r = task.get();

// 4. 线程池（生产唯一推荐）
ExecutorService pool = Executors.newFixedThreadPool(8);
Future<String> f = pool.submit(() -> "hello");
```

### 1.2 线程状态
```
NEW → RUNNABLE → BLOCKED / WAITING / TIMED_WAITING → TERMINATED
```

| 状态 | 触发 |
| --- | --- |
| RUNNABLE | start() 后 |
| BLOCKED | 等待 monitor 锁（synchronized） |
| WAITING | wait() / join() / LockSupport.park() |
| TIMED_WAITING | sleep(t) / wait(t) / parkNanos |
| TERMINATED | run 结束 |

### 1.3 sleep vs wait
| 对比 | sleep | wait |
| --- | --- | --- |
| 类 | Thread | Object |
| 释放锁 | ❌ | ✅ |
| 必须在同步块 | ❌ | ✅ |
| 唤醒 | 时间到/中断 | notify/notifyAll/中断 |

### 1.4 中断
中断是协作式信号，不是强制停止：
```java
while (!Thread.currentThread().isInterrupted()) {
    try {
        // 工作
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();   // 重置中断标志
        break;
    }
}
```

---

## 2. synchronized 与 Lock

> mark word 内幕、锁膨胀全路径、JDK 15+ 偏向锁废弃后的退化事故、虚拟线程下 pin 的根因 → **[synchronized 内幕（深文）](/article/java/synchronized)**

### 2.1 synchronized
- JDK 6 后引入锁升级（无锁→偏向→轻量→重量），见 [JMM 文章](/article/jvm/jmm#7-synchronized-与锁优化)；
- 可重入；
- 异常自动释放；
- 不可中断、不可设超时。

### 2.2 ReentrantLock
```java
ReentrantLock lock = new ReentrantLock(true);  // 公平锁
lock.lock();
try {
    // 临界区
} finally {
    lock.unlock();        // 必须 finally 里释放
}

// 可中断
lock.lockInterruptibly();
// 可超时
lock.tryLock(1, TimeUnit.SECONDS);
```

| 对比 | synchronized | ReentrantLock |
| --- | --- | --- |
| 可重入 | ✅ | ✅ |
| 可中断 | ❌ | ✅ |
| 可超时 | ❌ | ✅ |
| 公平锁 | ❌ | 可选 |
| 多个 Condition | ❌ | ✅ |
| 性能 | 现代版本基本持平 | — |

### 2.3 ReadWriteLock & StampedLock
- `ReentrantReadWriteLock`：读读共享、读写/写写互斥。读多写少更优；
- `StampedLock`（JDK8）：进阶版，提供**乐观读**，性能更好但 API 复杂、不可重入。

---

## 3. AQS（速记）

> 完整源码追踪、Condition 队列与 sync 队列的转移机制、公平锁吞吐崩塌等 9 个生产案例 → **[AQS 源码深读（深文）](/article/java/aqs)**

`AbstractQueuedSynchronizer` 是 J.U.C 的核心，**ReentrantLock / Semaphore / CountDownLatch / FutureTask 都基于它**。

```
state（int / long） + 一个 CLH 队列（双向链表）
```

- `state` 表示锁/信号资源；
- 抢锁失败的线程封装成 Node 入队，park 阻塞；
- 持锁线程释放后 unpark 队首线程。

```java
// 子类只需重写：
tryAcquire(int)        // 排他模式
tryRelease(int)
tryAcquireShared(int)  // 共享模式
tryReleaseShared(int)
```

> 把 AQS 想象成"一个 int + 一个等待队列"，所有同步器都是它的派生策略。

---

## 4. 并发工具类

### 4.1 CountDownLatch
"等所有人到齐再开车"：
```java
CountDownLatch latch = new CountDownLatch(3);
// worker
latch.countDown();
// 主线程
latch.await();   // 一次性，不可重置
```

### 4.2 CyclicBarrier
"凑够 N 个就出发，发完可以重来"：
```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("满员"));
barrier.await();
```

### 4.3 Semaphore
"令牌池，限流神器"：
```java
Semaphore sema = new Semaphore(10);
sema.acquire();
try { /* 业务 */ } finally { sema.release(); }
```

### 4.4 Phaser
更灵活的 CyclicBarrier，支持动态增减参与者。

### 4.5 Exchanger
两个线程交换数据，少用。

---

## 5. 原子类与 CAS

### 5.1 Atomic 家族
```java
AtomicInteger / AtomicLong / AtomicReference
AtomicIntegerArray / AtomicLongArray
AtomicStampedReference        // 解决 ABA
LongAdder / LongAccumulator   // 高并发计数（分段累加）
```

> **高竞争下用 LongAdder 替代 AtomicLong**：内部用 Cell 数组分散热点，最后 sum() 累加。

### 5.2 CAS 三大问题
1. **ABA**：`AtomicStampedReference`；
2. **自旋开销**：高竞争下不如锁；
3. **只能保证单变量原子性**：用 `AtomicReference` 包对象。

---

## 6. 线程池（速记）

> 这一节只给面试速答。**写代码会遇到的坑、源码层细节、生产事故复盘**全在 → **[线程池生产实战（深文）](/article/java/threadpool)**。

### 6.1 7 参数
```java
new ThreadPoolExecutor(
    corePoolSize, maximumPoolSize, keepAliveTime, TimeUnit,
    workQueue, threadFactory, rejectedExecutionHandler);
```

### 6.2 任务流转
```
来任务 → core 没满？开 core
       → 满 → 入队
              → 队列满？开非 core 到 max
                       → 满 → 拒绝策略
```

### 6.3 4 种拒绝策略
| 策略 | 行为 |
| --- | --- |
| `AbortPolicy`（默认） | 抛 RejectedExecutionException |
| `CallerRunsPolicy` | 提交线程自己跑（反压） |
| `DiscardPolicy` | 静默丢 |
| `DiscardOldestPolicy` | 丢队首 |

> 实战选型、JMH 实测对比 → [深文 §3](/article/java/threadpool#3-实测数据拒绝策略不是平等的)

### 6.4 不要用 Executors（阿里规约）
`newFixedThreadPool` / `newCachedThreadPool` / `newSingleThreadExecutor` / `newScheduledThreadPool` 全部用无界队列或无界线程数 → OOM。**生产永远手写 ThreadPoolExecutor**。

### 6.5 核心 / 最大怎么定
- **CPU 密集**：核数 + 1
- **IO 密集**：核数 × (1 + 平均等待/平均计算)，常见 2N~4N
- **业务峰值**：先压测，按 P99 倒推

### 6.6 提交的异常会被吞
`pool.submit(task)` 抛的异常进 Future，没人 `f.get()` 就永远沉默；`scheduleAtFixedRate` 抛异常后**直接停止后续调度**。
→ 全套解决方案（afterExecute / Spring ErrorHandler / 验证用例）见 [深文 §4 案例 ①](/article/java/threadpool#案例--异常被-future-吞--周期任务静默死亡)

### 6.7 还会踩的坑（深文里全部展开）
- 优雅停机被 K8s SIGKILL 强杀，DB 留脏数据
- 池内 join 池内任务 → 死锁
- 不重启动态调 `core / max` 的正确顺序
- MDC / TraceId 跨线程池失踪
- `CompletableFuture.supplyAsync` 默认池被污染，全站 parallelStream 卡死
- `LinkedBlockingQueue<>(1_000_000)` 看似有界，其实 OOM 炸弹
- JDK 21 把 FixedThreadPool 换成虚拟线程反而更慢

→ 8 个案例 + 上线检查清单 20 条 → [线程池生产实战](/article/java/threadpool)

---

## 7. ThreadLocal 与 InheritableThreadLocal

```java
ThreadLocal<User> CTX = new ThreadLocal<>();
CTX.set(user);
try {
    // ... 当前线程任意位置 CTX.get()
} finally {
    CTX.remove();              // 铁律
}
```

### 7.1 内部结构
- 每个 `Thread` 持有 `ThreadLocalMap`；
- key 是 ThreadLocal 实例（**弱引用**），value 是真正的对象（**强引用**）；
- 线程池中线程不死 → value 可能永远不释放 → **内存泄漏**。
- 用完 `remove()`！

### 7.2 父子线程传递
- `InheritableThreadLocal`：父线程 → 新建子线程时复制；
- 但**线程池场景失效**（子线程是预先创建的）；
- 阿里 **TransmittableThreadLocal**（TTL）解决此问题：装饰 Runnable 在 submit 时捕获、执行时回放。

---

## 8. CompletableFuture（异步编排）

```java
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(this::loadUser);
CompletableFuture<Integer> f2 = CompletableFuture.supplyAsync(this::loadScore);

f1.thenCombine(f2, (u, s) -> u + ":" + s)
  .thenApply(String::toUpperCase)
  .exceptionally(e -> "FAIL")
  .thenAccept(System.out::println);

// 全部完成
CompletableFuture.allOf(f1, f2).join();
// 任意一个完成
CompletableFuture.anyOf(f1, f2).join();
```

> 默认线程池是 `ForkJoinPool.commonPool`，**生产必须传自定义 Executor**，避免阻塞通用池。

---

## 9. 虚拟线程（JDK21+）

> [虚拟线程深入](/article/java/virtual-thread)（使用与生产案例）+ [Project Loom 内部](/article/jvm/loom)（Continuation 物理实现 + 9 个 JVM 内核事故）

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 10000; i++) {
        executor.submit(() -> httpCall());
    }
}
```
- 适合 IO 密集，单机轻松百万线程；
- 不要在虚拟线程中做 CPU 密集；
- 避免在虚拟线程里持有 `synchronized` 做长 IO（会 pin 住载体线程，JDK21 已大幅缓解）。

---

## 10. 死锁预防

四个必要条件：互斥、占有且等待、不可剥夺、循环等待。打破其中一个即可：
- **统一加锁顺序**（最常用）；
- **设置超时**：`tryLock`；
- **避免长事务持锁**；
- **死锁检测**：`jstack` 找 `Found one Java-level deadlock`。

---

## 11. 高频面试题速记

1. **synchronized 与 Lock 区别？** 见第 2 节表。
2. **AQS 原理？** state + CLH 队列，子类重写 tryAcquire/tryRelease。
3. **ReentrantLock 公平 vs 非公平？** 公平按队列顺序；非公平允许新线程插队，吞吐高。
4. **线程池 7 参数与流转？** 见第 6 节。
5. **为什么不能用 Executors.newFixedThreadPool？** 队列无界，OOM。
6. **ThreadLocal 为什么会内存泄漏？** key 弱引用 value 强引用，线程池长期持有。
7. **CompletableFuture 默认线程池是哪个？** ForkJoinPool.commonPool，生产必传自定义 Executor。
8. **虚拟线程和平台线程区别？** 由 JVM 调度到少量平台线程，IO 密集场景吞吐巨幅提升。
9. **CAS 的问题？** ABA、自旋开销、只能单变量。
10. **怎么写一个生产可用的 LRU？** LinkedHashMap + accessOrder；并发场景用 Caffeine。

---

> 上一篇：[集合框架](/article/java/collection) ｜ 相关：[JMM](/article/jvm/jmm)
