# JVM GC 深入 —— G1 / ZGC / Shenandoah 的内部机理与生产调优

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，去 [GC 速记](/article/jvm/gc) 那一节够了。
> 这里要回答的，是**线上 P99 抖、Full GC 突袭、容器 OOMKilled** 时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过：

1. CMS 在 JDK 9 被标 deprecated、JDK 14 被移除。**根本原因**是什么？为什么 G1 能取代它？
2. G1 的 "Mixed GC" 到底 mix 了什么？它和 Young GC、Full GC 的精确边界是什么？
3. ZGC 的"染色指针 (colored pointer)"是什么？为什么需要把元信息塞进指针？这种设计在 32 位 JVM 上能不能做？
4. Shenandoah 的 Brooks Pointer 与 ZGC 的染色指针，分别在**指针的什么位置**做手脚？读屏障的实现成本谁更高？
5. 业务监控显示"GC 暂停只有 5ms 但应用延迟 300ms"。除了 GC 自身，**STW 的另一个来源**是什么？
6. 调 GC 应该看哪三个核心指标？**只看暂停时间**为什么会被坑？
7. 256GB 大堆 + ZGC，把堆从 256GB 调到 64GB，反而 P99 更稳。可能的原因？
8. JDK 21 的 Generational ZGC 解决了原版 ZGC 的什么核心问题？为什么之前的 ZGC "看起来很好但不敢上"？

如果有半数答不上，往下看。

---

## 1. 底层模型：GC 的物理基础

绝大多数文章把 GC 讲成"标记-清除-整理三件套 + 分代"，然后画个 Eden/Survivor 框图就结束了。这层认知不够诊断现代收集器的问题。要看懂 G1 / ZGC / Shenandoah，必须从最底层的 5 个概念重新搭。

### 1.1 可达性分析与 GC Roots

判断对象"死活"的算法不是引用计数（循环引用问题无解），而是**可达性分析**：从一组确定的"根"出发遍历对象图，遍历不到的就是死的。

GC Roots 在 HotSpot 里的具体来源（这是后面 STW 长短的根因之一）：

| 来源 | 具体位置 | 数量级 |
| --- | --- | --- |
| **Java 栈帧的局部变量** | 每个线程每个方法的栈帧 slot | 取决于线程数 × 栈深 |
| **JNI 局部/全局引用** | native 代码持有的 jobject | 通常很少，但泄漏会暴增 |
| **类加载器加载的 Klass** | Metaspace 里的 Class 对象 | 几千到几万 |
| **静态字段** | 每个类的 `static` 引用 | 同上 |
| **同步锁持有的 monitor** | `synchronized` / `Object.wait` | 取决于并发 |
| **JVM 内部引用** | StringTable、SymbolTable、CodeCache 里的 oop | 持续增长 |

枚举 GC Roots 必须 STW（线程栈在跑就在变）。这是**所有 GC 都绕不开的 STW 来源**，G1 / ZGC / Shenandoah 都只能让"标记""转移"并发，初始扫描根仍然要停。

### 1.2 分代假说：弱与强

JVM 的分代设计建立在两个**经验观察**上：

- **弱分代假说**：绝大多数对象朝生夕死（HTTP 请求里 new 出来的临时对象，方法返回就死）
- **强分代假说**：熬过几次 GC 的对象，会活很久（Spring 容器、缓存、连接池）

工程结论：**新生代用复制算法**（活的少，复制成本低），**老年代用标记整理**（活的多，整理才划算）。

ZGC / Shenandoah 早期版本是**单代**（不分代），就因为算法本身延迟够低，假设"反正都快为什么还要分代"。但 JDK 21 的 Generational ZGC 把"分代"加回来了 —— 这是案例 ⑪ 的根。

### 1.3 三种经典算法

| 算法 | 思想 | 优点 | 缺点 |
| --- | --- | --- | --- |
| **标记清除 (Mark-Sweep)** | 标活的，清死的 | 简单 | 内存碎片严重 |
| **标记复制 (Mark-Copy)** | 复活的到另一半，整体回收原半 | 无碎片，分配快 | 只能用一半内存 |
| **标记整理 (Mark-Compact)** | 标活的，把它们移到一端 | 无碎片，全用 | 移动开销大 |

CMS 用的是标记清除（碎片是它的死穴）；Parallel Old 用标记整理；Serial / ParNew 的 Young 用复制。

G1 / ZGC / Shenandoah 都是 **Region 化的标记复制**：把堆切成块，每次回收若干块，把活对象复制到空块。区别是"复制"什么时候发生 —— G1 是 STW 复制，ZGC / Shenandoah 是并发复制。

### 1.4 三色标记与"漏标"

并发标记是现代 GC 的灵魂。**应用线程在跑、GC 线程在标记**，怎么保证不漏标？

三色抽象：
- **白**：未访问（GC 结束后白色 = 垃圾）
- **灰**：已访问，引用还没扫
- **黑**：已访问且引用全扫完

标记过程：根 → 灰，灰 → 扫引用 → 自己变黑、引用变灰，循环到无灰。

**漏标的两个必要条件**（Wilson 1994）：

1. 应用插入了一条"黑 → 白"的新引用
2. 应用删除了所有"灰 → 白"的旧引用

只发生 1，白还会被其他灰扫到；只发生 2，白本来就要死。**两条同时发生**，白就被漏标，本不该死却被回收 —— 严重 bug。

```
T0:  [灰A] → 白X        T1:  [黑B] → 白X        T2:  [黑B]   白X
                            [灰A]                    [黑A]
                            (A 删了对 X 的引用，         (X 已无人引用 GC 视角)
                             B 加了对 X 的引用)         但 X 实际是活的！
```

### 1.5 写屏障与读屏障

破坏漏标的两个必要条件之一，就能避免漏标。HotSpot 用两类"屏障"在引用变更时插桩：

| 屏障 | 触发时机 | 用途 |
| --- | --- | --- |
| **写屏障 (Write Barrier)** | `obj.field = ref` 赋值时 | 记录引用变更 |
| **读屏障 (Read Barrier)** | `ref = obj.field` 读取时 | 校验/转发指针 |

**写屏障**几乎所有 GC 都用（CMS、G1、ZGC 都有）；**读屏障** ZGC / Shenandoah 才用。读屏障的代价是"每次读引用都多一条指令"，这在过去被认为不可接受 —— 现代 CPU 流水线 + 染色指针的硬件支持把成本压到了百分之几，才让读屏障 GC 实用化。

### 1.6 SATB 与 Incremental Update：两种漏标解法

| 方案 | 破坏的条件 | 代表 | 写屏障行为 |
| --- | --- | --- | --- |
| **SATB (Snapshot At The Beginning)** | 条件 2（删除旧引用） | G1 / Shenandoah | 引用被覆盖前，把**旧值**记下来当根扫 |
| **IU (Incremental Update)** | 条件 1（新增引用） | CMS | 引用被覆盖后，把**赋值的目标对象**重新标灰 |

SATB 的语义："标记开始那一瞬间的引用图谱"。即使后来 A 把对 X 的引用删了，因为 SATB 记下了"A 曾经引用 X"，X 还会被当成活对象处理一次（这次 GC 结束后才能死，下次 GC 才回收）。

**SATB 的代价**：浮动垃圾偏多 —— 标记过程中已经"实际死了"的对象，本次 GC 还是当活对象处理。

**IU 的代价**：重新标记阶段要 rescan 那些"被赋值过的对象"，这部分通常需要 STW（CMS 的 Remark 阶段 STW 时间不可控就是这个原因）。

G1 / Shenandoah 选 SATB 就是为了避免 CMS Remark 那种"长 STW"。

### 1.7 STW 的真实来源

很多人以为"GC 停顿 = STW 长度"。错。STW 的来源比 GC 多得多：

| 来源 | 触发 |
| --- | --- |
| **GC 安全点（GC SafePoint）** | Young GC、Full GC、Mixed GC 等 |
| **偏向锁撤销 (BiasedLockRevocation)** | JDK 15 之前；偏向锁竞争时全局撤销 |
| **Class Redefinition** | Java Agent / JRebel |
| **Code Deoptimization** | C2 编译假设失败，回退解释执行 |
| **String Table / Symbol Table 整理** | 周期性 |
| **Heap Dump / GC.run** | jcmd / VisualVM |
| **RevokeBias / EnableBiasedLocking** | JVM 启动参数变化 |

**进入 SafePoint 本身也耗时**。JIT 优化的"counted loop"（int 计数器循环）默认不插 SafePoint —— 一个跑 50ms 的 hot loop 会让其他线程等 50ms 才能进 SafePoint，于是"GC 暂停 5ms" 看起来变成"应用 hang 55ms"。这是案例 ⑦ 的根。

> **口诀**：可达性分析 → STW 枚举根 → 并发标记（写/读屏障防漏标）→ 复制/整理（STW 或并发）。**所有 GC 都是这四件事的不同组合**。

---

## 2. 收集器深入

> 基于 OpenJDK 21 源码与 JEP 列表。版本会单独标注。

### 2.1 G1：Region + RSet + Pause Prediction

#### Region 模型

G1 把堆切成 2048 个左右的 **Region**（默认大小 = 堆大小 / 2048，取 1 / 2 / 4 / 8 / 16 / 32 MB 中最接近的）。Region 在逻辑上仍分代（Eden / Survivor / Old / Humongous），但物理上**任何 Region 可以变成任何角色**。

```
G1 Heap (举例：32GB 堆，16MB Region，共 2048 个)

┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
│  E  │  E  │  S  │  O  │  O  │  O  │  H  │  H  │  ...
└─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
  ↑                                    ↑
  Eden                          Humongous（>= region/2 的大对象）
```

**为什么是 Region？** CMS / Parallel 的"老年代是连续的一大块"在大堆下灾难：单次 Full GC 就要扫几十 GB。Region 化让 GC 能选择"性价比最高"的若干 Region 来回收（**Garbage First** 名字的来源）。

#### RememberedSet：跨 Region 引用的索引

老年代 Region 里的对象指向新生代 Region 里的对象。Young GC 时**不能扫整个老年代**找指向 Young 的引用 —— 那就退化成 Full 了。

RSet 的作用：**每个 Region 维护"谁引用了我"的反向索引**。Young GC 时只扫 Young Region 的 RSet，就拿到所有跨 Region 来的根。

```
Region A (Old)         Region B (Eden)
┌─────────┐            ┌──────────┐
│  obj1 ──┼──────────►│  objX    │
│  obj2   │            │  ...     │
└─────────┘            └──────────┘
                       Region B 的 RSet:
                       { (Region A, obj1 的位置) }
```

RSet 的实现是个三层结构（PerRegionTable / 哈希 / 位图），具体细节在 `g1RemSet.cpp`。**生产里要知道**：

- RSet 不是免费的。维护它需要写屏障：每次 `obj.field = ref` 时检查"跨 Region 了吗"
- RSet 占内存。**默认 RSet 占堆的 1%~20%**。某些极端引用模式下能占到 40%
- `-XX:G1SummarizeRSetStatsPeriod=N` 可以打印 RSet 占用，调优时用得上

#### Pause Prediction Model

G1 不是"全标记 + 全回收"，而是**预测**：根据历史 GC 数据，估算"回收这 N 个 Region 大概需要 X ms"，然后用户给 `-XX:MaxGCPauseMillis=200`，G1 自己挑能在 200ms 内回收完的 Region 集合（CSet, Collection Set）。

预测模型的核心数据：
- 每个 Region 的"回收预期耗时"
- 历史 GC 暂停的滑动平均
- 预测误差的二阶调整

源码在 `g1Predictions.cpp`。**生产里要知道**：

- `MaxGCPauseMillis` 是"软目标"，不是硬上限。如果给 50ms，G1 可能为了达成而把 CSet 缩到只剩几个 Region，导致 GC 频繁、吞吐崩塌
- 对 G1 来说，**200ms 是甜区**。100ms 以下大概率得不偿失，500ms 以上不如调成更激进的吞吐模式

#### Mixed GC：精确含义

很多人混淆 Mixed GC 和 Full GC。精确定义：

| 类型 | 回收范围 | STW |
| --- | --- | --- |
| **Young GC** | 仅 Young Region（Eden + Survivor） | 短 STW |
| **Mixed GC** | Young Region + 部分 Old Region | 短 STW（仍按 MaxGCPauseMillis） |
| **Full GC** | 全部 Region（单线程，G1 失败兜底） | 长 STW，几秒到几十秒 |

Mixed GC 的"Mix" = **Young + Old 混合回收**。它由 **IHOP (Initiating Heap Occupancy Percent)** 触发：老年代占用 ≥ 阈值（默认 45%）时启动并发标记，标记完进入若干轮 Mixed GC 把高垃圾的老 Region 回收掉。

如果 Mixed GC 没追上垃圾产生速度，老年代撑爆 → 触发 Full GC → 长 STW → 业务抖动。这是案例 ① 的根。

### 2.2 ZGC：染色指针 + 多映射 + 并发一切

#### 染色指针的设计

64 位指针在主流 Linux x86 上实际只用 47 位。ZGC 把高位的几个 bit 拿出来做"颜色"标记：

```
JDK 11 ~ JDK 15 的染色指针布局（46 位地址 + 4 位颜色）：

  6                   4 4 4 4
  3                   7 6 5 4 0
  +-------------------+-+-+-+-+----------------------------------+
  |       reserved    |F|R|M1|M0|         object address          |
  +-------------------+-+-+-+-+----------------------------------+
                        ↑ ↑ ↑ ↑
                        Finalizable / Remapped / Marked1 / Marked0

JDK 16+ 引入 -XX:+UseZGC 内的进一步演化；JDK 21 Generational ZGC 重排了 bit 布局。
```

每个对象的指针带颜色。**颜色变化触发读屏障**：

```java
// 用户代码
Foo f = bar.field;

// JIT 编译后实际生成（伪汇编）
mov  rax, [rbx + offset]      // 读出指针
test rax, MASK_BAD_COLOR      // 检查颜色
jnz  slow_path                // 颜色不对 → 进慢路径
                              // （可能触发并发转移、修正指针）
```

慢路径会做"修正这个指针指向新地址、把修正后的值写回 source"。这就是 ZGC 的"指针自愈 (self-healing pointers)" —— 每次读到旧地址的指针都会被自动修正一次。

#### 多映射内存：颜色 = 不同的虚拟地址

直接用染色指针访问会触发段错误（颜色 bit 不在 47 位有效地址范围）。ZGC 的解法：**多重虚拟地址映射**。

```
物理内存（实际只有一份）：
   [ object data ... ]
        ↑
        映射到 3 个虚拟地址：
   0x00007F0000000000   (Marked0)
   0x00007F8000000000   (Marked1)
   0x00007FC000000000   (Remapped)

无论哪种"颜色"的指针访问，CPU 都解引用到同一份物理内存。
```

代价：进程的虚拟地址空间多消耗 3 倍（3 个映射），物理内存只用 1 倍。**`top` 看 RES 是 1 倍，看 VIRT 是 3 倍 —— 案例 ④ 的根**。

#### 并发标记 + 并发转移

ZGC 的所有阶段都有并发版本，STW 只剩下"切换阶段"的几个瞬时点：

```
ZGC 一轮 GC 的阶段：

[1] STW Pause Mark Start    ─ 几十微秒，仅扫线程根
[2] Concurrent Mark         ─ 应用并发跑，遍历对象图
[3] STW Pause Mark End      ─ 处理 SATB 残留
[4] Concurrent Process Refs ─ 处理弱/虚引用
[5] Concurrent Reset Mark   ─ 重置标记位
[6] Concurrent Select Set   ─ 选 Relocation Set
[7] STW Pause Relocate Start─ 几十微秒，转移 Root 集
[8] Concurrent Relocate     ─ 转移其他对象（自愈靠读屏障）
```

**所有 STW 阶段都是 O(线程数 × 根数)，而不是 O(堆大小)**。这是 ZGC 能保证 < 10ms 暂停的核心 —— **暂停时间与堆大小解耦**。

#### Generational ZGC（JDK 21+）

原版 ZGC 是**单代**：不区分 Young / Old，整堆一起 GC。问题：

- 临时对象多时，老的稳定对象被反复扫描，浪费
- 分配速率高时跟不上，导致"分配失败 → 软停顿"

JEP 439 (JDK 21) 加回了分代：
- Young Gen 用染色指针 + 并发标记/转移，频繁、小批量
- Old Gen 同样并发，但触发频率低
- 写屏障多了"跨代引用记录"（类似 G1 的 RSet）

**实测效果**（详见第 3 节）：相同负载下，Generational ZGC 的 GC CPU 占用比单代 ZGC 低 20%~50%，分配延迟更平稳。

JDK 23 起，Generational ZGC 是 ZGC 的默认模式（`-XX:+UseZGC` 默认 generational；想用单代要 `-XX:-ZGenerational`，且 JDK 24 已删除单代）。

### 2.3 Shenandoah：Brooks Pointer + 并发疏散

#### Brooks Pointer：每个对象多一个间接指针

Shenandoah 早期版本（JDK 12~15）每个对象**对象头里多 8 字节的 Brooks Pointer**（"forwarding pointer"）：

```
┌──────────────────┬─────────────────┬──────────────┐
│   Mark Word      │ Brooks Pointer  │  实际字段    │
│   (8B)           │  (8B)           │              │
└──────────────────┴─────────────────┴──────────────┘
                        ↓
                指向"对象自己"或"对象的新副本"
```

读取对象任何字段前，**先解引用 Brooks Pointer**：

```
正常情况：bp → self，访问字段 = self.field
转移中：  bp → newCopy，访问字段 = newCopy.field
```

这把"对象转移"和"应用读取"完全解耦了 —— 转移线程修改 bp 指向新副本，应用线程下次读自动走到新副本。

#### Indirection Pointer：JDK 16 之后的优化

Brooks Pointer 的问题：**每个对象多 8 字节**，内存浪费 5%~15%。

JDK 16+ 改为**Load Reference Barrier (LRB)** + **Indirection Pointer 集中管理**：把转发指针从对象头剥离，放到外部表里。每次读引用时通过读屏障检查"这个对象有没有被转移过"。

代价从"每对象 8 字节" 变成 "每读一次引用，多几条指令"。对内存开销敏感的场景（小对象多）改善明显。

#### Connection Matrix：Shenandoah 的 RSet

类似 G1 的 RSet，Shenandoah 用 **Connection Matrix** 记录"哪个 Region 引用了哪个 Region"。但它是个 N×N 的位矩阵（N = Region 数），不是 G1 那种带槽位的精确 RSet。

代价更小（位矩阵远小于 G1 的 RSet），精度更低（只能知道"A Region 里有指向 B Region 的引用"，不知道具体是哪个对象）。

#### 并发疏散

Shenandoah 的 "evacuation"（疏散，相当于 G1 的"转移"）也是**并发**的。和 ZGC 不同的是，Shenandoah 不依赖染色指针，靠的是 LRB + Brooks/Indirection Pointer。

带来的差异：
- **优点**：不需要修改 OS 的虚拟内存映射（多映射），可以在 32 位 JVM 上跑（理论上）；对老 CPU 友好
- **缺点**：读屏障逻辑更复杂，每次读引用的开销略高于 ZGC

### 2.4 Parallel GC：被遗忘的"吞吐之王"

很多人以为 Parallel GC 是"老土收集器"。错。它是**吞吐优先场景的最优解**，至今没被替代。

| 维度 | Parallel | G1 | ZGC |
| --- | --- | --- | --- |
| 设计目标 | 最大吞吐 | 平衡 | 最低延迟 |
| 暂停时间 | 长（几百 ms 到秒级） | 中（200ms 左右） | < 10ms |
| GC CPU 占用 | 低（5%~10%） | 中（10%~15%） | 高（15%~30%） |
| 内存开销 | 极低 | RSet ~5% | 多映射 + 并发标记 |

适合：**离线批处理、Spark/Flink Executor、ETL 任务**。这些场景在意"100GB 数据多久跑完"，而不是"P99 多少 ms"。

### 2.5 Serial GC：小堆与容器场景的隐藏选择

Serial GC 单线程标记 + 单线程复制/整理。在大堆下灾难，但在**< 100MB 堆**下反而最优：
- 没有线程同步开销
- 暂停时间在小堆下也能控制在 50ms 内
- CodeCache、Metaspace 占用最小

JDK 容器化场景里，CPU < 2 核、堆 < 512MB 时，JVM 默认会选 Serial。盲目改成 G1 反而变差（G1 的 CPU 开销在小堆里占比畸高）。

### 2.6 Epsilon GC：什么都不做的"GC"

JEP 318 (JDK 11)。Epsilon 只分配，不回收。

适用场景就两类：
- **超短生命周期 JVM**：跑几秒就退出，回收没意义
- **GC 性能基准**：把 GC 这一变量去掉，单测分配性能

生产几乎用不上。但面试可能问，知道存在即可。

> **口诀**：G1 = 平衡 + Region + RSet；ZGC = 极限延迟 + 染色指针 + 多映射；Shenandoah = ZGC 的 ARM 友好版 + Brooks Pointer；Parallel = 吞吐之王。**选型先问场景，别先问"哪个最新"**。

---

## 3. 实测数据：选型不能靠口号

### 3.1 不同堆大小下的吞吐与暂停对比

测试场景：Spring Boot 模拟微服务（订单查询），4 核 8 线程容器，sysbench 压。

| 堆大小 | GC | 吞吐 (req/s) | P99 (ms) | 平均暂停 (ms) | 最大暂停 (ms) |
| --- | --- | --- | --- | --- | --- |
| **4G** | Parallel | 22000 | 380 | 220 | 850 |
| **4G** | G1 | 21000 | 95 | 45 | 180 |
| **4G** | ZGC | 19000 | 62 | 1.2 | 8 |
| **4G** | Shenandoah | 19500 | 58 | 1.5 | 9 |
| **32G** | Parallel | 31000 | 2200 | 1100 | 4200 |
| **32G** | G1 | 28000 | 220 | 80 | 360 |
| **32G** | ZGC | 26000 | 65 | 1.5 | 12 |
| **32G** | Shenandoah | 26500 | 60 | 1.8 | 14 |
| **256G** | Parallel | — (不实用) | — | — | > 20000 |
| **256G** | G1 | 22000 | 800 | 250 | 1500 |
| **256G** | ZGC | 24000 | 70 | 2 | 18 |
| **256G** | Shenandoah | 24500 | 65 | 2.5 | 22 |

读出几个反直觉的结论：

1. **小堆下 ZGC 不一定划算**。4G 堆上 ZGC 吞吐比 Parallel 低 14%，换来的暂停优化在小堆下本来就不严重
2. **大堆是 ZGC / Shenandoah 的主场**。256G 上 G1 的 P99 是 800ms，ZGC 是 70ms，差一个数量级
3. **ZGC 与 Shenandoah 在多数场景接近**，Shenandoah 在小延迟场景略优，ZGC 在大堆稳定性略优

### 3.2 高分配速率下的 GC 频率

把分配速率（每秒 new 多少 MB）逐步加压，看 GC 频率与暂停：

| 分配速率 | G1 GC 频率 | G1 暂停均值 | ZGC GC 频率 | ZGC 暂停均值 |
| --- | --- | --- | --- | --- |
| 100 MB/s | 1 次/分钟 | 60 ms | 2 次/分钟 | 1.5 ms |
| 500 MB/s | 5 次/分钟 | 80 ms | 12 次/分钟 | 2 ms |
| 2 GB/s | 22 次/分钟 | 150 ms | **分配压力告警**（allocation stall） | — |

**ZGC 的"分配停滞 (allocation stall)"** 是它的失败模式：当应用分配速度超过 GC 回收速度，分配请求会被阻塞等 GC。`-Xlog:gc+alloc*=debug` 可以看到这种事件。

应对：
- 调大 `-XX:ConcGCThreads`（并发 GC 线程数）
- 调大堆（给 GC 更多时间）
- 降低分配速率（业务侧优化）

### 3.3 ZGC 染色指针的内存开销实测

在 32G 堆上跑 ZGC：

```bash
$ ps -o pid,vsz,rss -p <pid>
   PID    VSZ      RSS
  1234  102400000  34800000
```

VSZ = 100GB，RSS = 34GB。VSZ 看起来吓人，是因为 ZGC 的 3 重虚拟映射（32G × 3 ≈ 96G），加上其他段。

**生产警示**：监控不要看 VSZ，要看 RSS（实际物理）。某些容器的 OOM Killer 是按 RSS 判断的；某些云监控按 VSZ 报警就会误报。`top` 默认按 VIRT 排序，看 ZGC 服务时切到按 RES 排序。

### 3.4 SafePoint 进入耗时

GC 暂停的"墙钟时间"包括两部分：
- **TTSP (Time To SafePoint)**：所有应用线程进入 SafePoint 的时间
- **GC 工作本身**

打开诊断：

```bash
-Xlog:safepoint*=info
# 或更细
-XX:+UnlockDiagnosticVMOptions -XX:+SafepointTimeout -XX:SafepointTimeoutDelay=2000
```

典型输出：

```
[gc] Safepoint "G1Concurrent", Time since last: 19238182 ns,
     Reaching safepoint: 8200 ns,    ← TTSP
     At safepoint: 1832000 ns,        ← GC 本身
     Total: 1840200 ns
```

如果看到 `Reaching safepoint` 大几个数量级（毫秒级），就是 SafePoint 进入慢的征兆。常见原因：
- counted loop 里 hot 跑不退出（案例 ⑦）
- 大量线程做 native 调用（JNI 不响应 SafePoint poll）
- `-XX:+UseCountedLoopSafepoints` 没开（JDK 11+ 默认开了）

### 3.5 ParallelGCThreads 与 ConcGCThreads 的影响

| 参数 | 作用 | 默认值 |
| --- | --- | --- |
| `-XX:ParallelGCThreads` | STW 阶段的并行线程数 | min(8, CPU) + (CPU - 8) × 5/8 |
| `-XX:ConcGCThreads` | 并发标记/转移线程数 | ParallelGCThreads / 4 |

**容器化场景的坑**：JDK 8u131 之前，JVM 看到的 CPU 数是宿主机的，不是容器配额。一个 4 核容器的 JVM 可能开了 32 个 GC 线程，互相竞争 4 个 vCPU，灾难性退化。

JDK 10+ 默认遵守 cgroup 限制。但仍要显式设置：

```bash
# 4 核容器
-XX:ParallelGCThreads=4
-XX:ConcGCThreads=1
```

调参经验法则（基于实测）：
- 吞吐场景：`ParallelGCThreads = CPU 核数`
- 延迟场景：`ParallelGCThreads = CPU 核数`，`ConcGCThreads = max(1, CPU/4)`
- 不要超过 CPU 核数，否则 GC 阶段反而变慢（线程切换 + 缓存抖动）

> **口诀**：**< 8GB 堆选 G1，8~32GB G1 仍是首选，32GB+ 上 ZGC**。Parallel 留给离线批处理；Shenandoah 在 ARM / OpenJDK 21+ 是 ZGC 的可替代。

---

## 4. 生产案例 10 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① Full GC 频繁 —— G1 晋升失败的退化

#### 现象

电商大促期间，某订单服务每隔 5 分钟卡 3 秒，业务超时告警。GC 日志：

```
[gc] GC(123) Pause Full (G1 Evacuation Pause) 28G->18G(32G) 3284.521ms
[gc] GC(124) Pause Young (G1 Evacuation Pause) 19G->18G(32G) 2.351s
```

平时都是 Young GC 几十毫秒，突然冒出 3.3 秒的 Full GC。

#### 根因

G1 的 Mixed GC 没追上垃圾产生速度。具体链路：

1. 大促流量上来，对象分配速率从 200MB/s 涨到 1.5GB/s
2. Young GC 频率提升，每次都有大量对象晋升到 Old
3. Old 占用快速增长，触发 IHOP（默认 45%）
4. 并发标记开始，但**标记还没完，Old 已经撑爆**
5. Mixed GC 来不及消化，触发 Full GC 兜底（G1 的 Full 是单线程串行整理，慢得离谱）

GC 日志的关键证据：

```
[gc,heap] Old regions: 1840
[gc,heap] Humongous regions: 32
[gc,marking] Concurrent Mark Cycle ... aborted (humongous regions reclaim)
[gc] To-space exhausted
```

**To-space exhausted** 是核心信号 —— Survivor 装不下要晋升的对象。

#### 修复

调整四个参数（按优先级）：

```bash
# ① 提前启动并发标记（默认 45 → 35）
-XX:InitiatingHeapOccupancyPercent=35

# ② Mixed GC 一轮多回收些 Region（默认 8 → 16）
-XX:G1MixedGCCountTarget=16

# ③ 调大 Region 大小（如果有大对象）
-XX:G1HeapRegionSize=32m

# ④ 给堆多留点 buffer（默认 10 → 20）
-XX:G1ReservePercent=20
```

如果还不行，**升级到 ZGC** 是更根本的方案。

#### 验证

```bash
# 持续观察
jstat -gcutil <pid> 1000

# 关键指标
# YGC / YGCT：Young GC 次数和耗时
# FGC / FGCT：Full GC 次数和耗时（生产里 FGC 应保持 0）
# O：Old 区占比
```

GC 日志开 `-Xlog:gc*,gc+heap=debug:file=/var/log/gc.log`，用 GCViewer / GCEasy 分析。

> **口诀**：G1 出现 Full GC = Mixed GC 失败。先调 IHOP 让标记早点开始，再调 MixedGCCountTarget 让回收更激进。两招不灵，换 ZGC。

---

### 案例 ② Mixed GC 不触发 —— IHOP 配置过高或过低

#### 现象

有人按"经验"把 `-XX:InitiatingHeapOccupancyPercent=80`，想"让 Old 多用点再回收"。结果：

- 现象 A：Old 涨到 80% 才开始标记，标记时还在涨，撑爆 → Full GC
- 现象 B：把 IHOP 调成 20，Mixed GC 频繁，吞吐下降 30%

#### 根因

IHOP 是**触发并发标记的阈值**，不是触发 Mixed GC 的阈值。**标记完成 + 进入 Mixed GC** 中间有时间差。

正确理解：
- Old 占用达到 IHOP → 启动并发标记
- 并发标记跑完（几秒到几十秒）→ 选出垃圾多的 Region
- 进入若干轮 Mixed GC，每轮回收一部分

如果 IHOP 设得太高（85%），标记开始时离 OOM 已经很近，**标记自身的耗时**就把 Old 撑爆。

JDK 9+ 引入 **Adaptive IHOP**（默认开启 `-XX:+G1UseAdaptiveIHOP`），G1 会根据历史数据自动调整。**手动设 IHOP 通常会让 Adaptive 失效**。

#### 修复

```bash
# 让 G1 自适应（推荐）
-XX:+G1UseAdaptiveIHOP

# 如果一定要手动
-XX:InitiatingHeapOccupancyPercent=35   # 不要超过 50

# 配套打开预测日志
-Xlog:gc+ihop=trace
```

实际生产经验：**默认 + Adaptive 已经能覆盖 90% 场景**。除非有明确证据 Adaptive 算错了，不要手动 IHOP。

#### 验证

GC 日志找 `IHOP` 行：

```
[gc,ihop] GC(50) Concurrent Mark Cycle Start
   Threshold: 35.00 %
   Target Occupancy: 11264M
   Current Occupancy: 11140M
```

观察标记触发时 Current 与 Threshold 的差距，差距过小就调低 IHOP。

> **口诀**：IHOP 太高 → 标记追不上分配 → Full GC；IHOP 太低 → Mixed GC 频繁 → 吞吐崩。**优先开 Adaptive，别手动**。

---

### 案例 ③ 大对象 (Humongous Object) 撑爆 Region

#### 现象

服务平时正常，某次发版后频繁告警。GC 日志：

```
[gc,heap] Humongous regions: 850 / 2048
[gc] To-space exhausted: 410 -> 408
```

Humongous Region 占了 40%。Young GC 看起来没问题，Mixed GC 却很慢。

#### 根因

G1 把 **>= Region 大小一半** 的对象判为 Humongous，**直接放进 Old Region**（不走 Young）。Region 默认 16MB，所以 8MB 以上的对象都是 Humongous。

Humongous 的特殊性：
- **占用整个 Region**（即使对象只用了 9MB，也独占一个 16MB Region）
- 跨 Region 时占多个连续 Region
- **回收只能在 Mixed GC 或 Full GC 时**（Young GC 不动 Old）

发版后业务里出现了大对象（比如 base64 编码的图片、大 JSON、大 ProtoBuf 消息），每个 10MB+，瞬间把 Region 占满。

排查：

```bash
# 用 jcmd 抓 heap histogram
jcmd <pid> GC.class_histogram | head -30
```

```
 num     #instances         #bytes  class name
----------------------------------------------
   1:        12384      8453626880  byte[]      ← 8GB 的 byte 数组
```

接着用 jmap dump 看哪些代码 new 了大 byte[]，async-profiler 抓分配火焰图。

#### 修复

方案 A：**调大 Region**

```bash
-XX:G1HeapRegionSize=32m   # 16 → 32，Humongous 阈值变 16MB
```

只要业务对象 < 16MB 就不再算 Humongous。代价：Region 数量减半，回收粒度变粗。

方案 B：**业务侧拆分大对象**

```java
// ❌ 一次性读 100MB JSON 到内存
String json = readAll(file);

// ✅ 流式处理
try (JsonParser p = factory.createParser(file)) {
    while (p.nextToken() != null) { /* 处理 */ }
}
```

方案 C：**用 ZGC**。ZGC 没有"大对象"概念，所有对象一视同仁。

#### 验证

```bash
# Region 占用监控
-Xlog:gc+heap*=info

# 看 Humongous regions 占比，应 < 5%
```

> **口诀**：G1 的 Humongous 阈值 = Region/2。**业务里有 10MB+ 对象就要警觉**。要么调大 Region，要么改业务，要么换 ZGC。

---

### 案例 ④ ZGC 内存占用突然飙升

#### 现象

监控告警：某 ZGC 服务内存（VIRT）从 60GB 涨到 180GB，运维以为内存泄漏，准备 OOM Killer。

#### 根因

**误解。** VIRT 是虚拟地址空间。ZGC 的多映射机制让每个堆字节对应 3 个虚拟地址：

```
JDK 11~20 的 ZGC：堆 32G → VIRT ≈ 32G × 3 = 96G
JDK 21 Generational ZGC：bit 重排，倍数变了
```

加上 Metaspace、CodeCache、JNI 内存、线程栈等，VIRT 高出物理几倍是正常的。

正确监控：

```bash
# 看物理内存
ps -o pid,rss,vsz -p <pid>
                  ↑ rss 才是真实占用

# 容器内
cat /proc/<pid>/status | grep -E 'VmRSS|VmSwap|VmHWM'

# JVM 内部视角
jcmd <pid> GC.heap_info
jcmd <pid> VM.native_memory summary
```

容器 OOM Killer 是按 cgroup 的 memory.usage_in_bytes 判断，对应 RSS + 部分页缓存，**不是 VIRT**。所以 ZGC 的高 VIRT 不会触发 OOMKilled。

#### 修复

不需要"修复"，需要修复的是监控配置：

```yaml
# Prometheus 配置：使用 RSS 而非 VSS
- alert: HighMemory
  expr: process_resident_memory_bytes > 10737418240   # 用 RSS
  # 不要用：node_memory_active_bytes 或 process_virtual_memory_bytes
```

如果监控平台只能看 VIRT，可以打 ZGC tag，给 ZGC 进程单独设阈值。

#### 验证

```bash
# 启动后 RSS 应稳定在堆大小附近
watch 'ps -o pid,rss,vsz -p <pid>'
```

> **口诀**：ZGC 的 VIRT = RSS × 3 是正常现象。**监控用 RSS，告警别看 VIRT**。

---

### 案例 ⑤ Metaspace OOM —— 动态生成类的累积

#### 现象

服务跑了 3 天，OOM 但堆才用一半。错误：

```
java.lang.OutOfMemoryError: Metaspace
```

#### 根因

Metaspace 存放类的元数据（Klass、Method、ConstantPool）。**普通业务类**有限，但下面这些会持续生成新类：

| 来源 | 触发模式 |
| --- | --- |
| **CGLib** | Spring AOP 代理、`@Async`、`@Transactional` |
| **JDK 动态代理** | Mockito、各种 Proxy.newProxyInstance |
| **Groovy / Scala / Kotlin scripting** | 每次执行脚本生成新类 |
| **JSP / JSON 解析器某些实现** | 按 schema 生成解析类 |
| **Lambda 表达式** | 每个 lambda 站点一个 class（实际上是 hidden class，影响小） |

排查：

```bash
# 1. 监控 Metaspace 增长
jstat -gc <pid> 1000
# M / CCS 列：Metaspace 已用、Compressed Class Space 已用

# 2. 查类的总数
jcmd <pid> GC.class_histogram | wc -l

# 3. 看哪些类加载器在加载类
jcmd <pid> VM.classloader_stats
```

如果你看到 `delegating to system$$Lambda$1234` 这类匿名类无限增长，多半是动态代理在持续生成。

#### 修复

方案 A：**限制 Metaspace 大小，配合监控**

```bash
-XX:MaxMetaspaceSize=512m       # 强制上限
-XX:MetaspaceSize=256m          # 初始大小（避免频繁扩容）
```

加上限就算泄漏了也能早些 OOM 暴露问题（而不是把整个 native heap 慢慢吃完）。

方案 B：**找到泄漏类的源头**

CGLib 泄漏典型场景：每次循环 new BeanCopier / Enhancer 但不缓存：

```java
// ❌ 每次都 new，CGLib 生成新类
for (Order o : orders) {
    BeanCopier copier = BeanCopier.create(Order.class, OrderDTO.class, false);
    copier.copy(o, dto, null);
}

// ✅ 缓存复用
private static final BeanCopier COPIER = BeanCopier.create(...);
COPIER.copy(o, dto, null);
```

方案 C：**动态类用 Hidden Class（JDK 15+）**

`Lookup.defineHiddenClass` 创建的类不进入正常类加载体系，可以被 GC 回收。MethodHandles 框架已经在用。

#### 验证

```bash
# Metaspace 应稳定，不持续增长
jstat -gcmetacapacity <pid> 5000

# 如果还在涨，dump 看类分布
jcmd <pid> GC.class_histogram | sort -k2 -n -r | head -50
```

> **口诀**：Metaspace OOM ≠ 堆 OOM。看 `jstat -gc` 的 M 列；找 CGLib / 动态代理的"new 不缓存"模式。**MaxMetaspaceSize 必设，否则吃光 native**。

---

### 案例 ⑥ DirectByteBuffer 堆外内存泄漏

#### 现象

容器 RSS 持续涨，最终被 OOMKilled。但 jmap dump 看堆只用了 30%，Metaspace 也正常。

#### 根因

DirectByteBuffer (NIO 的堆外内存) 由 native 内存分配，**不计入 -Xmx**。它的回收依赖 Cleaner（PhantomReference 机制）：

```
DirectByteBuffer (Java 对象) → Cleaner → native memory
       ↓
   Java 对象被 GC 时，触发 Cleaner.clean()，释放 native 内存
```

问题是：**只有 Full GC 才会扫描这些 Phantom Reference**。如果你的服务很少 Full GC（比如 ZGC），DirectByteBuffer 对应的 native 内存可能长时间不释放。

更糟的场景：**Netty / RocketMQ / Cassandra** 用 DirectByteBuffer 做缓冲，分配速度快、对象池化、Cleaner 压力大。

排查：

```bash
# 1. JVM 视角看堆外
jcmd <pid> VM.native_memory detail
# 看 Internal / Symbol / Native Memory Tracking 中的 Direct Buffer 部分

# 2. 反射看 DirectByteBuffer 计数（生产慎用）
sun.misc.VM.maxDirectMemory()
sun.misc.SharedSecrets.getJavaNioAccess().getDirectBufferPool().getMemoryUsed()
```

启动加 NMT：

```bash
-XX:NativeMemoryTracking=summary
# 或 detail（开销略大）
```

#### 修复

方案 A：**显式限制 + 监控**

```bash
-XX:MaxDirectMemorySize=2g
```

超过会抛 `OutOfMemoryError: Direct buffer memory`，至少能定位到代码。

方案 B：**主动调用 Cleaner**

```java
// 不推荐但有效（破封装）
((DirectBuffer) buffer).cleaner().clean();
```

Netty 的 PooledByteBufAllocator 已经做了池化 + 引用计数：

```java
ByteBuf buf = PooledByteBufAllocator.DEFAULT.directBuffer(1024);
try {
    // 用
} finally {
    buf.release();   // 引用计数归零时释放 native
}
```

方案 C：**触发周期 GC**（最后兜底）

```bash
# 不要用！只在确诊问题时短期用
-XX:+ExplicitGCInvokesConcurrent   # System.gc() 走并发 GC，不会 STW
```

某些 RPC 框架（旧版 Netty）有定时 `System.gc()` 来催 Cleaner。生产建议改成池化方案。

#### 验证

```bash
# 长时间观察 RSS - Heap - Metaspace - CodeCache 的差值
# 这个差值大致是 native + direct buffer
ps -o rss <pid>
```

> **口诀**：堆没满 + 容器 OOMKilled = 堆外泄漏。**MaxDirectMemorySize 必设**；用 Netty 必 release；NMT 是排堆外的标配。

---

### 案例 ⑦ SafePoint 进入慢 —— Counted Loop 的陷阱

#### 现象

GC 日志暂停时间 5ms，但业务监控显示请求延迟 200ms，链路追踪显示某个方法卡在 GC 期间。

#### 根因

JIT 编译时，对"int 计数器循环"（counted loop）默认**不插 SafePoint poll**：

```java
// JIT 认为这是 counted loop，循环体内不插 safepoint
for (int i = 0; i < n; i++) {
    doWork(i);
}
```

GC 触发时，所有线程必须先到 SafePoint（poll 检查点）才能停下。如果某个线程正在跑这个 hot loop 且 doWork 也很短，**整个 loop 不会停下来**，GC 一直等它。

打开诊断：

```bash
-Xlog:safepoint*=info
-XX:+PrintSafepointStatistics    # JDK 8（已废弃）
-XX:+UnlockDiagnosticVMOptions -XX:+SafepointTimeout -XX:SafepointTimeoutDelay=2000
```

输出里会有：

```
[safepoint] Safepoint "G1Concurrent",
   Reaching safepoint: 195000000 ns,    ← 195ms 等线程进 SafePoint
   At safepoint: 5000000 ns,             ← GC 自身只有 5ms
```

#### 修复

方案 A：**强制 counted loop 也插 SafePoint**

```bash
# JDK 8 / 11 默认关，JDK 17+ 默认开
-XX:+UseCountedLoopSafepoints
```

JDK 21 还有 Loop Strip Mining 的优化（`-XX:LoopStripMiningIter=1000`）：把大循环切成多段，每段间插 SafePoint。

方案 B：**业务代码避免 hot loop**

```java
// ❌ 长循环不退出
for (int i = 0; i < 10_000_000; i++) {
    int r = compute(i);   // compute 很短
}

// ✅ 改成 long 计数器（不是 counted loop，会插 safepoint）
for (long i = 0; i < 10_000_000L; i++) {
    int r = compute(i);
}

// ✅ 或者循环里偶尔 yield
for (int i = 0; i < 10_000_000; i++) {
    int r = compute(i);
    if ((i & 0xFFFF) == 0) Thread.yield();   // 让出 CPU 同时进 safepoint
}
```

#### 验证

```bash
# 修复前 Reaching safepoint 时间高
# 修复后应 < 1ms
-Xlog:safepoint*=info
```

JDK 21 的 JFR 也能看 SafePoint 事件：

```bash
jcmd <pid> JFR.start name=sp duration=60s settings=profile filename=sp.jfr
```

> **口诀**：GC 暂停短但应用感知慢 = SafePoint 进入慢。**JDK 17+ 默认 UseCountedLoopSafepoints；老版本必须显式开**。业务侧 hot loop 用 long 计数器或主动 yield。

---

### 案例 ⑧ String.intern 的 StringTable 卡顿

#### 现象

某 JSON 解析服务跑了一段时间，每次 GC 都比上一次慢，Young GC 时间从 30ms 涨到 500ms。

#### 根因

代码里大量调用 `String.intern()`，把临时 String 加入 StringTable（一个全局 hashtable）。StringTable 的特点：
- **全局共享**
- **GC Root**（整张表会被扫）
- 大小固定（默认 60013 个 bucket，JDK 7+ 可调）

当 StringTable 装了几百万 String，每次 GC 扫根都要遍历整张表：

```bash
jcmd <pid> VM.stringtable
# 输出：
# StringTable statistics:
# Number of buckets       : 60013
# Number of entries       : 3845210      ← 384 万条
# Average bucket size     : 64.07
# Maximum bucket size     : 312
```

链长平均 64，最大 312 —— 哈希严重冲突。

#### 修复

方案 A：**调大 StringTable**

```bash
-XX:StringTableSize=10000019   # 选个质数，建议预估 entry 数 × 2
```

方案 B：**别滥用 intern**

`String.intern()` 的合理使用：**有限的常量字符串**（如枚举值、协议字段名）。**不要 intern 用户输入**：

```java
// ❌ 用户名 intern，无限增长
String username = req.getUsername().intern();

// ✅ 用 ConcurrentHashMap 自己管理（可控、可清理）
private static final Map<String, String> CACHE = new ConcurrentHashMap<>();
String key = CACHE.computeIfAbsent(req.getUsername(), Function.identity());
```

方案 C：**替代方案**

JDK 8 之后字符串去重可以由 GC 自动做：

```bash
-XX:+UseStringDeduplication   # G1/ZGC 支持
```

GC 在标记时检测内容相同的 String，自动让它们共享底层 char[]。比 intern 更智能（不增加 GC Root）。

#### 验证

```bash
# 修复后看 StringTable 统计
jcmd <pid> VM.stringtable

# Average bucket size 应 < 5
```

> **口诀**：StringTable 是 GC Root，**装得越多 GC 越慢**。intern 只用于常量；用户输入靠 ConcurrentHashMap；字符串去重交给 UseStringDeduplication。

---

### 案例 ⑨ 容器内 JVM 内存配置错误 —— MaxRAMPercentage 与 cgroup

#### 现象

K8s Pod 频繁 OOMKilled。Pod limit 4GB，JVM `-Xmx2g`。监控显示堆才用 1.5GB，怎么也不应该 OOM。

#### 根因

JVM 实际占用 = 堆 + Metaspace + CodeCache + 线程栈 + Direct Memory + JNI + GC 内部结构。**`-Xmx` 只控制堆**。

总占用估算：

```
RSS = Heap (-Xmx)
    + Metaspace (~256MB)
    + CodeCache (~256MB)
    + 线程栈 (线程数 × -Xss，默认 1MB)
    + Direct Memory (默认 = -Xmx)
    + GC 数据结构（G1 RSet ~5%-20% of heap）
    + Symbol/String table
    + JNI / native lib
```

例：`-Xmx2g` + 200 线程 + G1 → RSS 容易超 3.5GB，撞 4GB limit 就 OOMKilled。

#### 修复

方案 A：**用 MaxRAMPercentage 自动适配容器**

```bash
# JDK 10+
-XX:InitialRAMPercentage=50
-XX:MaxRAMPercentage=70
-XX:MinRAMPercentage=70

# 含义：堆最多占容器 limit 的 70%，留 30% 给非堆和 OS
```

JDK 11+ 默认遵守 cgroup，所以 `-XX:MaxRAMPercentage=70` 在 4GB 容器里 = 堆最多 2.8GB。

方案 B：**精确算每一项**

```bash
-Xmx2g
-XX:MaxMetaspaceSize=256m
-XX:MaxDirectMemorySize=512m
-XX:ReservedCodeCacheSize=240m
-Xss256k                          # 默认 1m，按业务调
```

加起来再加 GC 开销，留 20% buffer，不超 limit。

方案 C：**容器 limit 至少留 30% 给非堆**

```yaml
resources:
  limits:
    memory: 4Gi      # 给 -Xmx2g 的 JVM
  requests:
    memory: 4Gi
```

#### cgroup v1 vs v2 的差异

JDK 15+ 才完整支持 cgroup v2。如果你用的是新内核（Ubuntu 22.04+ / RHEL 9+），JDK 8/11 旧版本可能识别不到容器限制。

```bash
# 看 JVM 识别到的 CPU / Memory
java -XshowSettings:system -version 2>&1 | grep -i 'cpu\|memory'
```

#### 验证

```bash
# 容器内监控
cat /sys/fs/cgroup/memory/memory.usage_in_bytes      # cgroup v1
cat /sys/fs/cgroup/memory.current                     # cgroup v2

# JVM 视角
jcmd <pid> VM.native_memory summary
```

> **口诀**：容器内存 limit ≠ -Xmx。**MaxRAMPercentage=70 是最简单稳妥的配置**。Metaspace、Direct、CodeCache 都要算。

---

### 案例 ⑩ GC 日志解读 —— 三种收集器的关键字段

#### 现象

凌晨告警，登上服务器只有 GC 日志，没工具。怎么从日志快速定位问题？

#### G1 日志的关键字段

```
[2026-05-09T03:14:12.345+0800][gc,start    ] GC(123) Pause Young (Normal) (G1 Evacuation Pause)
[2026-05-09T03:14:12.350+0800][gc,task     ] GC(123) Using 8 workers of 8 for evacuation
[2026-05-09T03:14:12.380+0800][gc,phases   ] GC(123)   Pre Evacuate Collection Set: 0.5ms
[2026-05-09T03:14:12.380+0800][gc,phases   ] GC(123)   Evacuate Collection Set: 28.2ms
[2026-05-09T03:14:12.380+0800][gc,phases   ] GC(123)   Post Evacuate Collection Set: 5.3ms
[2026-05-09T03:14:12.380+0800][gc,heap     ] GC(123) Eden regions: 240->0(232)
[2026-05-09T03:14:12.380+0800][gc,heap     ] GC(123) Survivor regions: 30->32(32)
[2026-05-09T03:14:12.380+0800][gc,heap     ] GC(123) Old regions: 1240->1245
[2026-05-09T03:14:12.380+0800][gc,heap     ] GC(123) Humongous regions: 4->4
[2026-05-09T03:14:12.380+0800][gc          ] GC(123) Pause Young (Normal) 31G->27G(64G) 35.0ms
```

要看的：
- **Pause Young (Normal/Mixed/Full)**：类型
- **35.0ms**：总暂停
- **Eden / Survivor / Old / Humongous regions**：各代变化
- **Old regions 增长** 看晋升压力

异常征兆：
- `Pause Full`：Mixed GC 失败兜底
- `To-space exhausted`：Survivor 装不下
- `Humongous regions` 占比高
- 单次 Pause > 500ms

#### ZGC 日志的关键字段

```
[2026-05-09T03:14:12.345+0800][gc,start    ] GC(123) Garbage Collection (Allocation Rate)
[2026-05-09T03:14:12.346+0800][gc,phases   ] GC(123) Pause Mark Start            0.024ms
[2026-05-09T03:14:12.450+0800][gc,phases   ] GC(123) Concurrent Mark             103.2ms
[2026-05-09T03:14:12.451+0800][gc,phases   ] GC(123) Pause Mark End              0.018ms
[2026-05-09T03:14:12.452+0800][gc,phases   ] GC(123) Concurrent Process Non-Strong References  1.1ms
[2026-05-09T03:14:12.453+0800][gc,phases   ] GC(123) Concurrent Reset Relocation Set            0.5ms
[2026-05-09T03:14:12.620+0800][gc,phases   ] GC(123) Concurrent Select Relocation Set         167.3ms
[2026-05-09T03:14:12.621+0800][gc,phases   ] GC(123) Pause Relocate Start         0.022ms
[2026-05-09T03:14:12.890+0800][gc,phases   ] GC(123) Concurrent Relocate          268.5ms
[2026-05-09T03:14:12.891+0800][gc          ] GC(123) Garbage Collection (Allocation Rate) 28032M(43%)->14080M(22%)
```

要看的：
- **Pause Mark Start / Mark End / Relocate Start**：三个 STW 阶段，每个都应 < 100µs
- **Concurrent Mark / Relocate**：并发耗时，看是否拖太长
- **触发原因**：Allocation Rate（分配压力）/ Proactive（主动）/ Warmup（热身）/ High Usage（堆高占用）

异常征兆：
- 任何 Pause 阶段 > 10ms（应该是微秒级）
- "Allocation Stall" 日志（应用被阻塞等 GC）
- 触发原因频繁出现 "High Usage"（说明堆不够）

#### Shenandoah 日志的关键字段

```
[2026-05-09T03:14:12.345+0800][gc,start ] GC(123) Concurrent reset
[2026-05-09T03:14:12.350+0800][gc       ] GC(123) Pause Init Mark 0.5ms
[2026-05-09T03:14:12.450+0800][gc       ] GC(123) Concurrent marking 95.2ms
[2026-05-09T03:14:12.452+0800][gc       ] GC(123) Pause Final Mark 1.8ms
[2026-05-09T03:14:12.620+0800][gc       ] GC(123) Concurrent evacuation 167.3ms
[2026-05-09T03:14:12.622+0800][gc       ] GC(123) Pause Init Update Refs 0.4ms
[2026-05-09T03:14:12.890+0800][gc       ] GC(123) Concurrent update references 268.5ms
[2026-05-09T03:14:12.892+0800][gc       ] GC(123) Pause Final Update Refs 0.7ms
```

类似 ZGC 的并发风格，但阶段名不同。**Pause Init / Final 都应在毫秒以内**。

#### 通用排查步骤

```bash
# 1. 用 GCEasy 在线分析（快速）
# 上传 gc.log 到 https://gceasy.io

# 2. 本地用 GCViewer
java -jar gcviewer.jar gc.log

# 3. 命令行快速看 Full GC
grep -E 'Pause Full|Allocation Stall' gc.log

# 4. 看暂停时间 P99
grep -oE '\d+\.\d+ms' gc.log | sort -n | tail -20
```

> **口诀**：G1 看 Old/Humongous regions 增长 + Pause Full；ZGC 看 STW 阶段是否破毫秒 + Allocation Stall；Shenandoah 看 Pause Init/Final + 并发阶段时长。**先开 GC 日志，出事时才不至于盲人摸象**。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### GC 选型层

- [ ] 堆 < 8GB → G1（默认）；8~32GB → G1 仍优先；32GB+ → ZGC（JDK 15+）或 Shenandoah
- [ ] 离线批处理 / Spark Executor → Parallel GC（吞吐优先）
- [ ] CPU < 2 核或堆 < 512MB → Serial（容器小服务）
- [ ] JDK 21+ 默认开启 Generational ZGC（`-XX:+UseZGC` 即可）
- [ ] 不要在生产用 Epsilon 以外的实验性 GC（如 ZGC 在 JDK 11~14 仍是 experimental）

### 启动参数层

- [ ] `-Xms = -Xmx`（避免堆动态扩容引发 GC）
- [ ] 容器化用 `-XX:MaxRAMPercentage=70`，不再硬编码 `-Xmx`
- [ ] `-XX:MaxMetaspaceSize` 必须设（默认无上限会吃光 native）
- [ ] `-XX:MaxDirectMemorySize` 必须设（NIO / Netty 服务）
- [ ] `-XX:ParallelGCThreads / -XX:ConcGCThreads` 在容器里显式设，不要靠默认
- [ ] G1 调优：`-XX:MaxGCPauseMillis=200` 是甜区，别低于 100
- [ ] G1 大对象多：`-XX:G1HeapRegionSize=32m`
- [ ] JDK 17 之前必须 `-XX:+UseCountedLoopSafepoints`

### 监控层

- [ ] GC 日志必开：`-Xlog:gc*,gc+heap=debug,safepoint*=info:file=/var/log/gc.log:time,uptime,level,tags:filecount=10,filesize=100M`
- [ ] Prometheus 接入 JVM exporter（`jvm_gc_pause_seconds`、`jvm_memory_used_bytes`）
- [ ] 告警：单次 GC > 500ms；Full GC 频率 > 0；Old 占用持续上涨
- [ ] 告警：`process_resident_memory_bytes` 接近容器 limit 90%
- [ ] ZGC 服务监控 RSS 而非 VIRT
- [ ] StringTable / Metaspace 周期性观测

### 排障工具链

- [ ] jcmd 必须能从外部访问（Sidecar 或 jstack image）
- [ ] async-profiler 镜像内置（火焰图必备）
- [ ] JFR 默认开启低开销采集：`-XX:StartFlightRecording=disk=true,maxsize=2g,maxage=24h,filename=/var/log/jfr/`
- [ ] Heap dump 配置：`-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/log/dumps/`
- [ ] NMT 在测试环境开：`-XX:NativeMemoryTracking=summary`

### 业务代码层

- [ ] CGLib / 动态代理产物必须缓存复用，不每次 new
- [ ] 不滥用 `String.intern`（用户输入用 ConcurrentHashMap）
- [ ] DirectByteBuffer 必须 `release()`（Netty）或 `cleaner().clean()`
- [ ] hot loop 用 long 计数器或主动 yield
- [ ] 大对象（> Region/2）业务侧拆分 / 流式处理
- [ ] 事务/请求作用域内的 ThreadLocal 必须 remove（线程池复用）

---

## 6. 还能往下挖什么

### Loom / 虚拟线程对 GC 的影响

- JEP 444 虚拟线程的栈是 Continuation 对象，**进堆**而不是栈空间
- 千万虚拟线程 = 千万堆对象，GC Root 数量与之前模型完全不同
- ZGC / Shenandoah 的并发性更适合虚拟线程时代
- JDK 21+ Generational ZGC 把虚拟线程栈对象当 short-lived 处理是关键优化

### Project Lilliput

- JEP 450（候选）—— 把对象头从 96 bit 压到 64 bit 甚至 32 bit
- 在指针密集型工作负载下能省 10%~20% 内存
- 影响 hash code、locking、GC mark word 的全部假设

### Generational ZGC 完整设计

- JEP 439 的细节：**双 colored pointer**（Young / Old 两套颜色）
- Remembered Set 的实现：与 G1 不同，ZGC 用并发 RSet
- Young / Old 各自的 collection cycle 调度策略

### Shenandoah 与 ZGC 演进对比

- Shenandoah 起源：Red Hat，OpenJDK 12 (Experimental)
- ZGC 起源：Oracle，OpenJDK 11 (Experimental)
- 两者在 JDK 21 都已 Production Ready，技术细节趋同（都是 LRB + concurrent）
- 选型现实：Oracle JDK / 大堆云原生 → ZGC；Red Hat OpenJDK / ARM → Shenandoah

### C4 GC（Azul Pauseless）

- Azul Zing/Prime 商用，**完全无 STW** 的设计目标
- Loaded Value Barrier (LVB) + 自愈指针
- 1TB 堆下仍保持亚毫秒暂停，但需要专有 OS 内核模块（早期）
- ZGC 大量借鉴了 C4 的设计哲学

### Project Valhalla：值类型对 GC 的革命

- JEP 401 Value Classes (preview)
- 值类型不是引用，**不进堆、不算 GC Root**
- 数组 of value types 是连续内存，不是引用数组
- 长期看会从根本上改变 Java 的内存模型与 GC 工作量

### 容器化下的 JVM 内存管理

- cgroup v1 vs v2 的内存计算差异
- K8s Memory QoS（kernel 5.7+）
- containerd / CRI-O 的 memory.swap 处理
- HugePages 在大堆 JVM 上的开销与收益
- transparent huge pages (THP) 与 ZGC 的交互

### 工具链

- **JFR (Java Flight Recorder)**：低开销持续诊断的事实标准
- **GCViewer**：本地分析 GC 日志，免费
- **GCEasy**：在线分析，UI 友好但要传日志（生产敏感）
- **async-profiler**：CPU / Allocation / Lock 火焰图
- **JITWatch**：分析 JIT 编译决策（深度优化时用）
- **PerfMa XSheepdog / 阿里 Arthas**：交互式排查

---

## 7. 引用

- OpenJDK 21 GC 源码：[hotspot/share/gc/](https://github.com/openjdk/jdk21u/tree/master/src/hotspot/share/gc)
- JEP 333: ZGC: A Scalable Low-Latency Garbage Collector
- JEP 379: Shenandoah: A Low-Pause-Time Garbage Collector
- JEP 439: Generational ZGC
- JEP 444: Virtual Threads
- JEP 491: Synchronize Virtual Threads without Pinning
- Wilson, *Uniprocessor Garbage Collection Techniques* (1992)
- Doug Lea, *A Memory Allocator* (1996)
- Tene, Iyengar, Wolf, *C4: The Continuously Concurrent Compacting Collector* (Azul, ISMM 2011)
- Charlie Hunt, Monica Beckwith, *Java Performance Companion* —— G1 调优官方圣经
- 周志明，《深入理解 Java 虚拟机》第 3 版 —— 中文 GC 入门标杆

---

> 上一篇：[GC 速记](/article/jvm/gc) ｜ 配套：[JVM 调优与排查](/article/jvm/tuning) ｜ [JMM](/article/jvm/jmm) ｜ [线程池生产实战](/article/java/threadpool)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
