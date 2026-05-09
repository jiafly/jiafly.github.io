# 垃圾收集机制（速记）

> JVM 三大子系统：类加载、运行时数据区、执行引擎，而 **GC 就是执行引擎里最常被问的部分**。本文混合"面试八股"和"系统讲解"，一遍读懂。
>
> G1 / ZGC / Shenandoah 内部机理、染色指针、Brooks Pointer、SafePoint Bias、10 个 GC 生产事故 → **[JVM GC 深入（深文）](/article/jvm/gc-deep)**

---

## 1. 一句话八股

> **垃圾收集 = 找出不再被引用的对象 + 回收它们占用的内存。**
> 现代 HotSpot 主要使用 **可达性分析 + 分代收集 + 多种收集器组合（CMS / G1 / ZGC / Shenandoah）**。

---

## 2. 怎么判断对象"死了"？

### 2.1 引用计数法（已不用）
每个对象维护一个引用计数器，引用 +1，失效 -1，归零即死。
**致命问题**：循环引用无法回收。HotSpot 不使用此法。

### 2.2 可达性分析（GC Roots）
从一组 **GC Roots** 出发向下搜索，搜不到的对象 = 不可达 = 可回收。

可作为 GC Roots 的对象：

| GC Root 类型 | 说明 |
| --- | --- |
| 虚拟机栈中引用的对象 | 方法局部变量、方法参数 |
| 方法区静态属性引用的对象 | `static` 字段 |
| 方法区常量引用的对象 | `final` 常量、字符串常量池 |
| 本地方法栈 JNI 引用的对象 | Native 方法持有的对象 |
| 同步锁持有的对象 | `synchronized` 锁住的对象 |

---

## 3. 四种引用强度（高频考点）

| 引用类型 | 回收时机 | 典型用途 |
| --- | --- | --- |
| **强引用** Strong | 永不回收（除非不可达） | `Object o = new Object();` |
| **软引用** Soft | 内存不足时回收 | 缓存（图片、字典） |
| **弱引用** Weak | 下一次 GC 必回收 | `WeakHashMap`、`ThreadLocal` 的 key |
| **虚引用** Phantom | 任意时间，仅做通知 | `Cleaner`、堆外内存释放 |

```java
SoftReference<byte[]> soft = new SoftReference<>(new byte[10 << 20]);
WeakReference<Object> weak = new WeakReference<>(new Object());
ReferenceQueue<Object> q = new ReferenceQueue<>();
PhantomReference<Object> phantom = new PhantomReference<>(new Object(), q);
```

> **面试陷阱**：`ThreadLocal` 内存泄漏的根因 = key 是弱引用、value 是强引用。线程池长期不释放线程时，value 会一直被 Entry 持有。**用完 `remove()` 是铁律**。

---

## 4. 三大基础回收算法

| 算法 | 思路 | 优点 | 缺点 | 用在哪 |
| --- | --- | --- | --- | --- |
| **标记-清除** | 标记后直接清除 | 简单 | 内存碎片 | CMS |
| **复制** | 存活对象复制到新区域，旧区域整片清空 | 无碎片、效率高 | 浪费一半空间 | 新生代 |
| **标记-整理** | 标记后让存活对象向一端移动 | 无碎片、不浪费空间 | 需要移动对象，开销大 | 老年代 |

---

## 5. 分代收集（HotSpot 默认思路）

**经验事实**：大多数对象朝生夕死，少数对象长期存活。

```
┌─────────────────── 堆 ───────────────────┐
│  新生代 Young Gen          │  老年代 Old Gen │
│  Eden : S0 : S1 = 8:1:1   │                │
└────────────────────────────────────────────┘
                            │
                       元空间 Metaspace（JDK8+，方法区实现，使用本地内存）
```

- **新生代**：复制算法，每次 Minor GC 在 Eden + 一个 Survivor 之间复制存活对象。
- **老年代**：标记-整理 / 标记-清除。
- **晋升规则**：
  1. 在 Survivor 中熬过 `MaxTenuringThreshold`（默认 15）次 GC 后晋升老年代；
  2. 大对象（超过 `PretenureSizeThreshold`）直接进老年代；
  3. **动态年龄判断**：Survivor 中相同年龄对象总和超过一半空间时，>= 该年龄的全部晋升。

---

## 6. 主流收集器对照表

| 收集器 | 区域 | 算法 | 模式 | 关注点 |
| --- | --- | --- | --- | --- |
| Serial | 新生代 | 复制 | 单线程 STW | 简单、客户端 |
| ParNew | 新生代 | 复制 | 多线程 STW | CMS 搭档 |
| Parallel Scavenge | 新生代 | 复制 | 多线程 STW | **吞吐量** |
| Serial Old | 老年代 | 标记-整理 | 单线程 STW | 备份 |
| Parallel Old | 老年代 | 标记-整理 | 多线程 STW | 吞吐量 |
| **CMS** | 老年代 | 标记-清除 | 并发 | **低停顿**（已废弃） |
| **G1** | 全堆（Region） | 标记-整理+复制 | 并发+分代 | **可预测停顿**（JDK9+ 默认） |
| **ZGC** | 全堆 | 着色指针+读屏障 | 并发 | **<10ms 停顿，TB 级堆** |
| **Shenandoah** | 全堆 | 并发标记-整理 | 并发 | 低停顿，OpenJDK |

### G1 关键特征（必背）
- 把堆划分为 **2048 个左右的 Region**，每个 1~32MB；
- 仍保留新生代/老年代逻辑，但物理上不连续；
- **Humongous 区**：占 Region 一半以上的大对象单独存放；
- 通过 `-XX:MaxGCPauseMillis` 设置目标停顿时间；
- **三色标记 + SATB**（Snapshot-At-The-Beginning）解决并发标记中漏标问题。

### ZGC 关键特征
- **染色指针**：在 64 位指针的高位保存 GC 信息（4 个标志位）；
- **读屏障**：每次读对象引用都做检查，触发并发处理；
- 几乎全部并发，**STW 仅在 Roots 扫描的几毫秒**；
- JDK 15 正式可用，JDK 21 分代 ZGC（Generational ZGC）。

---

## 7. STW 与并发标记

**STW（Stop The World）**：所有应用线程暂停，只剩 GC 线程跑。

并发标记里的"漏标"问题：
- 用户线程改了对象引用 → 已经被标记为黑色的对象指向了一个白色对象 → 白色对象被错误回收。

**两种解决方案**：

| 方案 | 思路 | 用在哪 |
| --- | --- | --- |
| **Incremental Update**（增量更新） | 黑色对象新增引用时，把黑色变回灰色，重新扫描 | CMS |
| **SATB**（开始时快照） | 灰色对象的引用被删除前，把旧引用记录下来，最后再扫描 | G1 |

---

## 8. Minor GC / Major GC / Full GC

| 名词 | 触发条件 | 范围 |
| --- | --- | --- |
| Minor GC（Young GC） | Eden 区放不下 | 仅新生代 |
| Major GC | 老年代空间不足 | 仅老年代（CMS 概念） |
| **Full GC** | 老年代不足、Metaspace 不足、`System.gc()`、CMS 失败、晋升失败 | **全堆 + 方法区** |

> 排查思路：先用 `jstat -gc` 看频率，**Full GC 频繁 = 必须立刻处理**。

---

## 9. 实战：常见 GC 问题与排查

### 9.1 Full GC 频繁
**可能原因**：
1. 大对象/集合直接进老年代；
2. 老年代空间太小（`-Xmx` 配置不合理）；
3. 内存泄漏（缓存未清、ThreadLocal 未 remove、监听器未注销）；
4. `System.gc()` 被显式调用 → 加 `-XX:+DisableExplicitGC`。

**排查命令**：
```bash
jstat -gc <pid> 1000          # 看 GC 频率与耗时
jmap -histo:live <pid>        # 看活对象 Top N
jmap -dump:live,format=b,file=heap.hprof <pid>   # dump 后用 MAT 分析
```

### 9.2 Young GC 耗时长
- Survivor 太小，对象提前晋升；
- 跨代引用太多（老年代引用新生代，扫描成本高）；
- G1 中 RememberedSet 维护成本高。

### 9.3 GC 日志关键参数

```bash
# JDK 8
-Xloggc:/path/gc.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps \
-XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=20M

# JDK 9+ 统一使用 -Xlog
-Xlog:gc*=info:file=/path/gc.log:time,uptime:filecount=5,filesize=20M
```

---

## 10. 高频面试题速记

1. **可达性分析中 GC Roots 包含哪些？** → 见第 2.2 节表。
2. **CMS 和 G1 的区别？** → CMS 标记-清除有碎片、不分代；G1 分 Region、可预测停顿、用 SATB。
3. **G1 如何避免漏标？** → SATB（开始时快照）。
4. **什么时候发生 Full GC？** → 老年代/Metaspace 不足、`System.gc()`、CMS 并发失败、晋升失败。
5. **ZGC 为什么这么快？** → 染色指针 + 读屏障，几乎全并发。
6. **新生代为什么用复制算法？** → 朝生夕死、存活对象少、复制成本低。
7. **元空间为什么替换永久代？** → 永久代大小固定易 OOM、与堆耦合。元空间用本地内存，仅受机器内存限制（仍可设 `MaxMetaspaceSize`）。
8. **三色标记**：白（未访问）、灰（自己被访问、字段未扫完）、黑（自己和字段都被扫完）。

---

## 11. 推荐阅读

- 周志明《深入理解 Java 虚拟机（第三版）》第 3~5 章
- Oracle [HotSpot Garbage Collection Tuning Guide](https://docs.oracle.com/en/java/javase/21/gctuning/)
- R Charles《The Garbage Collection Handbook》

> 下一步：[类加载机制](/article/jvm/classloader) | [Java 内存模型](/article/jvm/jmm)
