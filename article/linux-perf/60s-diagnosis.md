# Linux 60 秒诊断 —— Brendan Gregg 标准动作 + 真实场景剧本

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想速记，去 [Linux 性能工程综述](/article/linux-perf/) 那一节够了。
> 这里要回答的，是**半夜被告警叫起、SSH 进机器只有 60 秒做出第一判断**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. Brendan Gregg 的 60 秒诊断 10 条命令分别是哪些？每条**主要看哪一类瓶颈**（CPU / 内存 / IO / 网络 / 内核）？凭记忆敲出顺序。
2. `vmstat 1` 输出里的 `r` 列是 **RUNNABLE**（包含正在跑的）还是只是 **RUNNING**？这个数字 = 8 在 4 核机器上意味着什么？
3. `uptime` 给的 `load average` 三个数 1/5/15 分钟，**它们的相对关系**告诉你什么？1 分钟远高于 15 分钟 vs 1 分钟远低于 15 分钟，分别是什么场景？
4. `mpstat -P ALL 1` 的核心价值是什么？为什么 `top` / `vmstat` 看 CPU 不够、必须看每核？给一个"`top` CPU 100% 但其实没事"的反例。
5. `pidstat -d 1` 的 `kB_rd/s` / `kB_wr/s` / `kB_ccwr/s` 三列分别什么含义？为什么 `iostat` 看磁盘忙、却找不到是哪个进程？
6. `iostat -xz 1` 的 `await` 和 `svctm` 差别是什么？为什么 RHEL 7+ 上 `svctm` 已经被打上"不要再用"的标签？现在该看哪些列？
7. `sar` 的历史数据从哪里来？`/var/log/sa/saXX` 是怎么生成的？没装 `sysstat` 的机器，事后想看半小时前的指标怎么办？
8. `free -h` 的 `available` 和 `free` 的差别是什么？什么时候 `free` 极小但系统其实没有内存压力？什么时候 `available` 极小才该真正紧张？

如果有半数答不上，往下看。

---

## 1. 底层模型：从症状到瓶颈类型

绝大多数人对"性能诊断"的认知止于"看一眼 `top`"。但 `top` 给的是**当前快照**，看不到上下文切换、看不到每核分布、看不到磁盘队列深度。要在 60 秒内做出第一判断，必须有一个**结构化的扫描顺序**。

### 1.1 USE 模型在 Linux 上的具体落地

USE 模型（Brendan Gregg 提出）把每个资源按三个维度看：

| 维度 | 英文 | 含义 | Linux 上看哪 |
| --- | --- | --- | --- |
| **使用率** | Utilization | 资源繁忙的时间占比 | `mpstat`、`iostat` 的 `%util` |
| **饱和度** | Saturation | 排队等待该资源的请求量 | `vmstat` 的 `r` 和 `b`、`iostat` 的 `aqu-sz` |
| **错误** | Errors | 资源相关的错误计数 | `dmesg`、`netstat -s`、`ifconfig` 的 errors |

**为什么"使用率高"不等于"有问题"**：磁盘 100% util 但 await 只有 1ms，应用根本不在意；CPU 100% 但 run queue 一直是 1，说明负载饱满地跑着、没有等待。

**为什么"使用率低"也可能有问题**：CPU 30% 但其中 1 个核 100%（GIL / 单线程任务），整体响应慢；磁盘 5% util 但 await 突然 200ms（IO 偶发抖动），应用 P99 会爆。

`60 秒诊断`的 10 条命令，本质上是**沿着 USE 模型把所有资源扫一遍**。

### 1.2 性能瓶颈的"四个层级"

```
┌─────────────────────────────────────────┐
│  L1 进程层：哪个进程出了问题             │  top / pidstat
├─────────────────────────────────────────┤
│  L2 子系统层：哪类资源出了问题           │  vmstat / iostat / sar
├─────────────────────────────────────────┤
│  L3 内核层：内核的何种行为引起的         │  dmesg / perf / ftrace
├─────────────────────────────────────────┤
│  L4 硬件层：硬件是否真的不够用           │  mpstat / iostat -x / ipmi
└─────────────────────────────────────────┘
```

**60 秒诊断的目的是定位 L2**（"是 CPU 还是 IO 还是网络"），下一步才决定向上钻 L1（找进程）还是向下钻 L3/L4（找内核或硬件证据）。

不要一上来就 `strace`、就 `perf`，那是钻 L3 的工具，跳过 L2 直接钻 L3 等于盲人摸象。

### 1.3 60 秒诊断的命令清单

Brendan Gregg 在 Netflix 性能工程团队总结的标准 10 条：

```bash
uptime
dmesg | tail
vmstat 1
mpstat -P ALL 1
pidstat 1
iostat -xz 1
free -m
sar -n DEV 1
sar -n TCP,ETCP 1
top
```

每条 5~10 秒，整套跑完正好 60 秒。每条覆盖的"判断面"：

| 命令 | 主要看 | 一句话总结 |
| --- | --- | --- |
| `uptime` | load average | 总体负荷趋势（1/5/15 分钟） |
| `dmesg \| tail` | 内核错误 | 有没有 OOM kill / TCP drop / 硬件错误 |
| `vmstat 1` | run queue / 内存 / swap / IO / CPU | 一行看出系统全貌 |
| `mpstat -P ALL 1` | 每核 CPU 利用率 | 是不是单核热点 |
| `pidstat 1` | 进程级 CPU | 哪个进程在烧 CPU |
| `iostat -xz 1` | 磁盘 IO | 哪个盘忙、await 多少 |
| `free -m` | 内存 / buff / cache | 是真的紧张还是误读 |
| `sar -n DEV 1` | 网络流量 | 哪个网卡跑满了 |
| `sar -n TCP,ETCP 1` | TCP 错误 | 重传 / 拒绝 / 主动关闭 |
| `top` | 综合 | 兜底确认 |

> **口诀**：60 秒诊断不是"看一眼 top"，而是**沿 USE 模型把 5 类资源系统扫一遍**，目的是把瓶颈定位到 L2 子系统层。

### 1.4 从"症状"到"瓶颈类型"的快速分类

收到的告警通常是一句模糊的话："服务慢"、"接口超时"、"CPU 高"。不要被告警的措辞带偏。先扫一遍 USE，再做分类：

| 你看到的 | 大概率瓶颈 | 第一反应工具 |
| --- | --- | --- |
| `load avg` 高 + `vmstat r` 高 + 各核 CPU 都高 | **CPU 真的不够** | `pidstat` 找进程、`perf top` 找热点 |
| `load avg` 高 + `vmstat r` 低 + `vmstat b` 高 | **IO 等待** | `iostat -x`、`pidstat -d` |
| `load avg` 高 + 单核 100%、其他核空 | **单线程瓶颈** | `pidstat -t` 看线程、`perf` 看栈 |
| `vmstat si/so` 非零 | **swap 在动**，内存压力 | `free`、`/proc/PID/oom_score` |
| `free` 看 `available` 极少 | **内存压力**或大文件页缓存 | `cat /proc/meminfo` 细看 |
| `dmesg` 有 OOM | **触发过 OOM kill** | 先看哪个进程被杀，再决定是限内存还是改代码 |
| `sar -n DEV` 看到带宽接近网卡上限 | **网络打满** | `iftop`、`nethogs`、应用侧限流 |
| `sar -n ETCP` 的 `retrans` 非零 | **TCP 重传**，网络质量差或对端过载 | `ss -ti`、抓包 |

**这张表是本文的核心**。后面所有命令解读和案例都是在填充这张表的细节。

> **口诀**：先 USE 扫一遍 → 定位 L2 子系统 → 再向上找进程或向下找内核证据。**别跳级**。

---

## 2. 命令切片：10 条命令深度解读

> 每条命令贴出真实输出样例 + 关键字段含义 + 反直觉点。版本基于 Linux 5.x + sysstat 12.x。

### 2.1 uptime —— load average 三档

```bash
$ uptime
 14:22:38 up 32 days,  4:10,  3 users,  load average: 12.45, 8.21, 5.03
```

`load average` 的三个数字是 **1 分钟 / 5 分钟 / 15 分钟**的移动平均。Linux 上的 load 不只是 CPU，**包含 RUNNABLE + UNINTERRUPTIBLE（D 状态）的进程数**。

#### 三档的相对关系告诉你什么

```
1m > 5m > 15m  →  负荷在上升（事件正在发生）
1m < 5m < 15m  →  负荷在下降（事件已经过去）
1m ≈ 5m ≈ 15m  →  稳态（持续高 / 持续低都可能）
```

收到告警时**先看趋势**：1m 高 5m 低 = 突发；1m 低 5m 高 = 已过去（你来晚了）。

#### load = 多少算高

经典口诀："load < CPU 核数"。但这里的 CPU 核数是**逻辑核**（包括超线程），且这个口诀**只在不含 D 状态进程时**才成立。

更准确的做法：
- `nproc` 看核数
- `vmstat 1` 看 `r`（runnable）和 `b`（blocked，D 状态）
- 如果 load 高但 `r` 不高，说明是 IO 等待撑起来的 load —— 不是 CPU 不够，是磁盘或网络

#### 真实生产例子

某 4 核机器 load average 达到 80。值班一看以为 CPU 爆了，结果 `mpstat` 看每核 CPU 都不到 20%。

`vmstat` 一看 `b` 列 = 76 —— 76 个进程卡在 D 状态。`iostat` 看磁盘 await 飙到 800ms，问题是 NFS 挂载点 hung 了，不是 CPU 不够。

> **口诀**：load 高第一反应**不是**"加机器"，是**先看 r 还是 b 撑起来的**。r 高才是 CPU 不够，b 高是 IO 阻塞。

### 2.2 dmesg | tail —— 内核错误

```bash
$ dmesg --ctime | tail -20
[Fri Mar 15 14:18:42 2024] Out of memory: Killed process 18934 (java) total-vm:8192000kB, ...
[Fri Mar 15 14:18:43 2024] oom_reaper: reaped process 18934 (java), now anon-rss:0kB, ...
[Fri Mar 15 14:19:01 2024] TCP: out of memory -- consider tuning tcp_mem
[Fri Mar 15 14:20:15 2024] nf_conntrack: nf_conntrack: table full, dropping packet
```

#### 必须立刻警觉的几类

| 关键字 | 含义 | 处理方向 |
| --- | --- | --- |
| `Out of memory: Killed process` | OOM Killer 杀过进程 | 看 oom_score、限内存或改代码 |
| `nf_conntrack: table full` | 连接跟踪表满 | 调 `nf_conntrack_max` 或关 conntrack |
| `TCP: out of memory` | TCP 缓冲区不够 | 调 `tcp_mem` |
| `tcp: too many orphaned sockets` | 半关闭连接太多 | 看应用是不是 close 出问题 |
| `task XXX blocked for more than 120 seconds` | 任务卡在内核 120s | 多半 IO hung，看磁盘 |
| `Buffer I/O error on device` | 磁盘错误 | 立刻换盘 |
| `EDAC`、`MCE` | 内存 / CPU 硬件错误 | 报修 |

#### 用 `--ctime` 加时间戳

裸 `dmesg` 输出的是开机后的秒数，对人不友好。`--ctime` 转成绝对时间，方便和告警时间对齐。新机器还可以用 `journalctl -k` 拿到带时间的内核日志。

#### 反直觉的点

`dmesg` 是个**环形缓冲区**，默认大小 1MB（可调 `dmesg -s`）。如果一直在打日志，**老的会被覆盖**。生产排查时如果发现 `dmesg` 只有最近几分钟的内容，说明日志被冲掉了，要赶紧 `journalctl -k --since "1 hour ago"` 抢救。

> **口诀**：诊断先扫 `dmesg`。**OOM、conntrack 满、TCP 内存、IO hung、硬件错误**这五类是必查关键字，秒级判断"是不是触发过内核兜底动作"。

### 2.3 vmstat 1 —— 进程 / 内存 / IO / CPU

`vmstat` 是单条命令信息密度最高的工具，一行覆盖五大子系统。

```bash
$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 8  0      0  3.2G   240M  12.5G    0    0    52   180  120   85 35  5 58  2  0
 9  1      0  3.2G   240M  12.5G    0    0    48   220 8220 14210 75 12  8  5  0
12  0      0  3.1G   240M  12.5G    0    0     0   260 9100 16500 88 10  1  1  0
```

#### 关键字段逐列点透

| 列 | 含义 | 关键阈值 / 解读 |
| --- | --- | --- |
| **r** | runnable 进程数（含正在跑 + 等 CPU） | `> CPU 核数`持续 = CPU 饱和 |
| **b** | uninterruptible（D）状态进程数 | 持续 > 0 = IO / 内核 hung |
| **swpd** | 已用 swap | 非 0 不一定有问题，看 si/so |
| **free** | 空闲内存 | 不是关键，看 `available` 更准 |
| **buff/cache** | 文件页缓存 | 这部分内存是**可回收**的 |
| **si/so** | swap in / swap out（KB/s） | **任何持续非 0 都是警报** |
| **bi/bo** | block in / block out（KB/s） | 看趋势，配合 iostat |
| **in** | 中断数 | 突增看是哪种中断（`/proc/interrupts`） |
| **cs** | 上下文切换数 | 突增 = 锁竞争 / 线程过多 |
| **us/sy/id/wa/st** | 用户态/内核态/空闲/IO 等待/被偷 CPU | 五项加起来 = 100% |

#### 第一行是"开机以来平均"

`vmstat 1` 第一行**不是当前 1 秒的值**，是**开机到现在的平均值**。诊断要看从第二行开始的 1 秒粒度数据。

很多新人盯着第一行 `wa = 0.5%` 说"没事啊"，其实第二行开始 `wa = 80%`。

#### r 列的精确含义

`r` 列在不同内核版本含义微妙不同。**Linux 2.6 之后 `r` = 正在 CPU 上跑 + 在 run queue 里等的总和**（即 RUNNABLE）。

意思：`r = 8` 在 4 核机器上，意味着此刻有 8 个进程"想跑"，但只有 4 个能真正占着 CPU，**4 个在排队**。

经验阈值：`r` 持续 > CPU 核数的 1.5 倍 = CPU 已经是瓶颈。

#### cs 暴增的常见原因

| cs 数量级 | 解读 |
| --- | --- |
| 几百 ~ 几千 | 正常 |
| 上万 | 多线程应用，正常 |
| 几十万 | 锁竞争 / 自旋 / 调度抖动 |
| 上百万 | 几乎肯定有问题，看 `pidstat -w` 找元凶 |

#### st 列 —— 虚拟机才有

`st`（steal）= **被 hypervisor 偷走的 CPU 时间**。在云上才有意义。如果 `st` 持续 > 5%，说明你的虚拟机被同宿主邻居挤了，要么投诉云厂商、要么换实例规格。

> **口诀**：vmstat 是**单条命令信息密度之王**。看 r 找 CPU 饱和、看 b 找 IO 阻塞、看 si/so 找内存压力、看 cs 找锁竞争、看 st 找邻居打架。

### 2.4 mpstat -P ALL 1 —— 每核 CPU

```bash
$ mpstat -P ALL 1
14:25:01     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
14:25:02     all   72.51    0.00    8.21    1.10    0.00    0.51    0.00    0.00    0.00   17.67
14:25:02       0   99.00    0.00    1.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
14:25:02       1    5.05    0.00    2.02    1.01    0.00    0.00    0.00    0.00    0.00   91.92
14:25:02       2   80.00    0.00   18.00    0.00    0.00    2.00    0.00    0.00    0.00    0.00
14:25:02       3   85.00    0.00   12.00    1.00    0.00    1.00    0.00    0.00    0.00    1.00
```

#### 为什么必须看每核

`top` 默认只给"全核平均"。在 4 核机器上 CPU 显示 50%，可能是：
- **场景 A**：4 核都各跑 50%（健康，可以扩容）
- **场景 B**：1 核跑 100%、3 核跑 30%（GIL / 单线程瓶颈，扩核没用）
- **场景 C**：2 核跑 100%、2 核跑 0%（负载不均，亲和性配置错）

三种场景的修复方向**完全不同**。`mpstat -P ALL` 一眼看穿。

#### %sys 高 vs %usr 高

| %usr 高 | 应用代码烧 CPU | `perf top -p PID` 看用户栈 |
| --- | --- | --- |
| **%sys 高** | 内核态烧 CPU（系统调用、锁、IO 中断） | `perf top` 看内核栈，关注 `_raw_spin_lock` / `kmalloc` |
| **%iowait 高** | CPU 空闲但有进程在等 IO | `iostat`、`pidstat -d` |
| **%soft 高** | 软中断处理（多见于网络） | `cat /proc/softirqs`、看是不是 RPS 没开 |

#### %iowait 的常见误解

`%iowait` 不是"CPU 在干 IO 的时间"，而是**"CPU 空闲，但有进程在等 IO 完成"的时间**。

意思：`%iowait = 50%` 不代表 CPU 紧张，反而代表 CPU 闲着等 IO。瓶颈在磁盘，不在 CPU。

如果你的应用 CPU 密集 + IO 偶尔，`%iowait` 会被压低（因为 CPU 一直忙）；这时即使 IO 很慢，`%iowait` 也不会反应出来。

> **口诀**：CPU 整体看 `top` 不够，必须 `mpstat -P ALL` 看每核。**单核饱和 ≠ 整体饱和**，扩核之前先看是不是单线程瓶颈。

### 2.5 pidstat 1 —— 进程级 CPU

```bash
$ pidstat 1
14:30:02      UID       PID    %usr %system  %guest    %CPU   CPU  Command
14:30:03        0      1543    0.00    1.00    0.00    1.00     2  systemd
14:30:03     1000    18934   75.00   12.00    0.00   87.00     1  java
14:30:03     1000    19102   30.00    5.00    0.00   35.00     0  redis-server
```

#### 为什么用 pidstat 而不是 top

| 维度 | `top` | `pidstat` |
| --- | --- | --- |
| 输出 | 实时刷屏 | 滚动追加（可以保留） |
| 历史 | 只能看当前 | 多次采样直接对比 |
| 脚本友好 | 难解析 | 标准列，易 awk |
| 子选项 | 有限 | `-d` IO、`-r` 内存、`-w` 上下文切换、`-t` 线程 |

诊断时建议直接 `pidstat 1` 跑 30 秒，输出重定向到文件：

```bash
pidstat 1 30 > /tmp/pidstat.log
```

事后慢慢分析，比盯着 `top` 好太多。

#### -t 看线程（Java 应用必备）

```bash
pidstat -t -p <PID> 1
```

输出会把每个线程列出来。Java 应用看到 `TID` 后用：

```bash
# 把十进制 TID 转 16 进制
printf '%x\n' <TID>

# jstack 里搜 nid=0x<那个十六进制>
jstack <PID> | grep -A 20 'nid=0x...'
```

直接定位到具体 Java 线程的栈。

#### -d 看 IO

```bash
pidstat -d 1
14:32:01      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
14:32:02     1000    18934      0.00   1024.00      0.00      45  java
14:32:02       27    19500   2048.00      0.00      0.00      12  mysqld
```

| 列 | 含义 |
| --- | --- |
| `kB_rd/s` | 进程从磁盘读的速率 |
| `kB_wr/s` | 进程写到磁盘的速率 |
| `kB_ccwr/s` | 进程取消的写（被覆盖了） |
| `iodelay` | 进程等 IO 的时钟数（重要） |

`iostat` 看到磁盘忙 + `pidstat -d` 直接定位是哪个进程在 IO。这是 60 秒诊断里**最关键的串联**。

#### 瞬时进程的盲区

`pidstat 1` 每秒采样一次。如果是个秒级的瞬时进程（cron 跑的脚本、定时任务），`pidstat` **可能完全捕捉不到**。这时换工具：

```bash
# 用 perf record 跑 5 秒，捕获所有 fork 出来的进程
perf record -e sched:sched_process_exec -a sleep 5
perf script
```

或者用 BCC 工具集的 `execsnoop`（见案例 ⑤）。

> **口诀**：进程级 CPU / IO / 内存 / 上下文切换都用 `pidstat`，比 `top` 易解析、可保留、可串联。**Java 加 -t 看线程**，配合 jstack 直达栈。

### 2.6 iostat -xz 1 —— 磁盘 IO

```bash
$ iostat -xz 1
Device  r/s    w/s    rkB/s   wkB/s   rrqm/s wrqm/s %rrqm %wrqm  r_await w_await aqu-sz rareq-sz wareq-sz svctm  %util
sda     0.00   320.00 0.00    20480.00  0.00 80.00  0.00  20.00  0.00    25.50   8.20   0.00     64.00    3.10   100.00
nvme0n1 100.00 50.00  6400.00 3200.00   0.00 5.00   0.00  9.09   0.50    1.20    0.20   64.00    64.00    0.50   8.00
```

#### `-z` 的妙用

`-z` 表示"跳过完全没活动的设备"。生产机磁盘多，没有 `-z` 输出全是 0，眼睛累。

#### 关键字段（新版 sysstat）

| 列 | 含义 | 阈值参考 |
| --- | --- | --- |
| **r/s, w/s** | 读 / 写 IOPS | 看是否接近盘的额定值 |
| **rkB/s, wkB/s** | 读 / 写带宽 | 看是否接近带宽上限 |
| **rrqm/s, wrqm/s** | 合并的 IO 请求数 | 高 = 顺序 IO 被合并了 |
| **r_await, w_await** | 读 / 写平均等待时间（ms） | SSD < 1ms，HDD < 10ms |
| **aqu-sz** | 平均队列深度 | > 1 说明 IO 在排队 |
| **%util** | 设备繁忙时间占比 | 接近 100% 不一定真饱和（NVMe 可并行） |

#### svctm 已经被废弃

老资料里说"看 svctm"。RHEL 7+ 和新版 sysstat 已经把 `svctm` 标记为"过时、不可信"，因为现代 SSD/NVMe 是**多队列**的，单个 IO 的"服务时间"算不准。

现在该看的是 `r_await` / `w_await`，配合 `aqu-sz` 判断"是慢还是排队"。

#### %util = 100% 不一定饱和

机械盘是单队列，`%util = 100%` 确实是满了。但 **NVMe SSD 有几千个并行队列**，`%util = 100%` 只是"任意时刻有 IO"，远不到能力上限。要看 IOPS 是否接近规格书的额定值。

#### 顺序 vs 随机

```
rareq-sz / wareq-sz:  平均每个请求的大小
  - 接近 4KB    → 随机小 IO
  - 接近 128KB+ → 顺序大 IO
```

随机小 IO 的瓶颈是 IOPS，顺序大 IO 的瓶颈是带宽。修复方向不同。

> **口诀**：iostat 看 `await` 不看 `svctm`；看 `aqu-sz` 判断排队；看 `rareq-sz` 判断顺序还是随机。`%util = 100%` 在 NVMe 上不等于饱和。

### 2.7 free -h —— 内存

```bash
$ free -h
              total        used        free      shared  buff/cache   available
Mem:           62Gi        18Gi       2.5Gi       512Mi        41Gi        43Gi
Swap:         8.0Gi          0B       8.0Gi
```

#### available vs free

`free` 列是**严格意义的空闲内存**（没人用）。这个数字在 Linux 上**几乎永远很小** —— 因为内核会把所有空闲内存拿去做页缓存。

`available` 列是**"应用真正能用的内存"**，包括 free + 可回收的 buff/cache。这个才是判断"内存是否紧张"的指标。

阈值参考：
- `available > total × 20%`：放心
- `available < total × 10%`：警惕，看是不是有进程在涨
- `available < total × 5%`：危险，OOM Killer 可能介入

#### buff vs cache

| 项 | 含义 |
| --- | --- |
| `buff` | 块设备缓冲（裸读写、文件系统元数据） |
| `cache` | 文件页缓存（page cache，应用读过的文件内容） |

90% 情况下你不需要分清，看 `available` 就够。

#### shared 是共享内存

`tmpfs`、`/dev/shm` 这些占的内存计入 `shared`。Redis 共享内存大对象、PHP-FPM 等会让这一项很高。

#### swap 已用 = 0 不代表 swap 没动过

`free` 给的是当前快照。一个进程 swap out 后又 swap in，`Swap used` 可能回到 0，但中间发生过的延迟已经造成了。要看 `vmstat` 的 `si/so` 历史。

#### 真实场景

一个 64G 机器，`free` 显示 free = 200M，新人慌了。一看 `available` = 30G，buff/cache = 35G —— 完全没问题，是 ES / 数据库正常吃了页缓存。

反过来，free = 5G 看起来挺多，但 `available` = 5.2G，说明几乎所有内存都被应用真实占用了，离 OOM 一步之遥。

> **口诀**：**忘掉 `free` 列，看 `available`**。Linux 不会让内存"闲着"，但内存压力的真实指标是 available，不是 free。

### 2.8 sar -n DEV 1 —— 网络流量

```bash
$ sar -n DEV 1
14:35:01    IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
14:35:02    eth0   12500.00  10800.00  18432.00  12288.00      0.00      0.00      5.00     14.74
14:35:02      lo     350.00    350.00     45.00     45.00      0.00      0.00      0.00      0.00
```

#### 关键列

| 列 | 含义 |
| --- | --- |
| `rxpck/s, txpck/s` | 收 / 发包数 |
| `rxkB/s, txkB/s` | 收 / 发字节数（KB） |
| `%ifutil` | 接口利用率（基于网卡声明的速率） |

#### 算带宽是否打满

`%ifutil` 直接告诉你网卡占用比。但要小心：
- **千兆网卡**理论 1000 Mbps = 125 MB/s = 128000 KB/s
- 双向流量都算，所以 `%ifutil` = (rxkB/s + txkB/s) / 单向能力

某些云上的"千兆"是**共享带宽**，实际可能更低，要查实例规格。

#### 包速率也是瓶颈

不只是带宽。**小包情境下**（比如游戏、IoT），瓶颈往往是 PPS（每秒包数）而不是带宽。10 万 PPS 在某些网卡上就到上限了，但带宽看起来才几十 MB/s。

PPS 高时关注：
- `cat /proc/interrupts | grep eth` 看中断分布
- 是否单核处理网卡中断（看 `mpstat` 单核 %soft 高）
- RPS / RFS / RSS 是否打开

> **口诀**：网络瓶颈分两类，**带宽打满**看 `%ifutil`，**包速率打满**看 `rxpck/s` + 单核 `%soft`。

### 2.9 sar -n TCP,ETCP 1 —— TCP 统计

```bash
$ sar -n TCP,ETCP 1
14:38:01    active/s passive/s    iseg/s    oseg/s
14:38:02      120.00     35.00  18000.00  16500.00

14:38:01    atmptf/s  estres/s retrans/s isegerr/s   orsts/s
14:38:02        2.00     5.00     45.00      0.00     12.00
```

#### TCP 段（核心建连）

| 列 | 含义 |
| --- | --- |
| `active/s` | **主动**建连数（你 connect 别人） |
| `passive/s` | **被动**建连数（别人 connect 你） |
| `iseg/s` | 收到的 TCP 段 |
| `oseg/s` | 发送的 TCP 段 |

`active` 突然飙高 = 你的应用在大量发起新连接（多见于连接池配错、HTTP keep-alive 没开）。

#### ETCP 段（错误统计）

| 列 | 含义 | 严重性 |
| --- | --- | --- |
| `atmptf/s` | 建连失败（attempts fail） | 看是不是端口扫描或对端拒绝 |
| `estres/s` | 已建连接被 RST 重置 | 看应用是不是异常关连接 |
| **`retrans/s`** | **TCP 重传段数** | **网络质量指标**，> 0.1% 总流量 = 警报 |
| `isegerr/s` | 收到的错误段 | 极少见，硬件 / 链路问题 |
| `orsts/s` | 发出的 RST 段 | 应用主动重置（连接池超时常见） |

#### 重传率怎么算

```
重传率 = retrans/s ÷ oseg/s × 100%
```

健康值 < 0.1%，> 1% 就要排查。重传高意味着：
- 网络丢包（云上跨可用区、跨 region）
- 接收端处理不过来（zero window）
- 中间设备丢包

#### 连接队列也很关键

`sar -n TCP` 不显示，要单独看：

```bash
ss -ltn        # 看监听端口的 backlog 和当前 acceptq
netstat -s | grep -E "listen|overflowed|dropped"
```

`overflowed` / `dropped` 非 0 = SYN queue 或 accept queue 满了，新连接被丢。

> **口诀**：TCP 看 `retrans/s` 找网络质量，看 `estres/s` 找连接异常关闭，看 `passive/s` 是否突增找连接池配置问题。配合 `ss -ltn` 看连接队列。

### 2.10 top —— 综合兜底

```bash
$ top
top - 14:42:18 up 32 days,  4:30,  3 users,  load average: 8.21, 7.45, 5.03
Tasks: 412 total,   3 running, 409 sleeping,   0 stopped,   0 zombie
%Cpu(s): 72.5 us,  8.2 sy,  0.0 ni, 18.1 id,  0.5 wa,  0.0 hi,  0.7 si,  0.0 st
MiB Mem :  64384.0 total,   2540.5 free,  18432.2 used,  43411.3 buff/cache

   PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
 18934 app       20   0 8192.0m   4.2g  18.5m S 320.5   6.7  18:42.10 java
 19500 mysql     20   0 6144.0m   3.1g  12.0m S  85.2   4.9   4:21.50 mysqld
```

#### top 常用快捷键

| 键 | 作用 |
| --- | --- |
| `1` | 展开每核 CPU |
| `H` | 切换"线程模式"（看每个线程） |
| `M` | 按内存排序 |
| `P` | 按 CPU 排序（默认） |
| `T` | 按 CUMULATIVE TIME 排序 |
| `c` | 显示完整命令行 |
| `e` | 切换内存单位 |
| `f` | 选择显示哪些列（添加 nTH 看线程数） |
| `W` | 保存当前配置到 `~/.toprc` |

#### %CPU > 100%

`%CPU` 列是相对**单核**的，一个进程跑满 4 核 = 400%。这是为什么 `top` 里看到 `320.5%` 不奇怪 —— 该进程有 3-4 个线程在烧 CPU。

要看相对全机的占比，按 `Shift + I`（不规则切换）。

#### 反直觉点

`top` 的 CPU 列是**两次采样之间的平均**。如果你刚 `top` 进来看到 0%，是因为还没采样完。等几秒。

`VIRT` 是虚拟内存（包括 mmap 的文件、未实际使用的预留），**不是真实占用**。看 `RES`（resident）才是真实物理内存。

#### 相比 htop / btop

`top` 是兜底工具，所有 Linux 必有。`htop` 更友好（彩色、树形、易终止）。`btop` 更现代（图表）。生产排查推荐 `htop`，但脚本化 / 远程登录还是 `top`。

> **口诀**：`top` 是兜底确认。`1` 展开每核、`H` 看线程、`M` 排序内存、`c` 看完整命令是必练快捷键。

---

## 3. 实战剧本：5 个完整诊断过程

> 每个剧本是一个 60 秒走完的真实诊断流程。读这一节的姿势是：盖住命令输出后面的解读，自己先想一遍"下一步该敲什么"。

### 3.1 剧本 A：CPU 飙高

#### 告警内容

> 「prod-app-03 CPU 持续 > 90%，已超过 5 分钟」

#### 第 1 步：uptime + dmesg

```bash
$ uptime
 14:50:12 up 45 days,  load average: 18.21, 16.45, 12.03

$ dmesg --ctime | tail -5
# 没有异常
```

判断：load 18，机器是 8 核 → CPU 饱和；趋势上升（1m > 5m > 15m），事件正在发生；内核没报错，排除 OOM / 硬件问题。

#### 第 2 步：vmstat + mpstat 区分场景

```bash
$ vmstat 1 3
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
18  0      0  3.2G   240M  12.5G    0    0    20    80 12200 18500 88  8  3  1  0
20  0      0  3.2G   240M  12.5G    0    0    16    72 12500 18800 90  7  2  1  0
```

`r = 18~20`，确认 CPU 排队（远超 8 核）；`b = 0` 排除 IO 阻塞；`us = 88%`，是用户态烧 CPU。

```bash
$ mpstat -P ALL 1 2
14:52:01     CPU    %usr   %sys  %iowait    %idle
14:52:02     all   88.00   8.00    1.00     3.00
14:52:02       0   90.00   8.00    1.00     1.00
14:52:02       1   85.00   9.00    2.00     4.00
14:52:02       2   88.00   7.00    0.00     5.00
14:52:02       3   91.00   6.00    1.00     2.00
... (4-7 类似)
```

各核都在 85~91%，**不是单核瓶颈**，是真整体 CPU 紧张。

#### 第 3 步：pidstat 找进程

```bash
$ pidstat 1 3
14:53:01      UID       PID    %usr %system   %CPU   CPU  Command
14:53:02     1000    18934  580.00   45.00  625.00     0  java
14:53:02     1000    19102   12.00    3.00   15.00     1  redis-server
```

`java` 进程吃了 625% CPU（占了 8 核里的 6.25 核），元凶就是它。

#### 第 4 步：定位 Java 内的热点线程

```bash
# 看哪些线程在烧
$ pidstat -t -p 18934 1 2 | sort -k 8 -n -r | head
14:54:01      UID      TGID       TID    %usr %system   %CPU  Command
14:54:02     1000     18934     19120   95.00    2.00   97.00  GcThread-1
14:54:02     1000     18934     19121   95.00    2.00   97.00  GcThread-2
14:54:02     1000     18934     19122   90.00    2.00   92.00  GcThread-3
...

# TID = 19120 转十六进制
$ printf '%x\n' 19120
4ab0

# jstack 看栈
$ jstack 18934 | grep -A 20 'nid=0x4ab0'
"GcThread-1" prio=10 tid=0x... nid=0x4ab0 runnable
   ...内核态在做 GC
```

判断：所有热点线程都是 GC 线程，问题是 GC 风暴（参考 [JVM 调优与排查](/article/jvm/tuning)）。

#### 第 5 步：闭环

GC 风暴的根因可能是大对象、内存泄漏、堆不够大。下一步去看 GC 日志、jstat、堆 dump。这已经超出 60 秒诊断范围，但 60 秒诊断已经把方向**精确锁定**到了 JVM 层，节省了 10 分钟瞎找。

> **口诀**：CPU 飙高的 60 秒走法 = `uptime → vmstat → mpstat → pidstat → pidstat -t → jstack`。每步都是上一步的细化，**别跳级**。

### 3.2 剧本 B：上下文切换暴增

#### 告警

> 「prod-bg-01 cs 突增到 80 万/秒」

#### 第 1 步：vmstat 验证

```bash
$ vmstat 1 5
 r  b   swpd   free   buff  cache   si   so    bi    bo     in     cs  us sy id
 5  0      0  8.2G   240M   2.5G    0    0     8    40   85000 820000  35 45 20
```

确认 cs = 82 万/秒，sy = 45%（内核态高，符合大量上下文切换的特征）。

#### 第 2 步：pidstat -w 找哪个进程在切

```bash
$ pidstat -w 1 3 | sort -k 4 -n -r | head
15:00:01      UID       PID   cswch/s nvcswch/s  Command
15:00:02     1000    20100   180000.00  500000.00  worker
15:00:02     1000    18934    25000.00   12000.00  java
```

| 列 | 含义 |
| --- | --- |
| `cswch/s` | **自愿**切换：进程主动让出 CPU（等 IO / 等锁） |
| `nvcswch/s` | **非自愿**切换：被调度器抢占（CPU 不够 / 时间片用完） |

`worker` 进程 nvcswch/s = 50 万 —— 大量被抢占 = 它跑的线程数远超核数。

#### 第 3 步：看进程内线程数

```bash
$ ps -L -p 20100 | wc -l
8024
```

8000+ 线程，机器 8 核，平均每个核要调度 1000+ 线程，调度器拼命切换。

#### 第 4 步：定性

`worker` 是个老 PHP 项目，配置错把 worker 数从 50 改成 5000。改回 200 后 cs 立刻回落。

> **口诀**：cs 暴增 → `pidstat -w` 看进程，`nvcswch` 高 = 线程过多被抢占，`cswch` 高 = 锁竞争 / IO 等待。

### 3.3 剧本 C：内存压力

#### 告警

> 「prod-cache-02 available memory < 5%」

#### 第 1 步：free + vmstat

```bash
$ free -h
              total        used        free      shared  buff/cache   available
Mem:          126Gi       118Gi       1.2Gi       512Mi       6.8Gi       3.5Gi
Swap:         8.0Gi       4.2Gi       3.8Gi

$ vmstat 1 3
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa
 2  3  4288K  1.2G   240M   6.6G   85   320   120   480 5200 9800 25 18 50  7
```

警报全亮：available 3.5G、Swap used 4.2G、`si = 85, so = 320` 持续在 swap。

#### 第 2 步：找内存大户

```bash
$ ps aux --sort=-%mem | head
USER       PID %CPU %MEM    VSZ   RSS  COMMAND
redis    19102  3.5 75.0  100G   95G  redis-server
app      18934  2.0 15.0  20G    18G  java
```

Redis 吃了 95G 内存。这是单实例 Redis 配错 maxmemory 的典型现象。

#### 第 3 步：确认 OOM 风险

```bash
# 看每个进程的 oom_score（高 = 容易被杀）
$ for pid in $(ps -eo pid --no-headers); do
    score=$(cat /proc/$pid/oom_score 2>/dev/null)
    [ -n "$score" ] && echo "$score $pid $(cat /proc/$pid/comm 2>/dev/null)"
  done | sort -nr | head
1820 19102 redis-server
540  18934 java
```

Redis oom_score 最高 = 下一次 OOM 必杀它。

#### 第 4 步：紧急动作

```bash
# 紧急释放页缓存（治标）
echo 3 > /proc/sys/vm/drop_caches

# 调小 Redis 的 maxmemory（治本）
redis-cli config set maxmemory 80gb
redis-cli config set maxmemory-policy allkeys-lru
```

> **口诀**：内存压力 = `free` 看 available + `vmstat` 看 si/so + `ps --sort=-%mem` 找元凶 + `/proc/PID/oom_score` 预判 OOM 受害者。

### 3.4 剧本 D：IO 瓶颈

#### 告警

> 「DB 接口 P99 从 50ms 飙到 800ms」

#### 第 1 步：vmstat 看 b 列

```bash
$ vmstat 1 3
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa
 2  8      0  4.2G   240M   8.5G    0    0  2400 18500 8200 12000  10  5 35 50
```

`b = 8`、`wa = 50%`，确定是 IO 阻塞。

#### 第 2 步：iostat 找盘

```bash
$ iostat -xz 1 2
Device  r/s    w/s    rkB/s   wkB/s   r_await w_await aqu-sz  %util
sda     5.00   1200.0  80.00   72000.0  2.00   180.50  220.5   100.0
```

`sda` 写入 70 MB/s，w_await = 180ms，aqu-sz = 220 —— 大量 IO 在排队。

#### 第 3 步：pidstat -d 找进程

```bash
$ pidstat -d 1 3
15:20:01      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
15:20:02       27    19500     200.00  68000.00      0.00    1820  mysqld
```

mysqld 在写 68 MB/s。

#### 第 4 步：判断顺序还是随机

```bash
$ iostat -xz 1 1 | grep sda
Device  r/s    w/s    rkB/s   wkB/s   ... wareq-sz ... %util
sda     5.00   1200.0  80.00   72000.0 ... 60.00    ... 100.0
```

`wareq-sz = 60 KB`（介于随机 4K 和顺序 128K 之间）→ MySQL 的 redo log + 数据页混合写入。

#### 第 5 步：闭环

看 MySQL 是不是有大事务在 commit，看 `innodb_io_capacity` 和 `innodb_flush_log_at_trx_commit` 配置。如果 SSD 还出现 180ms 延迟，硬件可能也出问题，看 `smartctl -a /dev/sda`。

> **口诀**：IO 瓶颈 = `vmstat b/wa → iostat -xz → pidstat -d → iostat 看 wareq-sz 判顺序/随机`。

### 3.5 剧本 E：网络异常

#### 告警

> 「跨可用区调用大量超时」

#### 第 1 步：sar -n DEV 看流量

```bash
$ sar -n DEV 1 3
14:42:01    IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   %ifutil
14:42:02    eth0   18000.00  17500.00  22000.00  20000.00      18.5
```

带宽 18.5%，远没打满，**不是带宽问题**。

#### 第 2 步：sar -n TCP,ETCP 看错误

```bash
$ sar -n TCP,ETCP 1 3
14:43:01    active/s passive/s    iseg/s    oseg/s
14:43:02      350.00    180.00  18000.00  17500.00

14:43:01    atmptf/s  estres/s retrans/s isegerr/s   orsts/s
14:43:02       18.00     45.00    420.00      0.00     85.00
```

`retrans/s = 420`，重传率 = 420 / 17500 = **2.4%**，远超 0.1% 阈值。
`estres/s = 45`：建立的连接被 RST 重置。

#### 第 3 步：定位是哪个对端

```bash
# 看连接状态分布
$ ss -tan state established | awk '{print $4}' | sort | uniq -c | sort -nr | head
   850 10.10.1.5:8080
   420 10.20.3.8:3306
   180 10.10.1.6:8080

# 看具体哪些连接重传高
$ ss -ti dst 10.20.3.8 | grep -A 1 ESTAB | grep retrans
   ... retrans:0/12 ...
   ... retrans:3/45 ...
```

10.20.3.8 是另一可用区的 MySQL，重传集中在这里。

#### 第 4 步：闭环

`ping` + `mtr` 验证网络质量：

```bash
$ mtr -rwbzc 100 10.20.3.8
HOST: ...
  1.|-- gw01    0.0%   100    0.5  ...
  2.|-- az-router  3.0%   100   2.5  ...   ← 中间设备丢包 3%
  3.|-- 10.20.3.8  2.0%   100   8.5  ...
```

跨可用区路由器丢包，工单云厂商。同时应用侧加重试 + 超时降级。

> **口诀**：网络问题 = `sar -n DEV` 看带宽 → `sar -n ETCP` 看重传 → `ss -ti` 找具体对端 → `mtr` 定位丢包位置。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① "sysadmin 提示 CPU 高"，本质是应用 Bug

#### 现象

运维群里：「prod-api-05 CPU 100% 持续 30 分钟，要不要重启？」值班同学习惯性 `top` 看一眼，确认 java 进程 800% CPU，准备重启。

#### 根因

冷静下来按 60 秒走完：
- `mpstat`：8 核全部 95%+ → 真整体 CPU 紧张
- `pidstat -t -p PID`：定位到一个叫 `report-thread-3` 的线程持续 99%
- `jstack` 看栈：在跑一个 `for (i = 0; i < list.size(); i++)` 的循环，list 是 `ArrayList`

代码：

```java
List<Long> list = service.getAllOrderIds();   // 返回 ~200 万条
for (int i = 0; i < list.size(); i++) {
    if (list.contains(target)) { ... }        // ★ contains 是 O(n)
}
```

`ArrayList.contains` 是线性扫描。200 万 × 200 万 = 4 万亿次比较，单线程死循环式烧 CPU。

#### 修复

```java
Set<Long> set = new HashSet<>(list);   // 转成 HashSet
for (Long id : list) {
    if (set.contains(target)) { ... }   // O(1)
}
```

#### 验证

```bash
# 修复前
$ pidstat -t -p <PID> 1 5 | grep report-thread
... 99.0 ...

# 修复后
$ pidstat -t -p <PID> 1 5 | grep report-thread
... 5.0 ...
```

#### 教训

**重启是逃避，不是解决**。用 60 秒诊断花 5 分钟定位到代码层的 O(n²)，比重启 + 一周后再次发生再排查省得多。

> **口诀**：CPU 高的告警，**先沿 60 秒走到代码层**再决定要不要重启。重启只能掩盖问题。

### 案例 ② 宿主与容器视图差异

#### 现象

K8s Pod 内的 Java 应用频繁 OOMKilled。`free -h` 在容器内看显示有 30G 可用，但容器 limit 只有 4G。开发问："明明还有内存，为什么杀我？"

#### 根因

Linux 容器化是用 cgroups 限制，**但 `/proc/meminfo` 仍然展示宿主机视图**（除非启用了 `lxcfs` 之类的代理）。

后果：
- 容器内 `free -h` 看到的是宿主的 32G 内存
- JVM 通过 `Runtime.getRuntime().maxMemory()` 默认按宿主内存推断堆大小（老版本 JDK）
- 实际 cgroup limit 4G，JVM 用到 5G → 被 OOMKiller 杀

JDK 8u191 之后默认 `-XX:+UseContainerSupport` 才能正确读 cgroup，老版本要手动加。

#### 修复

**JVM 侧**：

```bash
# JDK 8u191+
java -XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 ...

# 或者直接指定堆
java -Xmx3g ...
```

**容器内查看真实视图**：

```bash
# memory limit
cat /sys/fs/cgroup/memory/memory.limit_in_bytes

# memory usage（cgroup v1）
cat /sys/fs/cgroup/memory/memory.usage_in_bytes

# cgroup v2
cat /sys/fs/cgroup/memory.max
cat /sys/fs/cgroup/memory.current

# CPU 限制
cat /sys/fs/cgroup/cpu/cpu.cfs_quota_us
cat /sys/fs/cgroup/cpu/cpu.cfs_period_us
# 实际可用 CPU = quota / period
```

更友好的工具：[lxcfs](https://github.com/lxc/lxcfs) 把 `/proc/meminfo` 等代理成 cgroup 视图。但 K8s 默认不装。

#### 验证

```bash
# 容器内
$ java -XX:+PrintFlagsFinal -version | grep MaxHeapSize
# 看到的应该 ≈ cgroup limit × 75%
```

#### 教训

**容器里看到的 `top` / `free` / `mpstat` 都是骗人的**（除非用 lxcfs）。诊断容器化应用时，**直接看 cgroup 文件**才是真实视图。

> **口诀**：容器排查别信 `free` / `top`，看 `/sys/fs/cgroup/...` 才是真。JVM 加 `-XX:+UseContainerSupport`。

### 案例 ③ load average 高，CPU 却低

#### 现象

「prod-nfs-client load 80，用户反馈卡」。一看 CPU 各核都 < 20%，"机器没事啊？"

#### 根因

```bash
$ uptime
 load average: 80.21, 75.45, 60.03

$ vmstat 1 3
 r  b   ...   in   cs us sy id wa st
 2 75      0  500  800  5  3 30 62  0
```

`b = 75`、`wa = 62%` —— 75 个进程卡在 D 状态等 IO，CPU 几乎闲着。

```bash
$ ps aux | awk '$8 == "D" {print}' | head
USER       PID %CPU %MEM  ... STAT  COMMAND
app      19100  0.0  0.5  ... D     /usr/bin/python /opt/app/sync.py
app      19101  0.0  0.5  ... D     /usr/bin/python /opt/app/sync.py
... (75 个类似的)

$ cat /proc/19100/wchan
nfs_wait_on_request
```

NFS 卡了。所有 worker 都在等 NFS 响应。

#### 修复

短期：杀 hang 住的 NFS 挂载、改用本地缓存。
长期：去掉对 NFS 的同步依赖，加超时（NFS 默认 hard mount 是无超时的，必须改 soft + timeo）。

```bash
# /etc/fstab
nfs-server:/share /mnt/nfs nfs soft,timeo=50,retrans=3,bg 0 0
```

#### 验证

```bash
# 找所有 D 状态进程
ps aux | awk '$8 ~ /D/ {print}'

# 持续监控 b 列
vmstat 1 | awk '{print $2}'
```

#### 教训

**load 包含 D 状态进程**，不要看到 load 高就以为是 CPU 不够。**先看 `r` 还是 `b` 撑起来的**。

> **口诀**：load = r + b。`b` 高 → 找 D 状态进程的 wchan，多半是 IO/NFS/磁盘 hung。

### 案例 ④ vmstat r 列高 —— 真 CPU 不够

#### 现象

「服务从 4 核机迁到 2 核机后，QPS 没变但 P99 翻倍」。CPU 看起来才 60%，"应该够吧？"

#### 根因

```bash
$ vmstat 1 3
 r  b   ...   us sy id
 6  0      0  55  8 37
 7  0      0  58  7 35
```

`r = 6~7` 在 2 核机器上 = 4-5 个进程在排队等 CPU。

虽然 CPU 利用率 65% 看起来"够"，但**调度延迟**（任务从 runnable 到真正运行的等待时间）已经显著上升。

理论支持：M/M/1 排队模型告诉我们，**CPU 利用率 70% 时延迟翻倍，80% 时延迟翻 4 倍**。所以"CPU 60% 够用"是错觉。

#### 修复

加机器 / 扩核 / 优化代码减少 CPU 时间。

更高级：用 `pressure-stall information`（PSI，Linux 4.20+）看真正的 CPU 压力：

```bash
$ cat /proc/pressure/cpu
some avg10=15.20 avg60=12.50 avg300=8.30 total=...
full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

`some avg10 = 15%` 表示过去 10 秒里 15% 的时间至少有 1 个进程在等 CPU。这个指标比"CPU 利用率"敏感得多。

#### 验证

加核后 `r` 列应回落到 ≤ 核数，PSI 的 `some` 应回落到 < 5%。

> **口诀**：CPU 利用率 70% 不代表"够"。看 `vmstat r` 和 `/proc/pressure/cpu` 才是真饱和度。

### 案例 ⑤ pidstat 找不到罪魁 —— 瞬时进程

#### 现象

`top` 看 CPU 偶发飙到 80%，几秒后回落。`pidstat 1` 怎么也抓不到那个进程。

#### 根因

瞬时进程：cron 任务、定时脚本、shell 子进程。它们存活几秒，pidstat 1 秒采样可能完全错过。

#### 修复 / 排查

**方法 A：用 BCC 的 execsnoop**

```bash
$ sudo execsnoop-bpfcc
PCOMM            PID    PPID   RET ARGS
sh               19200  1      0   /bin/sh -c /opt/scripts/cleanup.sh
gzip             19201  19200  0   /bin/gzip -f /var/log/big.log
```

每个 exec() 系统调用都被捕获，**无论进程多短命**。

**方法 B：perf record**

```bash
$ sudo perf record -e sched:sched_process_exec -a sleep 30
$ sudo perf script | head
sh 19200 [003] 1234567.890123: sched:sched_process_exec: filename=/bin/sh ...
```

**方法 C：atop 长期监控**

```bash
# 每 10 秒记录所有进程（包括短命的）
atop -w /var/log/atop/atop.log 10

# 事后回放
atop -r /var/log/atop/atop.log
```

#### 验证

execsnoop 捕获到 cleanup.sh 后，看 crontab 找到对应任务，确认是它在烧 CPU。

#### 教训

**pidstat 是采样的，瞬时进程是它的盲区**。BCC `execsnoop` 是 60 秒诊断的"高级补充"，所有生产机器建议常驻 atop。

> **口诀**：找不到进程？用 `execsnoop`（BCC）或 `perf sched:sched_process_exec` 抓所有 exec。

### 案例 ⑥ iostat await 高，但应用响应正常

#### 现象

监控告警 "sda await 200ms"，DBA 急着排查 MySQL，结果业务说"接口正常啊"。

#### 根因

`iostat` 的 `await` 是**该设备所有 IO 的平均等待时间**，不是某个进程的。

实际情况：
- mysqld 在做小量 commit，IOPS 低、await 几 ms
- 一个备份脚本在 `dd` 大文件，单个 IO 等待长
- 平均下来 await = 200ms

业务感受到的是 mysqld 的几 ms，不是平均的 200ms。

#### 排查

```bash
# 看每个进程的 IO 行为
$ pidstat -d 1 5
... 看到一个 dd 进程在 100MB/s 写

# 进一步看每个 IO 的延迟（BCC biolatency）
$ sudo biolatency-bpfcc -D 10
disk = 'sda'
   usecs           : count    distribution
   0 -> 1          : 0        |                       |
   1 -> 3          : 0        |                       |
   ...
   8192 -> 16383   : 12       |******                 |
   16384 -> 32767  : 5        |**                     |
```

可以按进程过滤：

```bash
$ sudo biosnoop-bpfcc | head
TIME(s)      COMM           PID    DISK    T  SECTOR    BYTES  LAT(ms)
0.000000     dd             19500  sda     W  10485760  4096   180.5
0.000050     mysqld         18934  sda     W  20480     4096   2.1
```

mysqld 的 IO 真实延迟 = 2.1ms，跟 `iostat` 的 200ms 平均完全是两回事。

#### 修复

把备份任务挪到独立磁盘 / 限速：

```bash
# 用 ionice 降低备份进程的 IO 优先级
ionice -c 3 -p <backup_pid>

# 或者用 cgroup blkio 限制
echo "8:0 1048576" > /sys/fs/cgroup/blkio/backup/blkio.throttle.write_bps_device
```

#### 验证

mysqld 的 P99 不变，备份变慢但能跑完。

> **口诀**：`iostat await` 是设备级平均，不代表你关心的进程。**用 BCC `biolatency` / `biosnoop` 看进程级 IO 延迟**。

### 案例 ⑦ 误读 free 的 buff/cache

#### 现象

新人排查："这台机器 free 只有 200M，肯定要 OOM 了！"

#### 根因

```bash
$ free -h
              total        used        free      shared  buff/cache   available
Mem:           62Gi        18Gi       200Mi       512Mi        43Gi        43Gi
```

`free = 200M`、`available = 43G`。新人只看 `free` 列，没看 `available`。

Linux 设计上**"空闲内存就是浪费"**，所有不用的内存都拿去做页缓存（cache）。这部分内存**应用申请时立刻可回收**，不是被占用。

#### 验证

```bash
# 强制清掉页缓存
$ sync && echo 3 > /proc/sys/vm/drop_caches

$ free -h
              total        used        free      shared  buff/cache   available
Mem:           62Gi        18Gi        43Gi       512Mi       1.0Gi        43Gi
```

果然，"free" 一下变成 43G。**这是个验证操作，生产慎用**：清掉页缓存后所有读请求都要重新走磁盘，应用 P99 会暴涨。

#### 修复

不需要修复，是认知问题。培训时反复强调：
- **判断内存压力看 `available`**，不看 `free`
- `available < 总内存 10%` 才警惕
- 不要手动 `drop_caches`（除非你真的想清掉）

#### 教训

`drop_caches` 是把双刃剑。Brendan Gregg 写过一篇文章专门讲："如果你的运维同学告诉你他每天 drop_caches 一次提升性能 —— 立刻阻止他"。

> **口诀**：**忘掉 `free` 列**。判断内存就看 `available`。drop_caches 是诊断工具，不是优化手段。

### 案例 ⑧ sar 历史数据缺失

#### 现象

凌晨 3 点服务出问题，早上 9 点排查时想看凌晨 3 点的 CPU / IO 历史，`sar` 报错：

```bash
$ sar -u -f /var/log/sa/sa15
Cannot open /var/log/sa/sa15: No such file or directory
```

#### 根因

`sar` 的历史数据来自 `sysstat` 包的 cron 任务（每 10 分钟跑 `sa1` 写 `/var/log/sa/saXX`）。

很多最小化镜像 / 容器**默认不装 sysstat**。出事后才发现没数据。

```bash
# 检查
$ rpm -q sysstat       # CentOS / RHEL
$ dpkg -l sysstat       # Ubuntu / Debian
$ cat /etc/cron.d/sysstat
```

#### 修复

```bash
# 装 sysstat
$ yum install sysstat   # 或 apt install sysstat

# 启用并打开采样
$ sed -i 's/false/true/' /etc/default/sysstat   # Debian
$ systemctl enable --now sysstat

# 验证
$ ls /var/log/sa/
sa01  sa02  sa03  ...
```

默认采样间隔是 10 分钟，对突发故障**粒度太粗**。改成 1 分钟：

```bash
$ vim /etc/cron.d/sysstat
# 把
*/10 * * * * root /usr/lib64/sa/sa1 1 1
# 改成
* * * * * root /usr/lib64/sa/sa1 60 1
```

#### 没装 sysstat 怎么救？

事后救不回来。**未来**可以装 Prometheus + node_exporter，所有指标都进时序库。

或者用 `atop`（默认每 10 分钟自动滚动 + 写日志）：

```bash
# 看历史
$ atop -r /var/log/atop/atop_20240315 -b 03:00 -e 03:30
```

#### 验证

故障再发时，`sar -u -f /var/log/sa/saXX -s 03:00:00 -e 03:30:00` 能回放当时数据。

> **口诀**：所有生产机器**必须装 sysstat / atop / node_exporter**之一，否则事后排查只能瞎猜。

### 案例 ⑨ top 看 CPU 100% 但 mpstat 各核都不忙

#### 现象

Python 应用 `top` 显示 100% CPU，但 8 核机器 `mpstat -P ALL` 各核才 12.5% 左右。"哪儿出问题了？"

#### 根因

CPython 的 GIL（Global Interpreter Lock）：**同一时刻只有一个线程能跑 Python 字节码**。即使你开 100 个线程，整体 CPU 利用率上限是单核 = 100%（即 8 核机器上的 12.5%）。

`top` 显示的 100% 是**单进程相对单核**的占用。这个进程实际只用了 1 个核。

类似的"看起来全用、实际单核"的场景：
- Python（CPython）GIL
- Node.js 主线程（不开 cluster / worker_threads）
- 老版本 Redis 单线程
- 某些用 `select()` 而非 `epoll()` 的老代码

#### 验证

```bash
# pidstat -t 看线程，会发现只有一个线程在烧
$ pidstat -t -p <PID> 1
... TID=...  %CPU=98.5  Command=python
... TID=...  %CPU=0.5   Command=python
... TID=...  %CPU=0.5   Command=python
```

#### 修复

| 场景 | 方案 |
| --- | --- |
| Python CPU 密集 | multiprocessing 或 Cython / C 扩展（释放 GIL） |
| Python IO 密集 | asyncio 或 gevent |
| Node.js | cluster 模块 / PM2 多实例 |
| Redis 6+ | 开启 `io-threads` |

#### 教训

**单核瓶颈是扩核解决不了的**。看到 `top` CPU 100% 第一反应不是"加机器"，是 `mpstat -P ALL` 看是不是单核。

> **口诀**：`top` 单进程 100%，必须 `mpstat` 验证是真饱和还是单核瓶颈。**单线程瓶颈扩核没用**，要拆进程或异步化。

---

## 5. 上线检查清单

打印贴墙 / 加进 SRE 应急手册。

### 工具准备
- [ ] 所有生产机器装 `sysstat`（或 atop）并开启自动采样
- [ ] 装 `bcc-tools`（CentOS）/ `bpfcc-tools`（Ubuntu）方便用 BPF 工具
- [ ] 装 `htop` / `iotop` / `iftop` / `nethogs`
- [ ] node_exporter + Prometheus 接入
- [ ] dmesg 改成 `--ctime` 默认（alias）

### 60 秒诊断快捷化
- [ ] 把 60 秒诊断 10 条命令封装成一个脚本（如 `60s.sh`）
- [ ] 脚本输出带时间戳，落盘到 `/tmp/diag-$(date +%s).log`
- [ ] 应急手册第一条写"先跑 `60s.sh`"
- [ ] 团队定期演练（每月至少 1 次故障演练）

### 容器特殊准备
- [ ] 容器镜像里装上 `procps` / `iputils` / `iproute2` 等基础工具
- [ ] 知道直接看 `/sys/fs/cgroup/...` 的路径（cgroup v1 vs v2 不同）
- [ ] JVM 加 `-XX:+UseContainerSupport`（JDK 8u191+）
- [ ] 应用日志带 `hostname` + `pod ip`，便于关联

### 告警分类
- [ ] CPU 告警：区分整体 CPU、单核 CPU、`load average`、`r` 列
- [ ] 内存告警：用 `available`，不是 `free`
- [ ] IO 告警：用 `await` + `aqu-sz`，不要再看 `svctm`
- [ ] 网络告警：分带宽、PPS、重传率三个维度

### 反应速度
- [ ] SSH 进机器 30 秒内开始第一条命令（提前演练肌肉记忆）
- [ ] **不要重启机器**，先按 60 秒走完
- [ ] 排查日志用 `tee` 落盘，避免事后想看找不到
- [ ] 重大故障期间多开 tmux 窗口并行排查

### 反模式（不要做）
- [ ] 不要 `drop_caches` "清缓存优化"
- [ ] 不要在生产 `strace -p` 长时间跟踪（会让进程慢 10 倍）
- [ ] 不要随便重启来"看看会不会好"
- [ ] 不要看到 load 高就加机器
- [ ] 不要在容器内信 `top` / `free` 的输出
- [ ] 不要忽视 `dmesg` 里的内核警告

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### Brendan Gregg 的体系
- [《Systems Performance》第二版](https://www.brendangregg.com/systems-performance-2nd-edition-book.html) —— 性能工程圣经
- 他的博客 [brendangregg.com](https://www.brendangregg.com/) 是免费的金矿
- USE 方法、热图、火焰图三大方法论的原文

### eBPF 工具集
- BCC 工具集：`execsnoop` / `opensnoop` / `biolatency` / `biosnoop` / `tcplife` / `runqlat`
- bpftrace 一行命令脚本，比 BCC 更简洁
- libbpf-tools（Cilium 维护）—— 不依赖 Python，更轻
- 配合阅读 [BPF Performance Tools](https://www.brendangregg.com/bpf-performance-tools-book.html)

### perf 高级用法
- `perf record / perf report / perf top` 基础三件套
- `perf sched` 调度延迟分析
- `perf lock` 锁竞争分析
- `perf c2c` cache 行竞争（伪共享）
- 配合 [火焰图实战](/article/linux-perf/flame-graph)

### off-CPU 分析
- Brendan Gregg 提出的概念：除了"在 CPU 上跑"，还要看"为什么不在 CPU 上"
- 工具：`offcputime`（BCC）、`runqlat`、`wakeuptime`
- 解决"系统看起来空闲，应用还是慢"的关键

### 容器内的特殊性
- cgroup v1 vs v2 的字段差异
- `lxcfs` 如何把 `/proc` 代理成 cgroup 视图
- K8s `kubectl top` 与节点 `top` 的差异
- container-aware 工具：`crictl`、`nerdctl`
- Pod 排查的标准流程：`describe` → `logs` → `exec` → `debug ephemeral container`

### 云上的盲点
- AWS CloudWatch / 阿里云监控的采样间隔是 1 分钟，**短于 1 分钟的尖刺看不到**
- 云盘的 IOPS / 带宽是分级配置的，要看实例规格
- 跨可用区延迟 vs 同可用区延迟差一个数量级
- spot / preemptible 实例的"被回收"在 dmesg 里有迹可循
- 弹性网卡的多队列要手动开（`ethtool -L`）

### SRE 应急响应手册
- Google SRE Book 第 14 章 "Managing Incidents"
- ICS（Incident Command System）应急指挥体系
- Postmortem 模板（无指责、根因分析）
- Chaos Engineering：故意制造故障验证响应能力

### Linux 内核可观测性
- `/proc/pressure/{cpu,memory,io}` —— PSI（Linux 4.20+）的真饱和度
- `ftrace` —— 内核级追踪
- `trace-cmd` / `kernelshark` GUI 分析
- Latency Tracer：定位内核长延迟的源头

---

## 7. 引用

- Brendan Gregg, *Systems Performance: Enterprise and the Cloud*, 2nd Edition, 2020
- Brendan Gregg, [Linux Performance Analysis in 60,000 Milliseconds](https://netflixtechblog.com/linux-performance-analysis-in-60-000-milliseconds-accc10403c55), Netflix Tech Blog, 2015
- Brendan Gregg, *BPF Performance Tools*, 2019
- Linux 内核文档：[Documentation/admin-guide/pm/working-state.rst](https://www.kernel.org/doc/html/latest/)
- sysstat 文档：[github.com/sysstat/sysstat](https://github.com/sysstat/sysstat)
- BCC tools：[github.com/iovisor/bcc](https://github.com/iovisor/bcc)
- bpftrace：[github.com/bpftrace/bpftrace](https://github.com/bpftrace/bpftrace)
- Google SRE Book：[sre.google/sre-book](https://sre.google/sre-book/table-of-contents/)
- Linux 内核 PSI 提案：[lwn.net/Articles/759781/](https://lwn.net/Articles/759781/)

---

> 配套：[Linux 与性能工程主题](/article/linux-perf/) | [火焰图实战](/article/linux-perf/flame-graph) | [JVM 调优与排查](/article/jvm/tuning)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你觉得这个深度对，告诉我下一篇该写哪个。
