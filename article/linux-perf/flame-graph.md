# 火焰图实战 —— 从 perf 到 async-profiler 到连续剖析的全栈定位

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，看 [Linux 与性能工程综述](/article/linux-perf/) 那一节就够了。
> 这里要回答的，是**线上 CPU 飙到 100%、P99 抖到天上、半夜被告警叫起来**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. on-CPU、off-CPU、wallclock 三种火焰图分别能告诉你什么？哪种能定位"线程在等什么"？
2. `perf record -F 99 -g` 和 `async-profiler -e cpu` 的采样原理差在哪里？为什么对 JVM 应用首选后者？
3. SafePoint Bias 是什么？为什么 perf 的 JVM 火焰图常常把热点错指到无辜的 `JIT_xxx` 桩函数上？
4. 你在容器里抓出来的火焰图栈顶全是 `[unknown]` —— 是缺什么？需要怎么修？
5. 火焰图的横轴和纵轴分别表示什么？横轴宽度大 = 慢吗？
6. 差分火焰图（Differential Flame Graph）怎么用？上线前后想对比哪一段变红了，命令是什么？
7. 连续剖析（Continuous Profiling，Pyroscope/Parca）相比一次性 `perf record` 的工程价值是什么？多花的存储成本怎么控制？
8. 用户态 unwinding 的 DWARF / frame-pointer / LBR 三种方式各有什么坑？为什么 JVM 必须开 `-XX:+PreserveFramePointer`？

如果有半数答不上，往下看。

---

## 1. 底层模型：采样、栈展开、聚合

绝大多数人对火焰图的认知止于"找最宽的那一格"。但这层认知答不上面的 8 个问题。要看懂它们，必须下沉到三件事：**采样原理、栈展开（stack unwinding）、可视化聚合**。

### 1.1 性能分析方法论：USE / RED / 60 秒诊断

进剖析（profiling）之前，先有方向感。三套互补的方法论：

- **USE（Brendan Gregg）**：对**资源**（CPU / mem / disk / net），看 Utilization（利用率）/ Saturation（饱和度，等待队列）/ Errors（错误）。**适用于"系统是不是健康"**。
- **RED（Tom Wilkie）**：对**服务**，看 Rate（请求速率）/ Errors / Duration（耗时分布）。**适用于"业务是不是健康"**。
- **60 秒诊断**：登上一台陌生机器，依次跑 `uptime / dmesg | tail / vmstat 1 / mpstat -P ALL 1 / pidstat 1 / iostat -xz 1 / free -m / sar -n DEV 1 / sar -n TCP,ETCP 1 / top`，10 个命令出大致定性。

火焰图是 **USE 之后的下钻工具**：
- 通过 USE 知道是 "CPU saturation 高" → 抓 **on-CPU 火焰图** 看 CPU 上跑什么
- 通过 USE 知道 "CPU 不高但 P99 抖动" → 抓 **off-CPU 火焰图** 看线程在等什么
- 通过 RED 知道 "P99 比 P50 高一个数量级" → 抓 **wallclock 火焰图** 找到一类慢请求的真实瓶颈

> 没有方法论指引的火焰图就是花花绿绿的糖纸。先用 USE/RED 定性，再用火焰图定位。

### 1.2 采样剖析 vs 插桩剖析

剖析（profiling）的两条根本路线：

| 类型 | 原理 | 代价 | 失真 |
| --- | --- | --- | --- |
| **采样（sampling）** | 定时打断进程，记录当前栈 | 极低（< 5% CPU 通常 < 1%） | 短而稀疏的函数会被错过 |
| **插桩（instrumentation）** | 在每个函数 enter/exit 插探针 | 高（10x ~ 100x 慢） | 几乎无失真，但探针本身改变了行为（observer effect） |

火焰图属于**采样**派。它的核心假设是：**采样次数 ∝ 实际耗时**。如果某函数在 1000 次采样里出现了 300 次，它大概占了 30% 的 CPU 时间。

采样频率（frequency）的选择：
- `perf record -F 99`：每秒 99 次。**99 不是 100**，是为了避开周期性事件的"莫尔条纹"——比如某个心跳是 100Hz，跟 100Hz 采样会同步上抢拍同一个栈。
- `async-profiler -i 10ms`：默认 10 ms 一次。
- 频率越高越准，但也越贵。**生产环境 99Hz 通常是甜点**。

### 1.3 perf 的 PMU + perf events 模型

`perf` 本质是 Linux 内核暴露的 `perf_event_open(2)` 系统调用的用户态包装。它能监听三类事件：

- **硬件事件（PMU, Performance Monitoring Unit）**：CPU 内部计数器，如 `cycles`、`instructions`、`cache-misses`、`branch-misses`、`LLC-load-misses`。**没有软件代价**，由 CPU 计数到溢出后发中断。
- **软件事件**：`cpu-clock`、`task-clock`、`context-switches`、`page-faults`。内核维护。
- **追踪事件（tracepoints / kprobes / uprobes）**：内核或用户态指定函数被调用时触发。

火焰图的"on-CPU"采样默认用 `cycles`（硬件）或 `cpu-clock`（软件）。**`cycles` 更准但需要 PMU 支持，云虚拟机/容器里经常被禁用**，那就退到 `cpu-clock`。

```bash
# 检查 PMU 是否可用
perf stat -e cycles,instructions sleep 1

# 如果输出 <not supported>，就只能用 cpu-clock
perf record -F 99 -e cpu-clock -g sleep 30
```

### 1.4 火焰图的可视化原理

Brendan Gregg 在 2011 年发明的火焰图（flame graph）只是一种 **SVG 渲染规则**，理解它之后你会发现"看图能力"瞬间提升一个量级。

- **横轴 = 样本数**（不是时间！）。两个函数横向并列，左右顺序按字母排序，**没有时间含义**。横轴宽度 = 这个函数（连同它在某条调用栈上的所有上层调用）出现的采样次数占比。
- **纵轴 = 调用栈深度**。最下面是根（`main` / `JavaThread::run` / `start_thread`），往上是被调用者。某一格的正上方是它在采样里被观察到调用的函数。
- **颜色无信息含义**（暖色调只是为了"火焰"视觉效果）。差分火焰图除外（红=变慢、蓝=变快）。
- **每一格的宽度** = 这个 frame 在 stack-collapsed 输出里的"折叠次数"。

数据流：

```
采样的栈            stack-collapse              flamegraph.pl
[A,B,C]             A;B;C 1
[A,B,C]      ───►   A;B;D 1            ───►    SVG
[A,B,D]             A;B;C 1
[A,B,C]
```

`stackcollapse-perf.pl` 干的事：把每个采样的栈从上到下用分号拼成一行，相同的行计数。`flamegraph.pl` 读这个 collapsed 格式，按"前缀相同的栈合并到同一根上"画成图。

**反直觉的点**：火焰图不是按时间画的，**横轴的左→右没有"先发生→后发生"的语义**。它只是一种"调用栈的层次直方图"。

### 1.5 on-CPU vs off-CPU vs wallclock

三种最常用的火焰图，物理含义完全不同：

| 类型 | 采样什么 | 触发时机 | 回答的问题 |
| --- | --- | --- | --- |
| **on-CPU** | CPU 上正在跑的栈 | 定时（基于 `cycles` 或 `cpu-clock`） | 谁在烧 CPU？ |
| **off-CPU** | 线程被调度出 CPU 时的栈 | `sched:sched_switch` tracepoint | 线程在等什么？（锁/IO/sleep） |
| **wallclock** | 不区分跑没跑，所有线程的栈 | 定时（基于 wall time） | 一个请求实际花在哪？（含等待） |

什么时候选哪个？

- **CPU 利用率高、想找热点函数** → on-CPU
- **CPU 不高但 P99 高、有"卡"的现象** → off-CPU（看锁、看 IO 等待）
- **想看一个 RPC 请求里时间花在哪** → wallclock + 单线程过滤

async-profiler 同时支持三种：
```bash
./asprof -e cpu  -d 30 -f cpu.html  <pid>     # on-CPU
./asprof -e wall -d 30 -f wall.html <pid>     # wallclock
./asprof -e lock -d 30 -f lock.html <pid>     # 锁竞争 (off-CPU 的子集)
./asprof -e alloc -d 30 -f alloc.html <pid>   # 内存分配
```

> 上 CPU 的栈告诉你"在干活"；下 CPU 的栈告诉你"在等活"。两张图通常要一起看。

### 1.6 SafePoint Bias 与为什么 JVM 要用 async-profiler

这是 JVM 性能分析的最深的一个坑，几乎所有"perf 火焰图骗了我"的故事都源于它。

**SafePoint（安全点）**：JVM 为了执行"全局停顿"操作（GC、deopt、偏向锁撤销等）而约定的"线程检查点"。每个 Java 线程跑到 safepoint 时都会去 poll 一下"要不要停"。Hotspot 的 safepoint 通常埋在：
- 方法返回处
- 长循环的回边（back-edge）
- 解释执行的 bytecode 边界

**普通 JVM profiler（如 hprof / 早期 VisualVM）的工作方式**：调用 `getAllStackTraces()`，这个 API 内部要在 safepoint 上拿栈。结果是：**只有跑到 safepoint 的线程的栈被采到，没到 safepoint 的根本没记**。

**SafePoint Bias 的后果**：你以为采的是"均匀采样"，实际是"safepoint 上的偏倚采样"。某些 JIT 优化掉的循环、内联到没有 safepoint 的热路径，**永远采不到**；而某些频繁回边的方法被过度高估。

**`perf record` 对 JVM 的另一个问题**：JIT 编译出来的代码没有 ELF 符号，perf 记到 PC（程序计数器）后查不到函数名，全显示 `[unknown]` 或 `0x7fxxxxxxxxxx`。需要：

```bash
# 1. 让 JVM 输出符号表给 perf
java -XX:+PreserveFramePointer \
     -XX:+UnlockDiagnosticVMOptions \
     -XX:+DebugNonSafepoints \
     -agentpath:/path/to/libperfmap.so \
     ...

# 2. perf 会去读 /tmp/perf-<pid>.map
perf record -F 99 -g -p <pid> -- sleep 30
```

但即使配了 `perf-map-agent`，**inline 优化掉的方法栈帧仍然丢失**，这是 perf 看 JVM 的"二阶失真"。

**async-profiler 的解决方案**：用 `AsyncGetCallTrace()` —— 这是 Hotspot 的非公开但被广泛使用的 API，它**不要求 safepoint**，直接在信号处理器里走 JVM 的 stack walker 解栈。这就是为什么 async-profiler 对 JVM 应用更准。

代价：`AsyncGetCallTrace` 偶尔会丢失某些 inlined 的 frame，但相比 SafePoint Bias 的偏倚是数量级的改善。

> JVM 应用首选 async-profiler。perf 留给原生代码（C/C++/Rust/Go）以及"我想看到 syscall 内核栈"的场景。

### 1.7 用户态与内核态的 unwinding

栈展开（stack unwinding）是采样剖析最难的一环。三种主流方式：

| 方式 | 原理 | 优点 | 缺点 |
| --- | --- | --- | --- |
| **frame pointer (FP)** | 沿 `%rbp` 链回溯 | 极快，开销几乎为零 | 编译器可能优化掉 FP（`-fomit-frame-pointer`） |
| **DWARF (.eh_frame)** | 解析 DWARF CFI（Call Frame Information） | 不依赖 FP | 慢、栈大、内核里用 `--call-graph dwarf` 时要拷贝 16K 用户栈 |
| **LBR (Last Branch Record)** | CPU 硬件记录最近 16~32 次分支 | 极快、准 | 栈深有限（32 帧），需要新硬件 |

实务建议：

- **新进程从源码编译** → 加 `-fno-omit-frame-pointer`，用 `--call-graph fp`
- **不能改编译参数（线上二进制）** → 用 `--call-graph dwarf,16384`
- **只关心栈底几层** → 用 `--call-graph lbr`

JVM 的 `-XX:+PreserveFramePointer`（JDK 8u60+）就是要求 JIT 也维护 frame pointer。**不开这个，JVM 火焰图永远是断的**。

容器里栈断裂的另一个原因：**调试符号缺失**。Alpine / distroless 镜像只装了 release 二进制，没有 `.eh_frame` 也没有 `-debuginfo` 包。看到栈顶全是 `[unknown]` 时，先确认 `glibc-debuginfo` / `libstdc++-debuginfo` 是否装了。

---

## 2. 工具与命令切片

每段命令完整可复制；版本不同时给替代。

### 2.1 perf 全套：record + script + stackcollapse + flamegraph

最经典的火焰图生成全流程，针对原生程序：

```bash
# 1. 安装（按发行版）
sudo apt install linux-tools-$(uname -r)        # Debian/Ubuntu
sudo yum install perf                            # CentOS/RHEL

# 2. 拉 FlameGraph 工具
git clone https://github.com/brendangregg/FlameGraph
export PATH=$PATH:$(pwd)/FlameGraph

# 3. 采样 30 秒（PID 模式）
sudo perf record -F 99 -p <pid> -g --call-graph fp -- sleep 30

# 或采样整个系统
sudo perf record -F 99 -a -g --call-graph dwarf,16384 -- sleep 30

# 4. 解析栈
sudo perf script > out.perf

# 5. 折叠
./stackcollapse-perf.pl out.perf > out.folded

# 6. 渲染 SVG
./flamegraph.pl out.folded > flame.svg

# 7. 浏览器打开
xdg-open flame.svg
```

每一步的常见踩坑：

- **`perf record -p <pid>` 必须 sudo**：`perf_event_paranoid` 默认 = 2，普通用户拿不到 kernel 栈。临时降权：`echo 1 > /proc/sys/kernel/perf_event_paranoid`。
- **生产容器里 perf 通常装不了**：宿主跑 perf，加 `--ns` 参数让 perf 知道目标进程在哪个 namespace。
- **`-F 99` 对 4 核机器约 400 次/秒采样**：一般够。极少数时候要 `-F 999`。
- **`sleep 30` 是 perf 的"作用时长"模板**：你也可以 `perf record ... ./your-binary` 跑完即停。

### 2.2 async-profiler 的多 event

```bash
# 1. 下载
wget https://github.com/async-profiler/async-profiler/releases/download/v3.0/async-profiler-3.0-linux-x64.tar.gz
tar xzf async-profiler-3.0-linux-x64.tar.gz
cd async-profiler-3.0-linux-x64

# 2. on-CPU 采样
./bin/asprof -d 30 -f cpu.html <pid>

# 3. wallclock（含等待）
./bin/asprof -e wall -d 30 -f wall.html -t <pid>
# -t 表示按线程拆分，否则所有线程的栈混在一起

# 4. 锁竞争
./bin/asprof -e lock -d 30 -f lock.html <pid>

# 5. 内存分配（按 TLAB 慢路径采样，开销极低）
./bin/asprof -e alloc -d 30 -f alloc.html <pid>

# 6. JFR 格式（可在 JMC 里二次分析）
./bin/asprof -e cpu -d 60 -o jfr -f cpu.jfr <pid>

# 7. 多 event 同时
./bin/asprof -e cpu,alloc -d 30 -f profile.jfr <pid>
```

关键参数：
- `-i N`：采样间隔（默认 10ms for CPU, 100us for alloc）
- `-t`：按线程分组
- `--alloc 1m`：alloc 模式只采样 ≥1MB 的分配
- `--lock 10ms`：lock 模式只采样持锁 ≥10ms 的事件

启动时直接挂载 agent（适合短生命周期进程）：
```bash
java -agentpath:./libasyncProfiler.so=start,event=cpu,file=cpu.html,duration=30 \
     -jar app.jar
```

### 2.3 jcmd JFR.start + JMC

JDK 自带的 JFR（Java Flight Recorder），无需额外 agent：

```bash
# 启动一个 60 秒的录制
jcmd <pid> JFR.start name=myrecording settings=profile duration=60s filename=record.jfr

# 主动 dump
jcmd <pid> JFR.dump name=myrecording filename=now.jfr

# 停止
jcmd <pid> JFR.stop name=myrecording

# 也可以启动时开
java -XX:StartFlightRecording=duration=60s,filename=app.jfr,settings=profile -jar app.jar
```

`settings=default` 与 `settings=profile` 的差别：
- `default`：约 1% 开销，事件粒度粗
- `profile`：约 2% 开销，含方法采样、TLAB 分配、对象统计

JFR 文件用 [JDK Mission Control](https://www.oracle.com/java/technologies/jdk-mission-control.html) 打开。它的"Method Profiling"页可以直接生成火焰图视图。

JFR 的优势是**和 JVM 深度集成**：能拿到 GC 事件、JIT 编译事件、类加载事件、线程 park 事件。劣势是看栈不如 async-profiler 准，且分析门槛高（JMC 学习曲线）。

### 2.4 arthas profiler 命令

[arthas](https://github.com/alibaba/arthas) 是阿里开源的 JVM 诊断工具，内置 async-profiler。**生产环境最适合的入口**，因为它支持 attach 后再退出，不影响进程。

```bash
# 启动 arthas（attach 到 pid）
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar <pid>

# 进入 arthas 交互式 shell 后
[arthas@1234]$ profiler start --duration 30 --event cpu
[arthas@1234]$ profiler stop --file /tmp/cpu.html

# wallclock
[arthas@1234]$ profiler start --event wall --duration 30
[arthas@1234]$ profiler stop --file /tmp/wall.html

# 锁竞争
[arthas@1234]$ profiler start --event lock --duration 30
[arthas@1234]$ profiler stop --file /tmp/lock.html
```

arthas 的 `profiler` 命令底层就是 async-profiler，但它的 attach / detach 机制对生产环境更友好 —— **绝大多数公司不会让你直接 ssh 上线机跑 root 的 perf**，但允许你用 arthas attach。

### 2.5 Pyroscope 连续剖析的 Agent 部署

[Pyroscope](https://github.com/grafana/pyroscope)（已并入 Grafana Labs）是事实上的开源连续剖析（continuous profiling）方案。它的核心思想：**把 profiling 从"一次性人工触发"变成"7x24 后台采样 + 时序存储"**，遇到事故时直接回看历史。

JVM 接入：

```bash
# 1. 下载 Java agent
wget https://github.com/grafana/pyroscope-java/releases/download/v0.13.0/pyroscope.jar

# 2. 启动
java -javaagent:pyroscope.jar \
     -Dpyroscope.application.name=my-app \
     -Dpyroscope.server.address=http://pyroscope:4040 \
     -Dpyroscope.profiling.interval=10ms \
     -Dpyroscope.format=jfr \
     -Dpyroscope.profiler.event=cpu \
     -Dpyroscope.profiler.alloc=512k \
     -Dpyroscope.profiler.lock=10ms \
     -jar app.jar
```

部署 Pyroscope Server（最简单的 docker-compose）：

```yaml
version: '3'
services:
  pyroscope:
    image: grafana/pyroscope:latest
    ports:
      - "4040:4040"
    volumes:
      - pyroscope-data:/var/lib/pyroscope
volumes:
  pyroscope-data:
```

接入后在 Pyroscope UI（或 Grafana 的 Pyroscope 数据源）里能看到：
- 任意时间段的 on-CPU / alloc / lock 火焰图
- **diff 视图**：选两个时间段，自动差分
- 按 label（pod / version / region）下钻

成本控制（这是连续剖析最受质疑的点）：
- 默认采样频率 100Hz，每条样本约 100~500 字节，单实例每天 ~5GB（压缩后 < 500MB）
- Pyroscope 用 Apache Parquet + ClickHouse-like 列存，10 节点集群跑 1000 个实例不在话下
- 真实公司案例：Uber、Netflix 都跑了千节点级的持续剖析

### 2.6 差分火焰图（differential flame graph）

上线一个新版本，发现 P99 涨了 30ms。怎么定位是哪个函数变慢了？**差分火焰图**。

```bash
# 1. 上线前抓一份
./asprof -d 60 -e cpu -o collapsed -f before.folded <pid>

# 2. 上线后抓一份
./asprof -d 60 -e cpu -o collapsed -f after.folded <pid>

# 3. 用 Brendan Gregg 的 difffolded 工具
./difffolded.pl before.folded after.folded > diff.folded
./flamegraph.pl --negate diff.folded > diff.svg
```

读图规则：
- **红色越深** = 这个函数在 after 里采样次数比 before 多（**变慢**）
- **蓝色越深** = 这个函数在 after 里少了（**变快或被优化掉**）
- 灰色 = 没变

差分火焰图是回归定位的"上帝视角"，**比看 P99 监控数字找根因高效一个量级**。

Pyroscope 的 UI 里直接有 diff 模式，不需要手动跑工具。

### 2.7 容器里的火焰图：符号、frame pointer、cgroup

容器里抓火焰图的常见坑：

**坑 1：容器里没 perf**

宿主装 perf，attach 容器进程：
```bash
# 在宿主上
docker inspect --format '{{.State.Pid}}' <container>
# 假设输出 12345
sudo perf record -F 99 -p 12345 -g -- sleep 30
```

注意 perf 拿到的栈是宿主视角的 PID/路径，但**符号表在容器里**。需要在容器里跑 `perf script` 或者把 `/proc/12345/root` 当作符号路径传给 perf。

**坑 2：JVM 启动时没开 PreserveFramePointer**

容器化的 Spring Boot Dockerfile 必加：
```dockerfile
ENV JAVA_OPTS="-XX:+PreserveFramePointer \
               -XX:+UnlockDiagnosticVMOptions \
               -XX:+DebugNonSafepoints"
ENTRYPOINT java $JAVA_OPTS -jar app.jar
```

`-XX:+DebugNonSafepoints` 让 JIT 在非 safepoint 处也保留行号信息，async-profiler 拿栈时更准。

**坑 3：alpine / distroless 缺 debuginfo**

alpine 用 musl libc 不带 DWARF。换成 `eclipse-temurin:21-jre-jammy`（基于 Ubuntu）或显式装 debug 包：
```dockerfile
RUN apt-get install -y libc6-dbg libstdc++6-dbg
```

**坑 4：cgroup v2 下 perf 默认拿不到事件**

修 systemd cgroup 配置：
```bash
# /etc/systemd/system.conf
DefaultCPUAccounting=yes
DefaultMemoryAccounting=yes
```

或者给容器加 `--cap-add=SYS_ADMIN --cap-add=PERFMON`。

> 容器里抓火焰图的"基线作业"：PreserveFramePointer + DebugNonSafepoints + 非 alpine 镜像 + arthas 入口。这套配齐，90% 的栈断裂问题不会出现。

---

## 3. 实测案例：用真实场景练读图能力

读图能力是肌肉记忆。这一节用 5 个**典型场景**演示火焰图的"读法"，等价于解题套路。

### 3.1 GC 长停顿的火焰图

**症状**：监控显示 P99 偶发飙到 800ms，业务日志里没有错误，jstat 看 Old Gen GC 频繁。抓一份 on-CPU 火焰图。

**典型栈片段**：

```
SystemDictionary::resolve_or_null
  └── ConstantPool::klass_at_impl
       └── SymbolTable::lookup_only
            └── StringTable::intern             ← ★ 占栈 25%
                 └── 业务代码 String.intern() 调用
```

**读法**：栈顶看到 `StringTable::intern` 这种"哈希表查找"占大头时，第一反应是**字符串去重**。但更可疑的是另一个常见模式：

```
G1ParTask::work
  └── G1RootProcessor::process_strong_roots
       └── StringTable::weak_oops_do            ← ★ GC 线程在扫弱引用
```

GC 线程扫 StringTable 占用大量时间 = 应用代码大量 `String.intern()` = 表撑大 = GC 扫表慢。**这是个反馈循环**。

**修复**：搜业务代码里的 `String.intern()`，换成自定义 `ConcurrentHashMap` 缓存（你能控制 evict 策略，String.intern 不能）。或者调大 `-XX:StringTableSize=1000003`（用质数）。

### 3.2 锁竞争的火焰图

**症状**：CPU 利用率正常（30%），但 QPS 上不去，加机器也没用。抓 **wallclock** 或 **lock** 火焰图。

**典型栈片段**（async-profiler `-e lock`）：

```
java/util/concurrent/locks/ReentrantLock.lock
  └── 业务方法 com.foo.OrderService.placeOrder
       └── ParkBlocker AbstractQueuedSynchronizer$Node    ← ★ 占栈 80%
```

栈顶是 `ParkBlocker` + 占栈 80% = **极度锁竞争**。继续看：往下走是哪个业务类的哪个方法在持锁？

`-e lock` 模式还会标注**持锁对象的类名**，比如：
```
[Object: com.foo.cache.LocalCache$1@xxx]
```

**读法**：80% 的等待集中在一把锁 → 找出这个锁的持有者 → 通常是某个全局单例的 `synchronized` 方法。

**修复**：
- 锁粒度太大 → 拆分（按 key 哈希分多把锁）
- 锁内有 IO → 把 IO 移出锁
- 读多写少 → `ReadWriteLock` 或 `StampedLock`
- 真的需要原子 → `AtomicLong` / `LongAdder`

### 3.3 反射开销的火焰图

**症状**：QPS 比预期低 2 倍，CPU 跑满。on-CPU 火焰图看：

```
sun/reflect/GeneratedMethodAccessor*.invoke
  └── java/lang/reflect/Method.invoke
       └── com.fasterxml.jackson.databind.deser.BeanDeserializer.deserializeFromObject
            └── 业务 RestController.handle
```

栈中频繁出现 `Method.invoke` + `GeneratedMethodAccessor*` = **反射调用占主成本**。

Jackson 的反序列化大量走反射。**虽然它在第 16 次调用后会把反射换成动态生成的字节码**（`GeneratedMethodAccessor`），但仍有显著的间接调用开销。

**修复**：
- 升级到 Jackson 2.13+，启用 [BlackbirdModule](https://github.com/FasterXML/jackson-modules-base/tree/master/blackbird)，用 LambdaMetafactory 取代反射
- 或者整体换到代码生成式的序列化方案（如 [DSL-JSON](https://github.com/ngs-doo/dsl-json)）
- POJO 字段尽量用 `public`，省掉 setter 反射

### 3.4 网络阻塞的火焰图

**症状**：业务调用下游 RPC 偶发慢，下游又把锅甩回来说"我没收到请求"。抓 **wallclock** 火焰图：

```
java/net/SocketInputStream.socketRead0   ← native，占栈 60%
  └── SocketInputStream.read
       └── HttpClient.read
            └── 业务 OrderClient.fetchOrder
```

`socketRead0` 占 60% = 你的线程 60% 时间在等下游响应。

**怎么定位是哪个下游？** 看栈下面的业务方法：`OrderClient.fetchOrder`。结合 SkyWalking / OpenTelemetry 的 span，能进一步定位是哪个 endpoint。

**反向思考**：如果你看到 `socketRead0` 但栈里没有任何业务方法，全是 Netty 的 `EpollEventLoop` —— 那不是问题，是**事件循环在 epoll 上空等**。见 3.5。

**修复**：
- 下游慢 → 加超时 + 熔断（Resilience4j / Sentinel）
- 下游不稳 → 改异步调用（CompletableFuture + 独立池）
- 短链接 → 换连接池（HttpClient 5 / OkHttp）

### 3.5 误读案例：`epoll_wait` 占 80% 不是问题

**症状**：抓到一份火焰图，`epoll_wait` 占了 80%。**新手第一反应：epoll 有问题，要换 io_uring**。

```
sys_epoll_wait
  └── do_epoll_wait
       └── ep_poll
            └── schedule_hrtimeout_range
                 └── ★ 占栈 80%
```

**真相**：on-CPU 火焰图采的是"在 CPU 上的栈"。`epoll_wait` 看似在栈上，但**它本质是个让线程睡眠的 syscall**，"在 CPU 上"是个假象 —— 它可能只是恰好在采样的瞬间还没切走。

**正确的读法**：检查这个进程的 CPU 利用率。如果 CPU 利用率只有 5%，那这 80% 是 80% × 5% = **4% 的真实 CPU 时间**，毫无问题。这只是说明**该线程绝大多数时间空闲在等事件**。

**怎么避免误读**：
- on-CPU 火焰图永远要配 CPU 利用率一起看
- async-profiler 的 wallclock 模式才能告诉你"这个线程实际有多少时间在干什么"
- 看 `epoll_wait` / `pthread_cond_wait` / `futex_wait` / `read` (block) 这种 "等待型 syscall" 高占比时，第一反应应该是 "**线程闲，没问题**"，除非业务量很大却 CPU 又低

### 3.6 反向误读案例：`futex_wait` 占 80% 也可能是真问题

刚才说"等待型 syscall 占比高 = 线程闲"，但这条规则有个例外。

如果你抓的是 **off-CPU 火焰图**（不是 on-CPU），栈顶看到 `futex_wait` 占 80%，**这反而是问题信号**。

为什么？off-CPU 采样的物理含义是 "**线程被调度出去，正在等什么**"。栈顶是 `futex_wait` = 大部分时间在等锁。`futex_wait` 是 `pthread_mutex_lock` / `synchronized` / `ReentrantLock` 等所有锁原语的最终落点。

诊断流程：
1. 拿 off-CPU 火焰图栈顶看是哪个 `futex_wait`
2. 沿栈往下找业务方法名
3. 找到调用链上的具体锁对象（看变量名 / 单例类名）
4. 进一步用 `-e lock` 模式确认锁竞争次数

> on-CPU 看到 `epoll_wait` 高 = 闲；off-CPU 看到 `futex_wait` 高 = 锁竞争。**同一个 syscall 在不同火焰图里含义相反**。这是初学者最容易栽的坑。

### 3.7 GC roots 扫描占栈 —— 隐藏的"对象图过大"

**症状**：YGC 频率正常但单次时间长（200ms+）。on-CPU 火焰图：

```
G1ParTask::work
  └── G1RootProcessor::evacuate_roots
       └── G1RootProcessor::process_strong_roots
            └── ClassLoaderDataGraph::roots_cld_do      ← ★ 占 GC 时间 40%
```

栈顶是 ClassLoader 相关的 root scan = **加载的类太多**。常见于：
- 用了 Groovy / Nashorn / 动态代理大量生成类
- Hibernate 的 `@DynamicUpdate` 给每个实体生成代理类
- 使用 ASM / Javassist 动态字节码生成且没复用 ClassLoader

**修复**：
- 升级到 G1 / ZGC（Shenandoah / ZGC 对大对象图更友好）
- `jcmd <pid> GC.class_stats` 看哪些类异常多
- 限制动态类生成的复用粒度

> GC 火焰图的栈顶不止是 `do_oop`。看到 `ClassLoaderData` / `StringTable` / `SymbolTable` 占栈高时，对应的根因都不一样。

> 火焰图的"宽度"是采样数比例，不是绝对时间。**宽 ≠ 慢**。
> 永远先问"这个进程 CPU 多少？"再去解读图里的占比。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① CPU 飙到 100% 但业务无新增 —— "GC 线程把核烧穿了"

#### 现象
监控显示某 Pod 的 CPU 从 30% 突然飙到 100%，并卡在 100% 不下。同时 QPS 没有增长，DB 调用没增加，下游 RPC 调用量持平。业务方完全没察觉。

#### 根因
登录 Pod，`top -Hp <pid>` 看线程级 CPU：

```
PID    TID    %CPU  COMMAND
1234   1289   98.7  GC Thread#0
1234   1290   97.2  GC Thread#1
1234   1291   97.5  GC Thread#2
1234   1292   96.9  GC Thread#3
```

**4 个 GC Thread 全打满**。基本可以确认是 GC 在跑。继续抓 async-profiler：

```bash
./asprof -d 30 -e cpu -f gc.html <pid>
```

火焰图栈底全是 `JavaThread::thread_main_inner` → `GCTaskThread::run` 的分支，栈中频繁出现：

```
G1CollectedHeap::do_collection_pause_at_safepoint
  └── G1ParEvacuateFollowersClosure::do_void
       └── G1ParScanThreadState::trim_queue
            └── G1ParScanClosure::do_oop      ← ★ 占栈 70%
```

并行扫描 oop（对象引用）占 70% = **GC 在拼命扫存活对象**。

为什么突然？查 GC log：
```
[1234.567s][info][gc] GC(8901) Pause Young (Concurrent Start) (G1 Humongous Allocation) 230M->180M(2048M) 1500ms
```

**Humongous Allocation**（G1 里超过 region 一半大小的对象，默认 region 8MB → humongous 阈值 4MB）。每次分配都触发 CMS-like 的 concurrent cycle，频繁 GC。

业务侧追溯：上游接口刚改了，每次返回一个 8MB 的 `byte[]` 给我们做缓存。

#### 修复
1. **临时**：扩大 G1 region size：
   ```
   -XX:G1HeapRegionSize=32M
   ```
   把 region 加大，对象不再被判为 humongous。
2. **根本**：改业务，大对象分片或者只缓存指针。
3. **可选**：换 ZGC（对大对象更友好，无 humongous 概念）。

#### 验证
- GC log 里 `Humongous` 字样消失
- jstat -gc：YGC 间隔从 1s 涨到 30s
- CPU 回到 30%

> **口诀**：CPU 飙高第一步永远是 `top -Hp` 看是哪些线程烧 CPU。GC Thread / VM Thread / C2 CompilerThread 高 CPU 的根因都不在业务代码里，要分别去查 GC log / safepoint log / JIT 编译队列。

---

### 案例 ② P99 抖动但 CPU 平稳 —— "DB 连接池争用，off-CPU 才看见"

#### 现象
Grafana 显示某接口 P99 从 50ms 涨到 600ms，但 P50 仍是 20ms。CPU 利用率全程平稳在 40%。on-CPU 火焰图看不出任何异常，热点函数都是预期的业务方法。

#### 根因
**这是教科书式的 off-CPU 场景**。on-CPU 看不到的瓶颈，要去 wallclock / off-CPU 找。

```bash
./asprof -d 30 -e wall -t -f wall.html <pid>
```

`-t` 按线程拆开，找慢线程。点开一个 P99 线程的栈：

```
HikariCP$ConnectionBag.borrow             ← ★ 占该线程 90%
  └── java.util.concurrent.SynchronousQueue.poll
       └── LockSupport.parkNanos                ← 在等连接
            └── 业务 DAO.findOrder
```

90% 时间在 HikariCP 的 borrow 上 = **连接池没连接，线程在等**。

继续验证。HikariCP 暴露指标：
```
hikaricp.connections.usage     # 当前用了多少
hikaricp.connections.max       # 配置的最大
hikaricp.connections.pending   # 多少线程在等
```

`pending` 长期 > 0 = 池满了。

为什么？看 SQL 慢查询日志：DB 端某条 SQL 偶发跑 500ms（缺索引），业务并发抢连接 → 池被占满 → 后续请求等 → P99 飙升。

#### 修复
- **直接修因**：给那条慢 SQL 加索引（看慢查询日志找出来）
- **缓冲**：HikariCP 调大 `maximumPoolSize`（但 DB 端连接数有上限，别盲目调）
- **预防**：`setConnectionTimeout(3000)` 短超时 + 上层重试，避免雪崩

#### 验证
- 重新抓 wallclock，`HikariCP.borrow` 占比应 < 5%
- `hikaricp.connections.pending` 维持在 0
- P99 回到 50ms 内

> **口诀**：CPU 平稳但 P99 抖 = **必抓 off-CPU 或 wallclock 火焰图**。on-CPU 看不到等待，看不到锁，看不到 IO 阻塞。

---

### 案例 ③ 反射密集库的"隐性"开销定位 —— "Jackson 吃了 30% CPU"

#### 现象
某 Spring Boot 服务 CPU 利用率 70%，QPS 5000。本以为业务逻辑该是热点，抓火焰图后发现栈顶被 Jackson 占据。

```
com.fasterxml.jackson.databind.ObjectMapper.writeValueAsString    ← ★ 占 30%
  └── BeanSerializer.serialize
       └── BeanPropertyWriter.serializeAsField
            └── sun.reflect.GeneratedMethodAccessor*.invoke
```

#### 根因
对每个出参对象都要 `writeValueAsString` 序列化。BeanPropertyWriter 走反射调 getter。即使 Jackson 在第 16 次调用后切到字节码生成（`GeneratedMethodAccessor`），间接调用开销仍然不小。

更严重的是**没复用 ObjectMapper**：

```java
// ❌ 每次新建
public String toJson(Object o) {
    return new ObjectMapper().writeValueAsString(o);
}
```

`new ObjectMapper()` 内部要初始化数十个 `_DefaultModule`，并构建 `SerializerProvider` 缓存。每次新建 = 缓存被丢弃 = 反射回到第 1 次调用。

#### 修复
**修因 1**：ObjectMapper 全局单例。

```java
@Configuration
public class JacksonConfig {
    @Bean public ObjectMapper objectMapper() {
        ObjectMapper m = new ObjectMapper();
        m.registerModule(new BlackbirdModule());   // ★
        m.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        return m;
    }
}
```

`BlackbirdModule` 用 LambdaMetafactory 替代反射，**JDK 11+ 才能用**，序列化 CPU 开销可降 50%。Jackson 老版本用 `AfterburnerModule`（基于 ASM 字节码生成）。

**修因 2**：高频接口用代码生成式序列化。

[DSL-JSON](https://github.com/ngs-doo/dsl-json) 在编译期生成专用 ser/deser 类，性能比 Jackson + Blackbird 还快 30%。代价是要装注解处理器。

#### 验证
重新抓火焰图，`Method.invoke` 在栈中应几乎消失，BeanSerializer 直接走 BlackbirdMethod 调用。

> **口诀**：栈中看到 `Method.invoke` / `MethodHandle.invokeExact` 占栈高 = 反射或动态代理开销。**ObjectMapper 必须单例 + BlackbirdModule** 是 Jackson 老用户经常漏的一刀。

---

### 案例 ④ ThreadLocal.get 占栈 40% —— "ThreadLocalMap 退化成线性扫描"

#### 现象
某请求处理函数的火焰图看着特别奇怪：

```
java/lang/ThreadLocal.get                ← ★ 占栈 40%
  └── ThreadLocal$ThreadLocalMap.getEntry
       └── ThreadLocal$ThreadLocalMap.getEntryAfterMiss     ← ★ 名字就有 "AfterMiss"
            └── ThreadLocal$ThreadLocalMap.expungeStaleEntry
```

`getEntryAfterMiss` 出现 = 哈希表第一次查找没命中，开始线性探测。`expungeStaleEntry` 出现 = 在清理"陈旧"的 entry。

#### 根因
ThreadLocal 的存储是 `ThreadLocalMap`，本质是**开放寻址哈希表**（不是链地址法）。它的 key 是 `WeakReference<ThreadLocal>`：

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;
    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

key 是弱引用 → ThreadLocal 对象被回收时 key 变成 null，但 **value 不会自动清** → 形成"stale entry"。哈希冲突时要线性探测，遇到 stale 就清掉。

**触发条件**：
- 大量短生命周期的 ThreadLocal 对象（比如每个请求 new 一个匿名 `ThreadLocal<>`）
- 哈希冲突堆积

线程池里 worker 是复用的 → ThreadLocalMap 跟着复用 → 长时间运行后表里堆满了 stale entry → `getEntry` 频繁退化到线性扫描。

#### 修复
**根本修因**：ThreadLocal 应该是 `static final` 的"全局少量"模式，不是"每请求一个"模式。

```java
// ❌ 错的
public class RequestContext {
    public RequestContext() {
        ThreadLocal<User> tl = new ThreadLocal<>();  // 每次 new，灾难
        ...
    }
}

// ✅ 对的
public class RequestContext {
    private static final ThreadLocal<User> CURRENT_USER = new ThreadLocal<>();
}
```

**用完必须 remove**：

```java
try {
    CURRENT_USER.set(u);
    handle();
} finally {
    CURRENT_USER.remove();   // ★
}
```

不 remove 不仅有 stale entry，还有**内存泄漏**（worker 持有 user 对象不释放）。

如果业务真的需要"很多 ThreadLocal"，换 [TransmittableThreadLocal](https://github.com/alibaba/transmittable-thread-local) 或者干脆用 `ScopedValue`（JDK 21+）。

#### 验证
重新抓火焰图，`ThreadLocal.get` 占比应 < 1%，不再出现 `getEntryAfterMiss` / `expungeStaleEntry`。

> **口诀**：栈里看到 `ThreadLocal.getEntryAfterMiss` 高占比 = ThreadLocalMap 退化。**ThreadLocal 必须是 static final**；用完必须 `remove()`。

---

### 案例 ⑤ JVM SafePoint 偏差 —— "perf 把热点指错了"

#### 现象
某 CPU 密集型计算服务，用 `perf record + flamegraph` 抓到的火焰图里栈顶是：

```
JIT_xxx_stub
  └── java.lang.Math.cos
```

业务团队改了 `Math.cos` 的实现想优化，结果 CPU 没下降。后来用 async-profiler 重抓，发现真正的热点是另一个内联进 hot loop 的方法 `vectorize()`。

#### 根因
**这是 SafePoint Bias 的典型案例**。

回顾 1.6：perf 采到 PC 后查 `/tmp/perf-<pid>.map`，但 JIT 优化通常做了：
1. **方法内联**：`vectorize()` 被内联进调用者，没有独立 frame
2. **safepoint 省略**：在没有循环回边的 hot path 上 JIT 会**省略 safepoint poll** 以提升性能
3. perf-map-agent 默认只在 safepoint 处发布符号，省略 safepoint 的代码段就只能映射到包含它的"包络方法"上

结果：真正的热点 `vectorize()` 在 perf 火焰图里"消失"了，被错指到容器它的方法上。

**用 async-profiler 重抓**：`AsyncGetCallTrace` 不依赖 safepoint，配合 `-XX:+DebugNonSafepoints` 的非 safepoint 行号信息，能看到真正的内联调用栈：

```
com.foo.Calc.compute
  └── (inlined) com.foo.Calc.vectorize     ← ★ 真热点
       └── (inlined) java.lang.Math.fma
```

#### 修复
1. **永远用 async-profiler 看 JVM 火焰图**，不用 perf（除非你要看 syscall 内核栈）
2. JVM 启动加 `-XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints -XX:+PreserveFramePointer`
3. 真要用 perf 看 JVM，用 [perf-map-agent](https://github.com/jvm-profiling-tools/perf-map-agent) 并配 `unfoldall=true` 让 inline 也展开

#### 验证
两个工具同时抓，对比栈顶函数。async-profiler 应能看到 inline 之后的真实热点。

> **口诀**：JVM 应用的 CPU 火焰图永远用 async-profiler。perf 看 JVM = SafePoint Bias + inline 丢失 = 至少两层失真。

---

### 案例 ⑥ 容器内符号缺失 —— "火焰图全是 [unknown]"

#### 现象
公司新 Pod 上线后第一次抓火焰图，打开 SVG 一看："谁是 alice？"

```
[unknown]
  └── [unknown]
       └── [unknown]
            └── 0x7f12345678ab
```

栈深 20 层，没一个能看的函数名。

#### 根因
这是容器化时代最常见的火焰图坑。原因可能多个：

1. **JVM 没开 `-XX:+PreserveFramePointer`**：JIT 代码段没维护 RBP，stack walker 走不通
2. **没装 perf-map-agent**：perf 不知道 JIT 代码段的符号
3. **基础镜像是 Alpine**：musl libc 没有 glibc 的 `.eh_frame`
4. **distroless 镜像**：连 `/proc/<pid>/maps` 里的 so 都没有 debuginfo
5. **使用了 `-Xcomp` 强制编译**，但没启用调试符号选项

#### 修复

**JVM 侧**：
```dockerfile
ENV JAVA_TOOL_OPTIONS="-XX:+UnlockDiagnosticVMOptions \
                       -XX:+DebugNonSafepoints \
                       -XX:+PreserveFramePointer"
```

**镜像侧**：换基础镜像。
```dockerfile
# ❌ 别用 alpine 跑 JVM 搞 profiling
# FROM eclipse-temurin:21-jre-alpine

# ✅ 用 jammy / focal
FROM eclipse-temurin:21-jre-jammy
```

**符号侧**（如果非要用 perf）：
```dockerfile
RUN apt-get install -y libc6-dbg libstdc++6-dbg
```

**async-profiler 通常不需要单独装符号**，它直接用 JVM 的 `AsyncGetCallTrace`。如果 async-profiler 也是 `[unknown]`，几乎就是 PreserveFramePointer 没开。

#### 验证

```bash
# 检查 JVM 启动参数
jcmd <pid> VM.flags | grep -E 'PreserveFramePointer|DebugNonSafepoints'

# 应该看到
-XX:+DebugNonSafepoints
-XX:+PreserveFramePointer
```

抓一个 5 秒的火焰图看栈完整性。栈顶是有意义的方法名 = OK。

> **口诀**：容器化 JVM 的"profiling 三件套"必加：`PreserveFramePointer + DebugNonSafepoints + UnlockDiagnosticVMOptions`。基础镜像不要用 alpine。

---

### 案例 ⑦ async-profiler alloc 模式定位内存泄漏

#### 现象
某后台服务运行 12 小时后 OOM。`-Xmx4g`，dump 文件 4GB。MAT 看 `byte[]` 占了 3GB，retainer 路径指向某个 `ConcurrentHashMap`，但代码里这个 Map 应该有 evict 的，逻辑看起来没问题。

#### 根因
传统 OOM 分析靠 heap dump，看的是**当前快照**。但泄漏问题往往是"谁分配的"比"谁持有的"更关键。

用 async-profiler 的 `alloc` 事件：
```bash
./asprof -e alloc --alloc 1m -d 600 -f alloc.html <pid>
# 600 秒 = 10 分钟
# --alloc 1m 只采样 ≥1MB 的分配
```

`alloc` 模式的原理：HotSpot 在 TLAB（Thread Local Allocation Buffer）满 / 直接分配大对象到 old gen 时会调 slow path，async-profiler 在 slow path 上挂钩子采样。**开销极低**，因为只采"大对象 + 慢路径"。

火焰图栈顶看：
```
java.util.HashMap.resize                  ← 哪里在大量分配
  └── HashMap.putVal
       └── 业务 com.foo.MetricCollector.report
            └── BizService.handle
```

`MetricCollector.report` 在大量调用 `HashMap.put`。看代码：

```java
public class MetricCollector {
    private static final Map<String, Long> COUNTERS = new ConcurrentHashMap<>();
    public void report(String key, long value) {
        COUNTERS.merge(key, value, Long::sum);
    }
}
```

`key` 是用户 ID + 方法名拼接的，**有上千万种唯一值**。Map 永远在涨，没 evict，最终撑爆堆。

#### 修复
- 加 LRU：换成 Caffeine `Cache.builder().maximumSize(100_000).build()`
- 或换成 Micrometer 的 `Counter`（按 tag 维度，但 tag 应该是低基数）

#### 验证
重新抓 alloc，`MetricCollector.report` 不再出现在栈顶。Heap usage 长期稳定。

> **口诀**：内存泄漏不要只看 dump，**要看"谁在分配"**。async-profiler `-e alloc` 是定位分配热点的最快路径。`--alloc 1m` 只采大对象，几乎零开销，可以常开。

---

### 案例 ⑧ 差分火焰图找回归 —— "上线 v2.3 后 P99 涨了 30ms"

#### 现象
v2.3 灰度上线后，监控显示 P99 从 80ms 涨到 110ms。Code Review 看不出可疑改动。

#### 根因
直接抓两份火焰图做差分。

```bash
# 在 v2.2 灰度 Pod 上
./asprof -d 60 -e cpu -o collapsed -f v22.folded <pid>

# 在 v2.3 灰度 Pod 上
./asprof -d 60 -e cpu -o collapsed -f v23.folded <pid>

# 差分
./difffolded.pl v22.folded v23.folded > diff.folded
./flamegraph.pl --negate diff.folded > diff.svg
```

打开 `diff.svg`，红色越深的格子越值得怀疑。看到一片深红：

```
java.util.regex.Pattern.compile          ← ★ 深红
  └── 业务 ValidationService.validate
       └── 业务 RestController.handle
```

`Pattern.compile` 在 v2.3 里采样次数飙升。看 git diff：

```diff
+ public boolean validate(String input) {
+     Pattern p = Pattern.compile("^[a-zA-Z0-9]{3,20}$");   // ★ 每次调用都 compile
+     return p.matcher(input).matches();
+ }
```

新加的"每次调用都编译正则"是元凶。Pattern.compile 是个相对重的操作（涉及 NFA 构建）。

#### 修复
```java
private static final Pattern P = Pattern.compile("^[a-zA-Z0-9]{3,20}$");

public boolean validate(String input) {
    return P.matcher(input).matches();
}
```

#### 验证
再抓一次差分火焰图，`Pattern.compile` 应消失。P99 回到 80ms。

> **口诀**：版本对比性能问题的最高效定位方式 = **差分火焰图**。一张图比读几百行 git diff + 几小时 trace 还快。

---

### 案例 ⑨ 连续剖析发现"周期性"隐性热点 —— "每月 1 号那 5 分钟"

#### 现象
某账单服务平时一切正常。但运维报告"每月 1 号凌晨 0:05 - 0:10 之间 P99 飙升"。问题持续了半年没人定位 —— 因为告警过去了再去抓火焰图，已经看不到现场。

#### 根因
这正是**连续剖析**（Continuous Profiling）解决的问题。Pyroscope 持续采集 + 时序存储，事后回看任意时间段的火焰图。

部署 Pyroscope 后等下个月 1 号事故复现，到 UI 里选时间窗 `[2024-05-01 00:05, 2024-05-01 00:10]`：

```
com.foo.MonthlyJob.aggregateAll          ← ★ 占该时段 95%
  └── java.util.stream.Collectors.toList
       └── HashMap.put
            └── (深栈)
```

`MonthlyJob.aggregateAll` 在做月度聚合，在内存里用 stream 把全月数据 group 起来。常态运行时这个方法不跑，所以平时火焰图看不到。

代码里有个 cron：
```java
@Scheduled(cron = "0 5 0 1 * ?")   // 每月 1 号 0:05
public void aggregateAll() { ... }
```

#### 修复
- 业务逻辑：聚合下沉到 DB 做（GROUP BY）或异步分批
- 调度：错峰，避免与其他月初任务挤在同一窗口
- 未来：定期 review 所有 `@Scheduled`，识别"月度尖峰"

#### 验证
下个月 1 号在 Pyroscope 上看 0:05 - 0:10 时段，`MonthlyJob.aggregateAll` 不再占主。P99 平稳。

> **口诀**：**周期性 / 偶发性 / 短脉冲性**热点必须靠连续剖析。事后回看才是核心价值。一次性 perf 抓到的概率几乎为零。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### JVM 启动参数
- [ ] `-XX:+UnlockDiagnosticVMOptions` 开诊断选项
- [ ] `-XX:+PreserveFramePointer` 保留 frame pointer，否则火焰图栈断裂
- [ ] `-XX:+DebugNonSafepoints` 让非 safepoint 处也有行号信息
- [ ] `-Djdk.attach.allowAttachSelf=true`（async-profiler attach 需要）
- [ ] 容器跑 JVM 不要用 alpine 基础镜像，用 jammy / ubi9

### 工具链
- [ ] 业务镜像里预装 [arthas](https://arthas.aliyun.com/)（线上 attach 入口）
- [ ] CI 镜像里有 async-profiler，方便压测时一键抓
- [ ] 持有 `perf` 的宿主权限的 SRE 知道怎么 `--ns` attach 容器
- [ ] FlameGraph + difffolded 工具脚本预置在 ops 仓库

### 容器与符号
- [ ] 镜像里 glibc-debuginfo / libstdc++-debuginfo 装好（perf 需要）
- [ ] cgroup v2 环境验证 `perf` 能拿到事件（必要时给 `--cap-add=PERFMON`）
- [ ] `/proc/sys/kernel/perf_event_paranoid <= 2`（生产环境通常 = 2，足够）
- [ ] `kernel.kptr_restrict = 0`（看内核栈需要）

### 采样规范
- [ ] 默认采样频率 99Hz / 10ms，避免被周期性事件抢拍
- [ ] CPU 利用率 + 火焰图必须**一起看**，单独看火焰图易误读
- [ ] 抓 wallclock / off-CPU 时务必带 `-t`（按线程拆分）
- [ ] 抓 alloc 时加 `--alloc 1m`，避免小对象采样过密

### 连续剖析
- [ ] 关键服务接 Pyroscope / Parca，留 7 天以上历史
- [ ] 接 Grafana 数据源，告警时一键关联火焰图
- [ ] 监控 Pyroscope server 自身的存储水位，定期归档
- [ ] CI/CD pipeline 里集成 diff 火焰图，回归大于阈值阻断发布

### 读图能力
- [ ] 团队 onboarding 培训有"火焰图读法"环节
- [ ] 内部知识库收录"误读案例集"（epoll_wait、SafePoint Bias 等）
- [ ] 每次事故复盘必须附上当时的火焰图 + 解读

### 性能边界
- [ ] async-profiler 常驻 < 2% CPU 开销（通常远低于此）
- [ ] Pyroscope agent 默认 100Hz，per pod < 5GB/天 原始数据
- [ ] perf 大流量场景注意 ring buffer 不够会丢样本：`perf record -m 256M`

---

## 6. 还能往下挖什么

### Brendan Gregg 《Systems Performance》关键章节
- 第 6 章 CPU：USE 方法、调度器、剖析方法学
- 第 9 章 Disks：IO 队列、调度算法、队列论
- 第 13 章 perf 的全部用法（不止 record，还有 stat / trace / annotate）
- 第 14 章 Ftrace（perf 的"亲戚"，更轻量的内核事件追踪）
- 这本书是性能工程的"九阴真经"，每年回读一次都有新收获

### eBPF + 现代剖析
- [Parca](https://www.parca.dev/) —— 基于 eBPF 的 always-on 系统级剖析，零侵入
- [Pixie](https://px.dev/) —— Kubernetes 原生可观测性，eBPF 自动注入
- BCC / bpftrace 写自定义 profiler，几行脚本搞定特殊采样
- eBPF 抓栈支持 LBR / FP / DWARF 三种，且能跨语言

### bpftrace 一行脚本
```bash
# 谁在大量分配大对象
bpftrace -e 'usdt::libc:malloc /arg0 > 1024*1024/ { @[ustack] = count(); }'

# 谁在做 page fault
bpftrace -e 'kprobe:handle_mm_fault { @[comm] = count(); } interval:s:1 { print(@); clear(@); }'

# 谁的 read 系统调用慢
bpftrace -e 'tracepoint:syscalls:sys_enter_read { @start[tid] = nsecs; }
             tracepoint:syscalls:sys_exit_read /@start[tid]/ {
                 @ms = hist((nsecs - @start[tid]) / 1000000);
                 delete(@start[tid]);
             }'
```

bpftrace 是"现场救火"的神器，遇到不寻常的现象时能立刻写一段脚本验证假设。

### JFR 与 JMC 高级用法
- 自定义事件：`@Name("biz.OrderCreated") class OrderEvent extends Event`
- JFR 流式 API（JDK 14+）：实时消费 JFR 事件
- JMC 的 Automated Analysis：自动找出 GC、JIT、锁竞争问题
- JFR + Flame Graph 联动：[jfr-flamegraph](https://github.com/billywhizz/jfr-flamegraph)

### Linux perf 的高级模式
- `perf trace`：strace 的高性能替代
- `perf sched`：调度延迟分析
- `perf c2c`：cache line 伪共享检测（NUMA 性能调优必备）
- `perf mem`：内存访问采样（哪些访问 miss L1/L2/L3 cache）
- `perf annotate`：把 perf 采样按指令展开，看哪行汇编最热

### 连续剖析的成本与隐私
- 千节点级集群的剖析成本控制（采样率自适应、按 label 抽样）
- 火焰图含函数名 → 可能泄漏业务逻辑，需要"脱敏"处理
- GDPR 场景下 profiling 数据的合规性
- 内部 Pyroscope 替代品：[Polar Signals](https://polarsignals.com/) (Parca 商业版)、[Datadog Continuous Profiler](https://docs.datadoghq.com/profiler/)、[Grafana Cloud Profiles](https://grafana.com/products/cloud/profiling-for-observability/)

### GPU / NIC / SSD 的剖析
- GPU：NVIDIA Nsight Systems、NVTX 标签
- NIC：RDMA + 内核 tracepoints + DPDK 自带的 perf counters
- SSD：blktrace + 队列延迟分布
- 这一层是性能调优的"第三象限"，用得不多但用一次救一次

---

## 7. 引用

- Brendan Gregg, *Flame Graphs* —— [原始论文与可视化规则](https://www.brendangregg.com/flamegraphs.html)
- Brendan Gregg, *Systems Performance: Enterprise and the Cloud (2nd ed.)* —— 性能工程圣经
- async-profiler 项目主页：[async-profiler/async-profiler](https://github.com/async-profiler/async-profiler)
- AsyncGetCallTrace 的设计与陷阱：Krzysztof Ślusarski 的[博客系列](https://krzysztofslusarski.github.io/)
- perf 用户手册：[perf wiki](https://perf.wiki.kernel.org/)
- Pyroscope 文档：[Continuous Profiling](https://grafana.com/docs/pyroscope/)
- Brendan Gregg, *Off-CPU Analysis* —— [off-CPU 火焰图源出处](https://www.brendangregg.com/offcpuanalysis.html)
- Nitsan Wakart 系列博客：[psy-lob-saw](http://psy-lob-saw.blogspot.com/)（safepoint bias / JVM profiling 内幕）
- Aleksey Shipilëv, *Java Memory Layout* + *Nanotrusting the Nanotime* 系列
- arthas 文档：[Alibaba Arthas](https://arthas.aliyun.com/doc/)

---

> 配套：[Linux 与性能工程主题导览](/article/linux-perf/) ｜ [JVM 调优与排查](/article/jvm/tuning) ｜ [线程池生产实战](/article/java/threadpool)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你觉得这个深度对，告诉我下一篇该写哪个。
