# synchronized 内幕 —— 从对象头到锁膨胀的全路径

> 这是按 [桃花岛深度模板](/STYLE) 写的第三篇深文。如果你只想刷面试，看 [并发综述](/article/java/concurrent#2-synchronized-与-lock) 那一节够了。
> 这里要回答的，是**写代码遇到锁问题、半夜被 P99 抖动告警叫起来**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. 64 位 HotSpot 上一个普通 Java 对象的对象头有多少字节？mark word 里到底编码了哪些信息？给一张位级别的布局图。
2. "偏向锁 → 轻量级锁 → 重量级锁"是单向不可逆的吗？什么条件下会触发"偏向锁撤销"？为什么撤销代价这么高？
3. JEP 374 为什么把偏向锁默认关掉、并最终在 JDK 18 弃用？给三个理由。
4. `ObjectMonitor` 内部有 `_EntryList`、`_WaitSet`、`_cxq` 三个队列，每个分别存什么线程？`Object.wait()` 让线程进哪个？`notify()` 把它移到哪个？
5. 虚拟线程在 `synchronized` 块里做阻塞 IO 会"pin carrier thread"，根源是什么？JEP 491 在 JDK 24 是怎么解决的？
6. 锁粗化（lock coarsening）和锁消除（lock elision）真的会触发吗？写一段代码让 JIT 触发它，然后用什么命令验证？
7. 同样高竞争场景下，`synchronized` 和 `ReentrantLock` 到底谁快？给一个能让前者快、再给一个让后者快的具体场景。
8. `Object.wait/notify` 和 `LockSupport.park/unpark` 在"前后顺序"上有什么本质差别？为什么 AQS 选择了后者？

如果有半数答不上，往下看。

---

## 1. 底层模型：对象头与 ObjectMonitor

绝大多数文章把 synchronized 讲到"锁升级 4 个阶段"就结束了。但这层认知解释不了上面 8 个问题。要看懂它们，必须下沉到 HotSpot 的对象内存布局和 ObjectMonitor 数据结构。

### 1.1 Java 对象的内存布局

任何一个 Java 对象在堆上的物理结构固定为三段：

```
┌─────────────────────────────────────┐
│ Object Header  对象头                │
│   ├── Mark Word     (8 字节, 64 位)  │ ← 锁状态都藏在这里
│   ├── Klass Pointer (4 或 8 字节)    │ ← 指向类元数据
│   └── [Array Length] (4 字节, 仅数组) │
├─────────────────────────────────────┤
│ Instance Data  实例字段              │
│   ├── long/double  (8 字节对齐)      │
│   ├── int/float    (4 字节对齐)      │
│   ├── short/char   (2 字节对齐)      │
│   ├── byte/boolean (1 字节)          │
│   └── reference    (4 或 8 字节)     │
├─────────────────────────────────────┤
│ Padding  对齐填充                    │
│   └── 让整个对象 8 字节对齐           │
└─────────────────────────────────────┘
```

注意几点：

1. **对象头的 Klass Pointer 默认开了压缩（CompressedOops）= 4 字节**。关闭压缩或堆 > 32GB 时变成 8 字节。
2. **整个对象按 8 字节对齐**（`-XX:ObjectAlignmentInBytes` 默认 8）。这是为什么 `new Object()` 在 64 位 HotSpot 上是 16 字节而不是 12 字节。
3. **Mark Word 是 synchronized 的核心战场** —— 锁的所有"轻量级状态"都直接编码在这 8 字节里，不申请额外空间。

观察工具 [JOL](https://openjdk.org/projects/code-tools/jol/)：

```java
System.out.println(ClassLayout.parseInstance(new Object()).toPrintable());
```

会输出每个字段的偏移、大小、当前 mark word 的位级别值。**生产排查锁问题这是必备工具**。

### 1.2 Mark Word 在 64 位 HotSpot 下的编码

Mark Word 共 64 位，根据低 3 位的 `lock` 标志位编码 5 种状态。下面是 JDK 17 之前（含偏向锁）的完整布局：

```
            ┌─────────────────────────────────────────────────────┬──────┬───┐
无锁/Normal │ unused:25  hashcode:31  unused:1  age:4  biased:1   │ 01   │ 0 │
            ├─────────────────────────────────────────────────────┼──────┼───┤
偏向锁/Biased│ thread:54  epoch:2  unused:1  age:4                 │ 01   │ 1 │
            ├─────────────────────────────────────────────────────┴──────┴───┤
轻量级锁/Lw │ pointer_to_lock_record:62                                  00 │
            ├────────────────────────────────────────────────────────────────┤
重量级锁/Hv │ pointer_to_heavyweight_monitor:62                          10 │
            ├────────────────────────────────────────────────────────────────┤
GC标记/Marked│ <used by GC, e.g. forwarding pointer>                      11 │
            └────────────────────────────────────────────────────────────────┘
                                                                  lock:2  bias:1
```

逐档解读：

- **无锁（01 + biased=0）**：高位存对象的 `identityHashCode`（lazy 计算，第一次 `hashCode()` 才填）+ 分代年龄 `age`（4 位 = 最多 15 代，所以 GC 提升老年代默认阈值 = 15 不是巧合）。
- **偏向（01 + biased=1）**：高 54 位存"偏向到的线程 ID 指针"（实际是 `JavaThread*` 的低位），`epoch` 用于批量重偏向。
- **轻量级（00）**：mark word 整个被替换为"指向当前线程栈上 Lock Record 的指针"。Lock Record 里反向存了原 mark word 的备份（叫 displaced mark word）。
- **重量级（10）**：mark word 变为"指向 ObjectMonitor 的指针"。锁的真正语义都搬到 ObjectMonitor 里。
- **GC 标记（11）**：被 GC 占用，比如 forwarding pointer，与 synchronized 无关。

**反直觉点**：调用 `Object.hashCode()` 会让无锁状态写入 hashcode 位，**之后这个对象就再也不能用偏向锁** —— 因为 hashcode 占了 31 位，跟偏向锁需要存 thread+epoch 的位完全冲突。这是案例 ① 里 P99 抖动的根之一。

### 1.3 ObjectMonitor 的内部结构

重量级锁的真正实现在 C++ 类 `ObjectMonitor`（`hotspot/share/runtime/objectMonitor.hpp`）。简化版字段：

```cpp
class ObjectMonitor {
  void*           _header;          // 备份的原 mark word
  oop             _object;          // 关联的 Java 对象
  void*           _owner;           // ★ 当前持锁线程指针（NULL 表示无主）
  jlong           _previous_owner_tid;
  ObjectMonitor*  _next_om;         // 链表，挂在线程的 free / used 列表上
  intptr_t        _recursions;      // ★ 重入次数
  ObjectWaiter*   _EntryList;       // ★ 同步队列：等待获取锁的线程
  ObjectWaiter*   _cxq;             // ★ Contention Queue：刚来的竞争者，先进 cxq
  ObjectWaiter*   _WaitSet;         // ★ 等待集：调用了 wait() 的线程
  jint            _waiters;         // _WaitSet 大小
  jint            _contentions;     // 活跃竞争者数
  volatile int    _SpinDuration;    // 自适应自旋时长
  // ... 还有 succ、Responsible 等优化字段
};
```

视觉上：

```
                    ObjectMonitor
                   ┌─────────────────────────────────────┐
   持有锁的线程 ──→ │ _owner = T1   _recursions = 2       │
                   ├─────────────────────────────────────┤
                   │ _cxq        →  T9 → T8 → T7         │ 新来的竞争者，CAS 头插
                   │ _EntryList  →  T6 → T5 → T4         │ 排队等锁
                   │ _WaitSet    →  T3 ⇄ T2              │ wait() 进来的，等 notify
                   └─────────────────────────────────────┘

  T1.unlock() 时：从 _EntryList 唤醒一个；如果 EntryList 空，把 cxq 倒灌过来
  T1.notify()  时：从 _WaitSet 摘一个，挂到 _EntryList 或 _cxq
  T1.notifyAll: 整个 _WaitSet 都搬走
```

为什么有 `_cxq` 又有 `_EntryList`？

- `_cxq`（Contention Queue）：高竞争下大量线程瞬间到来，每个线程**用 CAS 头插到 cxq**。CAS 头插是无锁的，吞吐高。
- `_EntryList`：真正"排队"的队列。当 owner 释放锁、要选下一个继承者时，会先去 `EntryList`，没有再把 `cxq` 整个"翻转"过来。

这是典型的"**生产快路径、消费慢路径**"设计 —— 进队列(竞争发生)很频繁要快，出队列(只有 owner 释放时)较少可以慢。

`_WaitSet` 完全独立：它装的是调用了 `Object.wait()` 主动放弃锁的线程，**和"等锁"的线程不在一个集合里**。`notify()` 把线程从 `_WaitSet` 搬到 `_EntryList`/`_cxq`，意味着"它被放回队列，等待重新拿锁"。

### 1.4 锁状态转换图

```
     无锁 (lock=01, bias=0)
       │
       │ 第一次 synchronized 进入  &  -XX:+UseBiasedLocking 开
       ▼
     偏向 (lock=01, bias=1, thread=T1)
       │
       │ 同一线程 T1 反复进入：CAS 都不需要做，直接拿锁
       │
       │ 另一线程 T2 来了：触发"偏向锁撤销"（必须在 safepoint）
       ▼
     无锁 / 直接进 → 轻量级 (lock=00, ptr→Lock Record)
       │
       │ T1 的栈帧分配 Lock Record，CAS 把 mark word 换成 ptr
       │ 失败 → 锁竞争存在 → 自旋一定次数
       ▼
     重量级 (lock=10, ptr→ObjectMonitor)
       │
       │ 自旋仍失败 / wait()  → 真正阻塞，进入 ObjectMonitor 等待队列
       │ 操作系统挂起 (pthread_cond_wait)
       ▼
     释放后保持 lock=10，下次进入仍走 ObjectMonitor 重路径

注意：
  - 偏向 → 轻量：必须经过"撤销"（revoke），STW
  - 轻量 → 重量：在 InflatedEnter 路径里，单线程瞬间完成，不 STW
  - 重量 → 轻量：HotSpot 直到很晚才支持"deflation"（JDK 15+ 的 -XX:+MonitorDeflationInterval）
```

**关键认知**：状态不是简单的"递增"。

- 偏向锁撤销代价**极高**（要在所有线程到达安全点时统一处理），高频偏向撤销是案例 ① 的元凶
- 锁膨胀（轻 → 重）相对廉价，但**膨胀后即使竞争消失也长期不会降级**，导致后续都走 monitor 慢路径
- JDK 15 引入的 monitor deflation 才让重量级锁有机会"自动瘦身"

> **口诀**：mark word 是 synchronized 的"省钱手段"，能不申请 ObjectMonitor 就不申请；ObjectMonitor 是"兜底机制"，一旦申请就长期不还。

---

## 2. 源码切片：HotSpot 关键路径

> 基于 OpenJDK 17（含偏向锁）+ JDK 21（已删除偏向锁）对比。注释为本文添加。

### 2.1 markWord.hpp 中的位定义

```cpp
// hotspot/share/oops/markWord.hpp 简化
class markWord {
  uintptr_t _value;

  // 低 3 位的 lock 编码
  static const int lock_bits           = 2;
  static const int biased_lock_bits    = 1;
  static const int age_bits            = 4;
  static const int hash_bits           = 31;

  static const uintptr_t lock_mask         = right_n_bits(lock_bits);     // 0x3
  static const uintptr_t biased_lock_mask  = right_n_bits(lock_bits + biased_lock_bits); // 0x7

  static const uintptr_t unlocked_value          = 1;   // 001
  static const uintptr_t biased_lock_pattern     = 5;   // 101
  static const uintptr_t locked_value            = 0;   // 000  → 轻量级
  static const uintptr_t monitor_value           = 2;   // 010  → 重量级
  static const uintptr_t marked_value            = 3;   // 011  → GC

  bool is_neutral()        const { return (_value & biased_lock_mask) == unlocked_value; }
  bool is_biased()         const { return (_value & biased_lock_mask) == biased_lock_pattern; }
  bool has_monitor()       const { return (_value & monitor_value) != 0 && (_value & lock_mask) == monitor_value; }

  ObjectMonitor* monitor() const {
    assert(has_monitor(), "");
    return (ObjectMonitor*)(_value & ~monitor_value);    // 抹掉低 2 位标志位
  }
};
```

**逐段点透**：

- **位掩码体系**：`lock_mask` 取低 2 位决定锁状态；`biased_lock_mask` 多看一位区分"偏向"和"无锁"。
- **`monitor()` 方法**：把 mark word 当指针用前先 `& ~monitor_value` 抹掉标志位 —— ObjectMonitor 是 8 字节对齐的，所以低 3 位天生为 0，可以借去存标志位。这是 HotSpot 大量使用的**指针位偷藏**技巧。
- **`unlocked_value = 1`**：注意"无锁"的 lock 位是 `01` 而不是 `00`。`00` 留给了"轻量级锁"。这避免了"new 出来的对象天然带轻量级锁标记"的尴尬。

### 2.2 ObjectSynchronizer::quick_enter（轻量级加锁快路径）

```cpp
// hotspot/share/runtime/synchronizer.cpp 简化
bool ObjectSynchronizer::quick_enter(oop obj, JavaThread* current,
                                     BasicLock* lock) {
  markWord mark = obj->mark();

  // ① 已经是重量级 monitor：尝试快速重入
  if (mark.has_monitor()) {
    ObjectMonitor* m = mark.monitor();
    if (m->owner() == current) {
      m->set_recursions(m->recursions() + 1);
      return true;          // 重入成功，零开销
    }
    return false;           // 让慢路径处理竞争
  }

  // ② 无锁状态：CAS 尝试把 mark 换成指向 BasicLock 的指针
  if (mark.is_neutral()) {
    lock->set_displaced_header(mark);                    // 备份原 mark
    if (obj->cas_set_mark(markWord::from_pointer(lock), mark) == mark) {
      return true;          // CAS 成功，轻量级加锁完成
    }
    return false;           // CAS 失败，竞争存在，进慢路径
  }

  return false;
}
```

**逐段点透**：

- **`mark.has_monitor()` 优先判断**：因为已膨胀的对象走重路径反而是常态（一旦升级很难降回去）。这里只做"自己重入"的最快通道，避免每次重入都走完整 monitor enter。
- **`displaced_header`**：把原 mark word 存到当前线程栈帧的 BasicLock 里。释放时如果 mark 还指向这个 BasicLock，CAS 把原 mark 写回 —— **整个加解锁不需要 OS 调用**。这是"轻量级"的关键。
- **CAS 失败的处理**：直接返回 false，不在这里自旋。自旋逻辑放在更上层的 `enter()` 慢路径里 —— 把 quick path 写得越短越好，编译器能内联。

### 2.3 ObjectSynchronizer::inflate（锁膨胀过程）

```cpp
// hotspot/share/runtime/synchronizer.cpp 简化
ObjectMonitor* ObjectSynchronizer::inflate(JavaThread* current, oop object,
                                            const InflateCause cause) {
  for (;;) {
    const markWord mark = object->mark_acquire();

    // 已经是重量级，直接返回
    if (mark.has_monitor()) {
      return mark.monitor();
    }

    // ① 当前是轻量级锁：把 monitor 接管它
    if (mark.has_locker()) {
      ObjectMonitor* m = new ObjectMonitor(object);
      m->set_header(mark.displaced_mark_helper());     // 把 displaced mark 交给 monitor 保管
      m->set_owner_from_BasicLock(mark.locker());      // 持锁人就是当前轻量级锁的持有者
      m->set_recursions(0);

      // CAS：mark 从"指向 BasicLock"变成"指向 monitor"
      markWord mw = markWord::encode(m);
      if (object->cas_set_mark(mw, mark) != mark) {
        delete m;
        continue;        // 别人抢先膨胀了，重试
      }
      return m;
    }

    // ② 当前是无锁：直接挂个 monitor，owner 留空
    if (mark.is_neutral()) {
      ObjectMonitor* m = new ObjectMonitor(object);
      m->set_header(mark);
      markWord mw = markWord::encode(m);
      if (object->cas_set_mark(mw, mark) != mark) {
        delete m;
        continue;
      }
      return m;
    }
  }
}
```

**逐段点透**：

- **接管轻量级锁**：膨胀的关键是把当前持有 BasicLock 的那个线程"无感地变成 monitor 的 owner"。原来 mark 指向栈上 BasicLock，现在 mark 指向堆上 monitor，但**owner 字段直接复制 BasicLock 的持有者** —— 当前持锁线程根本不知道发生了什么，下次 unlock 走 monitor 路径而已。
- **CAS 失败重试**：膨胀本身可能多线程同时尝试。任意一个成功了，其他人都走"已经是 monitor"分支。
- **monitor 的内存分配**：`new ObjectMonitor` 走 native 堆，不走 Java 堆。所以 monitor 的生命周期不受 GC 控制，靠 `MonitorDeflationThread`（JDK 15+）回收。
- **没有降级**：你看不到任何"monitor 收回去退化为轻量"的代码。一旦膨胀，至少要等到 monitor deflation 周期才有机会。

### 2.4 ObjectMonitor::enter（重量级锁的核心）

```cpp
// hotspot/share/runtime/objectMonitor.cpp 极度简化
bool ObjectMonitor::enter(JavaThread* current) {
  // ① 第一次 CAS 尝试：还没人持锁就直接拿
  void* prev = nullptr;
  if (Atomic::cmpxchg(&_owner, prev, current) == nullptr) {
    return true;
  }

  // ② 重入检查
  if (_owner == current) {
    _recursions++;
    return true;
  }

  // ③ 自适应自旋
  if (TrySpin(current)) {     // 默认自旋 10 次左右，根据历史成功率调整
    return true;
  }

  // ④ 真正阻塞：进队列 + park
  EnterI(current);
  return true;
}

void ObjectMonitor::EnterI(JavaThread* current) {
  ObjectWaiter node(current);
  node.TState = ObjectWaiter::TS_CXQ;

  // CAS 头插到 _cxq
  ObjectWaiter* head;
  do {
    head = _cxq;
    node._next = head;
  } while (Atomic::cmpxchg(&_cxq, head, &node) != head);

  // park 自己（操作系统层面的挂起）
  for (;;) {
    if (TryLock(current)) break;     // 中途被唤醒，再 try 一次
    current->_ParkEvent->park();      // 真阻塞，进入内核态
    if (TryLock(current)) break;
  }

  // 拿到锁了，从队列摘掉自己
  UnlinkAfterAcquire(current, &node);
}
```

**逐段点透**：

- **三级竞争**：CAS → 自旋 → park。一级比一级慢。`TrySpin` 是"用户态自旋"，park 是"内核态阻塞"。
- **TrySpin 的"自适应"**：HotSpot 跟踪每个 monitor 历史上"自旋多少次能拿到锁"，动态调整 `_SpinDuration`。如果总是自旋失败，下次干脆少自旋或不自旋直接 park —— 自旋本身烧 CPU，烧着烧着不如让出去。
- **`_cxq` 头插用 CAS**：高竞争下大量线程同时入队，CAS 头插是性能最高的并发栈操作。代价是顺序变成"后进先出"，对公平性不友好（这就是为什么 synchronized 是非公平锁）。
- **park 的本质**：`pthread_cond_wait`（Linux）或 `WaitForSingleObject`（Windows）。这是从用户态切到内核态的开销，通常 1~10 微秒，是 CAS 加锁的 1000 倍以上。

### 2.5 ObjectMonitor::exit（释放与唤醒）

```cpp
// hotspot/share/runtime/objectMonitor.cpp 简化
void ObjectMonitor::exit(JavaThread* current) {
  // 重入检查
  if (_recursions > 0) {
    _recursions--;
    return;
  }

  _owner = nullptr;          // 释放
  OrderAccess::storeload();  // 强 fence：保证 owner=null 对其他 CPU 立即可见

  // 选下一个被唤醒的线程
  ObjectWaiter* w = _EntryList;
  if (w != nullptr) {
    ExitEpilog(current, w);
    return;
  }

  // EntryList 空了，把 cxq 整体翻转过来当作新的 EntryList
  w = _cxq;
  if (w != nullptr) {
    if (Atomic::cmpxchg(&_cxq, w, (ObjectWaiter*)nullptr) != w) {
      return;  // 有新线程进 cxq，让它自己去抢
    }
    // 翻转成 EntryList
    ObjectWaiter* q = nullptr;
    while (w != nullptr) {
      ObjectWaiter* next = w->_next;
      w->_next = q;
      q = w;
      w = next;
    }
    _EntryList = q;
    ExitEpilog(current, q);
  }
}

void ObjectMonitor::ExitEpilog(JavaThread* current, ObjectWaiter* w) {
  w->TState = ObjectWaiter::TS_ENTER;
  ParkEvent* ev = w->_event;
  ev->unpark();              // 唤醒它
  // 注意：unpark 只是通知，被唤醒的线程还要自己再 CAS 抢锁
}
```

**逐段点透**：

- **`storeload` fence 不可省**：x86 的 store 默认对其他核**不立即可见**，必须靠 mfence 让 owner=null 在 unpark 之前真正落地，否则被唤醒的线程立刻 try 时可能还看到旧的 owner。这就是 synchronized 自带 happens-before 语义的源码出处。
- **cxq 翻转**：cxq 是 LIFO（后进先出），翻转一次后变成 FIFO 给 EntryList 用。这是个"批量转换"，平摊到每个 await 线程开销很小。
- **唤醒后还要再 CAS 抢锁**：HotSpot 没有"直接传递锁"，被 unpark 的线程醒来仍要 `Atomic::cmpxchg(&_owner, ...)`。这给了正在自旋的"野生竞争者"插队的机会 → 非公平 → 总吞吐高。
- **没有"全部唤醒"**：每次 exit 只 unpark 一个线程，避免惊群（thundering herd）。

### 2.6 BiasedLocking::revoke_at_safepoint（偏向锁撤销）

```cpp
// hotspot/share/runtime/biasedLocking.cpp 简化（JDK 15 之前）
void BiasedLocking::revoke_at_safepoint(Handle h_obj) {
  assert(SafepointSynchronize::is_at_safepoint(), "must be at safepoint");

  oop obj = h_obj();
  markWord mark = obj->mark();
  if (!mark.has_bias_pattern()) return;

  // ① 找到偏向到的线程
  JavaThread* biased_thread = mark.biased_locker();

  if (biased_thread == nullptr) {
    // 已经不偏向任何活线程，直接复原成无锁
    obj->set_mark(markWord::prototype());
    return;
  }

  // ② 检查这个线程当前是不是真在持锁
  bool is_actively_locked = false;
  for (StackFrame* f : biased_thread->frames()) {
    BasicObjectLock* mon = f->find_monitor_for(obj);
    if (mon != nullptr) {
      is_actively_locked = true;
      // 偏向锁活跃 → 升级为轻量级锁，把 BasicLock 填好
      mon->lock()->set_displaced_header(markWord::unused_mark());
      obj->set_mark(markWord::encode(mon->lock()));
    }
  }

  if (!is_actively_locked) {
    obj->set_mark(markWord::prototype());     // 没在持，就还原无锁
  }
}
```

**逐段点透**：

- **`assert at_safepoint`**：撤销必须在 safepoint 进行 —— **所有 Java 线程都要停下来**，VM 才敢扫描偏向线程的栈。这是 STW 的根源。
- **扫描栈帧**：要判断"被偏向的线程当前还在不在锁里"，必须遍历它的栈找有没有相关的 BasicObjectLock。栈深的线程扫描慢。
- **批量撤销 / 批量重偏向**：HotSpot 实际上有 `BiasedLocking::bulk_revoke` —— 同一个 class 的对象被频繁撤销超阈值时，整个 class 的所有实例都不再偏向。这是"防止单点撤销不断"的优化。

为什么 JEP 374（JDK 15）默认关闭、JDK 18 弃用？三个理由：

1. **STW 开销**：每次撤销都要 safepoint，现代应用启动期高频撤销直接拉爆 P99
2. **现代 CPU 上 CAS 已经很快**：偏向锁省下的"CAS 一次"在 ARM/Apple Silicon 上只值几纳秒，远小于撤销代价
3. **代码复杂度**：偏向锁逻辑在 HotSpot 里散落 70+ 处，维护成本巨大

### 2.7 JIT 的锁优化：lock coarsening 与 lock elision

```java
// 原始代码 —— 看起来加锁 3 次
public String concat(String a, String b, String c) {
    StringBuffer sb = new StringBuffer();
    sb.append(a);    // synchronized
    sb.append(b);    // synchronized
    sb.append(c);    // synchronized
    return sb.toString();
}
```

JIT 的优化：

**锁消除（Lock Elision）**：通过逃逸分析（Escape Analysis）发现 `sb` 不会逃逸到方法外，**当前线程独占**，所以加锁完全没意义 —— 直接擦掉所有锁。

**锁粗化（Lock Coarsening）**：如果上面的 `sb` 是个外部传入的、不能消除的对象，JIT 会发现"连续三次对同一对象加锁解锁"，把它合并成"加一次锁、做三次 append、解一次锁"，省掉中间的解锁/加锁开销。

观察 JIT 优化效果：

```bash
# 1. 让 JVM 打印优化日志
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+PrintEscapeAnalysis \
     -XX:+PrintEliminateLocks \
     -XX:+PrintCompilation \
     -XX:CompileCommand=print,*Foo.concat \
     YourApp

# 2. 用 JITWatch 可视化
# 3. JMH 实测对比 -XX:-EliminateLocks vs 默认
```

**反直觉点**：

- **逃逸分析需要 server 编译器（C2）**，client 模式（C1）不做。生产 JVM 默认开 server，但启动初期 C1 阶段加锁不会被消除。
- **锁粗化的临界**：循环里反复加锁的场景，JIT 会**把锁提到循环外**。但如果循环里有阻塞调用，粗化反而会让阻塞延长 → JIT 不会粗化跨阻塞调用。

> **口诀**：mark word 决定锁状态，ObjectMonitor 决定竞争行为，JIT 决定能不能"假装没加锁"。三层都要看，单层都不够。

---

## 3. 实测数据

### 3.1 单线程加解锁开销三档对比

JMH 测一个最简单的"循环加锁、改个 int、释放"。同一段代码用三种 JVM 配置跑：

```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Thread)
@Fork(value = 1, jvmArgsAppend = {"-XX:-UseBiasedLocking"})  // 通过 @Param 切换
public class LockBench {

    private final Object lock = new Object();
    private long counter;

    @Benchmark
    public long syncIncrement() {
        synchronized (lock) {
            return ++counter;
        }
    }
}
```

JDK 11 + 不同 JVM 参数下的典型结果（Apple M2 / Linux x86_64 数据近似）：

| 配置 | 单次 synchronized 进出耗时 |
| --- | --- |
| `-XX:+UseBiasedLocking`（偏向命中） | **~3 ns** |
| `-XX:-UseBiasedLocking`（轻量级） | **~12 ns** |
| 强制重量级（多线程争抢一次后） | **~80 ns**（无竞争重入） |
| 多线程真竞争 | **2~10 µs**（park/unpark） |

数量级差距：偏向 ↔ 轻量级 4 倍，轻量级 ↔ 重量级 7 倍，无竞争重量级 ↔ 真竞争重量级 100 倍。

**反直觉点**：偏向锁省下的那 9ns，只有在**真实生产里 99% 的同步对象只被一个线程访问**时才值。一旦这个假设破灭（高频偏向撤销），它甚至比直接用轻量级慢。

### 3.2 多线程竞争：synchronized vs ReentrantLock vs Atomic

测同一个 `counter++` 操作，N 个线程并发：

```java
@State(Scope.Benchmark)
@Threads(8)   // 8 线程
public class CounterBench {

    long plain;
    final Object monitor = new Object();
    final ReentrantLock lock = new ReentrantLock();
    final AtomicLong atomic = new AtomicLong();
    final LongAdder adder = new LongAdder();

    @Benchmark public long sync() {
        synchronized (monitor) { return ++plain; }
    }
    @Benchmark public long reentrant() {
        lock.lock(); try { return ++plain; } finally { lock.unlock(); }
    }
    @Benchmark public long atomicCas() {
        return atomic.incrementAndGet();
    }
    @Benchmark public long longAdder() {
        adder.increment(); return adder.sum();
    }
}
```

8 核机器 8 线程典型吞吐（单位：million ops/s，越大越好）：

| 实现 | 1 线程 | 4 线程 | 8 线程 | 32 线程 |
| --- | --- | --- | --- | --- |
| `synchronized` | 80 | 18 | 9 | 6 |
| `ReentrantLock` | 60 | 22 | 12 | 8 |
| `AtomicLong` | 100 | 35 | 18 | 10 |
| `LongAdder` | 95 | 280 | 520 | 1100 |

观察：

- **单线程**：synchronized 反而最快（偏向锁加持）—— 但前提是没被偏向撤销过。
- **中低竞争（4~8 线程）**：AtomicLong > ReentrantLock > synchronized。差距不大，约 1.5~2 倍。
- **高竞争（32+ 线程）**：synchronized 与 ReentrantLock 都退化到 park/unpark；LongAdder 通过分桶累加完全碾压。
- **超高竞争场景**：换数据结构比换锁更划算。看到 synchronized 慢了别第一反应换 ReentrantLock，先想"能不能不要这个锁"。

### 3.3 JDK 8 → 15 → 21 的 synchronized 演进

定性数据（基于多家厂商基准 + OpenJDK 邮件列表披露）：

| 维度 | JDK 8 | JDK 15 | JDK 21 |
| --- | --- | --- | --- |
| 偏向锁 | 默认开 | 默认关（JEP 374） | 已删除 |
| 单线程加锁开销 | 3 ns（偏向命中） | 12 ns（轻量级） | 12 ns |
| 启动期偏向撤销开销 | **巨大**（高频 STW） | 已消除 | 已消除 |
| ObjectMonitor 回收 | 几乎不回收 | MonitorDeflation 引入 | 持续优化 |
| `Object.wait/notify` 性能 | baseline | +5% | +10% |
| 虚拟线程兼容 | — | — | 仍 pin（JDK 21）；JEP 491 在 JDK 24 修复 |

主要里程碑：

- **JDK 8 → 11**：基本无变化。偏向锁仍是默认。
- **JDK 15（JEP 374）**：偏向锁默认关。理由是"维护成本 > 收益"。
- **JDK 18（JEP 410）**：偏向锁实现彻底删除。
- **JDK 21**：synchronized 可与虚拟线程一起使用，但 `synchronized` 块内若发生阻塞 IO 会**pin** carrier。
- **JDK 24（JEP 491）**：彻底解决虚拟线程在 synchronized 内的 pin 问题，synchronized 终于可以"无脑"用在虚拟线程里。

**生产升级提示**：JDK 8 → JDK 11+ 后如果某些应用 P99 反而上升，怀疑是偏向锁带来的撤销开销没了 → 但同时有大量同步对象只被单线程访问，这部分原本"白嫖偏向"的代码每次都要走 CAS。

### 3.4 火焰图能告诉你什么

用 [async-profiler](https://github.com/async-profiler/async-profiler) 的 lock 模式：

```bash
# 1. CPU 火焰图：synchronized 加锁竞争位置
./profiler.sh -e cpu -d 30 -f cpu.html <pid>

# 2. lock 火焰图：每个 monitor 的等待时长聚合
./profiler.sh -e lock -d 30 -f lock.html <pid>

# 3. wall-clock 火焰图：识别 park/unpark 占栈
./profiler.sh -e wall -d 30 -f wall.html <pid>
```

典型问题信号：

- **`ObjectMonitor::EnterI` 火焰栈高**：高竞争锁，考虑改 LongAdder / 分段锁 / 无锁结构
- **`pthread_cond_wait` 占栈很多**：线程在 Object.wait 内阻塞，可能 wait 条件没在循环里检查
- **`SafepointSynchronize::do_call_back` + `BiasedLocking::revoke`**：大量偏向锁撤销 → JDK 8 上加 `-XX:-UseBiasedLocking` 测试
- **`Unsafe.park` + 栈底是 `synchronized` 块**：典型重量级锁竞争，配合 lock 火焰图找出对象

JFR 也能抓 monitor 事件：

```bash
jcmd <pid> JFR.start name=lock duration=60s settings=profile filename=lock.jfr
# 之后用 JDK Mission Control 打开 lock.jfr，看 "Java Monitor Blocked" 事件
```

`jdk.JavaMonitorEnter` 事件直接告诉你：哪个对象、谁阻塞了谁、阻塞了多久。比 jstack 看快照精准 100 倍。

> **口诀**：synchronized 的性能问题永远先看竞争模式 —— 单线程问题看偏向，中竞争看 CAS，高竞争看 park。火焰图三栈联看（CPU + lock + wall）才能定位。

---

## 4. 生产案例 8 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这是本文长尾价值的核心。

### 案例 ① 偏向锁撤销的 STW 抖动 —— "短任务高频加锁的 P99 反常"

#### 现象

JDK 8 服务，业务逻辑里大量短同步块（每次 < 1 µs）。监控显示 P99 偶发 5~10ms 抖动，CPU、GC、IO 都没异常。

抓火焰图发现栈底大量出现：

```
SafepointSynchronize::block
    BiasedLocking::revoke_at_safepoint
        VM_RevokeBias::doit
```

#### 根因

业务里的某个共享集合（比如 `Collections.synchronizedMap` 包出来的），原本被线程 A 持续偏向。某次另一个线程 B 偶发访问 → 触发"偏向锁撤销"。

撤销过程：
1. **触发 safepoint**：所有 Java 线程必须停下来 → 整个 JVM STW
2. **VM 线程扫描偏向线程的栈**：检查是否还在持锁
3. 撤销 → 升级为轻量级或还原无锁

撤销本身只要几十微秒，但**触发 safepoint 的 "TTSP（Time To Safepoint）"代价巨大** —— 跑得很 hot 的循环线程到 safepoint 可能要几毫秒。

更糟的是 HotSpot 的"批量撤销"机制：同一个 class 撤销次数超阈值（默认 20）会触发 `bulk_revoke`，**对该 class 的所有实例集中撤销**。一次性几千个对象的撤销在 safepoint 里串行做，STW 直接突破 100ms。

观察方法：

```bash
java -XX:+PrintSafepointStatistics \
     -XX:PrintSafepointStatisticsCount=1 \
     -XX:+SafepointTimeout \
     YourApp

# 或者用 -Xlog:safepoint*
```

输出中找 `RevokeBias` / `BulkRevokeBias` 的频率和耗时。

#### 修复

**最简单**：关偏向锁。

```bash
java -XX:-UseBiasedLocking ...
```

代价：单线程加锁慢 9ns，对绝大多数业务无感。

**更激进**：升级到 JDK 11+ 后默认就是这样了；JDK 8 上等于"提前预演"行为。

**保留偏向但延迟启用**：JVM 启动时大量类初始化都会用 synchronized，提前关偏向能避免启动期的撤销风暴。

```bash
-XX:BiasedLockingStartupDelay=0   # 默认 4000ms，意思是 JVM 启动 4s 后才开偏向
                                  # 设 0 = 立刻开；设很大 = 永不开
```

#### 验证

```bash
# 改前：抓一段时间安全点统计
-XX:+PrintSafepointStatistics → 看 RevokeBias 行的次数和总耗时

# 改后：对比同时段
# 期望 RevokeBias 耗时归零，P99 平稳
```

> **口诀**：偏向锁不适合**多线程偶发竞争**的场景。JDK 8 上看到 P99 抖 + safepoint 火焰栈 → 直接 `-XX:-UseBiasedLocking`。

---

### 案例 ② JDK 15+ 偏向锁默认关闭后的性能下降事故

#### 现象

某团队从 JDK 11 升级到 JDK 17。压测发现某个核心接口 QPS 下降 8%。火焰图看 `ObjectSynchronizer::fast_enter` 占用比之前高一截。

#### 根因

业务大量使用 `Hashtable` / `Vector` / `Collections.synchronizedXxx` 这类**单线程访问的同步容器**。JDK 11 时偏向锁默认开，这些容器的 synchronized 几乎零开销。JDK 15+ 偏向锁默认关，每次进入都要做 CAS。

虽然单次 CAS 只多 9ns，但**一个高频接口可能进出几十个同步块**，累计起来就是百分位级别的退化。

#### 修复

**短期**：JVM 参数显式开偏向锁（JDK 15~17 仍支持）。

```bash
java -XX:+UseBiasedLocking ...
```

JDK 18 之后这个参数被移除，加了直接报错，**这条路就走不通了**。

**长期**：把单线程访问的同步容器换成非同步版本。

```java
// ❌ 用了 Hashtable，每次操作都加锁
Hashtable<String, User> cache = new Hashtable<>();

// ✅ 单线程访问就用 HashMap，不要锁
HashMap<String, User> cache = new HashMap<>();

// ✅ 多线程访问用 ConcurrentHashMap，不要 Collections.synchronizedMap
ConcurrentHashMap<String, User> cache = new ConcurrentHashMap<>();
```

`Collections.synchronizedMap` 是粗粒度大锁，性能远不如 ConcurrentHashMap 的分段策略。

**根治**：JIT 的逃逸分析能把"局部变量上的同步"消除。如果某个对象只在一个方法栈内使用，把它改成局部变量而不是字段，让 JIT 自动消除锁。

```java
// ❌ 字段级 sb，逃逸分析失败
private StringBuffer sb = new StringBuffer();
public String make(String a, String b) {
    sb.setLength(0);
    sb.append(a); sb.append(b);
    return sb.toString();
}

// ✅ 局部变量 sb，且 JIT 能识别 sb 不逃逸 → 锁消除
public String make(String a, String b) {
    StringBuffer sb = new StringBuffer();
    sb.append(a); sb.append(b);
    return sb.toString();
}
```

更好的做法：直接用 `StringBuilder`（无锁版）。

#### 验证

```bash
# 看 JIT 是否真做了锁消除
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintEliminateLocks ...

# 期望输出：++++ Eliminating: <object>
```

JMH 对比同样代码 `-XX:-DoEscapeAnalysis` vs 默认，差异即逃逸分析省下来的锁开销。

> **口诀**：升 JDK 后单线程同步容器的开销会浮上来。**根治不是恢复偏向锁，是删掉不必要的 synchronized**。

---

### 案例 ③ 锁膨胀链路 —— "竞争一过 monitor 不还，长期慢路径"

#### 现象

某个对象在系统启动时被几个线程短暂竞争了一下，之后基本只有一个线程访问。但火焰图显示这个对象的同步操作长期走 `ObjectMonitor::enter` 重路径，没有降级回轻量级。

#### 根因

HotSpot **不会主动**把重量级锁降级回轻量级。一旦膨胀，mark word 就一直指向 ObjectMonitor，下次再进入也走 monitor enter（虽然没竞争时是 quick 路径，但仍要做 owner CAS、recursions 操作，比"轻量级 + 备份 mark"重）。

JDK 15 之前这个 monitor 几乎永远不会被回收。**一个被竞争过一次的对象，余生都是重量级**。

观察方法：

```java
// 用 JOL 看 mark word
import org.openjdk.jol.info.ClassLayout;

System.out.println(ClassLayout.parseInstance(yourLock).toPrintable());
// 看输出最后两位：
//   01 → 偏向 / 无锁
//   00 → 轻量级
//   10 → 重量级（已膨胀）
```

#### 修复

**JDK 15+：开 MonitorDeflation**（默认开）。

```bash
# 这两个参数 JDK 15+ 默认开，确认没人改
-XX:+UseMonitorInflation
-XX:GuaranteedAsyncDeflationInterval=1000   # 单位 ms
```

GuaranteedAsyncDeflationInterval 控制后台线程多久扫一遍尝试 deflate 闲置 monitor。默认值合理，**别去调这个参数**除非你 100% 知道在干什么。

**业务层**：避免大量短生命周期对象用 synchronized。如果一个对象只活几毫秒就被多线程争抢一下，膨胀成本远大于业务收益。

**结构改造**：高竞争场景考虑：
- `LongAdder` 替代 synchronized 的计数
- `ConcurrentHashMap` 替代 `synchronized(map)`
- `AtomicReference` + CAS 替代"读改写"型的锁

#### 验证

```bash
# 用 JFR 抓 monitor 事件
jcmd <pid> JFR.start name=mon settings=profile duration=120s filename=m.jfr

# 在 JMC 里看 "Java Monitor Inflation" / "Deflation" 事件
# 健康状态：膨胀和缩减数量大致平衡
```

> **口诀**：**膨胀容易，回收难**。JDK 15 之前膨胀就是单程票；JDK 15 之后有了缩减，但仍有滞后。代码里要避免"偶发竞争 + 长期独占"的反模式。

---

### 案例 ④ 同步代码块过大 → P99 飙升

#### 现象

某 RPC 接口 P99 从 50ms 飙到 800ms。jstack 抓栈：

```
"http-nio-exec-21" BLOCKED on java.lang.Object@1a2b3c
    at com.foo.UserService.update(UserService.java:120)
    - waiting to lock <0x000000076b4a3a40> (a java.lang.Object)

"http-nio-exec-7" RUNNABLE
    at java.net.SocketInputStream.socketRead0(Native Method)
    at com.foo.RemoteClient.fetchProfile(RemoteClient.java:48)
    - locked <0x000000076b4a3a40> (a java.lang.Object)
```

数十个 worker 都在 BLOCKED on 同一个 Object，而持有锁的 worker 在做远程 IO。

#### 根因

代码：

```java
public synchronized User update(long uid) {
    User u = userMap.get(uid);
    Profile p = remoteClient.fetchProfile(uid);   // ★ 阻塞 IO 在锁内
    u.applyProfile(p);
    userMap.put(uid, u);
    return u;
}
```

整个方法被 synchronized 包住，包括了一个 50ms 的远程调用。所有其他线程的 update 都在等这个远程 IO 完成。

ObjectMonitor 的 _EntryList 会越排越长。后到的请求 P99 = 平均锁等待时间 × 队列长度。

#### 修复

**第一步：缩小同步块范围**。锁只保护"必须保护"的临界区。

```java
public User update(long uid) {
    Profile p = remoteClient.fetchProfile(uid);   // ★ 远程 IO 移出锁外
    synchronized (this) {                          // 临界区只剩纯内存操作
        User u = userMap.get(uid);
        u.applyProfile(p);
        userMap.put(uid, u);
        return u;
    }
}
```

**第二步：分段锁**。如果不同 uid 的更新不应该互相阻塞，按 uid 分段。

```java
private final Object[] segments = new Object[16];
{ for (int i = 0; i < segments.length; i++) segments[i] = new Object(); }

public User update(long uid) {
    Profile p = remoteClient.fetchProfile(uid);
    Object lock = segments[(int)(uid & 15)];
    synchronized (lock) {
        // ...
    }
}
```

**第三步：用并发容器**。`ConcurrentHashMap` + `compute` 已经内部分段了，根本不用自己锁。

```java
private final ConcurrentHashMap<Long, User> userMap = new ConcurrentHashMap<>();

public User update(long uid) {
    Profile p = remoteClient.fetchProfile(uid);
    return userMap.compute(uid, (k, u) -> {
        u.applyProfile(p);
        return u;
    });
}
```

注意 `compute` 的回调是在该 key 的桶锁内执行的，**回调里不要做 IO/阻塞**，否则又退化成案例 ④ 自己。

#### 验证

```bash
# 修复前：抓 lock 火焰图
./profiler.sh -e lock -d 60 -f before.html <pid>
# 修复后再抓
./profiler.sh -e lock -d 60 -f after.html <pid>

# 期望 ObjectMonitor::EnterI 这一栈消失或大幅下降
```

更直接的指标：监控 `wait time / lock`，从 jstack 看 BLOCKED 线程数和栈分布。

> **口诀**：synchronized 的临界区**只能含纯内存操作**。任何 IO、远程调用、长计算都必须挪出去。

---

### 案例 ⑤ 虚拟线程的 pinning 事故

#### 现象

JDK 21 服务，业务用 `Executors.newVirtualThreadPerTaskExecutor()` 跑大量并发请求。监控显示吞吐远低于预期，且偶发"突然集体卡住"。

启动加 `-Djdk.tracePinnedThreads=full`，看到大量：

```
Thread[#27,VirtualThread[<unnamed>]/runnable@ForkJoinPool-1-worker-3]
    java.base/java.lang.VirtualThread.parkOnCarrierThread(VirtualThread.java:660)
    java.base/jdk.internal.misc.Unsafe.park(...)
    java.base/sun.nio.ch.SocketChannelImpl.read(...)
    com.foo.RemoteClient.send(RemoteClient.java:120)
    com.foo.UserService.update(UserService.java:50)  <== monitors:1
```

`<== monitors:1` 表示这个虚拟线程在 `synchronized` 块内做了阻塞 IO。

#### 根因

虚拟线程的工作原理：跑在少量 carrier（载体平台线程）上，遇到阻塞操作时**自动 unmount**（卸载），让 carrier 去跑别的虚拟线程。

但 JDK 21 的 `synchronized` 实现有个限制：**当虚拟线程持有 monitor 时，carrier 不能 unmount** —— 因为 monitor 的 _owner 字段直接绑定的是 carrier 平台线程的 ID，unmount 后这个 owner 就指错了。

后果：
- 该虚拟线程在 IO 上阻塞 50ms
- 它的 carrier（假设是 ForkJoinPool 里的某个平台线程）被钉住 50ms 不能调度
- 默认 carrier 数 = CPU 核数，假设 8 核
- 8 个 IO 同时发生 → 所有 carrier 都被钉死 → 整个虚拟线程系统停摆

#### 修复

**最简单**：把 `synchronized` 换成 `ReentrantLock`。`ReentrantLock` 内部基于 AQS，能正确处理虚拟线程的 mount/unmount。

```java
// ❌ pin 现场
public synchronized User update(long uid) {
    return remoteClient.fetch(uid);   // 阻塞 IO
}

// ✅ ReentrantLock 不会 pin
private final ReentrantLock lock = new ReentrantLock();

public User update(long uid) {
    lock.lock();
    try {
        return remoteClient.fetch(uid);
    } finally {
        lock.unlock();
    }
}
```

**更好**：把 IO 移出锁外（同案例 ④）。锁内零 IO，pin 不 pin 就无所谓了。

**最根本**：升级到 JDK 24。**JEP 491 "Synchronize Virtual Threads without Pinning"** 在 JDK 24 GA，把 monitor 的 owner 改成绑定虚拟线程而非 carrier，从根上解决 pin。

诊断工具：

```bash
# JDK 21
java -Djdk.tracePinnedThreads=short ...   # 简短：每个 pin 只打栈底一行
java -Djdk.tracePinnedThreads=full ...    # 完整：打全栈

# JFR
jcmd <pid> JFR.start settings=profile duration=60s filename=p.jfr
# 在 JMC 里看 "jdk.VirtualThreadPinned" 事件
```

#### 验证

```bash
# 修复前
java -Djdk.tracePinnedThreads=short ...
# 看到大量 pin 警告

# 修复后
# 期望：pin 警告归零；吞吐恢复
```

> **口诀**：JDK 21 上虚拟线程 + synchronized + 阻塞 IO = pin。JDK 24（JEP 491）解决前，把 synchronized 换成 ReentrantLock。

---

### 案例 ⑥ wait/notify 用错 —— "条件没在循环里"

#### 现象

生产者-消费者模式实现的队列，偶发数据丢失或消费者卡死。

```java
class BugQueue<T> {
    private final Queue<T> q = new LinkedList<>();
    private final int capacity;

    public synchronized void put(T item) throws InterruptedException {
        if (q.size() == capacity) {
            wait();                      // ★ if 不是 while
        }
        q.add(item);
        notify();                        // ★ notify 不是 notifyAll
    }

    public synchronized T take() throws InterruptedException {
        if (q.isEmpty()) {
            wait();                      // ★ if 不是 while
        }
        T item = q.remove();
        notify();
        return item;
    }
}
```

#### 根因

**问题 1：条件用 if 而非 while**

`wait()` 返回后，被唤醒的线程**不能假设条件仍成立**。原因：
- **虚假唤醒（spurious wakeup）**：JVM 实现允许 wait 在没有 notify 的情况下醒来（POSIX `pthread_cond_wait` 文档明确允许）
- **多消费者竞争**：consumer A 被 notify 醒来，还没 reacquire 锁，consumer B 已经把数据消费了。A 拿到锁时 q 已经空了

正确写法：

```java
while (q.isEmpty()) {
    wait();
}
```

**问题 2：notify 而非 notifyAll**

考虑这个场景：队列容量 5，2 个 producer + 2 个 consumer。
- consumer A、B 都在 take，q 空，都 wait
- producer X put 一条 → notify（只唤醒一个，假设是 A）
- producer Y put 一条 → notify（假设又唤醒 A，因为 A 还没拿到锁）→ B 永远不被唤醒

实际上 HotSpot 的 notify 选谁是不确定的，但理论上 **notify 可能反复唤醒同一个线程**，让其他线程饿死。

#### 修复

```java
class CorrectQueue<T> {
    private final Queue<T> q = new LinkedList<>();
    private final int capacity;

    public synchronized void put(T item) throws InterruptedException {
        while (q.size() == capacity) {     // ★ while
            wait();
        }
        q.add(item);
        notifyAll();                        // ★ notifyAll
    }

    public synchronized T take() throws InterruptedException {
        while (q.isEmpty()) {               // ★ while
            wait();
        }
        T item = q.remove();
        notifyAll();
        return item;
    }
}
```

**更好**：用 `Condition`（JUC 的版本）。可以分别为 "notFull" 和 "notEmpty" 创建独立 Condition，put 只唤醒 take（不会误唤醒别的 put），反之亦然。

```java
class BetterQueue<T> {
    private final Queue<T> q = new LinkedList<>();
    private final int capacity;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (q.size() == capacity) notFull.await();
            q.add(item);
            notEmpty.signal();        // 精确唤醒 take 等待者
        } finally { lock.unlock(); }
    }
    // take 类似
}
```

`signal` 比 `notifyAll` 减少惊群，但前提是 Condition 用对（每个 Condition 等待的"是同一个语义"）。

**最简单**：用 `ArrayBlockingQueue` / `LinkedBlockingQueue` —— JDK 已经写好的、正确的、性能调优过的实现。**不要重复造轮子**。

#### 验证

写一个能复现"条件失效"的并发测试：

```java
@Test
void noLostWakeups() throws Exception {
    var q = new BugQueue<Integer>(2);
    int n = 1_000_000;
    var producer = new Thread(() -> {
        for (int i = 0; i < n; i++) q.put(i);
    });
    var sum = new AtomicLong();
    var consumer1 = new Thread(() -> { for (int i = 0; i < n/2; i++) sum.addAndGet(q.take()); });
    var consumer2 = new Thread(() -> { for (int i = 0; i < n/2; i++) sum.addAndGet(q.take()); });
    producer.start(); consumer1.start(); consumer2.start();
    producer.join(); consumer1.join(); consumer2.join();
    assertEquals((long)n*(n-1)/2, sum.get());
}
```

错误版本会偶发卡死或数据错乱；正确版本通过。

> **口诀**：**wait 永远在 while 里；醒来重检条件；优先 notifyAll（或 Condition 精确 signal）**。三条都做不到的话，用 BlockingQueue。

---

### 案例 ⑦ 锁顺序不一致导致的死锁

#### 现象

转账业务偶发卡死，特征：两个用户互相转账时复现率最高。

```java
class Account {
    private long balance;

    public void transfer(Account to, long amount) {
        synchronized (this) {
            synchronized (to) {
                this.balance -= amount;
                to.balance += amount;
            }
        }
    }
}
```

`A.transfer(B)` 和 `B.transfer(A)` 同时发生 → 死锁。

#### 根因

经典哲学家就餐问题。事务 1 持有 A 等 B；事务 2 持有 B 等 A。

jstack 输出（节选）：

```
Found one Java-level deadlock:
=============================
"thread-1":
  waiting to lock monitor 0x00007f9b ... (object 0x000000076bbb0010, a Account),
  which is held by "thread-2"
"thread-2":
  waiting to lock monitor 0x00007f9b ... (object 0x000000076bbb0008, a Account),
  which is held by "thread-1"

Java stack information for the threads listed above:
===================================================
"thread-1":
    at Account.transfer(Account.java:11)
    - waiting to lock <0x000000076bbb0010> (a Account)
    - locked <0x000000076bbb0008> (a Account)
"thread-2":
    at Account.transfer(Account.java:11)
    - waiting to lock <0x000000076bbb0008> (a Account)
    - locked <0x000000076bbb0010> (a Account)
```

`Found one Java-level deadlock:` 是 JVM 的死锁检测器主动告诉你 —— jstack 内置了简单的环检测。

#### 修复

**核心策略：所有锁必须按全局一致的顺序获取**。

最常见的实现：用对象的 `identityHashCode` 排序。

```java
public void transfer(Account to, long amount) {
    Account first, second;
    int cmp = System.identityHashCode(this) - System.identityHashCode(to);
    if (cmp < 0) { first = this; second = to; }
    else if (cmp > 0) { first = to; second = this; }
    else {
        // hashCode 撞了，用一把"决斗锁"避免歧义
        synchronized (TIE_LOCK) {
            synchronized (this) { synchronized (to) {
                doTransfer(to, amount);
            }}
        }
        return;
    }
    synchronized (first) {
        synchronized (second) {
            doTransfer(to, amount);
        }
    }
}
```

注意 hashCode 撞了的兜底：identityHashCode 是 31 位，理论上有冲突可能。`TIE_LOCK` 是个全局静态对象，撞 hash 时所有线程在它上排队，避免错乱。

**更好**：用 `tryLock` 超时 + 退避。

```java
public boolean transfer(Account to, long amount) {
    while (true) {
        if (this.lock.tryLock()) {
            try {
                if (to.lock.tryLock(100, TimeUnit.MILLISECONDS)) {
                    try { doTransfer(to, amount); return true; }
                    finally { to.lock.unlock(); }
                }
            } catch (InterruptedException e) { Thread.currentThread().interrupt(); return false; }
            finally { this.lock.unlock(); }
        }
        // 没拿到，退避后重试
        Thread.sleep(ThreadLocalRandom.current().nextInt(10, 50));
    }
}
```

`synchronized` 没有 `tryLock`，所以这种退避策略**只能用 ReentrantLock**。

**最根本**：业务侧重新设计。比如转账走"单进单出账"模型 —— 一个全局账户队列串行处理所有转账，没有锁、没有死锁。

#### 验证

```bash
# 模拟死锁
# 用 jstack 周期性抓栈，搜 "Found one Java-level deadlock"
while true; do
    jstack <pid> | grep -A 2 "Found one Java-level deadlock" && break
    sleep 1
done

# 修复后跑同样测试，应该永远不出现
```

JFR 也能抓死锁事件：`jdk.JavaMonitorEnter` 配合 `wait time > N` 的过滤。

> **口诀**：**多锁场景必须有全局锁顺序**。`identityHashCode` 排序是最简单的实现；能用 `tryLock` 退避更稳。

---

### 案例 ⑧ String.intern() / Class 对象被业务误用为锁

#### 现象

某接口偶发卡死，jstack 看到大量线程 BLOCKED on 同一个 String 对象：

```
"http-nio-exec-12" BLOCKED
    at com.foo.OrderService.process(OrderService.java:45)
    - waiting to lock <0x00000007a1b2c3d4> (a java.lang.String)
```

但代码里**搜不到这个 String 在哪被加锁**。

#### 根因

代码：

```java
public void process(String orderId) {
    synchronized (orderId.intern()) {
        // 业务逻辑，本意是"按 orderId 串行化"
    }
}
```

`String.intern()` 返回的是 JVM 字符串常量池里的对象。**所有内容相同的 String，intern() 后是同一个对象**。

问题 1：常量池是 JVM 全局共享的。**别的 ClassLoader 加载的代码、第三方库、JVM 自身**，只要 intern 了相同字符串就会拿到同一个对象。锁竞争范围远超你想象。

问题 2：常量池的字符串**永远不被 GC**（JDK 7 后 intern 池在堆上，理论上可被回收，但只有当字符串完全不被引用时；业务代码持有引用就一直在）。每个 orderId 都 intern 一次，**字符串常量池无限增长 → OOM**。

更隐蔽的同类问题：

```java
synchronized (Order.class) { ... }   // ★ Class 对象是 JVM 全局唯一
```

`Order.class` 是该类的全局唯一实例。在它上加锁等于"全进程对该类的同步"。如果多个不同业务都对 `Order.class` 加锁，互相阻塞。

```java
synchronized ("LOCK") { ... }        // ★ 字符串字面量也是常量池对象
```

字面量字符串编译期就在常量池里。多个类的代码用同一字面量 → 同一把锁。

```java
synchronized (Long.valueOf(uid)) { ... }   // ★ -128~127 的 Long 是缓存的同一对象
```

`Long.valueOf` 对 `-128~127` 走 LongCache，返回缓存对象。两个 uid=1 的请求会争同一把锁；uid=200 各自一把。**行为和值有关，违反直觉**。

#### 修复

**铁律：锁对象必须是"业务私有的"且"生命周期可控的"**。

按 orderId 串行化的正确写法：

```java
private final ConcurrentHashMap<String, Object> locks = new ConcurrentHashMap<>();

public void process(String orderId) {
    Object lock = locks.computeIfAbsent(orderId, k -> new Object());
    synchronized (lock) {
        // ...
    }
}
```

但这样 `locks` 会无限增长。改进：

```java
// 用 Caffeine / Guava 的有界 LoadingCache
private final LoadingCache<String, Object> locks = Caffeine.newBuilder()
    .expireAfterAccess(10, TimeUnit.MINUTES)
    .maximumSize(10_000)
    .build(k -> new Object());

public void process(String orderId) {
    synchronized (locks.get(orderId)) {
        // ...
    }
}
```

注意：缓存淘汰会带来"先后两次拿同一 orderId 的锁不一定是同一个 Object"的风险 —— 如果在淘汰时刻有线程刚好持锁，新请求拿到新 lock，**临界区不互斥**。在线下评估这个风险。如果不能容忍，用 striped lock：

```java
// Guava Striped 锁，N 个 stripe，按 hash 分配
private final Striped<Lock> stripes = Striped.lock(256);

public void process(String orderId) {
    Lock lock = stripes.get(orderId);
    lock.lock();
    try { /* ... */ } finally { lock.unlock(); }
}
```

这种锁是固定数量，不会无限增长，但**不同 orderId 可能映射到同一 stripe**导致额外争抢。256 个 stripe 对一般业务足够。

#### 验证

```bash
# 修复前：抓 lock 火焰图，看是否存在 String / Class 对象的高频争抢
./profiler.sh -e lock -d 60 -f lock.html <pid>

# 字符串常量池占用
jcmd <pid> VM.metaspace
# 或
jmap -histo:live <pid> | grep java.lang.String
```

> **口诀**：**永远不要在全局共享对象（String、Class、Integer/Long 缓存对象）上加锁**。锁对象必须 `new` 出来私有的，最好放在业务的 Map / Striped 里管理。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 锁对象选择
- [ ] **不在 String / String.intern() / 字符串字面量上加锁**
- [ ] **不在 Class 对象 / Integer/Long 缓存对象上加锁**
- [ ] 锁对象是 `private final Object lock = new Object()` 或业务私有对象
- [ ] 同一资源的所有并发访问都用**同一把锁**（不要"读用 A 锁、写用 B 锁"）

### 临界区大小
- [ ] synchronized 块内**不含**远程调用 / 数据库 IO / 文件 IO
- [ ] 临界区只做"必须保护的最小操作"
- [ ] 大方法不要 `public synchronized void method()`，改为方法内的精确 synchronized 块

### wait/notify
- [ ] `wait()` 永远在 `while (condition)` 循环里，不是 `if`
- [ ] 默认用 `notifyAll()`；只在确定语义时才用 `notify()`
- [ ] 优先用 `Condition`（精确 signal）或 `BlockingQueue`（已实现的正确版）
- [ ] 不要在 wait/notify 上重复造队列轮子

### 死锁预防
- [ ] 多锁场景有**全局锁顺序**（推荐 identityHashCode 排序）
- [ ] 嵌套锁不超过 2 层
- [ ] 高风险场景用 `ReentrantLock.tryLock(timeout)` + 退避
- [ ] 周期性抓 jstack 检查 "Found one Java-level deadlock"

### 性能
- [ ] 高频计数用 `LongAdder`，不要 `synchronized` + ++
- [ ] 多线程 Map 用 `ConcurrentHashMap`，不要 `Collections.synchronizedMap`
- [ ] 单线程访问的容器用普通 `HashMap`，不要 `Hashtable`
- [ ] 高竞争场景考虑无锁（CAS）数据结构、分段锁、读写锁

### JVM 参数（按 JDK 版本）
- [ ] JDK 8/11：评估 `-XX:-UseBiasedLocking` 是否能改善 P99
- [ ] JDK 15~17：知道偏向锁默认关；旧代码可临时 `-XX:+UseBiasedLocking` 回退
- [ ] JDK 18+：偏向锁参数已删除，不能再加
- [ ] JDK 21：跑一次 `-Djdk.tracePinnedThreads=short` 验证虚拟线程没 pin
- [ ] JDK 24+：JEP 491 落地后可放心在虚拟线程里用 synchronized

### 监控
- [ ] JFR 持续开 `jdk.JavaMonitorEnter` / `jdk.JavaMonitorWait` 事件
- [ ] 监控 `java.lang:type=Threading` 的 `BlockedCount` / `BlockedTime`
- [ ] 应用层埋点：关键临界区的入锁/出锁时间，监控 P99
- [ ] 死锁检测：周期 jstack + 关键字告警

### 虚拟线程
- [ ] JDK 21 上：`synchronized` 块内**严禁阻塞 IO**
- [ ] 高竞争锁优先 `ReentrantLock` 而非 `synchronized`（虚拟线程友好）
- [ ] 业务里有 IO 的代码段评估能否用虚拟线程，CPU 密集的部分仍用平台线程池

---

## 6. 还能往下挖什么

读完上面 8 个案例只是起点。如果想再深一层：

### HotSpot 内部
- **JEP 491（JDK 24）的实现**：把 ObjectMonitor 的 owner 从"绑定 carrier 平台线程"改为"绑定虚拟线程或平台线程二选一"，关键提交在 `objectMonitor.cpp` 和 `Continuation` 框架的交互。
- **`LightweightSynchronization`**：JEP 425 之后的 HotSpot 重构，把 BasicLock / ObjectMonitor 的边界进一步统一。看 `lightweightSynchronizer.hpp` 的演化。
- **MonitorDeflation 详细机制**：`MonitorList::deflate` 的具体扫描策略，与 ZGC/G1 的并发标记如何配合。
- **`BiasedLocking` 历史**：JDK 18 之前的 `biasedLocking.cpp` 仍在 OpenJDK Git 历史里，对比删除前后能学到"如何安全删除一个核心特性"。

### Loom 与并发
- **Continuation 与 Monitor 的边界**：`VirtualThread.unmount` 的判定逻辑、`yieldContinuation` 在 monitor 内为什么不能调用、JEP 491 怎么破除这个限制。
- **StructuredTaskScope（JEP 505）**：结构化并发把"作用域"显式化，配合 Lock 时如何避免"作用域外仍持锁"的反模式。
- **ScopedValue（JEP 506）**：ThreadLocal 的不可变替代，在 synchronized 块内的传播语义。

### 锁的现代演化
- **RCU（Read-Copy-Update）思想**：Linux 内核大量用 RCU 替代读多写少的锁。Java 的 `CopyOnWriteArrayList` / `volatile` 引用 swap 是简化版 RCU。
- **HTM（Hardware Transactional Memory）**：Intel TSX 提供的硬件事务，理论上能让无竞争的临界区零开销。HotSpot 试过 `-XX:+UseRTMLocking`，但实际生产收益有限。
- **MCS Lock / CLH Lock**：AQS 的祖先，公平队列锁的两个经典实现。读 Doug Lea 的论文能理解 AQS 为什么这么设计。

### ZGC / G1 与 Monitor 释放
- **ZGC 的 Concurrent Class Unloading**：类卸载时怎么处理"该类对象上的 monitor"，看 `ZUnload` 的实现。
- **G1 的 Remark 阶段**：会扫描 monitor 引用的对象，长队列的 monitor 会让 Remark 变慢。
- **JFR `jdk.GCPhasePause` + `jdk.JavaMonitorWait` 联看**：定位"GC 让 monitor 等待时间剧增"的场景。

### 分布式锁
- **本地 synchronized 解决不了跨机问题**：Redis Redlock、ZooKeeper EPHEMERAL 节点、etcd Lease 是常见替代。
- **Redlock 的争议**：Martin Kleppmann vs antirez 的争论是分布式锁正确性的经典讨论，强烈推荐读完。

---

## 7. 引用

- OpenJDK 21 源码：[markWord.hpp](https://github.com/openjdk/jdk21u/blob/master/src/hotspot/share/oops/markWord.hpp)、[objectMonitor.cpp](https://github.com/openjdk/jdk21u/blob/master/src/hotspot/share/runtime/objectMonitor.cpp)、[synchronizer.cpp](https://github.com/openjdk/jdk21u/blob/master/src/hotspot/share/runtime/synchronizer.cpp)
- JEP 374: [Disable and Deprecate Biased Locking](https://openjdk.org/jeps/374)
- JEP 410: [Remove the Experimental AOT and JIT Compiler](https://openjdk.org/jeps/410)（间接相关，含 Biased Locking 删除背景）
- JEP 444: [Virtual Threads](https://openjdk.org/jeps/444)
- JEP 491: [Synchronize Virtual Threads without Pinning](https://openjdk.org/jeps/491)
- Doug Lea, *The java.util.concurrent Synchronizer Framework*（AQS 论文，2005）
- Dave Dice, *Implementing Fast Java Monitors with Relaxed-Locks*（Sun Labs 2001，HotSpot 锁优化思想源头）
- Brian Goetz, *Java Concurrency in Practice*（并发圣经，第 13~14 章详解 Lock）
- [JOL（Java Object Layout）](https://openjdk.org/projects/code-tools/jol/)：观察 mark word 的官方工具
- [async-profiler](https://github.com/async-profiler/async-profiler)：抓 lock / wall / cpu 火焰图
- 你假笨, [《synchronized 的轻量级锁的解锁过程为什么使用 cas 而不是 volatile？》](https://www.javadoop.com/) —— 国内 HotSpot 锁源码解读最深的系列

---

> 上一篇综述：[并发编程总览](/article/java/concurrent#2-synchronized-与-lock) ｜ 配套：[JMM](/article/jvm/jmm) ｜ [线程池生产实战](/article/java/threadpool)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
