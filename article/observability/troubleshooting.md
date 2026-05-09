# 故障定位方法论 —— "5 分钟定位 P99 飙升"的工程剧本

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，去 [可观测性主题](/article/observability/) 那一节就够了。
> 这里要回答的，是**告警响起、值班群炸锅、领导在线问"多久能恢复"**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. 告警来了 P99 从 50ms 飙到 2s，你第一步看哪张图？为什么不应该一上来就翻代码？
2. USE（Utilization / Saturation / Errors）方法论怎么"反向溯源"？给一个完整的演练步骤。
3. CPU 100% 不一定是业务问题。怎么用 `jstack + top -H -p <pid>` 一分钟定位是 GC 还是业务？
4. 一个线程在 jstack 里显示 RUNNABLE，但实际上"卡住"了，可能在干什么？
5. 火焰图里 `epoll_wait` 占了 60% 栈高，是问题吗？
6. 监控里 DB 慢，应用层 trace 也显示慢 SQL —— 真因一定是 SQL 慢吗？给两种可能的"伪 DB 慢"。
7. 同一个故障，A 同学定位 5 分钟，B 同学定位 2 小时，差的不是工具熟练度，差的是什么？
8. 修复后告警没了，你就敢说"修好了"吗？怎么验证"根因"而不是"触发因素"？

如果有半数答不上，往下看。

---

## 1. 底层模型：故障定位是一场"反向推理"

绝大多数人对故障定位的认知止于"看监控、看日志、抓堆栈"。这层认知能解决 50% 的简单问题，剩下 50% 的复杂故障会让你定位 2 小时以上。要把定位时间压到 5 分钟以内，必须先把"方法论"刻进肌肉记忆。

### 1.1 性能分析方法论：USE / RED / Four Golden Signals

业界三大经典方法论，覆盖不同视角：

| 方法 | 提出者 | 视角 | 核心指标 |
| --- | --- | --- | --- |
| **USE** | Brendan Gregg | 资源（机器/容器） | Utilization（利用率）/ Saturation（饱和度）/ Errors（错误数） |
| **RED** | Tom Wilkie | 服务（请求） | Rate（QPS）/ Errors（错误率）/ Duration（延迟） |
| **Four Golden Signals** | Google SRE | 用户体验 | Latency / Traffic / Errors / Saturation |

它们不是互斥的。**USE 看资源、RED 看服务、Four Golden Signals 看用户**。一次完整的定位通常要在三层之间来回跳。

### 1.2 USE 方法的"反向溯源"

USE 最强的地方是**穷举法**：把所有资源（CPU / 内存 / 磁盘 / 网络 / 文件描述符 / 线程 / 锁）按 U/S/E 三个维度过一遍，**总有一个先报警**。

实战流程（默记）：

```
第一步：列出所有资源
   CPU / Memory / Disk IO / Network / FD / Thread / Lock / GC

第二步：每个资源问三个问题
   - 利用率高吗？     (Utilization)
   - 队列在排队吗？   (Saturation)
   - 有错误吗？       (Errors)

第三步：从"先报警"的资源往下追
   利用率 100% → 看是谁在用
   饱和度高     → 看队列里堆的什么
   错误数飙升   → 看错误的具体类型
```

为什么强调"反向"？正向思维是"我猜是 X 问题"，反向思维是"哪里最先报警"。**正向容易先入为主，反向不会漏。**

### 1.3 故障传播链路

一个用户请求会经过多层。哪层"先慢"，决定了你该从哪开始查：

```
用户感知层    用户报"打不开"
   │
   ▼
入口层        CDN / WAF / SLB / Nginx
   │
   ▼
网关层        API Gateway / 服务网格
   │
   ▼
应用层        Java / Go 进程
   │
   ▼
中间件层      Redis / MQ / DB Proxy
   │
   ▼
存储层        MySQL / ES / 对象存储
   │
   ▼
基础设施层    K8s / 网络 / 磁盘 / 内核
```

**铁律一**：故障**自下而上**传播（基础设施抖动 → 存储慢 → 应用慢 → 用户感知）。
**铁律二**：定位**自上而下**进行（先看用户体验、再看应用层、再下钻）。

混了顺序就会"用户已经骂街，你还在看磁盘"。

### 1.4 因果链常见模式

把生产里反复出现的因果链刻在脑子里，一看现象就能猜根因：

| 现象 | 常见因果链 |
| --- | --- |
| **P99 飙升、QPS 不变** | GC → STW（Stop The World）→ 所有请求被冻结 |
| **P99 飙升、QPS 也升** | 上游重试风暴 → 雪崩 |
| **P99 飙升、CPU 不高** | DB 慢 / 下游慢 / 锁等待 → 线程被挂起 |
| **CPU 100%、但业务没变** | 反射 / 序列化 / GC 线程 / 死循环 |
| **OOM、堆没满** | 元空间 / 直接内存 / Native 栈泄漏 |
| **5xx 突增、机器健康** | 连接池满 / FD 用完 / 线程池拒绝 |
| **间歇性 5xx** | 容器漂移 / NAT 超时 / DNS 抖动 |
| **网络抖动、应用日志正常** | 内核重传 / TCP 队列满 / 网卡丢包 |

这张表配合 USE 方法用：**先看哪个资源饱和，再套对应的因果链**。

### 1.5 5 分钟定位的"标准动作"

把"前 5 分钟"切成**60 秒五段**，每段都有明确的动作，不要乱看：

```
0-60s：对齐现象
   - 是 P99 还是 P50？
   - 是全量还是某个接口？
   - 是某台机器还是全集群？
   - 几点开始？此前有发布/变更吗？

60-120s：定位资源饱和点
   - 看 CPU / Memory / GC / 线程池 / 连接池
   - 看 DB 慢查询、Redis 慢命令、MQ 堆积
   - 看下游服务的 P99 对比

120-180s：抓现场
   - 应用层：jstack / arthas thread / async-profiler
   - 系统层：top -H / iostat / vmstat
   - 网络层：netstat / ss / 流量图
   - DB 层：show processlist / pt-query-digest

180-240s：验证假设
   - 找一个最可疑的根因
   - 用更精细的工具去**证伪**或**证实**
   - 千万别在这一步开始改代码

240-300s：止损
   - 如果根因清楚 → 修复或回滚
   - 如果根因不清楚 → 先扩容/熔断/限流
   - 然后再下一个 5 分钟继续追
```

记住：**止损优先于查根因**。先把用户体验救回来，再慢慢找原因。

### 1.6 根因（root cause）vs 触发因素（trigger）

这是最容易混的概念，但分不清就会"修了表象、过几天复发"。

举个例子：你的服务 OOM 了。

- **触发因素**：流量峰值、某个大请求、某次发布
- **真根因**：内存泄漏、不合理的缓存上限、ThreadLocal 没清理

**铁律**：**修复完要问自己 —— 如果触发因素再来一次，会不会再出问题？**

| 维度 | 触发因素 | 真根因 |
| --- | --- | --- |
| **特征** | 偶然、外部 | 必然、设计 |
| **修复方式** | 回滚、限流、扩容 | 改代码、改架构 |
| **复发可能** | 再有触发就再发 | 设计不改就持续 |

很多团队定位故障停留在"找到触发因素"层面，于是同一个根因换个触发因素就再爆一次。这就是为什么有的团队一年发生 8 次"同样"的故障。

> **口诀**：**USE 反向溯源、5 分钟标准动作、止损优先于查根因、根因不是触发因素**。这四条记牢，故障定位就有了骨架。

---

## 2. 工具切片：你必须熟练到肌肉记忆的 7 把刀

每个工具一段命令、一段说明、一段"什么时候用 / 什么时候不用"。命令完整可复制，不要拼凑。

### 2.1 jcmd 系列：JVM 进程级诊断

`jcmd` 是 JDK 自带的"瑞士军刀"，所有 JVM 都有，第一手现场抓取首选。

```bash
# 列出所有 Java 进程
jcmd

# 看 JVM 配置（看是不是参数没生效）
jcmd <pid> VM.flags
jcmd <pid> VM.system_properties

# 抓 thread dump（jstack 的现代替代）
jcmd <pid> Thread.print > thread.dump

# 抓堆 histogram（不抓全堆，只看类直方图，秒级）
jcmd <pid> GC.class_histogram | head -30

# 抓完整堆 dump（线上慎用，会 STW 几秒到几十秒）
jcmd <pid> GC.heap_dump /tmp/heap.hprof

# GC 信息
jcmd <pid> GC.heap_info

# JFR：飞行记录器（最强大）
jcmd <pid> JFR.start name=diag duration=60s filename=/tmp/diag.jfr
jcmd <pid> JFR.dump name=diag filename=/tmp/diag.jfr
jcmd <pid> JFR.stop name=diag
```

**什么时候用**：第一手现场抓取、配合脚本批量收集、不需要装额外工具。
**什么时候不用**：需要"实时观察"某个方法的入参/返回值（用 arthas）。

### 2.2 arthas：在线 Java 诊断神器

阿里开源的 Java 在线诊断工具，能不重启、不改代码地观察运行时。

```bash
# 启动（attach 到目标进程）
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar

# 选择目标进程，然后进入交互界面
```

最常用的几个命令：

```bash
# 看 CPU 最高的几个线程及其栈（替代 top -H + jstack）
thread -n 5

# 看死锁
thread -b

# 看类加载情况（找类冲突）
sc -d com.example.MyClass

# 查方法被哪些类引用（找入口）
stack com.example.UserService getUser

# 实时观察方法的入参和返回值
watch com.example.UserService getUser '{params, returnObj, throwExp}' -x 3

# 监控方法调用次数和耗时
monitor -c 5 com.example.UserService getUser

# trace 方法内部每一步耗时
trace com.example.UserService getUser

# 实时反编译运行中的类（看是不是热部署生效了）
jad com.example.UserService

# 跑 async-profiler（集成在 arthas 里）
profiler start --event cpu
profiler stop --file /tmp/flame.html
```

**什么时候用**：需要看运行时数据、动态观察某个方法、临时改变量。
**什么时候不用**：高频热点方法（watch 会拖慢应用 30% 以上）；生产敏感环境（attach 本身有 1-2s STW）。

### 2.3 async-profiler：CPU / Wall / Alloc / Lock 多 event

业界事实标准的 Java 性能剖析工具，比 JProfiler/YourKit 在生产环境更靠谱。

```bash
# 下载
wget https://github.com/async-profiler/async-profiler/releases/download/v3.0/async-profiler-3.0-linux-x64.tar.gz
tar xzf async-profiler-3.0-linux-x64.tar.gz
cd async-profiler-3.0-linux-x64

# CPU 火焰图（最常用）
./bin/asprof -d 30 -f /tmp/cpu.html <pid>

# Wall-clock（看真实墙上时间，IO 等待也能抓到）
./bin/asprof -d 30 -e wall -f /tmp/wall.html <pid>

# 内存分配火焰图（找内存分配热点 → 减 GC 压力）
./bin/asprof -d 30 -e alloc -f /tmp/alloc.html <pid>

# 锁竞争火焰图（找 synchronized / ReentrantLock 热点）
./bin/asprof -d 30 -e lock -f /tmp/lock.html <pid>

# 多 event 一起抓
./bin/asprof -d 60 -e cpu,alloc,lock -f /tmp/multi.jfr <pid>

# JFR 输出（用 JMC / IntelliJ 打开）
./bin/asprof -d 60 -o jfr -f /tmp/profile.jfr <pid>
```

**什么时候用**：CPU 高、内存分配高、锁竞争严重时定位热点。**`-e wall` 是排查"看似不忙但很慢"的神器**。
**什么时候不用**：偶发性问题（必须能复现才能 profile）；分布式问题（单机 profile 看不到下游）。

### 2.4 JFR + JMC：长期持续诊断

JFR（Java Flight Recorder）是 JDK 内置的低开销诊断工具（< 1% CPU 开销），适合**生产常开**。

```bash
# 启动一个连续 1 小时的录制
jcmd <pid> JFR.start name=continuous duration=1h filename=/tmp/jfr-%t.jfr

# 看当前正在录制的
jcmd <pid> JFR.check

# 任意时刻 dump 当前录制的内容
jcmd <pid> JFR.dump name=continuous filename=/tmp/now.jfr

# 启动参数里直接开（推荐生产用）
-XX:StartFlightRecording=settings=profile,duration=1h,filename=/var/log/app/jfr-%t.jfr
```

JMC（JDK Mission Control）是配套的可视化分析工具，下载：[https://www.oracle.com/java/technologies/jdk-mission-control.html](https://www.oracle.com/java/technologies/jdk-mission-control.html)

JFR 的事件分类（你必须知道的）：

| 事件类型 | 用途 |
| --- | --- |
| `jdk.GarbageCollection` | GC 详情 |
| `jdk.ThreadPark` | 线程 park 等待（找锁竞争） |
| `jdk.JavaMonitorEnter` | synchronized 竞争 |
| `jdk.SocketRead` / `jdk.SocketWrite` | 网络 IO |
| `jdk.FileRead` / `jdk.FileWrite` | 文件 IO |
| `jdk.ExecutionSample` | CPU 采样 |
| `jdk.AllocationSample` | 内存分配采样 |

**什么时候用**：生产长期开启、复盘历史问题、对比"故障前后"差异。
**什么时候不用**：要看具体方法的入参（用 arthas watch）。

### 2.5 perf + bpftrace：内核级

JVM 工具看不到内核内部。`perf` 和 `bpftrace` 是 Linux 上的"显微镜"。

```bash
# 看进程级 CPU 事件
perf top -p <pid>

# 抓 30 秒采样，生成火焰图（需要 FlameGraph 项目）
perf record -F 99 -p <pid> -g -- sleep 30
perf script | ./stackcollapse-perf.pl | ./flamegraph.pl > /tmp/perf.svg

# 看系统调用 top
perf trace -p <pid> --duration 100  # 只看 > 100ms 的系统调用

# bpftrace：看磁盘 IO 延迟分布
bpftrace -e 'kprobe:vfs_read /pid == 12345/ { @[comm] = hist(elapsed); }'

# 看 TCP 重传
bpftrace -e 'kprobe:tcp_retransmit_skb { printf("%s pid=%d\n", comm, pid); }'

# 看哪个进程在打开文件
bpftrace -e 'tracepoint:syscalls:sys_enter_openat { @[comm] = count(); }'
```

**什么时候用**：JVM 工具看不出问题（怀疑内核 / Native / 文件 IO）；分析 GC 之外的 STW。
**什么时候不用**：纯 Java 业务问题（杀鸡用牛刀）。

### 2.6 tcpdump + Wireshark：网络层

排查"网络抖动"、"重传"、"连接被 reset"，没有这两个工具几乎不可能定位。

```bash
# 抓某个端口的流量（生产抓包要严格限制时间和大小）
tcpdump -i any -w /tmp/cap.pcap -G 60 -W 1 port 3306

# 抓某个对端 IP
tcpdump -i any -w /tmp/cap.pcap host 10.0.0.1

# 只看握手、挥手、reset
tcpdump -i any 'tcp[tcpflags] & (tcp-syn|tcp-fin|tcp-rst) != 0'

# 抓 HTTP 请求头（看是哪个 URL 慢）
tcpdump -i any -A -s 0 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'

# 把 pcap 拖到本地用 Wireshark 看
# 关键过滤器：
#   tcp.analysis.retransmission       # 重传
#   tcp.analysis.duplicate_ack        # 重复 ACK
#   tcp.flags.reset == 1              # RST
#   tcp.window_size == 0              # 零窗口
```

**什么时候用**：5xx 但应用层日志正常、超时但下游说自己没收到、间歇性丢包。
**什么时候不用**：明确是应用逻辑问题（杀鸡用牛刀）。

### 2.7 strace / ltrace：系统调用追踪

诊断"卡在系统调用"的最直接工具。

```bash
# 跟踪某进程的所有系统调用（开销大，慎用）
strace -p <pid> -f -tt -T -o /tmp/strace.log

# 只看耗时 > 100ms 的系统调用
strace -p <pid> -T -e trace=all 2>&1 | awk '/<.*>/ && $NF+0 > 0.1 {print}'

# 统计哪种系统调用最多
strace -p <pid> -c

# 跟踪 open / connect 这种点状事件
strace -p <pid> -e trace=open,openat,connect

# 跟库函数调用
ltrace -p <pid>
```

**什么时候用**：进程"卡住但 jstack 看不出来"（多半卡在 native 调用）；找文件被打开但没关。
**什么时候不用**：高吞吐场景（strace 会让进程变慢 5-10 倍）。

### 2.8 各工具的"快速选择树"

```
现象                              首选工具
─────────────────────────────────────────────
CPU 高                         → top -H + jstack / arthas thread -n 5
CPU 火焰图                     → async-profiler -e cpu
Wall-clock 火焰图              → async-profiler -e wall
GC 频繁                        → jcmd GC.heap_info / JFR
内存泄漏                       → jcmd GC.class_histogram → 堆 dump → MAT
线程死锁                       → jstack -l / arthas thread -b
锁竞争                         → async-profiler -e lock / JFR JavaMonitorEnter
网络抖动                       → tcpdump + Wireshark
慢系统调用                     → strace -T / perf trace
内核级 IO 延迟                 → bpftrace
看运行时变量值                 → arthas watch
看方法被谁调用                 → arthas stack
持续低开销监控                 → JFR 常开
```

> **口诀**：**jcmd 抓快照、arthas 看实时、async-profiler 找热点、JFR 留长期、perf/bpftrace 下内核、tcpdump 看网络、strace 看系统调用**。七把刀对应七个层次。

---

## 3. 实战剧本：5 个完整定位过程

每个剧本从"告警响起"开始，到"根因确认"结束。命令完整可复制，时间轴标注到秒。

### 剧本 ① P99 飙升 → 5 分钟定位是慢 SQL

#### 告警

```
[2026-01-15 14:23:01] PagerDuty
service=order-svc
metric=http_p99_ms
value=2840
threshold=500
duration=2min
```

P99 在 2 分钟内从 50ms 飙到 2.8s，QPS 没变。

#### 0-60s：对齐现象

打开监控看板（Grafana），看三件事：

```
1. 是哪个接口？        → /api/order/list (95% 的延迟来自这个接口)
2. 是哪台机器？        → 全集群 (排除单机故障)
3. 是不是发布触发？    → 13:50 有过发布
```

CPU、内存、GC 都正常。线程池活跃数从 5 飙到 95（接近 max=100）。

第一个假设：**线程都阻塞在某个慢操作**。

#### 60-120s：定位资源饱和点

线程池满了，去看线程都卡在哪：

```bash
# SSH 到一台问题机器
ssh order-svc-01

# 抓 thread dump
jcmd $(pgrep -f order-svc) Thread.print > /tmp/td.log

# 看哪个状态的线程最多
grep "java.lang.Thread.State" /tmp/td.log | sort | uniq -c | sort -rn
```

输出：

```
   78 java.lang.Thread.State: WAITING (parking)
   12 java.lang.Thread.State: TIMED_WAITING (sleeping)
    8 java.lang.Thread.State: RUNNABLE
```

78 个线程在 WAITING。看是等什么：

```bash
# 抓出 WAITING 线程的栈
grep -B 1 "WAITING (parking)" /tmp/td.log | head -50
```

看到大量栈是：

```
"http-nio-8080-exec-23" WAITING ...
    at jdk.internal.misc.Unsafe.park(Native Method)
    at java.util.concurrent.locks.LockSupport.park
    at com.zaxxer.hikari.pool.HikariPool.getConnection
    ...
    at com.example.OrderService.list
```

线程都卡在等 DB 连接！连接池满了。

#### 120-180s：抓现场

去 DB 看是不是 SQL 慢：

```sql
-- 看当前正在跑的 SQL
SHOW PROCESSLIST;

-- 看慢查询
SELECT * FROM information_schema.PROCESSLIST 
WHERE TIME > 1 AND COMMAND != 'Sleep'
ORDER BY TIME DESC LIMIT 20;
```

发现 30+ 个连接都在跑同一条 SQL，每条耗时 5-10s：

```sql
SELECT * FROM order WHERE user_id = ? AND status IN (1,2,3) ORDER BY create_time DESC LIMIT 20;
```

#### 180-240s：验证假设

`EXPLAIN` 这条 SQL：

```sql
EXPLAIN SELECT * FROM order WHERE user_id = 12345 AND status IN (1,2,3) ORDER BY create_time DESC LIMIT 20;
```

输出：

```
type=ALL  rows=8500000  Extra=Using where; Using filesort
```

**全表扫描 + filesort**！再看索引：

```sql
SHOW INDEX FROM order;
```

发现 `idx_user_id` 索引在 13:50 的发布里被某个 DBA 操作意外删除了。

#### 240-300s：止损

立即恢复索引：

```sql
ALTER TABLE order ADD INDEX idx_user_id_status_ctime (user_id, status, create_time);
```

10s 内 P99 回落到 60ms（接近正常）。

#### 复盘

- **触发因素**：发布时操作了 DB
- **根因 1**：DDL 没走 review 流程
- **根因 2**：监控没有"索引变化"告警
- **根因 3**：服务对单条 SQL 的超时设的太松（没有 fail fast）

> **口诀**：P99 飙升 + QPS 不变 + 线程池满 = **下游慢**。从线程栈往下扒，5 分钟内必能定位到具体的"等待对象"。

---

### 剧本 ② CPU 100% → 定位反射热点

#### 告警

```
[2026-02-08 09:11:33] PagerDuty
service=user-svc
metric=cpu_usage
value=98.5%
threshold=85%
duration=3min
```

CPU 持续 98%，但 QPS 没明显变化。

#### 0-60s：对齐现象

监控板看到：
- CPU 用户态高（不是 sys）
- GC 频率正常（排除 GC 线程"假高")
- 接口延迟略升（30ms → 80ms）但还能用

#### 60-120s：找最热的线程

```bash
# 找 Java 进程
PID=$(pgrep -f user-svc)

# 看这个进程内 CPU 最高的几个线程
top -H -p $PID -n 1 | head -20
```

输出：

```
   PID USER      PR  NI    VIRT    RES  ... %CPU  %MEM
 12453 app       20   0   8.5g  4.2g       95.3   12.1
 12454 app       20   0   8.5g  4.2g       91.7   12.1
 12455 app       20   0   8.5g  4.2g       89.2   12.1
```

记下 TID 12453（十进制），转十六进制：

```bash
printf '%x\n' 12453
# 30a5
```

#### 120-180s：抓栈

```bash
# 抓 thread dump
jcmd $PID Thread.print > /tmp/td.log

# 找这个 nid（注意小写 0x 前缀）
grep -A 20 "nid=0x30a5" /tmp/td.log
```

输出：

```
"http-nio-8080-exec-15" #145 ... nid=0x30a5 runnable [0x...]
   java.lang.Thread.State: RUNNABLE
    at sun.reflect.GeneratedMethodAccessor82.invoke(Unknown Source)
    at sun.reflect.DelegatingMethodAccessorImpl.invoke
    at java.lang.reflect.Method.invoke
    at com.example.util.BeanCopier.copy
    at com.example.user.UserConverter.toDto
    at com.example.user.UserService.getUser
```

反射在执行 BeanCopier。但反射本身不至于打满 CPU，再看几个高 CPU 线程，发现都卡在同一处。

#### 180-240s：用 async-profiler 火焰图证实

```bash
./bin/asprof -d 30 -f /tmp/cpu.html $PID
```

火焰图显示 **70% 的 CPU 时间在 BeanCopier 内部的 Method.invoke**。

进一步看 BeanCopier 实现：

```java
public static <T> T copy(Object src, Class<T> targetClass) {
    T target = targetClass.getDeclaredConstructor().newInstance();
    for (Method getter : src.getClass().getMethods()) {
        if (getter.getName().startsWith("get")) {
            String setterName = "set" + getter.getName().substring(3);
            Method setter = targetClass.getMethod(setterName, getter.getReturnType());
            setter.invoke(target, getter.invoke(src));
        }
    }
    return target;
}
```

**每次调用都用 reflection 找 method**！没有任何缓存。

#### 240-300s：止损

紧急方案：把 BeanCopier 换成 MapStruct（编译期生成代码，零反射）：

```java
@Mapper
public interface UserConverter {
    UserConverter INSTANCE = Mappers.getMapper(UserConverter.class);
    UserDto toDto(User user);
}
```

或者临时方案：给 BeanCopier 加 `ConcurrentHashMap` 缓存 method 解析。发布 hotfix 后 CPU 回落到 30%。

#### 复盘

- **触发因素**：流量正常，但用户增加了一些字段，BeanCopier 反射的字段更多
- **根因**：反射没有缓存，每次调用都做 Class 解析
- **教训**：性能敏感路径上的"看似无害的工具方法"，必须 profile 一次

> **口诀**：CPU 100% + 业务正常 = **某个被高频调用的方法热点**。`top -H + jstack + 火焰图`三件套，5 分钟内必能定位到具体方法。

---

### 剧本 ③ OOM → 定位 ThreadLocal 泄漏

#### 告警

```
[2026-03-12 02:45:12] OOMKilled
container=user-svc-pod-x9j2
exit_code=137
restart_count=4 in last 1h
```

容器频繁 OOM 重启。看 Prometheus 内存图，呈"锯齿状"持续上升。

#### 0-60s：对齐现象

锯齿状上升 = 内存有泄漏，每次 GC 没回收完整。
- 重启间隔约 15 分钟
- QPS 是平时的 1.2 倍（凌晨批处理）
- 没有发布

#### 60-120s：抓堆 histogram

`-XX:+HeapDumpOnOutOfMemoryError` 已经开了，但 OOM 时被 K8s 杀掉了，dump 没写完。改用 jcmd 主动抓：

```bash
PID=$(pgrep -f user-svc)

# 先看 heap 占用
jcmd $PID GC.heap_info

# 抓 class histogram（轻量、秒级）
jcmd $PID GC.class_histogram | head -30
```

输出：

```
 num     #instances         #bytes  class name
   1:      4523412     1808964800  com.example.user.UserContext
   2:      4523412      361873000  java.lang.String
   3:       125000        2000000  byte[]
   ...
```

**450 万个 UserContext 实例**！这个对象本应每个请求一个，QPS 才 2k，怎么积累了 450 万？

#### 120-180s：找谁在持有

```bash
# 抓 heap dump（这次手动控制，不让 K8s 杀掉）
jcmd $PID GC.heap_dump /tmp/heap.hprof

# 用 MAT 或 jhat 分析
# Mac 上推荐：JProfiler / VisualVM / IntelliJ Profiler
```

打开 dump 找 UserContext 的 GC Root。发现路径：

```
GC Root → ThreadLocalMap (in Thread 'http-nio-8080-exec-12')
       → Entry [key=UserContextHolder.CTX, value=UserContext]
       → UserContext { user, perm, ... }
```

`http-nio-8080-exec-*` 是 Tomcat worker 线程。它的 ThreadLocal 里塞了 UserContext，但**没人 remove**。

#### 180-240s：找代码

```bash
grep -r "UserContextHolder.set" --include="*.java"
```

找到一处过滤器：

```java
@Component
public class UserContextFilter implements Filter {
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
        UserContext ctx = parseFromHeader((HttpServletRequest) req);
        UserContextHolder.set(ctx);
        chain.doFilter(req, res);
        // ★ 没有 finally 里的 UserContextHolder.remove()
    }
}
```

`Tomcat` 的 worker 线程是复用的，`UserContext` 一旦 set 就一直挂在 ThreadLocalMap 里。下一个请求来时 `set` 新的，**但旧的对象不会立刻被回收**（ThreadLocalMap 是弱引用 key 但强引用 value）。

更糟的是，UserContext 里持有了 `permList`（一个 List<Permission>），平均 10MB。450 万个 ≈ 4.5TB 引用关系，但实际 GC Root 路径上只有 4-5 万个真正强引用，剩下的是 GC 没来得及清理的 ThreadLocalMap entry。

#### 240-300s：止损

修复代码：

```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
    UserContext ctx = parseFromHeader((HttpServletRequest) req);
    UserContextHolder.set(ctx);
    try {
        chain.doFilter(req, res);
    } finally {
        UserContextHolder.remove();   // ★ 必须 remove
    }
}
```

紧急回滚到上一版本（上一版本的过滤器恰好有 remove）。同时把 `UserContext` 里的 `permList` 改成按需懒加载，避免一次塞 10MB。

#### 复盘

- **触发因素**：QPS 升高让泄漏的累积速度超过 GC 速度
- **根因**：ThreadLocal 没 remove
- **更深根因**：代码 review 没有"ThreadLocal must remove"的硬性规则
- **教训**：所有 `ThreadLocal.set` 必须配 try-finally `remove`

> **口诀**：OOM 不是"内存不够"，是"内存被某个对象占满"。`class_histogram → heap_dump → GC Root`三步，10 分钟内定位泄漏对象。

---

### 剧本 ④ 网络抖动 → 定位 NAT 超时

#### 告警

```
[2026-04-03 11:08:22] PagerDuty
service=order-svc
metric=downstream_call_error_rate
target=payment-svc
value=15%
threshold=2%
```

调下游 payment-svc 的失败率从 0.1% 飙到 15%，但 payment-svc 自己监控显示一切正常。

#### 0-60s：对齐现象

- order-svc 看到 15% 调用失败
- payment-svc 自己日志说"我没收到这些请求"
- 失败现象：connection timeout（连接超时，不是响应慢）
- 不是全失败，是 15% 概率失败

第一假设：**网络层有抖动**。

#### 60-120s：看连接状态

```bash
# 在 order-svc 容器内
PID=$(pgrep -f order-svc)

# 看 TCP 连接状态分布
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c
```

输出：

```
  234 ESTAB
   18 TIME-WAIT
   12 SYN-SENT      # ★ 异常多
    3 LAST-ACK
```

`SYN-SENT` 多说明发出去的 SYN 没收到 SYN-ACK。看具体目标：

```bash
ss -tan state syn-sent
```

```
... 10.0.0.123:54312    10.0.10.45:8080  
... 10.0.0.123:54315    10.0.10.45:8080  
... 10.0.0.123:54328    10.0.10.45:8080  
```

都打到 10.0.10.45（payment-svc 的 ClusterIP），但握手不上。

#### 120-180s：抓包

```bash
# 在 order-svc 节点抓包
tcpdump -i any -w /tmp/cap.pcap host 10.0.10.45 -G 60 -W 1
```

```bash
# 在 payment-svc 节点也抓包
tcpdump -i any -w /tmp/cap.pcap port 8080 -G 60 -W 1
```

把两边 pcap 拖到本地用 Wireshark 对比。发现：

- order-svc 这边发了 SYN
- payment-svc 那边**有的能收到、有的收不到**
- 收不到的 SYN 在中间某处被丢了

#### 180-240s：定位丢包点

K8s 里 Service 的 ClusterIP 走的是 iptables/IPVS 做 NAT。看 `conntrack` 表：

```bash
# 看 conntrack 表大小
sudo conntrack -C

# 看表上限
sysctl net.netfilter.nf_conntrack_max

# 看溢出计数器
cat /proc/net/stat/nf_conntrack | awk 'NR>1 {sum_drop+=$3} END {print "drop:", sum_drop}'

# 看 dmesg 是否有 conntrack 满的日志
dmesg | grep -i conntrack
```

输出：

```
table size: 524000
nf_conntrack_max: 524288
drop: 12834
dmesg: nf_conntrack: table full, dropping packet.
```

**conntrack 表满了**！每个 TCP 连接都要在 conntrack 里占一条记录，包括 NAT 的转换。表满之后新建连接的 SYN 就被丢了（不是被 RST，是被悄悄丢，所以表现为超时）。

#### 240-300s：止损

紧急扩容 conntrack 表：

```bash
# 立即生效
sudo sysctl -w net.netfilter.nf_conntrack_max=2097152

# 持久化
echo "net.netfilter.nf_conntrack_max=2097152" | sudo tee -a /etc/sysctl.conf
```

15 秒内失败率回落到 0.1%。

#### 复盘

- **触发因素**：order-svc 短连接配置（每次调用建新连接）+ QPS 上升
- **根因 1**：conntrack 表上限没有按集群规模调整（默认值对中等规模 K8s 集群不够）
- **根因 2**：order-svc 应该用连接池而不是短连接
- **根因 3**：conntrack 用满没有告警

后续动作：
1. 把 conntrack 监控接入 Prometheus（`node_nf_conntrack_entries`）
2. order-svc 改用 OkHttp 连接池
3. 全集群 conntrack 表按节点规格自动调整

> **口诀**：网络抖动 + 应用层无错日志 + 间歇性超时 = **基础设施层问题**。`ss + tcpdump + conntrack` 是排查 K8s 网络问题的三件套。

---

### 剧本 ⑤ 间歇性 5xx → 定位连接池配置错误

#### 告警

```
[2026-05-08 16:44:51] PagerDuty
service=order-svc
metric=http_5xx_rate
value=3.2%
threshold=1%
duration=continuous, but pulsating
```

5xx 错误率呈"心电图"状：每隔 30-60s 飙一波，持续几秒，又恢复正常。

#### 0-60s：对齐现象

- 错误日志写："Connection is not available, request timed out after 30000ms"（HikariCP）
- QPS 没变化、CPU 正常、GC 正常
- 影响所有接口（不是单接口）

第一假设：**DB 连接池在某些时刻拿不到连接**。

#### 60-120s：看连接池监控

打开 HikariCP 的 Micrometer 指标：

```
hikaricp_connections_active{pool="db"}     = 50  (峰值=50)
hikaricp_connections_idle{pool="db"}        = 0
hikaricp_connections_pending{pool="db"}     = 30
hikaricp_connections_max{pool="db"}         = 50
```

连接池 max=50 经常被打满，30 个请求在等。

但奇怪的是平均响应时间不慢：50 / (avg 50ms) = 1000 QPS 上限，实际 QPS 才 600，按理不该打满。

#### 120-180s：找慢连接

```sql
-- 在 DB 端看连接的真实耗时
SELECT id, user, host, command, time, state, info
FROM information_schema.PROCESSLIST
WHERE command != 'Sleep'
ORDER BY time DESC
LIMIT 20;
```

发现有 5-6 个连接长期处于 "Sleep" 状态超过 30 秒。

更奇怪：监控里 `hikaricp_connections_idle = 0`，但 DB 端却看到 Sleep 连接。这两边对不上 → **连接被应用层"持有"，但应用没在用**。

#### 180-240s：抓应用栈

```bash
jcmd $(pgrep -f order-svc) Thread.print > /tmp/td.log

# 看持有 DB 连接但在干别的事的线程
grep -B 2 -A 30 "HikariProxyConnection" /tmp/td.log | head -100
```

看到栈：

```
"http-nio-8080-exec-23" RUNNABLE
   java.lang.Thread.State: RUNNABLE
    at java.net.SocketInputStream.socketRead0(Native Method)
    at java.net.SocketInputStream.socketRead
    ...
    at org.elasticsearch.client.RestClient.performRequest
    at com.example.SearchService.search
    at com.example.OrderService.getOrders   ★ 持有 DB 连接
    at ...
    at org.springframework.transaction.interceptor.TransactionInterceptor.invoke
```

`getOrders` 在事务里调了 ES 搜索！事务持有 DB 连接，去等 ES 响应。如果 ES 慢，DB 连接就被白白占着。

代码：

```java
@Transactional
public List<Order> getOrders(String keyword) {
    List<Long> ids = searchService.search(keyword);   // ★ ES 调用
    return orderRepository.findByIds(ids);
}
```

#### 240-300s：止损

紧急：去掉 `@Transactional`（这个方法只查不改，根本不需要事务）：

```java
public List<Order> getOrders(String keyword) {
    List<Long> ids = searchService.search(keyword);
    return orderRepository.findByIds(ids);
}
```

5xx 错误率立刻降到 0.05%。

#### 复盘

- **触发因素**：ES 集群偶发慢查询（某个分片在做 segment merge）
- **根因 1**：`@Transactional` 滥用：只读方法不需要事务
- **根因 2**：事务里调远程服务，外部抖动会放大成 DB 连接池耗尽
- **根因 3**：HikariCP 监控没接 leak detection

后续动作：
1. 全代码扫描："`@Transactional` 方法里禁止远程调用"
2. 接入 HikariCP `leakDetectionThreshold=10000`
3. 加监控：`hikaricp_connections_pending > 0` 告警

> **口诀**：间歇性 5xx + 连接池满 + DB 端有空闲 sleep 连接 = **事务持有连接做了它不该做的事**。监控对不上，必有"持有但未用"。

---

## 4. 生产案例 9 则 —— 现象/根因/修复/验证

> 上面 5 个剧本是"完整定位流程"，下面 9 个案例是更聚焦的"误判 / 教训"。

### 案例 ① 误判：CPU 高以为是业务，实际是 GC 线程

#### 现象

监控告警 CPU 90%。打开 `top` 一看，`java` 进程 CPU 90%。开始查业务代码热点。

#### 根因

实际看 `top -H -p <pid>`：

```
   PID    %CPU  COMMAND
 23451    78.2  GC Thread#0
 23452    72.4  GC Thread#1
 23453    71.8  GC Thread#2
 23454    70.5  GC Thread#3
 12453     8.2  http-nio-8080-exec-1
```

CPU 都被 **GC 线程** 占了，不是业务线程。

进一步看 GC 状况：

```bash
jcmd <pid> GC.heap_info
```

```
heap usage: 4.8 GB / 5 GB (96%)
young gen: 1.5 GB / 1.6 GB (94%)
old gen: 3.3 GB / 3.4 GB (97%)
```

老年代用满 97%，CMS 在拼命做 concurrent mark + sweep，吃掉了 4 个 CPU 核。

实际**业务一切正常，这是 GC "漂亮地"在工作**，但用户感受是 CPU 高。

#### 修复

短期：扩堆（`-Xmx5g` → `-Xmx8g`），让 GC 不要这么拼。

长期：分析为什么老年代满 —— 一般是缓存对象太多、或者短生命周期对象被错放进了老年代。改 GC 算法（CMS → G1 / ZGC）通常也能缓解。

#### 验证

```bash
# 持续观察 GC 频率
jstat -gc <pid> 1000 30

# Full GC 不再频繁出现
```

> **教训**：CPU 高第一个动作是 `top -H`，看是哪类线程，不是直接看业务。**GC 线程命名规则**：CMS/G1 是 `GC Thread#N`，ZGC 是 `ZGC Major/Minor`，Shenandoah 是 `Shenandoah`。

---

### 案例 ② 误判：DB 慢以为是查询，实际是连接池配错

#### 现象

监控显示 DB 操作 P99 = 5s。一群人冲去优化 SQL、加索引、查 EXPLAIN。

#### 根因

EXPLAIN 出来索引完美，DB 端 `SHOW PROCESSLIST` 看到的查询都很快（< 10ms）。

应用层"DB 慢"的指标实际包含了**等待连接 + 真正执行**两部分。HikariCP 的 `hikaricp_connections_pending` 才是关键：

```
hikaricp_connections_pending = 35  ★ 35 个请求在排队等连接
hikaricp_connections_max     = 10  ★ 池只配了 10 个！
```

5s 的 P99 里，4.9s 是在等连接，0.1s 是真的查 DB。

#### 修复

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 50      # 改大
      minimum-idle: 10
      connection-timeout: 3000   # 等不到就 fail fast，不要默认 30s
      leak-detection-threshold: 10000   # 持有 > 10s 报警
```

#### 验证

P99 立刻从 5s 回到 80ms。

> **教训**："DB 慢"必须区分**真慢（执行慢）**和**伪慢（等连接慢）**。HikariCP 的 `pending` 指标是分水岭。

---

### 案例 ③ 太慢：定位耗时 2 小时（应该 5 分钟）

#### 现象

某次 P99 飙升告警，团队 2 小时才定位是某个第三方 SDK 的死循环 bug。复盘时大家说"找不到"。

#### 根因（流程问题）

回看 2 小时的定位过程：

```
14:00 告警响 → 大家在群里讨论"是不是发布"
14:15 看了发布记录，最近无发布
14:25 怀疑数据库，DBA 来看了 30 分钟
14:55 怀疑 Redis，Redis 同学看了 20 分钟
15:15 怀疑下游，下游同学排查 30 分钟
15:45 才有人说"要不抓个 thread dump"
15:50 thread dump 一出，5 秒看出死循环
```

**真正"看 dump"只用了 5 秒，前面 1 小时 50 分都在猜**。

#### 修复（流程）

定立"定位优先级":

```
Step 1（< 30s）：监控板看现象 → 锁定方向
Step 2（< 60s）：thread dump + heap histogram + GC log → 抓现场
Step 3（< 5min）：根据现场数据，定向问下游/DB/SRE
Step 4（> 5min）：还没定位 → 立刻止损（限流、扩容、回滚），再慢慢找
```

不要让 Step 1 / 2 拖到 30 分钟之后才做。

#### 验证

下次故障时 follow checklist。**定位时间从 2 小时降到 8 分钟**。

> **教训**：5 分钟定位的不是"工具",是"先抓现场再思考"的流程。**抓现场（dump、profile、日志）应该是告警后 60 秒内的本能动作**，而不是猜来猜去之后的最后选项。

---

### 案例 ④ 根因找错：修了表象问题，过几天复发

#### 现象

服务 OOM。看 dump 发现 `byte[]` 占了 8GB。代码定位到一处图片处理：

```java
byte[] image = downloadImage(url);   // 30MB 大图
byte[] thumbnail = resize(image);
return thumbnail;
```

修复："把图片改成流式处理，不一次性读到内存"。上线后 OOM 解决。

3 天后，同样位置又 OOM。

#### 根因

第一次修复只是减少了**单次内存峰值**，但**没解决"为什么有大量并发请求处理大图"的问题**。

实际真根因：上游某个新接入的客户端，调用频率是其他客户端的 100 倍，且图片尺寸都是 30MB+。流式处理虽然降了单次峰值，但并发数没限，整体内存还是被吃完。

#### 修复

第二轮修复：
1. 加客户端级限流：单客户端 QPS ≤ 10
2. 加图片大小校验：> 10MB 直接拒绝
3. 加内存熔断：进程 RSS > 80% 时拒绝新请求

#### 验证

跟踪 14 天没有 OOM。

> **教训**：每次"修复"完要问一遍：**触发因素再来一次会不会再爆？** 如果会，就还没找到根因。第一次的修复是"症状级"的，第二次的修复才是"系统级"的。

---

### 案例 ⑤ Thread dump 解读错误（线程在 RUNNABLE 但实际等 IO）

#### 现象

服务"卡住"。jstack 看大量线程在 RUNNABLE 状态，CPU 却很低（5%）。

```
"http-nio-8080-exec-23" #145 RUNNABLE
   java.lang.Thread.State: RUNNABLE
    at java.net.SocketInputStream.socketRead0(Native Method)
    at java.net.SocketInputStream.socketRead
    at java.net.SocketInputStream.read
    ...
    at sun.net.www.protocol.http.HttpURLConnection.getInputStream
    at com.example.PaymentClient.callPayment
```

新人看了说："线程都在跑啊，应该 CPU 满才对，怎么会卡？"

#### 根因

JVM 的 thread state 和操作系统的 thread state 不一样。

**`socketRead0` 是 native 方法**。Java 看到"调用了 native"就标 RUNNABLE，但实际上 native 调用 `read()` 系统调用后，**操作系统层面线程在 `D` 状态（不可中断睡眠）等待网卡数据**。

| Java 状态 | OS 状态 | CPU 实际占用 |
| --- | --- | --- |
| RUNNABLE 跑业务代码 | R (running) | 高 |
| RUNNABLE 在 socketRead | D (uninterruptible sleep) | 0 |
| RUNNABLE 在文件 read | D | 0 |
| BLOCKED 等 monitor | S (sleep) | 0 |
| WAITING / TIMED_WAITING | S | 0 |

所以 `RUNNABLE + socketRead0/socketAccept/FileInputStream.readBytes` = **实际在等 IO，不是在跑**。

#### 修复（识别）

看到大量这样的 RUNNABLE 栈，应该问：
1. 等的是哪个对端？（`netstat -anp | grep <pid>` 看 ESTAB 对端）
2. 对端响应慢吗？
3. 自己有没有超时？

#### 验证

去对端服务看 P99，确认是对端慢。然后给客户端加超时：

```java
HttpURLConnection conn = ...;
conn.setConnectTimeout(3000);
conn.setReadTimeout(5000);
```

> **教训**：**RUNNABLE ≠ 真的在跑 CPU**。看到 native 方法（`socketRead0`、`accept0`、`read`、`epollWait`）时，要按 IO 等待来理解。`async-profiler -e wall`（墙上时间）能直接抓到这些"假 RUNNABLE"的等待。

---

### 案例 ⑥ 火焰图误读（epoll_wait 占栈不是问题）

#### 现象

抓了个 CPU 火焰图，发现 `epoll_wait` 占了栈高 60%。新人说："这就是瓶颈，得优化 epoll！"

#### 根因

这是**对火焰图最常见的误读**之一。

`epoll_wait` 是 Netty / Tomcat NIO / Node.js 这类异步框架的"主循环出口"。**它阻塞等事件**。CPU 火焰图里它占栈高，意味着"线程大部分时间都在等事件"，**这是好事，说明你的进程没忙到爆**。

类似的"看起来高但其实是好事"的栈：

| 栈帧 | 含义 |
| --- | --- |
| `epoll_wait` / `kevent` | 事件循环在等事件 |
| `Object.wait` / `LockSupport.park` | 线程在 idle |
| `pthread_cond_wait` | C 库在等条件变量 |
| `accept` | 服务端在等新连接 |
| `poll` / `select` | 在等 IO |

#### 修复（识别）

火焰图看 CPU 时，**要看的是"在跑业务代码"那部分的占比**。`epoll_wait` 在采样里占 60% = 你的进程 60% 时间是 idle，剩下 40% 才是真正干活。

如果想真的找瓶颈，应该用：

```bash
# 排除"等待"的栈，只看活跃 CPU
async-profiler -e cpu -X "java/lang/Thread.run;sun/nio/ch/EPollSelectorImpl.doSelect" -f /tmp/cpu.html <pid>

# 或者用 wall-clock 火焰图，看真正在等什么
async-profiler -e wall -f /tmp/wall.html <pid>
```

#### 验证

去掉 epoll_wait 后看，业务方法的占比就清楚了。

> **教训**：CPU 火焰图里的"等待型 syscall"高占比不是问题。**真正的 CPU 瓶颈一定是业务代码或框架代码的栈帧深而宽**。

---

### 案例 ⑦ 监控数据延迟导致定位时机错过

#### 现象

某次 P99 飙升，告警 5 分钟后才响。SRE 冲过来定位，但故障已经自愈了 3 分钟。监控板上看不到现场，只看到一个尖峰。

#### 根因

监控链路：
```
应用 → 本地 buffer (1min) → Pushgateway → Prometheus 抓取 (15s)
     → Grafana 渲染 (auto refresh 1min)
```

总延迟 = 1min + 15s + 1min = **2 分 15 秒**。
告警评估窗口又是 3 分钟。
所以告警响起时，故障已经发生了 5+ 分钟。

故障如果只持续 5 分钟，**告警响时它已经结束**。

#### 修复

1. **应用直接 push 关键指标到 Prometheus**（不走 buffer）
2. **告警评估窗口缩短**：P99 用 1min 窗口，错误率用 30s 窗口
3. **保留 Profiling 历史**：服务持续开 JFR，故障后能回看

```yaml
# prometheus.yml
rule_files:
  - rules/p99.yml
scrape_configs:
  - job_name: 'app'
    scrape_interval: 5s   # 缩短抓取间隔
```

```yaml
# rules/p99.yml
groups:
- name: p99
  rules:
  - alert: P99High
    expr: histogram_quantile(0.99, http_duration_bucket) > 0.5
    for: 30s   # 缩短评估窗口
```

应用层启动加 JFR 常开：

```bash
-XX:StartFlightRecording=settings=profile,duration=4h,filename=/var/log/app/jfr-%t.jfr
```

#### 验证

下次故障告警 30s 内响起，且 JFR 能回放故障现场。

> **教训**：**告警的延迟 = 监控管道延迟 + 评估窗口延迟**。任何一环过长，"5 分钟内定位"就成空话。每个团队都该测一次自己的告警端到端延迟。

---

### 案例 ⑧ 多人协作时信息断层

#### 现象

某次大故障，10 个人在群里讨论。SRE 说"看是 DB"、DBA 说"DB 没问题"、应用说"代码没改"、网络说"网络正常"。30 分钟过去了，每个人都在重复抓自己负责领域的现场，但**没人把现场拼起来**。

#### 根因

故障定位的"协作模型"不对。每个人都在自己领域做"局部最优"，没有"全局指挥"。

#### 修复（流程）

定立故障应急角色：

```
Incident Commander (IC)：决策、不下沉到细节、负责协调
   - 当前推进到哪一步？
   - 下一步谁做什么？
   - 是否需要止损（扩容/限流/回滚）？

Tech Lead (TL)：技术深入、抓现场、读栈
   - 看 dump、看监控、判断方向

Communicator：对外沟通
   - 跟客户/老板汇报进度
   - 不打扰 IC 和 TL

Scribe：记录时间轴
   - 14:23 告警
   - 14:25 抓 dump
   - 14:28 怀疑 DB
   - ... 每个动作记一行
```

群里发"现场链接表"：

```
Thread dump:    https://oss/td-2026010815.log
Heap histogram: https://oss/hist-2026010815.log
JFR:            https://oss/jfr-2026010815.jfr
监控板:         https://grafana/d/order/order-svc?from=2026-01-08T14:00
DB 慢查询:      https://dms/slow-2026010815
```

让所有人看同一份现场，避免重复抓。

#### 验证

定立角色后，下次同样规模故障定位时间从 30 分钟降到 8 分钟。

> **教训**：**大故障的瓶颈不在工具，在协作**。Incident Commander 模型来自 Google SRE，工程团队 > 5 人就该用。

---

### 案例 ⑨ 没有上一次的基线，无法识别"变化"

#### 现象

服务"感觉慢了"，但没有人知道是真的慢了还是错觉。监控里 P99 是 80ms，没人记得"上周这时候是多少"。

最后查了一周前的 Grafana 截图，发现去年 P99 一直是 30ms，最近 3 个月慢慢爬到 80ms。

#### 根因

没有"基线"概念。服务的性能指标是**变化**的，但团队只关注"绝对值是否触发告警"，不关注"变化趋势"。

#### 修复

1. **保留长周期监控**：Prometheus retention 至少 90 天
2. **加"趋势告警"**：P99 持续上升超过 50% / 连续 7 天的告警
3. **做"性能巡检"**：每周对比关键指标 vs 上周，写复盘
4. **做"性能基线"**：每个版本上线前后，自动跑一遍压测，对比上一版本基线

```yaml
- alert: P99Trending
  expr: |
    avg_over_time(p99[1d]) > 1.5 * avg_over_time(p99[1d] offset 7d)
  for: 1h
  annotations:
    summary: "P99 比 7 天前上升 50%"
```

#### 验证

加了趋势告警后，下个季度发现一处"温水煮青蛙"式性能退化（每周 P99 涨 5%），及时回滚。

> **教训**：**故障定位不是"出问题才看"**。基线、趋势、对比 —— 这三件事必须日常做，否则你永远是"被告警追着跑"。

---

## 5. 上线检查清单

打印贴墙 / 加进 SRE 运维 checklist。

### 工具准备

- [ ] 每个 Java 进程启动参数加 `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/log/app/`
- [ ] 启动参数加 JFR 常开：`-XX:StartFlightRecording=settings=profile,duration=4h,filename=/var/log/app/jfr-%t.jfr`
- [ ] 启动参数加 GC 日志：`-Xlog:gc*=info:file=/var/log/app/gc-%t.log:time,uptime:filecount=10,filesize=100M`
- [ ] arthas、async-profiler 二进制预装在容器镜像里
- [ ] 容器允许执行 `jcmd / jstack / jmap`（设置 `--cap-add SYS_PTRACE`）
- [ ] tcpdump、ss、netstat、conntrack-tools 在 SRE 节点工具箱里

### 监控

- [ ] 每个服务的 P50/P95/P99/P999 都有图、有告警
- [ ] 告警评估窗口 ≤ 1 分钟（不要 5 分钟才响）
- [ ] 监控抓取间隔 ≤ 15 秒
- [ ] DB 慢查询接入告警（pt-query-digest 周报）
- [ ] HikariCP / Druid 连接池指标接入：active / idle / pending / leak
- [ ] commonPool 监控：active / queued / steal
- [ ] JVM 关键指标：heap_used / heap_max / gc_count / gc_time
- [ ] 系统层：CPU / Load / Memory / Disk / Network / FD
- [ ] K8s 容器层：cpu_throttling / memory_working_set / restart_count
- [ ] conntrack 表使用率（`node_nf_conntrack_entries`）

### 演练

- [ ] 每季度一次混沌演练（kill pod / 断网 / DB 主备切换）
- [ ] 新人入职 1 个月内做一次"假告警"演练
- [ ] 每年至少一次 Game Day（业务全员参与的故障模拟）
- [ ] 故障应急 runbook 必须可被任何 oncall 在不联系作者的情况下执行
- [ ] 关键命令做成 alias / 脚本，例如 `tdump <service>` 直接抓 dump

### 复盘

- [ ] 每次 P0/P1 故障 48 小时内出复盘文档
- [ ] 复盘必须包含：时间轴 / 根因 / 触发因素 / 止损动作 / 改进项
- [ ] 改进项必须有 owner 和 deadline
- [ ] 同类故障半年内复发 → 必须升级到团队 leader review
- [ ] 复盘结论沉淀进团队知识库（可搜索）
- [ ] 月度复盘：本月 top 3 故障类型 / 工具改进点 / 流程优化点

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### SRE 工作簿与故障复盘文化
- Google SRE Book / Workbook 必读，重点看 Chapter 14 (Managing Incidents) / Chapter 15 (Postmortem Culture)
- Blameless Postmortem 的实践模板（参考 Etsy / PagerDuty 公开模板）
- 故障分级（P0~P3）的客观标准如何制定，避免"按老板情绪"分级
- "5 Whys" 分析法的局限：复杂系统故障常常没有单一根因

### 混沌工程作为训练
- **Chaos Mesh**（CNCF）：K8s 原生混沌平台，支持网络、Pod、IO、Stress 故障注入
- **Litmus**（CNCF）：另一个 K8s 混沌平台，更偏 GitOps
- **ChaosBlade**（阿里）：物理机/容器/JVM 多平台混沌工具
- 混沌实验的"爆炸半径"控制：从单机 → 单可用区 → 全集群的渐进式扩展

### eBPF 在故障定位中的应用
- **bcc-tools** 工具集：`tcpconnect / tcpretrans / biolatency / runqlat`
- **bpftrace** 单行命令解决"我想看某个内核事件"的需求
- **Pixie**（CNCF）：基于 eBPF 的 K8s 观测平台，自动给 HTTP/gRPC/MySQL 加 trace
- **Parca**（CNCF）：基于 eBPF 的持续 profiling，比 async-profiler 开销更低
- 学习路径：先看 Brendan Gregg 的 "BPF Performance Tools" 一书

### 基于 LLM 的智能根因分析
- 微软的 [PromptKG](https://arxiv.org/abs/2310.07845)：用 LLM 关联 metrics / logs / traces 推断根因
- Datadog Watchdog / New Relic Lookout：商业产品里的异常检测和根因推荐
- 开源探索：[K8sGPT](https://github.com/k8sgpt-ai/k8sgpt) 用 LLM 解读 K8s 事件
- 当前局限：LLM 善于"模式识别"，不善于"因果推理"，故障的因果链常常需要工程师补全

### Distributed Tracing 在跨服务故障中的价值
- **OpenTelemetry**：分布式追踪的事实标准
- **Jaeger / Tempo / Zipkin**：开源的 trace 后端
- Trace 的采样策略：1% 头部采样 vs 100% 尾部采样（基于错误/延迟决定保留）
- Trace 的 SQL 集成：自动给慢 SQL 关联到引发它的 trace
- "代码级 profiling + trace 关联"：知道是某次慢 trace 之后，能直接看到那次的火焰图

### 战时演练（Game Day）
- Amazon / Netflix 的实践：每月一次全公司参与的"假故障"
- Game Day 的设计原则：必须让"真实业务方"也参与，而不只是工程团队
- "Wheel of Misfortune"：随机抽人到桌前模拟应对一个故障，验证 runbook 实用性
- 衡量指标：MTTD（Mean Time To Detect）/ MTTA（Mean Time To Acknowledge）/ MTTR（Mean Time To Restore）

### Profile Guided Optimization (PGO)
- 用持续 profile 数据反向指导代码优化（Go / Rust 已支持）
- Java 上的等价物：JIT 的 `-XX:+UseProfiledCode` + JFR 持续采样
- 把"线上数据"喂回"开发流程"，避免主观优化错误的热点

### 长尾延迟的 Tail at Scale
- Jeff Dean 的经典论文 *The Tail at Scale*（2013）必读
- Hedged Request（对冲请求）：发两个请求取先返回的，降 P99
- Backup Request 的代价：放大了 2 倍流量，必须配限流
- "微服务化让 P99 雪上加霜"的数学解释

---

## 7. 引用

- Brendan Gregg, *Systems Performance: Enterprise and the Cloud, 2nd ed.*（2020）—— USE 方法论的圣经
- Brendan Gregg, *BPF Performance Tools*（2019）—— eBPF 和 bpftrace 必读
- Google SRE Book / Workbook：[https://sre.google/books/](https://sre.google/books/)
- Tom Wilkie, *The RED Method*：[https://thenewstack.io/monitoring-microservices-red-method/](https://thenewstack.io/monitoring-microservices-red-method/)
- Jeff Dean & Luiz Barroso, *The Tail at Scale*（CACM, 2013）
- async-profiler：[https://github.com/async-profiler/async-profiler](https://github.com/async-profiler/async-profiler)
- arthas 文档：[https://arthas.aliyun.com/doc/](https://arthas.aliyun.com/doc/)
- JEP 328: Flight Recorder
- Brendan Gregg's Flame Graphs：[https://www.brendangregg.com/flamegraphs.html](https://www.brendangregg.com/flamegraphs.html)
- Chaos Mesh 文档：[https://chaos-mesh.org/docs/](https://chaos-mesh.org/docs/)
- 阿里 [TransmittableThreadLocal](https://github.com/alibaba/transmittable-thread-local) 在故障注入场景的应用

---

> 配套：[可观测性主题](/article/observability/) | [Prometheus 指标设计](/article/observability/prometheus) | [火焰图实战](/article/linux-perf/flame-graph) | [JVM 调优与排查](/article/jvm/tuning)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
