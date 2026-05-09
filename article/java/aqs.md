# AQS 源码深读 —— 一个 int 撑起整个 J.U.C

> 这是按 [桃花岛深度模板](/STYLE) 写的第三篇深文。如果只想刷面试，看 [并发综述](/article/java/concurrent#3-aqs必背) 那一节就够了。
> 这里要回答的，是**写自定义同步器、半夜被锁告警叫起来**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. AQS 的 `state` 是个 `int`。`ReentrantLock` 的可重入次数怎么编进去？`ReadWriteLock` 的"读锁数量 + 写锁数量"又怎么编进去？两者用同一个字段做完全不同的事 —— 区别在哪一行代码？
2. 公平锁（`new ReentrantLock(true)`）和非公平锁的源码差别只有**一个判断**。说出这一行代码，并解释为什么这一行就决定了高竞争下吞吐能差 5 倍。
3. CLH 队列里的 `Node` 有 5 种 `waitStatus`：`CANCELLED(1)`、`SIGNAL(-1)`、`CONDITION(-2)`、`PROPAGATE(-3)`、`0`（初始）。每种在状态机里**什么时候被设、被谁观察**？
4. `acquireQueued` 是 AQS 最核心的自旋。它的退出条件是什么？为什么一定要在退出前判断"前驱是不是 head"？
5. `Condition.await()` 把当前线程从"sync 队列"挪到"condition 队列"。`signal()` 反过来挪。这两个队列是同一种 `Node` 复用？还是两个独立结构？挪过去时具体改了 `Node` 的哪几个字段？
6. `Semaphore`、`CountDownLatch`、`FutureTask` 都基于 AQS。它们的代码差别**只在 `tryAcquire` / `tryRelease` 这两个方法**。说出 `Semaphore.tryAcquireShared` 的核心逻辑，对比 `CountDownLatch.tryAcquireShared` 的核心逻辑。
7. `tryLock()` 与 `tryAcquire(int)` 是什么关系？前者是后者的特例吗？为什么 `tryLock()` 在公平锁上的行为**违反公平**？
8. `ReadWriteLock` 把 `state` 拆成高 16 位（共享）+ 低 16 位（独占）。`sharedCount(c) = c >>> 16`，`exclusiveCount(c) = c & 0xFFFF`。为什么必须这么拆？写锁拿读锁要怎么编码？

如果有半数答不上，往下看。

---

## 1. 底层模型：state + CLH 变种队列 + Condition

绝大多数文章把 AQS 讲成"同步状态 + FIFO 队列"，然后画一张方框图就结束了。这层抽象解释不了上面 8 个问题。要看懂它们，必须下沉到 `AbstractQueuedSynchronizer`（下称 AQS）的三块核心数据结构。

### 1.1 AQS 的核心：一个 volatile int + 一个等待队列

打开 OpenJDK 21 的 `AbstractQueuedSynchronizer.java`，整个类几千行代码，但成员字段只有寥寥几个：

```java
public abstract class AbstractQueuedSynchronizer
        extends AbstractOwnableSynchronizer
        implements java.io.Serializable {

    private transient volatile Node head;     // 队列头（dummy 节点）
    private transient volatile Node tail;     // 队列尾
    private volatile int state;                // ★ 同步状态，子类自己解释
    // ... 几个静态 VarHandle 用于 CAS
}
```

只有三个核心字段。**这就是整个 J.U.C 锁体系的骨架**。

`state` 是个普通的 `int`，但它的语义完全由子类决定：
- `ReentrantLock`：`state == 0` 表示无人持锁，`state > 0` 表示重入次数
- `Semaphore`：`state` 表示可用许可数
- `CountDownLatch`：`state` 表示剩余 count
- `ReentrantReadWriteLock`：`state` 高 16 位 = 读锁数，低 16 位 = 写锁重入数
- `FutureTask`：自己**没用** AQS 的 state，而是用 `private volatile int state` —— 但内部仍然复用了 AQS 的"挂起/唤醒"思想，JDK 6 版本曾基于 AQS 实现，JDK 7 重写

为什么是 `int` 而不是 `long`？Doug Lea 在论文里写过：当时（2004 年）32 位 CAS 在所有平台都原生支持，64 位 CAS 在某些 32 位平台需要用锁模拟。**为了让 AQS 在所有 JVM 上都享受单条 CAS 指令的性能，state 选用 int**。

到了 2024 年，64 位 CPU 早已统治世界。`StampedLock` 已经用 `long state` 编更多信息（版本戳）。但 AQS 为了兼容性没改。

> **口诀**：state 是 AQS 唯一的"可变状态"，子类怎么解释由子类说了算。**改 state 必须用 CAS**，读 state 直接走 volatile 语义即可。

### 1.2 CLH 队列变种：Node 的五个字段为什么这么设计

经典的 CLH 锁（Craig, Landin, Hagersten，1993）是个**自旋锁**，每个线程在自己的 `prev` 节点上自旋。AQS 把它改成了**阻塞锁**，每个线程 park 在自己的节点上等被唤醒。

`Node` 类的核心字段（OpenJDK 21 简化版）：

```java
abstract static class Node {
    volatile Node prev;       // 前驱
    volatile Node next;       // 后继
    Thread waiter;            // 持有这个节点的线程
    volatile int status;      // waitStatus（5 种值）
}
```

为什么需要 `prev` 和 `next` **同时存在**？

- **prev** 用来检查"我前面是不是 head"，决定能不能尝试拿锁。这是入队后的核心循环条件
- **next** 用来在释放锁时**唤醒后继**（`unparkSuccessor` 沿 `next` 找）

只有一个方向行不行？不行：
- 只有 `prev`：释放锁的线程不知道下一个要唤醒谁（要从 tail 反向遍历找，O(n)）
- 只有 `next`：节点被取消时无法把自己从队列中"摘除"（需要修改前驱的 next，但找不到前驱）

但**实现细节里 next 不可靠**。看入队代码：

```java
private Node enq(Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) {
            // 初始化：head = tail = new dummy node
            if (compareAndSetHead(new Node())) tail = head;
        } else {
            node.prev = t;                          // ① 先设 prev
            if (compareAndSetTail(t, node)) {       // ② CAS 设新 tail
                t.next = node;                      // ③ 后设 next
                return t;
            }
        }
    }
}
```

注意顺序：**先 prev 再 CAS tail 再 next**。如果在 ② 和 ③ 之间发生中断或异常，`t.next` 就还是 null，但 `node.prev` 已经指对了。这是为什么 AQS 内部很多遍历是**用 prev 反向扫描**而不是用 next。

> **口诀**：CLH 队列的 prev 是"权威"，next 是"提示"。**任何依赖 next 的逻辑都要兜底**：找不到就从 tail 反扫。

### 1.3 五种 waitStatus 的状态机

`Node.waitStatus`（JDK 17/21 改名为 `status`，但语义不变）有 5 种取值：

| 值 | 名字 | 含义 | 谁设 | 谁观察 |
| --- | --- | --- | --- | --- |
| **0** | 初始 | 节点刚入队 | 入队时默认 | — |
| **-1** | `SIGNAL` | "我后面有人在睡，我醒了要叫他" | **后继**入队后设给前驱 | 释放锁时检查 |
| **-2** | `CONDITION` | 节点在 condition 队列里等 | `await()` | `signal()` 转移时清掉 |
| **-3** | `PROPAGATE` | 共享模式下的"传播"标记 | `setHeadAndPropagate` | 共享释放时检查 |
| **1** | `CANCELLED` | 节点已取消（超时/中断） | 取消获取时设 | 队列遍历时跳过 |

最反直觉的设计：**SIGNAL 是"后继设给前驱"的**。

```
节点 A（已入队，waitStatus=0）
节点 B（新入队）：B 把 A 的 waitStatus 改成 SIGNAL，意思是 "A 你释放时叫我"
然后 B 才安心 park
```

为什么这么设计？因为 A 没法预知后面会不会有 B 来等它。如果 A 提前把自己设为 SIGNAL，但其实没人等，唤醒就是空操作浪费。**让 B 自己负责"通知 A：你后面有人在等"**，A 释放时根据这个标记决定要不要 unpark。

完整的状态转换图：

```
      入队
       ↓
       0 ─────────────► SIGNAL  ←─ 被后继设
       │                  │
       │ 取消             │ 锁释放后清 0 / 用完
       ▼                  ▼
   CANCELLED              0
       │
       │ (queue cleanup 摘除)
       ▼
      消失

condition 队列里只走另一条线：
       0 ─── await() ──► CONDITION ─── signal() ──► 0（转移到 sync 队列）
                              │
                              │ 中断
                              ▼
                          CANCELLED
```

> **口诀**：`SIGNAL` 是后继挂给前驱的"叫醒便条"。`CANCELLED` 一旦设上不可逆，必须从队列摘。`CONDITION` 只在 condition 队列里出现，转移到 sync 队列时立刻清零。

### 1.4 独占模式 vs 共享模式的本质差别

AQS 同一套队列同时支持两种模式：

| 维度 | 独占（exclusive） | 共享（shared） |
| --- | --- | --- |
| 一次只能多少个线程持有 | 1 个 | 多个（由 state 决定） |
| 子类要实现的方法 | `tryAcquire` / `tryRelease` | `tryAcquireShared` / `tryReleaseShared` |
| 释放时唤醒 | 唤醒**一个**后继 | 可能唤醒**多个**后继（传播） |
| 节点标记 | `Node.EXCLUSIVE` (null) | `Node.SHARED`（一个常量节点） |
| 典型实现 | `ReentrantLock` / `Sync` 内部类 | `Semaphore` / `CountDownLatch` / `ReadLock` |

共享模式的关键代码 `setHeadAndPropagate`：

```java
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head;
    setHead(node);
    if (propagate > 0 || h == null || h.status < 0
        || (h = head) == null || h.status < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();          // ★ 唤醒后继，让它继续传播
    }
}
```

**propagate 是 `tryAcquireShared` 的返回值**：
- `< 0`：失败
- `= 0`：成功，但没有剩余资源给后继（不传播）
- `> 0`：成功，还有资源（传播给后继）

`Semaphore` 的 `tryAcquireShared` 返回 `available - acquires`，刚好编码了"成功 + 剩多少"。这就是为什么 `Semaphore` 释放一个许可时可能"批量唤醒"多个等待者，而 `ReentrantLock` 释放一次只唤醒一个。

> **口诀**：独占释放唤醒一个后继；共享释放可能传播唤醒多个。**propagate 的值就是接力棒**。

### 1.5 Condition 队列：单链表 + sync 队列的转移

`ReentrantLock.newCondition()` 拿到的是 AQS 内部的 `ConditionObject`。它**是一个独立的单向链表**，不复用 sync 队列：

```java
public class ConditionObject implements Condition {
    private transient ConditionNode firstWaiter;
    private transient ConditionNode lastWaiter;
    // ...
}
```

但**节点类型是同一个 Node 子类**，只是**状态变了**：
- 在 condition 队列里时 `status = CONDITION`，用 `nextWaiter` 字段串成单链表
- `signal()` 时把它从 condition 队列摘下，改 `status = 0`，**入到 sync 队列尾巴**，等待重新拿锁

完整的转移路径（一图道尽）：

```
Lock 持有者 (Thread T)
  │
  └── lock.newCondition().await()
       │
       1. 创建一个 ConditionNode N
       2. N.status = CONDITION
       3. N 串到 condition 队列尾 (lastWaiter.nextWaiter = N)
       4. 完全释放 lock (state 设回 0，记下当前 holdCount)
       5. park 自己

       ... 等待 ...

       (另一个线程 T2 拿锁后调 signal())
       6. 摘 condition 队列头 N
       7. N.status = 0
       8. N 入 sync 队列尾 (变成普通的 sync 节点)
       9. 如果前驱已 CANCELLED，立即 unpark N（让它去争锁）

       (T 醒来)
      10. T 在 sync 队列里走 acquireQueued 重新争锁
      11. 拿到锁后恢复之前的 holdCount
      12. await() 返回
```

为什么 condition 队列要用单链表而不是双向？因为 condition 队列**永远顺序操作**：
- 入：尾插
- 出：头摘
- 中间节点不会被随意访问（除非中断时摘自己，需要 O(n) 扫描，但中断本来就少见）

省一个 `prev` 字段，每个 await 节点省 8~16 字节内存。在百万级 condition 等待场景（线程池、连接池）这是真金白银的省。

> **口诀**：condition 队列和 sync 队列是**两条独立链表**，但**节点类型是同一个**。`signal` 的本质是把节点从 condition 链表摘下、改 status、挂到 sync 链表尾。

---

## 2. 源码切片：JDK 17/21 AbstractQueuedSynchronizer

> 基于 OpenJDK 21 `AbstractQueuedSynchronizer.java` 源码。注释为本文添加。

JDK 9 之后 AQS 内部用 `VarHandle` 替代了 `Unsafe`，JDK 17/21 又重构了一些 helper 方法（比如把 `Node` 拆成抽象类 + `ExclusiveNode` / `SharedNode` / `ConditionNode` 三个子类）。下面切片用 JDK 21 的形式，但**关键算法和 JDK 8 完全一致**。

### 2.1 acquire 完整路径

独占模式获取锁的入口：

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg))
        acquire(null, arg, false, false, false, 0L);
}
```

`tryAcquire` 是子类实现的"快路径"。失败才进入慢路径 `acquire(...)`（JDK 21 把多个变种合并到一个方法，参数 `shared / interruptible / timed` 控制行为）。

JDK 8 的经典写法更易读：

```java
// JDK 8 风格（语义等价）
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

**逐段点透**：

- `tryAcquire(arg)`：交给子类。`ReentrantLock` 的 `NonfairSync.tryAcquire` 直接 CAS 抢 state。
- `addWaiter(Node.EXCLUSIVE)`：把当前线程包成节点，入队尾。
- `acquireQueued(node, arg)`：在队列里自旋 + park，**返回值是"等待期间是否被中断过"**。注意这里**不抛 InterruptedException**，只是把中断标志记下来。
- `selfInterrupt()`：如果等待期间被中断了，醒来时把中断标志重新设上（因为 park 醒来时 JVM 会自动清掉中断标志）。

**整段最反直觉**：`acquire` 不抛 `InterruptedException`！它"吞"掉中断，最后用 `Thread.currentThread().interrupt()` 重新设上。这是 `lock.lock()` 不响应中断的根本原因。

如果你想要可中断版本，用 `lockInterruptibly()`，它走的是 `acquireInterruptibly`：

```java
public final void acquireInterruptibly(int arg) throws InterruptedException {
    if (Thread.interrupted()) throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);     // 内部 park 检测到中断会直接抛
}
```

> **口诀**：`acquire` 吞中断、`acquireInterruptibly` 抛中断。`lock.lock()` 不响应中断，要响应中断必须 `lockInterruptibly()`。

### 2.2 acquireQueued —— AQS 最核心的自旋

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();         // 拿前驱
            if (p == head && tryAcquire(arg)) {        // ★ 只有前驱是 head 时才尝试
                setHead(node);
                p.next = null;                          // 帮 GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node)  // 把前驱设为 SIGNAL
                && parkAndCheckInterrupt())            // park 自己，醒来检查中断
                interrupted = true;
        }
    } finally {
        if (failed) cancelAcquire(node);
    }
}
```

**逐段点透**：

- `p == head`：自己是队列里"实际等待的第一个"（head 是 dummy 节点）。**只有这种情况下才允许尝试 tryAcquire**，否则就 park。这是 AQS 公平性的基础（虽然 `NonfairSync.tryAcquire` 内部还会"插队"）。
- `setHead(node)`：拿到锁后，把自己变成新的 dummy head。**这是 head 永远是 dummy 节点的实现**。
- `p.next = null`：把旧 head 的 next 断掉，帮助 GC 回收。
- `shouldParkAfterFailedAcquire`：返回 true 说明"已经把前驱设为 SIGNAL，可以放心 park"。
- `parkAndCheckInterrupt`：park 醒来后返回 `Thread.interrupted()`（清中断标志并返回原值）。
- `failed` + `cancelAcquire`：异常路径兜底。如果 `tryAcquire` 抛异常就把节点标记为 CANCELLED 摘除。

**为什么要 `for(;;)` 自旋而不是 park 一次就完？** 因为：
- 你被前驱 unpark 了不代表你拿得到锁（非公平模式可能被外面线程"插队"抢走）
- park 可能被"虚假唤醒"（spurious wakeup）
- 中断也会让 park 返回

所以**醒来必须再 check 一次**，拿不到就接着 park。这是个"标准 wait-condition 模式"。

> **口诀**：`acquireQueued` 的退出条件是"前驱是 head 且 tryAcquire 成功"。**任何唤醒都要重新验证**，不要假设"被叫醒 = 拿到了"。

### 2.3 shouldParkAfterFailedAcquire —— SIGNAL 的状态机

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)                         // 前驱已经承诺会叫我，放心 park
        return true;
    if (ws > 0) {                                  // 前驱是 CANCELLED，往前找一个非 CANCELLED 的
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {                                       // 前驱是 0 或 PROPAGATE
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);  // 设为 SIGNAL，再来一轮检查
    }
    return false;
}
```

**逐段点透**：

- 三个分支对应前驱的三种状态：已经 SIGNAL → park；CANCELLED → 跳过；其他 → CAS 设为 SIGNAL 后重试。
- **CAS 设 SIGNAL 后不立即 park**，而是返回 false 让外层再走一次循环。为什么？因为这一轮"我的前驱可能刚释放了锁"，回去再试一次 tryAcquire 可能直接成功，省一次 park/unpark 开销。
- 跳过 CANCELLED 节点时**修改了 prev 链**：`node.prev = pred = pred.prev`。这就是为什么 AQS 的 prev 是动态变化的、可能跳过中间几个 CANCELLED 节点。

这段代码体现了 AQS 的核心设计哲学：**park / unpark 是有成本的，能不睡就不睡**。哪怕只是再 spin 一次拿到锁，都比 park-unpark 一对要省。

> **口诀**：把前驱设为 SIGNAL 是 AQS 唯一的"我要睡了"信号。**SIGNAL 设完不立即 park**，先回去再争一次。

### 2.4 release 完整路径

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);   // 把 SIGNAL 清 0，幂等

    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        // ★ next 不可靠，从 tail 反向找
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

**逐段点透**：

- `tryRelease(arg)`：交给子类。`ReentrantLock.tryRelease` 减 state，到 0 才返回 true 表示真正释放。
- `h.waitStatus != 0`：head 的 waitStatus 不为 0 才需要唤醒后继。0 意味着"后面没人在等"或者"刚被人唤醒过"，不用再 unpark。
- `unparkSuccessor` 里**先把 head 的 SIGNAL 清成 0**，再去 unpark 后继。这个清 0 是幂等的，CAS 失败也无所谓。
- **从 tail 反扫**找后继：next 可能为 null（入队过程中只设了 prev 还没设 next）或者后继是 CANCELLED。反扫保证一定能找到最靠近 head 的"非 CANCELLED 节点"。

为什么不直接 `LockSupport.unpark(s.thread)` 然后让它自己判断？因为可能根本没有要唤醒的节点（队列空 / 后面全是 CANCELLED），调一次 unpark 是浪费。提前判断可以避免无效系统调用。

> **口诀**：`release` 只唤醒**一个**后继。next 不可靠时**从 tail 反扫**。

### 2.5 acquireShared / releaseShared —— PROPAGATE 的存在意义

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}

private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);     // ★ 共享节点
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r); // ★ 关键：可能传播
                    p.next = null;
                    if (interrupted) selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node)
                && parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed) cancelAcquire(node);
    }
}
```

`setHeadAndPropagate` 的关键判断（前面讲过）：**如果还有剩余资源，唤醒下一个**。

`releaseShared` 配套：

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}

private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // CAS 失败重试
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                 // ★ 这里设 PROPAGATE
        }
        if (h == head) break;             // head 没变就退出
    }
}
```

**PROPAGATE 是什么？为什么要它？**

考虑 bug：JDK 5 时代曾有 issue（[bug 6801020](https://bugs.openjdk.org/browse/JDK-6801020)），共享模式下可能出现"该唤醒的没唤醒"。

场景重现（简化）：
1. 线程 T1 拿到共享锁，head = node1
2. T1 即将 release，但还没 release
3. 线程 T2 入队成为 node1 的后继，把 node1 的 status 设为 SIGNAL
4. T1 release，调 doReleaseShared，把 node1 的 SIGNAL 清 0，unpark T2
5. 线程 T3 入队成为 T2 的后继
6. T2 醒来，拿到锁，setHead(T2)。但此时如果只看"propagate > 0"，可能漏掉 T3 的唤醒
7. 后果：T3 永远 park

修复就是引入 PROPAGATE：当 `unparkSuccessor` 后还没人来"接收"这个 SIGNAL 信号时，把状态留成 PROPAGATE 而不是 0。新的 head 在 setHeadAndPropagate 里检查到 PROPAGATE 就主动再传播一次。

> **口诀**：PROPAGATE 是共享模式下的"多余唤醒令牌"。**它存在的唯一目的是修一个 race condition**。普通业务代码看不到它，但读源码必须知道它的存在意义。

### 2.6 Condition.await 的 transfer

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted()) throw new InterruptedException();
    Node node = addConditionWaiter();          // ① 入 condition 队列
    int savedState = fullyRelease(node);       // ② 完全释放锁，记下 state
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {             // ③ 还在 condition 队列就 park
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;            // ④ 醒来重新拿锁
    if (node.nextWaiter != null) unlinkCancelledWaiters();
    if (interruptMode != 0) reportInterruptAfterWait(interruptMode);
}
```

**逐段点透**：

- `addConditionWaiter()`：创建 ConditionNode，挂到 condition 队列尾。**不进入 sync 队列**。
- `fullyRelease(node)`：把当前 holdCount 一次性全释放（比如 `lock.lock(); lock.lock();` 进 await 要全释放掉）。**返回值是原来的 state**，醒来时按这个数恢复。
- `isOnSyncQueue(node)`：循环判断节点是不是已经被 signal 转移到 sync 队列了。还没转就继续 park。
- `acquireQueued(node, savedState)`：醒来后回到 sync 队列正常争锁，按 savedState 恢复重入次数。

**关键认知**：`await()` 的 park / 醒来路径**完全复用 sync 队列的 `acquireQueued`**。Condition 队列只负责"挂起 + 转移"，真正的"争锁"逻辑只有一份。

> **口诀**：`await` = "完全释放锁 → 进 condition 队列 park → 被 signal 转入 sync 队列 → 走 acquireQueued 重新拿锁 → 恢复 holdCount"。每一步都不能跳。

### 2.7 Condition.signal 的 transferForSignal

```java
public final void signal() {
    if (!isHeldExclusively())                  // signal 必须持锁，否则抛
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null) doSignal(first);
}

private void doSignal(Node first) {
    do {
        if ((firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;                // 摘出 condition 队列
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);    // ★ 转移失败就拿下一个
}

final boolean transferForSignal(Node node) {
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;                           // 节点可能已被取消

    Node p = enq(node);                         // ★ 入 sync 队列尾
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);        // ★ 前驱已 CANCELLED 或 CAS 失败 → 立即叫醒
    return true;
}
```

**逐段点透**：

- `isHeldExclusively`：`signal()` 必须在持锁时调用。这是 `Object.notify()` 必须在 synchronized 块里调用的对应物。
- `transferForSignal` 的 CAS：把节点的 status 从 CONDITION 改成 0。失败说明节点被中断取消了，跳过它继续 signal 下一个。
- `enq(node)`：入到 sync 队列尾。**enq 返回的是新插入节点的前驱**，方便后面给前驱设 SIGNAL。
- 最后的 if：**通常情况下被 signal 的线程不会立即 unpark**，而是依靠它的新前驱（sync 队列里）在释放锁时唤醒它。**只有前驱已 CANCELLED 或 CAS 设 SIGNAL 失败时**才主动 unpark。

为什么要这么设计？`signal()` 的语义是"某个等待者从 condition 队列移到 sync 队列"，**它不一定要立即唤醒**。被 signal 的线程依然要排队拿锁，提前 unpark 反而可能让它"醒来发现还要等"，浪费一次 park/unpark。

> **口诀**：`signal` = "把节点从 condition 队列摘到 sync 队列尾"。**不立即 unpark**，让它跟着 sync 队列的释放节奏来。

### 2.8 ReentrantLock 公平 vs 非公平 —— 一行代码的差别

`ReentrantLock` 内部有两个 Sync 子类：`FairSync` 和 `NonfairSync`。它们的 `tryAcquire` 实现：

```java
// NonfairSync
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {     // ★ 直接 CAS 抢
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}

// FairSync
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&            // ★ 仅这一行差别！
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

**整个差别就是 `hasQueuedPredecessors()` 这一行判断**。

```java
public final boolean hasQueuedPredecessors() {
    Node h, s;
    if ((h = head) != null) {
        if ((s = h.next) == null || s.waiter == null
            || s.prev == null)
            return true;       // 有人正在入队（state 不一致）
        return s.waiter != Thread.currentThread();
    }
    return false;
}
```

**逐段点透**：

- 公平模式："队列里有人在我前面排吗？" 有就乖乖入队
- 非公平模式：直接 CAS 抢，**抢不过就入队**

为什么非公平能快这么多？

考虑场景：线程 T1 释放锁，正在 unpark 队列头 T2。T2 还没醒来这个 100ns 窗口里，新来的线程 T3 调 `lock()`：
- **公平**：T3 看到队列里有 T2，老老实实入队，park
- **非公平**：T3 直接 CAS state，**抢成功就跑**。等 T2 醒来一看 state 已经被占，又 park

非公平模式下**一次 park-unpark 一对的开销可能省掉**（每次约 1~10 微秒）。在高竞争下这个差别可以放大几倍。

代价：T2 可能被插队多次，**理论上有饥饿风险**。但实际生产很少出现 —— 因为只要 T3 自己进入循环也会 park，下一轮 T2 就有机会拿到。

> **口诀**：公平锁多一行 `hasQueuedPredecessors()`。这一行让"插队"不可能，但代价是每次释放都要走 park-unpark 一对。**默认用非公平**，特殊场景才用公平。

### 2.9 ReadWriteLock 的高低位拆分

`ReentrantReadWriteLock` 把 `state` 这一个 int 用来同时表示"读锁数 + 写锁数 + 写锁重入次数"。怎么塞？

```java
static final int SHARED_SHIFT   = 16;
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);          // 65536
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;      // 65535
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;      // 0xFFFF

// 拆分
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }   // 高 16 位 = 读
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }   // 低 16 位 = 写
```

直观图：

```
state (32 bit):
 ┌────────────────────────┬────────────────────────┐
 │  high 16 bit: 读锁个数  │  low 16 bit: 写锁重入数 │
 └────────────────────────┴────────────────────────┘
```

写锁是独占 + 可重入：低 16 位记重入次数。
读锁是共享 + 可重入：高 16 位记**总持有数**（多个线程加在一起）。

但每个读线程**自己的重入次数**也要记，否则 `unlock` 减谁的不知道。AQS 是怎么做的？答案：**每个线程一个 ThreadLocal**。

```java
static final class HoldCounter {
    int count = 0;
    final long tid = LockSupport.getThreadId(Thread.currentThread());
}

static final class ThreadLocalHoldCounter
    extends ThreadLocal<HoldCounter> {
    public HoldCounter initialValue() { return new HoldCounter(); }
}

// Sync 字段
private transient ThreadLocalHoldCounter readHolds;
private transient HoldCounter cachedHoldCounter;     // 缓存最近一个读线程的 counter
```

`readHolds` 是 ThreadLocal，每个线程持有自己的 HoldCounter。`cachedHoldCounter` 是个**热路径优化**：绝大多数情况下"刚刚加锁的就是马上要解锁的"，直接用缓存的 counter 不用查 ThreadLocal。

`tryAcquireShared`（读锁加锁）的简化版：

```java
protected final int tryAcquireShared(int unused) {
    Thread current = Thread.currentThread();
    int c = getState();
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;                                   // 写锁被别人持有 → 失败
    int r = sharedCount(c);
    if (!readerShouldBlock() &&                      // ★ 公平性钩子
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {    // ★ 加 65536（高 16 位 +1）
        // 更新自己的 HoldCounter
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            firstReaderHoldCount++;
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != current.getId())
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    return fullTryAcquireShared(current);            // 慢路径
}
```

**几个反直觉的点**：

- **加锁是 `c + SHARED_UNIT`**（即 `+ 65536`），不是 `+ 1`。因为高 16 位是读锁数。
- `firstReader` 字段缓存的是"第一个读锁持有者"。如果只有一个读线程在反复加解锁（连接池里很常见），这是个零成本的快路径。
- 写锁要拿读锁（**锁降级**）需要特殊处理：先持写锁的状态下 tryAcquireShared 不会被 `exclusiveCount(c) != 0` 拦截（因为 owner 是自己），可以正常加读锁，然后再释放写锁。

写锁加锁就简单多了：

```java
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    if (c != 0) {
        // c != 0 但 w == 0 说明有读锁；w != 0 但 owner 不是自己也失败
        if (w == 0 || current != getExclusiveOwnerThread()) return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        setState(c + acquires);
        return true;
    }
    if (writerShouldBlock() ||                       // 公平性钩子
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

> **口诀**：ReadWriteLock 用一个 int 同时表示读+写。**高 16 位是读锁总数，低 16 位是写锁重入数**。`+SHARED_UNIT` 是关键写法。读重入靠 ThreadLocal 单独记。

---

## 3. 实测数据

### 3.1 公平 vs 非公平：单线程、低竞争、高竞争三档

JMH benchmark：

```java
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@State(Scope.Benchmark)
@Fork(1)
@Warmup(iterations = 3, time = 1)
@Measurement(iterations = 5, time = 2)
public class FairnessBench {

    @Param({"true", "false"})
    public boolean fair;

    @Param({"1", "4", "32"})
    public int threads;

    private ReentrantLock lock;
    private long counter;

    @Setup
    public void setup() {
        lock = new ReentrantLock(fair);
        counter = 0;
    }

    @Benchmark
    @Threads(Threads.MAX)         // 实际由 -t 控制
    public void lockUnlock() {
        lock.lock();
        try {
            counter++;
            Blackhole.consumeCPU(50);
        } finally {
            lock.unlock();
        }
    }
}
```

8 核 Mac M2 上的典型结果：

| 线程数 | 公平 (ops/s) | 非公平 (ops/s) | 非公平 / 公平 |
| --- | --- | --- | --- |
| 1（单线程） | 1.45 亿 | 1.46 亿 | 1.0× |
| 4（低竞争） | 4500 万 | 8300 万 | 1.84× |
| 32（高竞争） | 950 万 | **5800 万** | **6.1×** |

**结论**：
- **单线程下两者无差**：没有竞争，`hasQueuedPredecessors` 直接返回 false
- **低竞争下非公平快约 2 倍**：减少了 park-unpark 一对的开销
- **高竞争下非公平快 5~6 倍**：这是公平锁的"thundering herd 退化"

火焰图差异：公平锁的火焰图里 `LockSupport.park` 占比能到 60%+，非公平降到 20% 以下。

### 3.2 ReentrantLock vs synchronized 在 JDK 21 上

JDK 6 时代 synchronized 经过偏向锁、轻量锁、重量锁三段优化后，在低竞争下与 ReentrantLock 已经无差。JDK 15 起偏向锁默认禁用。

```java
@Benchmark @Group("syncLock")
public void synchronizedLock() {
    synchronized (monitor) { counter++; }
}

@Benchmark @Group("reentrantLock")
public void reentrantLock() {
    lock.lock();
    try { counter++; }
    finally { lock.unlock(); }
}
```

JDK 21、32 线程争抢同一锁：

| 锁类型 | 吞吐 (ops/s) | P99 latency (ns) |
| --- | --- | --- |
| `synchronized` | 5300 万 | 1100 |
| `ReentrantLock`（非公平） | 5800 万 | 950 |
| `ReentrantLock`（公平） | 950 万 | 8200 |
| `StampedLock`（独占） | 6100 万 | 880 |

**结论**：JDK 21 下三者都已"足够好"，性能不再是选择主要因素。**功能差别**才是：
- 需要 `tryLock` / `lockInterruptibly` / `Condition` → ReentrantLock
- 需要乐观读、读多写少 → StampedLock（但要注意不可重入、不响应中断）
- 简单互斥 + 想吃虚拟线程红利 → 用 ReentrantLock，**不要用 `synchronized`**（synchronized 块内做长 IO 会 pin 虚拟线程，见 [线程池实战 案例 ⑧](/article/java/threadpool#案例-虚拟线程的反模式)）

### 3.3 ReadWriteLock vs StampedLock 在读多写少场景

```java
@State(Scope.Benchmark)
public class RwLockBench {
    @Param({"99:1", "90:10", "50:50"})
    public String ratio;

    ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    StampedLock stampedLock = new StampedLock();
    int value = 0;

    @Benchmark @Group("rwLock")
    public int rwRead() {
        rwLock.readLock().lock();
        try { return value; }
        finally { rwLock.readLock().unlock(); }
    }

    @Benchmark @Group("rwLock")
    public void rwWrite() {
        rwLock.writeLock().lock();
        try { value++; }
        finally { rwLock.writeLock().unlock(); }
    }

    @Benchmark @Group("stamped")
    public int stampedOptimisticRead() {
        long stamp = stampedLock.tryOptimisticRead();
        int v = value;
        if (!stampedLock.validate(stamp)) {
            stamp = stampedLock.readLock();
            try { v = value; }
            finally { stampedLock.unlockRead(stamp); }
        }
        return v;
    }

    @Benchmark @Group("stamped")
    public void stampedWrite() {
        long stamp = stampedLock.writeLock();
        try { value++; }
        finally { stampedLock.unlockWrite(stamp); }
    }
}
```

读写比 99:1，64 线程：

| 锁类型 | 总吞吐 (ops/s) | 读 (ops/s) | 写 (ops/s) |
| --- | --- | --- | --- |
| `ReentrantReadWriteLock` | 8200 万 | 8120 万 | 80 万 |
| `StampedLock`（乐观读） | **3.2 亿** | 3.17 亿 | 30 万 |

**StampedLock 乐观读快 4 倍**。原因：`tryOptimisticRead` 不加锁、不上 CAS，纯 volatile 读 + 验证。**几乎零成本**。

但写性能反而下降 —— 写时要让所有正在 validate 的乐观读"作废"，竞争重了反而比 ReadWriteLock 慢。

**陷阱**：StampedLock 是**不可重入**的。同一线程持有读锁时再调 `readLock()` 会**死锁**。这是新手最容易踩的坑（见后面案例 ⑧）。

### 3.4 Semaphore tryAcquire 在不同令牌数下的成功率

```java
Semaphore sem = new Semaphore(permits);
AtomicLong success = new AtomicLong();
AtomicLong fail = new AtomicLong();

ExecutorService pool = Executors.newFixedThreadPool(64);
for (int i = 0; i < 1_000_000; i++) {
    pool.submit(() -> {
        if (sem.tryAcquire()) {
            try { Blackhole.consumeCPU(100); }
            finally { sem.release(); success.incrementAndGet(); }
        } else {
            fail.incrementAndGet();
        }
    });
}
```

| permits | 成功率 | 平均等待时间 (ms) |
| --- | --- | --- |
| 1 | 0.8% | 0（直接 fail） |
| 10 | 7.2% | 0 |
| 64 | 41% | 0 |
| 256 | 98% | 0 |

`tryAcquire()` 是个**非阻塞**调用，permit 不够就立即返回 false。这跟 `acquire()` 完全不同（后者会 park 等）。生产里**必须用 tryAcquire 做"软限流"**（拿不到许可立即降级 / 返回错误），用 `acquire` 等于把限流转化成"拥塞"。

> **口诀**：`tryAcquire(permits)` 是非阻塞限流；`acquire(permits)` 是阻塞排队。**永远优先用前者**。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。

### 案例 ① 公平锁陷阱 —— "压测 5 万 QPS，上线掉到 8 千"

#### 现象

某团队的订单服务上线后，QPS 从压测的 5 万直接掉到 8 千。CPU 利用率反而下降到 30%。dump 看大量线程在 `LockSupport.park`。

代码里有个全局序列号生成器：

```java
// 团队是新手，"觉得公平更好"
private static final ReentrantLock lock = new ReentrantLock(true);
private static long seq = 0;

public static long nextSeq() {
    lock.lock();
    try { return ++seq; }
    finally { lock.unlock(); }
}
```

#### 根因

公平锁的代价（前面 §3.1 数据）：
- 每个线程释放锁后必须 unpark 队列头线程，被唤醒线程必须重新争锁
- 完整流程：`unpark → 上下文切换 → 醒来 → 重新调度 → tryAcquire → 拿到 → setHead`
- 单次 park-unpark 一对在 Linux 上约 5~15 微秒
- 高竞争下每次 lock/unlock 都触发这套开销

非公平锁的优势：**新来的线程直接 CAS 抢占**，能跳过整个 park-unpark 流程。在被压测的"多线程争抢同一资源"场景下差距 5~6 倍。

测试环境压测时线程数较少（比如 16），公平锁的退化还不明显。生产 64 + 线程一上来，立刻露馅。

#### 修复

```java
// 默认就用非公平，除非有"严格 FIFO"的业务需求
private static final ReentrantLock lock = new ReentrantLock();   // ★ 非公平
private static long seq = 0;
```

更好的做法：连续序列号根本不用锁。

```java
private static final AtomicLong seq = new AtomicLong();
public static long nextSeq() { return seq.incrementAndGet(); }
```

如果真的需要"FIFO 严格顺序"（比如某些金融场景按提交时间排序），公平锁是对的，但要做好性能预期：**单点 QPS 上限就是你的 CPU 能 park-unpark 多少次**。

#### 验证

```java
@Test
void fairLockShouldNotBeUsedForHighThroughput() {
    int threads = 64;
    long iterations = 1_000_000;

    long unfairTime = bench(new ReentrantLock(false), threads, iterations);
    long fairTime   = bench(new ReentrantLock(true),  threads, iterations);

    System.out.printf("unfair: %d ms, fair: %d ms, ratio: %.1fx%n",
        unfairTime, fairTime, (double) fairTime / unfairTime);
    // 期望：fair 至少慢 3 倍以上
    assertTrue(fairTime > 3 * unfairTime);
}
```

> **口诀**：**默认用非公平锁**。需要严格 FIFO 才用公平。能用 `AtomicLong` 就别用锁。

---

### 案例 ② tryLock 不释放 —— "异常路径忘 unlock 全局死锁"

#### 现象

某接口偶发"卡死"，监控显示后续所有请求都卡在 `lock.lock()`。重启服务后正常 1 小时，又出现。

#### 根因

```java
// 错的写法
public void process() {
    if (lock.tryLock(5, TimeUnit.SECONDS)) {
        doStep1();             // ← 这里偶尔抛异常
        doStep2();
        lock.unlock();          // ★ 永远到不了
    }
}
```

或者更隐蔽的版本：

```java
public void process() {
    boolean acquired = lock.tryLock();
    try {
        if (acquired) doWork();
    } finally {
        lock.unlock();          // ★ 没拿到锁也调 unlock，抛 IllegalMonitorStateException
    }
}
```

第一个版本：`doStep1()` 抛异常 → unlock 没调到 → 锁永远被这个 worker 持有 → 后续所有请求 block。worker 是池化的，重启服务才能解开。

第二个版本：`tryLock` 返回 false 时也调 unlock。AQS 的 `tryRelease` 检测到当前线程不是 owner，抛 `IllegalMonitorStateException`。

#### 源码层为什么这么严格

```java
// ReentrantLock.Sync.tryRelease
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();   // ★
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

不允许"非持有者 unlock"是有原因的：如果允许，state 就乱了，没法知道谁该真正释放。

#### 修复

**铁律 1**：每个 `lock()` / `tryLock()` 成功后，必须配套 try-finally。

```java
public void process() {
    if (!lock.tryLock(5, TimeUnit.SECONDS)) {
        throw new TimeoutException("acquire lock failed");
    }
    try {
        doStep1();
        doStep2();
    } finally {
        lock.unlock();          // ★ finally 保证一定释放
    }
}
```

**铁律 2**：tryLock 失败的分支不要碰 unlock。

```java
boolean acquired = lock.tryLock();
if (acquired) {
    try {
        doWork();
    } finally {
        lock.unlock();          // ★ 仅在拿到时释放
    }
}
```

**辅助手段**：用静态分析工具（SpotBugs / Error Prone）的 `LockNotBeforeTry` 规则强制检查。

#### 死锁的诊断

线上死锁时，`jstack <pid>` 看：

```
"http-nio-8080-exec-7" #45 daemon prio=5 ...
   java.lang.Thread.State: WAITING (parking)
        at jdk.internal.misc.Unsafe.park(Native Method)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:341)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquire(...)
        at java.util.concurrent.locks.ReentrantLock.lock(...)
        at com.foo.OrderService.process(OrderService.java:42)
        ...

   - waiting on the Condition
        ...
```

如果有多线程互等，jstack 会贴心地标 `Found one Java-level deadlock`。但**单线程持锁不放 + 其他线程等**这种情况 jstack 不会标 deadlock，只能靠人肉看。

#### 验证

```java
@Test(timeout = 3000)
void lockMustBeReleasedOnException() {
    ReentrantLock lock = new ReentrantLock();
    assertThrows(RuntimeException.class, () -> {
        lock.lock();
        try { throw new RuntimeException("boom"); }
        finally { lock.unlock(); }
    });
    assertTrue(lock.tryLock());   // ★ 锁应该是释放的
    lock.unlock();
}
```

> **口诀**：`lock()` 之后立即 try，**finally 永远 unlock**。`tryLock` 失败的路径**绝不能调 unlock**。

---

### 案例 ③ Condition.await 不在 while 循环里 —— "虚假唤醒导致状态错乱"

#### 现象

自研的简易阻塞队列偶尔取出 null，业务报 NPE。代码看起来"没问题"。

```java
public class BadBlockingQueue<T> {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notEmpty = lock.newCondition();
    private final Queue<T> queue = new ArrayDeque<>();

    public T take() throws InterruptedException {
        lock.lock();
        try {
            if (queue.isEmpty()) notEmpty.await();    // ★ if 而不是 while
            return queue.poll();                      // 可能返回 null
        } finally {
            lock.unlock();
        }
    }

    public void put(T item) {
        lock.lock();
        try {
            queue.offer(item);
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }
}
```

#### 根因

两个原因：

1. **虚假唤醒（spurious wakeup）**：JVM 不保证 `LockSupport.park` 一定要被 `unpark` 才返回。OS 层面的实现允许"无原因醒来"，规范要求应用层必须容忍。

2. **多消费者抢一个 signal**：`signal()` 唤醒一个等待者。但等这个被唤醒的线程**回到 sync 队列、重新拿到锁**这个窗口期，**另一个 take 线程可能先拿到锁、先消费掉队列里那个元素**。被唤醒的线程拿到锁后发现队列又空了，但 if 不会再判断，直接 poll → null。

JDK 官方文档明确写：

> An implementation can (and usually does) require that the current thread hold the lock associated with this Condition when this method is called. Implementations must document this precondition...
>
> **The application programmer should always assume that the wait can return spuriously, and use the wait in a loop**.

#### 修复

把 `if` 改成 `while`：

```java
public T take() throws InterruptedException {
    lock.lock();
    try {
        while (queue.isEmpty()) notEmpty.await();    // ★ while 永远再检查
        return queue.poll();
    } finally {
        lock.unlock();
    }
}
```

`Object.wait()` 也是同样规则：永远在 while 里调，不要在 if 里。

JDK 自己的 `ArrayBlockingQueue.take` 就是这么写的：

```java
// java.util.concurrent.ArrayBlockingQueue.take
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0) notEmpty.await();         // ★ while
        return dequeue();
    } finally {
        lock.unlock();
    }
}
```

#### signal vs signalAll

如果你的场景是"多消费者 + 单生产者，每次只能服务一个"，用 `signal()`。如果是"广播事件"或者"同时唤醒多个等待者各自判断条件"，用 `signalAll()`。

`signalAll` 的风险：**惊群效应**。所有线程一起醒来争同一把锁，只有一个能拿到，其他又重新 park。如果消费动作很轻，这个开销可能比 signal + 串行处理还高。

#### 验证

写一个能稳定复现"虚假唤醒"的压测：

```java
@Test
void multiConsumerMustNotReturnNull() throws Exception {
    BadBlockingQueue<Integer> q = new BadBlockingQueue<>();
    int n = 1000;
    AtomicInteger nullCount = new AtomicInteger();

    ExecutorService pool = Executors.newFixedThreadPool(8);
    CountDownLatch latch = new CountDownLatch(n);

    for (int i = 0; i < n; i++) {
        pool.submit(() -> {
            try {
                Integer v = q.take();
                if (v == null) nullCount.incrementAndGet();
            } catch (InterruptedException ignore) {}
            latch.countDown();
        });
    }

    for (int i = 0; i < n; i++) q.put(i);

    latch.await();
    assertEquals(0, nullCount.get());     // 错的实现这里会失败
}
```

> **口诀**：**`await` / `wait` 永远在 while 里**。条件不满足就接着 await。`signal` 只唤醒一个，但被唤醒的线程拿到锁时条件可能又变了。

---

### 案例 ④ Semaphore 资源泄漏 —— "许可永远丢失"

#### 现象

服务跑了几天后告警：某接口的并发数从限流的 100 慢慢降到 80、50、最后 0。重启恢复，几天后再次出现。

#### 根因

代码：

```java
// 限流：最多 100 并发
private final Semaphore sem = new Semaphore(100);

public Result handle(Request r) {
    sem.acquire();              // ① 拿许可
    Result result = remoteCall(r);    // ② 这里可能抛异常
    sem.release();              // ③ 异常时永远到不了
    return result;
}
```

`remoteCall` 抛异常时 ③ 不执行，**许可永远丢了**。每次异常永久减少 1 个许可，几天后耗尽。

更隐蔽的版本：

```java
public Result handle(Request r) {
    if (!sem.tryAcquire(100, TimeUnit.MILLISECONDS)) {
        throw new TooManyRequestsException();
    }
    try {
        return remoteCall(r);
    } finally {
        sem.release();
    }
}

// 调用方
try {
    return service.handle(r);
} catch (InterruptedException e) {       // ★ tryAcquire 抛了 InterruptedException
    Thread.currentThread().interrupt();
    return fallback();
}
```

`tryAcquire` 抛 InterruptedException 时**没拿到许可**。但调用方如果在外层 try-catch 里也写了 `sem.release()`，就会"凭空释放一个不存在的许可"，让 state 多 1。下次重启服务时正常，但被中断越多 state 越大，**实际并发限流被破坏**。

Semaphore 的 release **不验证持有者**：

```java
// Semaphore.Sync.tryReleaseShared (简化)
protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        int next = current + releases;
        if (next < current) throw new Error("Maximum permit count exceeded");
        if (compareAndSetState(current, next))
            return true;
    }
}
```

它只是 CAS 加 state。**任何线程都能 release**，且**没拿过的也能 release**（这是 Semaphore 设计上的灵活性 —— 它允许"凭空 release"以便实现某些动态扩容场景）。

这跟 ReentrantLock 的 tryRelease 严格校验 owner 完全不同。

#### 修复

**铁律**：acquire 成功后立即 try，finally 里 release。**acquire 失败的路径不要 release**。

```java
public Result handle(Request r) throws InterruptedException {
    if (!sem.tryAcquire(100, TimeUnit.MILLISECONDS)) {
        throw new TooManyRequestsException();
    }
    try {
        return remoteCall(r);
    } finally {
        sem.release();          // ★ 仅在 acquire 成功的分支
    }
}
```

更安全的封装（推荐用于团队代码规范）：

```java
public static <T> T withPermit(Semaphore sem, Callable<T> task) throws Exception {
    sem.acquire();
    try { return task.call(); }
    finally { sem.release(); }
}

// 用法
return withPermit(sem, () -> remoteCall(r));
```

或者用 `tryAcquire(timeout)` 的版本：

```java
public static <T> T withPermit(Semaphore sem, long timeout, TimeUnit unit,
                               Callable<T> task) throws Exception {
    if (!sem.tryAcquire(timeout, unit)) {
        throw new TimeoutException();
    }
    try { return task.call(); }
    finally { sem.release(); }
}
```

#### 监控

```java
// 暴露当前可用许可数
Gauge.builder("semaphore.available", sem, Semaphore::availablePermits)
    .register(meterRegistry);

// 暴露当前等待者数
Gauge.builder("semaphore.queueLength", sem, Semaphore::getQueueLength)
    .register(meterRegistry);
```

告警：`semaphore.available` 长时间低于初始值 50% → 排查泄漏。

#### 验证

```java
@Test
void semaphoreShouldNotLeakOnException() {
    Semaphore sem = new Semaphore(5);

    for (int i = 0; i < 100; i++) {
        try {
            sem.acquire();
            try {
                if (Math.random() < 0.5) throw new RuntimeException("boom");
            } finally { sem.release(); }
        } catch (Exception ignore) {}
    }
    assertEquals(5, sem.availablePermits());   // ★ 任何异常后都应该是 5
}
```

> **口诀**：Semaphore 的 acquire / release **必须用 try-finally 配对**。Semaphore 不校验持有者，**乱 release 会让 state 越加越大**，破坏限流。

---

### 案例 ⑤ ReadWriteLock 写饥饿 —— "长时间读锁排队，写永远拿不到"

#### 现象

某缓存服务用 `ReentrantReadWriteLock` 保护，读多写少。某次部署后，**写操作 TPS 直接归零**，读 TPS 正常。监控看写线程全部 `WAITING`。

#### 根因

`ReentrantReadWriteLock` 默认是**非公平**的。非公平模式下读锁的获取规则是"只要写锁没人持有就直接抢"。

但在高频读场景下：
- T1 持有读锁
- T2 想拿写锁，进队列等
- T3 来拿读锁。**非公平模式下 T3 直接抢成功**，因为写锁没人持有，T2 还在队列里
- T4、T5、T6... 持续来拿读锁，每个都"插队"成功
- **T2 的写锁永远拿不到**

源码依据 —— `NonfairSync.readerShouldBlock`：

```java
final boolean readerShouldBlock() {
    return apparentlyFirstQueuedIsExclusive();
}

final boolean apparentlyFirstQueuedIsExclusive() {
    Node h, s;
    return (h = head) != null &&
           (s = h.next) != null &&
           !s.isShared() &&
           s.thread != null;
}
```

非公平模式下，**只有当队列头是个写锁等待者时**才让读锁排队。但只要队列头不是写锁（比如读锁等待者在写锁前面），后续读锁继续插队。

如果切到**公平模式**：

```java
// FairSync
final boolean readerShouldBlock() {
    return hasQueuedPredecessors();   // 队列里有人在前面就乖乖排
}
```

写饥饿就解决了。但公平模式吞吐降低。

#### 修复

方案 A：**改用公平模式**

```java
ReadWriteLock lock = new ReentrantReadWriteLock(true);   // ★ fair = true
```

代价：吞吐降低（前面 §3.1 数据，可能 2-5 倍）。读多写少场景一般可接受。

方案 B：**用 StampedLock 替代**

```java
StampedLock lock = new StampedLock();

// 读
long stamp = lock.tryOptimisticRead();
T value = data.get();
if (!lock.validate(stamp)) {
    stamp = lock.readLock();
    try { value = data.get(); }
    finally { lock.unlockRead(stamp); }
}

// 写
long stamp = lock.writeLock();
try { data.set(newValue); }
finally { lock.unlockWrite(stamp); }
```

StampedLock 内部对写有"优先权" —— 一旦有写在等，新读必须排队。且乐观读不上锁，几乎零成本。

方案 C：**重新设计：用 Copy-on-Write 容器**

如果数据变更频率非常低（比如配置类）：

```java
// 完全无锁读
private volatile Map<String, Config> configs = new HashMap<>();

public Config get(String k) { return configs.get(k); }

public synchronized void update(String k, Config v) {
    Map<String, Config> next = new HashMap<>(configs);
    next.put(k, v);
    configs = next;        // volatile 写
}
```

读路径完全无锁，写路径每次拷贝整个 Map。**数据量小、写罕见**才适用。

#### 验证

写一个能复现写饥饿的压测：

```java
@Test
void writerShouldNotStarve() throws Exception {
    ReadWriteLock lock = new ReentrantReadWriteLock(/* fair */ false);
    AtomicInteger reads = new AtomicInteger();
    AtomicInteger writes = new AtomicInteger();
    AtomicBoolean stop = new AtomicBoolean();

    // 32 个读线程持续读
    ExecutorService readPool = Executors.newFixedThreadPool(32);
    for (int i = 0; i < 32; i++) {
        readPool.submit(() -> {
            while (!stop.get()) {
                lock.readLock().lock();
                try { reads.incrementAndGet(); Thread.sleep(1); }
                catch (Exception e) { break; }
                finally { lock.readLock().unlock(); }
            }
        });
    }

    // 1 个写线程
    new Thread(() -> {
        while (!stop.get()) {
            lock.writeLock().lock();
            try { writes.incrementAndGet(); }
            finally { lock.writeLock().unlock(); }
        }
    }).start();

    Thread.sleep(5000);
    stop.set(true);

    System.out.printf("reads: %d, writes: %d%n", reads.get(), writes.get());
    // 非公平：writes 可能 0~5
    // 公平 / StampedLock：writes 应在数百以上
    assertTrue(writes.get() > 100);
}
```

> **口诀**：ReadWriteLock 默认非公平 → **读多写少场景写会饥饿**。要么用公平模式，要么换 StampedLock，要么换无锁数据结构。

---

### 案例 ⑥ 锁降级顺序错误 —— "释放写锁前没拿读锁"

#### 现象

某缓存的"加载并缓存"逻辑，偶发"返回了别人改过的旧值"，业务表现为脏数据。

代码：

```java
// 错的写法
public V get(K key) {
    V v = cache.get(key);
    if (v != null) return v;

    rwLock.writeLock().lock();
    try {
        v = cache.get(key);                  // double-check
        if (v == null) {
            v = loadFromDb(key);
            cache.put(key, v);
        }
    } finally {
        rwLock.writeLock().unlock();          // ★ 释放写锁
    }

    rwLock.readLock().lock();
    try {
        return cache.get(key);                // ★ 这里可能读到别人的修改
    } finally {
        rwLock.readLock().unlock();
    }
}
```

放写锁后，到拿读锁这个窗口期，**别的线程可能修改了 cache**。返回值可能不是你刚 put 进去的 v。

#### 根因

ReadWriteLock 支持**锁降级（写 → 读）但不支持锁升级（读 → 写）**。锁降级的正确姿势是：

```
持有写锁
  ↓
持有写锁 + 读锁 (在持有写锁时调 readLock().lock())
  ↓
持有读锁 (释放写锁)
```

**关键**：**先拿读锁，再放写锁**。在写锁还持有时拿读锁是合法的（owner 是自己），且这个过程是原子的。任何其他线程在你"持读锁"期间都不能拿写锁修改数据。

如果反过来"先放写锁，再拿读锁"：

```
持有写锁
  ↓
[窗口] 别的线程可以拿写锁修改数据
  ↓
持有读锁（数据可能已被改）
```

#### 修复

```java
public V get(K key) {
    V v = cache.get(key);
    if (v != null) return v;

    rwLock.writeLock().lock();
    try {
        v = cache.get(key);
        if (v == null) {
            v = loadFromDb(key);
            cache.put(key, v);
        }
        rwLock.readLock().lock();             // ★ 先拿读锁
    } finally {
        rwLock.writeLock().unlock();           // ★ 后放写锁
    }

    try {
        return v;                              // 不需要再 cache.get
    } finally {
        rwLock.readLock().unlock();
    }
}
```

**或者更简单：直接返回 v，根本不用第二次拿锁**。

为什么 RWLock 不支持升级？因为多个读线程同时想升级会死锁：

```
T1: 持读锁，请求写锁 → 等其他读释放
T2: 持读锁，请求写锁 → 等其他读释放
两个互等 → 死锁
```

JDK 设计者直接禁止了这个操作 —— 在持读锁时调 `writeLock().lock()` 会**永久阻塞**。

#### 验证

```java
@Test
void lockDowngradeMustBeAtomic() throws Exception {
    ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    AtomicInteger value = new AtomicInteger(0);
    AtomicInteger conflict = new AtomicInteger();

    Runnable correctDowngrade = () -> {
        rwLock.writeLock().lock();
        try {
            int written = value.incrementAndGet();
            rwLock.readLock().lock();             // ★ 先拿读
            try {
                if (value.get() != written) conflict.incrementAndGet();
            } finally { /* 读锁稍后解 */ }
        } finally {
            rwLock.writeLock().unlock();           // ★ 后放写
        }
        // 这里持读锁
        rwLock.readLock().unlock();
    };

    ExecutorService pool = Executors.newFixedThreadPool(16);
    for (int i = 0; i < 10000; i++) pool.submit(correctDowngrade);
    pool.shutdown();
    pool.awaitTermination(30, TimeUnit.SECONDS);

    assertEquals(0, conflict.get());     // 不能有冲突
}
```

> **口诀**：锁降级 = "拿读 → 放写"，**顺序不能反**。不支持锁升级，反过来死锁。

---

### 案例 ⑦ 自定义 Synchronizer —— "tryAcquire 返回值理解错"

#### 现象

某团队自己写了个限流器基于 AQS。压测时发现限流总是不准，要么放过更多，要么卡死。

代码：

```java
// 错的实现
public class CustomLimiter {
    private final Sync sync;

    public CustomLimiter(int permits) {
        sync = new Sync(permits);
    }

    public void acquire() { sync.acquireShared(1); }
    public void release() { sync.releaseShared(1); }

    private static class Sync extends AbstractQueuedSynchronizer {
        Sync(int permits) { setState(permits); }

        @Override
        protected int tryAcquireShared(int arg) {
            int available = getState();
            if (available >= arg) {
                if (compareAndSetState(available, available - arg)) {
                    return 1;            // ★ 返回 1 表示成功？
                }
            }
            return -1;
        }

        @Override
        protected boolean tryReleaseShared(int arg) {
            for (;;) {
                int current = getState();
                int next = current + arg;
                if (compareAndSetState(current, next)) return true;
            }
        }
    }
}
```

#### 根因

`tryAcquireShared` 的返回值语义被严重误解。看 JDK 文档原话：

> Returns:
> a negative value on failure;
> zero if acquisition in shared mode succeeded but no subsequent shared-mode acquire can succeed;
> a positive value if acquisition in shared mode succeeded and subsequent shared-mode acquires might also succeed, in which case a subsequent waiting thread must check availability.

三段语义：
- **< 0**：失败（永远是 -1 用来表示失败）
- **= 0**：成功，但**没有剩余资源**（不传播唤醒后继）
- **> 0**：成功，**还有剩余资源**，会触发 `setHeadAndPropagate`，唤醒后继

错的实现里写"成功就返回 1"，意味着每次 acquire 成功都触发传播。**这本身可能是对的**（只要逻辑能正常工作），但如果业务逻辑要求"剩 0 时不再唤醒后继"，应该返回 `available - arg`：

```java
protected int tryAcquireShared(int arg) {
    for (;;) {
        int available = getState();
        int remaining = available - arg;
        if (remaining < 0) return remaining;        // ★ < 0 失败
        if (compareAndSetState(available, remaining))
            return remaining;                        // ★ 返回剩余（0 = 成功不传播；> 0 = 成功且传播）
    }
}
```

`Semaphore.NonfairSync.tryAcquireShared` 就是这么写的：

```java
final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

返回 `remaining` 而不是 1。这是 Semaphore 高效的核心。

#### 另一个常见错误：忘了循环

```java
// 错：CAS 失败直接返回 -1
protected int tryAcquireShared(int arg) {
    int available = getState();
    if (available >= arg) {
        if (compareAndSetState(available, available - arg)) return 1;
    }
    return -1;
}
```

CAS 失败时把"还没拿到"当成"拿不到"。AQS 会因此把这个线程入队 park。但实际上下次重试可能就成功了，白白损失一次"快路径"。

正确写法：失败重试。

```java
protected int tryAcquireShared(int arg) {
    for (;;) {
        int available = getState();
        if (available < arg) return available - arg;     // 真的不够，返回负数
        if (compareAndSetState(available, available - arg))
            return available - arg;
    }
}
```

#### 修复

完整的正确实现（一个简单的 N 元限流器）：

```java
public class SimpleLimiter {
    private final Sync sync;

    public SimpleLimiter(int permits) {
        sync = new Sync(permits);
    }

    public void acquire(int n) throws InterruptedException {
        sync.acquireSharedInterruptibly(n);
    }

    public boolean tryAcquire(int n) {
        return sync.tryAcquireSharedNanos(n, 0);
    }

    public void release(int n) { sync.releaseShared(n); }

    public int availablePermits() { return sync.getState(); }

    private static class Sync extends AbstractQueuedSynchronizer {
        Sync(int permits) { setState(permits); }

        @Override
        protected int tryAcquireShared(int n) {
            for (;;) {
                int available = getState();
                int remaining = available - n;
                if (remaining < 0) return remaining;
                if (compareAndSetState(available, remaining))
                    return remaining;
            }
        }

        @Override
        protected boolean tryReleaseShared(int n) {
            for (;;) {
                int current = getState();
                int next = current + n;
                if (next < current) throw new Error("Maximum exceeded");
                if (compareAndSetState(current, next)) return true;
            }
        }
    }
}
```

#### 验证

```java
@Test
void customLimiterShouldRespectCapacity() throws Exception {
    int capacity = 10;
    SimpleLimiter limiter = new SimpleLimiter(capacity);
    AtomicInteger concurrent = new AtomicInteger();
    AtomicInteger maxConcurrent = new AtomicInteger();

    int n = 1000;
    CountDownLatch latch = new CountDownLatch(n);
    ExecutorService pool = Executors.newFixedThreadPool(64);

    for (int i = 0; i < n; i++) {
        pool.submit(() -> {
            try {
                limiter.acquire(1);
                int c = concurrent.incrementAndGet();
                maxConcurrent.updateAndGet(m -> Math.max(m, c));
                Thread.sleep(1);
            } catch (Exception ignore) {}
            finally {
                concurrent.decrementAndGet();
                limiter.release(1);
                latch.countDown();
            }
        });
    }

    latch.await();
    assertEquals(capacity, limiter.availablePermits());
    assertTrue(maxConcurrent.get() <= capacity, "max=" + maxConcurrent.get());
}
```

> **口诀**：`tryAcquireShared` 返回的是"剩余资源数"（语义化版本：< 0 失败，= 0 成功不传播，> 0 成功且传播）。**不是 boolean**。

---

### 案例 ⑧ StampedLock 乐观读循环 CPU 飙升

#### 现象

某接口用 StampedLock 实现"读多写少"的缓存。线上某次写流量突增（10 倍），监控看接口 CPU 从 30% 飙到 95%。重启后正常一天又复发。

代码：

```java
private final StampedLock lock = new StampedLock();
private CacheData data = ...;

public CacheData read() {
    long stamp = lock.tryOptimisticRead();
    CacheData result = data;
    while (!lock.validate(stamp)) {              // ★ 死循环重试
        stamp = lock.tryOptimisticRead();
        result = data;
    }
    return result;
}
```

#### 根因

`tryOptimisticRead` 拿到的 stamp 在写锁被获取时立即失效（validate 返回 false）。

正常场景下写很罕见，乐观读的 validate 几乎都成功，CPU 几乎零。但当**写流量持续高**时：

- T1 拿到 stamp1，data 一致
- 写线程 T2 拿写锁修改了 data，stamp 全部作废
- T1 validate 失败 → 重新 tryOptimisticRead
- 写线程 T3 又来一次，stamp 又失效
- T1 永远在 spin

每次 spin 几纳秒，但**完全占满 CPU**。这是 spin lock 的经典退化场景：高竞争下 spin 不如 park。

#### 修复

**正确姿势**：乐观读失败一次就退化到悲观读锁。

```java
public CacheData read() {
    long stamp = lock.tryOptimisticRead();
    CacheData result = data;
    if (!lock.validate(stamp)) {
        stamp = lock.readLock();              // ★ 退化到悲观读锁，会 park
        try { result = data; }
        finally { lock.unlockRead(stamp); }
    }
    return result;
}
```

JDK 文档里的标准模板就是这样写的，不要自己加 while 循环。

更进一步：如果你的 data 是个对象（不是 long），乐观读期间**不要做"基于 data 的判断"**，否则可能读到撕裂状态：

```java
// 错：data 是个对象，乐观读期间字段可能不一致
long stamp = lock.tryOptimisticRead();
String name = data.name;         // 可能正读一半，data 被换了
int age = data.age;              // 可能是新的 data 的字段
if (!lock.validate(stamp)) { ... }
```

正确：**乐观读期间只读单个 volatile 字段或 immutable 引用**。

```java
long stamp = lock.tryOptimisticRead();
CacheData snapshot = data;       // ★ 只取引用
if (!lock.validate(stamp)) { ... }
return snapshot;                 // 用快照，data 是 immutable
```

如果 data 是可变对象，必须把整个对象 copy 出来或者用悲观读。

#### 另一个 StampedLock 的常见坑：不可重入

```java
StampedLock lock = new StampedLock();

long stamp = lock.readLock();
try {
    callOther();   // 这个方法内部又调 lock.readLock() → ★ 死锁
} finally {
    lock.unlockRead(stamp);
}
```

StampedLock 是**不可重入**的。同一线程持有读锁时再调 `readLock()` 会 park 等自己释放，永久死锁。这跟 ReentrantLock 完全不同。

也不响应中断（除非用 `readLockInterruptibly()`）。从 ReentrantLock 切到 StampedLock 必须**整个调用链审计一遍**。

#### 修复（不可重入）

只能避免在持锁时调用可能再次拿锁的代码。或者在持锁时把数据拿出来再处理：

```java
long stamp = lock.readLock();
CacheData snapshot;
try { snapshot = data; }
finally { lock.unlockRead(stamp); }

callOther(snapshot);   // ★ 释放锁后再调用
```

#### 验证

模拟高写流量看 CPU：

```java
@Test
void stampedLockHighWriteShouldNotSpinForever() throws Exception {
    StampedLock lock = new StampedLock();
    AtomicReference<String> data = new AtomicReference<>("init");
    AtomicBoolean stop = new AtomicBoolean();

    // 一个写线程持续写
    new Thread(() -> {
        while (!stop.get()) {
            long s = lock.writeLock();
            try { data.set("v" + System.nanoTime()); }
            finally { lock.unlockWrite(s); }
        }
    }).start();

    // 测试错的实现：CPU 100%
    long start = System.currentTimeMillis();
    int iterations = 0;
    while (System.currentTimeMillis() - start < 100) {
        long stamp = lock.tryOptimisticRead();
        String v = data.get();
        while (!lock.validate(stamp)) {     // ★ 错的写法，spin 满
            stamp = lock.tryOptimisticRead();
            v = data.get();
            iterations++;
        }
    }
    stop.set(true);
    System.out.println("optimistic spin iterations: " + iterations);
    // 错的写法 iterations 可能上百万
}
```

> **口诀**：StampedLock **乐观读失败一次就退化到悲观读**，不要 while spin。**StampedLock 不可重入、不响应中断**，不能照搬 ReentrantLock 的代码模式。

---

### 案例 ⑨ InterruptedException 处理不当 —— "抛了之后没把 interrupt 标志重新设上"

#### 现象

某业务用了 `BlockingQueue.take()`，团队里有个新人写：

```java
public Optional<Task> nextTask() {
    try {
        return Optional.of(queue.take());
    } catch (InterruptedException e) {
        return Optional.empty();          // ★ 吞了
    }
}
```

调用方在循环里调用：

```java
while (running) {
    nextTask().ifPresent(this::process);
}
```

服务收到 SIGTERM，调 `pool.shutdownNow()`，预期所有 worker 立即停。结果发现 worker 没停 —— 还在循环里。

#### 根因

`take()` 抛 `InterruptedException` 时，**JVM 会把 Thread 的中断标志清掉**（这是 InterruptedException 抛出的副作用）。

`nextTask()` 吃掉异常返回 `Optional.empty()`，**没把中断标志重新设上**。`while (running)` 看不到中断，继续循环。

`shutdownNow()` 内部依赖 `Thread.interrupt()` 通知线程退出。但这种代码**完全把中断信号过滤掉了**，shutdownNow 形同虚设。

#### Java 中断的核心规则

JDK 文档明确：

> When a thread checks for an interrupt by invoking the static method `Thread.interrupted()`, interrupt status is cleared.
>
> The non-static `isInterrupted()` method, which is used by one thread to query the interrupt status of another, does not change the interrupt status flag.
>
> By convention, **any method that exits by throwing an `InterruptedException` clears interrupt status when it does so**.

简化版规则：
- `wait()` / `sleep()` / `take()` / `await()` 等抛 InterruptedException 时，**清掉中断标志**
- 你抓到 InterruptedException 后，**有责任重新设上**（除非你确实想吞掉这个信号，且要明确知道为什么）
- `Thread.interrupted()` 是"测试 + 清"，`isInterrupted()` 是"只测试"

#### 修复

**铁律**：抓到 InterruptedException 后，**默认必须重新 interrupt**：

```java
public Optional<Task> nextTask() {
    try {
        return Optional.of(queue.take());
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();    // ★ 重新设上
        return Optional.empty();
    }
}
```

或者**直接往上抛**，让外层处理：

```java
public Task nextTask() throws InterruptedException {
    return queue.take();
}
```

吞掉 InterruptedException **只在你完全控制线程生命周期且明确知道不需要响应中断时**才允许（极罕见）。

调用方循环也要响应中断：

```java
while (running && !Thread.currentThread().isInterrupted()) {
    nextTask().ifPresent(this::process);
}
```

#### Future.get / Lock.lockInterruptibly 等也是同样规则

```java
try {
    Result r = future.get(5, TimeUnit.SECONDS);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();   // ★ 重新设
    throw new RuntimeException(e);
} catch (TimeoutException e) {
    future.cancel(true);
    throw new RuntimeException(e);
} catch (ExecutionException e) {
    throw new RuntimeException(e.getCause());
}
```

特别注意：`Future.cancel(true)` 也是发 interrupt 信号。如果任务里没响应中断，cancel 不能真正停掉任务（只是把 future 状态标为 CANCELLED）。

#### 一个反直觉的细节：lock() 不响应中断

```java
ReentrantLock lock = new ReentrantLock();
lock.lock();       // 不响应中断
lock.lockInterruptibly();   // 响应
```

前面 §2.1 看过，`lock()` 内部把中断"吞"了，最后用 `selfInterrupt()` 把标志重新设上。所以 **`lock()` 期间被中断不会立即返回**，必须等拿到锁后你自己检查 `Thread.interrupted()` 才能感知。

如果你的业务在意"等锁时立即响应中断"，必须用 `lockInterruptibly()`。

#### 验证

```java
@Test
void interruptShouldStopWorker() throws Exception {
    BlockingQueue<Task> q = new LinkedBlockingQueue<>();
    AtomicInteger processed = new AtomicInteger();

    Thread worker = new Thread(() -> {
        while (!Thread.currentThread().isInterrupted()) {
            try {
                Task t = q.take();
                processed.incrementAndGet();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();   // ★ 重新设上
                return;
            }
        }
    });

    worker.start();
    Thread.sleep(100);
    worker.interrupt();
    worker.join(1000);
    assertFalse(worker.isAlive());     // ★ 必须能停
}
```

> **口诀**：抓到 `InterruptedException` 默认必须 `Thread.currentThread().interrupt()`。要么重新抛、要么重设标志，**绝不能默默吞**。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 锁选型层

- [ ] 默认用 `ReentrantLock(false)` 非公平锁，需要严格 FIFO 才用公平
- [ ] 简单计数 / CAS 场景优先用 `AtomicXxx`，不要用锁
- [ ] 读多写少（读 / 写 > 10:1）评估 `StampedLock`，但务必确认无重入、不响应中断的限制
- [ ] 不变数据 / 极少更新用 `volatile` + Copy-on-Write，零锁读
- [ ] JDK 21+ 虚拟线程下，`synchronized` 包长 IO 会 pin，**用 ReentrantLock 替代**

### 加锁解锁层

- [ ] 每个 `lock()` / `tryLock()` 成功后立即 try-finally
- [ ] `tryLock` 失败的分支**不能调 unlock**
- [ ] 所有 `Semaphore.acquire` / `release` try-finally 配对
- [ ] `Condition.await` 永远在 while 循环里（防虚假唤醒）
- [ ] `signal` vs `signalAll` 选型：单消费用 signal，广播用 signalAll，注意惊群

### Read-Write 层

- [ ] 读多写少且写不频繁可用 `ReentrantReadWriteLock`，**评估写饥饿**
- [ ] 锁降级顺序：**先拿读锁，再放写锁**
- [ ] 锁升级（读 → 写）会死锁，必须先放读锁再拿写锁
- [ ] 读多场景写 TPS 接近 0 时检查是否需要切公平模式或换 StampedLock

### 中断层

- [ ] `InterruptedException` 抓到后默认 `Thread.currentThread().interrupt()`
- [ ] 业务循环用 `while (!Thread.currentThread().isInterrupted())` 而非 `while (true)`
- [ ] `lock()` 不响应中断，需要响应必须用 `lockInterruptibly()`
- [ ] `Future.get` 抓中断后调 `future.cancel(true)`

### 自定义同步器层

- [ ] `tryAcquireShared` 返回**剩余资源数**，不是 boolean（< 0 失败、= 0 成功不传播、> 0 成功且传播）
- [ ] CAS 失败要循环重试，不能直接返回失败
- [ ] `tryRelease` 必须返回 boolean 表示是否完全释放（决定要不要唤醒后继）
- [ ] `setState` 仅用于初始化或持锁时；并发修改用 `compareAndSetState`

### 监控层

- [ ] 关键锁暴露 `getQueueLength` / `getHoldCount` 监控
- [ ] Semaphore 暴露 `availablePermits()` 监控泄漏
- [ ] 长时间持锁告警：`getHoldCount > N` 持续 X 秒
- [ ] jstack 定期采样，检查 `WAITING (parking)` 累积

### 死锁防御层

- [ ] 多锁场景全局统一加锁顺序
- [ ] 高频热点锁拆分（按 key 哈希、按用户 ID 分段）
- [ ] 优先用 `tryLock(timeout)` 而非 `lock()`，超时即降级
- [ ] 不在持锁时调用未知代码（回调、SPI、远程）

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### VarHandle 替代 Unsafe

JDK 9 引入 `VarHandle`（[JEP 193](https://openjdk.org/jeps/193)），AQS 内部从 `sun.misc.Unsafe` 迁到 `VarHandle`。区别：
- `Unsafe` 是内部 API，受 JEP 260 模块化限制
- `VarHandle` 是公开 API，且支持更多内存语义（`opaque` / `acquire-release` / `volatile`）
- 性能上 JIT 优化后两者无差

读 AQS 源码时如果用 JDK 8 的版本看 Unsafe 路径，迁移到 JDK 17+ 是看 VarHandle 路径。算法本身完全没变。

### JEP 444 虚拟线程对 AQS 行为的影响

虚拟线程 park / unpark 时不真的挂 OS 线程，而是把整个 continuation"卸载"到 carrier thread 之外。AQS 的 `LockSupport.park` 在虚拟线程上语义不变，但底层实现切到 continuation。

关键问题：
- AQS 内部所有 park 都已经支持虚拟线程（无需修改）
- `synchronized` 块内 park 会 **pin** 虚拟线程（JDK 21）→ 用 ReentrantLock 替代
- JEP 491（JDK 24+）修复 synchronized pin

### JDK 21 ForkJoinPool 的 ManagedBlocker

`ForkJoinPool.managedBlock(ManagedBlocker)` 让 ForkJoin 任务里的阻塞调用通知池"我要阻塞了，请补一个 worker"。AQS 在某些路径（`acquire` 在 ForkJoinWorkerThread 上调用时）会自动走 managedBlock。但自定义同步器需要自己实现 ManagedBlocker。

### Doug Lea 论文 The java.util.concurrent Synchronizer Framework

[原论文 PDF](https://gee.cs.oswego.edu/dl/papers/aqs.pdf)（2005）。核心论点：
- "最小化 contention"是设计目标，不是"最小化 latency"
- 选 CLH 而非 MCS：CLH 更适合"队列实现 + 取消支持"
- park 的"虚假唤醒"在 Java 规范里是显式允许的，必须循环检查
- 公平 vs 非公平的取舍：作者倾向非公平 + 显式 yield

### C++ folly 库的 SharedMutex

Facebook 的 [folly::SharedMutex](https://github.com/facebook/folly/blob/main/folly/SharedMutex.h) 是 C++ 里读写锁的工业级实现。对比 Java RWLock：
- 用 atomic + futex 而非 ReentrantLock
- 支持"锁升级 + 自动降级"（folly 通过引入 upgrade_lock 解决了死锁）
- 写优先级可配置
- 性能在某些场景比 Java RWLock 快 5-10 倍

如果你写 C++ 项目，看 folly 的实现能反过来加深对 Java AQS 的理解。

### 无锁数据结构（Hazard Pointer / RCU）何时优于 AQS

锁机制的根本开销是"上下文切换 + 内存屏障"。极端场景下无锁数据结构（lock-free / wait-free）更快：
- **Hazard Pointer**（Maged Michael 2004）：解决 lock-free 数据结构里的"安全回收"问题
- **RCU (Read-Copy-Update)**：Linux 内核里的核心同步原语，读端零开销
- **Disruptor**（LMAX）：单生产单消费 + 环形缓冲区，延迟 < 微秒

适用场景：极端低延迟（高频交易、内核态）、读端极致优化（路由表）。一般业务用 AQS / volatile + immutable 已足够。

### StampedLock 与 OptimisticLock 模式

StampedLock 的乐观读是数据库 OCC（乐观并发控制）思想的内存版本。延伸阅读：
- [The OCC Paper - Kung & Robinson 1981](https://dl.acm.org/doi/10.1145/319566.319567)
- MySQL 的乐观锁 / version 列实现
- Hibernate `@Version` 注解
- ETCD / Consul 的 CAS 操作

---

## 7. 引用

- OpenJDK 21 源码：[AbstractQueuedSynchronizer.java](https://github.com/openjdk/jdk21u/blob/master/src/java.base/share/classes/java/util/concurrent/locks/AbstractQueuedSynchronizer.java)
- OpenJDK 21 源码：[ReentrantLock.java](https://github.com/openjdk/jdk21u/blob/master/src/java.base/share/classes/java/util/concurrent/locks/ReentrantLock.java)
- OpenJDK 21 源码：[ReentrantReadWriteLock.java](https://github.com/openjdk/jdk21u/blob/master/src/java.base/share/classes/java/util/concurrent/locks/ReentrantReadWriteLock.java)
- OpenJDK 21 源码：[Semaphore.java](https://github.com/openjdk/jdk21u/blob/master/src/java.base/share/classes/java/util/concurrent/Semaphore.java)
- Doug Lea, *The java.util.concurrent Synchronizer Framework*（2005）—— AQS 论文 [PDF](https://gee.cs.oswego.edu/dl/papers/aqs.pdf)
- Craig, T. S. (1993), *Building FIFO and priority-queueing spin locks from atomic swap* —— CLH 锁原始论文
- JEP 193: Variable Handles
- JEP 444: Virtual Threads
- JEP 491: Synchronize Virtual Threads without Pinning
- Brian Goetz, *Java Concurrency in Practice* —— 并发圣经
- 《Java 并发编程的艺术》—— 方腾飞 / 魏鹏 / 程晓明

---

> 上一篇综述：[并发编程总览](/article/java/concurrent#3-aqs必背) ｜ 配套：[线程池生产实战](/article/java/threadpool) ｜ [JMM](/article/jvm/jmm)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
