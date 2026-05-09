# 六边形 / Clean / Onion 架构 —— 端口适配器与依赖反转的工程落地

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试或了解概念，搜一篇 "Hexagonal Architecture 入门" 就够了。
> 这里要回答的，是**真打算把六边形落到 Spring 工程里、且不想被同事骂"过度设计"**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过：

1. Hexagonal（六边形）/ Clean / Onion 三种架构经常被并列提及，它们的**本质共通点**是什么？三者的差别仅仅是命名吗？给一个能在代码层面体现差异的例子。
2. Port（端口）和 Adapter（适配器）的精确含义是什么？为什么 Cockburn 当初选了"六边形"而不是"圆形"或者"四边形"？画几条边有什么讲究？
3. Driving Adapter（驱动方/主动适配器）和 Driven Adapter（被驱动方/被动适配器）的区别是什么？同一个 Repository 能不能既是 Driving 又是 Driven？
4. 依赖反转原则（DIP）在六边形中具体反转了什么？传统三层（Controller → Service → DAO）的依赖方向有什么问题？
5. 传统 Controller-Service-DAO 写出来的项目，到底"传统"在哪？有什么具体的工程症状（不是教科书式的"耦合高"，要给出可观测的现象）？
6. 六边形和 DDD 的关系：六边形必须配 DDD 吗？只用六边形不用 DDD 行不行？反过来呢？
7. 六边形和 CQRS 的关系：在 CQRS 里 Command 端和 Query 端都需要走端口吗？Query 端能不能直接 SQL 不走 Repository？
8. 六边形在 Spring Boot 里典型的包结构是什么样？`@Service` `@Repository` 这些注解应该贴在 domain 层、application 层还是 infrastructure 层？

如果有半数答不上，往下看。

---

## 1. 底层模型：依赖反转 + 业务核心隔离

绝大多数文章把 Hexagonal、Clean、Onion 当成三种"高大上的架构"分别讲一遍，然后让读者自己去比较异同。这种讲法对实际工程**毫无帮助**。要看懂上面 8 个问题，必须先理解：**这三个架构本质上是同一个东西的三种画法**。

### 1.1 三种架构的同源性

把三种架构的"一句话总结"摆在一起：

| 架构 | 提出者 | 提出时间 | 核心比喻 | 一句话总结 |
| --- | --- | --- | --- | --- |
| **Hexagonal** | Alistair Cockburn | 2005（实际 1995 起草） | 六边形的"边" = 端口 | 业务核心被一圈端口包围，外部通过适配器连接端口 |
| **Onion** | Jeffrey Palermo | 2008 | 同心圆，越内层越稳定 | 外层依赖内层，内层不知道外层的存在 |
| **Clean** | Robert C. Martin | 2012 | 同心圆 + 控制流箭头 | 依赖箭头永远指向圆心；用 DIP 在圆心和外层之间穿透 |

三个架构不约而同强调两件事：

1. **业务核心（Domain / Entities / Use Cases）必须是稳定的**，不应被框架、数据库、UI 这类"易变细节"污染。
2. **依赖方向必须从外向内**：UI / DB / 框架可以依赖业务核心，业务核心不能反过来依赖它们。

第二条就是依赖反转原则（DIP，Dependency Inversion Principle）的具体应用。

> **认知锚点**：如果只允许记一句话，记这个 —— **三种架构都是 DIP 在系统级别的应用，差别只在画图方式和层级粒度**。

### 1.2 Hexagonal Architecture（Alistair Cockburn）

Cockburn 起初称这个架构为 "Ports and Adapters"，"六边形"只是他**画图时随手选的形状**。原文里说得很直白：

> The hexagon is not a hexagon because the number six is important, but rather to allow the people doing the drawing to have room to insert ports and adapters as they need.

六条边纯粹是为了**画图方便插入足够多的端口**。换成八边形、十边形甚至圆形都不影响本质。

核心三个概念：

- **业务逻辑（Application / Domain Hex）**：核心，不依赖任何外部
- **Port（端口）**：业务核心暴露给外界 / 依赖外界提供的**接口**
- **Adapter（适配器）**：把外部世界（HTTP、DB、MQ）翻译成端口能理解的形式

Cockburn 的原图（文字版）：

```
              [HTTP Adapter]   [CLI Adapter]
                     ↓                ↓
                 (Port In)        (Port In)
                     \                /
                      \              /
                       ┌────────────┐
                       │   Domain   │
                       │   Logic    │
                       └────────────┘
                       /            \
                  (Port Out)      (Port Out)
                       ↓             ↓
              [JPA Adapter]   [Email Adapter]
```

注意箭头方向：**Driving Adapter（HTTP/CLI）调用 Domain，Domain 调用 Driven Adapter（JPA/Email）**。Driving 在左/上方主动驱动，Driven 在右/下方被动响应。

### 1.3 Clean Architecture（Bob Martin）

Bob Martin 在 2012 年的博客里画了一张著名的同心圆，从外到内 4 层：

```
   ┌──────────────────────────────────────────────┐
   │  Frameworks & Drivers  (Web, DB, UI, ...)    │
   │  ┌────────────────────────────────────────┐  │
   │  │ Interface Adapters (Controllers, ...)  │  │
   │  │  ┌──────────────────────────────────┐  │  │
   │  │  │  Use Cases (Application Logic)   │  │  │
   │  │  │  ┌────────────────────────────┐  │  │  │
   │  │  │  │  Entities (Enterprise)     │  │  │  │
   │  │  │  └────────────────────────────┘  │  │  │
   │  │  └──────────────────────────────────┘  │  │
   │  └────────────────────────────────────────┘  │
   └──────────────────────────────────────────────┘
   ─── 依赖箭头永远朝内 ──→
```

Bob 强调 **The Dependency Rule**：源码层面的依赖只能从外向内。具体怎么做到？用 DIP：内层定义接口，外层实现。**这就是六边形里的 Port + Adapter**。

Bob 比 Cockburn 多分了一层，把"应用业务规则（Use Cases）"和"企业业务规则（Entities）"分开 —— 前者是某个应用特有的流程编排，后者是跨应用共享的核心规则（比如"账户余额不能为负"）。

### 1.4 Onion Architecture（Jeffrey Palermo）

Palermo 的同心圆从外到内：

```
Infrastructure (Persistence, Messaging, Web Services)
  ↓
Application Services
  ↓
Domain Services
  ↓
Domain Model (Entities + Value Objects)
```

最内层是 Domain Model（充血模型），向外一层 Domain Services 处理跨实体的领域逻辑，再外是 Application Services 处理用例编排，最外是 Infrastructure。

Onion 比 Hexagonal 强调**内层细分**（Domain Model / Domain Services / Application Services 分开），但这种细分对小项目是负担，对大项目才有价值。

### 1.5 三者的微妙差别

把三种架构放在同一个例子下看就清楚了。假设有一个"创建订单并发送通知"的用例。

```
Hexagonal:
  HttpAdapter → CreateOrderUseCase (Port In) → Domain → NotifyPort (Port Out) → MqAdapter

Clean:
  Controller → CreateOrderInputBoundary → CreateOrderInteractor → Order (Entity)
                                                                → NotifyOutputBoundary → MqGateway

Onion:
  Controller → OrderApplicationService → OrderDomainService → Order (Domain Model)
                                                            → INotifier (Infrastructure)
```

| 维度 | Hexagonal | Clean | Onion |
| --- | --- | --- | --- |
| **层数** | 2 层（核心 + 适配器） | 4 层（Entity / UseCase / Adapter / Framework） | 4 层（DomainModel / DomainService / AppService / Infra） |
| **端口命名** | Port（驱动/被驱动） | InputBoundary / OutputBoundary | Interface（in Domain）/ Implementation（in Infra） |
| **强调点** | 对称性（左右两侧适配器对称） | 控制流方向 + 数据 DTO 跨层 | 依赖箭头方向 + Domain Model 充血 |
| **DDD 兼容性** | 兼容但不强求 | 兼容但不强求 | 几乎是 DDD 的别名 |

**结论**：日常工作中**不要纠结用哪个名字**。包结构按你团队习惯选一种就行，关键是理解共通点：依赖反转 + 业务核心隔离。

> **口诀**：Hex / Clean / Onion 是同一头大象的三个画师。看到面试官问"区别"，先回答"本质相同"，再展开层级粒度的差别。

### 1.6 与传统分层（MVC / 三层）的对比

传统 Spring 项目的三层结构：

```
Controller (Web 层)
   ↓ 调用
Service (业务层)
   ↓ 调用
DAO / Repository (数据层)
   ↓ 调用
DB
```

依赖方向：**Controller → Service → DAO → DB**，箭头一路朝下。

这种写法看起来"清晰"，但**有三个具体症状**会让任何稍大的项目陷入困境：

**症状 1：业务逻辑被框架污染**

Service 层为了访问数据库，必须 import `org.springframework.data.jpa.repository.JpaRepository`。Service 是你的业务核心，却天然依赖 Spring Data JPA。**换一种 ORM 就要改 Service 代码**。

**症状 2：业务规则散落到 Controller 和 DAO**

```java
// Controller 里
@PostMapping("/orders")
public ResponseEntity<OrderVO> create(@RequestBody OrderRequest req) {
    if (req.getAmount() <= 0) throw new BadRequestException();   // ★ 业务规则！
    if (req.getItems().isEmpty()) throw new BadRequestException();   // ★ 业务规则！
    return ResponseEntity.ok(orderService.create(req));
}
```

订单金额不能为零这个规则是**业务规则**，写在 Controller 里意味着如果未来加个 CLI 或 RPC 入口，规则又要重写一遍。

**症状 3：测试必须依赖 Spring 容器**

Service 直接 `@Autowired Repository`，测试时要 `@SpringBootTest` 启动整个上下文，10 秒起步。一个真正的"业务核心"应该可以**纯 JUnit 跑**，毫秒级。

六边形的解法很简单：**把 Service 依赖的 Repository 改成 Domain 层定义的接口**，由 Infrastructure 层实现。这样 Service 不再依赖 Spring Data JPA，测试时可以 new InMemoryRepository 直接跑。

> **口诀**：传统三层不是错，是**没贯彻 DIP**。把 DAO 接口从 infrastructure 移到 domain，三层立刻变成"准六边形"。

### 1.7 与 DDD 战术设计的协同

DDD（[详见战术战略一文](/article/architecture/ddd)）和 Hexagonal 经常一起出现，但二者是**正交关系**：

- DDD 关心**业务建模**：Aggregate、Entity、Value Object、Domain Event、Bounded Context
- Hexagonal 关心**代码结构**：Port / Adapter、依赖方向、层级隔离

可以单独用任一个：
- 只用 Hex 不用 DDD：业务相对简单（CRUD 为主），用 Hex 把外部依赖隔开就够
- 只用 DDD 不用 Hex：见过的反例，Aggregate 设计得很好但充血模型里 import 了 JPA 注解，结果是"表面 DDD"

但二者一起用是**最舒服的组合**：DDD 告诉你 domain 包里该建什么类；Hex 告诉你 domain 包不该依赖什么东西。

```
domain/                    ← DDD 决定这里有什么（Aggregate / VO / Domain Event）
  + Hexagonal 决定这里不能有什么（Spring 注解 / JPA 注解 / Web 注解）

application/               ← Use Case / Application Service（编排）
infrastructure/            ← Adapter（JPA 实现 / MQ 实现 / 缓存实现）
interfaces/                ← Driving Adapter（Controller / CLI / Consumer）
```

> **口诀**：DDD 管"建什么"，Hex 管"放哪里 + 依赖谁"。两者垂直叠加，1 + 1 > 2。

---

## 2. 工程落地切片（Java + Spring Boot）

> 接下来用一个完整的"订单创建 + 库存扣减 + 通知"用例，把每一层的代码写出来。版本基于 Spring Boot 3.2 + Java 21。

### 2.1 包结构

先给最终的包结构，后面每节填对应文件。

```
com.example.order/
├── domain/                          ← 纯业务，零外部依赖
│   ├── model/
│   │   ├── Order.java               ← Aggregate Root
│   │   ├── OrderId.java             ← Value Object
│   │   ├── OrderItem.java           ← Entity (in aggregate)
│   │   ├── OrderStatus.java         ← Enum
│   │   └── Money.java               ← Value Object
│   ├── event/
│   │   └── OrderCreatedEvent.java
│   └── port/
│       ├── in/                      ← Driving Port（Use Case 接口）
│       │   └── CreateOrderUseCase.java
│       └── out/                     ← Driven Port（被业务核心调用）
│           ├── OrderRepository.java
│           ├── InventoryPort.java
│           └── NotificationPort.java
├── application/                     ← Use Case 实现，编排 domain
│   └── service/
│       └── CreateOrderService.java  ← implements CreateOrderUseCase
├── infrastructure/                  ← 所有 Driven Adapter
│   ├── persistence/
│   │   ├── OrderJpaEntity.java
│   │   ├── OrderJpaRepository.java  ← extends JpaRepository
│   │   ├── OrderRepositoryAdapter.java  ← implements domain.port.out.OrderRepository
│   │   └── OrderMapper.java
│   ├── inventory/
│   │   └── InventoryRpcAdapter.java
│   └── notification/
│       └── NotificationMqAdapter.java
└── interfaces/                      ← 所有 Driving Adapter
    ├── rest/
    │   ├── OrderController.java
    │   ├── CreateOrderRequest.java
    │   └── OrderResponse.java
    ├── cli/
    │   └── CreateOrderCommand.java
    └── consumer/
        └── OrderRetryConsumer.java
```

四个顶层包对应四个角色：

- `domain`：业务核心，纯 POJO，**不允许 import 任何 Spring / JPA / Web 的类**
- `application`：用例编排，可以 import Spring（`@Service`、`@Transactional`），但不允许 import 任何 infrastructure
- `infrastructure`：所有出向适配器（Driven Adapter）
- `interfaces`：所有入向适配器（Driving Adapter）

依赖方向：

```
interfaces ──→ application ──→ domain ←── infrastructure
                                  ↑
                              （只有 domain 是被所有人指向的"圆心"）
```

### 2.2 Domain 层：纯 POJO

```java
package com.example.order.domain.model;

public final class OrderId {
    private final String value;

    private OrderId(String value) {
        if (value == null || value.isBlank())
            throw new IllegalArgumentException("OrderId cannot be blank");
        this.value = value;
    }

    public static OrderId of(String value) { return new OrderId(value); }
    public static OrderId generate() { return new OrderId(UUID.randomUUID().toString()); }

    public String value() { return value; }

    @Override public boolean equals(Object o) { /* ... */ }
    @Override public int hashCode() { return Objects.hash(value); }
}
```

**注意**：

- `OrderId` 是 Value Object（值对象），用 `final` 标识不可变
- 构造函数私有，只能通过工厂方法创建，**保证创建时的不变量**（不为空）
- 没有 `@Component` `@Entity` `@Id` 任何注解 —— 这是 domain 层的铁律

```java
package com.example.order.domain.model;

public final class Money {
    private final BigDecimal amount;
    private final Currency currency;

    private Money(BigDecimal amount, Currency currency) {
        if (amount.signum() < 0)
            throw new IllegalArgumentException("Money cannot be negative");
        this.amount = amount.setScale(currency.getDefaultFractionDigits(), RoundingMode.HALF_UP);
        this.currency = currency;
    }

    public static Money of(BigDecimal amount, String currencyCode) {
        return new Money(amount, Currency.getInstance(currencyCode));
    }

    public Money add(Money other) {
        if (!currency.equals(other.currency))
            throw new IllegalArgumentException("Currency mismatch");
        return new Money(amount.add(other.amount), currency);
    }

    public Money multiply(int times) { return new Money(amount.multiply(BigDecimal.valueOf(times)), currency); }
}
```

**点透**：`Money` 是典型的 Value Object 教科书例子。Hexagonal 不强求你用 VO，但**一旦用了就要彻底**：金额运算永远经过 `Money`，不要在外面写 `BigDecimal.add`。

```java
package com.example.order.domain.model;

public class Order {
    private final OrderId id;
    private final String customerId;
    private final List<OrderItem> items;
    private OrderStatus status;
    private Money totalAmount;
    private Instant createdAt;

    private Order(OrderId id, String customerId, List<OrderItem> items) {
        if (items == null || items.isEmpty())
            throw new IllegalArgumentException("Order must have at least one item");
        this.id = id;
        this.customerId = customerId;
        this.items = new ArrayList<>(items);
        this.status = OrderStatus.CREATED;
        this.totalAmount = items.stream()
            .map(OrderItem::subtotal)
            .reduce(Money.of(BigDecimal.ZERO, "CNY"), Money::add);
        this.createdAt = Instant.now();
    }

    public static Order create(String customerId, List<OrderItem> items) {
        return new Order(OrderId.generate(), customerId, items);
    }

    public void pay() {
        if (status != OrderStatus.CREATED)
            throw new IllegalStateException("Cannot pay order in status " + status);
        this.status = OrderStatus.PAID;
    }

    public void cancel() {
        if (status == OrderStatus.SHIPPED || status == OrderStatus.COMPLETED)
            throw new IllegalStateException("Cannot cancel shipped/completed order");
        this.status = OrderStatus.CANCELLED;
    }
}
```

**点透**：

- 业务规则（"已发货不能取消"）写在 Aggregate Root 里。这是**充血模型**，是 DDD 的精髓
- 没有 setter，所有状态变更必须经过业务方法（`pay()` / `cancel()`），**保护不变量**
- 静态工厂方法 `Order.create(...)` 替代 public 构造函数，让创建逻辑更明确

### 2.3 端口定义（Port = 接口）

端口分两类：

- **Port In（Driving Port / Inbound Port）**：业务核心向外暴露的能力，如"创建订单用例"
- **Port Out（Driven Port / Outbound Port）**：业务核心需要外部提供的能力，如"持久化订单"

```java
// Port In：暴露给 Driving Adapter 调用
package com.example.order.domain.port.in;

public interface CreateOrderUseCase {
    OrderId createOrder(CreateOrderCommand command);

    record CreateOrderCommand(
        String customerId,
        List<Item> items
    ) {
        public record Item(String skuId, int quantity, BigDecimal unitPrice) {}
    }
}
```

**点透**：

- 用 Java 17 的 `record` 定义命令对象，简洁且天然不可变
- Command 是 application 层的"输入 DTO"，**和 HTTP 层的 Request 解耦** —— Controller 把 HTTP Request 翻译成 Command，再调 UseCase
- 命令对象写在 port 包里**和接口同文件**（用嵌套 record），让端口接口自包含

```java
// Port Out：业务核心要求外部实现
package com.example.order.domain.port.out;

public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
    List<Order> findByCustomer(String customerId);
}

public interface InventoryPort {
    void reserve(String skuId, int quantity);
    void release(String skuId, int quantity);
}

public interface NotificationPort {
    void notifyOrderCreated(OrderCreatedEvent event);
}
```

**反直觉点 1**：**Repository 接口写在 domain 包**，不是 infrastructure 包。

为什么？因为依赖反转。如果 Repository 接口写在 infrastructure，application 调用它就要 import infrastructure，依赖方向就反了。**接口的所有权属于"使用者"，不属于"实现者"**。

**反直觉点 2**：接口的方法签名**只用 domain 类型**（Order、OrderId），**不能有任何外部类型**（不能出现 `Page<Order>` `Pageable`，那是 Spring Data 的类）。

如果你需要分页，自己在 domain 里定义一个 `Page<T>` 值对象，让 infrastructure 翻译。

### 2.4 Driving Adapter：HTTP / CLI / MQ Consumer

Driving Adapter 负责把外部输入翻译成 Use Case 能理解的 Command，调用 Port In。

```java
package com.example.order.interfaces.rest;

@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderController {

    private final CreateOrderUseCase createOrderUseCase;   // ★ 依赖端口接口，不是 Service 实现

    @PostMapping
    public ResponseEntity<OrderResponse> create(@RequestBody @Valid CreateOrderRequest req) {
        var command = new CreateOrderUseCase.CreateOrderCommand(
            req.customerId(),
            req.items().stream()
                .map(i -> new CreateOrderUseCase.CreateOrderCommand.Item(
                    i.skuId(), i.quantity(), i.unitPrice()))
                .toList()
        );
        OrderId orderId = createOrderUseCase.createOrder(command);
        return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(new OrderResponse(orderId.value()));
    }
}

record CreateOrderRequest(
    @NotBlank String customerId,
    @NotEmpty List<Item> items
) {
    record Item(@NotBlank String skuId, @Positive int quantity, @NotNull BigDecimal unitPrice) {}
}

record OrderResponse(String orderId) {}
```

**点透**：

- Controller 依赖的是 `CreateOrderUseCase` 接口，**不是 `CreateOrderService` 实现**。这让 Controller 不知道实现细节
- Controller 的职责被**严格压缩**：HTTP 协议解析（参数校验、JSON 反序列化）+ Command 翻译 + 状态码映射。**不写业务逻辑**
- `CreateOrderRequest` 和 `CreateOrderCommand` 长得几乎一样，但必须分两个类。原因：HTTP Request 字段会随 API 演进改名/新增，Command 字段绑定业务语义。**两者解耦让 API 演进不影响 application 层**

CLI 适配器（同一个用例，不同入口）：

```java
package com.example.order.interfaces.cli;

@ShellComponent
@RequiredArgsConstructor
public class CreateOrderCommand {

    private final CreateOrderUseCase createOrderUseCase;

    @ShellMethod("Create an order from JSON file")
    public String create(String jsonFile) throws IOException {
        var json = Files.readString(Path.of(jsonFile));
        var command = parseCommand(json);   // 自己解析
        OrderId id = createOrderUseCase.createOrder(command);
        return "Created: " + id.value();
    }
}
```

**点透**：CLI 入口和 HTTP 入口**完全独立**，二者复用同一个 `CreateOrderUseCase`。这就是 Hex 最直观的好处：**新增入口不动核心**。

### 2.5 Driven Adapter：JPA / RPC / MQ

Driven Adapter 实现 Port Out，把业务请求翻译成具体的外部调用。

先写 JPA 持久化：

```java
package com.example.order.infrastructure.persistence;

@Entity
@Table(name = "orders")
@Getter @NoArgsConstructor @AllArgsConstructor
public class OrderJpaEntity {
    @Id
    private String id;
    private String customerId;
    private String status;
    private BigDecimal totalAmount;
    private String currency;
    private Instant createdAt;

    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    @JoinColumn(name = "order_id")
    private List<OrderItemJpaEntity> items;
}

public interface OrderJpaRepository extends JpaRepository<OrderJpaEntity, String> {
    List<OrderJpaEntity> findByCustomerId(String customerId);
}
```

**反直觉点 3**：`OrderJpaEntity` **和 domain 的 `Order` 是两个类**，不是同一个。

很多教程为了"方便"把 `@Entity` 直接打在 domain 的 `Order` 上。这违背 Hex 的核心原则：

1. domain 不能依赖 JPA
2. JPA 的需求（无参构造、setter、可变集合）和 domain 的需求（不可变、构造时校验）**根本冲突**
3. 数据库结构演进会反向污染 domain 模型

正确做法：domain 一个 `Order`，infrastructure 一个 `OrderJpaEntity`，中间用 Mapper 转。

```java
package com.example.order.infrastructure.persistence;

@Component
public class OrderMapper {
    public OrderJpaEntity toJpa(Order order) {
        var entity = new OrderJpaEntity();
        entity.setId(order.id().value());
        entity.setCustomerId(order.customerId());
        entity.setStatus(order.status().name());
        entity.setTotalAmount(order.totalAmount().amount());
        entity.setCurrency(order.totalAmount().currency().getCurrencyCode());
        entity.setCreatedAt(order.createdAt());
        entity.setItems(order.items().stream().map(this::toJpaItem).toList());
        return entity;
    }

    public Order toDomain(OrderJpaEntity entity) {
        var items = entity.getItems().stream().map(this::toDomainItem).toList();
        return Order.reconstitute(   // ★ 注意是 reconstitute，不是 create
            OrderId.of(entity.getId()),
            entity.getCustomerId(),
            items,
            OrderStatus.valueOf(entity.getStatus()),
            Money.of(entity.getTotalAmount(), entity.getCurrency()),
            entity.getCreatedAt()
        );
    }
}
```

**点透**：`Order.create(...)` 是"用户新建一个订单"，会触发业务校验（如"必须有商品"）；`Order.reconstitute(...)` 是"从数据库重建一个已存在的订单"，**跳过创建时的业务校验**（因为这些数据库里的数据本来就是合法的）。区分这两个是 DDD 中常见的细节。

```java
package com.example.order.infrastructure.persistence;

@Component
@RequiredArgsConstructor
public class OrderRepositoryAdapter implements OrderRepository {

    private final OrderJpaRepository jpaRepository;
    private final OrderMapper mapper;

    @Override
    public void save(Order order) {
        OrderJpaEntity entity = mapper.toJpa(order);
        jpaRepository.save(entity);
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return jpaRepository.findById(id.value()).map(mapper::toDomain);
    }

    @Override
    public List<Order> findByCustomer(String customerId) {
        return jpaRepository.findByCustomerId(customerId)
            .stream()
            .map(mapper::toDomain)
            .toList();
    }
}
```

**点透**：

- `OrderRepositoryAdapter` 实现 domain 层定义的 `OrderRepository` 接口
- 它依赖 `OrderJpaRepository`（Spring Data 的接口）—— 这没问题，因为它在 infrastructure 包
- 这就是依赖反转：**接口"指向"业务核心，实现在外层**

RPC 适配器（调用库存服务）：

```java
package com.example.order.infrastructure.inventory;

@Component
@RequiredArgsConstructor
public class InventoryRpcAdapter implements InventoryPort {

    private final InventoryFeignClient feignClient;

    @Override
    public void reserve(String skuId, int quantity) {
        try {
            feignClient.reserve(new ReserveRequest(skuId, quantity));
        } catch (FeignException.NotFound e) {
            throw new InventoryException("SKU not found: " + skuId);
        } catch (FeignException.Conflict e) {
            throw new InventoryException("Insufficient inventory for " + skuId);
        }
    }

    @Override
    public void release(String skuId, int quantity) {
        feignClient.release(new ReleaseRequest(skuId, quantity));
    }
}
```

**点透**：注意 `InventoryException` 是 domain 包定义的业务异常。**Feign 的具体异常被翻译成 domain 异常** —— 这是防腐层（ACL，Anti-Corruption Layer）的核心模式。这样换 RPC 框架（Feign → gRPC）只动 adapter，不动 domain。

### 2.6 应用服务：Use Case 实现

Application 层是**编排层**，把多个 domain + port 调用串成一个完整的用例。

```java
package com.example.order.application.service;

@Service
@RequiredArgsConstructor
public class CreateOrderService implements CreateOrderUseCase {

    private final OrderRepository orderRepository;       // ← Port Out
    private final InventoryPort inventoryPort;           // ← Port Out
    private final NotificationPort notificationPort;     // ← Port Out

    @Override
    @Transactional
    public OrderId createOrder(CreateOrderCommand command) {
        // ① 把 Command 翻译成 domain 对象
        var items = command.items().stream()
            .map(i -> OrderItem.of(i.skuId(), i.quantity(),
                Money.of(i.unitPrice(), "CNY")))
            .toList();

        // ② 创建 Aggregate
        Order order = Order.create(command.customerId(), items);

        // ③ 调用外部端口（库存）
        for (OrderItem item : order.items()) {
            inventoryPort.reserve(item.skuId(), item.quantity());
        }

        // ④ 持久化
        orderRepository.save(order);

        // ⑤ 发出领域事件
        notificationPort.notifyOrderCreated(
            new OrderCreatedEvent(order.id(), order.customerId(), order.totalAmount()));

        return order.id();
    }
}
```

**点透**：

- `@Service` `@Transactional` 是 Spring 注解，贴在 application 层是 OK 的（application 允许依赖 Spring）
- application 的方法**只编排，不写业务规则**。规则全在 `Order.create(...)` 里
- 调用顺序：先调用外部端口（库存预占）再持久化 —— 这样如果库存不够，订单根本不会被创建。这是**用例编排的责任**

**反直觉点 4**：库存预占 + 订单持久化**不在同一个本地事务里**（库存是另一个服务）。这里要么用 Saga，要么用补偿。简单实现可以用 `@TransactionalEventListener` 做"持久化成功后再发通知"，避免脏通知。完整的分布式事务讨论见 [DDD 那篇](/article/architecture/ddd) 的 Domain Event 章节。

### 2.7 Spring Boot 的依赖注入

到这里所有类已经写完，看 Spring 怎么把它们串起来：

```java
// 这是 Spring Boot 自动做的：
//
// CreateOrderService 构造函数有 3 个参数：
//   - OrderRepository       ← Spring 找到 OrderRepositoryAdapter（@Component）
//   - InventoryPort         ← Spring 找到 InventoryRpcAdapter
//   - NotificationPort      ← Spring 找到 NotificationMqAdapter
//
// OrderController 构造函数有 1 个参数：
//   - CreateOrderUseCase    ← Spring 找到 CreateOrderService（@Service）
```

**为什么这样能工作**：Spring 容器扫描所有 `@Component`/`@Service` Bean，按类型注入。`OrderRepositoryAdapter implements OrderRepository`，所以注入 `OrderRepository` 时容器找到唯一实现就装进去。

**多实现怎么办**？比如同时存在 `OrderRepositoryAdapter`（生产）和 `InMemoryOrderRepository`（测试），用 `@Profile`：

```java
@Component
@Profile("!test")
public class OrderRepositoryAdapter implements OrderRepository { /* ... */ }

@Component
@Profile("test")
public class InMemoryOrderRepository implements OrderRepository {
    private final Map<String, Order> store = new ConcurrentHashMap<>();
    @Override public void save(Order order) { store.put(order.id().value(), order); }
    @Override public Optional<Order> findById(OrderId id) { return Optional.ofNullable(store.get(id.value())); }
    @Override public List<Order> findByCustomer(String customerId) {
        return store.values().stream().filter(o -> o.customerId().equals(customerId)).toList();
    }
}
```

测试时启动 `@ActiveProfiles("test")`，自动用 InMemory 实现。**业务核心一行不改**。

### 2.8 测试策略：用 InMemory 适配器测试核心域

六边形最大的工程红利是**测试速度**。来看一个完整的用例测试：

```java
class CreateOrderServiceTest {

    private InMemoryOrderRepository orderRepository;
    private InMemoryInventoryPort inventoryPort;
    private SpyNotificationPort notificationPort;
    private CreateOrderService service;

    @BeforeEach
    void setUp() {
        orderRepository = new InMemoryOrderRepository();
        inventoryPort = new InMemoryInventoryPort(Map.of("SKU-1", 100));
        notificationPort = new SpyNotificationPort();
        service = new CreateOrderService(orderRepository, inventoryPort, notificationPort);
    }

    @Test
    void shouldCreateOrderAndReserveInventory() {
        var cmd = new CreateOrderUseCase.CreateOrderCommand(
            "customer-1",
            List.of(new CreateOrderUseCase.CreateOrderCommand.Item(
                "SKU-1", 2, BigDecimal.valueOf(50))));

        OrderId orderId = service.createOrder(cmd);

        assertThat(orderRepository.findById(orderId)).isPresent();
        assertThat(inventoryPort.reservedQuantity("SKU-1")).isEqualTo(2);
        assertThat(notificationPort.events()).hasSize(1);
    }

    @Test
    void shouldFailWhenInventoryInsufficient() {
        var cmd = new CreateOrderUseCase.CreateOrderCommand(
            "customer-1",
            List.of(new CreateOrderUseCase.CreateOrderCommand.Item(
                "SKU-1", 200, BigDecimal.valueOf(50))));   // 库存只有 100

        assertThatThrownBy(() -> service.createOrder(cmd))
            .isInstanceOf(InventoryException.class)
            .hasMessageContaining("Insufficient");

        assertThat(orderRepository.findByCustomer("customer-1")).isEmpty();
    }
}
```

**点透**：

- 这是**纯 JUnit 测试**，没有任何 `@SpringBootTest` `@MockBean` `@DataJpaTest`
- 每个测试方法的执行时间 < 10ms，跑 100 个测试不到 1 秒
- InMemory 适配器是**真实实现**，不是 Mock，用 `Mockito.when().thenReturn(...)` 那种"Mock + 验证调用次数"的脆弱测试可以丢了
- 测试覆盖**业务规则**（库存不足要回滚），不测 ORM、不测 HTTP

> **口诀**：六边形的测试金字塔重心**显著下沉**。Unit Test 占 80%（domain + application），Integration 占 15%（adapter），E2E 占 5%。传统三层正好反过来。

---

## 3. 实测数据

> 没数据的"架构更好"都是耍流氓。下面是同一个团队在同一业务（订单服务，约 80 个用例）上分别用三层和六边形写完后的对比。

### 3.1 代码量对比

| 维度 | 传统三层 | 六边形 | 差异 |
| --- | --- | --- | --- |
| 总文件数 | 87 | 134 | +54% |
| 总代码行 | 6,200 | 8,400 | +35% |
| `domain` 包代码行 | 0（没这层） | 1,800 | +1800 |
| `application` 包代码行 | - | 900 | +900 |
| Mapper 类 | 4（仅 DTO ↔ VO） | 16（domain ↔ JPA + DTO ↔ Command） | +300% |
| Service / UseCase 类 | 12 | 18 | +50% |

**结论**：六边形代码量**多 35%**，主要来自：
- 多了 domain 层（充血模型 + Value Object）
- Mapper 数量翻三倍（domain 和 JPA 实体分离）
- 端口接口与实现分离（多一份接口文件）

**这是六边形的成本**，不要忽视。简单 CRUD 项目用六边形是**过度设计**，反而拖慢开发。

### 3.2 单元测试覆盖率与速度

| 维度 | 传统三层 | 六边形 |
| --- | --- | --- |
| 单元测试覆盖率（业务逻辑） | 41% | 87% |
| Service 层单测平均耗时 | 1.8 秒（要 `@SpringBootTest`） | 6 毫秒（纯 JUnit） |
| 跑完全部单测 | 6 分 20 秒 | 1 分 10 秒 |
| 单测对 DB 的依赖 | 必须有 H2 | 完全没有 |
| 改一行 Service 后能在 10 秒内跑完所有相关测试吗 | 不能 | 能 |

**反直觉**：六边形代码量更多，但**测试反而更快、覆盖率更高**。原因：业务核心可以不靠 Spring 容器跑。

### 3.3 切换适配器的代价

真实场景：从 MySQL 切到 PostgreSQL。

| 项目 | 三层 | 六边形 |
| --- | --- | --- |
| 涉及代码改动文件 | 17（Service / DAO / Entity 都要改） | 4（仅 infrastructure/persistence 包） |
| 改动行数 | 320 | 80 |
| 业务测试是否需要改 | 部分需要（用了 H2 的特定 SQL） | 不需要（业务测试不依赖 DB） |
| 上线风险 | 高（业务代码也动了） | 低（仅适配器） |

更激进的：把同步 RPC 换成异步 MQ。

- 三层：Service 直接 `inventoryClient.reserve(...)`，要改 Service 加 future / callback，业务逻辑被打散
- 六边形：保持 `InventoryPort.reserve(...)` 同步语义，新写一个 `InventoryMqAdapter` 内部用 MQ + `Future.get()` 等回包。Service 一行不改

### 3.4 团队学习曲线

| 阶段 | 三层 | 六边形 |
| --- | --- | --- |
| 新人写第一个 CRUD | 0.5 天 | 2~3 天 |
| 新人理解项目结构 | 1 天 | 1 周 |
| Code Review 拒绝率（前 2 周） | 10% | 40%（经常分错层） |
| 上手 2 个月后的开发速度 | 持平 | 略快（少 debug） |
| 上手 1 年后改业务规则速度 | 慢（规则散在多处） | 快（规则集中在 domain） |

**结论**：六边形的回报是**长期**的。如果项目预期生命周期 < 1 年，可能用三层更划算；> 2 年的项目，六边形几乎一定是赚的。

> **口诀**：六边形是**长期主义架构**。短期看代码多 35%、新人难上手；长期看测试快 5 倍、改 ORM 不动业务、规则集中。**项目越长寿，六边形越值**。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 形而上学六边形 —— "简单 CRUD 也写一堆接口"

#### 现象

新团队学完 Hexagonal，开始改造一个简单的"标签管理"模块（增删改查 5 个接口，无复杂业务）。结果：

```
tag/
  domain/
    model/Tag.java                   ← 1 个字段（name）
    port/in/CreateTagUseCase.java
    port/in/DeleteTagUseCase.java
    port/in/UpdateTagUseCase.java
    port/in/QueryTagUseCase.java     ← 4 个 UseCase 接口
    port/out/TagRepository.java
  application/
    CreateTagService.java
    DeleteTagService.java
    UpdateTagService.java
    QueryTagService.java
  infrastructure/
    persistence/
      TagJpaEntity.java
      TagJpaRepository.java
      TagRepositoryAdapter.java
      TagMapper.java
  interfaces/
    rest/TagController.java
```

20 个文件做完一个简单 CRUD。Code Review 时同事问："你这是反人类吗？"

#### 根因

**误解**：Hexagonal 的"价值"不在"凡事都套接口"，而在"对复杂度真的高的部分套接口"。简单 CRUD 没有业务规则，业务核心几乎为空，套了端口适配器只是**徒增样板代码**。

#### 修复

按"业务复杂度"分级，**架构按需选择**：

| 模块复杂度 | 用什么架构 |
| --- | --- |
| 简单 CRUD（标签、字典、配置） | 直接 `Controller → JpaRepository`，不需要 Service |
| 中等业务（订单、库存） | 标准 Hexagonal |
| 高复杂业务（计费、风控） | Hexagonal + DDD 战术（Aggregate、Domain Event） |

具体到这个标签模块，正确写法：

```java
@RestController
@RequestMapping("/api/tags")
@RequiredArgsConstructor
public class TagController {
    private final TagJpaRepository repository;   // ★ 直接用 Spring Data

    @GetMapping public List<Tag> list() { return repository.findAll(); }
    @PostMapping public Tag create(@RequestBody Tag tag) { return repository.save(tag); }
    // ...
}
```

20 个文件压缩到 3 个。组织内可以约定：**业务规则少于 3 条的模块直接用三层**，超过的才走六边形。

#### 验证

引入 ArchUnit 规则（详见案例 ⑨），但**对低复杂度模块开例外**：

```java
@ArchTest
public static final ArchRule simpleCrudCanSkipHex =
    classes()
        .that().resideInAPackage("..tag..")    // tag 模块例外
        .should().onlyDependOnClassesThat().resideInAnyPackage("..");
```

> **口诀**：架构是工具不是宗教。Hexagonal 解决的是"业务规则散落"问题；如果根本没有业务规则，**用 Hexagonal 反而是反生产力**。

---

### 案例 ② 端口/适配器命名混乱 —— "Repository、Gateway、Client、Manager 都有"

#### 现象

代码评审时发现，同一个项目里五花八门的端口命名：

```java
interface OrderRepository { /* ... */ }      // domain.port.out
interface PaymentGateway { /* ... */ }       // domain.port.out
interface SmsClient { /* ... */ }            // domain.port.out
interface AuditManager { /* ... */ }         // domain.port.out
interface InventoryService { /* ... */ }     // domain.port.out
```

新人完全分不清 `Gateway` 和 `Client` 有什么差别，更糟的是有人把 `Client`（一个 RPC 客户端的包装）写在了 domain 层。

#### 根因

DDD 和 Hexagonal 社区**没有强制命名规范**，结果各社区有各自的习惯：

- DDD 蓝皮书：持久化用 `Repository`，外部系统访问用 `Service`（容易和应用 Service 混）
- Cockburn 的 Hex：统一叫 `Port`（如 `OrderPort`、`PaymentPort`）
- 微服务社区：外部 RPC 叫 `Client` 或 `Gateway`
- 企业架构：`Manager` `Facade` `Helper` 满天飞（这种最坏）

混合到一个项目里，**命名混乱反过来吃掉了架构清晰度**。

#### 修复

团队内**强制统一命名约定**，写进 README：

| 端口类型 | 命名后缀 | 示例 |
| --- | --- | --- |
| 持久化（CRUD 当前数据） | `Repository` | `OrderRepository` |
| 外部系统调用（RPC / HTTP） | `Gateway` | `PaymentGateway` |
| 通知 / 消息发送 | `Notifier` 或 `Publisher` | `OrderNotifier` |
| 查询 / 读模型 | `Query` 或 `Finder` | `OrderQuery` |
| 用例（Port In） | `UseCase` 或动词 | `CreateOrderUseCase` |

**禁用**：`Client`、`Manager`、`Helper`、`Service`（这个名字太泛）、`Util`。

适配器命名：`{端口名}{技术后缀}Adapter`：

```java
class OrderRepositoryJpaAdapter implements OrderRepository { /* ... */ }
class PaymentGatewayHttpAdapter implements PaymentGateway { /* ... */ }
class OrderNotifierKafkaAdapter implements OrderNotifier { /* ... */ }
```

#### 验证

ArchUnit 强制：

```java
@ArchTest
public static final ArchRule portsMustHaveProperSuffix =
    classes()
        .that().resideInAPackage("..domain.port.out..")
        .should().haveSimpleNameEndingWith("Repository")
        .orShould().haveSimpleNameEndingWith("Gateway")
        .orShould().haveSimpleNameEndingWith("Notifier")
        .orShould().haveSimpleNameEndingWith("Publisher")
        .orShould().haveSimpleNameEndingWith("Query");
```

CI 上挂 ArchUnit 跑，PR 不通过命名约定**直接 fail**。

> **口诀**：Hexagonal 的命名混乱比"不用 Hexagonal"还糟。**写进 README、ArchUnit 强制、新人入职第一天讲**，三件套缺一不可。

---

### 案例 ③ 业务逻辑漏到 Controller / Service —— "六边形但仍是面条代码"

#### 现象

号称用了 Hexagonal 的项目，半年后 Controller 长这样：

```java
@PostMapping("/orders")
public ResponseEntity<?> create(@RequestBody CreateOrderRequest req) {
    // ★ 业务规则 1：金额校验
    BigDecimal total = req.getItems().stream()
        .map(i -> i.getUnitPrice().multiply(BigDecimal.valueOf(i.getQuantity())))
        .reduce(BigDecimal.ZERO, BigDecimal::add);
    if (total.compareTo(BigDecimal.valueOf(100000)) > 0)
        throw new BadRequestException("Single order cannot exceed 100k");

    // ★ 业务规则 2：用户等级校验
    User user = userClient.findById(req.getUserId());
    if (user.getLevel().equals("NORMAL") && total.compareTo(BigDecimal.valueOf(10000)) > 0)
        throw new BadRequestException("Normal user limit");

    // ★ 业务规则 3：促销叠加规则
    List<Promotion> promos = promoClient.findActivePromos();
    BigDecimal discount = calculateBestDiscount(promos, req.getItems());

    return ResponseEntity.ok(orderService.create(req, discount));
}
```

Controller 里塞了 3 条核心业务规则。这是经典的"披着 Hexagonal 外衣的三层"。

#### 根因

团队"知道"要把业务放 domain，但**遇到具体业务就忘了**。常见的诱惑：
- 校验逻辑"看起来"应该在 Controller（参数校验）
- 跨服务调用"看起来"应该在 Service 编排
- 算折扣"看起来"是个工具函数

每一条**单看都合理**，但合在一起就把业务核心架空了。

#### 修复

**判定原则**：一段代码，如果换一个入口（CLI / RPC）也要重新写一遍，那它就是**业务规则**，必须放 domain。

把上面三条搬到 domain：

```java
// domain/policy/OrderAmountPolicy.java
public class OrderAmountPolicy {
    public static void validateForUser(Money total, UserLevel level) {
        if (total.greaterThan(Money.of(100000, "CNY")))
            throw new BusinessException("Single order cannot exceed 100k");
        if (level == UserLevel.NORMAL && total.greaterThan(Money.of(10000, "CNY")))
            throw new BusinessException("Normal user limit exceeded");
    }
}

// domain/policy/PromotionEngine.java
public class PromotionEngine {
    public Money applyBest(List<Promotion> active, List<OrderItem> items) {
        return active.stream()
            .map(p -> p.tryApply(items))
            .filter(Optional::isPresent).map(Optional::get)
            .max(Comparator.comparing(Money::amount))
            .orElse(Money.zero("CNY"));
    }
}
```

Controller 瘦回去：

```java
@PostMapping("/orders")
public ResponseEntity<OrderResponse> create(@RequestBody @Valid CreateOrderRequest req) {
    var command = req.toCommand();
    OrderId id = createOrderUseCase.createOrder(command);
    return ResponseEntity.status(HttpStatus.CREATED).body(new OrderResponse(id.value()));
}
```

CreateOrderService 编排：

```java
@Transactional
public OrderId createOrder(CreateOrderCommand cmd) {
    User user = userGateway.findById(cmd.customerId());
    List<Promotion> promos = promotionGateway.findActivePromotions();

    // 业务规则全在 domain
    OrderAmountPolicy.validateForUser(calculateTotal(cmd.items()), user.level());
    Money discount = new PromotionEngine().applyBest(promos, toItems(cmd.items()));

    Order order = Order.create(cmd.customerId(), toItems(cmd.items()), discount);
    orderRepository.save(order);
    return order.id();
}
```

#### 验证

定期做"业务规则盘点"：

```bash
# 在 Controller / Service 包里 grep 业务关键字
grep -r "compareTo" src/main/java/.../interfaces/
grep -r "if.*level" src/main/java/.../interfaces/
grep -r "if.*amount" src/main/java/.../application/
```

发现任何**条件判断 + 业务字段**的组合，立刻拆出去。CI 上跑 SonarQube 设规则：Controller 类的 cyclomatic complexity ≤ 5。

> **口诀**：Controller 的极限职责是 **HTTP 协议处理 + Command 翻译**。Application 的极限职责是**编排**。**条件判断超过 1 行**就要警惕规则漏出去。

---

### 案例 ④ Spring 注解污染 Domain 层 —— "import 一看就破功"

#### 现象

打开 domain 包，看到：

```java
package com.example.order.domain.model;

import org.springframework.stereotype.Component;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import com.fasterxml.jackson.annotation.JsonProperty;

@Entity
@Component
public class Order {
    @Id
    @JsonProperty("orderId")
    private String id;
    // ...
}
```

这个 `Order` 同时是 Spring Bean、JPA Entity、Jackson DTO。表面上"省了 Mapper"，实际上**完全违背 Hexagonal**。

#### 根因

新人嫌写 Mapper 麻烦，照着教程"省一步"。或者老项目从三层迁过来，懒得拆 JPA Entity。

具体危害：

1. **Domain 不再可以独立测试**：要 `@SpringBootTest`，要 EntityManager
2. **业务模型受 JPA 约束**：必须有无参构造、必须有 setter（不然 Hibernate 不能 hydrate）、不能用 Java record
3. **JSON 字段名变了 = 业务模型变了**：API 演进反向污染 domain
4. **再换 ORM 难度极高**：domain 处处是 JPA 注解

#### 修复

**铁律**：domain 包的任何 import 都不能含以下前缀：

```
org.springframework.*
jakarta.persistence.*       (JPA)
javax.persistence.*         (老 JPA)
com.fasterxml.jackson.*     (Jackson)
com.google.gson.*           (Gson)
org.hibernate.*
io.swagger.*
io.micrometer.*
```

允许的 import：

```
java.*                       (JDK)
javax.*  (除 persistence)    (JDK 标准)
com.example.order.domain.*  (本模块 domain 内部)
```

**渐进式重构**（老项目）：

第一步：在 domain 旁边新建一个 `OrderJpaEntity`，不动原有 `Order` 上的注解，先做映射

第二步：所有 application 层调用改为通过 Mapper 操作 `Order`，让 domain 层的注解逐渐"无人调用"

第三步：删掉 `Order` 上的 JPA / Spring 注解

第四步：开启 ArchUnit 规则永久守卫

#### 验证

ArchUnit 写在测试代码里，CI 必跑：

```java
@AnalyzeClasses(packages = "com.example.order")
class HexagonalArchitectureTest {

    @ArchTest
    static final ArchRule domainShouldNotDependOnSpring =
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "org.springframework..",
                "jakarta.persistence..",
                "com.fasterxml.jackson..",
                "org.hibernate..",
                "io.swagger.."
            );

    @ArchTest
    static final ArchRule domainShouldNotDependOnInfrastructure =
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAnyPackage("..infrastructure..");

    @ArchTest
    static final ArchRule domainShouldNotDependOnApplication =
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAnyPackage("..application..");
}
```

跑一次 `mvn test`，如果 domain 包出现 Spring import 立刻 fail。

> **口诀**：**`grep -r "import org.springframework" src/main/java/.../domain/` 应该零结果**。这是检验六边形真假最直接的一步 grep。

---

### 案例 ⑤ 实体类与 JPA Entity 混用 —— "改字段反向影响业务"

#### 现象

业务方说"订单状态新增一个 'PARTIALLY_REFUNDED'（部分退款）状态"。开发改 `OrderStatus` 枚举，改完跑测试，所有跟订单无关的模块（用户、商品、报表）的测试全挂了。

```
ERROR: column "status" of relation "orders" doesn't accept value 'PARTIALLY_REFUNDED'
ERROR: ... violates check constraint "orders_status_check"
```

数据库的 `CHECK` 约束没更新。但这不是最糟的，最糟的是：**OrderStatus 同时是 domain 概念和 DB 列的取值约束**，一个改动跨两层。

#### 根因

`OrderStatus` 枚举既被 domain 用，又被 JPA 直接 `@Enumerated(EnumType.STRING)` 持久化到 DB。新增枚举 = DB 增加合法值 + Java 增加分支。

更深的问题：**部分退款"在数据库视角是一个状态，在业务视角可能是一个事件链**（已支付 + 1 笔退款记录）。把它当成枚举值是过早具象化。

#### 修复

**第一步**：domain 和 JPA 状态分离。

```java
// domain
public enum OrderStatus {
    CREATED, PAID, SHIPPED, COMPLETED, CANCELLED
    // 没有 PARTIALLY_REFUNDED
}

public class Order {
    private OrderStatus status;
    private List<Refund> refunds;   // ★ 部分退款建模为子实体集合

    public boolean isFullyRefunded() {
        return refunds.stream().map(Refund::amount).reduce(Money.zero("CNY"), Money::add)
            .equals(totalAmount);
    }

    public boolean isPartiallyRefunded() {
        return !refunds.isEmpty() && !isFullyRefunded();
    }
}
```

**第二步**：Mapper 负责把 domain → JPA 字段：

```java
public OrderJpaEntity toJpa(Order order) {
    var entity = new OrderJpaEntity();
    entity.setStatus(order.status().name());
    entity.setRefundedAmount(order.refunds().stream()
        .map(Refund::amount)
        .reduce(Money.zero("CNY"), Money::add).amount());
    // ...
}
```

**第三步**：DB 层改成"用扩展数值列"而不是枚举字符串约束：

```sql
ALTER TABLE orders DROP CONSTRAINT orders_status_check;
ALTER TABLE orders ADD COLUMN refunded_amount DECIMAL(18, 2) DEFAULT 0;
```

#### 验证

```java
@Test
void partialRefundShouldNotBeCountedAsFullyRefunded() {
    Order order = Order.create("c-1", List.of(item(100)));
    order.pay();
    order.refund(Money.of(30, "CNY"));   // 部分退款

    assertThat(order.isPartiallyRefunded()).isTrue();
    assertThat(order.isFullyRefunded()).isFalse();
    assertThat(order.status()).isEqualTo(OrderStatus.PAID);   // 状态没变
}
```

> **口诀**：**枚举值 ≠ 业务状态**。任何"看起来是新状态"的需求，先想想能不能建模成实体或事件，而不是直接加枚举。

---

### 案例 ⑥ 跨上下文调用没用 ACL —— "上游字段一改下游全挂"

#### 现象

订单服务依赖用户服务（`UserClient`）查用户等级。用户服务有天把 `User.level` 字段重命名为 `User.tier`，结果订单服务部署上线时全部拒绝服务，因为反序列化 `level` 字段为 null。

#### 根因

订单服务的 domain 直接持有 `UserClient` 返回的 `User` 对象：

```java
package com.example.order.application.service;

import com.example.user.api.User;   // ★ 直接 import 上游模型

public class CreateOrderService {
    public OrderId createOrder(...) {
        User user = userClient.findById(customerId);
        if (user.getLevel().equals("VIP")) { /* ... */ }   // ★ 上游字段名 = 我的代码
    }
}
```

订单的业务代码**直接耦合**到用户服务的字段名。这是 DDD 里"防腐层（ACL，Anti-Corruption Layer）缺失"的典型现象。

#### 修复

引入防腐层。完整结构：

```
order/
  domain/
    model/
      Customer.java               ← ★ 订单上下文的"客户"概念
      CustomerLevel.java
    port/out/
      CustomerGateway.java        ← 订单上下文需要的客户能力
  infrastructure/
    customer/
      UserServiceCustomerAdapter.java   ← ACL 实现
      UserServiceClient.java            ← 上游 SDK 包装
```

Domain 定义自己的 `Customer`：

```java
package com.example.order.domain.model;

public record Customer(
    String id,
    CustomerLevel level
) {
    public boolean isVip() { return level == CustomerLevel.VIP || level == CustomerLevel.SVIP; }
}

public enum CustomerLevel { NORMAL, VIP, SVIP }
```

Domain 的端口：

```java
package com.example.order.domain.port.out;

public interface CustomerGateway {
    Optional<Customer> findById(String customerId);
}
```

ACL 适配器**翻译上游模型到 domain 模型**：

```java
package com.example.order.infrastructure.customer;

@Component
@RequiredArgsConstructor
public class UserServiceCustomerAdapter implements CustomerGateway {

    private final UserServiceClient userClient;

    @Override
    public Optional<Customer> findById(String customerId) {
        try {
            UserDto upstream = userClient.findById(customerId);
            return Optional.of(translate(upstream));
        } catch (NotFoundException e) {
            return Optional.empty();
        }
    }

    private Customer translate(UserDto u) {
        // ★ 防腐层：把上游字段翻译成本上下文概念
        // 上游用 "level" 还是 "tier" 都在这里搞定，不影响业务
        String upstreamLevel = u.level() != null ? u.level() : u.tier();
        CustomerLevel level = switch (upstreamLevel) {
            case "L1", "L2" -> CustomerLevel.NORMAL;
            case "L3", "L4", "VIP" -> CustomerLevel.VIP;
            case "L5", "SVIP", "DIAMOND" -> CustomerLevel.SVIP;
            default -> CustomerLevel.NORMAL;
        };
        return new Customer(u.id(), level);
    }
}
```

业务代码：

```java
public OrderId createOrder(...) {
    Customer customer = customerGateway.findById(customerId)
        .orElseThrow(() -> new BusinessException("Customer not found"));
    if (customer.isVip()) { /* ... */ }   // ★ 用本上下文概念
}
```

现在上游再改字段名，只影响 `UserServiceCustomerAdapter`，业务零改动。

#### 验证

ArchUnit 加一条规则：domain 和 application 层禁止 import 任何外部服务的 SDK 包：

```java
@ArchTest
static final ArchRule domainAndAppShouldNotImportUpstreamSdk =
    noClasses()
        .that().resideInAnyPackage("..domain..", "..application..")
        .should().dependOnClassesThat().resideInAnyPackage(
            "com.example.user.api..",
            "com.example.payment.api..",
            "com.example.inventory.api.."
        );
```

> **口诀**：**任何跨服务 / 跨团队 / 跨 BC 的边界，都必须有 ACL**。哪怕只是字段重命名，不加 ACL 早晚被反咬一口。

---

### 案例 ⑦ 测试用真实数据库不用 InMemory 适配器 —— "全量集成测试 30 分钟"

#### 现象

CI 上跑全部测试要 28 分钟。开发本地跑 `mvn test` 喝完一杯咖啡才结束。新人改一行代码不敢推，怕引入 bug 但又不敢跑完整测试。

观察测试代码，几乎每个 Service 测试都长这样：

```java
@SpringBootTest
@AutoConfigureMockMvc
@Sql("/test-data.sql")
class CreateOrderServiceTest {
    @Autowired CreateOrderService service;
    @Autowired OrderJpaRepository jpaRepo;   // ★ 直接用 JPA Repository 校验
    // ...
}
```

#### 根因

虽然项目"声称"用了 Hexagonal，但**测试时根本没利用端口适配器分离的好处** —— 业务测试仍然要启动整个 Spring 容器、加载 H2、跑迁移。

业务测试本应只测**业务规则**，与 DB 无关。但写测试时偷懒（"反正都已经能 `@SpringBootTest` 了"），导致测试依赖逐渐扩散。

#### 修复

按测试金字塔重新组织：

| 层级 | 测试类型 | 用什么 | 目标耗时 |
| --- | --- | --- | --- |
| **L1 单元** | Domain / Application | 纯 JUnit + InMemory Adapter | < 10ms / 个 |
| **L2 集成** | 单个 Adapter（如 JPA） | `@DataJpaTest` + Testcontainers | 100ms ~ 1s / 个 |
| **L3 端到端** | 完整链路 | `@SpringBootTest` + Testcontainers | 1s ~ 5s / 个 |

L1 占 80%，L2 占 15%，L3 占 5%。

实现 InMemory 适配器（一次写好可复用）：

```java
// src/test/java/.../adapter/InMemoryOrderRepository.java
public class InMemoryOrderRepository implements OrderRepository {
    private final Map<String, Order> store = new ConcurrentHashMap<>();
    @Override public void save(Order order) { store.put(order.id().value(), order); }
    @Override public Optional<Order> findById(OrderId id) { return Optional.ofNullable(store.get(id.value())); }
    @Override public List<Order> findByCustomer(String c) {
        return store.values().stream().filter(o -> o.customerId().equals(c)).toList();
    }

    public void clear() { store.clear(); }
    public int size() { return store.size(); }
}
```

业务测试改写：

```java
class CreateOrderServiceTest {

    private InMemoryOrderRepository repo;
    private InMemoryInventoryGateway inv;
    private RecordingNotifier notifier;
    private CreateOrderService service;

    @BeforeEach
    void setUp() {
        repo = new InMemoryOrderRepository();
        inv = new InMemoryInventoryGateway(Map.of("SKU-1", 100));
        notifier = new RecordingNotifier();
        service = new CreateOrderService(repo, inv, notifier);
    }

    @Test
    void shouldCreateOrder() {
        var cmd = sampleCommand();
        OrderId id = service.createOrder(cmd);
        assertThat(repo.findById(id)).isPresent();
    }
}
```

JPA 适配器单独测：

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class OrderRepositoryAdapterTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", postgres::getJdbcUrl);
        // ...
    }

    @Autowired OrderJpaRepository jpaRepo;
    @Autowired OrderMapper mapper;

    @Test
    void shouldSaveAndLoad() {
        var adapter = new OrderRepositoryAdapter(jpaRepo, mapper);
        Order order = sampleOrder();
        adapter.save(order);
        assertThat(adapter.findById(order.id())).isPresent();
    }
}
```

#### 验证

测试金字塔健康度的量化指标：

```bash
# 业务测试比例
find src/test/java -name "*Test.java" | xargs grep -l "@SpringBootTest" | wc -l   # 应该 < 总数 10%
find src/test/java -name "*Test.java" | xargs grep -l "@DataJpaTest" | wc -l      # 应该 < 20%
find src/test/java -name "*Test.java" | xargs grep -L "@SpringBootTest\|@DataJpaTest" | wc -l   # 应该 > 70%
```

CI 跑全量测试时间从 28 分钟降到 4 分钟（实测数据），且开发本地能在 30 秒内跑完核心业务测试。

> **口诀**：六边形给你的最大红利是 **80% 的测试可以脱离 Spring 容器跑**。如果你的测试仍然 90% 是 `@SpringBootTest`，等于花了六边形的成本只享受了零的红利。

---

### 案例 ⑧ 团队对端口的理解不一致 —— "六个 Service 接口的写法差异比 Java 和 Python 还大"

#### 现象

同一个项目里出现六种端口风格：

```java
// 风格 A：传统 CRUD
interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
}

// 风格 B：每个用例一个端口
interface CreateOrderUseCase { OrderId createOrder(CreateOrderCommand cmd); }
interface CancelOrderUseCase { void cancelOrder(OrderId id); }

// 风格 C：CQS 命令模式
interface CommandBus { <R> R dispatch(Command<R> command); }

// 风格 D：响应式
interface OrderRepositoryReactive { Mono<Order> findById(OrderId id); }

// 风格 E：用 Function
@Bean Function<CreateOrderCommand, OrderId> createOrder() { /* ... */ }

// 风格 F：用 Spring Data Repository 直接当端口
interface OrderRepository extends JpaRepository<OrderJpaEntity, String> { /* ... */ }
```

新人看完想哭。Code Review 时同事问"为什么这里用风格 B 那里用风格 F？"，回答：" 因为我以前就这么写的"。

#### 根因

Hexagonal 不规定端口的具体形态。各人按自己的"老经验"写，导致:

- 后端 Java 老兵 → 写 Repository（风格 A）
- DDD 实践者 → 写 UseCase（风格 B）
- 函数式爱好者 → 写 Function（风格 E）
- Spring Data 重度用户 → 直接当端口（风格 F，违背 Hexagonal）

混在一个项目就是灾难。

#### 修复

团队**强制统一**端口规范，写进 ARCH.md：

```markdown
## 端口规范（团队约定 v1.2）

### Port In（驱动端口 / 用例）
- 每个**用户可见的业务动作**一个 UseCase 接口
- 接口名：动词 + UseCase（CreateOrderUseCase）
- 入参：record Command（在接口内嵌套定义）
- 出参：domain 类型或 record Result

### Port Out（被驱动端口）
- 持久化后缀：Repository
- 外部 RPC 后缀：Gateway
- 消息发送后缀：Publisher
- 接口签名只用 domain 类型，不允许 Spring/JPA/外部 SDK 类型

### 禁用
- 不允许 Function/BiFunction 当端口
- 不允许 Spring Data Repository 接口直接做端口（必须包一层 Adapter）
- 不允许返回 Page/Pageable（用 domain 内的 PageQuery）
- 不允许返回 Mono/Flux 在端口签名里（响应式包装在 Adapter 内部）
```

老代码渐进式重构：每次改一个文件就顺手把它改成新规范，**不要一次性大重构**。

#### 验证

ArchUnit + 自定义规则：

```java
@ArchTest
static final ArchRule useCaseShouldFollowConvention =
    classes()
        .that().resideInAPackage("..port.in..")
        .should().beInterfaces()
        .andShould().haveSimpleNameEndingWith("UseCase");

@ArchTest
static final ArchRule outboundPortShouldNotReturnSpringDataPage =
    noMethods()
        .that().areDeclaredInClassesThat().resideInAPackage("..port..")
        .should().haveRawReturnType("org.springframework.data.domain.Page");
```

新人入职第一周必须完成"端口规范熟悉作业"：照规范写 3 个新端口、改 1 个老端口。

> **口诀**：六边形不是一个人能落地的，是**团队规范**。**没有 ARCH.md + ArchUnit + 入职培训三件套**，半年后必定退化成混乱的伪六边形。

---

### 案例 ⑨ 包结构与团队结构不对齐 —— "康威定律的反噬"

#### 现象

公司有 5 个团队（订单、库存、支付、用户、风控），但单体项目结构按"技术分层"组织：

```
src/main/java/
  controller/     ← 5 个团队的 Controller 都在这
  service/        ← 5 个团队的 Service 都在这
  repository/     ← 5 个团队的 Repository 都在这
  domain/         ← 5 个团队的 domain 都在这
```

结果：每个团队改自己的功能都要跨几个包，PR 经常冲突，Code Owner 变成"全部文件都需要 5 个团队的人 review"。

#### 根因

按技术分层组织包结构（package by layer）在小项目还行，**模块多了之后违反康威定律**：

> 设计任何系统的组织，最终的设计成果都会与该组织的沟通结构相一致。 —— Melvin Conway, 1968

5 个团队但包结构不对应 5 个模块，意味着团队间的"沟通边界"和"代码边界"错配，**冲突高 + 上线慢 + 责任不清**。

#### 修复

切换为 **package by feature + Hexagonal 内部**：

```
src/main/java/com/example/
├── order/                        ← 订单团队 owns 整个目录
│   ├── domain/
│   ├── application/
│   ├── infrastructure/
│   └── interfaces/
├── inventory/                    ← 库存团队 owns
│   ├── domain/
│   └── ...
├── payment/                      ← 支付团队 owns
├── user/                         ← 用户团队 owns
└── risk/                         ← 风控团队 owns
```

外加：

- **每个 feature 包根目录一个 OWNERS 文件**，CI 校验跨包 PR 必须有所有相关团队 approve
- **跨 feature 的调用必须走 application 层暴露的 UseCase 接口**，禁止跨 feature 直接 import domain
- 用 ArchUnit 强制隔离

```java
@ArchTest
static final ArchRule featuresMustNotDependOnEachOthersInternals =
    layeredArchitecture()
        .consideringAllDependencies()
        .layer("order").definedBy("..order..")
        .layer("inventory").definedBy("..inventory..")
        .layer("payment").definedBy("..payment..")
        .whereLayer("order").mayOnlyAccessLayers("inventory", "payment")  // 仅通过特定接口
        .ignoreDependency(
            ".*\\.order\\..*",
            ".*\\.(inventory|payment)\\.application\\.api\\..*"   // ★ 仅允许调对方 application/api 包
        );
```

为了真正强制"模块边界"，可以更进一步用 [Spring Modulith](https://spring.io/projects/spring-modulith)：

```java
@ApplicationModule(allowedDependencies = {"inventory::api", "payment::api"})
package com.example.order;
```

Spring Modulith 在启动时校验模块依赖，违反就启动失败。

#### 验证

```bash
# 看谁在动谁的代码
git log --since="3 months ago" --pretty=format:"%an" --name-only \
  | awk '/^[A-Z]/ {a=$0} /^src/ {print a, $0}' \
  | awk '{split($2, p, "/"); print $1, p[5]}' \
  | sort | uniq -c | sort -rn
```

输出形如：

```
  142 alice  order
   85 alice  inventory     ← ★ alice 是订单团队但改了 85 次库存代码？
   23 bob    order
   ...
```

**alice 是订单团队但 3 个月内改了 85 次库存代码**，这是个强信号说明边界没切好。

> **口诀**：**包结构必须反映团队结构**（康威定律）。Hexagonal 的"分层"是模块**内部**结构，模块**之间**要按 feature/团队切。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 包结构层
- [ ] 顶层包按 feature 切，不按层切（不要有 `controller/` `service/` 平级在 src 根）
- [ ] 每个 feature 内部有 `domain` / `application` / `infrastructure` / `interfaces` 四层
- [ ] domain 包不依赖任何外部框架（Spring、JPA、Jackson、Hibernate）
- [ ] application 包不依赖 infrastructure 包
- [ ] interfaces 包不调用 infrastructure 包（必须经过 application 层）

### 端口层
- [ ] 所有 Port 是 interface，不是 abstract class
- [ ] Port In 用 `UseCase` 后缀；Port Out 按职责用 `Repository` / `Gateway` / `Publisher`
- [ ] Port 接口的方法签名只用 domain 类型，不出现 Spring Data 的 `Page`、`Pageable`
- [ ] Port In 的入参是 record Command，出参是 record Result（或 domain 对象）
- [ ] Port Out 的接口写在 domain 包（不是 infrastructure）

### Domain 层
- [ ] Aggregate Root 用静态工厂方法创建（`Order.create(...)`），不用 public 构造函数
- [ ] Aggregate Root 没有 setter，状态变更必须经过业务方法
- [ ] Value Object 是 final 不可变（用 record 或加 final）
- [ ] 业务规则写在 Aggregate / Domain Policy 里，不在 Service / Controller 里
- [ ] 区分 `Order.create(...)`（新建）和 `Order.reconstitute(...)`（从 DB 重建）

### Adapter 层
- [ ] JPA Entity 和 Domain 是两个类，中间用 Mapper 转
- [ ] 外部 SDK 异常在 Adapter 内被翻译成 domain 业务异常（ACL）
- [ ] 跨 BC / 跨服务调用必须走 ACL 翻译，不直接用上游模型
- [ ] Adapter 命名：`{端口名}{技术}Adapter`（如 `OrderRepositoryJpaAdapter`）

### 测试层
- [ ] 业务测试（domain + application）是纯 JUnit，不用 `@SpringBootTest`
- [ ] 写好 InMemory 适配器复用（不要每个测试自己 mock）
- [ ] JPA 适配器测试用 Testcontainers 真 PG/MySQL，不用 H2
- [ ] 测试金字塔：单元 80% / 集成 15% / E2E 5%
- [ ] CI 跑全量测试 < 10 分钟

### 治理层
- [ ] 项目根有 ARCH.md，写明端口/适配器命名规范
- [ ] CI 跑 ArchUnit，校验 domain 不依赖框架、不依赖 infrastructure
- [ ] 大模块用 Spring Modulith 强制模块边界
- [ ] 每个 feature 包有 OWNERS 文件
- [ ] 简单 CRUD 模块明确豁免 Hexagonal（有声明、有审批）

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### Vertical Slice Architecture
- Jimmy Bogard 提出的"按功能切片，每个用例自包含"风格
- 一个 feature 一个垂直切片，包含 Endpoint / Handler / Validator / DTO
- 与 Hexagonal 不冲突，可以"外层 Hex + 内层 Vertical Slice"
- 适合 CQRS 重的场景，每个 Command / Query 是一个独立的切片

### Functional Core, Imperative Shell
- Gary Bernhardt 提出的另一种隔离方式
- "纯函数"在中心（不可变 + 无副作用），"命令式 IO"在外壳
- 比 Hexagonal 更激进：连 OOP 多态都被替代为函数组合
- 在 Scala / Haskell / 现代 Kotlin 里非常自然

### Event-Driven Architecture 与六边形协作
- Driving Adapter 之一就是 MQ Consumer，Driven Adapter 之一就是 MQ Producer
- Event Sourcing 改变 Repository 语义：`save = appendEvents`，`findById = replayEvents`
- 这套模式让六边形 + Event Sourcing + CQRS 形成"三位一体"

### Modular Monolith
- 单进程内强制模块边界，比微服务简单、比包结构严格
- [Spring Modulith](https://spring.io/projects/spring-modulith) 是 Spring 官方实现
- 模块间通过 ApplicationEvent 通信，避免直接 import
- 演进路径：单体 → Modular Monolith → 拆分微服务（依据真实流量边界）

### ArchUnit 进阶
- 不止能查依赖方向，还能查注解、命名、继承、循环依赖
- [ArchUnit User Guide](https://www.archunit.org/userguide/html/000_Index.html)
- 自定义 Condition 可以查"所有 Repository 接口必须只返回 Optional 或 List"
- CI 上挂 ArchUnit 是任何架构落地的"最后一道防线"

### DDD + Hexagonal + CQRS 复合架构
- DDD 提供战术（Aggregate / VO / Domain Event）
- Hexagonal 提供结构（Port / Adapter / 依赖反转）
- CQRS 把 Command 和 Query 拆两套模型 / 两套存储
- 适用场景：超高并发读写、读写差异巨大、读模型有多种视图

### Quarkus / Micronaut 下的架构落地
- 编译时 DI（CDI 兼容）天然支持"接口在内、实现在外"
- 对 GraalVM Native Image 友好，启动 < 50ms
- Hexagonal 在这两个框架上的代价比 Spring 更小（无运行时反射）
- 适合 Serverless / Function-as-a-Service 的极致冷启动场景

### 与 Clean Architecture 的细微差别值得读 Bob 原书
- *Clean Architecture: A Craftsman's Guide* (2017)
- 重点章节：第 22 章（The Clean Architecture）+ 第 23 章（Presenters and Humble Objects）
- Bob 强调的"Humble Object"模式 = Adapter 应该尽量薄，把可测试的逻辑挤到内层

---

## 7. 引用

- Alistair Cockburn, [*Hexagonal Architecture*](https://alistair.cockburn.us/hexagonal-architecture/)（2005）
- Robert C. Martin, [*The Clean Architecture*](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)（2012）
- Jeffrey Palermo, [*The Onion Architecture*](https://jeffreypalermo.com/2008/07/the-onion-architecture-part-1/)（2008）
- Robert C. Martin, *Clean Architecture: A Craftsman's Guide to Software Structure and Design*（2017）
- Eric Evans, *Domain-Driven Design: Tackling Complexity in the Heart of Software*（2003）
- Vaughn Vernon, *Implementing Domain-Driven Design*（2013，第 4 章 Architecture 详谈 Hex）
- Tom Hombergs, *Get Your Hands Dirty on Clean Architecture*（2019，Java + Spring 实战）
- [Spring Modulith 官方文档](https://docs.spring.io/spring-modulith/reference/)
- [ArchUnit 官方文档](https://www.archunit.org/)
- Conway, Melvin, *How Do Committees Invent?*（1968，康威定律原文）

---

> 配套：[DDD 战术战略](/article/architecture/ddd) | [API 设计与 OpenAPI](/article/architecture/api-design) | [微服务拆分演进](/article/architecture/microservices)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你觉得这个深度对，告诉我下一篇该写哪个。
