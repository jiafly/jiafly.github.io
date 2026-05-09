# Linux 与性能工程

> 不是"Linux 命令大全"。这一节要讲的是：写 Java / Go 服务在 Linux 上跑出问题时，**怎么从内核往上看到根因**。

---

## 为什么单独开这块

很多 Java 工程师对 Linux 的认知止于 `top / netstat / df` 几个命令。但你定位生产问题时遇到的是：
- `top` 看 CPU 高，但 thread dump 看不出热点 —— 怎么办？
- `iostat` 显示磁盘 await 100ms，但应用慢的不只是磁盘相关 —— 中间发生了什么？
- 容器里 OOM 了，但宿主还有内存 —— cgroup 限制踩在哪？
- 网络包"明明发出去了但对方没收到" —— 怎么追？

这些问题需要的是：**内核基础 + 性能分析方法 + 现代工具链**。这一节按 [桃花岛深度模板](/STYLE) 切，每篇都从一个真实的"诊断故事"反向讲。

---

## 已完成的深文

- ✅ **[火焰图实战 —— 从 perf 到 async-profiler 到连续剖析](/article/linux-perf/flame-graph)** · 1215 行
- ✅ **[Linux 60 秒诊断 —— Brendan Gregg 标准动作 + 实战剧本](/article/linux-perf/60s-diagnosis)** · 1506 行
- ✅ **[Linux 内存模型与 OOM —— PageCache 到 OOM Killer 全景](/article/linux-perf/memory-oom)** · 1469 行
- ✅ **[Linux IO 模型深入 —— select / epoll / io_uring](/article/linux-perf/io-models)** · 1413 行
- ✅ **[容器与 cgroup —— Java 应用在 K8s 里的"误读"](/article/linux-perf/container-cgroup)** · 1484 行

---

## 规划中的深文

### 1. Linux 60 秒诊断方法论 ✅ 已完成 → [阅读深文](/article/linux-perf/60s-diagnosis)

**核心问题**：服务器告警，你登上去前 60 秒该敲什么命令？敲完怎么判断是 CPU / IO / 内存 / 网络问题？

**写作骨架（5 层）**：
- **原理推导**：USE 模型（Utilization / Saturation / Errors）的诊断顺序；性能瓶颈的"四个层级"
- **工具切片**：`uptime / dmesg / vmstat / mpstat / pidstat / iostat / sar / free / top` 的关键字段
- **实测案例**：5 个真实场景的 60 秒诊断（CPU / 上下文切换 / 内存压力 / IO / 网络）
- **生产案例**：被 sysadmin 提示但本质是应用 Bug、宿主与容器视图差异导致的误判
- **延伸**：Brendan Gregg 的《Systems Performance》、async-profiler 的快速使用

### 2. 火焰图实战 ✅ 已完成 → [阅读深文](/article/linux-perf/flame-graph)

**核心问题**：火焰图看着花花绿绿，怎么真正读出"热点函数"和"等待瓶颈"？on-CPU / off-CPU / wallclock 三种火焰图区别？

**写作骨架**：
- **原理推导**：采样原理（perf events / 用户态 unwinding）；on-CPU vs off-CPU 火焰图能告诉你什么
- **工具切片**：perf script 与 stackcollapse 的关键步骤；async-profiler 的多种 event；Pyroscope 的连续剖析
- **实测案例**：5 个真实火焰图的解读（GC 热点 / 锁竞争 / 系统调用 / 网络阻塞 / 反射开销）
- **生产案例**：误读火焰图（看到 epoll_wait 占 80% 以为是问题）、JVM SafePoint 偏差、容器内符号丢失
- **延伸**：Differential Flame Graph（变化对比）、Continuous Profiling、Linux 6 的 perf 改进

### 3. Linux 内存模型与 OOM ✅ 已完成 → [阅读深文](/article/linux-perf/memory-oom)

**核心问题**：`free -h` 看到 available 还有 4G，但应用突然被 OOM Killer 杀。`cat /proc/meminfo` 看到 Slab 占了 8G —— 谁在用？容器 OOM 与宿主 OOM 的区别？

**写作骨架**：
- **原理推导**：Linux 内存层级（Buddy / Slab / Page Cache / Anonymous / Swap）；overcommit 与 OOM Score
- **源码切片**：`/proc/meminfo` 各项的内核来源；OOM Killer 的选分逻辑
- **实测案例**：模拟内存压力下各指标的变化；用 `slabtop / smem` 找谁占内存
- **生产案例**：容器 cgroup 限制低于实际需求被 OOM、Page Cache 被惯性回收导致 IO 飙升、JVM 堆外内存（DirectByteBuffer）泄漏
- **延伸**：cgroup v2 的内存控制器改进、JVM 在容器中的 `-XX:MaxRAMPercentage`

### 4. 上下文切换、CPU 调度、亲和性

**核心问题**：`vmstat` 显示 `cs` 列每秒上百万 —— 谁在切？怎么定位？为什么"绑核"在某些场景能提升性能？

**写作骨架**：
- **原理推导**：进程上下文切换的代价；CFS 调度器的核心逻辑；NUMA 拓扑与跨节点访问
- **源码切片**：`schedule()` 的关键路径；`sched_setaffinity` 的应用场景
- **工具切片**：`pidstat -w` / `perf sched` / `numactl --hardware`
- **生产案例**：高频 GC 导致大量上下文切换、Java 线程数过多导致 cache miss 飙升、容器跨 NUMA 节点导致延迟翻倍
- **延伸**：CPU 亲和性在网络密集应用中的应用（Netty）、Linux 5.x 的 EAS 调度

### 5. IO 模型 ✅ 已完成 → [阅读深文](/article/linux-perf/io-models)

**核心问题**：epoll 真比 select 快多少？io_uring 凭什么是下一代？Java NIO 在新 IO 模型下要不要重写？

**写作骨架**：
- **原理推导**：阻塞 / 非阻塞 / 多路复用 / 异步 IO 四种模型的真实差别；epoll 的 ET / LT 模式
- **内核源码**：`epoll_wait` 的关键路径；io_uring 的 SQ / CQ 环形缓冲
- **实测数据**：epoll vs io_uring 在万并发连接下的吞吐与 CPU 对比
- **生产案例**：epoll 惊群（已修复但旧内核仍有）、io_uring 的兼容性陷阱、Netty 的 native transport
- **延伸**：JDK 24 引入 io_uring、Glommio / Tokio 的 io_uring 应用、用户态网络栈（DPDK）

### 6. 容器与 cgroup ✅ 已完成 → [阅读深文](/article/linux-perf/container-cgroup)

**核心问题**：Java 应用在容器里跑，`Runtime.availableProcessors()` 返回宿主核数还是 cgroup 限额？JVM 默认堆大小怎么算？为什么容器里的 OOMKilled 看不到 OutOfMemoryError？

**写作骨架**：
- **原理推导**：cgroup v1 vs v2 的资源隔离；Linux namespace 的边界；OOMKilled 的发生时机
- **源码切片**：JDK 17+ 的 `OperatingSystem` 类对 cgroup 的探测；Kubernetes 的 QoS 类
- **实测数据**：不同 cgroup 配置下 JVM 的 GC 行为；Memory Request / Limit 设置的影响
- **生产案例**：JVM 启动时探测错误分配过大堆导致 OOMKilled、Throttled 状态下 P99 飙升、跨节点 cgroup 配置不一致
- **延伸**：Kata Containers、gVisor 的隔离对性能的影响、虚拟化嵌套的代价

---

## 关联现有文章
- [JVM 调优与排查](/article/jvm/tuning) —— JVM 在 Linux 上的诊断
- [可观测性](/article/observability/) —— 系统级指标的设计
- [线程池生产实战](/article/java/threadpool) —— 线程模型与 Linux 调度

## 想看哪一篇？

[告诉我](/article/about) 你最想先看哪个主题的深文，我会优先排期。
