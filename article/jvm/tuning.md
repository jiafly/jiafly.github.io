# JVM 调优与异常排查

> 调优 ≠ 拍脑袋调参数。**先观测、再定位、最后调参**。本文给一套可直接套用的方法论 + 命令模板。
>
> 字节码 / C1+C2 分级编译 / 内联与逃逸分析 / Counted Loop / CodeCache / 9 个 JIT 生产事故 → **[JIT 与字节码深入（深文）](/article/jvm/jit)**
>
> GC 算法深入（G1/ZGC/Shenandoah）→ **[JVM GC 深入（深文）](/article/jvm/gc-deep)**

---

## 1. 调优三大目标

| 目标 | 关注指标 | 常见取舍 |
| --- | --- | --- |
| **吞吐量** | 单位时间完成请求数 | 接受较长 GC 停顿 |
| **延迟** | P99 / P999 响应时间 | 牺牲部分吞吐，G1/ZGC |
| **内存占用** | 堆/RSS 大小 | 牺牲性能，限堆 |

> **铁律：永远不要同时追求三个目标。** 上线前先和业务对齐 SLO。

---

## 2. 常用参数速查表

### 2.1 堆与栈
```bash
-Xms4g -Xmx4g                     # 堆初始/最大，生产建议相等
-Xmn2g                            # 新生代大小
-XX:NewRatio=2                    # 老:新 = 2:1（与 -Xmn 互斥）
-XX:SurvivorRatio=8               # Eden:S0:S1 = 8:1:1
-Xss512k                          # 单线程栈大小
-XX:MetaspaceSize=256m            # 元空间初始
-XX:MaxMetaspaceSize=512m         # 元空间上限（生产必加，避免无界）
```

### 2.2 GC 选择
```bash
# G1（JDK9+ 默认，多数场景的最佳起点）
-XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:G1HeapRegionSize=16m

# ZGC（JDK15+，超大堆/极低延迟）
-XX:+UseZGC -XX:+ZGenerational

# Parallel（吞吐优先，离线/批处理）
-XX:+UseParallelGC -XX:ParallelGCThreads=8
```

### 2.3 异常自动 dump（生产必备）
```bash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/dumps
-XX:OnOutOfMemoryError="kill -9 %p"   # OOM 后立刻退出，让 K8S 重启
-XX:+ExitOnOutOfMemoryError
```

### 2.4 GC 日志
```bash
# JDK 8
-Xloggc:/path/gc-%t.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps \
-XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=50M

# JDK 9+
-Xlog:gc*=info:file=/path/gc.log:time,uptime,level,tags:filecount=10,filesize=50M
```

---

## 3. 排查工具箱

### 3.1 命令行三件套

| 工具 | 用途 |
| --- | --- |
| `jps -lvm` | 列出 Java 进程 + 启动参数 |
| `jstat -gc <pid> 1000 10` | GC 频率与各代占用，每秒一次共 10 次 |
| `jstack <pid>` | 线程栈快照，排查死锁/线程飙高 |
| `jmap -histo:live <pid> \| head -30` | Top 30 活对象 |
| `jmap -dump:live,format=b,file=heap.hprof <pid>` | 导出堆 dump |
| `jcmd <pid> VM.flags` | 查看运行时所有 VM 参数 |
| `jcmd <pid> GC.heap_info` | 查看堆使用 |
| `jinfo -flag <name> <pid>` | 查/改单个参数（部分参数运行时可改） |

### 3.2 进阶工具

| 工具 | 场景 |
| --- | --- |
| **Arthas**（推荐） | 在线诊断、热修复、`watch` / `trace` / `monitor` |
| **MAT**（Eclipse） | 离线分析 .hprof，找 dominator/leak suspect |
| **JProfiler / YourKit** | 商业 profiler，CPU/内存/锁全方位 |
| **async-profiler** | 低开销火焰图，`./profiler.sh -d 30 -f flame.html <pid>` |
| **JFR + JMC** | JDK 自带的飞行记录器 |
| **GCEasy / GCViewer** | GC 日志在线/离线分析 |

### 3.3 Arthas 高频命令
```bash
dashboard                          # 实时大盘
thread -n 5                        # CPU Top5 线程
thread -b                          # 死锁检测
trace com.foo.Service hello        # 追踪方法耗时
watch com.foo.Service hello "{params, returnObj, throwExp}"
jad com.foo.Service                # 反编译运行时类
sc -d com.foo.*                    # 看类加载器
heapdump /tmp/heap.hprof
```

---

## 4. 实战：四类典型故障

### 4.1 CPU 飙高

```bash
top -Hp <pid>                      # 找 CPU 高的线程
printf '%x\n' <tid>                # 转 16 进制
jstack <pid> | grep -A 30 <hex>    # 找对应栈
```

**常见原因**：
- 死循环（`while(true)` 没 break）；
- 频繁 GC（看 jstat）；
- 正则回溯爆炸；
- 大量序列化/反序列化。

### 4.2 内存溢出 OOM

| 报错 | 原因 | 排查 |
| --- | --- | --- |
| `Java heap space` | 堆满 | dump → MAT 看大对象/泄漏 |
| `Metaspace` | 类加载过多 | `jcmd VM.metaspace`、查动态代理/字节码生成 |
| `unable to create new native thread` | 线程数超出系统/进程限制 | `ulimit -u`、检查线程池配置 |
| `Direct buffer memory` | 堆外内存满 | `-XX:MaxDirectMemorySize`、Netty `PooledAllocator` |
| `GC overhead limit exceeded` | 98% 时间 GC + 回收<2% | 多半是内存泄漏 |

**排查模板**：
```bash
jmap -dump:live,format=b,file=/tmp/heap.hprof <pid>
# 用 MAT 打开 → Leak Suspects → Dominator Tree
```

### 4.3 GC 异常

**Full GC 频繁的判断**：
```bash
jstat -gc <pid> 1000
# 看 FGC 列每秒/每分增长。10秒涨一次 = 异常。
```

排查清单：
1. `-Xmx` 太小？
2. 大对象进老年代？看 `-XX:PretenureSizeThreshold`；
3. Survivor 太小，对象提前晋升？
4. 缓存未限大小（HashMap/Caffeine 没设 `maximumSize`）；
5. Metaspace 不足触发？
6. 显式 `System.gc()` 调用 → `-XX:+DisableExplicitGC`。

### 4.4 线程问题

```bash
jstack <pid> > stack.txt
# 关注：
#   - "BLOCKED" 状态的线程数量
#   - "deadlock" 关键字
#   - 同名线程数（业务线程池配置是否合理）
```

死锁示例输出：
```
Found one Java-level deadlock:
"Thread-1": waiting to lock Monitor@0x... (which is held by "Thread-0")
"Thread-0": waiting to lock Monitor@0x... (which is held by "Thread-1")
```

---

## 5. 容器化场景的坑

### 5.1 不要让 JVM 误判内存
JDK8u191+ 已支持 cgroup 感知，但仍建议**显式指定**：
```bash
-XX:+UseContainerSupport                   # 默认开启
-XX:MaxRAMPercentage=75.0                  # 用容器内存的 75%
-XX:InitialRAMPercentage=75.0
-XX:MaxMetaspaceSize=512m
```

### 5.2 K8S 配置建议
- **request 与 limit 不要差太多**，避免 OOMKilled；
- liveness probe 不要过于敏感，GC 时刚好命中会重启风暴；
- 配 `preStop hook`：`sleep 15` 让流量摘除完再退出；
- 容器内默认时区是 UTC，加 `-Duser.timezone=Asia/Shanghai`。

---

## 6. 调优方法论（务必背会）

```
1. 定 SLO   → 吞吐 / P99 / 内存哪个优先？
2. 加监控   → GC 日志、APM、CPU/Mem/IO、业务指标
3. 压测     → 拿到稳态数据
4. 分析     → 找 GC、CPU、锁、IO 瓶颈
5. 假设     → 提出"是因为 X 导致 Y"
6. 验证     → 改一个参数，复测，看是否符合假设
7. 复盘     → 写下变更与效果（非常重要）
```

> 一次只改一个变量，避免把多个改动揉一起。

---

## 7. 一份"上线即用"的 JVM 参数模板（4C8G、Spring Boot Web）

```bash
JAVA_OPTS="
-server
-Xms4g -Xmx4g
-Xss512k
-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m
-XX:+UseG1GC -XX:MaxGCPauseMillis=200
-XX:InitiatingHeapOccupancyPercent=45
-XX:+ParallelRefProcEnabled
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dumps
-XX:+ExitOnOutOfMemoryError
-Xlog:gc*=info:file=/data/logs/gc.log:time,uptime:filecount=10,filesize=50M
-Duser.timezone=Asia/Shanghai
-Dfile.encoding=UTF-8
"
```

---

## 8. 高频面试题速记

1. **怎么排查 CPU 飙高？** top -Hp → printf %x → jstack。
2. **OOM 有哪些类型？怎么定位？** Heap、Metaspace、DirectBuffer、Stack、Native；先看哪个区，再 dump 分析。
3. **Full GC 频繁怎么办？** GC 日志 → jstat → jmap dump → MAT。
4. **G1 调优关键参数？** `MaxGCPauseMillis`、`G1HeapRegionSize`、`InitiatingHeapOccupancyPercent`。
5. **生产 JVM 参数模板？** 见第 7 节。
6. **JDK 8 vs JDK 17 调优区别？** 默认 GC 从 Parallel 变 G1；统一日志参数；ZGC 可用；JFR 永久免费。
7. **容器化 JVM 注意事项？** UseContainerSupport、MaxRAMPercentage、固定堆大小、配 dump 路径。

---

> 上一篇：[Java 内存模型](/article/jvm/jmm) ｜ 返回：[首页](/)
