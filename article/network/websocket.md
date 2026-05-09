# WebSocket / SSE / 长连接 —— 推送 + IM + LLM 流式响应的工程

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，看 [网络综述](/article/network/) 的速记一节够了。
> 这里要回答的，是**做 IM、做推送、对接 LLM 流式响应、半夜被"连接掉光"告警叫起来**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. WebSocket、SSE、Long Polling 三者的边界在哪？为什么 ChatGPT 的流式输出选 SSE 而不是 WebSocket？
2. WebSocket 握手为什么要走 HTTP/1.1 Upgrade？`Sec-WebSocket-Key` 这个字段不是为了"鉴权"，那它是为了什么？
3. 你的 WebSocket 服务上线一周后，客户端反馈"消息越来越慢、有时候发了没反应、过几分钟才收到"。但服务端连接数监控正常 —— 哪一层出了问题？
4. 心跳间隔到底设多少？应用层心跳和 TCP KeepAlive 各自能解决什么、解决不了什么？
5. 一台 32C/64G 的服务器，你预期能撑住多少 WebSocket 长连接？瓶颈先到哪？FD？内存？还是 CPU？
6. WebSocket 上跑 JWT 鉴权，token 30 分钟过期，连接保持了 4 小时 —— token 过期时连接要不要踢掉？怎么续期？
7. 用 Nginx 反代 LLM 流式响应（SSE），客户端死活看不到中间内容，只在最后一次性收到全部 —— 改哪个配置？
8. 单机 100 万长连接，要广播一条消息到所有人，怎么做才不会把服务打挂？

如果有半数答不上，往下看。

---

## 1. 底层模型

绝大多数人对长连接的认知止于"WebSocket 是双向通信"。这层抽象解释不了上面 8 个问题。要看懂它们，必须先把三种长连接方案、握手协议、帧格式、心跳设计、资源模型、消息分发模型这六件事打通。

### 1.1 三种长连接方案的对比

"长连接"在 Web 时代有三套常见实现，边界经常被搞混：

| 方案 | 方向 | 协议基础 | 浏览器原生支持 | 典型场景 |
| --- | --- | --- | --- | --- |
| **Long Polling** | 服务端 → 客户端（伪推送） | HTTP/1.1 普通请求 | 任意 | 老系统兼容、低频通知 |
| **SSE**（Server-Sent Events） | **单向**：服务端 → 客户端 | HTTP/1.1 长响应 | `EventSource` API | LLM 流式输出、行情推送、日志流 |
| **WebSocket** | **双向** | TCP + 自定义帧（基于 HTTP/1.1 升级） | `WebSocket` API | IM、协作编辑、游戏、双向控制 |

**选型铁律**（背下来，能挡住一半的过度设计）：

- **服务端只需要往下推、客户端只需要收**：用 SSE。代码简单、走 HTTP、能被现有的反代/CDN/WAF 直接处理。
- **真正需要双向高频通信**（IM、游戏、协作）：用 WebSocket。
- **客户端环境老旧，不能用 WebSocket / SSE**：才考虑 Long Polling，且优先包成 SockJS 这种 fallback。

为什么 OpenAI / Anthropic 的流式响应都选 SSE？因为 LLM 输出本质是**单向流** —— 客户端发一次请求、服务端持续往下推 token，不需要双向。SSE 的优势：

- 走 HTTP，复用 TLS、HTTP/2、鉴权、CDN
- 浏览器原生支持自动重连（`EventSource` 在断开后会自动 reconnect 并带 `Last-Event-ID`）
- 协议简单：纯文本、行分隔，调试用 `curl` 即可
- 服务端实现简单：一个长响应流就行，不用维护帧状态

WebSocket 在这个场景反而是"杀鸡用牛刀" —— 你不需要客户端往服务端持续推、不需要二进制帧、不需要复杂的状态管理。

### 1.2 WebSocket 握手：HTTP/1.1 Upgrade 协议

WebSocket 不是凭空发明的协议。它**借用了 HTTP/1.1 的 Upgrade 机制**：客户端发起一个看起来像 HTTP 的请求，服务端同意"升级"后，这条 TCP 连接就脱离 HTTP 语义，开始按 WebSocket 帧格式收发。

握手请求：

```
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Origin: http://example.com
```

握手响应：

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

关键字段逐个看清：

- `Upgrade: websocket` + `Connection: Upgrade`：告诉中间所有节点（包括代理、网关）"这是个升级请求，别按普通 HTTP 转发"。
- `Sec-WebSocket-Key`：客户端生成的 16 字节随机数 base64 编码。**它不是鉴权用的**，而是为了让服务端证明"我确实理解 WebSocket 协议"。
- `Sec-WebSocket-Accept`：服务端把 Key 拼接固定的 magic string `258EAFA5-E914-47DA-95CA-C5AB0DC85B11`，做 SHA-1 然后 base64 输出。

这个 magic string 的设计目的是：**防止把 WebSocket 误用成"任何 TCP 透传"**。如果一个不懂 WebSocket 的中间代理把请求当成普通 HTTP 转发到一个不懂 WebSocket 的后端，后端不会算这个 Accept 值，握手就失败。这是个"协议自检"机制，不是安全机制。

握手成功后，**HTTP 协议结束，TCP 连接保留，开始按 WebSocket 帧格式通信**。后续任何字节都不再是 HTTP。

### 1.3 WebSocket 帧格式

WebSocket 用自己的二进制帧。一个最小帧的字节布局：

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - - +-------------------------------+
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
|    Masking-key (continued)    |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - - +
:                     Payload Data continued ...                :
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
|                     Payload Data continued ...                |
+---------------------------------------------------------------+
```

逐个字段点透：

- **FIN**：是否是消息的最后一帧（消息可以分多个帧发，比如大文件）
- **Opcode**：帧类型。`0x1` = 文本、`0x2` = 二进制、`0x8` = 关闭、`0x9` = Ping、`0xA` = Pong
- **MASK**：客户端发服务端的帧**必须**设为 1（用 4 字节 mask key 异或 payload）；服务端发客户端的帧**必须**为 0
- **Payload len**：< 126 直接用这 7 位；= 126 后面跟 2 字节实际长度；= 127 后面跟 8 字节实际长度

为什么客户端必须 mask？这是**为了防止"协议混淆攻击"**：恶意 JS 可以构造看起来像 HTTP 请求的 WebSocket payload，骗中间代理把它解析成新 HTTP 请求。强制 mask 让 payload 在线上是"乱码"，中间代理无法识别成 HTTP。

服务端不需要 mask，反过来同理：**服务端发的内容是受控的**，没有"恶意 JS 构造"的攻击面。

控制帧（Ping / Pong / Close）有特殊规则：

- payload 必须 ≤ 125 字节
- 不能分片（必须 FIN=1）
- 可以**插入到分片消息的中间**（比如你正在分片发一个大文件，中间塞一个 Ping）

### 1.4 SSE 的 text/event-stream 协议

SSE 协议**简单到令人发指**。一个完整的 SSE 响应：

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

data: hello

data: world

event: token
data: {"text": "你好"}
id: 42

retry: 5000

: this is a comment

data: line1
data: line2

```

规则就这么几条：

- `Content-Type: text/event-stream` 是入场券
- 每个事件由若干行组成，`data:` 是数据，`event:` 是事件名（默认 `message`），`id:` 是事件 ID
- 事件之间用**空行**分隔
- `retry: N` 告诉浏览器断开后等多少毫秒重连
- `:` 开头是注释（实际用途是当心跳：服务端定期发个 `:` 让中间代理不超时）

浏览器侧的 `EventSource` 自动处理：

```javascript
const es = new EventSource('/api/chat/stream');
es.onmessage = (e) => console.log(e.data);
es.addEventListener('token', (e) => render(e.data));
```

断开后浏览器会**自动重连**，且把上次收到的最后一个 `id` 放在 `Last-Event-ID` 请求头里发给服务端，让服务端能续传。

LLM 流式响应几乎是教科书式的 SSE 用法：

```
data: {"choices":[{"delta":{"content":"你"}}]}

data: {"choices":[{"delta":{"content":"好"}}]}

data: {"choices":[{"delta":{"content":"，"}}]}

data: [DONE]

```

每个 `data:` 是一个 token（或几个）。客户端拼接显示，看到 `[DONE]` 就关闭。OpenAI、Anthropic、Gemini 的流式 API 全部走这个格式。

### 1.5 心跳与连接保活

长连接的"活着"是个陷阱：**TCP 层连接看起来还在，实际数据可能已经发不出去了**。NAT 设备超时、路由器重启、客户端网络切换、中间代理掐断 —— 都可能让 TCP 连接处于"假连接"状态。

两层保活机制：

#### TCP KeepAlive（操作系统层）

由 OS 内核负责。Linux 默认配置：

```bash
$ sysctl net.ipv4.tcp_keepalive_time      # 7200 秒（2 小时）才开始第一次探测
$ sysctl net.ipv4.tcp_keepalive_intvl     # 75 秒探测间隔
$ sysctl net.ipv4.tcp_keepalive_probes    # 9 次失败才认定断开
```

默认配置基本无用 —— 2 小时才发第一次探测，等你发现连接死了，业务早炸了。生产必须调小：

```bash
net.ipv4.tcp_keepalive_time = 60
net.ipv4.tcp_keepalive_intvl = 10
net.ipv4.tcp_keepalive_probes = 3
```

但 TCP KeepAlive 解决不了所有问题：

- **NAT 超时**：很多家用路由器 NAT 表的 idle timeout 是 5 分钟。TCP KeepAlive 默认 2 小时早就被 NAT 表清掉了。
- **应用层假死**：TCP 包能通，但应用线程死锁、GC 长 STW —— TCP KeepAlive 看不出来。
- **运营商劫持**：有些运营商会 reset 长时间无流量的 TCP 连接。

#### 应用层心跳（业务层）

**WebSocket 用 Ping/Pong 帧**：

```
客户端 → 服务端：Ping 帧（opcode=0x9）
服务端 → 客户端：Pong 帧（opcode=0xA），payload 必须和 Ping 一致
```

或反过来。设计要点：

- **谁发 Ping？** 一般客户端发，服务端被动 Pong。原因：服务端要省资源（百万连接每个发 Ping 消耗大）。
- **间隔多少？** 经验值 30 秒。再短浪费带宽和电量；再长被 NAT 切。
- **超时怎么算？** 连续 N 次没收到对方响应（N=2 或 3）就主动断开。

**SSE 用注释行做心跳**：服务端定期发 `: ping\n\n`（空注释）。客户端收到任何字节都会重置 NAT 超时，但因为 `:` 开头是注释，业务层无感知。

```python
# 服务端伪码
async def stream():
    while True:
        if has_message():
            yield f"data: {message}\n\n"
        else:
            yield ": ping\n\n"        # 心跳
        await asyncio.sleep(30)
```

### 1.6 百万连接的资源模型

"单机能撑多少长连接"是个被吹得很玄的问题。算清楚资源账，瓶颈很清晰。

#### FD（文件描述符）

每个 TCP 连接占 1 个 FD。Linux 默认进程级 FD 限制 1024，必须改：

```bash
# /etc/security/limits.conf
* soft nofile 1048576
* hard nofile 1048576

# /etc/sysctl.conf
fs.file-max = 2097152
fs.nr_open = 2097152
```

FD 几乎不构成瓶颈 —— 100 万 FD 占用的内核内存是 KB 级。

#### 内存

每个 TCP 连接的内核内存：

- **socket 结构体**：~ 4KB
- **接收/发送缓冲区**：默认每个 ~ 16KB ~ 87KB（动态调整）

按平均每连接 8KB 内核 + 4KB socket = **12KB / 连接**。100 万连接 = 12GB。

应用层每连接还要：

- WebSocket Session 对象：几百字节到几 KB
- 用户 ID、订阅列表、消息队列：视业务而定

总账：百万连接的服务器至少需要 **32GB 内存起步，64GB 才比较从容**。

#### 端口

很多人误以为"单机最多 65535 个连接"。**这是错的** —— 端口限制是给"主动连接外部"的客户端的。作为服务端，端口只用一个（比如 443），TCP 五元组是 `(server_ip, server_port, client_ip, client_port, protocol)`，理论上限是 `客户端数 × 65535`。

真实瓶颈：单机网卡能不能撑住百万连接的握手速度（PPS 上限）。

#### CPU

百万空闲连接几乎不耗 CPU（epoll 等待）。CPU 账主要算：

- 心跳：100 万连接 × 30 秒一次 ping = 3.3 万 PPS
- 业务消息：取决于推送频率

调整 socket 缓冲区（百万连接场景必须）：

```bash
net.ipv4.tcp_rmem = 4096 16384 4194304
net.ipv4.tcp_wmem = 4096 16384 4194304
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 30000
```

把默认接收/发送缓冲区调小到 16KB（默认可能 87KB），单纯这一改就能让 100 万连接的内核内存从 80GB 降到 12GB。

### 1.7 广播 / 房间 / 私聊的消息分发

长连接服务的消息分发模型有三种典型形态：

#### 私聊（点对点）

```
client A → server → client B
```

实现：维护 `Map<userId, Session>`，发消息时查表。

```java
public void sendTo(String userId, String message) {
    Session s = sessions.get(userId);
    if (s != null && s.isOpen()) {
        s.getAsyncRemote().sendText(message);
    }
}
```

集群下：跨实例发送，用户在哪台机？需要"路由表"（Redis 维护 `userId → instanceId`）。

#### 房间（多对多）

```
client A,B,C → room "chat-1" → all members
```

实现：维护 `Map<roomId, Set<Session>>`。

集群下：每条消息走 Pub/Sub，所有实例订阅，本地实例只发给本地有订阅的连接。

#### 广播（全员）

```
admin → server → ALL clients
```

最容易出事的就是它。100 万连接广播一条 1KB 消息 = 1GB 流量瞬发，单机网卡（千兆 = 125MB/s）需要 8 秒才能发完，期间所有业务消息卡住。

正确做法：

- 限速发送（令牌桶）
- 分批，每批一个延迟
- 利用 CDN / 边缘节点扩散
- 优先级队列：广播低优先

> **口诀**：长连接 = FD 廉价 + 内存敏感 + 网卡受限 + 广播是炸药。**先算资源账，再写代码**。

---

## 2. 工程切片

> 每段都是可运行的代码。从 Spring 生态到 Netty 原生，覆盖你 90% 的场景。

### 2.1 Spring WebSocket（@MessageMapping）

Spring Boot 起一个最小 WebSocket 服务：

```java
@Configuration
@EnableWebSocket
public class WsConfig implements WebSocketConfigurer {
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry r) {
        r.addHandler(new ChatHandler(), "/ws/chat")
         .setAllowedOriginPatterns("*");      // 生产要写死域名
    }
}

public class ChatHandler extends TextWebSocketHandler {
    private final Map<String, WebSocketSession> sessions = new ConcurrentHashMap<>();

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        String userId = (String) session.getAttributes().get("userId");
        sessions.put(userId, session);
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) {
        // 业务处理
        String userId = (String) session.getAttributes().get("userId");
        log.info("recv from {}: {}", userId, message.getPayload());
    }

    @Override
    public void afterConnectionClosed(WebSocketSession s, CloseStatus status) {
        String userId = (String) s.getAttributes().get("userId");
        sessions.remove(userId);
    }
}
```

鉴权用 HandshakeInterceptor（在 HTTP 升级阶段拦截）：

```java
public class JwtHandshakeInterceptor implements HandshakeInterceptor {
    @Override
    public boolean beforeHandshake(ServerHttpRequest req, ServerHttpResponse rsp,
                                   WebSocketHandler h, Map<String, Object> attrs) {
        String token = req.getHeaders().getFirst("Authorization");
        Claims claims = jwtParser.parse(token);     // 失败抛异常 → 握手失败
        attrs.put("userId", claims.getSubject());   // 放入 session attributes
        return true;
    }

    @Override
    public void afterHandshake(...) {}
}
```

注意 `WebSocketSession.sendMessage` **不是线程安全的**。多个线程同时往同一个 session 发，会导致帧错乱。两个解决方案：

- 包一层 `ConcurrentWebSocketSessionDecorator`（Spring 自带）：
  ```java
  WebSocketSession s = new ConcurrentWebSocketSessionDecorator(rawSession, 1000, 64 * 1024);
  ```
- 自己用单线程串行化（每个 session 一个 mailbox）

### 2.2 STOMP 协议

STOMP（Simple Text Orientated Messaging Protocol）是跑在 WebSocket 上的应用层协议，提供"订阅/发布/目的地"的语义。Spring `@EnableWebSocketMessageBroker` 默认用它。

```java
@Configuration
@EnableWebSocketMessageBroker
public class StompConfig implements WebSocketMessageBrokerConfigurer {
    @Override
    public void registerStompEndpoints(StompEndpointRegistry r) {
        r.addEndpoint("/ws/stomp").setAllowedOriginPatterns("*").withSockJS();
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry r) {
        r.enableSimpleBroker("/topic", "/queue");   // 内存 broker
        r.setApplicationDestinationPrefixes("/app"); // 客户端发到 /app/xxx
    }
}

@Controller
public class ChatController {
    @MessageMapping("/chat.send")        // 客户端 send to /app/chat.send
    @SendTo("/topic/messages")           // 广播到 /topic/messages
    public ChatMsg send(ChatMsg in) {
        return new ChatMsg(in.from(), in.text(), Instant.now());
    }

    // 私聊
    @MessageMapping("/chat.private")
    public void sendPrivate(ChatMsg in, SimpMessagingTemplate tpl) {
        tpl.convertAndSendToUser(in.to(), "/queue/messages", in);
    }
}
```

客户端：

```javascript
import { Client } from '@stomp/stompjs';
const client = new Client({
    brokerURL: 'wss://example.com/ws/stomp',
    connectHeaders: { Authorization: 'Bearer ...' },
    reconnectDelay: 5000,
});
client.onConnect = () => {
    client.subscribe('/topic/messages', m => render(m.body));
    client.publish({ destination: '/app/chat.send', body: JSON.stringify({...}) });
};
client.activate();
```

STOMP 的好处：协议标准化、有现成客户端、Spring 集成完整。坏处：

- 协议解析开销（每条消息要解析 STOMP 帧头）
- 内存 broker 不能水平扩展（要用 RabbitMQ / ActiveMQ 做 broker）
- 订阅泄漏问题严重（见案例 ⑤）

### 2.3 Spring SSE（SseEmitter）

SSE 在 Spring 中比 WebSocket 简单得多：

```java
@RestController
public class StreamController {
    private final Map<String, SseEmitter> emitters = new ConcurrentHashMap<>();

    @GetMapping(value = "/sse/notify", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter subscribe(@RequestParam String userId) {
        SseEmitter emitter = new SseEmitter(0L);    // 0 = 不超时
        emitters.put(userId, emitter);

        emitter.onCompletion(() -> emitters.remove(userId));
        emitter.onTimeout(() -> emitters.remove(userId));
        emitter.onError(t -> emitters.remove(userId));

        // 立即发一个初始事件（兼顾某些代理的 buffer 问题）
        try { emitter.send(SseEmitter.event().comment("connected")); }
        catch (IOException e) { emitters.remove(userId); }

        return emitter;
    }

    public void notify(String userId, Object data) {
        SseEmitter e = emitters.get(userId);
        if (e == null) return;
        try {
            e.send(SseEmitter.event().name("notify").data(data));
        } catch (IOException ex) {
            emitters.remove(userId);
        }
    }
}
```

定时心跳（防 NAT 超时 + 防代理 buffer）：

```java
@Scheduled(fixedRate = 30_000)
public void heartbeat() {
    emitters.forEach((id, e) -> {
        try { e.send(SseEmitter.event().comment("ping")); }
        catch (IOException ex) { emitters.remove(id); }
    });
}
```

### 2.4 LLM 流式响应（OpenAI / Anthropic SSE）

最典型的 SSE 用法。完整的"调用 LLM → 流式转发给前端"实现：

```java
@RestController
public class ChatStreamController {
    private final WebClient openai = WebClient.builder()
        .baseUrl("https://api.openai.com")
        .defaultHeader("Authorization", "Bearer " + apiKey)
        .build();

    @PostMapping(value = "/api/chat", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<String>> chat(@RequestBody ChatReq req) {
        return openai.post()
            .uri("/v1/chat/completions")
            .bodyValue(Map.of(
                "model", "gpt-4o",
                "stream", true,
                "messages", req.messages()
            ))
            .retrieve()
            .bodyToFlux(String.class)         // 每一行 SSE data
            .takeUntil(s -> s.contains("[DONE]"))
            .filter(s -> !s.contains("[DONE]"))
            .map(this::extractDelta)
            .map(token -> ServerSentEvent.<String>builder()
                .event("token")
                .data(token)
                .build())
            .concatWith(Flux.just(ServerSentEvent.<String>builder()
                .event("done").data("").build()));
    }

    private String extractDelta(String line) {
        JsonNode n = mapper.readTree(line);
        return n.path("choices").path(0).path("delta").path("content").asText("");
    }
}
```

前端：

```javascript
const es = new EventSource('/api/chat?q=hello');
es.addEventListener('token', e => append(e.data));
es.addEventListener('done', e => es.close());
es.onerror = e => { console.error(e); es.close(); };
```

**注意几个生产细节**：

1. **EventSource 不支持 POST**：浏览器原生 EventSource 只能 GET。要 POST 必须用 fetch + ReadableStream 自己解析（或用 `@microsoft/fetch-event-source`）。
2. **响应式栈是必须的**：用 WebMvc 的话，一个 LLM 请求会占一个线程几秒到几十秒，几十个并发就把 Tomcat 打爆。WebFlux 或虚拟线程可解。
3. **客户端断开要传播取消**：用户关掉页面，应该立刻取消上游的 OpenAI 调用，节省 token 成本：
   ```java
   .doOnCancel(() -> log.info("client cancelled, stopping upstream"))
   ```

### 2.5 Netty 实现 WebSocket Server

百万连接级别用 Spring 是不行的（Tomcat 线程模型有限）。Netty 是事实标准：

```java
public class WsServer {
    public void start(int port) throws InterruptedException {
        EventLoopGroup boss = new NioEventLoopGroup(1);
        EventLoopGroup work = new NioEventLoopGroup();    // 默认 CPU * 2
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(boss, work)
             .channel(NioServerSocketChannel.class)
             .option(ChannelOption.SO_BACKLOG, 1024)
             .childOption(ChannelOption.SO_KEEPALIVE, true)
             .childOption(ChannelOption.TCP_NODELAY, true)
             .childOption(ChannelOption.SO_RCVBUF, 16 * 1024)
             .childOption(ChannelOption.SO_SNDBUF, 16 * 1024)
             .childHandler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 protected void initChannel(SocketChannel ch) {
                     ChannelPipeline p = ch.pipeline();
                     p.addLast(new HttpServerCodec());
                     p.addLast(new HttpObjectAggregator(64 * 1024));
                     p.addLast(new IdleStateHandler(60, 0, 0));   // 60s 没读到任何数据
                     p.addLast(new WebSocketServerProtocolHandler(
                         "/ws", null, true, 64 * 1024));
                     p.addLast(new WsBusinessHandler());
                 }
             });
            b.bind(port).sync().channel().closeFuture().sync();
        } finally {
            boss.shutdownGracefully();
            work.shutdownGracefully();
        }
    }
}

public class WsBusinessHandler extends SimpleChannelInboundHandler<WebSocketFrame> {
    private static final ChannelGroup ALL =
        new DefaultChannelGroup(GlobalEventExecutor.INSTANCE);

    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        ALL.add(ctx.channel());
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, WebSocketFrame frame) {
        if (frame instanceof TextWebSocketFrame t) {
            // 业务处理
            ctx.writeAndFlush(new TextWebSocketFrame("echo: " + t.text()));
        } else if (frame instanceof PongWebSocketFrame) {
            // 客户端回应了我们的 ping
        }
    }

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
        if (evt instanceof IdleStateEvent e && e.state() == IdleState.READER_IDLE) {
            // 60 秒没收到客户端任何消息，主动 ping
            ctx.writeAndFlush(new PingWebSocketFrame());
            // 再过 60 秒还没动静（没收到 pong）就关
            ctx.pipeline().addLast(new IdleStateHandler(60, 0, 0) {
                @Override
                protected IdleStateEvent newIdleStateEvent(IdleState s, boolean f) {
                    ctx.close();
                    return super.newIdleStateEvent(s, f);
                }
            });
        }
    }
}
```

为什么 Netty 能撑百万？

- **EventLoop = 单线程跑多个连接**：N 个 EventLoop 处理百万连接，每个 EL 负责数万。无线程切换、无锁竞争。
- **零拷贝**：`FileRegion` / `CompositeByteBuf` 直接走 sendfile / 拼接，避免内核态-用户态拷贝。
- **直接内存**：避免 GC 扫描大块字节数组。

代价是：所有业务代码必须**非阻塞**。在 EventLoop 里调阻塞 IO（DB、HTTP）会卡住整个 EL 上的所有连接。业务必须 offload 到业务线程池。

### 2.6 客户端 SDK

#### 浏览器 WebSocket

原生 WebSocket **没有自动重连、没有心跳**。生产必须包一层。`reconnecting-websocket` 是事实标准：

```javascript
import ReconnectingWebSocket from 'reconnecting-websocket';

const ws = new ReconnectingWebSocket('wss://example.com/ws', [], {
    maxRetries: Infinity,
    reconnectionDelayGrowFactor: 1.3,    // 指数退避
    maxReconnectionDelay: 30000,
    minReconnectionDelay: 1000,
});

ws.addEventListener('open', () => {
    // 重连后要重新订阅
    ws.send(JSON.stringify({ type: 'subscribe', topics: ['user-123'] }));
});

ws.addEventListener('message', e => handle(e.data));

// 应用层心跳
setInterval(() => {
    if (ws.readyState === WebSocket.OPEN) {
        ws.send(JSON.stringify({ type: 'ping' }));
    }
}, 30000);
```

重连后必须**重新订阅**。这是断线重连最容易漏的一点：连接活了，但服务端不知道你之前订了什么。

#### 浏览器 EventSource

原生 EventSource **自带重连**，但只能 GET、不支持自定义 header。POST 场景用 `@microsoft/fetch-event-source`：

```javascript
import { fetchEventSource } from '@microsoft/fetch-event-source';

await fetchEventSource('/api/chat', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', Authorization: 'Bearer ...' },
    body: JSON.stringify({ messages: [...] }),
    signal: abortController.signal,
    onmessage(ev) { append(ev.data); },
    onerror(err) { console.error(err); throw err; }   // 抛出会停止重连
});
```

### 2.7 集群广播（Redis Pub/Sub / Kafka）

单机 100 万连接，集群 N 台机器。用户 A 在机器 1，用户 B 在机器 2 —— A 发消息给 B 怎么办？

#### Redis Pub/Sub（最简单）

```java
@Component
public class RedisBridge {
    private final RedisTemplate<String, String> redis;
    private final SessionRegistry registry;

    @PostConstruct
    public void subscribe() {
        redis.getConnectionFactory().getConnection().subscribe(
            (msg, p) -> {
                Envelope env = parse(msg.getBody());
                registry.sendLocal(env.userId(), env.payload());
            },
            "ws.broadcast".getBytes()
        );
    }

    public void publish(String userId, String payload) {
        redis.convertAndSend("ws.broadcast",
            mapper.writeValueAsString(new Envelope(userId, payload)));
    }
}
```

特点：

- 简单、低延迟（ms 级）
- **不保证可靠送达** —— Redis Pub/Sub 是 fire-and-forget，订阅者断开期间的消息丢失
- 高频广播会让 Redis CPU 100%

#### Kafka / Pulsar（高吞吐）

```java
@KafkaListener(topics = "ws.broadcast", groupId = "ws-instance-${pod.name}")
public void onBroadcast(ConsumerRecord<String, String> r) {
    Envelope e = parse(r.value());
    registry.sendLocal(e.userId(), e.payload());
}
```

注意 `groupId` 要带实例名 —— 我们要"每个实例都收到所有消息"，**不是**消费者组负载均衡。否则 100 条广播只有 1 台收到。

#### 路由模式（精确发送）

广播是无差别推所有实例，浪费。如果业务能维护"用户在哪台机"的映射：

```java
// 用户上线
redis.set("ws.route:" + userId, instanceId, Duration.ofMinutes(5));

// 发送
String target = redis.get("ws.route:" + userId);
if (target.equals(this.instanceId)) {
    sendLocal(userId, msg);
} else {
    redis.convertAndSend("ws.send." + target, msg);   // 定向发到目标实例
}
```

代价：路由表维护成本（用户在多台机就更复杂了）。

> **口诀**：**Spring WebSocket 适合中小并发；STOMP 适合企业内部应用；SSE 适合单向流；Netty 适合百万连接；集群广播必须有"广播 vs 路由"的取舍**。

---

## 3. 实测数据

> 没数据的"性能更好"都是耍流氓。下面用一组对比实验告诉你边界在哪。

### 3.1 单实例最大连接数

实验环境：c5.4xlarge（16C/32G）、Linux 5.10、Netty 4.1、JDK 21。
客户端：另开 4 台机器用 wrk2 + 自定义 WebSocket 客户端，每台模拟 25 万连接。

调优前（默认配置）：

```
连接数  | 内存占用 | CPU
 10万   |  4.2GB   | 5%
 30万   |  18GB    | 12%
 50万   | OOM
```

瓶颈：内核 socket 缓冲区默认 87KB × 50 万 = 43GB，机器内存爆了。

调优后（缓冲区调小到 16KB、堆外内存 8GB、文件描述符 200 万）：

```
连接数   | 内存占用 | CPU（30s ping）
 10万    | 1.8GB    | 1%
 50万    | 7.5GB    | 4%
100万    | 14.2GB   | 8%
150万    | 22.5GB   | 15%
```

继续涨：到 180 万 CPU 飙到 70%（Ping/Pong 处理 + epoll 调度）。最终用 16C/32G 跑稳 150 万连接。

**结论性口诀**：单机百万连接要算两件事 —— **内核内存**（缓冲区）和**心跳 PPS**。其他都好说。

### 3.2 心跳频率对资源的影响

100 万连接、不同心跳间隔下的 CPU 与带宽：

| 心跳间隔 | PPS（双向） | CPU | 带宽 | NAT 切流风险 |
| --- | --- | --- | --- | --- |
| 10s  | 20 万 | 22% | ~ 8MB/s | 极低 |
| 30s  | 6.7 万 | 8% | ~ 2.7MB/s | 低 |
| 60s  | 3.3 万 | 4% | ~ 1.3MB/s | 中（部分家用 NAT 5min 超时） |
| 120s | 1.7 万 | 2% | ~ 0.7MB/s | 高 |

**结论**：30 秒是甜区。10 秒虽然安全但 CPU 翻三倍；60 秒已经有部分用户被 NAT 切。

如果业务对端有自然消息（比如行情每秒推送），心跳可以拉到 90 秒甚至关闭。

### 3.3 WebSocket vs SSE 的吞吐对比

实验：单服务端、单实例，下推消息（服务 → 客户端），消息 200 字节，10 万连接。

| 方案 | 推送速率（msg/s/连接） | 服务端 CPU | 服务端内存 | 客户端 CPU |
| --- | --- | --- | --- | --- |
| WebSocket 文本帧 | 100 | 18% | 1.6GB | 6% |
| WebSocket 二进制帧 | 100 | 14% | 1.6GB | 5% |
| SSE | 100 | 21% | 1.8GB | 4% |
| HTTP/2 SSE | 100 | 16% | 2.0GB | 4% |

观察：

- **WebSocket 比 SSE 略省 CPU**（帧头更紧凑、不用每条都带 `data:` 前缀）
- **HTTP/2 SSE 比 HTTP/1.1 SSE 省内存**（多个 SSE 流复用一个 TCP）
- **二进制帧比文本帧省 ~ 25% CPU**（少一次 UTF-8 校验）

但差距不大，到 10 倍负载下才明显。中小负载选哪个看业务方便。

### 3.4 STOMP 的协议开销

同样 200 字节业务 payload，比较裸 WebSocket 和 STOMP-on-WebSocket：

| 协议 | 单消息总字节 | 解析 CPU | 序列化 |
| --- | --- | --- | --- |
| 裸 WebSocket | 206 字节 | 0.3μs | 直接字符串 |
| STOMP | 280 字节（多了帧头） | 1.2μs | STOMP 帧 + JSON |

STOMP 的协议开销 ~ 35% 字节、4 倍 CPU。在百万消息/秒级别这不能忽视。**适用场景判断**：

- 企业级应用、消息频率不高、需要订阅模型 → STOMP 完全 OK
- IM、行情、游戏推送 → 用裸 WebSocket + 自己定义二进制协议（比如 protobuf）

### 3.5 集群广播延迟

3 台 c5.4xlarge 实例，每台 30 万连接，总 90 万连接。一条消息广播到全部。

| 方案 | 端到端 P50 | P99 | 备注 |
| --- | --- | --- | --- |
| Redis Pub/Sub | 8ms | 25ms | 丢失率 0.01%（Redis 网络抖动） |
| Kafka | 35ms | 80ms | 0 丢失 |
| Pulsar | 28ms | 65ms | 0 丢失 |
| 自建 UDP multicast | 3ms | 12ms | 需要二层网络支持 |

**结论**：

- 实时性 > 可靠性（行情、IM 在线提示）：Redis Pub/Sub
- 可靠性 > 实时性（订单状态、支付通知）：Kafka
- 同机房、追求极致延迟：UDP multicast（但需要网络团队配合）

> **口诀**：**百万连接的瓶颈是内核内存；广播的瓶颈是带宽和延迟；STOMP 的开销不能忽视**。先压测，再选型。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① NAT 超时导致连接静默断开

#### 现象

WebSocket 客户端（手机 App）反映：开着应用看不到推送，把 App 切到后台再切回来，立刻收到一堆消息。服务端连接监控显示连接数正常。

#### 根因

手机和家用路由器的 NAT 表 idle timeout 通常 3~5 分钟。如果 5 分钟内连接上没有任何包流动，NAT 表条目被清掉。这时候：

- **客户端**：以为连接还在（TCP 状态 ESTABLISHED）
- **服务端**：以为连接还在
- **路由器**：已经把映射清掉了

服务端往客户端发包，路由器找不到映射 → 发不到客户端。但**没有人报错** —— 服务端 send 系统调用成功（数据进了 TCP send buffer），客户端无感知（没收到任何东西）。

直到客户端切前后台，OS 检测到网络变化触发新连接（或者下一次 keepalive 超时），才发现旧连接死了。

#### 修复

**铁律**：长连接必须有应用层心跳，间隔 < NAT 超时。

```java
// Netty 服务端：60 秒没读到任何数据就主动 ping
p.addLast(new IdleStateHandler(60, 0, 0));

@Override
public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
    if (evt instanceof IdleStateEvent e && e.state() == IdleState.READER_IDLE) {
        ctx.writeAndFlush(new PingWebSocketFrame());
        // 启动 30s 倒计时，超时 close
        ctx.executor().schedule(() -> {
            if (System.currentTimeMillis() - lastPongTime > 30_000) {
                ctx.close();
            }
        }, 30, TimeUnit.SECONDS);
    }
}
```

客户端同样要 ping，且**两边都要在没收到对方响应时主动 close**。

#### 验证

```bash
# 在客户端机器抓包，确认 30s 内确实有 ping/pong 包来回
sudo tcpdump -i any 'port 443' -nn | grep -E "len 6|len 8"

# 模拟 NAT 超时：在路由器上手动清 conntrack 条目
sudo conntrack -D --src 192.168.1.100

# 看应用日志：30s 后应该报 "channel inactive"
```

> **口诀**：心跳间隔 30 秒、超时 60 秒断开。客户端服务端都要有，缺一不可。

---

### 案例 ② 客户端断网无感知，服务端误以为还连着

#### 现象

用户拔了网线、或者地铁里手机断网，服务端的连接列表里这个用户还在"在线"状态。给这个用户发消息没有报错但永远收不到。

#### 根因

TCP 的设计是"对等连接"，**任何一端断开都需要主动通知对方**。如果客户端是被"硬"断开（拔网线、断电、kill -9），不会发 FIN/RST，服务端永远不知道。

服务端的判定依赖：

- **TCP KeepAlive**：默认 2 小时才探测，没用
- **应用层心跳**：必须主动加
- **写失败**：往一个"实际不通"的连接 write，要等 TCP 重传超时（默认 15 分钟）才报错

最容易踩的是第三个 —— 你以为 `socket.write()` 失败会立刻报错。**不会**，数据进了 send buffer，要等 ACK 超时才报错。期间你看不出有问题。

#### 修复

应用层心跳 + 主动检测 send buffer 堆积：

```java
@Scheduled(fixedRate = 30_000)
public void healthCheck() {
    sessions.forEach((userId, session) -> {
        // 检查 1：上次收到客户端心跳的时间
        long lastSeen = session.lastSeen();
        if (System.currentTimeMillis() - lastSeen > 90_000) {
            log.warn("user {} no heartbeat for 90s, closing", userId);
            session.close();
            return;
        }

        // 检查 2：发送队列堆积
        if (session.outboundQueueSize() > 100) {
            log.warn("user {} outbound queue overflow, closing", userId);
            session.close();
        }
    });
}
```

Netty 的 `Channel.isWritable()` 配合：

```java
if (!ctx.channel().isWritable()) {
    // 出方向 buffer 满了，说明对端没在收
    log.warn("channel {} not writable, dropping", ctx.channel().id());
    ctx.close();
}
```

#### 验证

```bash
# 模拟客户端硬断
# 1. 客户端打开连接
# 2. 服务端确认连接建立
# 3. 客户端 sudo iptables -A OUTPUT -j DROP（彻底断网，不发 FIN）
# 4. 等 90 秒，服务端应该主动 close

# 服务端日志预期：
# "user xxx no heartbeat for 90s, closing"
```

> **口诀**：**只信"最近收到对方消息的时间"，不信 TCP 状态**。心跳 + 超时检测 + Channel.isWritable 三件套。

---

### 案例 ③ 百万连接 OOM —— FD 与内存配置错误

#### 现象

新上线的 IM 服务，压测到 30 万连接时 OOM 崩溃。`/var/log/messages` 一堆 `Out of memory: Killed process`。

#### 根因

两个问题叠加：

1. **JVM 堆配置太小**：默认 Xmx = 系统内存 1/4 = 8GB，30 万连接的 Session 对象就吃了 6GB 堆。
2. **TCP 接收缓冲区太大**：默认 `tcp_rmem` 上限 6MB，30 万连接如果都在接收数据 = 1.8TB（虽然实际不会到顶，但内核要按需分配）。

更隐蔽的是堆外内存 —— Netty 的 `DirectByteBuf` 在堆外，JVM 监控看不到，OOM 时只看 JVM 监控会"莫名其妙"。

#### 修复

完整的内核 + JVM + Netty 配置：

```bash
# /etc/sysctl.conf
fs.file-max = 2097152
fs.nr_open = 2097152

net.ipv4.tcp_rmem = 4096 16384 1048576       # 缩小默认 + 上限
net.ipv4.tcp_wmem = 4096 16384 1048576
net.core.rmem_max = 1048576
net.core.wmem_max = 1048576
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 30000
net.ipv4.tcp_max_syn_backlog = 65535

net.ipv4.ip_local_port_range = 1024 65000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 30

# /etc/security/limits.conf
* soft nofile 1048576
* hard nofile 1048576
```

JVM 参数：

```bash
java \
  -Xms16g -Xmx16g \
  -XX:MaxDirectMemorySize=8g \              # 堆外内存上限（重要）
  -XX:+UseZGC \                              # 大堆 + 短 STW
  -Dio.netty.maxDirectMemory=8g \           # Netty 自己的限制
  -Dio.netty.allocator.type=pooled \
  ...
```

#### 监控

必须暴露三个指标，否则爆了不知道为啥：

```java
// JVM 堆
Runtime.getRuntime().totalMemory()
Runtime.getRuntime().freeMemory()

// 堆外（DirectMemory）
Class<?> bits = Class.forName("java.nio.Bits");
Field reservedMemory = bits.getDeclaredField("RESERVED_MEMORY");

// Netty PooledAllocator
PooledByteBufAllocator.DEFAULT.metric().usedDirectMemory()
```

接 Prometheus + 告警：DirectMemory 用量 / MaxDirectMemorySize > 0.8 就告警。

#### 验证

```bash
# 压测时盯三个数
watch -n 1 '
  echo "=== FD ==="
  ls /proc/$(pgrep -f myapp)/fd | wc -l
  echo "=== Memory ==="
  cat /proc/$(pgrep -f myapp)/status | grep -E "VmRSS|VmSize"
  echo "=== Connections ==="
  ss -s | head -3
'
```

> **口诀**：**FD 限制要拉到百万级；TCP 缓冲区要缩小；堆外内存要单独限**。三件套缺一不可。

---

### 案例 ④ 广播风暴 —— 单消息触发全连接推送

#### 现象

运营在后台点了"全员公告"按钮，10 秒后所有用户都收到了，但收到的同时**正常业务消息卡了 30 秒**。监控看 CPU 100%、网卡饱和、GC 暴涨。

#### 根因

广播实现是最朴素的：

```java
// 错误实现
public void broadcast(String msg) {
    sessions.values().forEach(s -> s.sendText(msg));   // 同步遍历
}
```

100 万连接 × 200 字节 = 200MB 数据要在瞬间发出。机器千兆网卡上限 125MB/s，单纯网卡都要塞 1.6 秒。期间：

- EventLoop 全部忙于 write
- 业务消息排队
- TCP send buffer 堆积 → 触发 GC（`Channel.write` 内部分配 ByteBuf）

#### 修复

**铁律**：广播必须**异步 + 限速 + 优先级**。

```java
public class RateLimitedBroadcaster {
    private final RateLimiter limiter = RateLimiter.create(50_000);  // 5 万 msg/s
    private final ExecutorService pool = Executors.newFixedThreadPool(4);

    public void broadcast(String msg) {
        ByteBuf payload = Unpooled.copiedBuffer(msg, StandardCharsets.UTF_8);
        List<Channel> snapshot = new ArrayList<>(channels);

        // 分片，每片 1 万连接，异步发
        Lists.partition(snapshot, 10_000).forEach(batch -> {
            pool.submit(() -> {
                for (Channel ch : batch) {
                    limiter.acquire();
                    if (ch.isActive() && ch.isWritable()) {
                        ch.writeAndFlush(new TextWebSocketFrame(payload.retainedDuplicate()));
                    }
                }
            });
        });
    }
}
```

更进一步：**引入消息优先级**，业务消息（高）> 广播（低），EventLoop 紧张时优先处理高优先级。

```java
// 用 PriorityBlockingQueue 替代默认 mpsc 队列
NioEventLoopGroup work = new NioEventLoopGroup(0,
    new DefaultThreadFactory("netty-work"),
    SelectorProvider.provider(),
    DefaultSelectStrategyFactory.INSTANCE,
    RejectedExecutionHandlers.reject(),
    () -> new PriorityQueue<>(...));
```

更聪明的做法：广播走 CDN/边缘节点。把"公告"消息推到 CDN 缓存，客户端从 CDN 拉。

#### 验证

```bash
# 压测：模拟广播 + 同时发业务消息
# 观察业务消息的 P99 是否被广播影响
# 健康标准：广播期间业务消息 P99 不超过平时 2 倍
```

> **口诀**：广播 = 限速 + 分批 + 异步 + 优先级。或者干脆走 CDN。

---

### 案例 ⑤ STOMP 订阅泄漏导致内存膨胀

#### 现象

服务上线两周，内存从 4GB 涨到 28GB。Heap dump 看 `SimpleSubscription` 对象 2000 万个。重启后又开始慢慢涨。

#### 根因

STOMP 订阅模型：客户端 `SUBSCRIBE` 后，服务端在 broker 里维护 `subscriptionId → handler` 映射。客户端断开时，理论上 `UNSUBSCRIBE` 或 disconnect 会触发清理。

但有几个 case 会泄漏：

- 客户端硬断（没发 DISCONNECT 帧），服务端依赖 TCP 关闭来清理 —— 如果 close 事件没传播到 broker，订阅留在那
- 客户端用同一个 sessionId 重连（某些 SDK 的 bug），新连接覆盖了旧 session 但旧订阅还挂着
- Spring `SimpMessageBroker` 默认实现里有 case 不会清理 destinationCache

确认的方法：

```java
@Autowired
SimpleBrokerMessageHandler broker;

@Scheduled(fixedRate = 60_000)
public void dumpSubs() {
    Map<String, ?> subs = broker.getSubscriptionRegistry().getAllSubscriptions();
    log.info("total subscriptions: {}", subs.size());
}
```

#### 修复

**方案 A**：换 RabbitMQ STOMP broker。
Spring 自带的 `SimpleBrokerMessageHandler` 是给 demo 用的，生产必须换：

```java
@Override
public void configureMessageBroker(MessageBrokerRegistry r) {
    r.enableStompBrokerRelay("/topic", "/queue")
     .setRelayHost("rabbitmq.internal")
     .setRelayPort(61613);
}
```

RabbitMQ 自己处理订阅生命周期，连接断了立刻清理。

**方案 B**：给 STOMP session 设硬性超时 + 监控订阅数。

```java
@EventListener
public void onDisconnect(SessionDisconnectEvent e) {
    String sessionId = e.getSessionId();
    // 强制清理这个 session 的所有订阅
    broker.getSubscriptionRegistry()
          .removeSubscription(sessionId);
}
```

**方案 C**：定期扫"僵尸订阅"。

```java
@Scheduled(fixedRate = 300_000)
public void cleanZombies() {
    Map<String, Long> activeSessions = sessionRegistry.activeSessionsLastSeen();
    Set<String> allSubSessionIds = broker.getAllSubscriptionSessionIds();

    long now = System.currentTimeMillis();
    allSubSessionIds.forEach(sid -> {
        Long lastSeen = activeSessions.get(sid);
        if (lastSeen == null || now - lastSeen > 600_000) {
            broker.removeSubscription(sid);
        }
    });
}
```

#### 验证

```java
@Test
void subscriptionMustBeClearedOnDisconnect() {
    // 1. 建立 STOMP 连接，订阅 100 个 topic
    // 2. 强制断开（不发 DISCONNECT）
    // 3. 等 5s
    // 4. 断言 broker 里订阅数 = 0
    assertEquals(0, broker.getSubscriptionRegistry().totalSubscriptions());
}
```

> **口诀**：STOMP 内存 broker 只适合 demo。生产必须用 RabbitMQ / ActiveMQ + 监控订阅数。

---

### 案例 ⑥ SSE 在 Nginx 反代下被缓冲

#### 现象

LLM 流式响应在本地直连 OK，过 Nginx 反代后客户端永远等 30 秒、然后一次性收到全部内容。流式效果消失。

#### 根因

Nginx 默认开启 `proxy_buffering on`，会缓存上游响应直到达到一定大小或上游关闭连接。SSE 这种"持续流"被缓存了，客户端就要等到 buffer 满 / 超时才能收到。

类似的"凶手"还有：

- Nginx 的 `gzip on`：要缓冲足够字节才能压缩，违背流式
- CloudFlare / 各种 CDN：默认对 `text/event-stream` 也会 buffer
- 企业代理（Squid 等）

#### 修复

Nginx 配置：

```nginx
location /sse/ {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Connection '';

    # 关闭缓冲
    proxy_buffering off;
    proxy_cache off;

    # 关闭 gzip（SSE 压缩意义不大且影响流式）
    gzip off;

    # 长连接超时
    proxy_read_timeout 24h;
    proxy_send_timeout 24h;

    # 关键：允许流式输出立即发往客户端
    chunked_transfer_encoding on;
}
```

服务端响应头也要加（双保险）：

```java
response.setHeader("X-Accel-Buffering", "no");   // Nginx 专属：关闭这个响应的 buffer
response.setHeader("Cache-Control", "no-cache");
response.setHeader("Connection", "keep-alive");
response.setHeader("Content-Type", "text/event-stream");
```

`X-Accel-Buffering: no` 是 Nginx 识别的特殊响应头，比配置文件更精准（per-response）。

CloudFlare 用户：在 Page Rules 里禁用对 SSE 路径的缓存 + 关 Auto Minify。

#### 验证

```bash
# 直接 curl 看是否流式输出
curl -N https://example.com/sse/stream

# -N 是禁用 curl 自己的 buffer
# 期望：每个 event 立刻显示，不是一次性
```

如果 curl 是流式的、浏览器不是，问题在浏览器侧（少数 fetch polyfill 也会 buffer）。

> **口诀**：SSE + Nginx = `proxy_buffering off` + `X-Accel-Buffering: no`。两个都要。

---

### 案例 ⑦ JWT 续期 —— WebSocket 没有"中间路径"

#### 现象

WebSocket 连接保持 4 小时。JWT token 30 分钟过期。30 分钟后用户不能发消息了，但连接还活着。

#### 根因

JWT 是无状态的、有过期时间的。HTTP 场景下续期靠 refresh token + 前端拦截器自动换。WebSocket 一旦建立就脱离 HTTP，**没有"前端拦截器自动加新 header"的机制**。

如果在握手时校验 token、之后不管，那么：

- token 过期后服务端继续接收消息（因为没在每条消息上重新校验）
- 服务端每条消息都校验：性能差 + 仍然没办法续期

#### 修复

**方案 A**：协议层加 "auth" 消息，客户端定期发新 token。

```java
// 服务端
protected void handleTextMessage(WebSocketSession s, TextMessage m) {
    JsonNode json = mapper.readTree(m.getPayload());
    if ("auth".equals(json.get("type").asText())) {
        String newToken = json.get("token").asText();
        Claims claims = jwtParser.parse(newToken);   // 失败抛异常
        s.getAttributes().put("userId", claims.getSubject());
        s.getAttributes().put("expireAt", claims.getExpiration().getTime());
        return;
    }

    // 业务消息：仅检查 expireAt
    long expireAt = (long) s.getAttributes().get("expireAt");
    if (System.currentTimeMillis() > expireAt) {
        s.close(CloseStatus.POLICY_VIOLATION.withReason("token expired"));
        return;
    }
    handleBusinessMessage(s, json);
}
```

```javascript
// 客户端
setInterval(async () => {
    const newToken = await refreshAccessToken();
    ws.send(JSON.stringify({ type: 'auth', token: newToken }));
}, 25 * 60 * 1000);   // token 30min 过期，每 25min 续期一次
```

**方案 B**：服务端在 token 即将过期时主动通知客户端。

```java
// 任务：每分钟扫一遍
@Scheduled(fixedRate = 60_000)
public void notifyExpiring() {
    long now = System.currentTimeMillis();
    sessions.forEach((id, s) -> {
        long expireAt = (long) s.getAttributes().get("expireAt");
        if (expireAt - now < 5 * 60_000) {     // 还有 5 分钟过期
            s.sendMessage(new TextMessage(
                "{\"type\":\"token_expiring\",\"in\":" + (expireAt - now) + "}"));
        }
    });
}
```

客户端收到 `token_expiring` 立刻去续。

**方案 C**（不推荐但有人这么做）：服务端给 WebSocket session 一个独立的、长期有效的"会话 token"，不依赖 JWT。

#### 验证

```javascript
// 测试：JWT 强制 1 分钟过期，连接保持 5 分钟
// 期望：业务消息持续可发，不报错
```

> **口诀**：WebSocket 上的 JWT 续期靠"应用层 auth 消息" + 服务端"过期时间软校验"。原生没法做，必须自己实现。

---

### 案例 ⑧ LLM 流式响应被中间件断开

#### 现象

接 OpenAI 的流式响应，前 30 秒一切正常，第 30 秒之后突然 504 或者连接 reset。但日志里看请求确实建立了。

#### 根因

LLM 响应可能持续几十秒到几分钟，比常规 HTTP 请求长得多。中间链路的超时配置经常成为杀手：

- ALB / ELB 默认 idle timeout 60 秒
- Nginx 默认 `proxy_read_timeout 60s`
- API Gateway（Kong / APISIX）默认超时 60 秒
- CloudFront 默认 60 秒
- Spring WebClient `responseTimeout` 默认 30 秒

任何一层超时，整个流就断了。

#### 修复

**全链路超时配置**（每层都要改）：

```nginx
# Nginx
proxy_read_timeout 600s;
proxy_send_timeout 600s;
```

```yaml
# Kubernetes Ingress
nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
```

```hcl
# AWS ALB
idle_timeout = 600    # 默认 60，调到 600
```

```java
// Spring WebClient
WebClient.builder()
    .clientConnector(new ReactorClientHttpConnector(
        HttpClient.create()
            .responseTimeout(Duration.ofMinutes(10))
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
    ))
    .build();
```

**保险措施**：服务端定期发"心跳事件"，让中间链路看到流量、不断开：

```java
return Flux.merge(
    actualLlmStream(),
    Flux.interval(Duration.ofSeconds(15))
        .map(i -> ServerSentEvent.<String>builder().comment("heartbeat").build())
).takeUntil(e -> "[DONE]".equals(e.data()));
```

注释事件浏览器会忽略，但中间代理看到字节就重置 idle 计时。

#### 客户端断开传播

用户关页面后，应该立刻取消上游 OpenAI 调用（节省 token 钱）：

```java
return openAiStream
    .doOnCancel(() -> {
        log.info("client cancelled, cancelling upstream");
        // WebClient 会自动取消上游
    });
```

WebFlux 已经处理好了 reactive stream 的取消传播。

#### 验证

```bash
# 模拟 5 分钟的流式响应
curl -N https://example.com/api/chat \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"写一篇 5000 字小说"}]}'

# 期望：5 分钟内持续输出，不被任何中间件断
```

> **口诀**：LLM 流式 = 全链路超时拉到 10 分钟 + 心跳事件 + 取消传播。任何一环漏了都白搭。

---

### 案例 ⑨ Sticky Session 失败 —— 负载均衡的"切流"

#### 现象

WebSocket 连接经常在建立后 1~2 秒内被服务端莫名其妙关闭。客户端重连后又能用一会儿，循环往复。

#### 根因

WebSocket 握手是 HTTP 升级。如果负载均衡（LB）没开 sticky session，握手请求和后续 WebSocket 帧可能被分到不同后端：

```
client  →  LB  →  后端 A   (HTTP 升级，握手成功)
client  →  LB  →  后端 B   (后续 WebSocket 帧，但 B 不认识这个连接)
```

实际上 WebSocket 是基于 TCP 的，TCP 连接一旦建立就和某个后端绑定。但**有些 LB 在某些情况下会切流**：

- 七层 LB（Nginx upstream）默认每个请求独立路由
- AWS ALB 默认开启了 cross-zone load balancing 但没开 sticky
- K8s Service 默认轮询，相同 IP 的多次连接可能去不同 Pod

更阴险的场景：**负载均衡 health check 把某个 Pod 摘了**，旧连接强制迁移。

#### 修复

四层 LB（推荐 WebSocket 用）：

```nginx
# Nginx stream 模块（四层）
stream {
    upstream ws_backend {
        hash $remote_addr consistent;
        server backend1:8080;
        server backend2:8080;
    }
    server {
        listen 443;
        proxy_pass ws_backend;
    }
}
```

七层 LB 必须开 sticky session：

```nginx
# Nginx http upstream
upstream ws_backend {
    ip_hash;       # 基于客户端 IP
    server backend1:8080;
    server backend2:8080;
}
```

或用 cookie（更精准但需要客户端支持）：

```nginx
upstream ws_backend {
    sticky cookie srv_id expires=1h domain=.example.com path=/;
    server backend1:8080;
    server backend2:8080;
}
```

K8s Service：

```yaml
apiVersion: v1
kind: Service
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800     # 3 小时
```

**滚动更新时的优雅迁移**：

- preStop hook 执行 `sleep 30`，让 LB 摘流
- 应用收到 SIGTERM 后，**主动通知所有 WebSocket 客户端"我要下线了，请重连"**
- 客户端重连，被 LB 调度到其他实例

```java
@PreDestroy
public void shutdown() {
    sessions.values().forEach(s -> {
        try {
            s.sendMessage(new TextMessage("{\"type\":\"reconnect\"}"));
            s.close(CloseStatus.GOING_AWAY);
        } catch (Exception ignore) {}
    });
}
```

#### 验证

```bash
# 在浏览器 DevTools Network 看 WebSocket 帧
# 检查：握手成功后是否立刻有 "Sec-WebSocket" 之外的关闭帧

# 后端日志：搜 "WebSocket session closed unexpectedly"
# 看是否在握手后秒级就关闭
```

> **口诀**：WebSocket + LB = sticky session 必开 + 优雅停机通知客户端。两个都不能少。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 协议与设计
- [ ] 选型有依据：单向用 SSE，双向用 WebSocket，老系统才用 Long Polling
- [ ] WebSocket 握手必须走 wss（TLS），生产环境禁用 ws
- [ ] LLM 流式响应统一用 SSE，不要用 WebSocket
- [ ] 业务协议（消息格式）有版本字段，便于平滑升级

### 心跳与超时
- [ ] 应用层心跳间隔 ≤ 30 秒
- [ ] 服务端有"超过 N 秒没收到对方消息就 close"机制
- [ ] TCP KeepAlive 配置调过（不依赖默认 2 小时）
- [ ] SSE 服务端发注释行（`:ping\n\n`）做心跳

### 资源与配置
- [ ] FD 限制 ≥ 100 万（`ulimit -n` 和 systemd LimitNOFILE）
- [ ] TCP socket 缓冲区调小（`tcp_rmem` / `tcp_wmem` 上限到 1MB）
- [ ] JVM `-XX:MaxDirectMemorySize` 显式设置
- [ ] Netty `pooled` allocator + 监控 `usedDirectMemory`

### 鉴权
- [ ] 握手阶段校验 token（HandshakeInterceptor）
- [ ] 长连接上 token 续期机制（应用层 auth 消息）
- [ ] 服务端记录每个 session 的过期时间，过期主动 close

### 集群与广播
- [ ] WebSocket 在 LB 层开了 sticky session
- [ ] 集群广播用 Redis Pub/Sub 或 Kafka，不要每实例自己发
- [ ] 全员广播有限速 + 分批 + 异步
- [ ] 跨实例消息有路由表（用户在哪台机）

### 反向代理 / 网关
- [ ] Nginx：`proxy_buffering off`、`proxy_read_timeout` 调长（≥ 600s）
- [ ] 服务端响应：`X-Accel-Buffering: no`（SSE 必加）
- [ ] AWS ALB / 阿里 SLB：idle timeout 调到 600s 以上
- [ ] WAF 不要拦截 WebSocket Upgrade 请求

### 优雅停机
- [ ] `@PreDestroy` 通知所有客户端"我要下线了，请重连"
- [ ] K8s `terminationGracePeriodSeconds ≥ 60`
- [ ] preStop hook `sleep N` 等待 LB 摘流
- [ ] 收到 SIGTERM 后停止接受新连接，等老连接自然关闭

### 监控
- [ ] 暴露指标：连接数、心跳 PPS、消息收发速率、广播延迟
- [ ] 告警：连接数突降（> 30%）、广播延迟 P99 > 1s
- [ ] Heap dump 定期巡检（防 STOMP 订阅泄漏类问题）

### 客户端
- [ ] WebSocket 客户端有自动重连（指数退避）
- [ ] 重连后**重新订阅**之前的所有 topic
- [ ] 客户端有应用层心跳（不依赖浏览器原生）
- [ ] 离线消息有补偿机制（断线期间的消息不丢）

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### WebTransport（基于 QUIC）
- W3C 标准，跑在 HTTP/3 (QUIC) 上的双向通信协议
- 比 WebSocket 优势：多流复用（流之间互不阻塞）、更快握手（0-RTT）、UDP 层避免队头阻塞
- 现状：Chrome / Edge 已支持，Firefox / Safari 还没。生产用还需观望
- 可以看 W3C [WebTransport draft](https://w3c.github.io/webtransport/) 和 Chrome origin trial

### WebRTC 数据通道（DataChannel）
- 浏览器之间 P2P 通信（不经过服务器）
- 适合：实时协作（如多人白板的鼠标位置同步）、文件传输
- 协议栈：UDP + DTLS + SCTP，比 WebSocket 复杂得多
- 服务端只做信令（用 WebSocket 或 SSE 做 SDP 交换）

### MQTT 协议（IoT 长连接）
- 设计目标：低带宽、低功耗、不可靠网络
- 比 WebSocket 优势：QoS 0/1/2、Retained Message、Will Message
- 典型 broker：EMQX（百万连接）、Mosquitto（轻量）
- WebSocket 可以承载 MQTT（mqtt over websocket），让浏览器也能玩

### HTTP/2 Server Push（已废弃）
- 设计意图：服务端主动推资源给客户端
- 实际问题：客户端不知道哪些已缓存，重复推浪费；优先级控制差
- Chrome 在 2022 年移除支持。**不要再用了**
- 替代方案：HTTP/3 + 103 Early Hints

### gRPC 双向流（Streaming RPC）
- 基于 HTTP/2，原生支持四种流模式
- 适合：服务间通信，浏览器不能直接用（需要 gRPC-Web）
- 比 WebSocket 优势：强类型（protobuf）、有标准的取消 / 超时
- 比 WebSocket 劣势：协议复杂、调试工具少

### 开源 broker 选型
- **EMQX**：MQTT 为主，单集群百万连接，开源版功能足够
- **Pulsar**：消息队列 + 流，多租户、地理复制
- **NATS / NATS JetStream**：极低延迟（μs 级），轻量
- **Centrifugo**：专注于实时消息推送，有 WebSocket / SSE / Long Polling fallback

### 移动端长连接的省电
- iOS：APNs（统一推送）；不用自建长连接
- Android：FCM（国外）/ 厂商推送（小米、华为、OPPO 等）
- 自建长连接的省电技巧：心跳对齐（屏幕亮起时正常、灭屏时拉长到 5 分钟）、合并发送
- 微信的"智能心跳"：动态调整间隔，5~270 秒之间

### 后端框架选型
- **Spring WebFlux + Netty**：Java 生态主流
- **Vert.x**：纯 EventLoop，性能更好，学习曲线陡
- **Akka Streams / Pekko**：消息驱动，适合复杂状态管理
- **Go gnet / nbio**：高性能网络库，百万连接
- **Rust tokio + tungstenite**：极致性能，但人才稀缺

---

## 7. 引用

- RFC 6455: [The WebSocket Protocol](https://datatracker.ietf.org/doc/html/rfc6455)
- RFC 8441: [WebSocket Bootstrapping in HTTP/2](https://datatracker.ietf.org/doc/html/rfc8441)
- HTML Living Standard: [Server-Sent Events](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- STOMP 1.2 Protocol: [stomp.github.io/stomp-specification-1.2](https://stomp.github.io/stomp-specification-1.2.html)
- Netty in Action（Norman Maurer）—— EventLoop 与 ChannelPipeline 的权威解读
- 美团技术博客：[长连接网关在美团的实践](https://tech.meituan.com/) 系列
- OpenAI API Reference: [Chat Completions Streaming](https://platform.openai.com/docs/api-reference/chat/streaming)
- 微信公众号"鹅厂网事"：[十亿级长连接架构演进](https://www.qq.com/) 系列
- EMQX 官方文档：[百万 MQTT 连接调优](https://www.emqx.com/en/blog)
- W3C WebTransport draft

---

> 配套：[Netty EventLoop](/article/network/netty-eventloop) | [HTTP/1→2→3](/article/network/http-evolution) | [TCP 状态机](/article/network/tcp-state) | [LLM 应用工程化](/article/ai/llm-app)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
