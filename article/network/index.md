# 网络与 RPC

> TCP / HTTP / RPC 不只是面试题。这一节要讲的是：写网络代码、调 Netty 参数、定位连接堆积时**真正用得上的那些细节**。

---

## 为什么单独开这块

绝大多数 Java 后端工程师对网络的认知止于"TCP 三次握手 + HTTP 状态码"。但你写一个高性能 RPC 客户端、调 Netty 的 `SO_BACKLOG` 和 EventLoop 数、定位 `CLOSE_WAIT` 堆积时，需要的是：
- TCP 状态机的每个状态什么时候转
- HTTP/1.1 vs 2 vs 3 在多路复用上的本质差别
- Netty 的 EventLoop 模型、零拷贝、堆外内存
- gRPC / Dubbo 的协议设计权衡

这一节按 [桃花岛深度模板](/STYLE) 切，每篇深文都包含**抓包验证 + 内核源码 + 真实事故复盘**。

---

## 已完成的深文

- ✅ **[Netty EventLoop 与 ByteBuf —— Reactor 模式的工业级实现](/article/network/netty-eventloop)** · 1817 行
- ✅ **[TCP 状态机深入 —— SYN_FLOOD 到 CLOSE_WAIT 堆积](/article/network/tcp-state)** · 1420 行
- ✅ **[HTTP/1 → 2 → 3 演进 —— 多路复用与 QUIC](/article/network/http-evolution)** · 1268 行

---

## 规划中的深文

### 1. TCP 状态机与生产排查 ✅ 已完成 → [阅读深文](/article/network/tcp-state)

**核心问题**：为什么 `netstat` 看到几千个 CLOSE_WAIT？TIME_WAIT 真的需要调 `tcp_tw_reuse` 吗？SYN_FLOOD 怎么用 syncookie 防御？

**写作骨架（5 层）**：
- **原理推导**：11 个 TCP 状态的转换图（含异常路径）；为什么 TIME_WAIT 必须 2MSL；为什么 CLOSE_WAIT 是应用 Bug
- **内核源码**：Linux 5.x `tcp_input.c` 中 `tcp_rcv_state_process` 的状态机切片；`tcp_close` 的关闭流程
- **抓包实测**：用 tcpdump + Wireshark 验证四次挥手；模拟 backlog 满后的丢包
- **生产案例**：CLOSE_WAIT 堆积导致 OOM（应用未关连接）、TIME_WAIT 耗尽端口、SYN backlog 满导致连接丢失
- **延伸**：BBR 拥塞控制、Linux `tcp_*` sysctl 参数全梳理、eBPF 排查 TCP

### 2. HTTP/1.1 → 2 → 3 的演进与权衡 ✅ 已完成 → [阅读深文](/article/network/http-evolution)

**核心问题**：HTTP/2 的多路复用真的快？为什么有 HOL Blocking？QUIC（HTTP/3）凭什么能解决 TCP 的队头阻塞？

**写作骨架**：
- **原理推导**：1.1 的 keep-alive 与 pipeline 局限 → 2 的二进制分帧 + Stream → 3 的 QUIC 解耦传输
- **源码切片**：Netty HTTP/2 codec 的帧解析；OkHttp 客户端的连接复用
- **抓包实测**：用 `curl --http2` / `--http3` 抓包对比；高丢包率下 HTTP/2 vs 3 的性能曲线
- **生产案例**：HTTP/2 服务端推送被滥用导致带宽爆炸、连接静默断开（中间设备 NAT 超时）、客户端不支持 fallback
- **延伸**：gRPC over HTTP/2 的特殊用法、Cloudflare 的 QUIC 实测报告

### 3. Netty EventLoop 与 ByteBuf ✅ 已完成 → [阅读深文](/article/network/netty-eventloop)

**核心问题**：Netty 为什么快？`bossGroup` 和 `workerGroup` 怎么设？`PooledByteBufAllocator` 的内存泄漏怎么定位？

深文涵盖：Reactor 4 变体 / Netty 主从模型 / NioEventLoop.run + SelectedSelectionKeySet 优化 / Selector 重建绕 epoll bug / PooledByteBufAllocator jemalloc-like 实现 / native transport / 9 个生产事故（refCnt 泄漏、EventLoop 阻塞、HTTP/2 内存爆炸、SO_LINGER close 阻塞等）/ 25+ 条上线清单 / 7 个延伸方向（含 io_uring、虚拟线程取代 EventLoop 之辩）。

### 4. RPC 框架对比 —— Dubbo / gRPC / Tars / 自研

**核心问题**：Dubbo 3 为什么改用 Triple over HTTP/2？gRPC 在 Java 上的真实性能？为什么大厂都在自研 RPC？

**写作骨架**：
- **原理推导**：RPC 的核心组件（编解码、传输、序列化、服务发现、负载均衡、熔断）；同步/异步/流式的接口语义
- **源码切片**：Dubbo 3 的 Triple 协议帧格式；gRPC 的 `ManagedChannel` 与 `LoadBalancer` 实现；Tars 的 `tarsdecode`
- **实测数据**：Dubbo 2.7 vs 3、gRPC、Tars 在小包 / 大包 / 流式三种场景的吞吐与 CPU
- **生产案例**：序列化兼容性事故（Hessian / Protobuf 字段顺序）、连接池配置错误导致雪崩、超时配置在多层调用下的"超时叠加"
- **延伸**：Service Mesh 下 RPC 的演变（Sidecar 模式）、Apache Dubbo 与 Spring Cloud 的真实差距

### 5. 长连接与心跳 —— "网关连接断了为什么客户端没感知"

**核心问题**：你的 IM / 推送服务长连接经常"假死"，客户端以为还在线，服务端早就 RST 了。怎么设计心跳？为什么 KeepAlive 不够用？

**写作骨架**：
- **原理推导**：TCP KeepAlive 的局限（默认 2 小时、应用层不感知）；应用层心跳的必要性
- **源码切片**：Netty `IdleStateHandler` 的实现；MQTT keep-alive 协议
- **实测数据**：不同心跳间隔下的"假死率"；NAT 设备的连接超时实测（家用路由器 / 运营商 NAT）
- **生产案例**：客户端断网无感知、服务端心跳风暴打爆 CPU、心跳带业务数据导致带宽放大
- **延伸**：QUIC 的连接迁移、WebSocket Ping/Pong、HTTP/2 的 PING 帧

### 6. 限流的工程实现 —— 令牌桶 / 漏桶 / 滑动窗口的代码层

**核心问题**：Guava RateLimiter 适合 RPS 限流，但分布式怎么办？滑动窗口算法在 Redis 上有几种写法？为什么 Sentinel 比 Hystrix 更适合现代场景？

**写作骨架**：
- **原理推导**：4 种主流算法的几何对比；本地 vs 分布式限流的本质差别
- **源码切片**：Guava `SmoothBursty` 的 lazy 设计；Sentinel 的滑动窗口；Redis lua 脚本实现的滑动窗口
- **实测数据**：不同算法在突发流量下的"误杀率"和"漏过率"
- **生产案例**：限流过严导致正常流量被拒、Redis 限流脚本性能瓶颈、分布式限流的"小数定律"陷阱
- **延伸**：自适应限流（Netflix concurrency-limits）、TCP BBR 思想在 RPC 限流中的应用

---

## 关联现有文章
- [Java 并发编程](/article/java/concurrent) —— 服务端处理模型基础
- [线程池生产实战](/article/java/threadpool) —— 与 EventLoop 模型对比

## 想看哪一篇？

[告诉我](/article/about) 你最想先看哪个主题的深文，我会优先排期。
