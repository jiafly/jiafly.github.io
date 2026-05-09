# DDD 战术与战略设计 —— 从 Aggregate 到 Bounded Context 的工程落地

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试或了解概念，去搜一篇 "DDD 入门" 就够了。
> 这里要回答的，是**真打算把 DDD 落到 Spring 工程里、且不想被同事骂"过度设计"**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过：

1. Entity / Value Object / Aggregate Root 的精确边界是什么？为什么 `Money` 必须是 VO 而不是 Entity？同一个 `Address` 到底应该是 VO 还是 Entity，怎么判？
2. Bounded Context（限界上下文）按什么划？按业务功能、按团队、按数据库、还是按 URL 前缀？给出**反例**：哪种划法看起来很 DDD 但其实是错的。
3. Context Mapping 的 9 种关系（Partnership / Customer-Supplier / Conformist / ACL / OHS / PL / SK / BB / SS）你能不打开书各举一个项目里的例子吗？
4. Domain Event 和 MQ 消息是不是一回事？Domain Event 一定要异步吗？事务边界怎么处理？
5. CQRS 一定要配 Event Sourcing 吗？只用 CQRS 不用 ES 行不行？反过来呢？
6. 防腐层（Anti-Corruption Layer）和 Adapter / Facade 模式有什么区别？为什么不能直接用 Adapter 替代？
7. 一个 Bounded Context 是不是必须对应一个微服务？如果一个微服务里塞两个 BC 会怎样？
8. JPA 的 `@Entity` 和 DDD 的 Entity 是不是一回事？Repository 接口为什么要写在 domain 层？为什么用 JPA 时充血模型经常变形？

如果有半数答不上，往下看。

---

## 1. 战略设计（顶层）—— 在画类图之前先搞清楚画给谁看

绝大多数 DDD 落地失败，根因不是技术，是**战略层面没对齐**：架构师在画聚合，业务方在用另一套词，两边永远在不同频道。Eric Evans 那本"蓝皮书"前半本通篇在讲战略，不是凑字数，是因为没有战略，战术全是空中楼阁。

### 1.1 Ubiquitous Language（统一语言）—— 不是术语表，是契约

统一语言的核心不是"建一个 Glossary 文档"。它的本质是：**业务方说出来的词、代码里的类名、数据库表的字段、API 的字段，必须是同一个**。

错误示例：业务说"订单完成"，代码里类叫 `OrderTerminated`，数据库表字段叫 `order_status = 9`。三套语言。一旦业务变更"订单完成"的定义，三个地方都要改，且很容易遗漏一个。

正确做法：

```java
public enum OrderStatus {
    CREATED,           // 业务术语：已创建
    PAID,              // 业务术语：已支付
    SHIPPED,           // 业务术语：已发货
    COMPLETED,         // 业务术语：已完成（用户确认收货）
    CANCELLED,         // 业务术语：已取消
    REFUNDED;          // 业务术语：已退款
}
```

枚举名 = 业务方口头说的词。**不要**用 `STATUS_1 / STATUS_2`，**不要**自己造 `Terminated / Finalized` 这种词。

如何在团队里立这套语言？最有效的方法是 Event Storming（见 1.5），让业务方拿着便签纸自己写词、自己排顺序。

> **口诀**：统一语言不在 Wiki 里，在代码命名里、在 API 字段里、在表字段里。三处对不上，DDD 失败一半。

### 1.2 Bounded Context（限界上下文）的识别

Bounded Context 的定义被反复误解。它**不是模块**、**不是子系统**、**不是微服务**。它是**某个模型有效的边界**。

举例：电商系统里的 `Product` 这个词：

| 上下文 | Product 的含义 | 关键属性 |
| --- | --- | --- |
| 商品中心 | 商品定义、SKU、SPU、属性 | id、名称、品牌、规格、富文本详情 |
| 库存中心 | 仓库里的可售单元 | id、仓库号、批次、可用数量、锁定数量 |
| 订单中心 | 订单行里的快照 | id、下单时名称、下单时单价、数量 |
| 推荐中心 | 推荐图谱里的节点 | id、向量特征、点击率、转化率 |

四个 `Product` 共享一个 ID，但**模型完全不同**。如果非要用一个 `Product` 类涵盖所有属性，你会得到一个 200 字段的"上帝类"，每改一个字段都要全公司提心吊胆。

**Bounded Context 就是在画"哪些字段、哪些行为属于同一个模型"的边界**。

错误的边界划法：
- 按"系统的物理边界"划（数据库 / 项目）—— 这是结果不是原因
- 按"组织架构"划 —— 容易但僵化（康威定律的反面教训）
- 按"业务功能"粗划 —— 太粗，会把不同模型塞进一个 BC

正确的边界划法：
- **看模型一致性**：在某个范围内，同一个名词只有一个含义
- **看团队认知**：业务方在某个范围内说话不需要"翻译"
- **看变化频率**：一起变化的逻辑放一起

> **口诀**：Bounded Context 是**模型的有效范围**。一个名词跨边界含义不同时，就该划开。

### 1.3 Context Mapping 的 9 种关系

两个 BC 之间总要交互。Eric Evans 列了 9 种典型关系，每一种都是"上下游怎么协作"的契约：

| 关系 | 中文 | 谁强势 | 典型场景 |
| --- | --- | --- | --- |
| **Partnership** | 合作关系 | 平等 | 两个团队同进同退，要么一起成功要么一起失败（如订单 + 支付） |
| **Customer-Supplier** | 客户-供应商 | 上游强 | 上游要主动适配下游需求（如商品中心给搜索中心提供 dump） |
| **Conformist** | 跟随者 | 上游强 | 下游被迫吃下上游模型，没得选（如对接外部 ERP） |
| **Anti-Corruption Layer (ACL)** | 防腐层 | 下游自保 | 下游不愿被上游模型污染，自己加一层翻译 |
| **Open Host Service (OHS)** | 开放主机服务 | 上游服务多家 | 上游开通用 API/事件给多个下游消费 |
| **Published Language (PL)** | 发布语言 | 中立标准 | 用业界标准协议（如 ISO 20022 金融报文）作为交换格式 |
| **Shared Kernel (SK)** | 共享内核 | 双方共管 | 两个 BC 共享一小撮核心模型（要谨慎，改动需双方签字） |
| **Big Ball of Mud (BB)** | 大泥球 | 无 | 现实里到处都有 —— 老系统改不动，只能围着它转 |
| **Separate Ways (SS)** | 各行其道 | 无关 | 两个 BC 完全无关，就别强行集成 |

实战要点：
- **任何依赖外部系统（第三方、老系统）的接入点，默认上 ACL**。短期省事不上 ACL，长期就是上游一变你跟着重构。
- **Shared Kernel 慎用**。它看起来"复用代码"，实际上是"两个团队共享一个修改风险"。除非两边团队组织上紧密绑定，否则别用。
- **OHS + PL 是大型平台的标配**。比如阿里中台暴露的开放 API + JSON Schema。

> **口诀**：上下游关系决定改动成本。ACL 是个体保护伞，OHS+PL 是平台标配，SK 是核武器（用前看清剂量）。

### 1.4 Subdomain 的三类（Core / Supporting / Generic）

战略设计第二层划分：业务子域按重要性分三类。这个分类直接决定**资源投入策略**。

| 类型 | 定义 | 投入策略 | 例子 |
| --- | --- | --- | --- |
| **Core Domain** | 公司的核心竞争力 | 自己做、用最好的人、用 DDD 重型方法 | 滴滴的派单算法、淘宝的商品检索 |
| **Supporting Subdomain** | 重要但不差异化 | 自己做、轻型方法（CRUD 也行） | 后台管理、运营报表 |
| **Generic Subdomain** | 通用业务能力 | 买、用开源、外包 | 短信网关、对象存储、日志收集 |

错误：把所有子域都按 Core Domain 的标准做。结果资源全摊平，Core 反而做不好。

正确：**先识别 Core Domain，再决定哪些值得 DDD**。Generic 的东西就直接用 SaaS / 开源，别浪费团队精力造轮子。

> **口诀**：DDD 是给 Core Domain 用的。Generic 子域用 DDD 是杀鸡用牛刀，迟早被业务嫌"过度设计"。

### 1.5 Event Storming 工作坊方法

Event Storming 是 Alberto Brandolini 推广的一种识别 BC 的工作坊方法。它的核心：**让业务方主导，用便签纸把"事件"按时间排列**。

工作流程（粗版）：

```
Step 1: 大家在墙上贴橙色便签 = 领域事件（"已下单"、"已支付"、"已发货"）
        要求：过去时动词，业务方说什么就贴什么

Step 2: 蓝色便签 = 命令（"提交订单"、"发起支付"）
        命令在事件之前发生

Step 3: 黄色便签 = 角色（用户、商家、系统）
        每个命令由谁发起？

Step 4: 红色便签 = 热点（业务方有分歧的地方）
        这些是 BC 的潜在边界

Step 5: 把便签按时间从左到右排
        看哪些事件聚类在一起 → 这就是 BC
```

实战经验：
- 业务方必须到场，且必须参与贴便签。光架构师在那闭门造车没意义。
- 不要纠结颜色规范，工具是次要的。重要的是大家**对事件的命名达成一致**（这就是统一语言的雏形）。
- 半天到一天的时间，能搞定一个中等业务域。
- 输出物：一面墙的便签 + 拍照存档 + 一份 BC 草图。

> **口诀**：Event Storming 不是画 UML，是把业务方拉进来一起把"事件序列"摆出来。事件聚类的地方就是 BC。

### 1.6 微服务边界与 Bounded Context 的关系

经典误解："一个微服务 = 一个 BC"。这话**对一半**。

精确说法：
- **一个 BC 可以对应一个微服务**（最常见、最干净）
- **一个 BC 可以对应多个微服务**（BC 内部按读写、按性能特性拆，比如订单 BC 拆成 OrderCommand + OrderQuery）
- **多个 BC 不应该塞进一个微服务**（这是反模式 —— 一个进程内多个模型混着，迟早互相污染）

判断标准：
- 如果两个 BC 经常一起部署、一起改，且数据强一致 → 可以暂时同进程，但代码层用 package 隔离
- 如果两个 BC 的变化频率不同、团队不同 → 强烈建议拆服务
- 如果 BC 内部有"读多写少 vs 写多读少"的明显分化 → 一个 BC 拆成多个服务（CQRS 路线）

反模式：把一堆 BC 塞进一个 monolith，对外用 REST 假装独立。这种"伪微服务"运维成本和单体一样，但把所有调用都变成了网络调用 —— 既没拿到微服务的隔离好处，又付了网络调用的代价。

> **口诀**：BC 是模型边界，微服务是部署边界。两者通常对齐，但不是 1:1 强制。先想清楚 BC，再决定怎么拆服务。

---

## 2. 战术设计（落地）—— 把模型落到代码上

战略想清楚了，还得有战术工具把模型刻进代码。下面 7 个组件是 Eric Evans 战术设计的核心，每一个都对应"代码里写成什么样"。

### 2.1 Entity / Value Object / Aggregate

三个概念的精确边界，是 DDD 战术里最容易讲糊的。

**Entity（实体）**：有唯一标识、有生命周期、可变。两个 Entity 内容完全相同但 ID 不同，就是不同的 Entity。

```java
public class User {
    private final UserId id;    // 唯一标识，决定身份
    private String name;        // 可变属性
    private String email;
    // 比较 equals 时只看 id
}
```

**Value Object（值对象）**：没有唯一标识、不可变、由属性值定义身份。两个 VO 属性完全相同就是同一个 VO。

```java
public final class Money {
    private final BigDecimal amount;
    private final Currency currency;
    // 必须不可变；equals/hashCode 基于所有字段
    public Money add(Money other) {
        if (!this.currency.equals(other.currency))
            throw new IllegalArgumentException();
        return new Money(this.amount.add(other.amount), this.currency);
    }
}
```

判断方法：**问"两个东西的属性一样，它们是不是同一个？"**
- 是 → VO（如金额、地址、坐标）
- 不是 → Entity（如用户、订单）

陷阱：**`Address` 到底是 VO 还是 Entity？** 答：要看上下文。
- 在用户 BC 里，地址是用户填写的快照，没有独立生命周期 → VO
- 在物流 BC 里，地址是仓库网点，有自己的状态和编号 → Entity

**Aggregate（聚合）**：一组关联 Entity + VO 的集合，对外暴露唯一一个 Aggregate Root（聚合根）。聚合内部强一致，聚合之间最终一致。

```java
public class Order {                       // Aggregate Root
    private final OrderId id;
    private OrderStatus status;
    private Money totalAmount;             // VO
    private List<OrderLine> lines;         // 内部 Entity
    private Address shippingAddress;       // VO

    // 外部只能拿到 OrderId，不能直接拿 OrderLine 的 ID 去操作
    public void addLine(SkuId sku, int quantity, Money unitPrice) { ... }
}

public class OrderLine {                   // Entity，但只在 Order 内部
    private final OrderLineId id;
    private SkuId sku;
    private int quantity;
    private Money unitPrice;
    // package-private，外部碰不到
}
```

聚合的核心规矩：
- **外部只能引用 Aggregate Root**，不能直接引用聚合内部的 Entity
- 聚合根负责保证**内部一致性**（如订单总金额 = 各行金额之和）
- 修改只能通过聚合根的方法（如 `order.addLine(...)`），不能 `order.getLines().add(...)`

> **口诀**：Entity 看 ID，VO 看值，Aggregate 看一致性边界。聚合根是聚合的"门"。

### 2.2 Aggregate 设计原则（小聚合 + 引用根 + 一致性边界）

Vaughn Vernon 在《实现领域驱动设计》中总结了 4 条聚合设计原则。这 4 条是从无数失败项目里总结出来的，每一条违反都有具体后果：

**原则 1：聚合内部强一致，聚合之间最终一致**

聚合是事务边界。一个事务**只修改一个聚合**。跨聚合的一致性用 Domain Event + 最终一致性达成。

```java
// ❌ 一个事务里改两个聚合
@Transactional
public void confirmOrder(OrderId orderId) {
    Order order = orderRepo.find(orderId);
    order.confirm();
    Inventory inv = inventoryRepo.find(order.getSkuId());
    inv.decrease(order.getQuantity());     // ★ 跨聚合
    orderRepo.save(order);
    inventoryRepo.save(inv);
}

// ✅ 事务只改 Order，发事件让 Inventory 异步处理
@Transactional
public void confirmOrder(OrderId orderId) {
    Order order = orderRepo.find(orderId);
    order.confirm();   // 内部会发 OrderConfirmedEvent
    orderRepo.save(order);
}
```

**原则 2：通过 ID 引用其他聚合**

聚合内部不要持有其他聚合的对象引用。要么持 ID，要么不持。

```java
// ❌ 直接引用 User 对象
public class Order {
    private User customer;    // 跨聚合引用
}

// ✅ 只持 ID
public class Order {
    private CustomerId customerId;
}
```

为什么？两个原因：
1. **避免大对象图**：加载 Order 时不会顺带把 User 也加载（除非显式查）
2. **避免事务边界混乱**：不会有人写出 `order.getCustomer().updateName(...)` 这种跨聚合修改

**原则 3：聚合要小**

经验值：一个聚合内部的 Entity 数量控制在 10 个量级。聚合大了：
- 加载慢（一次查很多关联）
- 并发冲突高（任何修改都竞争同一个聚合的乐观锁）
- 内存占用大

判断聚合是否过大的信号：
- 加载耗时 > 100ms
- 并发更新经常失败重试
- 单聚合超过 50 个字段

**原则 4：用最终一致性替代跨聚合事务**

跨聚合的一致性用以下三种方式达成：
- Domain Event + MQ
- Saga 模式（见 [分布式事务](/article/distributed/dist-tx)）
- TCC

> **口诀**：聚合 = 事务边界。跨聚合用事件，聚合内用方法。

### 2.3 Domain Service / Application Service / Infrastructure Service 三层

DDD 里"Service"这个词被滥用。实际上有三种完全不同的 Service：

**Domain Service（领域服务）**：处理**跨聚合的领域逻辑**。当某个业务行为不属于任何单个 Entity / VO，但仍是领域逻辑时，放在 Domain Service。

```java
// 转账涉及两个账户聚合，不属于任何单一聚合
public class TransferService {        // Domain Service
    public void transfer(Account from, Account to, Money amount) {
        from.withdraw(amount);
        to.deposit(amount);
    }
}
```

特征：方法是无状态的、参数是聚合根、不涉及任何技术细节。

**Application Service（应用服务）**：处理**用例编排**。它是外部（Controller / MQ Consumer）和领域层的胶水：开事务、查仓储、调聚合方法、发事件、处理异常。

```java
@Service
public class OrderApplicationService {
    private final OrderRepository orderRepo;
    private final ProductGateway productGateway;
    private final DomainEventPublisher publisher;

    @Transactional
    public OrderId placeOrder(PlaceOrderCommand cmd) {
        Product product = productGateway.find(cmd.skuId());
        Order order = Order.create(cmd.customerId(), product, cmd.quantity());
        orderRepo.save(order);
        publisher.publish(new OrderPlacedEvent(order.getId()));
        return order.getId();
    }
}
```

特征：**不放业务规则**，只编排。所有业务规则在聚合或 Domain Service 里。

**Infrastructure Service（基础设施服务）**：处理**技术细节**，如发邮件、调外部 API、读文件。它是 domain 层定义的接口的实现。

```java
// domain 层定义接口
public interface NotificationService {
    void notify(UserId user, String message);
}

// infrastructure 层实现
@Component
public class SmsNotificationService implements NotificationService {
    public void notify(UserId user, String message) {
        smsClient.send(...);
    }
}
```

三者的边界检查：

| Service 类型 | 能 import Spring 吗 | 能 import JPA 吗 | 能持有领域状态吗 |
| --- | --- | --- | --- |
| Domain Service | ❌ | ❌ | ❌（无状态） |
| Application Service | ✅ | ❌（应该通过 Repo） | ❌ |
| Infrastructure Service | ✅ | ✅ | 视情况 |

> **口诀**：Domain 管业务、Application 管编排、Infrastructure 管技术。三层职责一旦混淆，DDD 沦为"贫血 DDD"。

### 2.4 Repository 模式

Repository 是聚合的"集合抽象"。它让上层代码看起来像在操作一个内存里的集合。

```java
public interface OrderRepository {
    Order find(OrderId id);
    Optional<Order> findOptional(OrderId id);
    void save(Order order);
    void remove(Order order);
    List<Order> findByCustomer(CustomerId customerId);
}
```

关键设计：
- **接口在 domain 层定义，实现在 infrastructure 层**（依赖倒置）
- **方法的参数和返回值都是聚合根，不是 DTO**
- **一个聚合根对应一个 Repository**（不是一张表对应一个）

为什么实现要在 infrastructure 层？因为 Repository 的实现涉及具体技术（JPA / MyBatis / MongoDB），这些是技术细节，不能反向污染 domain 层。

```
project/
├── domain/
│   └── order/
│       ├── Order.java               (聚合根)
│       ├── OrderRepository.java     (接口)
│       └── OrderLine.java
└── infrastructure/
    └── persistence/
        └── JpaOrderRepository.java  (实现 OrderRepository)
```

JPA 实现的小坑：JPA `@Entity` 直接作为聚合根经常变形（构造器、ID 策略、关联映射的限制）。两种思路：
- **轻型**：JPA 实体 = 聚合根，接受 JPA 的限制（getter/setter、空构造）
- **重型**：JPA 实体 ≠ 聚合根，仓储实现里手动转换（聚合根纯净，但代码量大）

> **口诀**：Repository = 聚合的集合假象。接口在领域，实现在基础设施。

### 2.5 Domain Event 与 Event-Driven

Domain Event 是"领域里发生的事"的显式表达。它和 MQ 消息**不是一回事**：

| 维度 | Domain Event | MQ 消息 |
| --- | --- | --- |
| 概念层 | 领域语义 | 技术传输 |
| 生命周期 | 业务发生瞬间产生 | 业务事件落到 MQ 后 |
| 命名 | 过去时（OrderPlaced） | 通常 topic 名 |
| 是否一定异步 | 不是（可同步可异步） | 通常异步 |
| 消费方 | 同 BC 内可能消费、跨 BC 通过 MQ | MQ 订阅者 |

Domain Event 的两种发布时机：
- **聚合内部发布**（最干净）：聚合方法内部产生事件，聚合根暂存，事务提交时统一发出
- **应用服务发布**（次之）：应用服务在 Repo.save 之后显式发出

落地示例：

```java
public class Order extends AggregateRoot {
    public void confirm() {
        if (this.status != OrderStatus.PAID)
            throw new IllegalStateException();
        this.status = OrderStatus.SHIPPED;
        // 聚合内部发布事件
        registerEvent(new OrderConfirmedEvent(this.id, Instant.now()));
    }
}

public abstract class AggregateRoot {
    @Transient
    private final List<DomainEvent> events = new ArrayList<>();

    protected void registerEvent(DomainEvent event) { events.add(event); }
    public List<DomainEvent> pullEvents() {
        var copy = List.copyOf(events);
        events.clear();
        return copy;
    }
}
```

应用服务在 save 后发布：

```java
@Transactional
public void confirmOrder(OrderId id) {
    Order order = orderRepo.find(id);
    order.confirm();
    orderRepo.save(order);
    // 拉出聚合内积累的事件，发到 MQ
    order.pullEvents().forEach(publisher::publish);
}
```

事务边界是个大坑：**事件发出去了，但事务回滚了 → 下游收到一个不存在的事件**。三种解决方案：
- **事务内同步发**（仅同进程）：用 Spring 的 `ApplicationEventPublisher` + `@TransactionalEventListener(phase = AFTER_COMMIT)`
- **本地事务表（Outbox Pattern）**：把事件先写本地表，事务提交后异步推 MQ
- **CDC 抓 binlog**：把事件直接写一张表，Debezium / Canal 抓出来

> **口诀**：Domain Event 是领域语义，MQ 是传输。事务边界用 Outbox 守住。

### 2.6 CQRS：Command 与 Query 分离

CQRS（Command Query Responsibility Segregation）是个命名很玄乎的简单想法：**写模型和读模型用不同的对象**。

为什么要分？
- **领域模型适合写**，但不适合读：聚合保证一致性的代价是模型复杂、关联多。一个查询可能要遍历多个聚合
- **读场景的 N+1、JOIN 多、聚合慢，导致领域模型变形**：为了读快，给聚合加冗余字段、加缓存、加 join，结果聚合越来越胖

CQRS 的方案：

```
       ┌──────────────┐                ┌─────────────┐
Web ── │ CommandHandler│ ─→ Aggregate ─→│ Write DB    │
       └──────────────┘                └─────────────┘
                                              │
                                              │ binlog / event
                                              ▼
       ┌──────────────┐                ┌─────────────┐
Web ── │ QueryHandler │ ─────────────→ │ Read Model  │
       └──────────────┘                └─────────────┘
```

写侧：用聚合，保证业务一致性。
读侧：用扁平化的 ReadModel（甚至直接 SQL 查），优化查询性能。

CQRS 的三种粒度：
1. **同库 CQRS**：写用聚合，读直接 SQL → 最轻量，没有同步问题
2. **异库 CQRS**：写库 + 读库分离，binlog 同步 → 适合读量大的场景
3. **CQRS + Event Sourcing**：写存事件，读侧聚合 → 最完整也最复杂

CQRS **不一定**要配 Event Sourcing。两者是正交的。

> **口诀**：CQRS = 写聚合，读扁平。能不上 ES 就不上，复杂度不在一个量级。

### 2.7 Event Sourcing：状态从事件推导

传统 CRUD：数据库存当前状态，每次更新覆盖。
Event Sourcing：数据库存事件流，当前状态 = 重放所有事件的结果。

```
传统：
  account.balance = 100  →  withdraw 30  →  balance = 70  (overwrite)

ES：
  events = [Created(balance=0), Deposited(100), Withdrew(30)]
  current_state = replay(events) = balance(70)
```

ES 的好处：
- **完整审计**：每个状态变更都有事件记录
- **时间旅行**：能回到任意时间点的状态
- **业务洞察**：事件流本身是业务分析的金矿
- **debug 友好**：bug 出现时能 replay 到出错那一刻

ES 的代价：
- **读取慢**：每次都要 replay → 必须配 Snapshot（每 N 个事件存一次快照）
- **schema 演化难**：旧事件结构改了怎么办？
- **跨聚合查询难**：必须通过事件投影到读模型
- **学习成本高**：团队心智负担重

Snapshot 示意：

```java
public Account loadAccount(AccountId id) {
    Snapshot snapshot = snapshotRepo.latest(id);          // 最新快照
    Account account = snapshot != null
        ? Account.fromSnapshot(snapshot)
        : Account.createEmpty(id);

    long fromVersion = snapshot != null ? snapshot.version() : 0;
    List<DomainEvent> events = eventStore.eventsAfter(id, fromVersion);

    for (DomainEvent e : events) {
        account.apply(e);                                  // 重放
    }
    return account;
}
```

ES 不是银弹。**只有强审计、复杂业务流转、事件本身有分析价值**的场景才值得上。普通 CRUD 业务上 ES 是自找麻烦。

> **口诀**：ES = 状态从事件推导。要审计、要时间旅行才上；普通业务别凑热闹。

---

## 3. 工程切片（Spring + DDD）

理论讲完了，落到代码长什么样？下面给一个"中型"的 Spring Boot + DDD 项目骨架。每段都是可运行代码，不是伪码。

### 3.1 包结构（domain / application / infrastructure / interfaces 四层）

```
com.example.order/
├── domain/                    (领域层，不依赖任何框架)
│   ├── model/
│   │   ├── Order.java         (Aggregate Root)
│   │   ├── OrderId.java       (VO)
│   │   ├── OrderLine.java     (Entity，包内可见)
│   │   ├── OrderStatus.java
│   │   └── Money.java         (VO)
│   ├── repository/
│   │   └── OrderRepository.java   (接口)
│   ├── event/
│   │   ├── OrderPlacedEvent.java
│   │   └── OrderConfirmedEvent.java
│   └── service/
│       └── PricingService.java    (Domain Service)
│
├── application/               (应用层，依赖 Spring，不依赖 ORM)
│   ├── command/
│   │   └── PlaceOrderCommand.java
│   ├── query/
│   │   └── OrderDetailQuery.java
│   └── service/
│       ├── OrderApplicationService.java
│       └── OrderQueryService.java
│
├── infrastructure/            (基础设施层，依赖 ORM、消息、外部 API)
│   ├── persistence/
│   │   ├── JpaOrderRepository.java
│   │   └── OrderJpaEntity.java
│   ├── messaging/
│   │   └── DomainEventPublisher.java
│   └── gateway/
│       ├── ProductGateway.java         (接口，定义在 domain)
│       └── ProductGatewayHttp.java     (实现)
│
└── interfaces/                (接口层，REST/gRPC/MQ Consumer)
    ├── rest/
    │   └── OrderController.java
    └── consumer/
        └── PaymentEventConsumer.java
```

依赖方向：`interfaces → application → domain ← infrastructure`

domain 层是核心，**它不依赖任何上层**。infrastructure 反向依赖 domain（实现 domain 定义的接口）。这就是依赖倒置原则在 DDD 里的体现。

### 3.2 Aggregate Root 的 Java 实现（充血模型）

充血模型：业务逻辑在领域对象方法里，不在 Service 里。

```java
package com.example.order.domain.model;

public class Order extends AggregateRoot {
    private final OrderId id;
    private final CustomerId customerId;
    private OrderStatus status;
    private Money totalAmount;
    private final List<OrderLine> lines = new ArrayList<>();
    private Address shippingAddress;
    private Instant createdAt;
    private long version;       // 乐观锁

    // 构造器私有，强制走工厂方法
    private Order(OrderId id, CustomerId customerId, Address shippingAddress) {
        this.id = id;
        this.customerId = customerId;
        this.shippingAddress = shippingAddress;
        this.status = OrderStatus.CREATED;
        this.totalAmount = Money.zero(Currency.CNY);
        this.createdAt = Instant.now();
    }

    // 工厂方法：保证创建时的所有不变量
    public static Order create(CustomerId customerId, Address address) {
        if (customerId == null || address == null)
            throw new IllegalArgumentException("customer/address required");
        Order order = new Order(OrderId.next(), customerId, address);
        order.registerEvent(new OrderCreatedEvent(order.id, customerId, Instant.now()));
        return order;
    }

    // 业务方法：业务规则在这里
    public void addLine(SkuId sku, int quantity, Money unitPrice) {
        if (this.status != OrderStatus.CREATED)
            throw new IllegalStateException("can not modify after submit");
        if (quantity <= 0)
            throw new IllegalArgumentException("quantity must > 0");
        OrderLine line = new OrderLine(OrderLineId.next(), sku, quantity, unitPrice);
        this.lines.add(line);
        recalculateTotal();
    }

    public void submit() {
        if (this.status != OrderStatus.CREATED)
            throw new IllegalStateException("only CREATED can submit");
        if (this.lines.isEmpty())
            throw new IllegalStateException("empty order");
        this.status = OrderStatus.PAID;
        registerEvent(new OrderSubmittedEvent(this.id, this.totalAmount));
    }

    public void confirm() {
        if (this.status != OrderStatus.PAID)
            throw new IllegalStateException("only PAID can confirm");
        this.status = OrderStatus.SHIPPED;
        registerEvent(new OrderConfirmedEvent(this.id, Instant.now()));
    }

    public void cancel(String reason) {
        if (this.status == OrderStatus.SHIPPED || this.status == OrderStatus.COMPLETED)
            throw new IllegalStateException("can not cancel after shipped");
        this.status = OrderStatus.CANCELLED;
        registerEvent(new OrderCancelledEvent(this.id, reason));
    }

    private void recalculateTotal() {
        this.totalAmount = lines.stream()
            .map(OrderLine::subtotal)
            .reduce(Money.zero(Currency.CNY), Money::add);
    }

    // 只暴露聚合根 ID 和状态，不暴露 lines 集合
    public OrderId getId() { return id; }
    public OrderStatus getStatus() { return status; }
    public Money getTotalAmount() { return totalAmount; }
    public List<OrderLine> getLines() { return Collections.unmodifiableList(lines); }
}
```

注意几个细节：
- **构造器私有 + 工厂方法**：强制创建时通过工厂，保证不变量
- **业务方法守卫状态机**：每个方法第一行先检查状态，非法状态直接抛
- **getLines 返回不可变列表**：外部拿到也改不了，保护聚合内部
- **业务规则全在这里**：`addLine` 里检查数量、状态；`submit` 里检查空订单。**不要让 Service 写这些**

### 3.3 Domain Event 的发布与订阅

聚合根基类：

```java
package com.example.order.domain.model;

public abstract class AggregateRoot {
    private final transient List<DomainEvent> events = new ArrayList<>();

    protected void registerEvent(DomainEvent event) {
        this.events.add(event);
    }

    public List<DomainEvent> pullEvents() {
        var copy = List.copyOf(events);
        events.clear();
        return copy;
    }
}

public interface DomainEvent {
    Instant occurredAt();
    String eventType();
}

public record OrderConfirmedEvent(OrderId orderId, Instant occurredAt) implements DomainEvent {
    public String eventType() { return "OrderConfirmed"; }
}
```

应用服务发布：

```java
@Service
public class OrderApplicationService {
    private final OrderRepository orderRepo;
    private final ApplicationEventPublisher publisher;

    @Transactional
    public void confirmOrder(OrderId id) {
        Order order = orderRepo.find(id);
        order.confirm();
        orderRepo.save(order);
        order.pullEvents().forEach(publisher::publishEvent);
    }
}
```

同进程订阅（事务内同步）：

```java
@Component
public class OrderEventListener {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onConfirmed(OrderConfirmedEvent event) {
        // 事务提交后才处理，避免事务回滚导致脏事件
        notifyCustomer(event.orderId());
    }
}
```

跨服务订阅（用 MQ + Outbox Pattern）：

```java
@Component
public class OutboxEventPublisher {
    private final OutboxRepository outboxRepo;

    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    public void persistEvent(DomainEvent event) {
        // 同事务写本地 outbox 表
        outboxRepo.save(new OutboxRecord(
            UUID.randomUUID(), event.eventType(),
            JsonSerializer.serialize(event), Instant.now(), false));
    }
}

@Component
public class OutboxRelay {
    @Scheduled(fixedDelay = 1000)
    public void relay() {
        List<OutboxRecord> pending = outboxRepo.findUnpublished(100);
        for (OutboxRecord r : pending) {
            mqTemplate.send(r.eventType(), r.payload());
            outboxRepo.markPublished(r.id());
        }
    }
}
```

为什么要 Outbox？因为"写 DB"和"发 MQ"不是同一个事务。Outbox 把事件先写到本地表（同事务保证），再异步搬到 MQ。

### 3.4 Repository 接口在 domain 层、实现在 infrastructure 层

domain 层接口：

```java
package com.example.order.domain.repository;

public interface OrderRepository {
    Order find(OrderId id);
    Optional<Order> findOptional(OrderId id);
    void save(Order order);
    void remove(Order order);
}
```

infrastructure 层实现（用 JPA 时常见的"双层模型"）：

```java
package com.example.order.infrastructure.persistence;

@Entity
@Table(name = "t_order")
class OrderJpaEntity {
    @Id private String id;
    private String customerId;
    private String status;
    private BigDecimal totalAmount;
    private String currency;
    @OneToMany(cascade = ALL, orphanRemoval = true, fetch = LAZY)
    private List<OrderLineJpaEntity> lines;
    @Version private Long version;
    // 全字段 getter/setter
}

@Repository
public class JpaOrderRepository implements OrderRepository {
    private final OrderJpaRepo jpa;     // Spring Data JPA 接口

    public Order find(OrderId id) {
        OrderJpaEntity po = jpa.findById(id.value())
            .orElseThrow(() -> new OrderNotFoundException(id));
        return toDomain(po);
    }

    public void save(Order order) {
        OrderJpaEntity po = jpa.findById(order.getId().value())
            .orElseGet(OrderJpaEntity::new);
        merge(po, order);
        jpa.save(po);
    }

    private Order toDomain(OrderJpaEntity po) {
        // PO -> 领域对象（用 reflection 或专用构造器）
        return OrderReconstructor.reconstruct(po);
    }

    private void merge(OrderJpaEntity po, Order order) {
        po.setId(order.getId().value());
        po.setStatus(order.getStatus().name());
        po.setTotalAmount(order.getTotalAmount().amount());
        // ...
    }
}
```

为什么要分领域对象和 JPA 实体？因为 JPA 的限制（无参构造、setter、关联映射）会污染领域模型。两层分开，领域纯净，但代价是 PO ↔ DO 转换代码量大。

简化方案：如果团队不接受双层映射，可以让 JPA 实体直接做聚合根，付出"领域对象有 setter"的代价。Vaughn Vernon 在《IDDD》里也认可这种妥协方案。

### 3.5 防腐层（Anti-Corruption Layer）的实现示例

防腐层场景：你的订单系统要调一个老的 ERP 系统拿商品价格。ERP 的 API 是 SOAP，字段命名诡异，模型一塌糊涂。

不加防腐层：

```java
// ❌ ERP 的模型直接污染你的领域
public class Order {
    public void addLineFromErp(ErpProductDto erpProduct) {
        // erpProduct.PROD_ID, erpProduct.UNIT_PRC_AMT, erpProduct.CURR_CD
        // ERP 的字段名直接出现在你的领域逻辑里
    }
}
```

加防腐层：

```java
// 你的领域定义自己的接口
package com.example.order.domain.gateway;

public interface ProductGateway {
    Product find(SkuId skuId);
}

public record Product(SkuId id, String name, Money unitPrice) {}
```

```java
// infrastructure 层实现，做 ERP 模型 → 领域模型的翻译
package com.example.order.infrastructure.gateway;

@Component
public class ErpProductGateway implements ProductGateway {
    private final ErpSoapClient erpClient;

    public Product find(SkuId skuId) {
        ErpProductDto erp = erpClient.queryProduct(skuId.value());
        return translate(erp);
    }

    private Product translate(ErpProductDto erp) {
        // 防腐层的核心：把 ERP 的诡异模型翻译成你的领域模型
        Currency currency = mapCurrency(erp.getCURR_CD());
        Money price = new Money(erp.getUNIT_PRC_AMT(), currency);
        return new Product(new SkuId(erp.getPROD_ID()), erp.getPROD_NM(), price);
    }

    private Currency mapCurrency(String erpCode) {
        return switch (erpCode) {
            case "01" -> Currency.CNY;
            case "02" -> Currency.USD;
            default -> throw new IllegalStateException("unknown ERP currency: " + erpCode);
        };
    }
}
```

ACL 和 Adapter 的区别：
- **Adapter** 关注接口适配（接口 A 转接口 B）
- **ACL** 关注模型隔离（外部模型 → 内部模型 + 业务语义校正）

ACL 比 Adapter 更"重"：它可能包含业务规则校正（如 ERP 给的 0.99 元当成"促销价"，但你的领域只接受"原价 + 折扣"）。这种语义层面的翻译，是 Adapter 模式不负责的。

> **口诀**：ACL 是给外部模型的"过滤器"。任何接入第三方/老系统都默认上 ACL。

### 3.6 CQRS 的读写分离实现

写侧（用聚合）：

```java
@Service
public class OrderApplicationService {
    @Transactional
    public OrderId placeOrder(PlaceOrderCommand cmd) {
        Order order = Order.create(cmd.customerId(), cmd.address());
        for (var line : cmd.lines()) {
            Product product = productGateway.find(line.skuId());
            order.addLine(product.id(), line.quantity(), product.unitPrice());
        }
        order.submit();
        orderRepo.save(order);
        return order.getId();
    }
}
```

读侧（直接查、扁平 DTO）：

```java
package com.example.order.application.query;

public record OrderDetailDTO(
    String orderId, String customerName, String status,
    BigDecimal totalAmount, String currency,
    List<OrderLineDTO> lines, String shippingAddress
) {}

@Service
public class OrderQueryService {
    private final JdbcTemplate jdbc;

    public OrderDetailDTO getOrderDetail(String orderId) {
        // 直接 SQL，一次 join 出来，不走聚合
        String sql = """
            SELECT o.id, c.name, o.status, o.total_amount, o.currency,
                   o.shipping_address
            FROM t_order o
            LEFT JOIN t_customer c ON c.id = o.customer_id
            WHERE o.id = ?
            """;
        // ...
    }

    public Page<OrderListItemDTO> listOrders(OrderQuery query, Pageable page) {
        // 读侧的查询条件可以非常灵活，按业务方需求随便加
        // 写侧的聚合保护一致性，读侧的 DTO 优化展示
    }
}
```

读模型刷新（如果写读异库）：

```java
@Component
public class OrderReadModelUpdater {
    @TransactionalEventListener(phase = AFTER_COMMIT)
    public void on(OrderConfirmedEvent event) {
        // 把变更同步到读库（或直接写 ES、Redis）
        readDb.update("UPDATE order_view SET status = 'SHIPPED' WHERE id = ?",
            event.orderId().value());
    }
}
```

CQRS 落地的最大坑：**最终一致延迟**。用户下完单立刻查询自己的订单，可能查不到（读库还没同步）。两种应对：
- **关键查询读主库**：核心查询（"我刚下的单"）走写库
- **回执机制**：下单返回临时单号，前端轮询查询

> **口诀**：写聚合，读扁平。写读异库时，关键路径要么读主库，要么前端等同步。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 贫血模型 —— "业务逻辑全堆在 Service 里"

#### 现象

电商订单服务，`OrderService` 类有 2000 行，每个 method 50~100 行。代码评审时大家都在改这个类，每次发版都有冲突。新人接手要花两周才能搞清楚一个状态如何流转。

#### 根因

典型的"贫血 DDD"：

```java
// 贫血的 Order 类（其实是个 DTO + ORM 映射）
public class Order {
    private Long id;
    private String customerId;
    private Integer status;
    private BigDecimal totalAmount;
    // ... 30 个字段，全是 getter/setter
}

// 业务全在 Service
@Service
public class OrderService {
    public void confirmOrder(Long orderId) {
        Order order = orderMapper.selectById(orderId);
        if (order.getStatus() != 2) {           // 业务规则在 Service 里
            throw new IllegalStateException();
        }
        order.setStatus(3);                      // 直接改字段
        order.setUpdatedAt(LocalDateTime.now());
        orderMapper.update(order);
        // 又一段业务逻辑：发短信
        smsService.send(order.getCustomerId(), "订单已确认");
        // 又一段业务逻辑：扣库存
        inventoryService.decrease(order.getSkuId(), order.getQty());
    }
    // ... 30 个类似的方法
}
```

问题：业务逻辑散落在 Service 各处，状态转换没有统一约束（任何 Service 都能 `setStatus(任意值)`）。聚合的"内部一致性"完全没有，本质上是面向数据库编程，不是面向领域编程。

#### 修复

把业务逻辑搬进 Order 类：

```java
public class Order {
    private OrderId id;
    private OrderStatus status;
    // ...

    // 状态转换由聚合自己负责
    public void confirm() {
        if (this.status != OrderStatus.PAID)
            throw new IllegalStateException("only PAID can confirm");
        this.status = OrderStatus.SHIPPED;
        registerEvent(new OrderConfirmedEvent(this.id));
    }
}

@Service
public class OrderApplicationService {
    @Transactional
    public void confirmOrder(OrderId id) {
        Order order = orderRepo.find(id);
        order.confirm();                  // 业务规则在聚合里
        orderRepo.save(order);
        order.pullEvents().forEach(publisher::publishEvent);
        // 短信、扣库存都通过事件订阅触发，不在主流程里
    }
}
```

效果：
- Service 缩短到 10 行以下
- 状态转换的非法路径被聚合的 if 拦住
- 短信、扣库存解耦到事件订阅者

#### 验证

```java
@Test
void cannotConfirmUnpaidOrder() {
    Order order = Order.create(...);
    assertThrows(IllegalStateException.class, order::confirm);
}

@Test
void confirmEmitsEvent() {
    Order order = Order.create(...);
    order.submit();   // 走到 PAID
    order.confirm();
    List<DomainEvent> events = order.pullEvents();
    assertTrue(events.stream().anyMatch(e -> e instanceof OrderConfirmedEvent));
}
```

> **口诀**：业务规则属于领域对象，不属于 Service。Service 长就是病。

---

### 案例 ② Aggregate 太大导致并发冲突 —— "大订单更新经常乐观锁失败"

#### 现象

商家批量改价场景。一个订单聚合里包含 100~500 行 OrderLine。客服每改一行价格就要更新整个聚合（聚合用乐观锁），并发改价时 80% 的请求触发 OptimisticLockException。

#### 根因

聚合定义不当：

```java
public class Order {
    private OrderId id;
    private List<OrderLine> lines;    // 几百行
    @Version private Long version;
    // 任何 line 改动都要重写整个 Order，version+1
}
```

任意 line 改动都竞争同一个 version。500 个客服同时改 500 个 line，全部排队等乐观锁。

更深层：这种"几百个 line 共享一个 version"的设计，是 Aggregate 边界划得太大。OrderLine 之间在业务上其实是独立的 —— 客服 A 改第 1 行不应该影响客服 B 改第 100 行。

#### 修复

**方案 A：拆聚合**。如果 OrderLine 在业务上能独立操作，就把它升格为独立聚合：

```java
public class Order {                    // 聚合根：订单元信息
    private OrderId id;
    private CustomerId customerId;
    private OrderStatus status;
    @Version private Long version;
    // 不再持 lines
}

public class OrderLine {                // 独立聚合根
    private OrderLineId id;
    private OrderId orderId;            // 引用，不持对象
    private SkuId sku;
    private int quantity;
    private Money unitPrice;
    @Version private Long version;
}

public class OrderLineRepository { ... }
```

代价：失去"一次加载整个订单"的便利。但这正是 Aggregate 设计的关键 trade-off。

**方案 B：拆 version**（如果 OrderLine 必须在 Order 聚合内）：把 version 加到 OrderLine 上，Order 不再用乐观锁，但这就要小心跨行的一致性约束。

**方案 C：CQRS 拆读写**：写时按聚合，读时按 join 平铺。改价命令分发到对应的 OrderLine 聚合，读侧给客服一个 join 后的视图。

#### 验证

并发测试：

```java
@Test
void concurrentLineUpdatesShouldNotConflict() throws Exception {
    OrderId orderId = createOrderWith500Lines();
    ExecutorService pool = Executors.newFixedThreadPool(50);
    CountDownLatch done = new CountDownLatch(500);
    AtomicInteger failures = new AtomicInteger();
    for (int i = 0; i < 500; i++) {
        int idx = i;
        pool.submit(() -> {
            try {
                orderLineService.updatePrice(orderId, idx, new Money(...));
            } catch (OptimisticLockException e) {
                failures.incrementAndGet();
            } finally { done.countDown(); }
        });
    }
    done.await();
    assertTrue(failures.get() < 10, "拆聚合后并发冲突应该 < 10");
}
```

> **口诀**：聚合大 = 锁热点。如果聚合内 Entity 能独立操作，考虑拆聚合。

---

### 案例 ③ Aggregate 太小导致一致性破坏 —— "订单和订单行不一致"

#### 现象

线上偶发"订单总金额 ≠ 各订单行之和"的脏数据。审计报表跑出来对不上，财务追责。

#### 根因

矫枉过正后的反向案例：把 OrderLine 拆成独立聚合，没人保护"总金额 = 行金额之和"这个不变量。

```java
// 拆完后：
public class Order { Money totalAmount; }
public class OrderLine { Money subtotal; }

@Service
public class OrderLineService {
    @Transactional
    public void updatePrice(OrderLineId id, Money newPrice) {
        OrderLine line = lineRepo.find(id);
        line.changePrice(newPrice);
        lineRepo.save(line);
        // ★ 忘了更新 Order.totalAmount
    }
}
```

订单行的小聚合改了，但订单的总金额没跟着改。"总金额 = 行金额之和"是个跨聚合的一致性，必须有人维护。

#### 修复

**方案 A：合回一个聚合**（最强一致）

如果总金额一致性比并发性能更重要，OrderLine 应该回归到 Order 聚合内部，通过聚合根的方法来改价：

```java
public class Order {
    public void changeLinePrice(OrderLineId lineId, Money newPrice) {
        OrderLine line = findLine(lineId);
        line.changePrice(newPrice);
        recalculateTotal();           // 聚合内部维护一致性
    }
}
```

**方案 B：用 Domain Event 做最终一致**（高并发场景）

OrderLine 改价后发事件，订单聚合订阅事件重新计算：

```java
public class OrderLine {
    public void changePrice(Money newPrice) {
        Money oldSubtotal = this.subtotal;
        this.unitPrice = newPrice;
        this.subtotal = newPrice.multiply(this.quantity);
        registerEvent(new OrderLinePriceChangedEvent(
            this.orderId, this.id, oldSubtotal, this.subtotal));
    }
}

@Component
class OrderTotalRecalculator {
    @TransactionalEventListener(phase = AFTER_COMMIT)
    public void on(OrderLinePriceChangedEvent event) {
        Order order = orderRepo.find(event.orderId());
        Money diff = event.newSubtotal().subtract(event.oldSubtotal());
        order.adjustTotal(diff);     // 增量调整，避免全量重算
        orderRepo.save(order);
    }
}
```

注意"最终一致"的代价：在事件处理完成前，Order.totalAmount 是过期的。如果有外部接口立即读，要么读时重算，要么接受短暂不一致。

#### 验证

```java
@Test
void totalAmountMustEqualSumOfLines() {
    OrderId id = createOrder();
    addLine(id, sku1, 2, money(100));
    addLine(id, sku2, 3, money(50));
    updateLinePrice(id, line1, money(120));
    eventually(() -> {       // 给最终一致一些时间
        Order order = orderRepo.find(id);
        Money sum = order.getLines().stream()
            .map(OrderLine::subtotal)
            .reduce(Money.zero(), Money::add);
        assertEquals(sum, order.getTotalAmount());
    });
}
```

> **口诀**：不变量决定聚合边界。"总金额 = 行金额之和"这种不变量必须有人守，要么强一致（合聚合），要么最终一致（订事件）。

---

### 案例 ④ Bounded Context 划分不当 —— "微服务间循环调用"

#### 现象

公司微服务化后，订单服务调商品服务拿价格、商品服务调订单服务统计销量、订单服务又调用户服务取地址、用户服务调订单服务取最近订单……整个调用图是一团乱麻。每发一次版要协调五个团队。

#### 根因

按"实体名词"划分服务，而不是按 Bounded Context：
- 订单服务 → Order 表
- 商品服务 → Product 表
- 用户服务 → User 表
- 地址服务 → Address 表

每个服务暴露 CRUD API，业务方调用方就需要"东拼西凑"才能完成一个用例 → 必然循环依赖。

正确的视角应该是：**按业务能力划 BC，而不是按数据实体划服务**。

#### 修复

按业务能力重划：

```
订单履约 BC：订单创建、订单流转、订单查询
   ↳ 这个 BC 自己持有"下单时的商品快照"、"用户地址快照"
   
商品中心 BC：商品定义、SKU、价格
   ↳ 通过领域事件广播"商品上架/下架/调价"，不主动调订单
   
用户中心 BC：用户信息、地址簿
   ↳ 提供查询接口供下单时拷贝快照

履约 BC 在下单瞬间从商品 BC 拿快照存起来；
后续订单状态流转完全在自己的 BC 内闭环；
不需要回查商品 BC、不需要回查用户 BC。
```

代码层面：订单聚合内**冗余存储**下单时的商品名、地址等：

```java
public class Order {
    private CustomerId customerId;       // 引用
    private Address shippingAddress;     // VO，下单时的快照
    private List<OrderLine> lines;
}

public class OrderLine {
    private SkuId sku;                   // 引用
    private String skuName;              // 下单时的快照
    private Money unitPrice;             // 下单时的快照
}
```

**为什么允许冗余？** 因为订单是个"在那个时间点发生的事实"，商品后来改名了、调价了，都不应该影响已下单。冗余快照既保证业务正确，又解耦了对商品服务的强依赖。

#### 验证

调用图分析：

```
重划前：
  订单 → 商品（取价格）
  订单 → 用户（取地址）  
  商品 → 订单（统计销量）
  用户 → 订单（最近订单）

重划后：
  订单 → 商品（仅下单瞬间，取快照）
  订单 → 用户（仅下单瞬间，取地址）
  数据分析 BC ← (订阅) 订单事件、商品事件
  
  无循环依赖
```

部署独立性：订单服务改动不再需要商品/用户服务配合发版。

> **口诀**：BC 按业务能力划，不按数据实体划。订单 BC 持有快照，不要每次回查。

---

### 案例 ⑤ 没用最终一致 —— "业务直接跨聚合改数据"

#### 现象

下单接口偶发"超卖"。库存明明只有 10 件，最后卖出 12 件。

#### 根因

```java
@Service
public class OrderService {
    @Transactional
    public OrderId placeOrder(PlaceOrderCommand cmd) {
        // 同一个事务里改两个聚合
        Inventory inv = inventoryRepo.find(cmd.skuId());
        inv.decrease(cmd.quantity());           // ★ 跨聚合
        inventoryRepo.save(inv);

        Order order = Order.create(...);
        orderRepo.save(order);
        return order.getId();
    }
}
```

看起来"事务保护一切"，但实际上：
- Inventory 用乐观锁（`@Version`），并发下单时大量请求重试
- 重试期间事务提交，超过一致性窗口
- 极端情况下两个事务都通过乐观锁检查（时序问题），出现超卖

更深层：库存和订单是**两个 BC** 的概念。一个事务跨两个 BC 是错的。

#### 修复

用 Saga 模式（详见 [分布式事务](/article/distributed/dist-tx)）：

```java
// Step 1: 订单服务先创建一个"待扣减"的订单
@Transactional
public OrderId placeOrder(PlaceOrderCommand cmd) {
    Order order = Order.create(cmd);     // 状态：CREATED
    orderRepo.save(order);
    publisher.publish(new OrderCreatedEvent(order.getId(), cmd.skuId(), cmd.quantity()));
    return order.getId();
}

// Step 2: 库存服务订阅事件，扣库存
@Component
class InventoryReducer {
    @KafkaListener(topics = "order-created")
    public void on(OrderCreatedEvent event) {
        try {
            Inventory inv = inventoryRepo.find(event.skuId());
            inv.decrease(event.quantity());      // 库存聚合自己保证不超卖
            inventoryRepo.save(inv);
            publisher.publish(new InventoryReducedEvent(event.orderId()));
        } catch (InsufficientStockException e) {
            publisher.publish(new InventoryReductionFailedEvent(event.orderId(), e.getMessage()));
        }
    }
}

// Step 3: 订单服务订阅库存结果事件
@Component
class OrderStateUpdater {
    @KafkaListener(topics = "inventory-reduced")
    public void on(InventoryReducedEvent event) {
        Order order = orderRepo.find(event.orderId());
        order.confirmInventory();            // 状态：INVENTORY_RESERVED
        orderRepo.save(order);
    }
    
    @KafkaListener(topics = "inventory-reduction-failed")
    public void on(InventoryReductionFailedEvent event) {
        Order order = orderRepo.find(event.orderId());
        order.cancel("inventory not enough");
        orderRepo.save(order);
    }
}
```

库存聚合内部保证不超卖（用 DB 唯一约束 / SELECT FOR UPDATE）。订单聚合按事件流转状态。**没有跨聚合事务**。

#### 验证

```java
@Test
void noOversoldInConcurrentOrders() throws Exception {
    setInventory(skuId, 10);
    ExecutorService pool = Executors.newFixedThreadPool(50);
    AtomicInteger success = new AtomicInteger();
    CountDownLatch done = new CountDownLatch(20);
    for (int i = 0; i < 20; i++) {
        pool.submit(() -> {
            try {
                OrderId id = orderService.placeOrder(...);
                eventually(() -> {
                    Order o = orderRepo.find(id);
                    if (o.getStatus() == OrderStatus.INVENTORY_RESERVED) {
                        success.incrementAndGet();
                    }
                });
            } finally { done.countDown(); }
        });
    }
    done.await();
    assertTrue(success.get() <= 10);
}
```

> **口诀**：跨聚合一致性用事件 + Saga，不要塞进一个事务。一个事务只改一个聚合。

---

### 案例 ⑥ Repository 实现里写了业务逻辑 —— "查询条件越加越乱"

#### 现象

`JpaOrderRepository` 类有 800 行，里面 30 个方法：`findByCustomerAndStatusAndDateBetween`、`findByCustomerWithLinesAndShippingAddress`……每加一个新查询场景都要新增方法。

#### 根因

```java
// ❌ Repository 里塞各种业务查询
public interface OrderRepository {
    Order find(OrderId id);
    void save(Order order);
    
    // 这些都跑进了 Repository
    List<Order> findByCustomerAndStatus(CustomerId id, OrderStatus s);
    List<Order> findOverdueOrders();
    int countByCustomerThisMonth(CustomerId id);
    BigDecimal sumAmountByCustomer(CustomerId id, LocalDate from, LocalDate to);
    Page<Order> searchByMultipleCriteria(OrderSearchCriteria criteria, Pageable page);
}
```

Repository 的本意是"聚合的集合假象"，只该有 CRUD-like 操作。一旦塞进各种业务查询，就变成了第二个 Service 类。

#### 修复

把"读"操作移到 Query Service（CQRS 思路）：

```java
// Repository 只保留聚合 CRUD
public interface OrderRepository {
    Order find(OrderId id);
    Optional<Order> findOptional(OrderId id);
    void save(Order order);
    void remove(Order order);
    // 最多保留几个简单的"按主键属性查"
    List<Order> findByCustomerId(CustomerId id);
}

// 复杂查询全在 Query Service
@Service
public class OrderQueryService {
    private final JdbcTemplate jdbc;     // 直接 JDBC，不用聚合

    public List<OrderListDTO> findOverdueOrders() { ... }
    public OrderStatisticsDTO calculateStatistics(CustomerId id, DateRange range) { ... }
    public Page<OrderListDTO> search(OrderSearchCriteria c, Pageable p) { ... }
}
```

为什么读不通过聚合？
- 读场景的查询需求多变，每变一次就改一次聚合，污染领域模型
- 读场景的性能要求高，加载完整聚合往往是浪费
- 读场景的 DTO 形状跟聚合形状不同（要 join、要冗余）

#### 验证

代码评审 checklist：
- Repository 接口方法数 ≤ 6（CRUD + 1~2 个按主属性查）
- 任何"展示 / 报表 / 列表"查询都进 Query Service
- Query Service 不修改任何状态

> **口诀**：Repository 给写侧用，Query Service 给读侧用。不要把 Query 塞进 Repository。

---

### 案例 ⑦ DDD 落地"重型化" —— "小团队学不会"

#### 现象

5 人的小团队，业务是个简单的 CRUD 后台。架构师强行上 DDD，每个表创建 6~8 个文件（聚合根、VO、仓储接口、仓储实现、PO、转换器、应用服务、命令对象）。开发效率从一周一个 feature 掉到两周一个。新人 onboard 两个月还没真正提交代码。

#### 根因

把 DDD 当成"必须全套上"的教条。现实是 DDD 的成本只有在**领域复杂、团队成熟、长期维护**时才能 ROI 为正。

判断 DDD 适用度的检查：
- **领域复杂度**：业务规则多、状态流转复杂、长期演化 → 适合
- **团队规模**：> 10 人，能承受学习曲线 → 适合
- **维护周期**：项目活 5 年以上 → 适合
- **业务方在场**：业务方愿意参与 Event Storming → 适合

如果上面 4 项有 2 项不满足，**别强上 DDD**。

#### 修复

按 Subdomain 分类区别对待：
- **Core Domain**：上 DDD（充血模型、Aggregate、Domain Event）
- **Supporting Subdomain**：用"轻 DDD"（包结构分层 + Service 编排，但允许贫血）
- **Generic Subdomain**：直接 CRUD，别上 DDD

5 人小团队 + 简单业务，路线选择：
- 用包结构表达领域边界（即使不上聚合）
- 命名遵循统一语言
- 业务逻辑能写在领域对象里就写，写不动就放 Service
- 重要状态机用枚举 + 状态转换矩阵守护

```java
// 轻型 DDD 示例：业务逻辑在领域对象，但没有聚合根/事件等重型组件
public class Order {
    private Long id;
    private OrderStatus status;
    private BigDecimal totalAmount;
    
    // 业务方法直接挂在 Entity 上
    public void confirm() {
        if (this.status != OrderStatus.PAID)
            throw new IllegalStateException();
        this.status = OrderStatus.SHIPPED;
    }
}

@Service
public class OrderService {
    @Transactional
    public void confirmOrder(Long id) {
        Order order = orderRepo.findById(id).orElseThrow();
        order.confirm();
        orderRepo.save(order);
    }
}
```

不强求 ACL、不强求 CQRS、不强求 Domain Event。能跑、好维护就够了。

#### 验证

"DDD 重型程度"自检：
- 实现一个简单 feature 要新建文件数 ≤ 4 → 健康
- > 8 → 重型化预警，考虑简化

团队效率指标：
- 新人 1 周内能独立提交 PR → 合理
- > 1 个月还学不会 → DDD 用法有问题

> **口诀**：DDD 是工具，不是教条。Core Domain 上重型，其他场景轻型化。

---

### 案例 ⑧ ORM 与充血模型冲突 —— "JPA 强制无参构造，VO 不能 final"

#### 现象

想用充血模型 + 不可变 VO，但 JPA 要求 `@Entity` 有无参构造、setter 可访问，VO 字段不能 final。一妥协就发现：所有 setter 都暴露了 → 任何人都能 `order.setStatus(...)` 绕过业务方法。

#### 根因

JPA 是 EJB 时代的设计，假设实体是"被框架填充"的 POJO。这和 DDD 的"实体自己保护不变量"哲学根本上冲突：
- JPA 要求字段可写（默认 setter 或 field 反射）
- JPA 关联映射用集合，破坏聚合内的封装
- JPA 的 `@Id` 通常是 `Long`，DDD 倾向于强类型 ID（`OrderId`）

#### 修复

**方案 A：双层模型**（最纯净，最重）

领域对象和 PO 完全分离：

```java
// domain 层：纯净的领域对象
public class Order {
    private final OrderId id;
    private OrderStatus status;
    // 私有构造、工厂方法、业务方法
    private Order(OrderId id) { this.id = id; }
    public static Order create(...) { ... }
}

// infrastructure 层：JPA 实体
@Entity
@Table(name = "t_order")
class OrderJpaEntity {
    @Id private String id;
    private String status;
    // setter/getter
}

// 仓储实现里手动转换
@Repository
public class JpaOrderRepository implements OrderRepository {
    public Order find(OrderId id) {
        OrderJpaEntity po = jpaRepo.findById(id.value()).orElseThrow();
        return OrderReconstructor.reconstruct(po);   // PO -> DO
    }
    public void save(Order order) {
        OrderJpaEntity po = OrderConverter.toPo(order);   // DO -> PO
        jpaRepo.save(po);
    }
}
```

代价：每个聚合多两个文件（PO + 转换器），改字段时要改三处。

**方案 B：用支持 DDD 的 ORM**

- **MyBatis**：手写 SQL + 手写 ResultHandler，DO 可以是任意结构
- **JOOQ**：类型安全的 SQL DSL，DO 不受框架约束
- **Spring Data JDBC**：比 JPA 简单，对 DDD 更友好

```java
// Spring Data JDBC，DO 可以保留私有构造、不可变 VO
@Table("t_order")
public class Order {
    @Id private final OrderId id;
    private OrderStatus status;
    @MappedCollection(idColumn = "order_id") private List<OrderLine> lines;
}
```

**方案 C：JPA + 妥协**

接受 JPA 的限制，但用以下手段缓解：
- 字段用 `protected` 而非 `public` setter
- VO 用 `@Embeddable` + 私有 setter
- 关联用 `@OneToMany(access = AccessLevel.PRIVATE)`
- 业务方法挂在 Entity 上，setter 仅供 JPA 反射用

```java
@Entity
public class Order extends AggregateRoot {
    @Id @Column private String id;
    
    @Enumerated(STRING)
    @Column(name = "status")
    private OrderStatus status;
    
    // 包私有的 setter，仅 JPA 用，业务代码看不到
    void setStatus(OrderStatus s) { this.status = s; }
    
    // 业务方法
    public void confirm() {
        if (this.status != OrderStatus.PAID) throw ...;
        this.status = OrderStatus.SHIPPED;     // 直接改字段，不走 setter
    }
}
```

#### 验证

"业务规则被绕过"测试：

```java
@Test
void cannotChangeStatusDirectly() {
    Order order = orderRepo.find(...);
    // 编译期就该过不去
    // order.setStatus(OrderStatus.SHIPPED);   // 应该 compile error 或 IDE warning
}
```

如果还能直接 setStatus，就是封装失败。

> **口诀**：JPA 和 DDD 是天敌。要么双层映射、要么换 ORM、要么忍痛包私有 setter。

---

### 案例 ⑨ Event Sourcing 的快照与重放性能

#### 现象

上线 Event Sourcing 半年后，某些聚合（如热门用户的账户）有 50000+ 事件。每次加载耗时 800ms+。监控显示读放大 100 倍。

#### 根因

Event Sourcing 的读模型 = 重放所有事件。事件越多，重放越慢。如果不做 Snapshot，事件数量增长 → 加载时间线性增长。

```java
public Account loadAccount(AccountId id) {
    Account account = Account.createEmpty(id);
    List<DomainEvent> events = eventStore.allEvents(id);   // 50000 个
    for (DomainEvent e : events) {
        account.apply(e);                                    // 50000 次方法调用
    }
    return account;
}
```

#### 修复

**Snapshot 策略**：每 N 个事件存一份当前状态快照。重放时从最近的 snapshot 开始。

```java
public Account loadAccount(AccountId id) {
    Snapshot snapshot = snapshotRepo.latest(id);
    Account account;
    long fromVersion;
    if (snapshot != null) {
        account = Account.fromSnapshot(snapshot);
        fromVersion = snapshot.version();
    } else {
        account = Account.createEmpty(id);
        fromVersion = 0;
    }
    
    // 只重放 snapshot 之后的事件
    List<DomainEvent> events = eventStore.eventsAfter(id, fromVersion);
    for (DomainEvent e : events) {
        account.apply(e);
    }
    return account;
}

// 每 1000 个事件存一次 snapshot
@Component
class SnapshotTaker {
    @EventListener
    public void afterEventStored(EventStored event) {
        if (event.version() % 1000 == 0) {
            Account account = loadAccount(event.aggregateId());
            snapshotRepo.save(account.toSnapshot());
        }
    }
}
```

效果：50000 事件 / 1000 = 平均加载时只重放 500 个事件。

**进阶问题：Schema 演化**

老事件结构改了怎么办？三种策略：
- **Upcaster**：加载老事件时即时翻译成新结构
- **事件版本号**：每个事件带 schemaVersion，apply 方法按版本分支
- **重写历史**：极端情况下，把所有老事件按新结构 dump 后重新存（影响审计语义，慎用）

```java
public class AccountDepositedEvent implements DomainEvent {
    int schemaVersion;
    BigDecimal amount;
    Currency currency;       // v2 才有
    
    public static AccountDepositedEvent fromBytes(byte[] bytes) {
        var raw = parse(bytes);
        if (raw.schemaVersion == 1) {
            // v1 没有 currency 字段，假设是 CNY
            return new AccountDepositedEvent(2, raw.amount, Currency.CNY);
        }
        return raw;
    }
}
```

#### 验证

```java
@Test
void loadShouldUseSnapshot() {
    AccountId id = AccountId.next();
    for (int i = 0; i < 5000; i++) {
        deposit(id, money(1));
    }
    
    long start = System.nanoTime();
    Account account = repo.find(id);
    long elapsedMs = (System.nanoTime() - start) / 1_000_000;
    
    assertTrue(elapsedMs < 100, "加载应在 100ms 内（依赖 snapshot）");
    assertEquals(money(5000), account.balance());
}
```

> **口诀**：ES 必配 Snapshot，按事件量定频率。Schema 演化用 Upcaster 兜底。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 战略层
- [ ] 每个新模块开工前先识别 Bounded Context（不是直接画类图）
- [ ] 统一语言落到代码命名（类名、方法名、字段名、API 字段）
- [ ] Subdomain 分类清晰：Core / Supporting / Generic 不混治
- [ ] 跨 BC 接入第三方系统默认上 ACL
- [ ] BC 边界与微服务边界对齐（一对一为主，BC 内部可拆多服务）
- [ ] Context Map 定期 review（半年一次）

### 战术层
- [ ] Aggregate Root 内部强一致，跨 Aggregate 用 Domain Event 最终一致
- [ ] 一个事务只修改一个 Aggregate
- [ ] Aggregate 通过 ID 引用其他 Aggregate，不持对象引用
- [ ] Aggregate 体量适中（10 量级 Entity）
- [ ] Value Object 必须不可变（final 字段、无 setter）
- [ ] Entity 比较 equals 只看 ID
- [ ] 业务规则写在领域对象里，不写在 Service 里（拒绝贫血）
- [ ] Repository 接口在 domain 层，实现在 infrastructure 层
- [ ] Repository 接口只暴露 CRUD-like 操作（≤ 6 个方法）
- [ ] 复杂查询走 Query Service（CQRS）

### 事件与一致性层
- [ ] Domain Event 命名用过去时（OrderConfirmed 而非 ConfirmOrder）
- [ ] 事件发布走 Outbox Pattern（避免事务回滚导致脏事件）
- [ ] 跨 BC 通信优先事件，不直接 RPC 调
- [ ] 关键查询的最终一致延迟有兜底（读主库 / 前端轮询）
- [ ] Saga 流程有补偿动作设计

### 工程层
- [ ] 包结构按 domain / application / infrastructure / interfaces 四层
- [ ] domain 层不依赖 Spring / JPA / 任何技术框架
- [ ] application 层不写业务规则（只编排）
- [ ] infrastructure 层实现 domain 定义的接口
- [ ] JPA 与领域模型有清晰边界（双层映射或包私有 setter）

### CQRS 与 ES 层
- [ ] CQRS 仅在读写差异显著时上（不为了 CQRS 而 CQRS）
- [ ] CQRS 异库时关键查询有"读主库"兜底
- [ ] Event Sourcing 仅在强审计 / 时间旅行场景上
- [ ] ES 必配 Snapshot
- [ ] ES 事件结构演化有 Upcaster 兜底
- [ ] ES 事件存储有备份策略

### 落地强度层
- [ ] DDD 强度匹配子域类型：Core 上重型、Supporting 轻型、Generic 不上
- [ ] 新人 onboard 1 周内能提 PR
- [ ] 实现一个简单 feature 新建文件 ≤ 4 个
- [ ] 业务方参与统一语言定义（Event Storming 或类似）

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### 架构模式
- **Hexagonal Architecture（六边形架构）**：Ports & Adapters，把"领域核心"放在中心、外部依赖通过 Port/Adapter 接入。和 DDD 完美契合，是 DDD 落地最自然的物理结构。
- **Clean Architecture**：Uncle Bob 提出的同心圆架构。和六边形本质相同，强调"依赖只能从外层指向内层"。
- **Onion Architecture**：Jeffrey Palermo 提出，强调"领域模型在最内层"。和六边形/Clean 是同一思路的不同表达。

### Event Storming 实战
- 完整流程：Big Picture → Process Modelling → Software Design 三阶段
- 远程团队怎么搞 Event Storming（Miro / Mural 工具）
- 输出物如何转为代码骨架（命令 → Application Service 方法、事件 → Domain Event 类、聚合候选 → Aggregate Root）

### Strategic DDD 与企业架构
- DDD 与 TOGAF / Zachman 的对照
- Bounded Context 在中台架构中的对应（业务中台、数据中台）
- 大型企业的"BC 治理"实践（华为业务对象建模、阿里中台拆分）

### DSL 与 DDD
- 内部 DSL 表达领域规则（Kotlin / Groovy 流式 API）
- 外部 DSL 表达业务流程（BPMN、规则引擎）
- 何时该上 DSL：业务方需要修改规则时

### DDD 与微服务的演进路径
- **从单体出发**：先按 BC 划包，将来再拆服务
- **绞杀者模式**（Strangler Fig）：老系统外面包一层新 BC，逐步迁移
- **API Gateway**：BFF 模式 + GraphQL 应对前端多 BC 聚合需求

### Saga 与分布式一致性
- Choreography Saga（事件驱动）vs Orchestration Saga（中央编排）
- TCC 与 Saga 的边界
- 详见 [分布式事务](/article/distributed/dist-tx)

### DDD 与函数式
- Scott Wlaschin 的《Domain Modeling Made Functional》—— 用 F# 类型系统表达领域不变量
- Java/Kotlin 中用 Sealed Class 表达"代数数据类型"
- "Make illegal states unrepresentable"：用类型系统而不是 if 来守护规则

---

## 7. 引用

- Eric Evans, *Domain-Driven Design: Tackling Complexity in the Heart of Software*（2003，蓝皮书 / 原典）
- Vaughn Vernon, *Implementing Domain-Driven Design*（2013，红皮书 / 实践指南）
- Vaughn Vernon, *Domain-Driven Design Distilled*（2016，简版入门）
- Scott Millett & Nick Tune, *Patterns, Principles, and Practices of Domain-Driven Design*（2015）
- Alberto Brandolini, [Introducing Event Storming](https://www.eventstorming.com/)
- Martin Fowler, [Bounded Context](https://martinfowler.com/bliki/BoundedContext.html) / [DDD Aggregate](https://martinfowler.com/bliki/DDD_Aggregate.html)
- Greg Young, [CQRS Documents](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf)
- Chris Richardson, *Microservices Patterns*（2018，Saga / Outbox 模式权威讲解）
- Scott Wlaschin, *Domain Modeling Made Functional*（2018，F# 视角）
- Khononov, *Learning Domain-Driven Design*（2021，现代 DDD 落地）

---

> 配套：[分布式事务](/article/distributed/dist-tx) | [系统设计主题](/article/system-design/) | [API 设计](/article/architecture/api-design)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你觉得这个深度对，告诉我下一篇该写哪个。
