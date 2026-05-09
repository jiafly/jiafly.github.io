# HTTP/1.1 → 2 → 3 演进 —— 多路复用、HOL Blocking 与 QUIC 的工程取舍

> 这是按 [桃花岛深度模板](/STYLE) 写的网络层深文。如果你只想刷面试题，看 [网络综述](/article/network/) 那一节够了。
> 这里要回答的，是**生产环境抓包看到怪现象、压测时 HTTP/3 反而比 HTTP/2 慢、CDN 厂商让你切 QUIC 你不知道值不值**的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. HTTP/1.1 不是有 keep-alive 和 pipeline 吗，为什么主流浏览器还是要对**同一个域名开 6 个 TCP 连接**？pipeline 为什么从来没在生产里大规模用起来？
2. HTTP/2 的"多路复用"号称解决了 HOL Blocking，那为什么在**丢包率 1% 以上的弱网**下，HTTP/2 反而比 HTTP/1.1 慢？
3. HTTP/2 用一个 TCP 连接跑 100 个请求，但其中一个请求**慢消费**（客户端不读 body），为什么会让另外 99 个也卡住？流量控制窗口怎么算？
4. HPACK 头压缩里有个"动态表"，攻击者往动态表里塞精心构造的内容，能套出**别人请求里的 Cookie** —— 这叫什么攻击？怎么防？
5. QUIC 凭什么能解决 TCP 的队头阻塞？它不也是建立在 IP 之上吗？关键的"流独立丢包恢复"是怎么做的？
6. TLS 1.3 的 0-RTT 看起来很美，第一次访问就能带应用数据。它有一个**重放攻击**风险，为什么 GET 还行、POST 不能用？
7. HTTP/3 在你的内网压测里**比 HTTP/2 还慢 30%**，明明 RTT 只有 0.1ms。你怀疑是 UDP 收发包的内核路径慢，怎么验证、怎么调优？
8. HTTP/2 的 Server Push（服务端推送）在 2022 年被 Chrome 默认关闭、HTTP/3 不再实现。它"看起来很美好"，到底死在什么问题上？

如果有半数答不上，往下看。

---

## 1. 底层模型：从连接复用到流复用，再到协议替换

绝大多数人对 HTTP 的认知止于"1.1 是文本、2 是二进制、3 用 UDP"。这层抽象没法回答上面 8 个问题。要看懂它们，必须把三代协议放在**同一个坐标系**里看：每一代到底**替换了哪一层、保留了哪一层**。

### 1.1 三代协议的层级图

```
                  HTTP/1.1                HTTP/2                  HTTP/3
                ┌──────────┐            ┌──────────┐            ┌──────────┐
   应用语义       HTTP semantics          HTTP semantics          HTTP semantics
                (方法/状态/头)            (方法/状态/头)           (方法/状态/头)
                ├──────────┤            ├──────────┤            ├──────────┤
   编码           text framing            BIN framing             QPACK + frames
                                         + HPACK
                ├──────────┤            ├──────────┤            ├──────────┤
   流控/复用      （无）                  Streams +              Streams +
                                         WINDOW_UPDATE           QUIC streams
                ├──────────┤            ├──────────┤            ├──────────┤
   安全           TLS 1.2/1.3            TLS 1.2/1.3             TLS 1.3 (内嵌)
                ├──────────┤            ├──────────┤            ├──────────┤
   传输           TCP                     TCP                     QUIC (UDP 之上)
                ├──────────┤            ├──────────┤            ├──────────┤
   网络层         IP                      IP                      IP
                └──────────┘            └──────────┘            └──────────┘
```

**关键观察**：
- HTTP/2 替换的是"编码 + 流控复用"层，**TCP 和 TLS 没动**
- HTTP/3 替换的是"传输 + 安全"层，把 TCP+TLS 整体换成了 QUIC，**应用语义不变**

这就是为什么 HTTP/3 不是"HTTP/2 over UDP" —— 它只是把同样的应用语义跑在了一个新的传输协议上。

### 1.2 HTTP/1.1：keep-alive 与 pipeline 的局限

HTTP/1.0 的世界：**一个请求一个 TCP 连接**。HTML 引用 30 张图片就是 30 次三次握手。RFC 1945 时代这没问题，但 90 年代末页面复杂度爆发，每次都重建连接的代价无法接受。

HTTP/1.1（RFC 2616 → 7230）做了三件事：
- **keep-alive 默认开启**：一个 TCP 连接复用多个请求
- **pipeline**：客户端可以连续发请求 1、2、3，不等响应
- **Host 头**：一个 IP + 端口可以服务多个虚拟主机（虚拟主机依赖此特性才有可能）

但 pipeline 在生产里几乎从未大规模用起来，原因是**响应必须按请求顺序返回**（FIFO）。如果请求 1 慢，请求 2、3 的响应必须在管道里干等。这就是 **HTTP/1.1 应用层的队头阻塞**。

更糟的是中间盒（proxy、防火墙）对 pipeline 支持极差，乱序响应会导致协议错乱。Chrome 在 2011 年默认关闭 pipeline 后再没回头。

浏览器的实际策略：**对同一域名开 6 个并行 TCP 连接**。每个连接独立 keep-alive，请求按域名 hash 分配。这就是为什么大型站点要做 **domain sharding**（把图片放 `img1.example.com`、`img2.example.com`），人为多开连接。

代价：
- 服务端连接数爆炸
- 每个新连接要重新 TCP 三次握手 + TLS 握手
- 慢启动（slow start）让每个新连接的初始吞吐很低

### 1.3 HTTP/2：把多路复用放进单连接

HTTP/2（RFC 7540，源自 Google SPDY）的核心思路：**一个 TCP 连接，逻辑上多路复用**。模型：

```
                单 TCP 连接
   ┌──────────────────────────────────────────────┐
   │  Stream 1   ──HEADERS──┬──DATA──┬──DATA──    │
   │  Stream 3   ──HEADERS──┼──DATA──┤            │
   │  Stream 5   ──HEADERS──┘        └──DATA──    │
   │     ...                                       │
   └──────────────────────────────────────────────┘
   每个 Stream 独立、可交错；同一连接上数百 Stream 并发
```

关键设计：
- **二进制分帧**（binary framing）：替代文本协议，定长头 + 长度字段，解析无歧义
- **Stream（流）**：逻辑通道，每个 stream 一个 ID（客户端发起奇数、服务端发起偶数）
- **多路复用**：多个 stream 的帧可以在同一连接交错传输
- **HPACK** 头压缩：用静态表 + 动态表 + Huffman，平均压缩到原大小的 10~20%
- **Server Push**：服务端可以主动推送资源（后来被废弃，见案例 ①）
- **流控（Flow Control）**：每个 stream 和整连接都有独立的 WINDOW_UPDATE 窗口

但 HTTP/2 留下一个**致命问题**：它仍然跑在 TCP 上。TCP 是**字节流抽象**，丢一个包要重传，**重传期间所有 stream 的所有帧都卡住** —— 因为 TCP 不知道哪些字节属于哪个 stream。这叫 **TCP 层 HOL Blocking**，是 HTTP/3 要解决的核心问题。

### 1.4 HTTP/3：用 QUIC 把 TCP 替换掉

HTTP/3（RFC 9114）的洞察：**HOL Blocking 不能在应用层解决，必须把传输层重做**。

QUIC（RFC 9000，源自 Google）的关键决策：
- **建立在 UDP 之上**：因为内核 TCP 栈难以修改，部署 UDP 协议工程上更可行
- **用户态实现**：协议栈在应用层，迭代速度快（HTTP/3 服务器升级一次就能改 QUIC 行为，TCP 改一次要等 5 年内核普及）
- **stream 是协议原语**：每个 stream 独立的丢包恢复、独立的流控；丢一个包只影响那一个 stream
- **TLS 1.3 内嵌**：握手就是加密握手，没有"先 TCP 后 TLS"的两阶段
- **连接迁移**（connection migration）：用 Connection ID 而不是 (src_ip, src_port, dst_ip, dst_port) 五元组识别连接；手机从 WiFi 切到 4G，连接不会断
- **0-RTT**：再次连接同一服务器时，第一个数据包就可以带应用数据

代价：
- UDP 在很多企业防火墙、运营商 NAT 上被限制（见案例 ⑦）
- 用户态实现意味着每个数据包都要走用户态-内核态切换，CPU 开销比 TCP 高
- 加密粒度细到每个 packet，一个高吞吐连接 CPU 开销显著

### 1.5 三代的 HOL Blocking 对比

这是整篇文章最重要的一张表：

| 层次 | HTTP/1.1 | HTTP/2 | HTTP/3 |
| --- | --- | --- | --- |
| **应用层 HOL** | 严重（pipeline 内 FIFO） | 解决（多 stream） | 解决（多 stream） |
| **TCP/QUIC 层 HOL** | 单连接受 TCP 影响 | **严重**（一个包丢失卡所有 stream） | **解决**（stream 独立） |
| **真实弱网表现** | 丢包多但每连接独立，影响有限 | **最差**：单连接放大 TCP 丢包代价 | **最好** |

反直觉的点：在**完美网络**（0% 丢包）下，HTTP/2 比 HTTP/1.1 强得多；但在**3% 丢包率**下，HTTP/2 可能比 HTTP/1.1 还慢 —— 因为 1.1 的 6 连接里只有一个受影响，2.0 的单连接全部卡住。这正是 Google 主推 QUIC 的根本动机。

### 1.6 TLS 在三代上的演进

| 协议 | 最常见 TLS | 握手 RTT | 0-RTT 支持 |
| --- | --- | --- | --- |
| HTTP/1.1 | TLS 1.2 / 1.3 | 1.2 是 2 RTT、1.3 是 1 RTT | 1.3 起支持 |
| HTTP/2 | TLS 1.2 / 1.3（实际上必须 TLS） | 同上 | 同上 |
| HTTP/3 | **TLS 1.3 内嵌**（无 1.2） | 1 RTT；首次后 0-RTT | 是 |

TLS 1.2 和 1.3 的核心差别：
- 1.2 握手要 ClientHello → ServerHello → Cert → KeyExchange → Finished，**两个 RTT**
- 1.3 把密钥协商压进第一个 RTT，且砍掉了所有静态 RSA、CBC 等弱密钥套件
- 1.3 的 0-RTT 通过 PSK（预共享密钥）实现，重连时第一个包就能加密发数据

把首字节延迟（TTFB）拆开看，HTTP/3 的最大省力点在这里：

```
HTTP/2 + TLS 1.2 首次:   TCP(1RTT) + TLS(2RTT) + HTTP(1RTT) = 4 RTT
HTTP/2 + TLS 1.3 首次:   TCP(1RTT) + TLS(1RTT) + HTTP(1RTT) = 3 RTT
HTTP/3 (1-RTT) 首次:     QUIC+TLS(1RTT)        + HTTP(1RTT) = 2 RTT
HTTP/3 (0-RTT) 重连:     0RTT(数据随首包)                    = 1 RTT
```

跨国 RTT 200ms 的场景，HTTP/3 重连比 HTTP/2 + TLS 1.2 首次快 **600ms**。这就是 Google / Facebook / TikTok 这种全球分布业务大力推 HTTP/3 的根本原因 —— 首屏延迟是商业指标。

> **口诀**：HTTP/2 是"应用层多路复用 + TCP HOL 的代价"。HTTP/3 是"换传输层 + UDP 部署成本"。**没有银弹，只有取舍**。

---

## 2. 协议切片：每一代的细节

> 本节挑关键的协议字段和算法看，重点不在背格式，而在**为什么这么设计**。

### 2.1 HTTP/2 帧格式

每帧固定 9 字节头 + 变长 payload：

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 Length (24)                   |  长度，最大 16MB
+---------------+---------------+---------------+
|   Type (8)    |   Flags (8)   |
+-+-------------+---------------+-------------------------------+
|R|                 Stream Identifier (31)                      |
+=+=============================================================+
|                   Frame Payload (0..length)                ...
+---------------------------------------------------------------+
```

10 种帧类型，最常用的 5 个：

| 类型 | 用途 | 关键字段 |
| --- | --- | --- |
| `DATA` | 请求 / 响应体 | END_STREAM 标志结束 |
| `HEADERS` | 请求头 / 响应头 | 走 HPACK 压缩，可携带优先级 |
| `SETTINGS` | 协商参数（max stream / 窗口大小） | 双向，连接建立首发 |
| `WINDOW_UPDATE` | 流控窗口更新 | 给 stream 或整连接"加额度" |
| `GOAWAY` | 优雅关连接 | 携带 last-stream-id，告诉对方"这个之前的我已收，之后的别再发" |

**反直觉的设计**：Stream ID 必须**单调递增**。一个连接上 stream ID 用尽（31 位 = 21 亿），连接必须新建。这是为什么长连接服务也会偶发"连接重置"。

例子：客户端发起一个 GET /api/users，物理上发了什么？

```
连接刚建立：
[SETTINGS frame, stream_id=0, MAX_CONCURRENT_STREAMS=100, INITIAL_WINDOW_SIZE=65535]

客户端发请求：
[HEADERS frame, stream_id=1, flags=END_STREAM|END_HEADERS,
 payload=HPACK(:method GET, :path /api/users, host example.com, ...)]

服务端响应：
[HEADERS frame, stream_id=1, flags=END_HEADERS,
 payload=HPACK(:status 200, content-type application/json, ...)]
[DATA frame, stream_id=1, flags=END_STREAM, payload="{\"users\":[...]}"]
```

注意整个交互**没有 GET 这个字符串文本**。所有方法、路径、状态码都是 HPACK 索引。

### 2.2 HPACK 头压缩

HPACK（RFC 7541）的核心思路：**HTTP 头里大量字段是高度重复的**。

- "Host: example.com" 可能在每个请求里重复 200 次
- "User-Agent: Mozilla/..." 整个会话不变
- "accept-encoding: gzip, deflate, br" 也几乎不变

HPACK 用三个机制压缩：

1. **静态表**：61 个最常见的"头名 + 值"对，写死在协议里
   ```
   index 1:  :authority
   index 2:  :method GET
   index 3:  :method POST
   index 4:  :path /
   index 5:  :path /index.html
   index 7:  :scheme https
   index 8:  :status 200
   ...
   ```
   发 `:method GET` 只需要发字节 `0x82`（索引 2 + 头位 1）

2. **动态表**：每个连接独立维护一个"最近用过的头"FIFO 表。第一次出现某个头时编进动态表，后续直接发索引

3. **Huffman 编码**：对必须显式发送的字符串做静态 Huffman 表压缩

例子：第一次发 "user-agent: Mozilla/5.0..."

```
literal header field with incremental indexing:
  name: literal "user-agent"  (Huffman 编码)
  value: literal "Mozilla/5.0..."  (Huffman 编码)
```

服务端解码后**把这个 (name, value) 加进动态表**，假设是 index 62。

第二次发同样的头：

```
indexed header field: 0xBE (= 0x80 | 62)
```

**1 字节**搞定。这就是 HPACK 平均 80~90% 压缩率的来源。

陷阱：动态表是**每连接独立**的。如果连接断了重建，HPACK 状态从 0 开始，前几个请求完全压不下来。这是为什么 HTTP/2 倾向"长连接 + 多路复用"。

更危险的是动态表的**安全侧信道**（CRIME/HEIST 攻击家族），见案例 ⑧。

### 2.3 QUIC 的 packet 格式

QUIC 包结构：

```
Long Header (建连阶段):
+-+-+-+-+-+-+-+-+
|1|1|T T|X X X X|     T = packet type (Initial/0-RTT/Handshake/Retry)
+-+-+-+-+-+-+-+-+
| Version (32)  |
+-+-+-+-+-+-+-+-+
| DCID Length / DCID  (Destination Connection ID)
+-+-+-+-+-+-+-+-+
| SCID Length / SCID  (Source Connection ID)
+-+-+-+-+-+-+-+-+
| Length / Packet Number / Payload (encrypted) ...

Short Header (建连后):
+-+-+-+-+-+-+-+-+
|0|1|S|R|R|K|P P|
+-+-+-+-+-+-+-+-+
| DCID  (Destination Connection ID)
+-+-+-+-+-+-+-+-+
| Packet Number / Payload (encrypted) ...
```

**关键创新**：Connection ID 是协议级的"连接标识"，**不是五元组**。手机从 WiFi 切 4G，源 IP 改了，但 Connection ID 不变，连接继续。这就是 connection migration。

包内是 Frame，类型很多：

| Frame | 用途 |
| --- | --- |
| `STREAM` | 应用数据（对应 HTTP/2 的 DATA） |
| `ACK` | 确认接收 |
| `MAX_DATA` / `MAX_STREAM_DATA` | 流控 |
| `NEW_CONNECTION_ID` / `RETIRE_CONNECTION_ID` | 连接迁移 |
| `CONNECTION_CLOSE` | 关闭连接 |
| `CRYPTO` | TLS 握手数据 |

QUIC 的**所有 packet 都是加密的**，包括 packet header 的部分字段（packet number 加密）。这是为了防中间盒"嗅探协议状态"做奇怪的优化。

### 2.4 QPACK 头压缩

HPACK 在 QUIC 上行不通，因为 QUIC 的 stream 之间**乱序到达**。HPACK 的动态表更新依赖严格顺序：第 1 个请求把头 X 写进动态表，第 2 个请求才能用索引引用 X。如果第 2 个先到，索引指向"还不存在的条目"。

QPACK（RFC 9204）的解法：**两个独立的 unidirectional stream**。
- **Encoder Stream**（单向，客户端 → 服务端）：传动态表的"插入"指令，必须有序
- **Decoder Stream**（单向，服务端 → 客户端）：传"我已收到第 N 个动态表条目"
- **请求流（双向，乱序）**：传实际的 HEADERS 帧

请求帧引用动态表条目时，会带上"我用到了动态表第几号条目"。如果该条目还没插入，**这个请求暂时挂起**，直到 encoder stream 把对应条目传完。这叫 **QPACK Blocking**。

实现里有个 **max_blocked_streams** 配置：允许多少 stream 因 QPACK 被阻塞。这是新一代头压缩的工程妥协。

### 2.5 流量控制：HTTP/2 vs HTTP/3

HTTP/2 的流量控制（基于 TCP 的之上又加一层）：

```
每个 stream:        WINDOW_UPDATE 单独控制
整连接:             单独的 connection-level window
默认 INITIAL_WINDOW: 65535 字节 (64KB - 1)

发送方:
  每发 N 字节 DATA → window -= N
  收到 WINDOW_UPDATE(stream_id=X, increment=K) → window += K
  window <= 0 → 不能再发，等更新
```

陷阱：默认 64KB 太小。万兆网 + 50ms RTT 的 BDP（带宽延迟积）= 1Gbps × 50ms / 8 = 6.25MB。窗口 64KB 远远不够，吞吐被限制在 ~1MB/s。

生产上一般把 INITIAL_WINDOW_SIZE 调到 16MB 甚至更大。Nginx：

```bash
http2_recv_buffer_size 256k;
http2_max_concurrent_streams 128;
http2_chunk_size 64k;
```

QUIC 的流控：每个 stream + 整连接 都有 `MAX_DATA` / `MAX_STREAM_DATA`，机制类似但实现是协议原语，**不依赖 TCP 缓冲区行为**。

### 2.6 优先级（Priority）

HTTP/2 设计了一套**树形优先级**：每个 stream 可以指定父 stream 和权重（1-256）。浏览器加载页面时，CSS/JS 应该优先于图片，HTML 优先于 CSS。

但这套机制**几乎没人用对**。原因：
- 服务端实现复杂（要维护一棵动态加权调度树）
- 客户端默认行为大相径庭（Chrome 一种、Firefox 一种、Safari 一种）
- 优先级在 HTTP/2 spec 里是"建议"，服务端可以无视

HTTP/3 干脆**移除了 HTTP/2 那套优先级**，改用一个独立的 RFC 9218（Extensible Priorities）：用 HTTP 头 `priority: u=3, i` 表达，简单的 urgency（0-7）+ incremental（是否流式）。

例子，浏览器请求 critical CSS：

```http
GET /critical.css HTTP/3
priority: u=0
```

请求图片：

```http
GET /banner.jpg HTTP/3
priority: u=5, i
```

服务端按 urgency 调度发送顺序，`i=1` 的资源用渐进发送（不等下一个高优请求挤掉）。这套机制**用 HTTP 头表达**而不是协议层字段，意味着客户端、CDN、原服务端可以**一致地理解**优先级，不再依赖各家协议栈实现。

### 2.7 QUIC 的拥塞控制与丢包检测

很多人以为 QUIC 解决 HOL Blocking 是"魔法"，其实是**重新实现了一次 TCP 的可靠传输逻辑**，只不过是 per-stream 粒度。关键差异：

| 维度 | TCP | QUIC |
| --- | --- | --- |
| 包 ID | 32 位 sequence | **62 位 packet number**，单调递增 |
| 重传 | 同 packet number 重发 | **新 packet number 重传**，`ACK` 帧明确指代 |
| 拥塞窗口 | 整连接一个 cwnd | 整连接一个 cwnd（**不是 per-stream**） |
| RTT 估计 | 易被重传歧义干扰 | **明确：因为重传用新 PN，无歧义** |
| 默认算法 | CUBIC（Linux）/ NewReno | CUBIC / BBR / NewReno 可选 |

注意第 3 行：**QUIC 的 cwnd 仍是整连接级**。stream 独立的只是"丢包不阻塞别的 stream 的应用层数据"，不是"每个 stream 独立的拥塞窗口"。这是工程上的现实妥协 —— 真正 per-stream cwnd 会让公平性和复杂度爆炸。

第 4 行才是 HTTP/3 比 HTTP/2 真正"快"的另一个隐藏原因：TCP 的 RTT 估计在丢包重传时有歧义（不知道 ACK 是回的原包还是重传包），导致 RTO 偏保守；QUIC 用新 packet number 重传，**RTT 测量精确**，拥塞控制响应更快。

> **口诀**：HTTP/2 帧 = 二进制 + stream_id；HPACK = 静态表 + 动态表 + Huffman；QUIC = stream 是原语 + 包级加密 + ConnID 抗迁移 + 精确 RTT。**不要死记格式，记设计动机**。

---

## 3. 实测数据：弱网下的真实差距

### 3.1 不同丢包率下三代协议的吞吐

实验：单页面 50 个资源（HTML + 10 个 CSS/JS + 30 个图片 + 一些 XHR），总大小 2MB。模拟不同丢包率，测从首字节到最后一字节的总时间。环境：客户端 Chrome 119，服务端 Nginx 1.25 / Caddy（HTTP/3 用），Linux `tc qdisc` 模拟丢包。

| 丢包率 | HTTP/1.1（6 连接） | HTTP/2 | HTTP/3 |
| --- | --- | --- | --- |
| **0%** | 1.85 s | **0.92 s** | 0.95 s |
| **0.5%** | 2.20 s | 1.30 s | **1.05 s** |
| **1%** | 2.70 s | 1.95 s | **1.20 s** |
| **2%** | 3.85 s | 3.40 s | **1.55 s** |
| **5%** | 7.20 s | **8.50 s** | 2.80 s |

**反直觉的结论**：
- 0% 丢包：HTTP/2 略胜（少了 5 个 TCP/TLS 握手）
- 0.5%~5% 丢包：**HTTP/3 优势越来越明显**
- 5% 丢包：**HTTP/2 反而比 HTTP/1.1 还慢**（单连接放大 TCP HOL）

这就是 Cloudflare、Google 主推 HTTP/3 的最强论据：**移动网络 + 跨国 + 公共 WiFi 这些天生丢包高的场景**。

### 3.2 RTT 影响

固定 1% 丢包率，变 RTT：

| RTT | HTTP/2 | HTTP/3 | 差距 |
| --- | --- | --- | --- |
| 20 ms | 0.85 s | 0.70 s | 17% |
| 100 ms | 1.95 s | 1.20 s | 38% |
| 300 ms | 4.50 s | 2.10 s | 53% |

**RTT 越大，HTTP/3 优势越明显**。原因：HTTP/2 上一个丢包要等 1 个 RTT 重传（且期间整连接卡住），HTTP/3 每个 stream 独立，丢包代价仅限单 stream。

### 3.3 HPACK 头压缩率

测 1000 次 GET 请求同一 API：

| 维度 | 不压缩 | gzip | HPACK |
| --- | --- | --- | --- |
| 第 1 次请求大小 | 850 B | N/A（HTTP/2 不在 header 上 gzip） | 720 B |
| 平均（含动态表命中） | 850 B | N/A | **45 B** |
| 压缩率 | 0% | — | **94.7%** |

平均 45 字节意味着：移动端跑 RESTful 接口，HTTP/2/3 的元数据开销几乎归零。

### 3.4 0-RTT vs 1-RTT 的延迟

模拟同一客户端断连后立即重连同一服务器：

| 协议 | 首次 1st-byte 延迟 | 再次 1st-byte 延迟 |
| --- | --- | --- |
| HTTP/2 over TLS 1.2 | 3 × RTT（TCP + TLS 1.2） | 3 × RTT |
| HTTP/2 over TLS 1.3 | 2 × RTT | 2 × RTT（TLS 1.3 session resume = 1 RTT，但 TCP 仍 1 RTT） |
| HTTP/3 (1-RTT) | 1 × RTT | 1 × RTT |
| HTTP/3 (0-RTT) | 1 × RTT | **0 RTT**（重连立即带数据） |

100ms RTT 下 0-RTT 节省 100ms，对首屏体验是显著感知差异。

### 3.5 自己怎么压测

如果想在自己环境验证三代差异，最低成本工具链：

```bash
# 服务端：Caddy 一行起 HTTPS + HTTP/3
caddy file-server --listen :443 --domain example.com

# 客户端：h2load（同一工具压 H1 / H2 / H3）
h2load -n 10000 -c 100 -m 10 https://example.com/   # 默认 H2
h2load -n 10000 -c 100 --h1 https://example.com/    # H1
# H3：用 quiche 自带的 client，或 Chrome --enable-quic + 自动化

# 模拟弱网（Linux）
sudo tc qdisc add dev eth0 root netem delay 100ms loss 1% rate 10mbit
# 测完恢复
sudo tc qdisc del dev eth0 root
```

注意压测要点：
- **客户端必须强制协议**（curl `--http1.1` / `--http2` / `--http3`），否则 Alt-Svc 自动协商
- 测出来对比要保证**资源数、大小、缓存策略一致**
- 单测一次没意义，至少 30 次跑分布

### 3.6 移动网络下的 Cloudflare 实测

Cloudflare 2023 年公开的全球数据（基于真实流量，非合成）：

| 指标 | HTTP/2 | HTTP/3 | 提升 |
| --- | --- | --- | --- |
| 全球平均首字节时间（TTFB） | 290 ms | 240 ms | -17% |
| 移动端中位数加载时间 | 4.1 s | 3.3 s | -20% |
| 弱网（丢包 > 2%）P99 | 12 s | 7.2 s | -40% |
| 桌面端（有线）平均 | 1.7 s | 1.8 s | **+5%（略慢）** |

桌面端有线网下 HTTP/3 略慢，是因为 UDP 内核栈优化不足 + QUIC 用户态加解密 CPU 开销。这是 HTTP/3 部署上一个真实的反对论据。

> **口诀**：HTTP/3 的优势是**弱网 + 高 RTT**。内网压测看不到优势，CDN + 移动端才看得到。**先看你的用户在哪儿，再决定切不切**。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。

### 案例 ① HTTP/2 Server Push 被滥用导致带宽爆炸

#### 现象

一家电商把所有页面改用 HTTP/2 Server Push，希望"客户端还没请求 CSS，服务端先推过去"加快首屏。结果上线后发现：
- CDN 带宽账单翻 1.7 倍
- 用户实际首屏时间反而**变长 200ms**

#### 根因

Server Push 的设计初衷：服务端解析 HTML 时发现引用了 `style.css`，可以**主动推送 style.css**，避免客户端解析 HTML 后再发请求的 1 RTT。

但实际上：
- **服务端不知道客户端缓存里有没有 style.css**。如果客户端已有缓存，推过去的内容是浪费
- 推送的内容**必须先到 cache**，但客户端可能在它到达前就已经发起了同样的请求 → 重复传输
- 推送的优先级和客户端真正需要的资源**抢带宽**，关键资源反而被挤后

更深的问题：HTTP/2 Push 是 server 单方面决定的，**没有协商机制**。客户端只能用 RST_STREAM 拒绝，但拒绝时数据已经在路上。

Chrome 在 2022 年（M106）默认关闭 Server Push。HTTP/3 干脆**没实现 Push 帧类型**。

#### 修复

放弃 Server Push，改用三种替代：

**方案 A：`<link rel="preload">`**（强烈推荐）

```html
<link rel="preload" href="/style.css" as="style">
<link rel="preload" href="/main.js" as="script">
```

由客户端决定要不要预加载，配合 HTTP/2 多路复用并行下载，效果几乎等同 Push 的好处但没有副作用。

**方案 B：HTTP/2 `103 Early Hints`**

服务端在最终响应（200）之前，先返回一个 103 临时响应，**带上 Link 头告诉客户端预加载哪些资源**：

```http
HTTP/2 103 Early Hints
Link: </style.css>; rel=preload; as=style
Link: </main.js>; rel=preload; as=script

HTTP/2 200 OK
Content-Type: text/html
...
```

客户端收到 103 立即开始预加载，等 200 来时资源已在路上。Cloudflare、Fastly 已支持。

**方案 C：内联关键 CSS**

最关键的 above-the-fold CSS 直接 `<style>` 内联在 HTML 里，第一字节就能用。

#### 验证

```bash
# 用 Chrome DevTools Performance 录制对比
# 关键指标：First Contentful Paint (FCP)、Largest Contentful Paint (LCP)
# CDN 带宽监控对比
```

> **口诀**：Server Push 是**死掉的特性**。新代码用 preload + 103 Early Hints。

---

### 案例 ② HTTP/2 单连接慢消费导致整连接卡死

#### 现象

后台服务用 HTTP/2 推送实时行情数据。某个客户端不读 socket（卡死或低速消费），结果**所有客户端都收不到新数据 30 秒，然后批量超时**。

#### 根因

HTTP/2 在单 TCP 连接上跑多 stream，**单个 TCP 连接的接收缓冲区是共享的**。如果客户端 A 不读，TCP 层接收窗口（recv window）很快归零，**TCP 通告 RWND=0 给服务端**。服务端 TCP 发不出任何字节，**整个连接所有 stream 都卡住**。

如果用了同一连接的还有 HTTP/2 stream-level flow control，stream 自己的窗口也会被卡死。

注意：这不只是 HTTP/2 的问题，HTTP/3 也类似 —— 但 HTTP/3 的 connection-level flow control 单独控制，且 stream 独立的丢包恢复让"局部慢消费"不会蔓延到全连接。但**应用层 connection-level window 卡死的话，HTTP/3 也会全连接受影响**。

更糟的是 NAT/中间代理：如果 NAT 设备把"长时间没数据"的连接定时关闭（NAT 表项过期），连接静默断开，应用层不知情。

#### 修复

**方案 A：客户端永远配读超时 + 主动断**

```java
// Java OkHttp 例子
OkHttpClient client = new OkHttpClient.Builder()
    .readTimeout(Duration.ofSeconds(10))
    .pingInterval(Duration.ofSeconds(30))   // 主动 PING 维护活性
    .build();
```

`pingInterval` 让客户端定期发 PING 帧，30s 没收到 PONG 就主动断。

**方案 B：服务端每 stream 独立超时控制**

Nginx：

```nginx
http2_idle_timeout 30s;
http2_recv_timeout 10s;       # 单帧接收超时
keepalive_timeout 60s;
```

**方案 C：用多个 HTTP/2 连接**（连接池）

不要把所有客户端流量塞到一个连接。后端如果是 gRPC 服务，连接池配多连接 + round-robin。

#### 验证

模拟慢消费客户端：

```bash
# nc 故意不读，但保持连接
nc -l 0.0.0.0 8080 &
sleep 9999
```

看其他客户端 P99 是否受影响。修复前会受到，修复后不会。

> **口诀**：HTTP/2 单连接是**双刃剑**。慢消费 / 阻塞 / 慢客户端会拖累整连接。**必配 PING + 客户端写超时 + 连接池多连接**。

---

### 案例 ③ 中间设备 NAT 超时导致 HTTP/2 连接静默断开

#### 现象

线上长连接 gRPC（基于 HTTP/2）服务，运营商网络下偶发"调用 X 秒后才报错"。客户端日志显示 RST、连接被对端关闭。但服务端日志显示该连接早在几分钟前就 idle，自己没主动关。

#### 根因

中间盒（电信级 NAT、家用路由器、企业防火墙）通常对**长时间空闲**的 TCP 连接做"超时清理"：
- 家用路由器：通常 5 分钟
- 企业 NAT：可能 30~60 秒（更激进，防 NAT 表项耗尽）
- 4G/5G 运营商网络：**通常更激进，10~60 秒**

NAT 表项被清后，连接的 IP-port 映射没了。后续任意一方发包，NAT 直接丢弃或回 RST。

HTTP/2 默认**不发心跳**。如果业务也没数据传，连接看似活着实则已死。

更隐蔽：很多 NAT 在表项清理时**不通知任何一方**，应用层完全不知道。下次发包才"发现"连接坏了，但此时业务请求已经积压。

#### 修复

**方案 A：HTTP/2 PING 帧**

PING 帧（type=0x6）是 HTTP/2 协议级心跳。客户端每 N 秒发一次，服务端必须回 PONG。

Nginx 配置：

```nginx
http2_idle_timeout 60s;        # 空闲超时
http2_max_requests 1000;       # 单连接最多请求数
```

OkHttp / Netty / gRPC 的客户端都有 `keepAliveTime` 配置。gRPC：

```java
ManagedChannelBuilder.forAddress(...)
    .keepAliveTime(30, TimeUnit.SECONDS)
    .keepAliveTimeout(10, TimeUnit.SECONDS)
    .keepAliveWithoutCalls(true)   // 即使无请求也心跳
    .build();
```

**方案 B：TCP keepalive**

OS 层心跳，更底层但不细粒度：

```bash
# Linux 默认 7200s（2 小时）才开始探测，太长
sysctl -w net.ipv4.tcp_keepalive_time=60
sysctl -w net.ipv4.tcp_keepalive_intvl=10
sysctl -w net.ipv4.tcp_keepalive_probes=3
```

应用层显式开启：

```java
socket.setKeepAlive(true);   // 才会用上 OS 的 keepalive
```

**方案 C：换 HTTP/3**

HTTP/3 用 UDP，但 QUIC 协议层有 PING 帧，且 connection ID 抗 NAT 切换，对中间盒友好。

#### 验证

```bash
# 用 conntrack 看 NAT 表项
conntrack -L | grep <connection-tuple>

# 模拟 NAT 超时：iptables 故意丢一段时间的包后再恢复
# 看应用层是否能在 PING 失败时及时检测到
```

> **口诀**：长连接必发心跳。**HTTP/2 用 PING 帧，TCP keepalive 是兜底**。NAT 不通知你连接坏了，自己心里要有数。

---

### 案例 ④ 客户端不支持 HTTP/3 时的 fallback 策略

#### 现象

切到 HTTP/3 后，监控发现 5% 的请求**走了 HTTP/1.1**，业务团队问"为什么有些用户没用 HTTP/3"。

#### 根因

浏览器对 HTTP/3 的发现机制是 **Alt-Svc**（Alternative Service）：

```http
HTTP/1.1 200 OK
Alt-Svc: h3=":443"; ma=86400
```

这个头告诉客户端"我同时也支持 HTTP/3，端口 443"。客户端收到后**下一次**才尝试 HTTP/3。但：

- 第一次请求必走 HTTP/2 或 HTTP/1.1
- 客户端可能不支持 HTTP/3（老浏览器、中间代理）
- 客户端尝试 HTTP/3 失败（UDP 被拦），会回退到 HTTP/2，且**短期内不再尝试 HTTP/3**

更糟：如果 ALPN 协商失败（见案例 ⑤），整体可能掉到 HTTP/1.1。

#### 修复

**方案 A：广播 Alt-Svc + HTTP/3 健康监控**

服务端持续通告 Alt-Svc，并监控**实际 HTTP/3 命中率**。健康指标：

```
http3_request_ratio = h3_requests / total_requests
```

正常应该 > 50%（视用户分布）。如果突然下跌，可能：
- CDN 节点 UDP 故障
- ISP 临时拦截 UDP 443

**方案 B：用 SVCB / HTTPS DNS 记录**

新一代发现机制（RFC 9460）：直接在 DNS 层告诉客户端"这个域名支持 HTTP/3"，**不需要先经过一次 HTTP/1.1 或 2**。

```
example.com.   IN  HTTPS  1 . alpn="h3,h2"
```

支持的客户端（Chrome 109+、Safari）会跳过 HTTP/1.1/2 直接尝试 HTTP/3。

**方案 C：客户端配 happy eyeballs**

并行尝试 HTTP/3 和 HTTP/2，谁先 ready 用谁。Cronet（Chrome 网络栈库）支持。

#### 验证

```bash
# 用 curl 强制 HTTP/3
curl --http3 -I https://example.com

# 服务端监控按协议分组 QPS
```

> **口诀**：**Alt-Svc 是 HTTP/3 的"广播"机制**，第一次请求总走 HTTP/2。SVCB DNS 记录是新一代解法。监控 h3_ratio。

---

### 案例 ⑤ ALPN 协商失败回到 HTTP/1.1

#### 现象

某 HTTPS 服务在 Java 客户端跑得好，在 Go 客户端却**全部走了 HTTP/1.1**。抓包看 TLS 握手成功，但 ALPN 没协商出 h2。

#### 根因

ALPN（Application-Layer Protocol Negotiation，RFC 7301）是 TLS 握手中的一个扩展。客户端在 ClientHello 里带：

```
ALPN: h2, http/1.1
```

服务端选一个回："我用 h2"。**如果服务端不支持 ALPN 或没配 h2**，会回 http/1.1（兜底），客户端就走 HTTP/1.1。

常见配置错误：
- Nginx 没装 OpenSSL 1.0.2+，ALPN 缺失
- Tomcat 没配 `<UpgradeProtocol className="org.apache.coyote.http2.Http2Protocol" />`
- 中间代理（HAProxy 老版本）剥离了 ALPN 扩展
- 服务端证书只有一个 SAN，不匹配客户端 SNI

#### 修复

**Nginx**：

```nginx
server {
    listen 443 ssl http2;       # 必须显式 http2
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:...;
    # OpenSSL 必须 >= 1.0.2 才支持 ALPN
}
```

**Tomcat**：

```xml
<Connector port="443" protocol="org.apache.coyote.http11.Http11AprProtocol"
           SSLEnabled="true">
    <UpgradeProtocol className="org.apache.coyote.http2.Http2Protocol"/>
    <SSLHostConfig>
        <Certificate certificateFile="..." />
    </SSLHostConfig>
</Connector>
```

**Spring Boot**：

```yaml
server:
  http2:
    enabled: true
  ssl:
    enabled: true
```

**HAProxy**：

```
frontend https
    bind *:443 ssl crt /etc/ssl/cert.pem alpn h2,http/1.1
```

#### 验证

```bash
# openssl 看协商出了什么
openssl s_client -alpn h2,http/1.1 -connect example.com:443 < /dev/null 2>&1 | grep ALPN
# 输出 "ALPN protocol: h2" 才是对的

# curl 看协议
curl -v https://example.com 2>&1 | grep "ALPN"
```

> **口诀**：HTTP/2 over TLS 强制依赖 ALPN。**Nginx 要 http2 关键字，Tomcat 要 UpgradeProtocol，OpenSSL 要 >= 1.0.2**。中间任何一环掉链子都会回 HTTP/1.1。

---

### 案例 ⑥ gRPC over HTTP/2 的连接复用问题

#### 现象

某后台用 gRPC 调用下游服务，单实例 QPS 上升到 5 万后，**响应延迟 P99 突然飙升到 800ms**，下游服务负载却很低。

#### 根因

gRPC 走 HTTP/2，**默认只用一个 TCP 连接**。HTTP/2 单连接的 `MAX_CONCURRENT_STREAMS` 默认 100~1000（依实现）。当并发请求 > MAX_CONCURRENT_STREAMS 时，新请求**排队等待已有 stream 完成**。

更糟：HTTP/2 的 stream ID 必须严格递增。客户端发起的 stream ID 是奇数（1、3、5...），服务端推送的是偶数。**当 stream ID 用尽（31 位）连接必须新建**。在 5 万 QPS 下，这个用尽时间从理论上的几十年缩短到几小时。

业务侧表现：连接池只有 1 个连接，所有请求挤一个连接的 stream 配额，**任意一个慢请求拖累所有人**（虽然多路复用避免了 HOL，但调度仍是单进程）。

#### 修复

**方案 A：连接池配多连接**

gRPC Java：

```java
ManagedChannel channel = ManagedChannelBuilder.forAddress(host, port)
    .usePlaintext()
    // 默认是 1 个连接，配多个 SubChannel
    .defaultLoadBalancingPolicy("round_robin")
    .build();
```

但 gRPC Java 的"多连接"实现有坑：DNS 多 A 记录才会建多连接，单 IP 还是 1 个。

更稳的做法：自己起多个 ManagedChannel，业务层 round-robin：

```java
List<ManagedChannel> channels = IntStream.range(0, 8)
    .mapToObj(i -> newChannel(host, port))
    .toList();

// 每次调用挑一个
ManagedChannel pick() {
    return channels.get(counter.getAndIncrement() % channels.size());
}
```

**方案 B：调大 MAX_CONCURRENT_STREAMS**

服务端：

```java
NettyServerBuilder.forPort(...)
    .maxConcurrentCallsPerConnection(10000)
    .build();
```

但这只是治标。

**方案 C：用 HTTP/3 / gRPC over QUIC**

实验性，部分实现已支持（Cloudflare、Envoy）。QUIC 单连接可以更激进地复用，且 stream ID 是 62 位，几乎无尽。

#### 验证

```bash
# netstat 看连接数
ss -tn dst :8080 | wc -l

# 监控 grpc 客户端的 active streams
```

> **口诀**：gRPC 单连接陷阱真实存在。**生产高 QPS 必须连接池 + 多连接**。

---

### 案例 ⑦ HTTP/3 在某些防火墙被阻断

#### 现象

CDN 切到 HTTP/3，发现**国内某地域用户 HTTP/3 命中率为 0**。该地域用户全部回退到 HTTP/2，监控看到的 HTTP/3 总命中率拉低 15%。

#### 根因

UDP 在很多企业、运营商、安全设备上被**默认拦截**：
- 企业防火墙：UDP 流量"看不懂"，简单粗暴丢
- 老式 NAT：UDP 状态保持成本高，限制并发
- 一些 ISP：UDP 443 被 QoS 降级（视频游戏混合流量优先）
- 部分国家政策：UDP 443 被深度包检测（DPI），疑似 VPN 隧道直接断

更隐蔽：UDP 包丢得**没有信号**。TCP 被拦会立即 RST，UDP 被丢就是"包没了"，QUIC 协议栈靠多次重试 + 超时检测才发现"这条路不通"。这个检测过程通常 2~5 秒。

#### 修复

**方案 A：保留 HTTP/2 fallback**

`Alt-Svc` 通告 HTTP/3，但 HTTP/2 服务必须保持运行。客户端 HTTP/3 失败会回退到 HTTP/2。

**方案 B：客户端"快速失败"配置**

QUIC 客户端可以配 idle timeout 短一点（5 秒）让失败更快：

```
QUIC idle_timeout = 5s   // 5 秒没响应就回退
```

**方案 C：监控按地域分组**

```
http3_success_rate by region, by ISP
```

发现某地域 < 30% 就把该地域**临时禁用 Alt-Svc 通告**，所有用户走 HTTP/2，避免 5 秒探测延迟。

**方案 D：双协议同测**

主动监测：服务端在每个地域部署探针，主动发 HTTP/3 请求测可达性。

#### 验证

```bash
# 模拟 UDP 被拦
sudo iptables -A OUTPUT -p udp --dport 443 -j DROP

# 看客户端会回退到什么协议、用了多少秒
curl -v --http3 https://example.com  # 失败，回 HTTP/2，看看延迟
```

> **口诀**：HTTP/3 部署必须留 HTTP/2 fallback。**UDP 被拦没信号，监控按地域 / ISP 分组**。

---

### 案例 ⑧ HPACK 动态表的安全侧信道（CRIME 类攻击）

#### 现象

安全团队报：某个用 HTTP/2 + HPACK 的 API，攻击者疑似能在不解 TLS 的情况下**推断出 Cookie 中的部分内容**。

#### 根因

HPACK 动态表压缩是**长度可观察**的：
- 同样的 header（已在动态表中）→ 1~2 字节
- 不同的 header → 几十字节

攻击者控制部分请求内容（比如某个 URL 参数），**注入猜测的字符串**。如果猜对了某个字段（如 Cookie 的前缀），HPACK 会用动态表压缩，整个请求大小突然变小。攻击者通过观察密文长度，**逐字符暴力破解 Cookie**。

这就是 CRIME（2012）/ HEIST（2016）/ TIME 攻击家族。HTTP/1.1 + gzip 也有类似漏洞。

HPACK 的设计者意识到了这点，引入了一个"never-indexed" 标志位（literal header field never indexed），但默认 Cookie 等敏感字段是否标 never-indexed **取决于实现**。Nginx 默认行为：所有未在静态表的 header 都进动态表。

#### 修复

**方案 A：敏感 header 标 never-indexed**

服务端：

```nginx
# Nginx 没有直接配置，需要源码补丁或用 OpenResty + Lua
```

应用层：明确把 Cookie 通过 `Set-Cookie` 走 `Secure; HttpOnly; SameSite=Strict`，并启用短周期 token 自动轮换。

**方案 B：CSRF 防御 + Origin 检查**

CRIME 攻击需要攻击者诱导用户的浏览器**发出包含猜测内容的请求**到目标站。严格的 CSRF 防御（token、SameSite）阻断攻击向量。

**方案 C：禁用 HPACK 压缩**

不现实（性能损失太大），但极端敏感场景可禁用动态表（HPACK SETTINGS 里 `SETTINGS_HEADER_TABLE_SIZE=0`）。

#### 验证

抓包看动态表大小是否随时间增长，关键字段是否被 indexed。Wireshark 可以解码 HPACK。

> **口诀**：**任何"基于明文长度可观察的压缩"都有侧信道风险**。HPACK / gzip / brotli 都是。Cookie 标 never-indexed + CSRF 防御 + 短期 token。

---

### 案例 ⑨ HTTP/3 0-RTT 重放被攻击者利用

#### 现象

某 API 上线 HTTP/3 + 0-RTT 后，安全团队报：**同一笔订单出现两次扣款**，攻击者似乎"重放"了下单请求。日志看下单是同一 trace_id，时间间隔几百毫秒。

#### 根因

0-RTT 的工作方式：客户端首次建连时拿到 NewSessionTicket，**重连时第一个 packet 用 PSK 加密 + 直接带应用数据**。这个数据包是**网络可见的密文**，攻击者无法解密但可以**抓下来重放**。

服务端解密后会执行 0-RTT 数据里的请求。**TLS 1.3 spec 明确要求服务端实现 anti-replay**（要么单次有效的 ticket，要么维护已见过的 nonce 集合），但工程上：
- 单次有效的 ticket → 全局共享状态，分布式部署难
- nonce bloom filter → 有概率漏判
- 多服务器集群之间不同步 → 同一重放包打不同实例都成功

更糟的是有些反向代理（早期 Envoy 配置）**默认不做 anti-replay**，依赖应用层幂等。

#### 修复

**方案 A：0-RTT 仅用于幂等请求**（最稳）

在网关层强制：

```
配置：仅 GET / HEAD 允许 0-RTT
其他方法：必须等 1-RTT 完成握手再处理
```

Cloudflare、Fastly 默认就是这个策略。

**方案 B：应用层幂等 token**

每个写请求带客户端生成的 idempotency-key，服务端 Redis 去重：

```http
POST /orders HTTP/3
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
```

服务端：

```
key := req.Header.Get("Idempotency-Key")
if redis.SetNX(key, "1", 24h) == false:
    return 409 Conflict
```

**方案 C：禁用 0-RTT**

极端敏感场景：

```
Nginx: ssl_early_data off;
```

代价：失去 1 RTT 的延迟优势。

#### 验证

```bash
# 用 openssl 模拟 0-RTT 重放
openssl s_client -connect example.com:443 -tls1_3 -sess_out sess.pem
echo -e "GET /admin/dosomething HTTP/1.1\r\nHost: example.com\r\n\r\n" | \
    openssl s_client -connect example.com:443 -tls1_3 -sess_in sess.pem -early_data /dev/stdin

# 重发同样的 early_data 看是否被拒
```

> **口诀**：0-RTT = "性能 vs 安全"取舍。**只允许幂等请求 + 应用层 idempotency-key + 监控可疑重复请求**。

---

### 案例 ⑩ HTTP/2 优先级配置错误导致核心资源被排后

#### 现象

用了 CDN 的 HTTP/2 后，发现页面 LCP（最大内容绘制）反而变慢。Chrome DevTools 看一些图片**比关键 CSS 先到达**。

#### 根因

HTTP/2 的优先级是**树形 + 权重**，但**实现散乱**：
- Chrome：CSS/字体高优先级、HTML 高、JS 中、图片低
- Firefox：用一棵更复杂的树
- Safari：基本不发优先级
- 服务端是否 honor 优先级**完全可选**

某些服务端（早期 Nginx 版本）干脆**忽略客户端发的 priority**，按 stream 创建顺序处理。结果浏览器先发的图片请求，比后发的 CSS 请求先得到响应。

更糟：CDN 中间层可能**重写或丢失优先级信息**。

#### 修复

**方案 A：升级到支持 RFC 9218 的服务**

新版 Cloudflare、Fastly 已支持 Extensible Priorities，用 HTTP 头表达：

```http
GET /critical.css HTTP/2
priority: u=1, i
```

`u=0` 最高，`u=7` 最低。`i`（incremental）= 是否流式渐进展示。

**方案 B：Nginx 1.18+ 配 http2_chunk_size**

```nginx
http2_chunk_size 8k;
```

帧分得更小，让交错调度更细粒度，减少大文件占满"窗口"的影响。

**方案 C：换 HTTP/3 + 优先级头**

HTTP/3 完全用 RFC 9218，更一致。

**方案 D：内联关键路径**

最稳的"优先级"是直接内联 critical CSS 进 HTML，不依赖任何协议层调度。

#### 验证

Chrome DevTools Performance / WebPageTest 看资源加载顺序：

```
HTML  ──────►
CSS    ──►          ★ 应该比图片先到
JS         ──►
img            ──►
```

> **口诀**：HTTP/2 的优先级是"愿景"不是"机制"。**真要保证关键资源先到，最稳是内联 + preload + 升级到 RFC 9218**。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 协议选择层
- [ ] 评估你的用户分布：移动 / 弱网占比高 → HTTP/3 优先；纯内网 / 稳定网络 → HTTP/2 够用
- [ ] HTTP/3 部署必须保留 HTTP/2 fallback，不能"All in"
- [ ] 服务端 OpenSSL 升级到 1.0.2+（ALPN 支持）
- [ ] TLS 1.3 优先；TLS 1.2 仅做兼容性兜底

### 连接管理层
- [ ] HTTP/2 客户端配 PING（Java OkHttp、Go HTTP2、gRPC 都有）
- [ ] PING 间隔 30s 以内，避开 NAT 60s 默认超时
- [ ] TCP keepalive 调到 60s，作为兜底（OS 默认 7200s 太长）
- [ ] gRPC / 高 QPS 客户端用连接池多连接，不能信赖单连接
- [ ] HTTP/2 服务端调 `MAX_CONCURRENT_STREAMS`（默认 100，生产建议 1000+）

### 流控与窗口
- [ ] HTTP/2 `INITIAL_WINDOW_SIZE` 调到 BDP 级别（万兆 + 50ms RTT 至少 16MB）
- [ ] Nginx：`http2_recv_buffer_size 256k;`
- [ ] 单连接慢消费检测：客户端读超时 + 服务端 stream 超时
- [ ] Backend 监控 stream 积压告警

### 安全层
- [ ] HTTP/3 必须 TLS 1.3，禁用 TLS 1.2 fallback
- [ ] 0-RTT 仅用于幂等请求（GET、HEAD），POST/PUT 禁用
- [ ] HPACK 敏感字段标 never-indexed（Cookie、Authorization）
- [ ] 配 CSRF token + SameSite=Strict 防 CRIME 类侧信道
- [ ] 关闭 Server Push（已废弃，没好处）

### 部署与运维
- [ ] CDN / 反代检查 ALPN 是否正确传透
- [ ] 监控按协议分组：h1/h2/h3 各自的 QPS、P99、错误率
- [ ] 监控按地域 / ISP 分组 HTTP/3 命中率，< 30% 临时禁用 Alt-Svc
- [ ] HTTP/3 服务的 UDP 接收缓冲区调大：`sysctl -w net.core.rmem_max=2500000`
- [ ] 关键资源用 `<link rel="preload">` + `103 Early Hints`，不用 Server Push
- [ ] Critical CSS 内联进 HTML

### 调试与监控
- [ ] curl --http3 / openssl s_client / Wireshark / nghttp 至少会用一种
- [ ] Chrome `chrome://net-export/` 抓 NetLog，事后分析协议级问题
- [ ] 服务端 access log 记录协议版本（`$server_protocol`）
- [ ] 抓包能解 TLS（SSLKEYLOGFILE 环境变量配 Chrome / Firefox）

---

## 6. 还能往下挖什么

读完上面只是"协议演进"的视野。更深方向：

### HTTP/3 0-RTT 的安全风险（重放攻击）
- 0-RTT 数据是**可重放的**：攻击者抓包后重发，服务端无法区分
- TLS 1.3 的 anti-replay 机制：server 维护"最近见过的 ticket + nonce"集合
- 实际上很多实现用 bloom filter，**有概率漏判**
- 工程结论：0-RTT 仅用于幂等请求（GET、HEAD），写操作禁用

### WebTransport（基于 QUIC）
- W3C 草案，提供"双向流 + 数据报"的浏览器 API，比 WebSocket 更通用
- 用 QUIC 的 stream + datagram，**支持不可靠传输**（适合实时游戏、音视频）
- 浏览器支持：Chrome 97+、Firefox 在路上
- 思考：什么时候用 WebTransport vs WebSocket vs WebRTC？

### WebSocket 与 HTTP/3 的协作
- RFC 9220：WebSocket over HTTP/3
- 实现：用 QUIC 的双向 stream + HTTP/3 的扩展 CONNECT 方法
- 优势：连接迁移、独立 stream 流控；劣势：兼容性差，2024 年仍有限部署
- 大部分场景：WebTransport 是 WebSocket 的"自然继承者"，不必非走 WebSocket over H3

### gRPC 在 HTTP/2 上的特殊用法
- gRPC 把 HTTP/2 当"流的容器"用，所有请求是 HTTP/2 stream + 自定义编码
- gRPC-Web：浏览器没法直接用 HTTP/2 trailers，需要代理转译（Envoy / grpc-web proxy）
- gRPC 双向流（bidi streaming）= 一个 stream 双向 DATA 帧 + 半关闭语义

### Cloudflare / Fastly 的 QUIC 实测
- Cloudflare 公开了 quiche（Rust 实现的 QUIC）
- Fastly 主推 quicly（C 实现）
- 两家都在做 BBR v2 + QUIC 的拥塞控制实验
- 关键工具：[Cloudflare Radar](https://radar.cloudflare.com/) 看全球 HTTP/3 占比

### 反向代理（Nginx / Envoy）的 HTTP/3 支持
- Nginx 1.25+ 实验性支持 HTTP/3
- Caddy 默认开 HTTP/3，最易用
- Envoy 1.18+ 支持 QUIC，生产级
- HAProxy 2.6+ 实验支持
- 选型考虑：稳定性 + 配置生态 + 运维熟悉度

### 拥塞控制
- TCP CUBIC（Linux 默认）vs BBR（Google 主推）
- QUIC 的拥塞控制是协议**应用层的**，可以每个连接独立选择算法
- BBR v2 在弱网更优，但生产部署仍需谨慎

### HTTP/3 的 CPU 开销
- 用户态加解密：每个 packet 一次，不像 TCP 可以批处理
- 解决方案：内核态 QUIC（实验性的 `eBPF` 加速）、UDP GSO/GRO offload
- AWS 已经在 ELB 上做了硬件 QUIC offload

---

## 7. 引用

- RFC 7540 — HTTP/2: [https://www.rfc-editor.org/rfc/rfc7540](https://www.rfc-editor.org/rfc/rfc7540)
- RFC 9113 — HTTP/2 (修订版，废弃 7540)
- RFC 9114 — HTTP/3
- RFC 9000 — QUIC: A UDP-Based Multiplexed and Secure Transport
- RFC 9001 — Using TLS to Secure QUIC
- RFC 9002 — QUIC Loss Detection and Congestion Control
- RFC 7541 — HPACK: Header Compression for HTTP/2
- RFC 9204 — QPACK: Field Compression for HTTP/3
- RFC 9218 — Extensible Prioritization Scheme for HTTP
- RFC 9460 — Service Binding and Parameter Specification via the DNS (SVCB / HTTPS)
- RFC 8470 — Using Early Data in HTTP（0-RTT 安全考量）
- Cloudflare Blog: [How HTTP/3 and QUIC are revolutionizing the web](https://blog.cloudflare.com/http-3-from-root-to-tip/)
- Cloudflare Blog: [Even faster connection establishment with QUIC 0-RTT resumption](https://blog.cloudflare.com/even-faster-connection-establishment-with-quic-0-rtt-resumption/)
- Daniel Stenberg, *HTTP/3 Explained*: [https://http3-explained.haxx.se/](https://http3-explained.haxx.se/)
- Ilya Grigorik, *High Performance Browser Networking*（O'Reilly，免费在线）
- Mark Nottingham, *RFC 9218: Why HTTP/2 priorities were redesigned*
- Google QUIC team papers (SIGCOMM 2017): *The QUIC Transport Protocol*
- Cloudflare quiche: [https://github.com/cloudflare/quiche](https://github.com/cloudflare/quiche)
- Fastly quicly: [https://github.com/h2o/quicly](https://github.com/h2o/quicly)

---

> 配套：[网络与 RPC 主题](/article/network/) | [TCP 状态机深入](/article/network/tcp-state) | [Netty EventLoop](/article/network/netty-eventloop)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
