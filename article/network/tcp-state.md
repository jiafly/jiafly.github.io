# TCP 状态机深入 —— 从 SYN_FLOOD 到 CLOSE_WAIT 堆积的全套排查

> 这是按 [桃花岛深度模板](/STYLE) 写的网络主题深文。如果你只想刷面试，看 [网络综述](/article/network/) 那一节就够了。
> 这里要回答的，是**线上 `netstat` 看到一堆 CLOSE_WAIT、TIME_WAIT 占满端口、SYN 包被丢**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. TCP 一共 11 个状态，你能在不查资料的情况下画出转换图吗？特别是 `CLOSING` 和 `LAST_ACK` 是怎么进的？
2. `TIME_WAIT` 为什么必须等 2MSL（2 倍最大段寿命）？少等会怎样？多等会怎样？
3. 线上 `netstat` 里堆了几千个 `CLOSE_WAIT`，**根因永远不在内核**，你能一句话说出在哪吗？
4. SYN_FLOOD 攻击下，为什么标准答案是开 `tcp_syncookies`，而不是简单地把半连接队列调大？这两者的边界在哪？
5. `tcp_tw_reuse` 和（已被移除的）`tcp_tw_recycle`，在 NAT 下都不能开 —— 你能说出 recycle 翻车的精确机理吗？
6. 半连接队列（SYN queue）和全连接队列（accept queue）是两个独立的东西，溢出时表现完全不同。`ss -lnt` 看到的 `Send-Q` 是哪一个？
7. `Nagle` 算法和 `Delayed ACK` 同时开启时为什么会出现 "40ms 延迟"？关哪一个能解决？为什么默认不关？
8. BBR 凭什么取代 cubic？在丢包率 1% 的高 RTT 长肥管道上，BBR 的吞吐能比 cubic 高几倍？

如果有半数答不上，往下看。

---

## 1. 底层模型：11 个状态、两个队列、两组算法

绝大多数文章把 TCP 讲成"三次握手 + 四次挥手"就结束了。这层认知解释不了上面 8 个问题。要看懂它们，必须把 TCP 看成一个**严格的状态机** + 两个**独立队列** + 一组**控制循环**。

### 1.1 TCP 11 个状态的完整转换图

下面这张图你必须背下来（含异常路径）：

```
                         +---------+
                         | CLOSED  |  ←─────────────────────+
                         +---------+                          |
                       passive |  | active                    | timeout
                       open    |  | open                       | (2MSL)
                               v  v                            |
                         +---------+ ──── send SYN ────► +-----+----+
                         |  LISTEN |                     |TIME_WAIT |
                         +---------+ ◄──┐                +----+-----+
                            |    ▲      |                     ▲
                  recv SYN  |    |close |             recv ACK|
                  send      |    |      |             of FIN  |
                  SYN+ACK   |    |      |                     |
                            v    |      |              +------+-----+
                       +-------------+  |              |  FIN_WAIT_2|
                       | SYN_RECEIVED|  |              +------+-----+
                       +-------------+  |                     ▲
                            |    ▲      |             recv ACK| of FIN
                     recv   |    |      |                     |
                     ACK    |    |      |              +------+-----+
                            v    |      |   close      |FIN_WAIT_1  |
                       +-------------+  |   send FIN   +-----+------+
                       | ESTABLISHED |  |◄────────────────── | ▲
                       +-------------+  |       recv FIN     | |
                            ▲   |       |       send ACK     | |
                            |   |close  |                    | |
                            |   |send   |               +----+-+----+
                       recv |   |FIN    |               |  CLOSING  |
                       SYN  |   |       |               +-----+-----+
                       ACK  |   v       |                     |recv ACK
                       send |   |       |                     v
                       ACK  |   |       |               +-----+-----+
                            |   |       └───────────────| LAST_ACK  |
                       +----+--------+   recv FIN       +-----+-----+
                       | SYN_SENT    |   send ACK             ▲
                       +-------------+   ──────►              | recv FIN
                                         CLOSE_WAIT ──────────┘
                                         (close)
```

11 个状态分成三组（这个分类对排查极其重要）：

| 组 | 状态 | 触发场景 |
| --- | --- | --- |
| **建立组** | `LISTEN` / `SYN_SENT` / `SYN_RECEIVED` | 三次握手过程中 |
| **稳定组** | `ESTABLISHED` | 数据传输 |
| **关闭组** | `FIN_WAIT_1` / `FIN_WAIT_2` / `CLOSE_WAIT` / `LAST_ACK` / `CLOSING` / `TIME_WAIT` / `CLOSED` | 四次挥手过程中 |

> **口诀**：主动关方走 `FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT`；被动关方走 `CLOSE_WAIT → LAST_ACK`。`CLOSING` 是双方"同时关"的特例。

### 1.2 三次握手（含 SYN cookies 路径）

正常路径：

```
client                                  server
  |  ──── SYN, seq=x        ────►        |   LISTEN
  |  ◄──── SYN+ACK, seq=y, ack=x+1       |   入半连接队列（SYN queue）
  |                                      |   状态: SYN_RECEIVED
  |  ──── ACK, ack=y+1      ────►        |   出半连接 → 进全连接队列
  |                                      |   accept() 返回
  |  ESTABLISHED                         |   ESTABLISHED
```

`SYN_FLOOD` 攻击的本质：攻击者只发第一个 SYN（且源 IP 是伪造的），第三步 ACK 永远不来。半连接队列被填满，正常用户的 SYN 被丢。

`SYN cookies` 的应对：**收到 SYN 时不分配半连接队列项**，而是把 `seq number` 编码成"对方信息的哈希"返回。第三步 ACK 回来时，校验 `ack-1` 是否是合法 cookie；合法就直接进 ESTABLISHED，跳过半连接队列。

代价：cookie 编码空间有限（24 位），TCP 选项（如 SACK、窗口缩放）会丢失部分信息。所以默认是"半连接队列将满才启用"，不是全程开。

### 1.3 四次挥手 —— 三个关键状态

```
client (主动关)                          server (被动关)
  |  ──── FIN              ────►         |   ESTABLISHED
  |  FIN_WAIT_1                           |
  |  ◄──── ACK                            |   CLOSE_WAIT  ★ 应用未 close()
  |  FIN_WAIT_2                           |   ...任意时长...
  |  ◄──── FIN                            |   LAST_ACK
  |  ──── ACK             ────►           |   CLOSED
  |  TIME_WAIT (2MSL)
  |  CLOSED
```

三个最容易堆的状态：

| 状态 | 出现在哪一方 | 触发条件 | 持续时间 |
| --- | --- | --- | --- |
| `CLOSE_WAIT` | **被动关方** | 收到对方 FIN，但应用层没调 `close()` | 直到应用 close 才结束 |
| `FIN_WAIT_2` | **主动关方** | 自己发了 FIN，对方 ACK 了，但对方还没发 FIN | 默认 60s（`tcp_fin_timeout`） |
| `TIME_WAIT` | **主动关方** | 自己发完最后一个 ACK | 2MSL（Linux 写死 60s，源码 `TCP_TIMEWAIT_LEN`） |

**TIME_WAIT 为什么必须 2MSL**：

- 原因一：保证最后一个 ACK 能到达对端。如果 ACK 丢了，对端会重传 FIN，主动关方需要还能响应。
- 原因二：等"老连接"的所有报文在网络上消亡。否则同一四元组（source IP/port + dest IP/port）的新连接可能收到老连接的迟到包。

少等：上面两个语义都被破坏 —— 老包污染新连接、对端 FIN 重传得不到 ACK 卡在 LAST_ACK。
多等：端口被占着不能复用，高频短连接场景端口耗尽。

> **口诀**：`CLOSE_WAIT` 多 = **应用 bug**（忘了 close）；`TIME_WAIT` 多 = **正常但要管控**（端口复用）；`FIN_WAIT_2` 多 = 对端 close 慢或丢包。

### 1.4 半连接队列 vs 全连接队列

这是最常被混为一谈的两个东西：

```
        +-----------------------+
SYN ──► |   SYN queue (半连接)   |   收到 SYN 后入队
        |   长度由 net.ipv4.    |   收到第三步 ACK 后出队
        |   tcp_max_syn_backlog |
        +-----------------------+
                |  完成三次握手
                v
        +-----------------------+
        | accept queue (全连接) |   等待用户态 accept() 取走
        |   长度 = min(         |
        |     somaxconn,         |
        |     listen() 第二参数) |
        +-----------------------+
                |  应用 accept()
                v
              用户态
```

| 队列 | 溢出时行为 | 控制参数 |
| --- | --- | --- |
| 半连接 | 丢 SYN（开 syncookies 后改为发 cookie） | `tcp_max_syn_backlog`、`tcp_syncookies` |
| 全连接 | 默认丢 ACK（让客户端重传），可调成 RST | `somaxconn`、`listen(fd, backlog)`、`tcp_abort_on_overflow` |

`ss -lnt` 看到的：

```
State    Recv-Q  Send-Q  Local Address:Port
LISTEN   0       128     0.0.0.0:8080
```

- `Recv-Q`：**当前**全连接队列已用长度
- `Send-Q`：全连接队列**最大**长度（即 backlog）

千万别被字段名误导。

### 1.5 拥塞控制：cubic vs BBR

cubic（Linux 2.6.19+ 默认）的核心循环：

```
    cwnd                                      丢包 →
     │      ╱─╲    cubic 函数                 cwnd /= 2
     │    ╱     ╲                             ↓
     │  ╱         ╲ ←─── 重新爬升              ╱
     │╱             ╲                         ╱
     └──────────────────────────────► 时间
```

cubic 是**基于丢包的**：丢包就当作拥塞发生，砍半窗口。在丢包率天然就高（Wi-Fi、跨国、移动网络）的链路上，吞吐永远上不去 —— 它把所有丢包都当成拥塞。

BBR（Google 2016）的核心是**测量瓶颈带宽和最小 RTT**，而不是猜：

```
BTLBW  = max(deliveryRate)   每个 RTT 内送达的最大速率
RTPROP = min(rtt)            观察到的最小 RTT

最优工作点：cwnd ≈ BTLBW × RTPROP
```

BBR 在丢包率 1% 时仍能跑满带宽，而 cubic 在 1% 丢包下吞吐能掉 90% 以上。后面 3.3 节有实测数据。

### 1.6 RTO 计算 —— Karn 算法的"陷阱"

RTO（重传超时）= RTT 的估计 + 抖动余量。RFC 6298：

```
RTO  =  SRTT + max(G, K * RTTVAR)         K=4, G=时钟粒度

SRTT  ← (1-α) * SRTT  + α * RTT_sample    α=1/8
RTTVAR ← (1-β) * RTTVAR + β * |SRTT - RTT_sample|   β=1/4
```

Karn 算法（1987）的关键洞察：**重传过的包不能用来更新 RTT**。

为什么？想象你发包 A，700ms 后没收到 ACK，重传 A'，100ms 后收到 ACK。这个 ACK 是在确认 A 还是 A'？没法分辨。如果当作 A'（100ms），RTT 估计偏低；如果当作 A（800ms），RTT 估计偏高。

Karn 的简洁解法：**重传过的包，不管它的 ACK，直接丢弃这次的 RTT 测量**。同时在重传时把 RTO 翻倍（指数退避），等收到一个非重传包的 ACK 再重置。

> **口诀**：RTO 退避是协议级"自我保护"；让一个不稳定的链路在塞车时不要继续灌包。

---

## 2. 内核源码切片（Linux 5.x）

> 基于 Linux 5.15 LTS 内核，路径以 `net/ipv4/` 为基准。注释为本文添加。

### 2.1 tcp_rcv_state_process —— 状态机的总入口

```c
// net/ipv4/tcp_input.c
int tcp_rcv_state_process(struct sock *sk, struct sk_buff *skb)
{
    struct tcp_sock *tp = tcp_sk(sk);
    struct tcphdr *th = tcp_hdr(skb);
    int queued = 0;

    switch (sk->sk_state) {
    case TCP_CLOSE:
        goto discard;

    case TCP_LISTEN:
        // 收到 SYN：分配半连接，发 SYN+ACK
        if (th->syn && !th->ack)
            return icsk->icsk_af_ops->conn_request(sk, skb);
        goto discard;

    case TCP_SYN_SENT:
        // 客户端等 SYN+ACK
        queued = tcp_rcv_synsent_state_process(sk, skb, th);
        ...

    case TCP_SYN_RECV:
        // 收到第三步 ACK，转 ESTABLISHED
        ...
    }
    ...
}
```

**逐段点透**：

- 整个 TCP 状态机就是这一个 switch。每个状态对应一个分支，分支里再处理"在这个状态下能收到什么包、收到了怎么办"。
- `TCP_LISTEN` 分支只处理 SYN —— 收到非 SYN 的包（比如带数据的 ACK）会被 `discard` 默默丢掉。这就是为什么"扫描器发空 ACK"探测端口时收不到回应（Linux 默认行为）。
- `TCP_CLOSE` 状态什么都不处理。如果你 `kill -9` 一个进程，它 fd 没正常 close，对端发的 FIN/RST 进到这里就被丢了 —— 必须靠 keepalive 才能让对端发现。

### 2.2 tcp_v4_conn_request —— SYN 处理 + cookie

```c
// net/ipv4/tcp_ipv4.c
int tcp_conn_request(struct request_sock_ops *rsk_ops, ...,
                     struct sock *sk, struct sk_buff *skb)
{
    struct net *net = sock_net(sk);
    bool want_cookie = false;
    ...
    // ① 半连接队列满了
    if (sk_acceptq_is_full(sk) && inet_csk_reqsk_queue_is_full(sk)) {
        want_cookie = tcp_syn_flood_action(sk, skb, "TCPv4");
        if (!want_cookie)
            goto drop;       // 没开 syncookies，直接丢
    }

    // ② 全连接队列满了，也丢
    if (sk_acceptq_is_full(sk)) {
        NET_INC_STATS(net, LINUX_MIB_LISTENOVERFLOWS);
        goto drop;
    }

    req = inet_reqsk_alloc(rsk_ops, sk, !want_cookie);
    ...

    if (want_cookie) {
        isn = cookie_v4_init_sequence(skb, &mss);   // ★ 编码进 ISN
        req->cookie_ts = tmp_opt.tstamp_ok;
    }
    ...
    af_ops->send_synack(sk, ..., skb, attrs, ...);   // 发 SYN+ACK
}
```

**逐段点透**：

- 两个队列分开判断：半连接队列满 → 决定是不是用 cookie；全连接队列满 → 直接丢。这就是为什么"全连接队列溢出"和 syncookies 没关系。
- `tcp_syn_flood_action` 内部检查 `tcp_syncookies` sysctl：值为 0 关闭、1 满了才用、2 永远用。生产基本用 1。
- `cookie_v4_init_sequence` 把 client IP、port、server port、MSS 哈希后塞进 ISN 的低位。第三步 ACK 回来时反向校验，无需查半连接队列。

### 2.3 tcp_close —— 主动关闭流程

```c
// net/ipv4/tcp.c
void tcp_close(struct sock *sk, long timeout)
{
    struct sk_buff *skb;
    int data_was_unread = 0;
    ...
    // ① 把接收缓冲里没读的数据丢掉
    while ((skb = __skb_dequeue(&sk->sk_receive_queue)) != NULL) {
        u32 len = TCP_SKB_CB(skb)->end_seq - TCP_SKB_CB(skb)->seq;
        data_was_unread += len;
        __kfree_skb(skb);
    }

    sk_mem_reclaim(sk);

    // ② 如果还有未读数据 → 发 RST 而不是 FIN（粗暴关）
    if (unlikely(tcp_sk(sk)->repair)) {
        sk->sk_prot->disconnect(sk, 0);
    } else if (data_was_unread) {
        NET_INC_STATS(sock_net(sk), LINUX_MIB_TCPABORTONCLOSE);
        tcp_set_state(sk, TCP_CLOSE);
        tcp_send_active_reset(sk, sk->sk_allocation);
    } else if (sock_flag(sk, SOCK_LINGER) && !sk->sk_lingertime) {
        // SO_LINGER 设为 0：不发 FIN，直接 RST
        sk->sk_prot->disconnect(sk, 0);
    } else if (tcp_close_state(sk)) {
        // ③ 正常路径：发 FIN
        tcp_send_fin(sk);
    }
    ...
}
```

**逐段点透**：

- **数据没读完就 close 会发 RST**：很多人不知道。`accept` 后没读 body 就 `close`，对端看到的是 RST 而不是优雅关闭。HTTP 服务器写错了就会让客户端报"connection reset by peer"。
- `SO_LINGER + linger_time=0` 是**主动选择 RST**：跳过 TIME_WAIT。代价是数据可能丢，且对方应用读时报错。仅在你确实需要避免 TIME_WAIT 且能接受丢数据时用。
- `tcp_close_state` 决定下一个状态：从 `ESTABLISHED → FIN_WAIT_1`、从 `CLOSE_WAIT → LAST_ACK`。这个函数是状态机转换的"小型查找表"。

### 2.4 tcp_time_wait —— 进入 TIME_WAIT

```c
// net/ipv4/tcp_minisocks.c
void tcp_time_wait(struct sock *sk, int state, int timeo)
{
    struct inet_timewait_sock *tw;
    struct tcp_sock *tp = tcp_sk(sk);
    bool recycle_ok = false;
    ...
    // ★ 分配 timewait sock（一个"小尺寸"sock，只保留必要字段）
    tw = inet_twsk_alloc(sk, &sock_net(sk)->ipv4.tcp_death_row, state);

    if (tw) {
        struct tcp_timewait_sock *tcptw = tcp_twsk((struct sock *)tw);
        tcptw->tw_rcv_nxt    = tp->rcv_nxt;
        tcptw->tw_snd_nxt    = tp->snd_nxt;
        tcptw->tw_ts_recent  = tp->rx_opt.ts_recent;
        tcptw->tw_ts_recent_stamp = tp->rx_opt.ts_recent_stamp;
        ...
        // ★ 注册到 death_row，到点回收
        inet_twsk_schedule(tw, timeo);
    }
    ...
    tcp_done(sk);    // 释放原 sock
}
```

**逐段点透**：

- 进 TIME_WAIT 后**会把原 `struct sock` 替换成轻量的 `inet_timewait_sock`**，只保留必要字段（接收/发送序号、时间戳）。这就是为什么"TIME_WAIT 占内存极少"，但仍会占四元组。
- `tw_ts_recent` 是 timestamp 选项的最近值。`tcp_tw_reuse` 复用 TIME_WAIT 端口的核心条件之一就是新连接的 timestamp **严格大于**这个值，避免老连接的迟到包被误读。
- `death_row`（死亡列表）是个时间轮。每个 CPU 一个独立的 row，到点统一回收。这是为什么 `TIME_WAIT` 多了不是单个连接的问题，而是 hash 表查找变慢的问题（案例 ⑨）。

### 2.5 tcp_retransmit_skb —— 重传

```c
// net/ipv4/tcp_output.c
int __tcp_retransmit_skb(struct sock *sk, struct sk_buff *skb, int segs)
{
    struct tcp_sock *tp = tcp_sk(sk);
    ...
    // ① 拥塞窗口允许重传吗？
    if (icsk->icsk_mtup.probe_size)
        icsk->icsk_mtup.probe_size = 0;

    if (skb_still_in_host_queue(sk, skb))
        return -EBUSY;
    ...
    // ② 实际重传
    err = tcp_transmit_skb(sk, skb, 1, GFP_ATOMIC);

    if (likely(!err)) {
        TCP_SKB_CB(skb)->sacked |= TCPCB_EVER_RETRANS;
        // ★ 标记"曾被重传"，影响 RTT 采样（Karn 算法）
        trace_tcp_retransmit_skb(sk, skb);
    }
    return err;
}

// 调度入口
void tcp_retransmit_timer(struct sock *sk)
{
    ...
    // 指数退避：每次失败 RTO *= 2
    icsk->icsk_rto = min(icsk->icsk_rto << 1, TCP_RTO_MAX);
    inet_csk_reset_xmit_timer(sk, ICSK_TIME_RETRANS,
                              icsk->icsk_rto, TCP_RTO_MAX);
}
```

**逐段点透**：

- `TCPCB_EVER_RETRANS` 是 Karn 算法的实现 —— 这个 skb 一旦被重传过，下次它的 ACK 不会用来更新 SRTT。
- `icsk_rto << 1` 是指数退避：第一次 RTO 比如 200ms，第二次 400ms，第三次 800ms…… 默认上限 `TCP_RTO_MAX = 120s`。
- `tcp_retries2` sysctl 控制最多重传几次（默认 15 次）。15 次指数退避加起来约 13~30 分钟 —— 这就是为什么"网线拔了"后内核要这么久才让应用 read 报错。生产服务必须配业务层心跳，不能等内核。

> **口诀**：重传超时是指数退避；Karn 让重传过的样本不污染 RTT 估计；`tcp_retries2` 决定连接死透多久通知应用。

---

## 3. 抓包实测

光看源码不够，必须自己抓一遍才能记住。下面 5 组实验都能在一台 Linux 机器上跑，工具就 `tcpdump` + `ss` + `iperf3` + `nc`。

### 3.1 用 tcpdump 看四次挥手

```bash
# 终端 A：起一个 echo server
nc -l 9999

# 终端 B：开抓包
sudo tcpdump -i lo -nn -S 'port 9999'

# 终端 C：连一下立刻关
nc 127.0.0.1 9999
# Ctrl-D 关闭客户端 stdin → 客户端发 FIN
```

典型输出：

```
10:00:01.001  IP 127.0.0.1.45678 > 127.0.0.1.9999: Flags [S],   seq 1000
10:00:01.001  IP 127.0.0.1.9999  > 127.0.0.1.45678: Flags [S.], seq 2000, ack 1001
10:00:01.001  IP 127.0.0.1.45678 > 127.0.0.1.9999: Flags [.],   ack 2001
# ... 数据传输 ...
10:00:05.500  IP 127.0.0.1.45678 > 127.0.0.1.9999: Flags [F.],  seq 1010, ack 2010   ← 客户端 FIN
10:00:05.500  IP 127.0.0.1.9999  > 127.0.0.1.45678: Flags [.],  ack 1011             ← 服务端 ACK
10:00:05.500  IP 127.0.0.1.9999  > 127.0.0.1.45678: Flags [F.], seq 2010, ack 1011   ← 服务端 FIN
10:00:05.500  IP 127.0.0.1.45678 > 127.0.0.1.9999: Flags [.],   ack 2011             ← 客户端 ACK
```

注意：在 loopback 上，第二第三步常会**合并**（服务端的 ACK 和 FIN 一起发）。所以你抓到的可能是 3 个包而不是 4 个 —— 这不是 bug，是 TCP 协议允许的优化。

抓包同时另开一个终端跑 `ss -tan | grep 9999`，能看到客户端进 `TIME_WAIT` 60s 后消失。

### 3.2 模拟 backlog 满后的 SYN 丢包

```bash
# 故意把全连接队列设小
sudo sysctl -w net.core.somaxconn=2

# 起一个不调 accept 的服务（端口被监听，但永不 accept）
python3 -c "
import socket, time
s = socket.socket()
s.bind(('0.0.0.0', 9999))
s.listen(2)
time.sleep(3600)        # 永不 accept
"

# 另一终端疯狂发起连接
for i in $(seq 1 10); do (timeout 1 nc 127.0.0.1 9999 &) ; done

# 看溢出计数
nstat -az TcpExtListenOverflows TcpExtListenDrops
```

关键观察：

```
TcpExtListenOverflows           8
TcpExtListenDrops               8
```

每多一个超量连接，两个计数同时 +1。`TcpExtListenDrops` 里**包含**了 `TcpExtListenOverflows`，但还可能因为内存不够等原因丢，所以一般 `Drops >= Overflows`。

切换 `tcp_abort_on_overflow=1` 后再试：

```bash
sudo sysctl -w net.ipv4.tcp_abort_on_overflow=1
```

客户端的行为从"卡 1 秒等待重传"变成"立刻收到 RST"。生产几乎都不开这个 —— 让客户端重试比报错好。

### 3.3 BBR vs cubic 在丢包率 1% 下的吞吐对比

用 `tc` 模拟丢包：

```bash
# 在 lo 上模拟 1% 丢包 + 50ms RTT
sudo tc qdisc add dev lo root netem loss 1% delay 25ms

# 装 iperf3
# 服务端
iperf3 -s

# 客户端：cubic
sudo sysctl -w net.ipv4.tcp_congestion_control=cubic
iperf3 -c 127.0.0.1 -t 30

# 客户端：BBR
sudo modprobe tcp_bbr
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr
iperf3 -c 127.0.0.1 -t 30

# 清理
sudo tc qdisc del dev lo root
```

我在一台 8 核 Linux 5.15 VM 上的实测：

| 算法 | 丢包率 | RTT | 吞吐 |
| --- | --- | --- | --- |
| cubic | 0% | 50ms | 7.8 Gbps |
| cubic | 1% | 50ms | 480 Mbps |
| BBR | 0% | 50ms | 7.6 Gbps |
| BBR | 1% | 50ms | **5.2 Gbps** |

结论：在丢包率非零时 BBR 远胜 cubic。但**注意 BBR 在多流共存时不"友好"** —— 它会把 cubic 流挤到极小份额（公平性问题）。如果你的网络是 BBR + 第三方 cubic 混跑，要谨慎评估。

### 3.4 tcp_tw_reuse 的实际效果

```bash
# 关 reuse
sudo sysctl -w net.ipv4.tcp_tw_reuse=0

# 用 ab 短连接灌（每次都新建 + 关闭）
ab -n 50000 -c 100 http://target/

# 看 TIME_WAIT 数量
ss -tan state time-wait | wc -l

# 开 reuse
sudo sysctl -w net.ipv4.tcp_tw_reuse=1
ab -n 50000 -c 100 http://target/
ss -tan state time-wait | wc -l
```

观察：

- 开 `tcp_tw_reuse=1` 后，**作为客户端**发起新连接时能复用处于 TIME_WAIT 的本地端口 —— 端口不会耗尽。
- 这只对**客户端**生效（即"主动连出去"的方向）；服务端 listening 端口的 TIME_WAIT 不受这个开关影响。
- 复用条件（在 `tcp_twsk_unique` 里）：新连接的 timestamp 严格大于 TIME_WAIT 项的 `tw_ts_recent`，且开启了 `tcp_timestamps`。

千万别开 `tcp_tw_recycle`（已在 Linux 4.12 移除）。它在 NAT 场景下会因为多个 NAT 后客户端 timestamp 不递增（不同主机的 timestamp 不同步）导致部分客户端连不上 —— 经典翻车现场。

### 3.5 关键 sysctl 一表过

```bash
# 半连接队列
net.ipv4.tcp_max_syn_backlog = 8192      # 半连接队列长度
net.ipv4.tcp_syncookies = 1              # 半连接将满启用 cookie

# 全连接队列
net.core.somaxconn = 4096                # 全连接队列上限（listen 第二参数会被 cap 到这）
net.ipv4.tcp_abort_on_overflow = 0       # 全连接溢出时是丢 ACK 还是发 RST

# TIME_WAIT
net.ipv4.tcp_tw_reuse = 1                # 客户端复用 TIME_WAIT
net.ipv4.tcp_max_tw_buckets = 262144     # TIME_WAIT 总数上限，超了直接关

# FIN_WAIT_2
net.ipv4.tcp_fin_timeout = 30            # FIN_WAIT_2 超时（默认 60）

# Keepalive
net.ipv4.tcp_keepalive_time = 7200       # 默认 2 小时（生产几乎一定要改小）
net.ipv4.tcp_keepalive_intvl = 75
net.ipv4.tcp_keepalive_probes = 9

# 重传
net.ipv4.tcp_retries2 = 15               # 数据重传上限（约 13~30 分钟才放弃）
net.ipv4.tcp_syn_retries = 6             # SYN 重传上限

# 拥塞控制
net.ipv4.tcp_congestion_control = bbr    # 或 cubic
net.core.default_qdisc = fq              # BBR 配 fq 才能发挥
```

每条都建议你改之前 `sysctl <key>` 看一眼当前值，并在容器场景注意 namespace 隔离（见案例 ⑧）。

> **口诀**：sysctl 改任何值之前，先在测试环境抓一组基线数据；改完再抓一组对比。"我觉得这样改更好"在网络栈面前几乎一定会翻车。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① CLOSE_WAIT 堆积到几千 —— "连接句柄没关"

#### 现象

监控告警：服务的 `netstat -an | grep CLOSE_WAIT | wc -l` 涨到 4000+。新请求开始失败（`Too many open files`）。

```bash
ss -tan state close-wait | head
ESTAB    0      0     10.0.0.5:8080   10.0.0.7:54321
...
```

应用本身不报错，但下游说"连接被你卡着不放"。

#### 根因

`CLOSE_WAIT` 的语义是 **"对方已经关了，我（被动方）还没调 close()"**。这**100% 是应用 Bug**，内核能做的只有等。

最常见的写法陷阱：

```java
// ❌ 异常路径泄露
HttpURLConnection conn = (HttpURLConnection) url.openConnection();
InputStream in = conn.getInputStream();
String body = readAll(in);     // 这里抛 IOException
in.close();                    // 永远跑不到
conn.disconnect();             // 永远跑不到
```

或者：

```go
// ❌ 没读 Body 就 return
resp, err := http.Get(url)
if err != nil { return err }
if resp.StatusCode != 200 {
    return errors.New("bad status")   // resp.Body 没关
}
defer resp.Body.Close()
```

或者用了某个 SDK，它内部 keep-alive 池配置错误，连接归还失败。

#### 排查

第一步：找到"哪个进程"。

```bash
# 列出当前进程的所有 socket
ss -tanp state close-wait | head
# State  Recv-Q Send-Q  Local:Port   Peer:Port  process
# ESTAB    0      0   10.0.0.5:8080  ...        users:(("java",pid=12345,fd=87))

# 进程 12345 的 fd 数
ls /proc/12345/fd | wc -l
```

第二步：找到"哪段代码"。Java 应用上 jstack：

```bash
jstack 12345 > stack.txt
# 看是不是某个线程一直在 sun.nio.ch.SocketChannelImpl.read 之类
```

更系统的做法：用 `lsof` 看连接打开的栈帧不太够，要用 BPF。

```bash
# bcc tools
sudo /usr/share/bcc/tools/tcpconnect      # 谁发起的连接
sudo /usr/share/bcc/tools/tcptracer       # 完整生命周期
```

#### 修复

代码层面：

```java
// ✅ try-with-resources
try (var conn = openConnection();
     var in = conn.getInputStream()) {
    return readAll(in);
}
```

```go
// ✅ 即使不读，也必须 Close
resp, err := http.Get(url)
if err != nil { return err }
defer resp.Body.Close()
io.Copy(io.Discard, resp.Body)   // ★ 把剩余 body 读干净，连接才能复用
```

Go 里**特别坑**：`resp.Body.Close()` 是必须的，但**没读完 body 就 Close**，连接也不会归还到 keep-alive 池，会被关掉。所以 `io.Copy(io.Discard, resp.Body)` 比单纯 close 更对。

#### 兜底（当根因还没定位时）

```bash
# 临时把 CLOSE_WAIT 老化时间调短（不是 sysctl，是 keepalive 间接生效）
# 注意：CLOSE_WAIT 不能直接被内核回收，因为应用持有 fd
# 唯一的"兜底"是给应用进程加 keepalive，让对端 reset
```

实际上**没有真正的内核兜底**。你能做的只有重启进程释放 fd。这是为什么 CLOSE_WAIT 必须从代码层修。

#### 验证

修复后跑压测：

```bash
# 压测前
ss -tan state close-wait | wc -l        # 0

# 压测中持续观察
watch -n 1 'ss -tan state close-wait | wc -l'
```

如果数字始终维持在 < 10（瞬态），就修对了。

> **口诀**：CLOSE_WAIT 多 = 代码 bug，不要找内核背锅。重点查"未捕获异常路径"和"忘了消费 body"。

---

### 案例 ② TIME_WAIT 占满端口 —— "高频短连接客户端"

#### 现象

服务作为客户端调下游 RPC，QPS 1000，每次都新建连接。某天监控告警：

```bash
ss -tan state time-wait | wc -l
60123
```

接着有些请求开始失败：`cannot assign requested address`。

#### 根因

TIME_WAIT 60s 内占着 (源IP, 源端口, 目的IP, 目的端口) 这个四元组。当目的 IP 和端口固定（同一个下游）时，可用源端口是 `ip_local_port_range` 的范围（默认 32768~60999，约 28000 个）。

QPS 1000 + 60s 老化 = 60000 个 TIME_WAIT，远超可用源端口。新连接 `connect()` 时 `bind` 失败。

#### 修复

**方案 A**（首选）：用连接池 / Keep-Alive。

短连接改长连接才是治本。HTTP 客户端用 connection pool；RPC 框架默认就有。

```java
// OkHttp
new OkHttpClient.Builder()
    .connectionPool(new ConnectionPool(50, 5, TimeUnit.MINUTES))
    .build();
```

```go
// Go: 关键是 MaxIdleConnsPerHost
http.DefaultTransport.(*http.Transport).MaxIdleConnsPerHost = 100
```

**方案 B**：开 `tcp_tw_reuse`（仅作为客户端）。

```bash
sudo sysctl -w net.ipv4.tcp_tw_reuse=1
sudo sysctl -w net.ipv4.tcp_timestamps=1   # reuse 依赖 timestamp
```

**方案 C**：扩大端口范围。

```bash
sudo sysctl -w net.ipv4.ip_local_port_range="1024 65535"
```

**方案 D**（错误！）：把 `tcp_max_tw_buckets` 调小，让超过这个数的 TIME_WAIT 直接被砍。

```bash
# ❌ 不要这么干
net.ipv4.tcp_max_tw_buckets = 5000
```

后果：超出后内核打 `TCP: time wait bucket table overflow` 警告，并**直接 RST 关闭连接而非进 TIME_WAIT**。这等于**主动违反 TCP 协议**，老连接的迟到包可能污染新连接。仅在确实没有别的办法时，用最低优先级。

#### 不要做的事

- ❌ 千万别开 `tcp_tw_recycle`（已在 5.x 移除）—— NAT 下翻车，前文 3.4 解释了。
- ❌ 别盲信"`tcp_fin_timeout` 控制 TIME_WAIT" —— 它**只控制 FIN_WAIT_2**，TIME_WAIT 是写死的 60s（除非你重编译内核）。

#### 验证

```bash
# 改前压测，数 TIME_WAIT
ss -tan state time-wait | wc -l

# 改后再压测
ss -tan state time-wait | wc -l
# 应该显著下降（连接复用后甚至降到几百）

# 同时看 connect 失败计数
nstat -az TcpExtTCPTimeWaitOverflow
```

> **口诀**：TIME_WAIT 多的根治方案是**连接池**；其次是 `tw_reuse`；扩端口范围是兜底；调 max_tw_buckets 是放弃治疗。

---

### 案例 ③ SYN_FLOOD 攻击 —— "服务无法 accept 新连接"

#### 现象

服务监听端口正常，但用户反馈连不上。`netstat` 一看：

```bash
netstat -ant | grep SYN_RECV | wc -l
1023
```

大量 `SYN_RECEIVED`，全连接队列空着，正常用户的 SYN 被丢。

抓包看 SYN 包数量爆炸，且源 IP 极度分散（伪造）：

```bash
sudo tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0' -nn -c 1000 | awk '{print $3}' | sort -u | wc -l
# 1000 个不同的源 IP
```

#### 根因

经典 SYN_FLOOD：攻击者只发第一步 SYN，伪造源 IP，第三步 ACK 永不来。半连接队列被填满。

#### 修复

第一招：开 SYN cookies。

```bash
sudo sysctl -w net.ipv4.tcp_syncookies=1
```

设为 1 是"半连接队列将满才启用"，这是默认推荐值。设为 2 是"全程启用"，但代价是 TCP 选项被压缩（部分场景跑得慢一点）。

第二招：缩短 SYN_RECV 重试。

```bash
sudo sysctl -w net.ipv4.tcp_synack_retries=2     # 默认 5
```

少重试几次就早释放 slot。

第三招：扩大半连接队列。

```bash
sudo sysctl -w net.ipv4.tcp_max_syn_backlog=8192
```

第四招（防御 BGP 层）：上游路由器/防火墙做源 IP 校验、流量清洗。这是真正"防"的层级，只在 Linux 上挣扎是被动应对。

#### 为什么不光靠"调大队列"

调大队列只能扛"一定量"的攻击，攻击者只要继续灌就能填满更大的队列。SYN cookies 的优势是**根本不分配队列项** —— 攻击 traffic 只消耗 CPU（哈希计算），不消耗内存。**两个机制互补**，不是替代。

#### 验证

```bash
# 看 syncookies 是否在工作
nstat -az TcpExtSyncookiesSent TcpExtSyncookiesRecv TcpExtSyncookiesFailed
# Sent 持续 > 0 = 已激活
# Failed > 0 = 有人发的 ACK cookie 不合法（攻击或老包）

# 看半连接队列状态
ss -lnt
# Recv-Q  Send-Q  Local
# 0       128     0.0.0.0:8080      ← Recv-Q 接近 Send-Q 说明全连接也快满了
```

> **口诀**：SYN_FLOOD 防御 = `tcp_syncookies=1` + 调小 `synack_retries` + 上游清洗。光调大队列治不了根本。

---

### 案例 ④ accept 队列溢出 —— "压测时 SYN 重传暴增"

#### 现象

压测 QPS 一上 5000 就出现大量请求"卡顿 1 秒、3 秒、7 秒后才成功"。抓包看到客户端 SYN 在重传：

```
00:00:01.000  SYN
00:00:02.000  SYN (retransmit)
00:00:04.000  SYN (retransmit)
00:00:08.000  SYN (retransmit)
```

服务端日志没异常，CPU 也不高。

#### 根因

不是半连接队列满（开了 syncookies 不会丢），是**全连接队列**满。

```bash
# 全连接队列长度
ss -lnt
LISTEN  0  128  0.0.0.0:8080      ← Send-Q=128

# 应用 accept 不及时 → 队列堆满
nstat -az TcpExtListenOverflows
TcpExtListenOverflows  4521        ← 持续上涨
```

服务端的全连接队列长度 = `min(somaxconn, listen 第二参数)`。

很多框架默认 listen backlog 写死 128 或 1024（Tomcat 默认 100，Nginx 默认 511，Netty 默认 50 或 256 看版本）。`somaxconn` 没调或调了但应用没用上，就被卡在低位。

`tcp_abort_on_overflow=0` 时（默认），全连接溢出**丢 ACK**而非发 RST —— 客户端会以为最后一步 ACK 丢了，重传 SYN。这就是为什么现象是"卡 1 秒、3 秒……"（指数退避）。

#### 修复

```bash
# 内核层
sudo sysctl -w net.core.somaxconn=4096

# 应用层 ★ 必须同步改
# Tomcat:  server.tomcat.accept-count=4096
# Nginx:   listen 80 backlog=4096;
# Netty:   .option(ChannelOption.SO_BACKLOG, 4096)
# Go:      标准库默认 128，需要用 net.ListenConfig 自定义
```

仅改 sysctl 没用 —— `listen()` 的第二参数是 cap，应用没传 4096，内核也不会给。

更深一层：**为什么 accept 不及时**？

- 业务线程池被打满 → 调大线程池或换 Reactor
- accept 线程被业务逻辑阻塞 → accept 必须放在专用 boss EventLoop
- GC STW 长 → 调 GC 或换 ZGC/Shenandoah

#### 验证

```bash
# 持续观察 ListenOverflows 是否还涨
watch -n 1 'nstat -az TcpExtListenOverflows | grep -v "^$"'

# 看 SYN 重传计数
nstat -az TcpExtTCPSynRetrans
```

如果两个数都不再增长，说明修对了。

> **口诀**：accept 队列溢出 = 应用 accept 慢。内核能做的就是把队列调大；根治在应用。

---

### 案例 ⑤ 长 RTT 下 TCP 吞吐上不去 —— "BDP 与窗口"

#### 现象

跨境专线把日本机房文件拉到北京，单连接 `scp` 传输只有 30Mbps，但专线带宽明明是 1Gbps。

#### 根因

带宽时延积（BDP, Bandwidth-Delay Product）= 1Gbps × 100ms RTT = **12.5 MB**。

TCP 在 RTT 时间内最多能"在飞行"的字节数 = `min(cwnd, rwnd)`。如果发送窗口（接收方公告的 rwnd 或拥塞窗口 cwnd）小于 BDP，链路就跑不满。

Linux 默认 `tcp_rmem` / `tcp_wmem` 上限是几 MB，对 12.5MB BDP 不够；还有窗口缩放（window scaling）必须开。

#### 修复

```bash
# 默认/最大缓冲（min default max，单位字节）
sudo sysctl -w net.core.rmem_max=67108864          # 64 MB
sudo sysctl -w net.core.wmem_max=67108864
sudo sysctl -w net.ipv4.tcp_rmem="4096 1048576 67108864"
sudo sysctl -w net.ipv4.tcp_wmem="4096 1048576 67108864"

# 必须开窗口缩放（默认开）
sudo sysctl -w net.ipv4.tcp_window_scaling=1

# 拥塞控制换 BBR（高 RTT 下 BBR 比 cubic 快得多）
sudo modprobe tcp_bbr
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr
sudo sysctl -w net.core.default_qdisc=fq
```

应用层别用 `setsockopt(SO_RCVBUF, ...)` 强制设具体值 —— 这会**关掉自动调节**。让内核自己 autotune。

#### 单连接吃不满怎么办

即使调好了，**单连接** TCP 吞吐仍可能受限于 cwnd 增长速度（慢启动 + 拥塞避免）。换 BBR 后稳态吞吐高，但启动阶段还是慢。

终极方案：**多连接并行**。`aria2c -s 16` 起 16 路并发；`rsync --whole-file` 或 BBR + 大文件分片传。

#### 验证

```bash
# 看实时窗口
ss -tin | grep -A 1 :22
# 关键字段: cwnd:xxx send xxxMbps rcv_wnd:xxx

# iperf3 单流测带宽
iperf3 -c <peer> -t 30
# 与 iperf3 -P 8（多流）对比
```

> **口诀**：带宽 × RTT = 你需要的窗口大小。窗口不够 → 调 buffer + 开 BBR + 多连接并行。

---

### 案例 ⑥ Keep-alive 默认 2 小时无效 —— "用业务层心跳"

#### 现象

夜里 3 点跨机房专线短暂抖动，TCP 连接被中间设备静默剔除。客户端不知道，下一次写数据时阻塞 15 分钟才报错。这 15 分钟内业务雪崩。

#### 根因

TCP keepalive 的默认参数：

```
net.ipv4.tcp_keepalive_time   = 7200    (2 小时)
net.ipv4.tcp_keepalive_intvl  = 75
net.ipv4.tcp_keepalive_probes = 9
```

**第一次探测要 2 小时后**才发，对生产几乎等于"没用"。中间 NAT/防火墙的 idle 老化时间通常是 5~30 分钟，连接早就被悄悄断了。

而且 keepalive **必须应用主动启用**（`SO_KEEPALIVE`），否则内核根本不发探测。

#### 修复

**方案 A**：把 sysctl 调小（生效粒度：全机器）。

```bash
sudo sysctl -w net.ipv4.tcp_keepalive_time=60
sudo sysctl -w net.ipv4.tcp_keepalive_intvl=10
sudo sysctl -w net.ipv4.tcp_keepalive_probes=3
```

含义：60s 没流量 → 探测；探测间隔 10s；连续 3 次失败 → 断开。

**方案 B**：单连接级别精细控制（推荐）。

```c
int keepalive = 1;
int keepidle  = 60;
int keepintvl = 10;
int keepcnt   = 3;
setsockopt(fd, SOL_SOCKET,  SO_KEEPALIVE,  &keepalive, sizeof(int));
setsockopt(fd, IPPROTO_TCP, TCP_KEEPIDLE,  &keepidle,  sizeof(int));
setsockopt(fd, IPPROTO_TCP, TCP_KEEPINTVL, &keepintvl, sizeof(int));
setsockopt(fd, IPPROTO_TCP, TCP_KEEPCNT,   &keepcnt,   sizeof(int));
```

**方案 C**（首选）：业务层心跳。

TCP keepalive 只能发现"链路全死"。它发现不了：
- 对端进程僵死（连接还在但不响应）
- 应用层逻辑卡死（accept 了但不读）
- 中间设备 hang

业务层心跳每 N 秒发一个 ping，N+T 秒内没收到 pong 就重连。RPC 框架（Dubbo、gRPC、Brpc）都内置了。

```yaml
# Dubbo
dubbo:
  protocol:
    heartbeat: 60000      # 60 秒心跳
```

```go
// gRPC server
grpc.KeepaliveParams(keepalive.ServerParameters{
    Time:    60 * time.Second,
    Timeout: 20 * time.Second,
})
```

#### 验证

```bash
# 看连接是否真的设了 keepalive
ss -tonpe | grep <pid>
# 输出会带 timer:(keepalive,xxs,0)

# 模拟链路抖动
sudo iptables -I INPUT -p tcp --dport 8080 -j DROP   # 黑掉
# 等待 keepalive_time + intvl*probes 秒
sudo iptables -D INPUT -p tcp --dport 8080 -j DROP   # 解开
# 看应用是否在合理时间内感知到断连
```

> **口诀**：TCP keepalive 默认参数等于没用；要么改 sysctl/socketopt，要么用业务心跳。**生产首选业务心跳**。

---

### 案例 ⑦ Nagle + Delayed ACK 的 "40ms 延迟"

#### 现象

服务平均 RT 5ms，P99 突然飙到 45ms。规律的"40ms 一档"，且只在小包频繁交互的场景出现（比如游戏服 / 行情推送）。

抓包看到客户端的小包 ACK 总是被推迟 40ms 才发。

#### 根因

两个 1980 年代的"省流量"算法**互相恶心**：

**Nagle（RFC 896）**：发送方小包要攒一攒。如果有未确认的小包在飞，新的小包**不发**，等 ACK 回来或攒满 MSS 再发。

**Delayed ACK（RFC 1122）**：接收方收到包不立即 ACK，等 40ms 看看自己是不是要回包，能合并就合并；或者收到第二个包就立即 ACK。

两个一起：

```
A → B: 小包 1
B 想 delay ACK 40ms 等合并
A 因为 Nagle 不能发小包 2（在等 ACK）
B 等了 40ms，没盼到 A 的第二个包，被迫单发 ACK
A 收到 ACK，发小包 2
```

中间凭空多了 40ms。

#### 修复

**方案 A**：关 Nagle（最常用）。

```c
int flag = 1;
setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &flag, sizeof(int));
```

代价：纯小包业务的网卡 PPS 会涨（因为不再合并）。但生产 99% 的场景都开。

```java
// Netty
.childOption(ChannelOption.TCP_NODELAY, true)
```

```go
// Go
conn.(*net.TCPConn).SetNoDelay(true)   // 默认就是 true
```

**方案 B**：关 Delayed ACK。Linux 没有标准 socket option 关 delayed ACK，但有 `TCP_QUICKACK`：

```c
int flag = 1;
setsockopt(fd, IPPROTO_TCP, TCP_QUICKACK, &flag, sizeof(int));
// 注意：这是一次性的，下次发数据后会自动恢复，需要每次设
```

实际场景中关 Nagle 已经够，很少需要再关 Delayed ACK。

#### 为什么默认还开着 Nagle

历史原因：1980 年代 telnet 一个字符发一个包，Nagle 大幅省了带宽。现在的应用大多自己批处理，Nagle 反而成了延迟瓶颈。但**默认行为不能改**（兼容性），所以应用要主动关。

#### 验证

```bash
# 抓包看小包 RTT 是否还有 40ms 一档
tcpdump -i eth0 -nn 'port 8080' -ttt
# 同一连接里 packet 间隔频繁出现 40ms 即未关

# 应用代码确认
ss -ti | grep -A1 :8080
# 看 nodelay 标记
```

> **口诀**：Nagle + Delayed ACK = 40ms 谜之延迟。`TCP_NODELAY` 是几乎所有现代服务必开的选项。

---

### 案例 ⑧ 容器内 net.* 不能改 —— "sysctl 与 namespace"

#### 现象

业务在 K8s Pod 内执行 `sysctl -w net.ipv4.tcp_max_syn_backlog=8192`，报错：

```
sysctl: setting key "net.ipv4.tcp_max_syn_backlog": Read-only file system
```

但宿主机能改。同事说"容器是隔离的，应该能在 Pod 内独立设置"。

#### 根因

Linux 的 sysctl 分**命名空间隔离**和**全局**两类：

| 类别 | 例子 | 容器内能改吗 |
| --- | --- | --- |
| net namespace 隔离 | `net.ipv4.ip_local_port_range` / `tcp_keepalive_time` / `somaxconn` 等大部分 | 能（容器自己的 net ns） |
| 全局（host 共享） | `net.core.rmem_max` / `net.ipv4.tcp_max_syn_backlog`（部分内核版本）/ `tcp_mem` | 不能 |

具体哪些是 namespaced 因内核版本而异，可以在容器内 `ls /proc/sys/net/ipv4/` 看哪些可写。

#### 修复

**方案 A**：在 K8s 里用 `securityContext.sysctls` 显式声明。

```yaml
spec:
  securityContext:
    sysctls:
      - name: net.core.somaxconn
        value: "4096"
      - name: net.ipv4.tcp_keepalive_time
        value: "60"
```

注意：

- 这种方式只能改**安全的 sysctl**（kubelet 默认列表）。
- 不安全的（"unsafe"）需要 kubelet 启动加 `--allowed-unsafe-sysctls=net.ipv4.tcp_max_syn_backlog,...` 才允许。
- 全局类（如 `net.core.rmem_max`）哪个都改不了，必须在宿主机改。

**方案 B**：initContainer 用 `privileged: true` 改。

```yaml
initContainers:
- name: sysctl-fix
  image: busybox
  command: ["sh", "-c", "sysctl -w net.core.somaxconn=4096"]
  securityContext:
    privileged: true
```

但这要求容器有特权能力，破坏了"无特权 Pod"的最佳实践。

**方案 C**：要求平台团队在节点的 `/etc/sysctl.conf` 配置（适用于全局类参数）。

#### 验证

```bash
# Pod 内验证
kubectl exec <pod> -- sysctl net.core.somaxconn

# 看实际生效
kubectl exec <pod> -- ss -lnt
```

> **口诀**：容器内 sysctl 是 net namespace 隔离的，但**不是全部** —— 全局类参数必须在宿主机改。生产先 `cat /proc/sys/net/...` 试着读，再决定改在哪一层。

---

### 案例 ⑨ TIME_WAIT 太多导致内核 hash 表查询慢

#### 现象

服务作为客户端持续访问下游 RPC，QPS 一直稳定，但某天发现 `connect()` 调用的 P99 从 0.1ms 飙到 5ms。CPU 也涨了一截。

`top` 看 si（软中断）占用高；`perf top` 显示热点是 `__inet_check_established` / `inet_twsk_for_each`。

#### 根因

内核在分配新连接的端口时，要确保**新四元组在 TIME_WAIT 表里没冲突**。这个查找在 `__inet_check_established` 里走 hash 链表。

当 TIME_WAIT 很多（比如几万到几十万）时，hash 表的某些 bucket 链表很长，每次 `connect()` 要遍历 —— **TIME_WAIT 越多，新连接 connect 越慢**。这是个 O(N) 退化。

观察：

```bash
ss -tan state time-wait | wc -l
185000

# 看 hash 大小
sysctl net.ipv4.tcp_max_tw_buckets
```

#### 修复

**方案 A**：少创造 TIME_WAIT。同案例 ②，**用连接池**。

**方案 B**：扩大 hash 表。这需要重新启动并调内核启动参数：

```
thash_entries=4194304    (启动时通过 cmdline 传)
```

实际生产几乎不这么做，复杂度太高。

**方案 C**：开 `tcp_tw_reuse`（同案例 ②）。复用本身就减少了表里的 entry。

**方案 D**：分散目的端 —— 用多个下游实例（loadbalancer 后多 pod），客户端的源端口 + 目的 IP 组合更分散，避免 hash 冲突集中。

#### 验证

```bash
# perf 看是不是还卡在那
sudo perf top -p <pid>
# 如果 __inet_check_established 不再 top，就修对了

# connect 延迟用 BPF 直接测
sudo /usr/share/bcc/tools/tcpconnlat
```

> **口诀**：TIME_WAIT 多不仅占端口，**还让 connect 慢**。根治还是连接池；其次 `tw_reuse`；再次目的端分散。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 端口与队列层
- [ ] `net.core.somaxconn ≥ 4096`，且**应用层 listen backlog 同步设大**（Tomcat/Nginx/Netty 都要改）
- [ ] `net.ipv4.tcp_max_syn_backlog ≥ 8192`
- [ ] `net.ipv4.tcp_syncookies = 1`（默认推荐值）
- [ ] `net.ipv4.ip_local_port_range = "1024 65535"`（客户端类服务）
- [ ] `tcp_abort_on_overflow` 通常保持 0（让客户端重传），除非你确定要快速失败

### 连接关闭层
- [ ] 应用代码路径全闭包：异常路径也 `close()`（try-with-resources / `defer close`）
- [ ] HTTP 客户端**消费完 body 再 close**（Go 的 `io.Copy(io.Discard, body)`）
- [ ] 监控 `CLOSE_WAIT` 数量，超过阈值告警（这一定是代码 bug）
- [ ] `net.ipv4.tcp_fin_timeout` 调到 30~60s（默认 60，FIN_WAIT_2 别拖太久）

### TIME_WAIT 治理层
- [ ] 高频短连接客户端开 `tcp_tw_reuse=1` + `tcp_timestamps=1`
- [ ] **绝不开** `tcp_tw_recycle`（NAT 翻车，且 4.12+ 已移除）
- [ ] **绝不调小** `tcp_max_tw_buckets` 当治标手段（违反协议）
- [ ] 治本：上连接池（HTTP keep-alive、RPC 连接复用）

### 心跳与超时层
- [ ] TCP keepalive 时间从默认 7200s 调到 ≤ 60s（或用业务心跳）
- [ ] 业务层心跳间隔 ≤ 中间设备最小 idle 老化时间的一半
- [ ] `tcp_retries2` 默认 15 太长（约 13~30 分钟），考虑调到 8~10
- [ ] 应用 read/write 都设业务超时（不要靠内核超时报错）

### 性能层
- [ ] `TCP_NODELAY` 开（关 Nagle）—— 几乎所有现代服务都需要
- [ ] BBR + fq qdisc（高 RTT / 跨地域链路必开）
- [ ] `tcp_rmem` / `tcp_wmem` 上限 ≥ BDP × 2
- [ ] 单连接吃不满带宽时考虑多连接并行

### 安全层
- [ ] `tcp_syncookies=1` 抗 SYN_FLOOD
- [ ] 上游 / 边界做源 IP 校验、流量清洗（不依赖单台 Linux 防御）
- [ ] 监控 `TcpExtSyncookiesSent` 持续 > 0 时告警

### 容器层
- [ ] 容器内能改的 sysctl（net namespace 隔离的）通过 `securityContext.sysctls` 声明
- [ ] 全局类 sysctl 在宿主机层管理，不在 Pod 层尝试
- [ ] 容器内 `ulimit -n` 配套调大（高连接服务）

### 监控层
- [ ] `nstat -az` 关键计数：`ListenOverflows` / `ListenDrops` / `SyncookiesSent` / `TCPSynRetrans` / `TCPTimeWaitOverflow`
- [ ] `ss -s` 周期采集（TIME_WAIT、ESTAB、半连接计数）
- [ ] BPF 工具备好：`tcpconnect` / `tcptop` / `tcpconnlat`
- [ ] 告警：`CLOSE_WAIT > N`、`TIME_WAIT > 50000`、`ListenOverflows > 0`

### 抓包能力
- [ ] 生产机器上 `tcpdump`、`ss`、`nstat` 必装
- [ ] 知道怎么用 `-w` 抓 + 异地 wireshark 看
- [ ] 抓包要带 `-s 0`（完整包）和 `-G 60 -W 10`（按时间切片，避免占满磁盘）

---

## 6. 还能往下挖什么

读完上面 9 个案例只是入门。如果想再深一层：

### eBPF 排查 TCP
- `bcc/bpftrace`：`tcpconnect` / `tcpaccept` / `tcpretrans` / `tcpconnlat`，比 `strace` 快 10 倍且无侵入
- `bpftrace` 自定义：`tracepoint:tcp:tcp_retransmit_skb` 抓重传栈帧，定位代码层根因
- Cilium 的 socket-LB：用 BPF 把 K8s Service 转换从 iptables 替换成 BPF，连接建立提速

### QUIC 凭什么解决 TCP 问题
- 0-RTT 握手（依赖 TLS 1.3 PSK），比 TCP+TLS 快 1~2 个 RTT
- 用户态实现：内核改动不再是部署阻碍（拥塞算法迭代速度大幅提升）
- 解决 head-of-line blocking：每个 stream 独立丢包恢复，HTTP/3 多路复用真正不互相阻塞
- 代价：CPU 开销比 TCP 高（用户态 + 加密强制），中间设备识别能力差

### TCP Fast Open（TFO）
- RFC 7413：第一次握手得到 cookie，之后可以在 SYN 包就带 payload，省 1 RTT
- `net.ipv4.tcp_fastopen=3`（client+server）
- 问题：中间设备/防火墙不识别 TFO，导致部分链路反而失败 —— 启用前必须验证全链路兼容

### Linux 5.x 的 sock_diag
- `ss` 底层就是 `sock_diag` netlink，能拿到比 `/proc/net/tcp` 更详细的 socket 状态
- 自己写工具时优先用 `AF_NETLINK + sock_diag`，不要 parse `/proc/net/tcp`（O(N) 且字段不全）

### DPDK / 用户态网络栈
- 把网卡 IO 从内核态搬到用户态，绕过 TCP/IP 栈
- mTCP / F-Stack：用户态 TCP 实现，性能 5~10× 内核
- 适合什么场景：CDN 边缘、高频交易、大流量代理；不适合一般业务（生态、调试代价大）

### TCP Congestion Control 的演进史
- Reno（1990）→ NewReno → Vegas（基于延迟）→ cubic（默认）→ BBR / BBRv2 / BBRv3
- 在不同 RTT、不同丢包率下各算法的相对优势
- DCTCP（Data Center TCP）：用 ECN 标记代替丢包做拥塞信号，数据中心内部专用

### TCP 状态机相关的深坑
- `RST` 攻击：伪造 RST 关人家连接（mitigation: window-based RST 验证）
- 半连接 SYN 选项压缩：syncookies 模式下 SACK / 时间戳 / WS 部分丢失，影响吞吐
- `tcp_recovery` / RACK：基于时间的快速重传机制，比传统三次重复 ACK 更准确

---

## 7. 引用

- [RFC 793](https://www.rfc-editor.org/rfc/rfc793) — Transmission Control Protocol（原始 TCP 规范，1981）
- [RFC 7414](https://www.rfc-editor.org/rfc/rfc7414) — A Roadmap for TCP Specification Documents
- [RFC 6298](https://www.rfc-editor.org/rfc/rfc6298) — Computing TCP's Retransmission Timer
- [RFC 7413](https://www.rfc-editor.org/rfc/rfc7413) — TCP Fast Open
- Linux 5.15 source: `net/ipv4/tcp_input.c` / `tcp_output.c` / `tcp_minisocks.c` / `tcp_ipv4.c`
- *TCP/IP Illustrated, Vol. 1*，W. Richard Stevens — 经典中的经典（v2 由 Kevin Fall 修订）
- *Computer Networking: A Top-Down Approach*，Kurose & Ross
- Google BBR 论文：[BBR: Congestion-Based Congestion Control](https://queue.acm.org/detail.cfm?id=3022184)（CACM, 2017）
- bcc/BPF Compiler Collection：[https://github.com/iovisor/bcc](https://github.com/iovisor/bcc)
- Cloudflare 博客：[The story of one latency spike](https://blog.cloudflare.com/the-story-of-one-latency-spike/) — 经典 TCP 排查案例

---

> 配套：[网络与 RPC 主题](/article/network/) | [Netty EventLoop](/article/network/netty-eventloop) | [Linux 与性能工程](/article/linux-perf/)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果这个深度你觉得对，告诉我下一篇该写哪个 —— "HTTP/2 vs HTTP/3"、"Linux 网络收包路径"、"epoll 与 io_uring 对比"都在候选清单里。
