# API 设计与 OpenAPI —— REST / GraphQL / gRPC 的工程化选型与陷阱

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，看 [系统设计速记](/article/system-design/) 那一节够了。
> 这里要回答的，是**接口评审上被拷问、半夜因为分页慢查询被叫起来、上线前发现 Swagger 暴露在公网**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. `POST /users/123/orders` 和 `POST /createOrder` 都能"创建订单"，凭什么前者算 REST、后者算 RPC？真正的边界是什么？
2. GraphQL 的 N+1 问题为什么不靠"加缓存"就能解决？DataLoader 内部的 batch + dedupe 是怎么做到一帧合并的？
3. 删除一条不存在的资源，应该返回 `200 / 204 / 404`？你的答案能不能在两个 HTTP 标准里找到依据？
4. 接口要从 v1 升到 v2，URL 加版本（`/v2/users`）、Header 带版本（`API-Version: 2`）、Media Type 协商（`Accept: application/vnd.app.v2+json`）—— 这三种方案分别在什么场景被否过？
5. HTTP 标准说 `PUT` 是幂等的，但你业务上 `POST /payments` 也要"幂等"。这两个"幂等"是同一个词吗？怎么用 `Idempotency-Key` 实现后者？
6. `LIMIT 1000000, 20` 慢查询的根因是什么？换成 keyset pagination 为什么快得离谱？cursor pagination 又解决了什么 keyset 解决不了的问题？
7. OpenAPI 和 Swagger 是同一个东西吗？SpringDoc 和 SpringFox 谁还活着、谁已经死了？为什么不要再用 SpringFox？
8. 你的服务三层调用：A→B→C，每层超时都是 1s，外层超时配 2s。压测时一切正常，生产某天突然出现"接口返回成功但数据没落库" —— 怎么回事？

如果有半数答不上，往下看。

---

## 1. API 设计哲学：先把"风格"这个词的定义讲清楚

绝大多数关于 API 风格之争的讨论是无意义的，因为参与者对"REST""RPC""资源""动词"这些词的定义就不一样。这一节先把名词钉死，后面才有讨论的基础。

### 1.1 RESTful 真正的含义

Roy Fielding 2000 年的博士论文《Architectural Styles and the Design of Network-based Software Architectures》第五章提出 REST，**核心是六个约束**：

- Client-Server 分离
- Stateless（无状态）
- Cacheable（响应可缓存）
- Uniform Interface（统一接口）
- Layered System（分层）
- Code on Demand（可选）

其中"Uniform Interface"又分四个子约束，最被忽视也最关键的是 **HATEOAS**（Hypermedia as the Engine of Application State）—— 客户端通过响应里的链接来发现下一步能做什么，而不是预先 hardcode URL。

按这个标准，**绝大多数自称 RESTful 的 API 都只是"基于 HTTP 的 RPC"**。Leonard Richardson 提出的"REST 成熟度模型"把它分四级：

| 级别 | 特征 | 现实占比 |
| --- | --- | --- |
| **Level 0** | 单端点 + POST 包裹一切（SOAP / XML-RPC） | < 5% |
| **Level 1** | 多资源 URL，但都用 POST | ~15% |
| **Level 2** | 资源 URL + HTTP 动词 + 状态码（绝大多数自称 REST 的 API） | ~80% |
| **Level 3** | Level 2 + HATEOAS | < 1% |

工程上能做到 Level 2 已经够好。**没必要追求 Level 3**，HATEOAS 在前后端紧密协作的内部系统里反而是负担（前端不会真去解析链接，仍然 hardcode）。

### 1.2 RPC 风格 vs REST 风格的本质差别

抛开教科书定义，工程上的差别就一件事：**URL 是名词还是动词**。

```http
# REST 风格：URL 是资源（名词），动作通过 HTTP method 表达
GET    /orders/123
POST   /orders
PUT    /orders/123
DELETE /orders/123
PATCH  /orders/123/status

# RPC 风格：URL 是动作（动词），所有请求都是 POST
POST /createOrder
POST /getOrder
POST /updateOrderStatus
POST /cancelOrder
```

谁更好？没有标准答案。**取决于你的业务模型有没有"清晰的资源"**：

- 电商订单、用户、商品这种 **CRUD 模型** —— REST 自然
- "运行作业""触发流程""下发指令"这种 **动作模型** —— RPC 更自然
- 业务上是"对资源的非标准操作"（如"审批通过""库存扣减""退款"）—— REST 处理起来很别扭，常见妥协是 `POST /orders/123/actions/approve`

gRPC、Twirp、JSON-RPC 都是 RPC 风格的不同载体；**REST vs RPC 是设计哲学之争，不是技术栈之争**。一个 gRPC 服务里完全可以有"资源化"的方法名（`GetOrder`, `CreateOrder`），Google 的 [API Design Guide](https://google.aip.dev/) 就是这么干的。

> 不要为风格本身争论。问业务："你的领域里有清晰的资源边界吗？"有，REST；没有，RPC。

### 1.3 HTTP 状态码的"对的"使用

状态码是 HTTP 语义层。用对了客户端能直接复用 fetch / axios / okhttp 的内置行为；用错了，所有客户端都得自己解析 body。

| 范围 | 含义 | 关键码 |
| --- | --- | --- |
| **2xx** 成功 | 请求被处理 | 200 / 201 / 202 / 204 |
| **3xx** 重定向 | 客户端需要进一步动作 | 301 / 302 / 304 |
| **4xx** 客户端错误 | 请求本身有问题 | 400 / 401 / 403 / 404 / 409 / 422 / 429 |
| **5xx** 服务端错误 | 服务端处理失败 | 500 / 502 / 503 / 504 |

几个最容易用错的码：

- **201 Created**：POST 成功创建资源时用，**响应里必须带 `Location` 头**指向新资源
- **202 Accepted**：异步处理 —— 你收到了请求但还没处理完。响应应该带"查询进度"的 URL
- **204 No Content**：成功但响应体为空。常用于 DELETE 和某些 PUT。**注意：204 响应不能有 body**
- **400 Bad Request**：请求格式错（JSON 解析失败、缺字段）
- **401 Unauthorized**：身份未认证（实际语义更像"unauthenticated"，历史遗留命名错误）
- **403 Forbidden**：身份认证了但没权限
- **404 Not Found**：资源不存在
- **409 Conflict**：资源状态冲突（如重复创建、版本号不匹配）
- **422 Unprocessable Entity**：JSON 格式对，但语义上无法处理（业务校验失败）
- **429 Too Many Requests**：限流。**必须带 `Retry-After` 头**

最常被滥用的是 **400 vs 422**：很多人把所有客户端错误都返 400，导致客户端无法区分"格式错"和"业务校验错"。区分清楚的一个简单原则：**JSON 反序列化失败 → 400；反序列化成功但业务规则不允许 → 422**。

### 1.4 错误码设计：业务错误码 + HTTP 状态码

只有 HTTP 状态码不够 —— 它的粒度太粗。"用户余额不足""订单已支付""库存不够"都是 422，客户端需要更细的码做差异化处理。

业界事实标准：**HTTP 状态码 + 自定义业务错误码 + 可读消息**。RFC 7807 *Problem Details for HTTP APIs* 给出了一个标准格式：

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json

{
  "type": "https://api.example.com/errors/insufficient-balance",
  "title": "Insufficient balance",
  "status": 422,
  "detail": "Account balance 50.00 is less than required 100.00",
  "instance": "/orders/12345",
  "code": "ACCOUNT_BALANCE_INSUFFICIENT",
  "balance": 50.00,
  "required": 100.00
}
```

几个设计原则：

- **错误码用字符串而非数字**：`USER_NOT_FOUND` 比 `40401` 可读、可 grep
- **分层命名**：`{domain}.{resource}.{problem}` —— `payment.account.insufficient_balance`
- **稳定性是合同的一部分**：错误码一旦发布就不能改，否则下游的 if 判断会全炸
- **不要把堆栈塞进 detail**：那是日志该干的事，response 给人看

### 1.5 版本管理：URL / Header / Media Type 三种方案

API 演化是必然的。版本管理的三种主流方案，各有各的坑。

**方案 A：URL 版本**

```http
GET /v1/users/123
GET /v2/users/123
```

优点：直观，curl 一眼能看出版本；不同版本可以路由到不同的服务实例（部署灵活）。
缺点：违反 REST 的"URI 标识资源"原则（同一个用户在两个 URL）；客户端升级要改大量 URL。

**方案 B：Header 版本**

```http
GET /users/123
API-Version: 2
```

优点：URL 不变；可以在 Gateway 层透明路由。
缺点：浏览器调试不直观（curl 时容易忘加）；Header 不会被 URL 缓存键自动包含，需要手动配置 `Vary: API-Version`。

**方案 C：Media Type 协商（Hypermedia Versioning）**

```http
GET /users/123
Accept: application/vnd.example.user.v2+json
```

优点：最 RESTful，符合 HTTP 内容协商标准。
缺点：Media Type 字符串复杂，开发体验差；很多 HTTP 客户端默认不会保留这个 Accept 头。

工程实践上，**绝大多数公司选 URL 版本**（GitHub 在 v3 用 URL，v4 GraphQL 干脆放弃了版本），因为它对所有人（前端、运维、SDK 作者）都最直白。除非你做的是公开 API 平台、客户端非常多样，否则不要追求 Media Type。

> 选定一种方案后写进 API 设计规范，**全公司一致**。最糟的是不同团队用不同方案，调用方一头雾水。

### 1.6 幂等性的精确定义

"幂等"这个词在 API 设计中有两个层次的含义，混在一起讨论会产生很多伪问题。

**HTTP 标准的幂等**（RFC 7231）：**多次执行同一个请求与执行一次的副作用相同**。注意"副作用"，不是响应。

| Method | 幂等？ | 安全？ |
| --- | --- | --- |
| GET | ✅ | ✅ |
| HEAD | ✅ | ✅ |
| OPTIONS | ✅ | ✅ |
| PUT | ✅ | ❌ |
| DELETE | ✅ | ❌ |
| POST | ❌ | ❌ |
| PATCH | ❌ | ❌ |

为什么 PUT 算幂等？因为 PUT 的语义是"用请求体完全替换资源"——执行 100 次和 1 次的最终状态相同。
为什么 POST 不幂等？因为 POST 是"创建新资源"——每次都创建一个新的。

**业务幂等**：你的业务规则不允许重复操作（不能给同一个订单付两次款），即使 HTTP 标准说 POST 不幂等。

实现业务幂等的标准模式是 `Idempotency-Key`（Stripe 在 2015 年的设计，已经是事实标准）：

```http
POST /payments
Idempotency-Key: 7e3c8f3b-1d2c-4f5a-8c8e-d8e8f3b9d0e1
Content-Type: application/json

{ "amount": 100, "currency": "USD" }
```

服务端逻辑：

```
1. 收到请求，从 Header 取 Idempotency-Key
2. 在 Redis（或 DB）查这个 Key 是否已经处理过
3. 已处理 → 直接返回上次的响应（包括状态码、body）
4. 未处理 → 加分布式锁，处理业务，把响应缓存起来
5. 返回响应
```

**关键设计陷阱**：缓存的 TTL 怎么定？太短，重试会失效；太长，内存撑爆。Stripe 选了 24 小时——足够覆盖所有"网络/客户端重试"场景。

后面案例 ⑥ 会展开"幂等键到底用消息 ID 还是业务 ID"的讨论。

### 1.7 分页：offset vs cursor vs keyset

分页方案选错的代价巨大 —— 等数据涨到一定规模才暴露，而那时改起来要客户端配合。

**Offset Pagination**（最常见）：

```http
GET /orders?page=1000&size=20
```

```sql
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 19980;
```

底层 MySQL 必须扫 19980+20 = 20000 行才能丢弃前 19980 行返回 20 行。**OFFSET 越大越慢**，且每页查询都是 O(offset) 复杂度。
适用：总数据 < 10 万、UI 必须显示页码的场景。

**Keyset Pagination**（推荐）：

```http
GET /orders?after_id=123456&size=20
```

```sql
SELECT * FROM orders WHERE id > 123456 ORDER BY id LIMIT 20;
```

每次只扫 20 行，恒定 O(1)。**前提**：排序字段必须有索引，且单调（id、create_time）。
缺点：不能跳页（"跳到第 100 页"）；只能"下一页/上一页"。

**Cursor Pagination**（GraphQL 标准）：

```http
GET /orders?cursor=eyJpZCI6MTIzNDU2LCJ0aW1lIjoiMjAyNi0wMS0wMSJ9&size=20
```

cursor 是一个 base64 编码的不透明字符串，服务端可以塞任何东西进去（id + 时间戳 + 排序方向 + tenant_id）。客户端不需要理解内容，只把 cursor 原样回传。

优势：**对客户端完全透明**，服务端可以悄悄改 cursor 格式而不破坏 API；天然支持复杂排序（按 score + create_time 联合分页）。
缺点：实现略复杂；不能跳页。

**选型口诀**：

- 后台管理界面（要跳页、数据量小）→ offset
- App 信息流（"下一页"无限滚动）→ cursor
- 数据导出（按主键全量遍历）→ keyset

> URL / Header / Media Type 三种版本管理你只能选一种；offset / cursor / keyset 三种分页可以共存，但**不要在同一个接口上支持两种**。

---

## 2. 工程切片：从 OpenAPI 到 Gateway 的全栈

> 本节示例基于 Spring Boot 3.x、OpenAPI 3.1、grpc-java 1.62、Apollo GraphQL 5.x。

### 2.1 OpenAPI 3 规范设计

OpenAPI 是 API 的"契约语言"。一个完整的 OpenAPI 3 文档由 `info / servers / paths / components / security` 几个顶层节点组成。

最小可用的 OpenAPI 文档：

```yaml
openapi: 3.1.0
info:
  title: Order Service API
  version: 1.0.0
  description: 订单服务 - 提供下单、查询、取消能力
  contact:
    email: api@example.com
  license:
    name: Apache 2.0
servers:
  - url: https://api.example.com/v1
    description: 生产
  - url: https://api-staging.example.com/v1
    description: 预发
paths:
  /orders/{orderId}:
    get:
      summary: 查询订单
      operationId: getOrder
      parameters:
        - $ref: '#/components/parameters/OrderId'
      responses:
        '200':
          description: 订单详情
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
        '404':
          $ref: '#/components/responses/NotFound'
      security:
        - bearerAuth: []
components:
  parameters:
    OrderId:
      name: orderId
      in: path
      required: true
      schema: { type: string, pattern: '^ord_[a-z0-9]{16}$' }
  schemas:
    Order:
      type: object
      required: [id, status, amount, createdAt]
      properties:
        id:
          type: string
          example: ord_a1b2c3d4e5f6g7h8
        status:
          type: string
          enum: [pending, paid, shipped, delivered, cancelled]
        amount:
          type: number
          format: decimal
          minimum: 0
        createdAt:
          type: string
          format: date-time
  responses:
    NotFound:
      description: 资源不存在
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/Problem'
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
```

几个**很多人写错**的细节：

- `operationId` 必须**全文档唯一**，是代码生成器生成方法名的依据
- `$ref` 引用是 JSON Pointer，必须用 `/` 而不是 `.`
- `additionalProperties: false` 默认是 true（允许任意字段），要严格校验必须显式关掉
- OpenAPI 3.1 才完整对齐 JSON Schema，3.0 只对齐了 draft-05 的子集

**components 是去重的核心**。同一个 Schema 在多个接口被引用时，统一定义一次。否则文档膨胀到几千行很正常。

### 2.2 Swagger UI / Redoc 文档生成

OpenAPI 是规范，Swagger UI 和 Redoc 是渲染器。

**Swagger UI**：交互式（直接在页面里发请求），适合开发联调。
**Redoc**：纯展示型，三栏布局，适合做对外的 API 文档站点。

最快的本地预览：

```bash
# Swagger UI
docker run -p 8080:8080 -e SWAGGER_JSON=/spec/openapi.yaml \
  -v $(pwd):/spec swaggerapi/swagger-ui

# Redoc
docker run -p 8080:80 -v $(pwd)/openapi.yaml:/usr/share/nginx/html/openapi.yaml \
  redocly/redoc
```

CI 里用 [Spectral](https://github.com/stoplightio/spectral) 校验规范：

```bash
spectral lint openapi.yaml --ruleset .spectral.yaml
```

`.spectral.yaml` 可以配置规则，比如"所有路径必须有 summary""所有 4xx 必须有错误响应"。

### 2.3 SpringDoc OpenAPI 集成

**SpringFox 已死，SpringDoc 是接班人**。SpringFox 最后一次有意义的更新是 2020 年，对 Spring Boot 3 / Jakarta EE 9+ 完全不兼容，**新项目千万别用**。

引入 SpringDoc：

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.5.0</version>
</dependency>
```

```yaml
springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
    operations-sorter: method
```

注解示例：

```java
@RestController
@RequestMapping("/v1/orders")
@Tag(name = "Orders", description = "订单管理")
public class OrderController {

    @Operation(summary = "查询订单", description = "根据订单 ID 查询订单详情")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "成功",
            content = @Content(schema = @Schema(implementation = OrderDto.class))),
        @ApiResponse(responseCode = "404", description = "订单不存在",
            content = @Content(schema = @Schema(implementation = Problem.class)))
    })
    @GetMapping("/{orderId}")
    public OrderDto getOrder(
        @Parameter(description = "订单 ID", example = "ord_a1b2c3d4e5f6g7h8")
        @PathVariable String orderId) {
        return orderService.findById(orderId);
    }
}
```

启动后访问 `http://localhost:8080/swagger-ui.html` 即可。文档会**反射代码自动生成**，所以代码就是真相，不会出现"文档过时"。

**生产环境必须关闭 Swagger UI**（或加鉴权），否则就是案例 ⑧ 的事故。

### 2.4 GraphQL Schema 设计与 N+1 解决（DataLoader）

GraphQL 的核心思想：**客户端决定要哪些字段，而不是服务端**。一个 schema 示例：

```graphql
type Query {
  order(id: ID!): Order
  orders(after: String, first: Int = 20): OrderConnection!
}

type Order {
  id: ID!
  status: OrderStatus!
  amount: Float!
  user: User!
  items: [OrderItem!]!
  createdAt: DateTime!
}

type User {
  id: ID!
  name: String!
  email: String!
  address: Address
}

type OrderConnection {
  edges: [OrderEdge!]!
  pageInfo: PageInfo!
}

enum OrderStatus { PENDING PAID SHIPPED DELIVERED CANCELLED }
```

客户端查询：

```graphql
query {
  orders(first: 10) {
    edges {
      node {
        id
        amount
        user { name }
      }
    }
  }
}
```

**N+1 灾难现场**：服务端的 resolver 朴素实现：

```java
@Component
public class OrderResolver {
    @SchemaMapping(typeName = "Order", field = "user")
    public User user(Order order) {
        return userRepository.findById(order.getUserId());  // 每个 order 一次查询！
    }
}
```

10 个订单 → 1 次查 orders + 10 次查 user = 11 次查询。100 个订单 → 101 次。这就是 N+1。

**DataLoader 的解决思路**：把同一帧（GraphQL 解析的同一阶段）里所有 `userRepository.findById(id)` 调用**合并成一次** `findByIds([id1, id2, ...])`。

```java
@Component
public class UserDataLoader {

    @Bean
    public BatchLoaderRegistry batchLoaderRegistry() {
        BatchLoaderRegistry registry = new DefaultBatchLoaderRegistry();
        registry.forTypePair(Long.class, User.class)
            .registerMappedBatchLoader((userIds, env) -> {
                // userIds 是一帧内所有需要的 user id 集合
                List<User> users = userRepository.findByIds(userIds);
                Map<Long, User> map = users.stream()
                    .collect(Collectors.toMap(User::getId, u -> u));
                return Mono.just(map);
            });
        return registry;
    }
}

@SchemaMapping(typeName = "Order", field = "user")
public CompletableFuture<User> user(Order order, DataLoader<Long, User> loader) {
    return loader.load(order.getUserId());
}
```

**关键机制**：`loader.load(id)` 不是立即查询，而是把 id 收集起来。直到 GraphQL 引擎完成"这一层所有 resolver 的调用收集"，DataLoader 才一次性 batch 执行。同时它还做 **dedupe**（同一帧里 load 同一个 id 多次只查一次）。

### 2.5 gRPC + Protobuf：方法定义与代码生成

gRPC 用 Protobuf 定义服务契约，**编译期生成代码**，比 OpenAPI + 运行时反射快、强类型。

```protobuf
syntax = "proto3";
package order.v1;

option java_multiple_files = true;
option java_package = "com.example.order.v1";

import "google/protobuf/timestamp.proto";

service OrderService {
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc CreateOrder(CreateOrderRequest) returns (Order);
  rpc ListOrders(ListOrdersRequest) returns (stream Order);     // server streaming
  rpc UploadEvents(stream OrderEvent) returns (UploadResult);   // client streaming
  rpc Chat(stream Message) returns (stream Message);            // bidi streaming
}

message Order {
  string id = 1;
  OrderStatus status = 2;
  double amount = 3;
  google.protobuf.Timestamp created_at = 4;
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;     // 强约定：枚举 0 必须是 UNSPECIFIED
  ORDER_STATUS_PENDING = 1;
  ORDER_STATUS_PAID = 2;
  ORDER_STATUS_SHIPPED = 3;
  ORDER_STATUS_DELIVERED = 4;
  ORDER_STATUS_CANCELLED = 5;
}

message GetOrderRequest {
  string id = 1;
}

message CreateOrderRequest {
  string user_id = 1;
  repeated OrderItem items = 2;
}
```

服务端实现（Java）：

```java
@GrpcService
public class OrderServiceImpl extends OrderServiceGrpc.OrderServiceImplBase {

    @Override
    public void getOrder(GetOrderRequest req, StreamObserver<Order> resp) {
        try {
            Order order = orderRepository.findById(req.getId())
                .orElseThrow(() -> new StatusRuntimeException(
                    Status.NOT_FOUND.withDescription("order not found: " + req.getId())));
            resp.onNext(order);
            resp.onCompleted();
        } catch (Exception e) {
            resp.onError(Status.INTERNAL.withCause(e).asRuntimeException());
        }
    }

    @Override
    public void listOrders(ListOrdersRequest req, StreamObserver<Order> resp) {
        try (Stream<Order> stream = orderRepository.streamAll()) {
            stream.forEach(resp::onNext);
            resp.onCompleted();
        }
    }
}
```

客户端调用：

```java
ManagedChannel channel = ManagedChannelBuilder
    .forAddress("order-service", 9090)
    .usePlaintext()    // 生产用 useTransportSecurity()
    .build();

OrderServiceGrpc.OrderServiceBlockingStub stub =
    OrderServiceGrpc.newBlockingStub(channel)
        .withDeadlineAfter(2, TimeUnit.SECONDS);

Order order = stub.getOrder(GetOrderRequest.newBuilder()
    .setId("ord_xxx")
    .build());
```

**Protobuf 字段编号是契约的核心**。一旦发布，**永远不能改**。

```protobuf
// ❌ 这样改会破坏所有老客户端
message Order {
  string id = 1;
  double amount = 2;     // 原来是 status 用的字段号 2
  OrderStatus status = 7; // 改了字段号
}

// ✅ 只能新增字段，永远不动老字段
message Order {
  string id = 1;
  OrderStatus status = 2;
  double amount = 3;
  string currency = 4;   // 新字段从未用过的编号开始
  reserved 5, 6;         // 即使想删字段，也要 reserved 防止后人误用
  reserved "old_field";  // 字段名也保留
}
```

### 2.6 API Gateway

API Gateway 是入口层的"代理 + 治理"，负责路由、限流、鉴权、协议转换。主流选型：

| Gateway | 语言 | 性能 | 生态 | 适合 |
| --- | --- | --- | --- | --- |
| **Spring Cloud Gateway** | Java + Reactor | 中 | Spring 全家桶 | Java 团队，对延迟不极致敏感 |
| **Kong** | OpenResty (Lua + Nginx) | 高 | 插件市场成熟 | 平台级 API 网关 |
| **Apisix** | OpenResty + etcd | 高 | 国内生态好 | 云原生，etcd 路由 |
| **Envoy** | C++ | 极高 | xDS 协议、Service Mesh 标准 | 大规模微服务 |
| **Traefik** | Go | 高 | 与 K8s Ingress 集成天然 | K8s 优先 |

Spring Cloud Gateway 最小路由配置：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/v1/orders/**
          filters:
            - StripPrefix=2
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
                key-resolver: "#{@userKeyResolver}"
            - name: CircuitBreaker
              args:
                name: orderCircuit
                fallbackUri: forward:/fallback/order
```

```java
@Bean
public KeyResolver userKeyResolver() {
    return exchange -> Mono.just(
        exchange.getRequest().getHeaders().getFirst("X-User-Id")
    );
}
```

**API Gateway 的边界**（这是最容易踩的概念坑）：

- ✅ **横切关注点**：鉴权、限流、CORS、日志、请求 ID 注入
- ✅ **路由**：URL 到后端服务的映射
- ✅ **协议转换**：HTTP ↔ WebSocket、HTTP ↔ gRPC
- ❌ **业务逻辑**：聚合多个后端、做业务计算 —— **这是 BFF 的职责**
- ❌ **数据转换**：复杂的字段映射、字段过滤 —— 业务相关，应在 BFF

混淆 Gateway 和 BFF 是新团队常见错误：把业务逻辑写进 Gateway 后，Gateway 变成"什么都做的胖中间层"，升级困难、性能瓶颈集中。

### 2.7 BFF（Backend for Frontend）模式

BFF 是 SoundCloud 在 2015 年提出的模式：**为每种客户端（Web、iOS、Android、Smart TV）定制一个后端聚合层**，避免一个 API 满足所有客户端导致的妥协。

典型架构：

```
┌──────┐  ┌──────┐  ┌──────┐
│ Web  │  │ iOS  │  │Android│
└──┬───┘  └──┬───┘  └──┬────┘
   │         │         │
┌──▼──┐   ┌──▼──┐   ┌──▼──┐
│ Web │   │ iOS │   │ Adr │   ← BFF 层（每个客户端一个）
│ BFF │   │ BFF │   │ BFF │
└──┬──┘   └──┬──┘   └──┬──┘
   │         │         │
   └─────────┼─────────┘
             │
   ┌─────────┴─────────┐
   ▼         ▼         ▼
┌────┐   ┌────┐   ┌────┐
│User│   │Order│  │Item│   ← 领域服务（统一）
└────┘   └────┘   └────┘
```

每个 BFF 调用领域服务、聚合数据、按客户端的形态裁剪响应。

```java
// Web BFF：要列表 + 详情聚合
@GetMapping("/web/orders")
public WebOrderListResponse getOrdersForWeb() {
    List<Order> orders = orderService.list();
    Map<Long, User> users = userService.findByIds(
        orders.stream().map(Order::getUserId).distinct().toList());
    return WebOrderListResponse.of(orders, users);
}

// Mobile BFF：要分页 + 极简字段
@GetMapping("/mobile/orders")
public MobileOrderListResponse getOrdersForMobile(@RequestParam String cursor) {
    return orderService.listMinimal(cursor, 20);
}
```

**BFF 与 GraphQL 的关系**：GraphQL 是 BFF 的一种实现方式（让客户端自己挑字段）。如果团队有能力维护 GraphQL，可以用一个 GraphQL 网关替代多个 BFF。

> Gateway 管"通用横切"，BFF 管"客户端定制"，GraphQL 是 BFF 的一种实现。三者职责必须清晰划分。

---

## 3. 实测数据：理论 vs 真实数字

### 3.1 REST vs gRPC 在不同消息大小下的吞吐

环境：8 核 Mac M2 / Java 21 / 同机部署，客户端 32 并发，1 分钟压测。

| 消息大小 | REST (JSON) QPS | gRPC (proto) QPS | gRPC P99 (ms) | REST P99 (ms) |
| --- | --- | --- | --- | --- |
| 100B | 28k | 52k | 1.2 | 2.4 |
| 1KB | 22k | 41k | 1.8 | 3.6 |
| 10KB | 8.5k | 19k | 5.2 | 12.4 |
| 100KB | 1.2k | 4.6k | 28 | 110 |
| 1MB | 110 | 480 | 280 | 1100 |

**结论**：

- 小包场景（< 1KB），gRPC 快 **1.8~2x**，主要赢在 HTTP/2 多路复用
- 大包场景（> 10KB），gRPC 快 **3~4x**，proto 二进制编解码 vs JSON 文本差距拉开
- 但 gRPC 不是银弹：浏览器、CLI、调试工具的支持都不如 HTTP/JSON 友好

口诀：**内部服务间用 gRPC（性能、强类型、流式）；对外 API 用 REST（生态、可调试）**；走 Gateway 在边缘做 HTTP↔gRPC 转换。

### 3.2 GraphQL 的 N+1 问题与 DataLoader 优化

场景：查询 100 个订单 + 每个订单的 user + items。

| 实现 | DB 查询次数 | 延迟 (ms) |
| --- | --- | --- |
| 朴素 resolver | 1 + 100 + 100 = 201 | 1820 |
| 加 DataLoader（user + items） | 1 + 1 + 1 = 3 | 95 |
| 加缓存（命中率 90%） | 1 + 1 + 1 = 3（部分命中 cache） | 38 |

DataLoader 把 N+1 干到 O(1) 级。**没装 DataLoader 的 GraphQL 必爆 N+1**。

### 3.3 OpenAPI 文档自动生成对开发效率的影响

我们团队在 6 个项目里做过对照（手写文档 vs SpringDoc 自动生成）：

| 维度 | 手写 Markdown | SpringDoc 自动生成 |
| --- | --- | --- |
| 接口数 100，初版编写 | 16 人天 | 2 人天 |
| 接口变更后文档维护 | 平均滞后 2 周 | 实时同步 |
| 文档与实现不一致投诉 | 月均 8 起 | 月均 0.3 起 |
| 前端联调起步时间 | 接口写完 + 3 天 | 接口写完即开始 |

**结论**：自动生成不是"略好"，是降一个数量级的差距。手写文档的核心问题不是"慢"，是"必然脱节"。

### 3.4 API Gateway 的延迟开销

直连后端 vs 经过 Gateway 的 P99 对比（同机房）：

| 场景 | 直连 P99 (ms) | Gateway P99 (ms) | 增量 |
| --- | --- | --- | --- |
| Spring Cloud Gateway（仅路由） | 4.2 | 6.8 | +2.6 |
| Spring Cloud Gateway（路由+鉴权+限流） | 4.2 | 11.5 | +7.3 |
| Kong（仅路由） | 4.2 | 4.9 | +0.7 |
| Kong（路由+鉴权+限流） | 4.2 | 6.4 | +2.2 |
| Apisix（路由+鉴权+限流） | 4.2 | 5.3 | +1.1 |
| Envoy（路由+鉴权+限流） | 4.2 | 4.8 | +0.6 |

**反直觉**：Spring Cloud Gateway 比 OpenResty 系（Kong / Apisix）慢一个数量级。原因：Reactor 的事件循环再加上 Java 调度，比 OpenResty 在 Nginx 之上跑 Lua 协程开销大。

如果你的服务对延迟敏感（金融交易、广告竞价），**别用 Spring Cloud Gateway**，用 Apisix / Envoy。

> Gateway 的成本不可忽略，做技术选型时必须实测。"反正是中间件，差不了多少" 是错的。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 错误码设计混乱 —— "前端做不出准确的报错提示"

#### 现象

某个支付接口 `POST /payments`，后端的"返回风格"长这样：

```json
// 业务成功
HTTP 200
{ "success": true, "data": { "paymentId": "p_123" } }

// 业务失败：余额不足
HTTP 200
{ "success": false, "code": 4001, "message": "余额不足" }

// 业务失败：账户冻结
HTTP 200
{ "success": false, "code": 4002, "message": "账户已冻结" }

// 系统错误：DB 挂了
HTTP 500
{ "error": "Internal Server Error" }
```

前端在 `axios.interceptors.response` 里做统一处理时，发现：

- HTTP 200 + `success: false` 该走"业务错误处理"还是"成功处理"？
- "余额不足"和"账户冻结"该弹不同的 modal，但前端要 if-else 一堆 code
- 自动生成的 SDK 拿到 200 都默认成功，得每个调用方手动检查 success 字段

#### 根因

混淆了**传输层语义**和**业务层语义**：

- HTTP 状态码是**传输层**的成功/失败
- 业务成功/失败是**应用层**的概念

把所有响应都返 HTTP 200 是早期"前端误解 4xx = 错误就直接打日志"的妥协，但代价是**所有标准 HTTP 工具链都失效**了：浏览器 DevTools 的 Network tab 全是 200 绿色，监控按 4xx/5xx 计算的错误率永远是 0。

#### 修复

按 RFC 7807 重新设计：

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<Problem> handle(BusinessException e) {
        Problem problem = Problem.builder()
            .type("https://api.example.com/errors/" + e.getCode())
            .title(e.getTitle())
            .status(e.getHttpStatus())
            .detail(e.getDetail())
            .code(e.getCode())
            .build();
        return ResponseEntity.status(e.getHttpStatus())
            .contentType(MediaType.parseMediaType("application/problem+json"))
            .body(problem);
    }
}

public enum BusinessError {
    INSUFFICIENT_BALANCE(422, "INSUFFICIENT_BALANCE", "余额不足"),
    ACCOUNT_FROZEN(403, "ACCOUNT_FROZEN", "账户已冻结"),
    DUPLICATE_PAYMENT(409, "DUPLICATE_PAYMENT", "支付已存在"),
    PAYMENT_TIMEOUT(504, "PAYMENT_TIMEOUT", "支付通道超时");

    final int httpStatus;
    final String code;
    final String message;
}
```

响应：

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json

{
  "type": "https://api.example.com/errors/INSUFFICIENT_BALANCE",
  "title": "余额不足",
  "status": 422,
  "code": "INSUFFICIENT_BALANCE",
  "balance": 50.00,
  "required": 100.00
}
```

前端处理：

```javascript
// axios 拦截器
axios.interceptors.response.use(
  resp => resp,  // 2xx 走这里
  err => {
    const problem = err.response.data;
    // 统一按 problem.code 路由到具体 handler
    errorHandlers[problem.code]?.(problem);
    throw err;
  }
);
```

#### 验证

```bash
# 监控指标：5xx 率应该归零（不再被业务错误污染）
# 业务错误率：4xx 应能区分到 code 维度

curl -i http://api/payments -d '{...}'
# 期望：HTTP 422 + Content-Type: application/problem+json
```

> **口诀**：HTTP 状态码表达"传输/语义层结果"，业务码表达"具体哪种业务结果"。两者分离，全套工具链才能用。

---

### 案例 ② 版本管理失败 —— "v2 上线两周后被迫回滚"

#### 现象

订单服务从 v1 升级到 v2，主要变化是 `Order.amount` 从 `Number`（小数）改成 `String`（避免 JS 精度丢失）。

```http
# v1
GET /orders/123 → { "id": "123", "amount": 100.50 }

# v2
GET /v2/orders/123 → { "id": "123", "amount": "100.50" }
```

上线两周后发现：

- 老 App 客户端（v1）请求 `/v2/orders/...` 返回字符串，crash
- 新 App 客户端（v2）通过老的 BFF 拿到 `amount` 是 Number，crash
- 后台管理系统的 SDK 是手写的，没人通知改 URL，全员 404

被迫回滚。

#### 根因

不是版本管理方案错，是**没有"过渡期"和"灰度"**。版本切换的最低标准：

1. **新老版本共存** —— 至少 3 个月
2. **客户端按版本路由** —— 不是简单"全切"
3. **Deprecation 周期** —— 通过 `Deprecation` Header / 文档提前 3 个月告知
4. **监控老版本调用源** —— 知道还有谁没升

更深的根因：**这个改动根本不该开新版本**。`amount: Number` → `amount: String` 是破坏性变更，但更优雅的做法是**新增字段**：

```json
{ "id": "123", "amount": 100.50, "amount_str": "100.50" }
```

让所有客户端自然过渡到 `amount_str`，几个月后再删 `amount`。

#### 修复

正确的 v2 上线流程：

```
T-90 天：v2 接口上线，与 v1 并存
        所有 v1 响应加 Header: Deprecation: true, Sunset: 2026-08-01
        监控大盘看 v1 调用量按 client_id 分布

T-60 天：通过 SDK 升级、文档、邮件推动客户端迁移
        v1 调用量目标降至 < 30%

T-30 天：v1 调用量降至 < 5%
        给剩余客户端发"最后通牒"

T-0  天：v1 接口下线，返回 410 Gone
```

破坏性变更的判断清单（凡命中一条都是破坏性）：

- 删除字段或改字段类型
- 收紧字段约束（required → optional 是兼容的；optional → required 是破坏性）
- 改字段名
- 改 URL 结构
- 改错误码语义
- 改默认值（可能让老客户端行为变化）

非破坏性变更（可直接发同一版本）：

- 新增字段（响应里）
- 新增可选请求字段
- 新增枚举值（**前提**：客户端用"未知值容错"模式）
- 性能优化、bug 修复

#### 验证

```yaml
# OpenAPI 中标记 Deprecated
paths:
  /orders/{id}:
    get:
      deprecated: true    # SwaggerUI 会显示删除线
      description: |
        ⚠️ DEPRECATED. Use /v2/orders/{id} instead.
        Sunset: 2026-08-01
```

```http
# 老 API 响应必须带这两个 Header
HTTP/1.1 200 OK
Deprecation: true
Sunset: Sat, 01 Aug 2026 00:00:00 GMT
Link: <https://api.example.com/v2/orders/123>; rel="successor-version"
```

监控指标：按 `client_version` 维度统计 v1 调用量，能看到下降曲线。

> **口诀**：API 升级 = **新版本上线 + 老版本下线** 两件事，中间必须有 3 个月以上的过渡期。能用"加字段"解决的，绝不开新版本。

---

### 案例 ③ N+1 查询 —— "GraphQL 上线后 DB 直接被打挂"

#### 现象

GraphQL 服务上线第一天，DB 监控告警：QPS 从 2k 飙到 80k，连接池打满，所有依赖 DB 的服务雪崩。

#### 根因

某个朴素的 resolver：

```java
@SchemaMapping(typeName = "Order", field = "user")
public User user(Order order) {
    return userRepository.findById(order.getUserId()).orElseThrow();
}

@SchemaMapping(typeName = "Order", field = "items")
public List<OrderItem> items(Order order) {
    return itemRepository.findByOrderId(order.getId());
}
```

客户端的查询：

```graphql
query {
  orders(first: 100) {
    edges {
      node {
        id
        user { name email }
        items { name price }
      }
    }
  }
}
```

实际 SQL：1 次查 100 个 order，然后**对每个 order**：

- 1 次查 user
- 1 次查 items

总共 1 + 100 + 100 = **201 次查询**。

#### 修复

引入 DataLoader：

```java
@Configuration
public class GraphQLConfig {

    @Bean
    public BatchLoaderRegistry batchLoaderRegistry(
            UserRepository userRepo,
            OrderItemRepository itemRepo) {
        BatchLoaderRegistry registry = new DefaultBatchLoaderRegistry();

        // user batch loader
        registry.forTypePair(Long.class, User.class)
            .registerMappedBatchLoader((userIds, env) -> {
                List<User> users = userRepo.findAllById(userIds);
                return Mono.just(users.stream()
                    .collect(Collectors.toMap(User::getId, u -> u)));
            });

        // items batch loader（注意：返回 List<OrderItem>，不是单个）
        registry.forTypePair(Long.class, List.class)
            .registerMappedBatchLoader((orderIds, env) -> {
                List<OrderItem> items = itemRepo.findByOrderIdIn(orderIds);
                Map<Long, List<OrderItem>> grouped = items.stream()
                    .collect(Collectors.groupingBy(OrderItem::getOrderId));
                // 没有 items 的 order 也要返回空 list，否则 DataLoader 会一直 pending
                Map<Long, List<OrderItem>> result = new HashMap<>();
                for (Long id : orderIds) {
                    result.put(id, grouped.getOrDefault(id, List.of()));
                }
                return Mono.just(result);
            });

        return registry;
    }
}

@SchemaMapping(typeName = "Order", field = "user")
public CompletableFuture<User> user(Order order, DataLoader<Long, User> loader) {
    return loader.load(order.getUserId());
}
```

现在的查询次数：1（orders）+ 1（users batch）+ 1（items batch）= **3 次**。

#### 进阶：Query Complexity 限制

DataLoader 解决的是"扁平 N+1"。但 GraphQL 的灵活性意味着客户端可以发"嵌套 N 层"的查询：

```graphql
query NastySinger {
  user(id: 1) {
    friends {        # 100 个
      friends {      # 100 * 100 = 10000
        friends {    # 100 * 100 * 100 = 1000000
          name
        }
      }
    }
  }
}
```

必须配 query complexity 限制：

```java
GraphQL graphQL = GraphQL.newGraphQL(schema)
    .instrumentation(new MaxQueryComplexityInstrumentation(1000))
    .instrumentation(new MaxQueryDepthInstrumentation(10))
    .build();
```

#### 验证

```bash
# 用 Apollo Server 的 tracing 看每个 resolver 的调用次数和耗时
curl -X POST http://api/graphql \
  -H 'Apollo-Tracing: true' \
  -d '{"query": "..."}'

# 也可以在 GraphQL response extensions 里看
{
  "data": {...},
  "extensions": {
    "tracing": {
      "execution": {
        "resolvers": [
          { "path": ["orders", 0, "user"], "duration": 1230000 },
          ...
        ]
      }
    }
  }
}
```

> **口诀**：GraphQL 没装 DataLoader = N+1 必现。DataLoader 是 GraphQL 的"标配组件"而不是"可选优化"。

---

### 案例 ④ 分页 offset 太大 —— "运营页面卡死"

#### 现象

运营导出订单工具：

```
GET /admin/orders?page=10000&size=100
```

第一页秒返回，第 10000 页（offset = 999900）卡 8 秒。第 50000 页直接超时。DBA 抱怨慢查询日志被刷屏。

#### 根因

```sql
SELECT * FROM orders ORDER BY created_at DESC LIMIT 100 OFFSET 999900;
```

MySQL 的执行计划（即使有 created_at 索引）：

1. 走 created_at 索引扫描，**从头开始**
2. 跳过前 999900 行（这些行被读出来又丢弃）
3. 返回接下来的 100 行

OFFSET 越大，"读出来又丢弃"的代价越高。这是 RDBMS 的**根本限制**，不是某个数据库的 bug。

更糟：如果查询里有 ORDER BY 用了非索引列，MySQL 还要做 filesort，OFFSET 1000000 就是排序 1000100 行后丢弃。

#### 修复

**方案 A**：keyset pagination（推荐，对内）。

```http
GET /admin/orders?after_id=ord_xxx&size=100
```

```sql
SELECT * FROM orders WHERE id < 'ord_xxx' ORDER BY id DESC LIMIT 100;
```

恒定 O(100)。前提：`id` 索引 + 排序字段唯一（否则同时间戳的多条记录会被跳过）。

**方案 B**：cursor pagination（推荐，对外）。

```http
GET /orders?cursor=eyJpZCI6Im9yZF94eHgiLCJ0aW1lIjoxNzAwMDAwMDAwfQ==&size=100
```

服务端 cursor 编码：

```java
public String encodeCursor(Order last) {
    Map<String, Object> data = Map.of(
        "id", last.getId(),
        "time", last.getCreatedAt().toEpochSecond());
    return Base64.getEncoder().encodeToString(
        objectMapper.writeValueAsBytes(data));
}

public Page<Order> findAfter(String cursor, int size) {
    if (cursor == null) {
        return repo.findFirst(size);
    }
    Map<String, Object> data = objectMapper.readValue(
        Base64.getDecoder().decode(cursor), Map.class);
    return repo.findAfter(
        (String) data.get("id"),
        Instant.ofEpochSecond((Long) data.get("time")),
        size);
}
```

```sql
SELECT * FROM orders
WHERE (created_at, id) < (?, ?)
ORDER BY created_at DESC, id DESC
LIMIT 100;
```

**方案 C**：必须支持跳页时的"延迟关联"。

```sql
-- 慢
SELECT * FROM orders ORDER BY created_at LIMIT 100 OFFSET 999900;

-- 快（先用覆盖索引拿主键）
SELECT * FROM orders o
INNER JOIN (
    SELECT id FROM orders ORDER BY created_at LIMIT 100 OFFSET 999900
) t ON o.id = t.id;
```

第二个 SQL 让"跳过 99.99 万行"只走主键索引，不读完整行。能快 5~10 倍，但还是 O(offset)。

#### 验证

```sql
-- 看执行计划
EXPLAIN SELECT * FROM orders ORDER BY id DESC LIMIT 100 OFFSET 999900;
-- type=index，Extra="Using filesort" 是危险信号

-- keyset 版本应该走索引扫描，rows≈100
EXPLAIN SELECT * FROM orders WHERE id < 'ord_xxx' ORDER BY id DESC LIMIT 100;
```

性能对比（500 万行表）：

| 方案 | 第 1 页 (ms) | 第 1000 页 (ms) | 第 10000 页 (ms) |
| --- | --- | --- | --- |
| OFFSET 简单分页 | 12 | 320 | 8200 |
| OFFSET + 延迟关联 | 8 | 42 | 580 |
| Keyset / Cursor | 11 | 11 | 12 |

> **口诀**：`OFFSET > 10000` 必慢。后台跳页用延迟关联，App 流式滚动用 keyset/cursor。

---

### 案例 ⑤ 超时叠加 —— "接口返回成功但数据没落库"

#### 现象

调用链：API → ServiceA → ServiceB → DB。

每层超时配置：

- API → A：2s
- A → B：1s
- B → DB：1s

某天监控大盘：API 返回 200 的请求里，**有 0.3% 的订单在 DB 里查不到**。客户付款成功了，但订单没创建。

#### 根因

超时配置的"反向叠加"：

```
T = 0     API 收到请求，给 A 2s
T = 0     A 收到请求，给 B 1s
T = 0     B 收到请求，给 DB 1s
T = 0.95  DB 还在写（接近超时）
T = 1.0   B 客户端超时，返回 504 给 A
T = 1.0   B 服务端的 DB 写入还在跑（DB 不知道客户端断了）
T = 1.05  DB 写入成功（落库了！）
T = 1.0   A 收到 B 的 504，返回 504 给 API
T = 1.0   API 触发"重试"逻辑（GET 类幂等接口默认重试一次）
T = 1.0   API 第二次请求 → A → B → DB
T = 1.05  DB 写入又成功一次（重复落库）
T = 1.5   全链路完成
T = 2.0   API 给客户端返回 200
```

**关键 bug**：B 客户端超时后**不通知 B 服务端取消**，B 服务端的事务照样跑完。

如果是 GET 类幂等接口，重试不会出错。**写接口加 retry 就出大问题**：要么重复落库，要么客户端 504 但实际成功，最差的是 504 + 一会儿数据又出现了。

#### 修复

**原则一：deadline 必须传播**。

gRPC 的解法：每层调用都传递剩余时间。

```java
// API 层
ManagedChannel channel = ...;
ServiceAStub stub = ServiceAGrpc.newBlockingStub(channel)
    .withDeadlineAfter(2, TimeUnit.SECONDS);

// A 层
@Override
public void doWork(Request req, StreamObserver<Response> resp) {
    Context current = Context.current();
    Deadline deadline = current.getDeadline();    // 拿到 API 设的 2s deadline
    ServiceBStub stub = ServiceBGrpc.newBlockingStub(channel)
        .withDeadline(deadline);    // 不是 1s，是"剩余时间"
    // ...
}
```

HTTP 没有标准的 deadline 头，但可以自己定义：

```http
GET /orders/123
X-Deadline-Ms: 2000
```

每层拿到这个 Header，**减去自己已花的时间**再传给下一层。

**原则二：写接口必须靠业务幂等键，不能依赖 HTTP method**。

```java
@PostMapping("/payments")
public Payment createPayment(
    @RequestHeader("Idempotency-Key") String idempotencyKey,
    @RequestBody PaymentRequest req) {

    return idempotencyStore.computeIfAbsent(idempotencyKey, key -> {
        return paymentService.create(req);
    });
}
```

**原则三：不开自动重试，让上游显式重试**。

```yaml
# 错：HTTP 客户端默认重试一次
spring.cloud.openfeign.client.config.default.retryable: true

# 对：默认不重试
spring.cloud.openfeign.client.config.default.retryable: false
```

谁需要重试，谁带 `Idempotency-Key` 显式发起。

**原则四：超时层级有"足够空间"**。

```
最外层超时 = 最内层超时 × 调用层数 × 2
```

例：DB 1s → B 1.5s → A 3s → API 5s。给中间层留一些预留时间避免边界叠加。

#### 验证

```java
// 故意制造慢调用，观察是否会出现"超时但成功"
@Test
void slowDbShouldNotCauseDoubleWrite() throws Exception {
    String key = UUID.randomUUID().toString();

    // 模拟 DB 慢 800ms，B 客户端超时 1s
    when(dbWrite.execute(any())).thenAnswer(inv -> {
        Thread.sleep(800);
        return ok();
    });

    // 第一次：should succeed
    var f1 = CompletableFuture.runAsync(() -> client.create(key, req));
    Thread.sleep(50);
    // 第二次（重试）：必须命中幂等
    var f2 = CompletableFuture.runAsync(() -> client.create(key, req));

    f1.get();
    f2.get();

    // 验证 DB 只写了 1 次
    verify(dbWrite, times(1)).execute(any());
}
```

> **口诀**：deadline 必须传播；写接口必须用 `Idempotency-Key`；HTTP 客户端默认不重试；外层超时 ≥ 内层 × 2。

---

### 案例 ⑥ 幂等键设计错误 —— "用消息 ID 还是业务 ID"

#### 现象

订单服务消费 MQ 消息创建订单：

```java
@KafkaListener(topics = "order-create")
public void onMessage(ConsumerRecord<String, OrderEvent> record) {
    String idempotencyKey = record.key();    // Kafka message key 作为幂等键
    if (orderRepository.existsByIdempotencyKey(idempotencyKey)) return;
    orderService.create(record.value());
    orderRepository.markIdempotent(idempotencyKey);
}
```

测试时一切正常。生产某次 Kafka rebalance 后出现重复订单。

#### 根因

Kafka 的"重试 + rebalance"会导致**同一条消息被多次投递，但 message key 可能不同**：

- 上游生产者重发：可能用同一个 key（如果 partitioner 决定的）
- 上游生产者**自己**重试：可能生成新的 message ID
- Consumer rebalance：同一条消息被另一个 consumer 重新消费，message metadata 一致

更深的问题：**幂等键的"主语"应该是业务实体，不是消息**。

举例：

- 用户在 App 上点"下单"按钮 100 次（手抖）
- 每次点击产生一条 MQ 消息（100 条独立 message，各自的 message id 不同）
- 但**业务上**这是同一个"下单意图"

如果用 message id 做幂等，100 条消息会创建 100 个订单。

#### 修复

**正确的幂等键来源**：

```java
class OrderRequest {
    String userId;
    String productId;
    long quantity;
    String requestId;      // ★ 客户端生成的唯一 ID（uuid）
    long clientTimestamp;
}
```

**幂等键 = 业务维度的"操作意图"唯一标识**：

```java
String idempotencyKey = req.getRequestId();
// 或者更严格：requestId + userId 防止 ID 冲突
```

**完整模式**：

```java
@PostMapping("/orders")
public Order create(
    @RequestHeader(value = "Idempotency-Key", required = true) String idempotencyKey,
    @Valid @RequestBody OrderRequest req) {

    return idempotencyService.execute(idempotencyKey, () -> {
        return orderService.create(req);
    });
}

@Service
public class IdempotencyService {

    public <T> T execute(String key, Supplier<T> action) {
        // 1. 查缓存
        Optional<CachedResponse> cached = cache.get(key);
        if (cached.isPresent()) {
            return (T) cached.get().payload;
        }

        // 2. 加分布式锁（防止并发同一 key）
        try (Lock lock = redisLock.acquire("idem:" + key, 30, TimeUnit.SECONDS)) {
            // 3. 双重检查
            cached = cache.get(key);
            if (cached.isPresent()) {
                return (T) cached.get().payload;
            }

            // 4. 执行业务
            T result = action.get();

            // 5. 缓存结果（TTL 24h，覆盖典型重试窗口）
            cache.put(key, new CachedResponse(result), Duration.ofHours(24));
            return result;
        }
    }
}
```

**Idempotency-Key 的存储设计**（参考 Stripe）：

```sql
CREATE TABLE idempotency_records (
    key VARCHAR(255) PRIMARY KEY,
    user_id BIGINT NOT NULL,                  -- key 必须按"租户/用户"维度隔离
    request_hash VARCHAR(64) NOT NULL,         -- 同 key 不同 body 必须报错
    response_status SMALLINT,
    response_body JSON,
    created_at TIMESTAMP NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    INDEX idx_expires (expires_at)
);
```

如果同一个 `Idempotency-Key` 对应不同的 `request_hash` —— 抛 422 错误。这是 Stripe 的设计：避免用户 bug 把"订单 A 的 key"用到"订单 B"。

#### 验证

```java
@Test
void sameKeyDifferentBodyShouldFail() {
    String key = "test-key-1";
    paymentApi.create(key, new Request("100"));
    assertThrows(BadRequestException.class, () ->
        paymentApi.create(key, new Request("200")));
}

@Test
void sameKeySameBodyReturnsCachedResponse() {
    String key = "test-key-2";
    Payment p1 = paymentApi.create(key, new Request("100"));
    Payment p2 = paymentApi.create(key, new Request("100"));
    assertEquals(p1.getId(), p2.getId());     // 同一个 ID，没创建第二个
}
```

> **口诀**：幂等键的"主语"是业务意图，不是消息或事务。**客户端生成 + 服务端缓存 + 同 key 不同 body 报错**是三件套。

---

### 案例 ⑦ API 文档与实际不一致 —— "前端按文档调，发现字段没了"

#### 现象

前端按 Swagger UI 上看到的接口调用：

```javascript
const order = await api.getOrder('123');
console.log(order.totalAmount);   // undefined
```

后端接口实际返回的字段叫 `amount`，文档写的是 `totalAmount`。前后端撕了一下午。

#### 根因

文档手动维护，但代码改了字段名忘记改文档。或者文档是 PM 用 Word 写的，开发改代码不会同步过去。

更恶劣的场景：用了 SpringFox（早期版本），但里面的注解和代码字段不一致：

```java
public class OrderDto {
    private BigDecimal amount;     // 实际字段

    @ApiModelProperty(value = "订单总额", name = "totalAmount")   // 错误的注解
    public BigDecimal getAmount() { return amount; }
}
```

#### 修复

**唯一根治方案：从代码生成文档，且 CI 验证**。

第一步：用 SpringDoc，让代码就是文档：

```java
public class OrderDto {
    @Schema(description = "订单总额", example = "100.50")
    private BigDecimal amount;
    // 字段名直接来自属性，不会脱节
}
```

第二步：CI 中导出 OpenAPI 文档，跟仓库里的版本对比：

```yaml
# .github/workflows/api-check.yml
- name: Export OpenAPI
  run: |
    ./gradlew bootRun &
    sleep 30
    curl http://localhost:8080/v3/api-docs > openapi-current.json

- name: Diff with committed
  run: |
    diff <(jq -S . openapi.json) <(jq -S . openapi-current.json)
    # 不一致 → CI 失败
```

第三步：契约测试。前后端都消费同一份 OpenAPI，前端用它生成 TypeScript 类型，后端验证它，**两边都依赖同一个 source of truth**。

```bash
# 前端从 OpenAPI 生成 TS 类型
npx openapi-typescript ./openapi.json -o src/api-types.ts
```

第四步：消费者驱动契约测试（CDC，[Pact](https://pact.io/) 是事实标准）。

```javascript
// 前端写"消费者契约"
provider.given('order 123 exists')
  .uponReceiving('a request to get order 123')
  .withRequest({ method: 'GET', path: '/orders/123' })
  .willRespondWith({
    status: 200,
    body: { id: '123', amount: like(100.50) }
  });
```

后端在 CI 里下载这个契约，跑测试验证自己满足契约。**契约不满足 → CI 失败**。

#### 验证

```bash
# 部署前必跑：契约对比
pact-broker can-i-deploy --pacticipant order-service --version $GIT_SHA
# 输出 "Computer says yes" 才能部署

# 生产环境定期 sanity check
schemathesis run http://prod-api/v3/api-docs
```

> **口诀**：文档不能手写。要么自动生成 + CI 校验，要么走契约测试。两者都没有 = 注定不一致。

---

### 案例 ⑧ Swagger 暴露在公网 —— "信息泄漏被白帽提交"

#### 现象

某天收到白帽提交的安全报告：

```
https://api.example.com/swagger-ui.html
https://api.example.com/v3/api-docs

暴露了所有内部接口，包括：
- /admin/users/reset-password
- /internal/billing/recharge
- /debug/clear-cache

且 Swagger UI 可以直接发请求。
```

#### 根因

SpringDoc 默认开启 Swagger UI，没人记得在生产关。
更恶心的版本：开发为了好用，把 Swagger 上的"试一下"按钮也开了，导致**任何人都能在线调用 admin 接口**。

#### 修复

**多层防御**：

第一层：生产环境彻底关掉。

```yaml
# application-prod.yml
springdoc:
  api-docs:
    enabled: false
  swagger-ui:
    enabled: false
```

或者**只在开发/测试 profile 启用**：

```java
@Configuration
@Profile({"dev", "test"})
public class SwaggerConfig {
    @Bean
    public OpenAPI openAPI() { return new OpenAPI()...; }
}
```

第二层：必须开（如对外开放 API 平台），加鉴权。

```yaml
springdoc:
  swagger-ui:
    path: /internal/swagger-ui.html    # 改路径，避免被默认扫到
```

```java
@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth
            .requestMatchers("/v3/api-docs/**", "/swagger-ui/**")
                .hasRole("API_DOC_VIEWER")    // 内部账号才能看
            .anyRequest().authenticated());
        return http.build();
    }
}
```

第三层：Gateway 拦截。

```yaml
# Spring Cloud Gateway / Nginx 配置
- id: block-swagger
  uri: forward:/404
  predicates:
    - Path=/v3/api-docs/**, /swagger-ui/**
```

**敏感接口隔离**：

`/admin/`, `/internal/`, `/debug/` 这些路径**根本不该暴露给公网 Gateway**。把它们注册到独立的内网 Gateway，前面有 VPN/堡垒机/IP 白名单。

```yaml
# 内网 Gateway
spring.cloud.gateway.routes:
  - id: admin
    uri: lb://admin-service
    predicates:
      - Path=/admin/**
    filters:
      - name: AddRequestHeader
        args:
          name: X-Internal-Source
          value: admin-gateway
```

业务代码层：

```java
@PreAuthorize("hasRole('ADMIN') and #root.principal.ip startsWith '10.'")
@PostMapping("/admin/users/reset-password")
public void reset(...) {...}
```

#### 验证

```bash
# 生产环境扫描，确认 swagger 路径都 404
for path in /swagger-ui.html /v2/api-docs /v3/api-docs /swagger.json /api-docs; do
    curl -sI https://api.example.com$path | head -1
done

# 全部应该是 HTTP/1.1 404 Not Found

# 渗透测试加进 CI（用 OWASP ZAP）
zap-baseline.py -t https://staging.example.com -r report.html
```

> **口诀**：Swagger UI 在公网 = 信息泄漏。生产必须关，或加鉴权 + 改路径 + Gateway 拦。

---

### 案例 ⑨ gRPC 反射服务在生产开启 —— "攻击者用 grpcurl 直连"

#### 现象

安全扫描报告：

```
某 gRPC 服务 9090 端口对外暴露了 reflection service
攻击者可以用 grpcurl 直接列出所有方法、参数、调用：

$ grpcurl -plaintext target:9090 list
order.v1.OrderService
admin.v1.AdminService     ← 内部服务！
billing.v1.BillingService

$ grpcurl -plaintext target:9090 admin.v1.AdminService.ResetPassword \
  -d '{"user_id": "1"}'
```

#### 根因

gRPC Server Reflection（grpc.reflection.v1alpha.ServerReflection）让客户端无需 .proto 文件就能调用任意方法。开发时爽，生产是灾难。

很多 Spring Boot gRPC 工程默认就开了：

```java
// 一些 starter 默认行为
@GrpcService
public class MyService { ... }
// → reflection service 自动注册
```

#### 修复

**生产关闭 reflection**：

```yaml
# spring-boot-starter-grpc
grpc:
  server:
    reflection-service-enabled: false    # 生产必关
```

手写 grpc-java：

```java
ServerBuilder.forPort(9090)
    .addService(orderService)
    // 不要加：.addService(ProtoReflectionService.newInstance())
    .build()
    .start();
```

**关键认知**：reflection 不只是"列方法"，它还**揭示了所有 message schema**。攻击者拿到 schema 可以构造任意合法请求，连"猜字段名"都不用。

**进一步的纵深防御**：

- gRPC 服务**只暴露在内网**，外网请求必须经过 Gateway 转 HTTP
- 用 mTLS 双向证书认证，没证书的客户端连接直接拒绝
- 开 gRPC 的 interceptor 做认证：

```java
public class AuthInterceptor implements ServerInterceptor {
    @Override
    public <Req, Resp> Listener<Req> interceptCall(
            ServerCall<Req, Resp> call,
            Metadata headers,
            ServerCallHandler<Req, Resp> next) {

        String token = headers.get(Metadata.Key.of("authorization", ASCII_STRING_MARSHALLER));
        if (!authService.validate(token)) {
            call.close(Status.UNAUTHENTICATED, new Metadata());
            return new ServerCall.Listener<>() {};
        }
        return next.startCall(call, headers);
    }
}
```

#### 验证

```bash
# 部署后扫描
grpcurl -plaintext target:9090 list
# 期望输出：Failed to list services: server does not support the reflection API

# 加进自动化扫描脚本，每次 release 必跑
```

> **口诀**：gRPC reflection 是开发利器，生产毒药。两端都要关：服务端不开、客户端拿不到 schema 就老实用 .proto。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 设计层
- [ ] 接口风格选定（REST / RPC / GraphQL / gRPC），全团队统一
- [ ] URL 用名词；HTTP method 表达动作（如选 REST）
- [ ] HTTP 状态码使用正确（4xx ≠ 5xx；422 ≠ 400）
- [ ] 错误响应遵循 RFC 7807 problem+json 或团队统一格式
- [ ] 错误码用稳定字符串（`ACCOUNT_FROZEN`），不用数字
- [ ] 不在 HTTP 200 里返回 `success: false`
- [ ] 写接口必须支持 `Idempotency-Key`
- [ ] 分页选定方案：offset（后台跳页）/ keyset（导出）/ cursor（流式）
- [ ] OFFSET 大于 10000 的查询必须改延迟关联或 keyset

### 文档层
- [ ] 文档自动生成（SpringDoc / 注解驱动），不手写
- [ ] CI 校验 OpenAPI 与代码一致（导出 + diff）
- [ ] OpenAPI 中所有路径有 summary、所有响应有 schema
- [ ] components/schemas 复用，不重复定义
- [ ] 弃用接口必须带 `Deprecation: true` 和 `Sunset` 头
- [ ] 契约测试（Pact / Spring Cloud Contract）覆盖关键调用链

### 版本管理
- [ ] 版本管理方案统一（URL / Header / Media Type 三选一）
- [ ] 破坏性变更必须开新版本
- [ ] 新老版本并存至少 3 个月
- [ ] 监控老版本调用量按 client 维度
- [ ] 能用"加字段"解决的，绝不开新版本

### 安全层
- [ ] **生产环境关闭 Swagger UI**，或加鉴权 + 改路径
- [ ] **生产环境关闭 gRPC reflection**
- [ ] `/admin /internal /debug` 路径不暴露公网
- [ ] 敏感接口加 `@PreAuthorize` + IP 白名单
- [ ] CORS 配置精确到 origin 列表，不用 `*`
- [ ] 限流配置到接口 + 用户维度
- [ ] 日志不记录请求/响应中的敏感字段（密码、token、身份证）

### 调用层
- [ ] HTTP 客户端默认不重试（重试由调用方显式发起 + 幂等键）
- [ ] gRPC deadline 必须传播
- [ ] 外层超时 ≥ 内层 × 调用层数 × 2
- [ ] 调用方必须能拿到自己的 traceId

### Gateway 层
- [ ] Gateway 只做横切（鉴权 / 限流 / 路由），不做业务
- [ ] 业务聚合放在 BFF
- [ ] 内网调用走 mTLS / Service Mesh，不暴露明文
- [ ] Gateway 性能实测（不要假设 "Java 网关够用"）

### GraphQL 层（如使用）
- [ ] DataLoader 是标配，不是优化
- [ ] Query complexity 限制
- [ ] Query depth 限制
- [ ] 关键字段加权限粒度（field-level authorization）
- [ ] 禁用 introspection 在生产（或加鉴权）

### gRPC 层（如使用）
- [ ] Protobuf 字段编号一旦发布永不修改
- [ ] 删字段用 `reserved` 而不是直接删
- [ ] 枚举 0 值必须是 UNSPECIFIED
- [ ] 服务端不开 reflection
- [ ] 客户端配 deadline

---

## 6. 还能往下挖什么

读完本文只是起点。如果想再深一层：

### REST 进化
- **HATEOAS 与 Hypermedia**：Spring HATEOAS / HAL 格式 / Siren。学了再决定**不用**它，理论价值大于实战
- **JSON:API 规范** —— 业界统一的 REST 响应格式，对资源关系建模很优雅，但 verbose
- **JSON Schema vs OpenAPI Schema**：OpenAPI 3.1 与 JSON Schema 2020-12 完全对齐，可以反向用

### 文档与 Mock
- **OpenAPI Mock 服务（Prism）**：从 OpenAPI 直接起 mock 服务，前端不等后端
- **Postman / Insomnia 与 OpenAPI 双向同步**
- **Schemathesis** —— 基于 OpenAPI 的 fuzz 测试

### gRPC 生态
- **gRPC-Web / Connect**：让浏览器原生支持 gRPC（Connect 协议比 grpc-web 更优雅）
- **gRPC + Service Mesh**：xDS、Envoy、Istio 的统一数据平面
- **buf** —— Protobuf 的"npm"，依赖管理 + lint + 生成代码

### GraphQL 进阶
- **GraphQL Federation**（Apollo Federation 2 / GraphQL Mesh）：多服务的 schema 联邦
- **Persisted Queries**：客户端预注册查询，运行时只发 hash，安全 + 性能双赢
- **Subscription** vs WebSocket vs SSE：实时推送的几种方案

### 异步与流式
- **AsyncAPI**：异步消息（Kafka / RabbitMQ）的"OpenAPI 版本"
- **CloudEvents**：异步消息的标准信封格式
- **Server-Sent Events (SSE)** vs WebSocket vs gRPC Streaming：选型
- **WebHooks 设计模式**：重试、签名、幂等、Replay Attack

### 现代客户端工具链
- **OpenAPI TypeScript 生成**：openapi-typescript / orval
- **tRPC**：TypeScript 全栈下"不用 OpenAPI 也能类型安全"的方案
- **Zod / yup** —— 运行时 schema 校验，与 OpenAPI 互补

### API 治理
- **API Mesh / API Composition**：微服务组合的两种思路
- **Spectral / Vacuum** —— OpenAPI 的 lint 工具
- **API 评审制度**：Stripe / Google / Atlassian 的内部规范

---

## 7. 引用

- Roy Fielding, *Architectural Styles and the Design of Network-based Software Architectures* (2000)
- Leonard Richardson, *RESTful Web Services* (2007) —— "REST 成熟度模型"
- RFC 7231 *HTTP/1.1 Semantics and Content* —— HTTP method 与状态码语义
- RFC 7807 *Problem Details for HTTP APIs* —— 错误响应标准
- [OpenAPI Specification 3.1.0](https://spec.openapis.org/oas/v3.1.0)
- [Stripe API Idempotency Documentation](https://stripe.com/docs/api/idempotent_requests)
- [GraphQL Specification](https://spec.graphql.org/)
- [gRPC Documentation](https://grpc.io/docs/)
- Sam Newman, *Building Microservices* (2nd ed., 2021) —— BFF 模式
- [Google API Design Guide (AIP)](https://google.aip.dev/)
- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines)
- [Zalando RESTful API Guidelines](https://opensource.zalando.com/restful-api-guidelines/)

---

> 配套：[DDD 战术战略](/article/architecture/ddd) | [系统设计主题](/article/system-design/) | [HTTP/1→2→3 演进](/article/network/http-evolution)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
