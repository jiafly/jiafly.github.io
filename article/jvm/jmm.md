# Java 内存模型（速记）

> JMM 不是 JVM 内存结构，而是 **多线程语义规范**。它定义了"线程如何看到彼此的写入"，是 `volatile` / `synchronized` / `final` / `Atomic*` 的理论基础。
>
> happens-before 8 条规则、x86/ARM 屏障、DCL 重排序证明、伪共享、9 个并发事故 → **[JMM 深入（深文）](/article/jvm/jmm-deep)**

---

## 1. 一句话八股

> **JMM 通过 happens-before 规则定义了一组先后可见性约束**，并通过内存屏障禁止特定的指令重排，从而保证**可见性、原子性、有序性**这"并发三特性"。

---

## 2. 为什么需要 JMM？

CPU 远快于内存，于是有 **L1/L2/L3 多级缓存**：每个核心读写自己的缓存，再异步回写主内存。

```
   Core1 [L1] [L2]              Core2 [L1] [L2]
        \              /
         共享 L3
            │
        主内存 (RAM)
```

带来三个问题：
- **可见性**：Core1 写了 `x=1`，Core2 不一定立刻看到；
- **原子性**：`i++` 是 read-modify-write，多线程会丢更新；
- **有序性**：编译器/CPU 会指令重排（StoreStore、StoreLoad、LoadLoad、LoadStore）。

JMM 抽象了这一切，让 Java 程序员不必关心具体硬件。

---

## 3. JMM 抽象模型

```
线程 A             线程 B
  │ 工作内存(WM-A)   │ 工作内存(WM-B)
  │      ↓           │      ↑
  └───── 主内存 (Main Memory) ─────┘
```

- 所有变量存在主内存；
- 每个线程有自己的工作内存（CPU 缓存 + 寄存器）；
- 线程对变量的所有操作必须在工作内存中进行；
- 不同线程间的变量传递必须通过主内存。

8 种原子操作：`lock` / `unlock` / `read` / `load` / `use` / `assign` / `store` / `write`（了解即可，已逐渐被 happens-before 替代）。

---

## 4. 三大并发特性

| 特性 | JMM 保证手段 |
| --- | --- |
| **原子性** | `synchronized`、`Lock`、`Atomic*`、CAS |
| **可见性** | `volatile`、`synchronized`、`final`、内存屏障 |
| **有序性** | `volatile`、`synchronized`、happens-before |

---

## 5. happens-before 规则（必背 8 条）

> A happens-before B：A 的执行结果对 B **可见**，且 A **逻辑上**在 B 之前。

1. **程序顺序规则**：单线程内，前面的操作 hb 后面的操作。
2. **监视器锁规则**：`unlock` hb 后续对同一锁的 `lock`。
3. **volatile 规则**：写 hb 后续对同一变量的读。
4. **传递性**：A hb B、B hb C ⇒ A hb C。
5. **start 规则**：`Thread.start()` hb 该线程内的所有动作。
6. **join 规则**：线程内所有动作 hb 其他线程从 `join()` 返回。
7. **interrupt 规则**：`interrupt()` hb 被中断线程检测到中断。
8. **finalize 规则**：构造完成 hb `finalize()` 开始。

---

## 6. volatile 详解（高频）

### 6.1 两个语义
1. **可见性**：写完立即刷主内存，读时从主内存拿。
2. **禁止指令重排**（通过内存屏障）。

### 6.2 不保证原子性
```java
volatile int n = 0;
// 多线程并发：n++ 仍会丢更新
n++;  // = read(n) + n+1 + write(n)，三步非原子
```

### 6.3 内存屏障实现
```
volatile 写：    StoreStore 屏障
                 普通写
                 volatile 写
                 StoreLoad 屏障

volatile 读：    volatile 读
                 LoadLoad 屏障
                 LoadStore 屏障
                 普通读
```

### 6.4 经典用法：双重检查锁定单例

```java
public class Singleton {
    private static volatile Singleton INSTANCE;          // ① volatile 必须

    public static Singleton getInstance() {
        if (INSTANCE == null) {                          // ② 第一次检查
            synchronized (Singleton.class) {
                if (INSTANCE == null) {                  // ③ 第二次检查
                    INSTANCE = new Singleton();          // ④ 非原子！
                }
            }
        }
        return INSTANCE;
    }
}
```

> 第 ④ 步 `new` 非原子，分三步：分配内存 → 初始化对象 → 引用赋值。
> 没有 `volatile`，2 和 3 可能重排，导致其他线程拿到**未初始化的对象**。

---

## 7. synchronized 与锁优化

### 7.1 实现层次
- 字节码：`monitorenter` / `monitorexit`；
- HotSpot：每个对象头有 **Mark Word**，记录锁状态。

### 7.2 锁的四种状态（升级单向）

```
无锁 → 偏向锁 → 轻量级锁（自旋）→ 重量级锁（OS Mutex）
```

| 锁状态 | 触发 | 性能特征 |
| --- | --- | --- |
| 偏向锁 | 单线程访问 | 几乎无开销（CAS 一次） |
| 轻量级锁 | 多线程交替访问，无竞争 | CAS 自旋 |
| 重量级锁 | 真正竞争 | OS 切换开销大 |

> JDK15 起官方 **废弃偏向锁**（默认关闭），高竞争场景偏向锁反而更慢。

### 7.3 锁优化技术
- **自旋**：先空转几圈再阻塞；
- **自适应自旋**：根据上次自旋情况动态调整圈数；
- **锁消除**：JIT 检测到不可能竞争（如方法局部 StringBuilder），消除锁；
- **锁粗化**：JIT 把多次小范围加锁合并成一次大范围加锁。

---

## 8. final 的内存语义

- **写 final 字段**：构造方法返回前，禁止把 final 字段的写重排到构造之外；
- **读 final 字段**：禁止读 final 字段重排到读对象引用之前。

效果：**对象引用一经发布，final 字段一定已经初始化完毕**（前提是构造里没把 `this` 泄漏出去）。

---

## 9. CAS 与 Atomic

```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();   // 底层 Unsafe.compareAndSwapInt
```

CAS 的三个问题：
1. **ABA 问题** → `AtomicStampedReference`（带版本号）；
2. **自旋开销大** → 高竞争下不如锁；
3. **只能保证单变量原子性** → 多变量用 `AtomicReference` 包对象。

---

## 10. 高频面试题速记

1. **JMM 是什么？为什么需要它？** → 抽象内存模型，屏蔽硬件差异，定义可见性/有序性/原子性。
2. **happens-before 8 条？** → 见第 5 节。
3. **`volatile` 解决什么问题？为什么不能保证原子性？** → 可见性 + 禁重排；`++` 是复合操作。
4. **DCL 单例为什么要 `volatile`？** → 阻止 `new` 操作重排导致其他线程读到未初始化对象。
5. **`synchronized` 的锁升级过程？** → 无锁 → 偏向锁 → 轻量级 → 重量级。
6. **CAS 的 ABA 问题怎么解？** → 加版本号（`AtomicStampedReference`）。
7. **`final` 字段为什么是线程安全的？** → 构造完成前的写不会被重排到构造结束之后。

---

## 11. 推荐阅读

- 周志明《深入理解 Java 虚拟机》第 12 章
- Doug Lea《Java Concurrency in Practice》（神书）
- JSR-133: Java Memory Model and Thread Specification

> 下一步：[JVM 调优与排查](/article/jvm/tuning) | [并发编程总览](/article/java/concurrent)
