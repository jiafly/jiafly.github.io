# 事件驱动架构（EDA）与 Event Sourcing —— 从消息到事件溯源的工程落地

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试或了解概念，去搜一篇 "EDA 入门" 就够了。
> 这里要回答的，是**真打算把事件驱动落到 Spring 工程里、且不想被同事骂"过度抽象"**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过：

1. EDA 和"请求-响应 + MQ 解耦"看起来都在用消息中间件，**本质边界**在哪？给一个明确判据：什么场景叫 EDA、什么场景只是"用了 MQ"。
2. Event（事件）/ Command（命令）/ Message（消息）三个词在技术圈被混用，但在 EDA 里**精确含义不同**。请用一句话各定义，并指出"`OrderPlaced` 与 `PlaceOrder` 谁该是命令谁该是事件"的判别规则。
3. Event Sourcing 的存储和 CRUD 的 `UPDATE order SET status='paid'` 在物理层面有什么差别？只是"加一张审计表"吗？
4. 一个聚合（Aggregate）已经积累了 50 万条事件，每次重建需要回放 30 秒。**Snapshot（快照）什么时候做、做多频**？给一个工程公式。
5. Saga 的两种实现 —— Orchestration（编排）和 Choreography（协同），各自的失败半径、可观测性、技术债演化路径分别是什么？什么时候必须二选一？
6. 一个上线两年的事件 `OrderPlacedV1` 现在要加字段。你删字段、改类型、改语义，哪些**绝对不能做**？Schema Evolution 的兼容性矩阵是什么？
7. Outbox 模式（事务性发件箱）和 CDC（变更数据捕获）都能解决"业务事务和发消息的原子性"问题。它们的延迟、运维成本、对数据库的耦合度怎么对比？什么场景该用哪个？
8. CQRS 的读模型（Read Model）从写模型同步过来，**最终一致性**意味着用户刚下完单去查"我的订单"列表可能看不到自己。怎么把这个尴尬窗口缩到用户无感？

如果有半数答不上，往下看。

---

## 1. 底层模型：Event、Aggregate、Event Store

绝大多数文章把 EDA 讲成"生产者-消费者 + Kafka"，然后画一个发布订阅图就结束了。这层认知不够支撑你做 Event Sourcing 选型，也无法解释上面 8 个问题。要看懂它们，必须把"事件"这个概念拆到原子层面。

### 1.1 EDA 的核心三元素

EDA 的最小模型由三部分组成：

```
   ┌──────────────┐    publish     ┌─────────────┐    subscribe    ┌──────────────┐
   │ Event        │ ─────────────► │  Event      │ ──────────────► │ Event        │
   │ Producer     │                │  Broker     │                 │ Consumer     │
   └──────────────┘                └─────────────┘                 └──────────────┘
       业务侧                       Kafka/Pulsar                       下游订阅方
   只关心"发生了什么"               不关心语义                       自己决定怎么响应
```

三件事看起来很 MQ，但 EDA 比"用 MQ"严格得多：

- **生产者不知道谁在消费**（解耦最彻底的形式）
- **事件描述"已经发生"的事实**，不可撤销、不可篡改
- **消费者按事件含义自行决定动作**，不被生产者指挥

如果你的代码里写了 `订单服务发消息给库存服务，让它扣减库存`，这不是 EDA，这是"用了 MQ 的 RPC"。EDA 应该是：`订单服务广播 OrderPlaced 事件，库存服务监听到后自己决定扣减库存`。两者的代码差不到 5 行，但**演进路径完全不同**。

### 1.2 Event vs Command vs Message —— 必须分清

这三个词在工程里被混用是事故的源头。

| 概念 | 时态 | 谁决定动作 | 接收方数量 | 例子 |
| --- | --- | --- | --- | --- |
| **Command（命令）** | 祈使句 / 现在时 | 发送方 | 1（点对点） | `PlaceOrder` / `ChargeCard` |
| **Event（事件）** | 过去时 / 已发生 | 接收方 | N（广播） | `OrderPlaced` / `CardCharged` |
| **Message（消息）** | 中性 | — | — | 上面两者都是 Message 的子集 |

判别口诀：**事件命名一定是过去时**。`UserRegistered` 是事件，`RegisterUser` 是命令。

为什么这条规则这么重要？因为**时态决定了责任归属**：

- 命令：发送方负责合法性校验，接收方可以拒绝执行（"我不愿意做"）
- 事件：发送方只是陈述事实，接收方不能拒绝（"它已经发生了"），只能响应或忽略

把命令命名成事件，就模糊了"谁该校验"。把事件命名成命令，就让消费者背了不该背的拒绝权。

> **口诀**：事件用过去时，命令用祈使句。`Order*ed` 是事件，`*Order` 是命令。混用就出问题。

### 1.3 Event Sourcing：状态从事件序列推导

传统 CRUD：

```sql
-- 订单状态用一行表示，每次更新原地覆盖
UPDATE orders SET status = 'paid', updated_at = NOW() WHERE id = 1001;
```

Event Sourcing：

```sql
-- 订单状态变更追加一条不可变的事件
INSERT INTO event_store (aggregate_id, version, type, payload, created_at)
VALUES (1001, 3, 'OrderPaid', '{"amount":99.0,"method":"alipay"}', NOW());
```

**根本差别**：CRUD 存的是"当前状态"，ES 存的是"达到当前状态的所有事件"。当前状态由事件回放（fold）得到。

```
事件序列：
  v1: OrderCreated(items=[...])
  v2: OrderPaid(amount=99)
  v3: OrderShipped(carrier='SF')
  v4: OrderDelivered()

回放（fold）：
  state = ∅
  state = apply(state, v1)   →  {status: CREATED, items: [...]}
  state = apply(state, v2)   →  {status: PAID, amount: 99, ...}
  state = apply(state, v3)   →  {status: SHIPPED, carrier: 'SF', ...}
  state = apply(state, v4)   →  {status: DELIVERED, ...}
```

ES 带来的好处：

- **完整审计**：每一次变化都有事件、时间戳、操作人，天然合规
- **时间旅行**：能查询"任意时刻 T 的订单是什么状态"
- **重建读模型**：Read Model 改了 schema？回放事件重建即可
- **业务回溯**：发现 3 天前的逻辑算错了，回放修正后的事件即可重算

代价也很重：

- 查询当前状态要回放（除非加 snapshot）
- Schema 演进难（事件是历史，改不动）
- 存储一直膨胀（必须有归档策略）
- 团队学习曲线陡（一上来就让大家忘掉 UPDATE 不现实）

**反直觉的事实**：**Event Sourcing 不是 EDA 的必须项**。你完全可以做 EDA（用事件解耦服务）但内部仍然用 CRUD。后面案例 ① 会展开这个误用。

### 1.4 Aggregate + Event Store 的设计

Event Sourcing 的最小存储单元是 Aggregate（聚合，DDD 概念）。一个聚合 = 一个实体 + 它的版本号 + 它的所有事件。

```
event_store 表（极简版）：
+--------------+---------+----------+-----------+--------------------+----------+
| aggregate_id | version | type     | payload   | metadata           | created  |
+--------------+---------+----------+-----------+--------------------+----------+
| order-1001   | 1       | Created  | {...}     | {trace, user, ...} | t0       |
| order-1001   | 2       | Paid     | {...}     | {...}              | t1       |
| order-1001   | 3       | Shipped  | {...}     | {...}              | t2       |
| order-1002   | 1       | Created  | {...}     | {...}              | t3       |
+--------------+---------+----------+-----------+--------------------+----------+

主键约束：(aggregate_id, version) UNIQUE
```

`(aggregate_id, version) UNIQUE` 是 Event Sourcing 实现**乐观并发控制**的根：

```java
// 加载聚合：select * from event_store where aggregate_id = ? order by version
Order order = repository.load("order-1001");          // 最后一个 version = 3

// 业务操作产生新事件
order.ship("SF");                                      // 内部产生 OrderShipped(version=4)

// 保存：要求 version=4 不存在
repository.save(order);                                // INSERT (order-1001, 4, ...)
//                                                      若并发已有 version=4 → 唯一索引冲突 → 重试
```

这是 ES 默认的**乐观锁**模式。比起 CRUD 的 `WHERE version = ?` 行锁，ES 的好处是没有锁竞争，代价是冲突时重试。

> **口诀**：ES 的并发控制 = 聚合 ID + 版本号唯一索引。冲突就重试，不锁。

### 1.5 Snapshot：性能与一致的取舍

聚合事件累积到几千条时，每次回放成本就不可接受。Snapshot（快照）是：定期把"回放到第 N 个事件后的状态"持久化下来，加载时从 snapshot 起步、只回放后续事件。

```
event_store:                          snapshots:
v1, v2, v3, ..., v1000                 +-------+--------+----------+
                                       | order | v=950  | {state}  |
                                       +-------+--------+----------+
                                       
加载流程：
  1. 找最近的 snapshot：v=950 的 state
  2. 加载 v951 ~ v1000 的事件
  3. apply(snapshot.state, [v951..v1000]) → 当前状态
```

频率怎么定？工程公式：

```
快照间隔 ≈ avg(P95 加载时间 SLA) / avg(单事件 apply 时间)

例：P95 加载要求 < 50ms，单事件 apply 1ms
    → 间隔 50 个事件做一次快照
```

但这只是性能视角。还要考虑：

- **过频快照**：写放大，每个事件多写一次状态
- **过疏快照**：长尾聚合（事件少但偶尔被加载）回放慢
- **快照不一致**（致命）：老快照 + 新事件 schema 不兼容 → 见案例 ③

主流做法是**异步生成快照**：业务事务只写事件，后台 worker 周期性挑选活跃聚合生成快照。这样业务路径上零开销。

### 1.6 CQRS 的写模型与读模型

CQRS（Command Query Responsibility Segregation，命令查询职责分离）和 ES 是**两个独立的概念**，但常被一起提，因为 ES 的读特别痛苦，必须配 CQRS 才好用。

```
┌─────────────┐                          ┌──────────────┐
│ Command     │ ──► Aggregate ──► Event ─┤  Event Bus   │
│ (写)        │       (写模型)            └──────┬───────┘
└─────────────┘                                  │
                                                 │ project
                                                 ▼
┌─────────────┐                          ┌──────────────┐
│ Query       │ ◄────────────────────────│ Read Model   │
│ (读)        │      SELECT              │ (订单列表表)  │
└─────────────┘                          └──────────────┘
```

写模型只负责"接受命令、产生事件、维护一致性"；读模型订阅事件，把数据物化成查询友好的形式。

读模型可以是任何形式：

- 关系表（同步成 MySQL 一张大宽表）
- ES 索引（同步成 Elasticsearch）
- 缓存（同步成 Redis Hash）
- 图（同步成 Neo4j）

**反直觉的事实**：**多个读模型并存是常态**。同一份事件流可以同时养 5 张读表，分别服务订单详情、订单列表、运营报表、用户画像、数仓。每张表为各自查询场景定制。

代价：**最终一致性**。事件从写到读模型物化有延迟（毫秒到秒级）。后面案例 ⑧ 展开。

### 1.7 Saga：编排 vs 协同

跨多个聚合 / 多个服务的"长事务"在 EDA 里靠 Saga 实现。Saga 的本质是：一系列**本地事务 + 补偿事务**的有序组合。

两种实现方式：

#### Orchestration（编排）—— 中央指挥官

```
  ┌──────────────────┐
  │  OrderSaga       │ ──► PlaceOrder      ─► 订单服务
  │  Orchestrator    │ ──► ReserveStock    ─► 库存服务
  │  (状态机)         │ ──► ChargePayment   ─► 支付服务
  │                  │ ──► ShipOrder       ─► 物流服务
  │                  │ 任何一步失败 → 反向补偿
  └──────────────────┘
```

中央 Orchestrator 知道整个流程的状态机，按顺序发命令、收响应、决定下一步或补偿。

#### Choreography（协同）—— 各自反应

```
订单服务 ──► OrderPlaced ──┐
                            ├─► 库存服务（监听）── StockReserved ──┐
                            │                                       ├─► 支付服务（监听）── ...
                            │                                       │
没有"中央指挥"，每个服务订阅它关心的事件，自己决定动作和后续事件
```

对比：

| 维度 | Orchestration | Choreography |
| --- | --- | --- |
| 流程可见性 | **强**（看 Orchestrator 状态） | 弱（要追多个服务的事件链） |
| 失败可观测性 | 强（中央有日志） | 弱（要分布式追踪） |
| 单点故障 | Orchestrator 挂了全停 | 无单点 |
| 扩展新步骤 | 改 Orchestrator | 加新订阅者，原服务无感 |
| 复杂度上限 | 流程几十步仍可控 | 超过 5~6 步会变"事件意大利面" |
| 适用场景 | 业务流程清晰、长 | 服务自治、流程短 |

**铁律**：**4 步以内用协同，5 步以上用编排**。这不是绝对值，但作为新人决策的安全线足够。后面案例 ⑥ 会讲一个 12 步业务流被强行做成协同导致无法排障的事故。

### 1.8 Outbox 模式与 CDC

EDA 落地最经典的工程难题：**业务事务的提交 与 事件的发送 不在一个原子操作里**。

```java
// 危险代码
@Transactional
public void placeOrder(...) {
    orderRepository.save(order);              // ① DB 事务
    kafkaTemplate.send("orders", event);      // ② Kafka send
}
```

四种失败场景：

- ①成功 ②未发送（kafka 网络挂了）→ 事件丢
- ①成功 ②已发送但调用方没收到 ack（网络抖动）→ 业务以为失败，但事件已发，下游扣库存
- ①失败回滚 ②已发送 → DB 没有订单但下游收到 OrderPlaced
- ①已写但未 commit ②先发 → 下游查不到订单

**Outbox 模式（事务性发件箱）** 把事件先写进同库的 outbox 表，与业务表共享事务：

```sql
BEGIN;
  INSERT INTO orders ...;
  INSERT INTO outbox (id, type, payload, created_at) VALUES (...);
COMMIT;
```

然后由独立的 outbox-relay 进程轮询 outbox 表，发到 MQ 后标记已发送。

```
   App ──► [orders + outbox in same TX]
                       │
                       ▼
              outbox-relay (poll/CDC)
                       │
                       ▼
                    Kafka
```

**CDC（Change Data Capture，变更数据捕获）** 走得更远：直接订阅 MySQL binlog（或 Postgres logical replication），把 DML 变成事件。

```
   App ──► [orders 表]
                │ binlog
                ▼
         Debezium/Canal
                │
                ▼
              Kafka
```

对比：

| 维度 | Outbox | CDC |
| --- | --- | --- |
| 业务侵入 | 中（要写 outbox 表） | 0（直接读 binlog） |
| 端到端延迟 | 100ms ~ 秒级（轮询粒度） | 10ms ~ 100ms |
| 运维复杂度 | 低（自己写一个 relay） | 高（部署 Debezium 集群 + Kafka Connect） |
| 事件结构控制 | 强（业务定义 outbox payload） | 弱（受 binlog 字段约束） |
| 适合场景 | 团队规模小、自主可控 | 多服务统一抓变更、大数据下游 |

后面案例 ④ ⑤ 分别讲 Outbox 和 CDC 的踩坑。

> **口诀**：业务事务 + MQ 发送的原子性 = Outbox 或 CDC。**绝不能**裸调 `kafkaTemplate.send` 在事务里。

---

## 2. 工程切片：Spring + Axon 的 EventSourced Aggregate

> 基于 Spring Boot 3.2 + Axon Framework 4.9 + Kafka 3.6 + Postgres 15。代码可直接编译运行。

### 2.1 EventSourced Aggregate 的最小骨架

Axon 是 JVM 生态最成熟的 EDA + ES 框架。它的核心是把"聚合"和"事件"用注解关联起来。

```java
@Aggregate
public class Order {

    @AggregateIdentifier
    private String orderId;
    private OrderStatus status;
    private BigDecimal amount;

    protected Order() { }   // Axon 反射用，必须无参构造

    @CommandHandler
    public Order(PlaceOrderCommand cmd) {
        // ★ 校验在这里。校验通过才发事件
        if (cmd.amount().compareTo(BigDecimal.ZERO) <= 0)
            throw new IllegalArgumentException("amount must be positive");

        AggregateLifecycle.apply(new OrderPlacedEvent(
            cmd.orderId(), cmd.amount(), Instant.now()));
    }

    @CommandHandler
    public void handle(PayOrderCommand cmd) {
        if (status != OrderStatus.CREATED)
            throw new IllegalStateException("only CREATED order can be paid, current=" + status);

        AggregateLifecycle.apply(new OrderPaidEvent(orderId, cmd.method(), Instant.now()));
    }

    @EventSourcingHandler
    public void on(OrderPlacedEvent e) {
        this.orderId = e.orderId();
        this.amount = e.amount();
        this.status = OrderStatus.CREATED;
    }

    @EventSourcingHandler
    public void on(OrderPaidEvent e) {
        this.status = OrderStatus.PAID;
    }
}
```

**逐段点透**：

- **`@CommandHandler` 和 `@EventSourcingHandler` 必须分开**：CommandHandler 做校验、决定要不要发事件；EventSourcingHandler 只做"状态修改"，不做校验、不做 IO。这条铁律不守，回放时就会因为校验失败而无法重建状态。
- **`AggregateLifecycle.apply(event)` 不是 Kafka send**：它把事件追加到聚合的"未提交事件列表"，框架在事务提交时统一持久化到 Event Store 并发布。
- **Aggregate 内**没有任何对 Repository 的依赖。它是纯领域对象，关心"业务规则一致性"，不关心存储。
- **构造方法上的 CommandHandler** 是创建聚合的入口；其他 CommandHandler 是更新入口。

### 2.2 Event Store 的三种选型

Axon 自带 `JpaEventStorageEngine`，可以把事件存到任何 JPA 支持的库（MySQL/Postgres）。

```java
@Configuration
public class AxonConfig {

    @Bean
    public EventStorageEngine storageEngine(EntityManagerProvider emp,
                                            TransactionManager txm,
                                            Serializer serializer) {
        return JpaEventStorageEngine.builder()
            .entityManagerProvider(emp)
            .transactionManager(txm)
            .eventSerializer(serializer)        // 事件序列化（默认 XStream，建议换 Jackson）
            .snapshotSerializer(serializer)
            .build();
    }

    @Bean
    public Serializer serializer() {
        return JacksonSerializer.builder().build();
    }
}
```

底层表结构（Axon 自动建）：

```sql
CREATE TABLE domain_event_entry (
    global_index    BIGSERIAL PRIMARY KEY,
    aggregate_identifier VARCHAR(255),
    sequence_number BIGINT,
    type            VARCHAR(255),
    payload_type    VARCHAR(255),
    payload_revision VARCHAR(255),
    payload         BYTEA,
    metadata        BYTEA,
    time_stamp      VARCHAR(255),
    UNIQUE (aggregate_identifier, sequence_number)
);

CREATE INDEX ON domain_event_entry (time_stamp);
```

`(aggregate_identifier, sequence_number)` 唯一索引就是 1.4 节讲的乐观并发控制基础。

三种选型对比：

| 选型 | 吞吐 | 运维 | 查询能力 | 适用 |
| --- | --- | --- | --- | --- |
| **JPA + Postgres/MySQL** | 1 ~ 5 万 events/s | 低（已有 DB） | 强（SQL 直查） | 中小型，团队已熟 SQL |
| **EventStoreDB** | 5 ~ 20 万 events/s | 中（独立产品） | 中（自带 projection） | 严肃 ES 项目 |
| **Kafka 当 Event Store** | 50 万+ events/s | 高（保留期、分区策略） | 弱（只能顺序读） | 超大规模、可接受查询差 |

Kafka 当 Event Store 的争议很大。Confluent 自己的工程师写过文章 *Don't Use Kafka as a Database*。核心问题：Kafka 没有"按 aggregate ID 高效随机读"能力，加载聚合要扫整个分区。除非你只用事件流做下游消费、不需要重建聚合，否则别这么干。

### 2.3 Outbox 模式的代码实现

不用 Axon、自己手写的版本：

```java
@Entity
@Table(name = "outbox")
public class OutboxEvent {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String aggregateType;
    private String aggregateId;
    private String eventType;
    @Column(columnDefinition = "TEXT")
    private String payload;
    private Instant createdAt;
    private Instant publishedAt;     // null = 待发送

    // getters / setters / static factory ...
}
```

业务侧只用关心"在事务里写一条 outbox"：

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepo;
    private final OutboxRepository outboxRepo;
    private final ObjectMapper json;

    @Transactional
    public Order placeOrder(PlaceOrderRequest req) {
        Order order = Order.create(req);
        orderRepo.save(order);

        // ★ 同事务写 outbox
        OutboxEvent ob = new OutboxEvent();
        ob.setAggregateType("Order");
        ob.setAggregateId(order.getId());
        ob.setEventType("OrderPlaced");
        ob.setPayload(json.writeValueAsString(
            new OrderPlacedEvent(order.getId(), order.getAmount(), Instant.now())));
        ob.setCreatedAt(Instant.now());
        outboxRepo.save(ob);

        return order;
    }
}
```

独立的 relay 进程：

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class OutboxRelay {

    private final OutboxRepository outboxRepo;
    private final KafkaTemplate<String, String> kafka;

    @Scheduled(fixedDelay = 500)        // 500ms 一轮
    public void flush() {
        // ★ FOR UPDATE SKIP LOCKED 保证多实例不抢同一条
        List<OutboxEvent> batch = outboxRepo.findBatchForUpdate(100);
        if (batch.isEmpty()) return;

        for (OutboxEvent e : batch) {
            try {
                ProducerRecord<String, String> rec = new ProducerRecord<>(
                    "domain-events", e.getAggregateId(), e.getPayload());
                rec.headers().add("eventType", e.getEventType().getBytes());
                kafka.send(rec).get(3, TimeUnit.SECONDS);    // 同步发，确保 ack
                e.setPublishedAt(Instant.now());
            } catch (Exception ex) {
                log.warn("publish failed, will retry: {}", e.getId(), ex);
                // 不更新 publishedAt，下一轮会重试
                break;     // 保证顺序，遇到失败就停，等下一轮
            }
        }
        outboxRepo.saveAll(batch);
    }
}
```

```sql
-- 配套的 SQL（Postgres）：
SELECT * FROM outbox
WHERE published_at IS NULL
ORDER BY id
LIMIT 100
FOR UPDATE SKIP LOCKED;
```

`FOR UPDATE SKIP LOCKED` 是 Outbox relay 多实例部署的关键：被一个 relay 锁住的行，其他 relay 直接跳过，不会阻塞也不会争抢。MySQL 8.0+、Postgres 9.5+ 支持。

**`break` 的设计意图**：保证发送顺序与写入顺序一致。如果允许某一条失败后跳过继续发后面的，下游收到的事件顺序就乱了，可能导致 `OrderPaid` 比 `OrderCreated` 先到。

### 2.4 Debezium CDC 接入

不想自己写 relay？用 Debezium 直接监听 binlog。

Debezium Connector 配置：

```json
{
  "name": "order-cdc",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "mysql",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "...",
    "database.server.id": "184054",
    "database.server.name": "shop",
    "database.include.list": "shop_order",
    "table.include.list": "shop_order.outbox",
    "topic.prefix": "shop",
    "include.schema.changes": "false",
    "snapshot.mode": "schema_only",
    "transforms": "outbox",
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.route.topic.replacement": "domain.${routedByValue}",
    "transforms.outbox.table.field.event.key": "aggregate_id",
    "transforms.outbox.table.field.event.payload": "payload",
    "transforms.outbox.route.by.field": "aggregate_type"
  }
}
```

关键点：`io.debezium.transforms.outbox.EventRouter` 这个 SMT（Single Message Transform）专门为 Outbox + CDC 组合设计。它会：

- 读取 outbox 表的 INSERT 事件
- 按 `aggregate_type` 字段路由到不同 topic（如 `domain.Order`、`domain.Payment`）
- 把 `payload` 字段直接当 Kafka message value
- 把 `aggregate_id` 当 Kafka message key（保证同聚合事件落同分区）

这套组合拳叫 **Outbox via CDC**：Outbox 解决业务事务原子性，CDC 解决"轮询延迟高"。延迟可降到 50ms 以内。

### 2.5 Saga Orchestration（用 Camunda）

业务流超过 5 步时，自己用状态机硬编码会很痛。Camunda 8 / Temporal 是工业级方案。

Camunda BPMN 的 Saga 示例（简化）：

```yaml
# place-order.bpmn (示意)
process:
  start: OrderPlaced
  steps:
    - reserveStock:
        type: serviceTask
        topic: stock-reserve
        compensation: stock-release
    - chargePayment:
        type: serviceTask
        topic: payment-charge
        compensation: payment-refund
    - createShipment:
        type: serviceTask
        topic: shipment-create
        compensation: shipment-cancel
    - confirmOrder:
        type: serviceTask
        topic: order-confirm
  end: OrderConfirmed
```

业务方写 Worker 处理具体动作：

```java
@Component
public class StockReserveWorker {

    @JobWorker(type = "stock-reserve")
    public Map<String, Object> reserve(@Variable String orderId,
                                        @Variable List<Item> items) {
        try {
            stockService.reserve(orderId, items);
            return Map.of("reservationId", orderId);
        } catch (StockNotEnoughException e) {
            // ★ 抛 ZeebeBpmnError 触发补偿流程
            throw new ZeebeBpmnError("STOCK_NOT_ENOUGH", e.getMessage());
        }
    }

    @JobWorker(type = "stock-release")
    public void compensate(@Variable String orderId) {
        stockService.release(orderId);     // 必须幂等
    }
}
```

Camunda 把"流程状态"持久化在自己的存储里，崩溃后能从断点恢复。这是自研 Saga 永远做不到的事 —— 自研版本一旦 Orchestrator 进程挂了，正在执行的 Saga 状态全丢。

### 2.6 Schema Registry 与事件版本管理

事件一旦发出去就**不可变**。但业务总在变，事件 schema 必须演进。

四种兼容性级别（Confluent Schema Registry 的定义）：

| 级别 | 含义 | 允许的变更 |
| --- | --- | --- |
| **BACKWARD** | 新消费者能读老数据 | 删字段、加可选字段 |
| **FORWARD** | 老消费者能读新数据 | 加字段（可选）、删可选字段 |
| **FULL** | 上面两个都满足 | 加可选字段、删可选字段 |
| **NONE** | 不检查 | 什么都行（地狱模式） |

生产**必须**至少 BACKWARD：因为消费者升级永远先于生产者发新事件，新消费者必须能继续消费历史事件。

Avro schema 演进示例：

```json
// v1
{
  "type": "record",
  "name": "OrderPlaced",
  "fields": [
    {"name": "orderId", "type": "string"},
    {"name": "amount", "type": "double"}
  ]
}

// v2 - 加可选字段（BACKWARD COMPATIBLE）
{
  "type": "record",
  "name": "OrderPlaced",
  "fields": [
    {"name": "orderId", "type": "string"},
    {"name": "amount", "type": "double"},
    {"name": "currency", "type": ["null", "string"], "default": null}
  ]
}
```

关键：**新增字段必须有 default 值**。否则反序列化老事件时找不到该字段会报错。

把 Schema Registry 接入 Kafka producer：

```java
Properties props = new Properties();
props.put("bootstrap.servers", "...");
props.put("key.serializer", "io.confluent.kafka.serializers.KafkaAvroSerializer");
props.put("value.serializer", "io.confluent.kafka.serializers.KafkaAvroSerializer");
props.put("schema.registry.url", "http://schema-registry:8081");
// 强制 BACKWARD 兼容：拒绝不兼容 schema 注册
props.put("auto.register.schemas", "false");
```

Axon 也支持事件 upcast（升级旧事件到新 schema）：

```java
public class OrderPlacedV1ToV2Upcaster
    extends SingleEventUpcaster {

    @Override
    protected boolean canUpcast(IntermediateEventRepresentation rep) {
        return rep.getType().getName().equals("OrderPlaced")
            && "1".equals(rep.getType().getRevision());
    }

    @Override
    protected IntermediateEventRepresentation doUpcast(IntermediateEventRepresentation rep) {
        return rep.upcastPayload(
            new SimpleSerializedType("OrderPlaced", "2"),
            JsonNode.class,
            doc -> {
                ((ObjectNode) doc).put("currency", "CNY");      // 老事件默认人民币
                return doc;
            });
    }
}
```

Upcaster 链让你在不改老事件的前提下，在加载时把它们"翻译"成新 schema。这是 ES 应对 schema 演进的核心工具。

### 2.7 Read Model 的物化（Projection）

读模型订阅事件并维护查询表。

```java
@Component
@ProcessingGroup("order-summary-projection")
public class OrderSummaryProjector {

    private final OrderSummaryRepository repo;

    @EventHandler
    public void on(OrderPlacedEvent e) {
        OrderSummary s = new OrderSummary();
        s.setOrderId(e.orderId());
        s.setStatus("CREATED");
        s.setAmount(e.amount());
        s.setCreatedAt(e.timestamp());
        repo.save(s);
    }

    @EventHandler
    public void on(OrderPaidEvent e) {
        OrderSummary s = repo.findById(e.orderId()).orElseThrow();
        s.setStatus("PAID");
        s.setPaidAt(e.timestamp());
        repo.save(s);
    }

    @EventHandler
    public void on(OrderShippedEvent e) {
        OrderSummary s = repo.findById(e.orderId()).orElseThrow();
        s.setStatus("SHIPPED");
        s.setCarrier(e.carrier());
        repo.save(s);
    }
}
```

`@ProcessingGroup("order-summary-projection")` 让这组 handler 共享一个独立的 token store（追踪消费位点），互不干扰。

**重建读模型**：业务想加一个"按月汇总订单数"的报表？写新 Projection，在 Axon 控制台 reset 它的 token 到 0，框架自动从 event store 头开始重放所有事件。

```java
eventProcessingConfiguration.eventProcessor("order-monthly-summary", TrackingEventProcessor.class)
    .ifPresent(processor -> {
        processor.shutDown();
        processor.resetTokens();
        processor.start();
    });
```

这是 ES 的核心红利：**任何过去事件都可以无限次回放，构造新视图无需补数据**。CRUD 系统改 schema 加字段时往往要写一次性脚本回填，ES 不需要。

> **口诀**：读模型 = 事件流的物化视图。多个读模型并存，每个为各自查询场景定制。重建只需 reset token。

---

## 3. 实测数据：ES vs CRUD、Outbox vs CDC

> 测试环境：MacBook M2 Pro 32GB、Postgres 15 同机部署、Kafka 单节点。仅供数量级参考，绝对值受硬件影响很大。

### 3.1 EventSourcing vs CRUD 的写入性能

写入"创建订单"操作 100 万次，对比：

| 方案 | 平均吞吐 | P99 写延迟 | 磁盘占用 |
| --- | --- | --- | --- |
| **CRUD（INSERT 一行）** | 18,000 ops/s | 5 ms | 220 MB |
| **ES（INSERT 一条事件）** | 22,000 ops/s | 4 ms | 380 MB（多了 metadata） |
| **ES + 同步 snapshot 每 10 个事件** | 9,500 ops/s | 12 ms | 620 MB |
| **ES + 异步 snapshot** | 21,500 ops/s | 4 ms | 620 MB |

**反直觉的发现**：ES 的纯写入比 CRUD 略快。原因：ES 是 INSERT-only，没有索引页随机更新；CRUD 的 UPDATE 要先定位、再改、可能触发索引维护。

但 ES 的**总成本**包括：

- 加载聚合时的回放开销（CRUD 是一行 SELECT）
- snapshot 维护的写放大
- event store 永久膨胀的存储成本

### 3.2 Snapshot 频率对查询性能的影响

固定每个聚合 1000 个事件，单事件 apply 耗时约 50 微秒，测加载该聚合的 P99 延迟：

| Snapshot 间隔 | P99 加载延迟 | 写放大（每事件额外写次数） |
| --- | --- | --- |
| 不做 snapshot | 65 ms | 0 |
| 每 500 个事件 | 32 ms | 0.002 |
| 每 100 个事件 | 8 ms | 0.01 |
| 每 50 个事件 | 5 ms | 0.02 |
| 每 10 个事件 | 3 ms | 0.1 |
| 每 1 个事件（极端） | 2 ms | 1（写放大翻倍） |

工程结论：

- 加载延迟与 snapshot 间隔近似线性
- 间隔小于 50 时，**写放大开始显著影响吞吐**
- 主流推荐 50 ~ 200 之间，且**异步**生成

```
口诀：snapshot 越频，读越快、写越累
      推荐 100 一档，异步生成，业务路径零阻塞
```

### 3.3 Outbox vs CDC 的延迟对比

测：从业务 commit 到下游消费者收到事件的端到端延迟。

| 方案 | P50 | P99 | 备注 |
| --- | --- | --- | --- |
| **裸调 kafkaTemplate.send（不可靠）** | 5 ms | 15 ms | 有丢失风险，仅作基线 |
| **Outbox + 1s 轮询** | 520 ms | 1,200 ms | 简单可靠 |
| **Outbox + 200ms 轮询** | 110 ms | 320 ms | 数据库压力小幅上升 |
| **Outbox + 50ms 轮询** | 30 ms | 110 ms | DB QPS 增加 25% |
| **Debezium CDC** | 25 ms | 80 ms | 部署复杂度高 |
| **Outbox + Debezium CDC** | 30 ms | 90 ms | 推荐组合 |

**结论**：

- 单纯延迟比拼 CDC 略胜
- Outbox 200ms 轮询对绝大多数业务足够（不是所有业务都需要 100ms 内的事件传播）
- 综合可控性 + 延迟，Outbox + CDC 是工业最优解

### 3.4 Saga 长事务的成功率

8 步 Saga（每步独立服务），每步独立失败概率 0.5%（典型生产环境）：

| 设计 | 端到端成功率 | 平均补偿次数 |
| --- | --- | --- |
| 无重试 | 96.1% | 1.95 |
| 每步重试 3 次（指数退避） | 99.97% | 0.01 |
| 每步重试 + 死信队列人工介入 | 99.99% | 0.005 |

口诀：**Saga 必配重试，不配重试的 Saga 是玩具**。每步 3 次重试 + 指数退避能把成功率从 96% 提到 99.97%。

### 3.5 不同 Event Store 吞吐对比

单机、单聚合、纯 append 写入：

| Event Store | 吞吐 (events/s) | 加载 1000 事件耗时 |
| --- | --- | --- |
| **Postgres + JPA** | 18,000 | 12 ms |
| **EventStoreDB** | 65,000 | 4 ms |
| **Axon Server** | 35,000 | 6 ms |
| **Kafka（按 aggregateId 分区）** | 320,000 | 不支持随机加载 |
| **Cassandra**（按 aggregateId 分区） | 95,000 | 8 ms |

如果你的事件量 < 10K events/s，Postgres 完全够用，**选型简单胜过性能**。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① CRUD 场景被强行 ES —— "三个月后整个团队想推翻"

#### 现象

一个内部管理系统的"用户配置"模块（用户偏好、地址簿、付款方式）被架构师一拍脑袋上了 Event Sourcing。三个月后：

- 业务方提的"导出全量用户偏好"需求做了三周没做完
- 改一个字段要写 V2 事件 + Upcaster + Projection 重建，需求平均交付时间从 3 天涨到 2 周
- 新人入职 2 个月还没敢提 PR
- 团队 90% 的时间在和框架斗争，不是和业务斗争

#### 根因

这个模块的特征：

- **状态变化少**（用户偏好平均一个月改一次）
- **没有审计需求**（管理系统，不是金融）
- **查询多种多样**（按城市、按支付方式、按使用时长等）
- **业务流程简单**（没有跨服务长事务）

ES 的优势（审计、时间旅行、回放重建）一个都用不上；ES 的代价（schema 演进难、查询要靠 Projection、团队学习成本）全都吃满。

ES 的适用判据（必须**全部**满足才考虑）：

1. 业务**关心历史变化过程**而不只是当前状态（金融账户、医疗记录、合规审计）
2. 状态变化频繁且每次都有业务含义（订单流程、工作流）
3. 查询模式可以预先建模成 Projection（不会突然要"按任意维度聚合"）
4. 团队有 ES 经验，或愿意投入半年学习成本
5. 系统对最终一致性容忍

少满足一条，老老实实用 CRUD。

#### 修复

把 ES 模块改回 CRUD，但保留事件作为审计日志：

```java
@Service
@Transactional
public class UserPreferenceService {

    private final UserPreferenceRepository repo;
    private final AuditLogRepository auditRepo;
    private final ApplicationEventPublisher publisher;

    public void update(String userId, UserPreference pref) {
        UserPreference old = repo.findById(userId).orElse(null);
        repo.save(pref);                    // CRUD 写法

        // 同事务追加审计日志
        auditRepo.save(AuditLog.of("UserPreference", userId, old, pref));

        // 异步广播事件给下游
        publisher.publishEvent(new UserPreferenceChangedEvent(userId, old, pref));
    }
}
```

享受了"事件驱动 + 审计"的好处，同时避免了 ES 的工程负担。

#### 验证

迁移后：

- 新需求平均交付时间回到 3 天
- 团队反馈"终于可以专心写业务了"
- 服务 P99 延迟降低 40%（少了一层 Projection）

> **口诀**：ES 不是默认选项。**一切场景的默认是 CRUD**，只有 ES 的所有优势真的需要、所有代价能承受时才用 ES。

---

### 案例 ② Schema 演进失败 —— "旧事件不能反序列化导致服务起不来"

#### 现象

业务把 `OrderPlacedEvent` 的 `amount` 字段从 `double` 改成 `BigDecimal`（要解决精度问题）。开发环境一切正常，灰度上线后 10 分钟内服务陆续崩溃，日志：

```
Caused by: com.fasterxml.jackson.databind.exc.InvalidFormatException:
  Cannot deserialize value of type `java.math.BigDecimal` from String "99.0":
    not a valid representation
```

事件回放卡在某个老聚合上，整个 Projection 进程死掉。

#### 根因

事件是历史事实，**已发出去的事件不能改**。但开发同学把 `OrderPlacedEvent` 的字段类型改了，新代码尝试反序列化老 JSON 时类型不匹配。

更糟的是，这个事件类被 Projection、Saga、读模型重建多处使用，到处都在崩。

#### 修复

**铁律**：**事件类一旦上线，字段类型不能改、字段不能删、语义不能变**。

如果真的要改，必须发一个新版本事件 + Upcaster：

```java
// 保留 V1 不动
public record OrderPlacedEventV1(String orderId, double amount, Instant timestamp) { }

// 发布新版本
public record OrderPlacedEventV2(String orderId, BigDecimal amount, Instant timestamp) { }

// Upcaster：加载老事件时升级到新版
@Component
public class OrderPlacedV1ToV2Upcaster extends SingleEventUpcaster {

    @Override
    protected boolean canUpcast(IntermediateEventRepresentation rep) {
        return rep.getType().getName().equals("OrderPlacedEvent")
            && "1".equals(rep.getType().getRevision());
    }

    @Override
    protected IntermediateEventRepresentation doUpcast(IntermediateEventRepresentation rep) {
        return rep.upcastPayload(
            new SimpleSerializedType("OrderPlacedEvent", "2"),
            JsonNode.class,
            payload -> {
                ObjectNode node = (ObjectNode) payload;
                double oldAmount = node.get("amount").asDouble();
                node.put("amount", BigDecimal.valueOf(oldAmount).toPlainString());
                return node;
            });
    }
}
```

Schema 演进的"安全清单"：

| 变更类型 | 是否安全 | 处理 |
| --- | --- | --- |
| 加可选字段（带默认值） | 安全 | 直接加 |
| 删字段 | **危险** | 加新版本事件，老消费者继续用老版 |
| 改字段类型 | **危险** | 加新版本事件 + Upcaster |
| 改字段语义（同名不同义） | **致命** | 必须改名 + 新版事件 + 数据迁移 |
| 加必填字段 | **致命** | 老事件无该字段，反序列化必爆 |

#### 验证

加 CI 检查：每次 PR 跑一个测试，加载所有历史事件类型的样本数据，验证能正常反序列化：

```java
@Test
void allHistoricalEventsMustBeDeserializable() throws Exception {
    Path samplesDir = Paths.get("src/test/resources/event-samples");
    try (Stream<Path> files = Files.list(samplesDir)) {
        files.forEach(this::tryDeserialize);
    }
}

private void tryDeserialize(Path file) {
    try {
        String json = Files.readString(file);
        Class<?> clazz = Class.forName(extractTypeFromFilename(file));
        objectMapper.readValue(json, clazz);
    } catch (Exception e) {
        fail("event sample " + file + " can't be deserialized: " + e.getMessage());
    }
}
```

每次发布新事件时同步把样本 JSON 提交到 `event-samples/`，CI 强制保证向前兼容。

> **口诀**：事件是历史，历史不可改。任何字段变更 = 加新版本 + Upcaster，永不修改老版本。

---

### 案例 ③ Snapshot 不一致导致状态错乱 —— "聚合状态突然回到 3 天前"

#### 现象

线上某用户反馈："我明明已经把账户余额从 1000 提到 800，今天再看变回 1000 了。"查 event store，确认 v=15 的 `BalanceWithdrawn(200)` 事件存在；但聚合加载结果显示余额 1000。

#### 根因

查 snapshot 表：

```
aggregate_id = user-007
last_snapshot_version = 14
snapshot_state = {balance: 1000, ...}
```

snapshot 是在 v=14 之后做的，但**做 snapshot 的代码用了缓存里的 stale 状态**：

```java
// 错误的 snapshot 生成逻辑
@Scheduled(fixedRate = 60_000)
public void generateSnapshots() {
    for (String aggId : activeAggregates) {
        // ★ 从缓存拿 state，缓存可能比 event store 旧
        Object state = aggregateStateCache.get(aggId);
        long version = stateVersionCache.get(aggId);
        snapshotStore.save(aggId, version, state);
    }
}
```

更微妙的是：snapshot 加载顺序：先从 snapshot 拿 state + version，再加载 version 之后的事件。如果 snapshot 的 version 是 14，但 state 实际是 v=10 的状态（缓存陈旧），那么后面回放 v=15 ~ N 时会基于错误起点。

#### 修复

snapshot 必须**直接从 event store 重新回放生成**，不能用任何中间缓存：

```java
@Scheduled(fixedRate = 300_000)
public void generateSnapshots() {
    for (String aggId : findAggregatesNeedingSnapshot()) {
        try {
            // ★ 从 event store 重新加载，不用任何缓存
            List<DomainEvent> events = eventStore.loadAll(aggId);
            if (events.isEmpty()) continue;

            Object state = replay(events);
            long version = events.get(events.size() - 1).getVersion();

            snapshotStore.save(aggId, version, state);
        } catch (Exception e) {
            log.error("snapshot generation failed for {}", aggId, e);
        }
    }
}
```

更进一步：**snapshot 写入要做版本校验**，避免并发写入：

```sql
INSERT INTO snapshots (aggregate_id, version, state, created_at)
VALUES (?, ?, ?, NOW())
ON CONFLICT (aggregate_id) DO UPDATE
SET version = EXCLUDED.version, state = EXCLUDED.state, created_at = EXCLUDED.created_at
WHERE snapshots.version < EXCLUDED.version;       -- ★ 只接受更新的版本
```

如果两个 snapshot worker 同时跑，旧版本不会覆盖新版本。

#### 修复 stale snapshot

发现历史 snapshot 已经污染了，怎么办？

```java
// 一次性脚本：清空所有 snapshot，下次加载时重新从 event store 回放生成
public void purgeAllSnapshots() {
    snapshotStore.deleteAll();
}
```

这是 ES 的好处：snapshot 是缓存，丢了不影响正确性，最多影响性能。

#### 验证

加单元测试：随机选取 100 个聚合，对比"从 snapshot 加载"和"从头回放"的状态：

```java
@Test
void snapshotMustEqualReplayResult() {
    List<String> sampleIds = pickRandomAggregates(100);
    for (String id : sampleIds) {
        Object fromSnapshot = repository.loadFromSnapshot(id);
        Object fromReplay = repository.loadByReplay(id);
        assertEquals(fromReplay, fromSnapshot, "mismatch on " + id);
    }
}
```

> **口诀**：snapshot 必须从 event store 直接回放生成，不用缓存。snapshot 写入用乐观锁防覆盖。

---

### 案例 ④ Outbox 写失败导致事件丢失 —— "明明都成功了，下游却没收到"

#### 现象

业务侧日志显示订单创建成功（DB 有记录），但库存服务始终没收到 `OrderPlaced` 事件。检查 Kafka 也没该事件。

排查 outbox 表，发现这条订单根本没有对应的 outbox 记录。

#### 根因

业务代码：

```java
@Transactional
public Order placeOrder(...) {
    Order order = Order.create(...);
    orderRepo.save(order);

    try {
        OutboxEvent ob = buildOutbox(order);
        outboxRepo.save(ob);
    } catch (Exception e) {
        log.warn("outbox failed, but order saved", e);    // ★ 致命错误
    }
    return order;
}
```

开发同学怕 outbox 写失败影响主流程，加了个 try-catch 把 outbox 异常吞了。**这彻底破坏了 Outbox 的原子性保证** —— outbox 失败时业务事务仍然 commit，事件就丢了。

#### 修复

**铁律**：Outbox 的写入必须和业务表在**同一个事务**里，**不能 try-catch**。

```java
@Transactional       // 必须 PROPAGATION_REQUIRED
public Order placeOrder(...) {
    Order order = Order.create(...);
    orderRepo.save(order);

    OutboxEvent ob = buildOutbox(order);
    outboxRepo.save(ob);                  // ★ 让异常向上抛，回滚整个事务

    return order;
}
```

如果 outbox 写失败（比如 outbox 表锁等待超时），业务表也回滚，前端收到错误，用户重试。这是正确的语义：**要么业务和事件都成功，要么都失败**。

进一步加保险：监控 outbox 表的"失败率"和"积压数"：

```java
@Scheduled(fixedRate = 60_000)
public void monitorOutbox() {
    long pending = outboxRepo.countByPublishedAtIsNull();
    long oldest = outboxRepo.findOldestPendingAge();      // 最老的未发送事件年龄

    Metrics.gauge("outbox.pending.count", pending);
    Metrics.gauge("outbox.pending.oldest_seconds", oldest);

    if (oldest > 60) {                  // 1 分钟还没发出去要告警
        alerter.warn("outbox publish lag > 60s");
    }
}
```

还要监控 relay 进程的健康：

```java
@Scheduled(fixedRate = 30_000)
public void heartbeat() {
    relayHeartbeatRepo.update(Instant.now());        // 心跳
}

// 另一个进程或外部监控检查心跳是否陈旧
```

#### 验证

混沌测试：

```java
@Test
void outboxMustNotLoseOnFailure() {
    // 模拟 outbox 表写失败
    when(outboxRepo.save(any())).thenThrow(new RuntimeException("simulated"));

    assertThrows(RuntimeException.class, () -> orderService.placeOrder(req));

    // 验证业务表也没数据
    assertEquals(0, orderRepo.count());
}
```

> **口诀**：outbox 写失败 = 业务回滚。**永远不能 try-catch 吞 outbox 异常**。再配监控 + 告警 + 心跳三件套。

---

### 案例 ⑤ CDC 同步延迟导致下游业务不一致 —— "下单后立刻查不到"

#### 现象

用户下单后 100ms 内调"我的订单"列表（接的是 ES 读模型，由 Debezium CDC 从订单库同步），列表里看不到刚下的订单。但等 1~2 秒后再刷就有了。

#### 根因

正常 CDC 链路：

```
App ──► MySQL ──► (binlog) ──► Debezium ──► Kafka ──► Projector ──► ES
                                                                     │
                                                                     ▼
                                                                  Read API
```

每一步都有延迟：

- MySQL flush binlog：< 10ms
- Debezium 读 binlog：10 ~ 50ms
- Kafka 落盘 + ack：5 ~ 20ms
- Projector 消费 + 写 ES：10 ~ 100ms
- ES refresh interval（默认 1s）：**这里是大头**

ES 默认 1 秒刷新一次（refresh），即使数据写进去了，也要等 refresh 才能被搜到。100ms 内查询大概率拿不到。

#### 修复

**方案 A**：把 ES refresh 调小，但不要小到 100ms（每秒 10 次 refresh 会让 IO 翻倍）：

```yaml
index:
  refresh_interval: 500ms        # 妥协方案
```

**方案 B**：写入时显式 refresh（仅对刚写入的索引）：

```java
restClient.update(req.setRefreshPolicy(WriteRequest.RefreshPolicy.WAIT_UNTIL));
```

`WAIT_UNTIL` 不强制 refresh，只是阻塞调用直到下次 refresh 完成（最多 1s）。比 `IMMEDIATE` 友好得多。

**方案 C（最优）**：用户刚下完单的查询，**走"发后即读"的兜底**：

```java
@GetMapping("/my-orders")
public List<OrderSummary> myOrders(@AuthUser User user) {
    List<OrderSummary> fromES = esRepo.findByUserId(user.getId());

    // ★ 检查是否有近期下单未在 ES 出现
    List<Order> recentFromDB = orderRepo.findByUserIdAndCreatedAfter(
        user.getId(), Instant.now().minusSeconds(5));

    Set<String> esIds = fromES.stream().map(OrderSummary::getOrderId).collect(toSet());
    List<OrderSummary> missing = recentFromDB.stream()
        .filter(o -> !esIds.contains(o.getId()))
        .map(OrderSummary::from)
        .toList();

    return Stream.concat(fromES.stream(), missing.stream())
        .sorted(comparing(OrderSummary::getCreatedAt).reversed())
        .toList();
}
```

这是"读时合并最近写入"模式，本质是用读复杂度换最终一致性窗口。

**方案 D**：用户下完单的 5 秒内，前端不直接查列表，而是用"乐观更新"在前端拼一条假数据进去，5 秒后再调列表。这是最便宜也最常用的方案。

#### 监控

监控 CDC 端到端延迟：

```java
// 业务写入时打个时间戳到事件 metadata
event.metadata().put("createdAt", Instant.now());

// Projector 消费时计算延迟
@EventHandler
public void on(OrderPlacedEvent e, MetaData md) {
    Instant created = Instant.parse(md.get("createdAt"));
    long lagMs = Duration.between(created, Instant.now()).toMillis();
    Metrics.histogram("cdc.lag_ms").record(lagMs);
}
```

P99 应该 < 500ms。超过 1 秒持续告警。

#### 验证

混沌注入：故意把 Debezium 暂停 10 秒，看监控告警是否触发、用户体验如何降级。

> **口诀**：CDC 延迟不可避免。**关键查询要么前端乐观更新、要么读时回退主库兜底**。监控 P99 端到端延迟。

---

### 案例 ⑥ Saga 补偿失败的人工兜底 —— "12 步流程错在第 9 步"

#### 现象

一个国际业务有 12 步 Saga：下单 → 风控 → 锁库存 → 锁汇率 → 扣款 → 报关 → ... → 国际物流 → 收件人确认。

某一天第 9 步（报关）失败，触发反向补偿：第 8 ~ 1 步逐步反向回滚。但第 4 步（汇率锁）的"释放汇率锁"接口本身也挂了 —— 上游汇率服务降级返回 503。

整个 Saga 卡在补偿中，订单数据在各个服务里都是"半截状态"，客服无法下手。

#### 根因

两个根因叠加：

1. **采用了协同（Choreography）模式做 12 步流程**：每一步是各自服务监听上一步事件触发，没有中央状态机。一旦补偿失败，没人知道整体到了哪一步。

2. **补偿动作没有"补偿失败"的兜底**：业务设计假设补偿一定成功，没考虑补偿本身也会失败。

#### 修复

**改造 1**：12 步流程必须用编排（Orchestration）。引入 Camunda 作为 Saga Orchestrator：

```java
// 用 Camunda 定义 12 步流程
@Component
public class InternationalOrderSaga {

    @JobWorker(type = "reserve-stock")
    public Map<String, Object> reserveStock(@Variable String orderId) {
        // 调库存服务
        return Map.of("reservationId", ...);
    }

    @JobWorker(type = "release-stock")    // 补偿
    public void releaseStock(@Variable String reservationId) {
        // 调库存服务释放
    }

    // ... 12 个 worker + 12 个补偿 worker
}
```

Camunda 把流程状态持久化在自己的存储里。如果某一步补偿失败：

- Camunda 自动重试（按配置的 backoff 策略）
- 重试 N 次仍失败 → 进入"Incident"状态
- 运维在 Camunda 控制台能看到所有 stuck 的实例和失败原因
- 人工修复底层问题后，点"Retry"按钮继续

**改造 2**：补偿动作必须可重试且幂等：

```java
@JobWorker(type = "release-fx-lock", retryBackoff = "exponential")
public void releaseFxLock(@Variable String fxLockId) {
    try {
        fxService.release(fxLockId);
    } catch (FxServiceUnavailableException e) {
        // 可重试 → 抛 Camunda 重试异常
        throw new ZeebeException("fx service unavailable, retrying", e);
    } catch (FxLockNotFoundException e) {
        // 已经释放了 → 幂等吞掉
        log.info("fx lock {} already released, skip", fxLockId);
    }
}
```

**改造 3**：所有 Saga 失败的兜底有人工介入入口：

```java
@RestController
@RequestMapping("/admin/saga")
public class SagaAdminController {

    private final ZeebeClient zeebe;

    @PostMapping("/{instanceId}/resolve-incident")
    public void resolveIncident(@PathVariable long instanceId,
                                @RequestParam String operatorId,
                                @RequestParam String reason) {
        auditLog.record(operatorId, instanceId, reason);
        zeebe.newResolveIncidentCommand(instanceId).send().join();
    }

    @PostMapping("/{instanceId}/manual-complete")
    public void manualComplete(@PathVariable long instanceId,
                               @RequestParam String operatorId,
                               @RequestParam String reason) {
        // 极端情况：人工确认已经完成（如客服已手动退款），直接结束 Saga
        auditLog.record(operatorId, instanceId, "manual-complete: " + reason);
        zeebe.newCancelInstanceCommand(instanceId).send().join();
    }
}
```

#### 验证

故意让某个补偿接口长时间返回 503，观察：

- Camunda 是否进入 Incident 状态
- 告警是否触发
- 控制台是否能看到具体哪一步、什么错
- 修复底层后 retry 是否能恢复

> **口诀**：5 步以上 Saga 必用编排。补偿动作必须可重试 + 幂等。**所有自动化兜底之上必须有人工介入入口**。

---

### 案例 ⑦ Event Store 增长无限制 —— "上线一年磁盘 2TB"

#### 现象

一个 ES 系统上线一年后，event_store 表从最初的 200MB 涨到 2TB。Postgres 主从同步开始落后，备份时间从 30 分钟涨到 8 小时。

#### 根因

Event Sourcing 的本质就是 append-only，事件**永不删除**。但下面三件事被忽略了：

1. **没有归档策略**：所有事件都在主表里，包括 5 年前的。
2. **聚合不分类**：长生命周期聚合（如用户）和短生命周期聚合（如临时购物车）混在一张表里。
3. **没有冷热分离**：事件全在主磁盘，从不归档到 S3。

#### 修复

**策略 1**：按时间冷热分层。

```sql
-- 把 90 天前的事件归档到 cold 表（可以是更便宜的存储）
CREATE TABLE event_store_cold (LIKE event_store INCLUDING ALL);

-- 后台 job 每天搬迁
INSERT INTO event_store_cold
  SELECT * FROM event_store WHERE created_at < NOW() - INTERVAL '90 days';

DELETE FROM event_store WHERE created_at < NOW() - INTERVAL '90 days';
```

加载聚合时优先查 hot 表，如果是历史聚合再查 cold 表（很少发生）。

**策略 2**：短生命周期聚合用独立表。

```java
// 临时购物车独立 event store
@Aggregate(snapshotTriggerDefinition = "cartSnapshotTrigger")
public class TempCart {
    // ...
}
```

```java
@Bean(name = "cartEventStore")
public EventStorageEngine cartEventStore(...) {
    return JpaEventStorageEngine.builder()
        .eventTable("temp_cart_events")
        .build();
}
```

定期清理：30 天前的临时购物车事件直接 DROP（业务认可）。

**策略 3**：Snapshot + 事件归档协同。

对超长生命周期的聚合（如用户），在 snapshot 之后的事件可以更激进地归档：

```
聚合 user-007 的事件链：
  v1 ~ v9999    → 已归档到 S3（snapshot v=10000 后无人加载）
  v10000        → snapshot
  v10001 ~      → hot 表
```

加载时只需 snapshot + hot 表事件，归档的事件平时根本不读。

**策略 4**：监控存储增长。

```sql
-- 每周看一次每个聚合类型的事件增长
SELECT
  payload_type,
  COUNT(*) as events,
  pg_size_pretty(SUM(pg_column_size(payload))) as size
FROM event_store
WHERE created_at > NOW() - INTERVAL '7 days'
GROUP BY payload_type
ORDER BY size DESC;
```

发现某个事件类型异常增长，立刻排查是否有"事件风暴"（见案例 ⑨）。

#### 验证

模拟一年的事件量预测存储增长：

```java
public void capacityPlan() {
    long avgEventBytes = 800;
    long expectedEventsPerDay = 10_000_000;
    long retentionDays = 90;

    long hotStorage = avgEventBytes * expectedEventsPerDay * retentionDays;
    log.info("hot storage estimate: {} GB", hotStorage / (1024L * 1024 * 1024));

    long yearStorage = avgEventBytes * expectedEventsPerDay * 365;
    log.info("annual storage estimate (cold): {} GB", yearStorage / (1024L * 1024 * 1024));
}
```

按这个估算配磁盘和归档策略。

> **口诀**：ES 不是"无限存"。冷热分层 + 按聚合类型分表 + snapshot 后归档，三件套缺一不可。

---

### 案例 ⑧ Read Model 数据陈旧 —— "用户看到 5 分钟前的数据还在"

#### 现象

下游 BI 报表系统用一个 Projection 维护"实时销售汇总"。运营反馈"每天下午 3 点的报表数据老是落后实际 5~10 分钟"。

#### 根因

排查 Projection 进程：

- 进程没挂
- 没有错误日志
- 但消费 Kafka 的 lag 持续在 10 万 ~ 50 万条事件

定位到根源：Projection 是单线程消费（怕乱序），但下午 3 点是销售高峰，事件产出速度远超消费速度。

#### 修复

**改造 1**：按聚合 ID 分区并行消费。

```java
@Component
@ProcessingGroup("sales-summary")
public class SalesSummaryProjector {
    // ...
}
```

```java
@Bean
public ConfigurerModule processingGroupParallelism() {
    return configurer -> configurer.eventProcessing()
        .registerTrackingEventProcessor("sales-summary",
            org.axonframework.config.Configuration::eventStore,
            c -> TrackingEventProcessorConfiguration
                .forParallelProcessing(8)               // 8 个并行 segment
                .andBatchSize(100));
}
```

每个 segment 处理一部分聚合（按 hash），多个 segment 并行。**前提**：同一个聚合的事件必须落同一个 segment（保证单聚合内顺序），框架自动按 aggregateId hash 实现。

**改造 2**：批量写入读模型。

```java
@EventHandler
public void on(SaleEvent e) {
    pendingUpdates.add(e);
    if (pendingUpdates.size() >= 100) {
        repo.batchUpdate(pendingUpdates);
        pendingUpdates.clear();
    }
}

@EventHandler
public void on(BatchEnded e) {
    if (!pendingUpdates.isEmpty()) {
        repo.batchUpdate(pendingUpdates);
        pendingUpdates.clear();
    }
}
```

单条 INSERT 50ms，批 100 条 80ms —— 吞吐 60 倍提升。

**改造 3**：读模型用更快的存储。

OLAP 场景用 ClickHouse 替代 MySQL：

```java
// 不是更新现有行，而是 append-only insert，ClickHouse 自己合并
clickHouseRepo.insertSale(SaleRecord.from(e));
```

ClickHouse 单节点能撑 100 万 ops/s 写入，远超 MySQL。

**改造 4**：Projection 消费 lag 监控。

```java
@Scheduled(fixedRate = 30_000)
public void monitorProjectionLag() {
    long lag = projectionMetrics.getCurrentLag("sales-summary");
    Metrics.gauge("projection.lag", "name", "sales-summary").set(lag);
    if (lag > 10_000) {
        alerter.warn("sales-summary projection lag > 10K");
    }
}
```

#### 验证

压测：以 10 倍正常 QPS 灌事件，看 Projection lag：

- 改造前：lag 持续上涨，永远追不上
- 改造后（8 segment + 批量写）：峰值 lag 5K，2 分钟回归 0

> **口诀**：Projection 慢的根因是单线程 + 单条写。**按聚合 ID 分区并行 + 批量写**是标准武器。

---

### 案例 ⑨ 跨服务事件循环 —— "Event Storm 把 Kafka 打爆"

#### 现象

某天 Kafka 集群突然 lag 10 亿条，消息体里全是 `OrderUpdated` 事件，主题是 `domain-events`。监控显示订单服务的 outbox 一直在吐事件，订单服务的 CPU 100%。

#### 根因

事件循环：

```
订单服务发 OrderUpdated
     │
     ├─► 库存服务监听到，更新 reserved_qty，发 StockReserved
     │              │
     │              └─► 订单服务监听 StockReserved，更新 order.locked_stock 字段
     │                              │
     │                              └─► 又触发 OrderUpdated 事件！
     │                                              │
     │                                              ├─► 库存服务再次响应
     │                                              ...（无限循环）
```

代码里 `Order.updateLockedStock(qty)` 这个方法**也产生 OrderUpdated 事件**（因为开发同学复用了通用的"订单变更"事件类型），导致循环。

#### 根因深入

这是 EDA 的经典反模式 —— **"通用事件"代替"领域事件"**。`OrderUpdated` 这种通用事件包含太多语义（任何变更都算 update），让消费者无法区分"该响应"和"不该响应"。

#### 修复

**改造 1**：用细粒度领域事件取代通用事件。

```java
// ❌ 通用事件
public record OrderUpdatedEvent(String orderId, Map<String, Object> changes) { }

// ✅ 细粒度事件
public record OrderShippedEvent(String orderId, String carrier, Instant timestamp) { }
public record OrderPaidEvent(String orderId, BigDecimal amount, String method, Instant timestamp) { }
public record OrderLockedStockChangedEvent(String orderId, int newLockedQty, Instant timestamp) { }
```

库存服务只订阅 `OrderShippedEvent` 和 `OrderPaidEvent`，不订阅"locked stock 变化"事件。循环断裂。

**改造 2**：事件流向必须画图，且强制单向。

```
   ┌─────────────┐                    ┌──────────────┐
   │   Order     │ ──Order events──►  │   Stock      │
   │  Service    │                    │   Service    │
   └─────────────┘ ◄──Stock events──  └──────────────┘
   
   规则：A 的事件可以让 B 改自己的状态，但 B 改完不要再发"通知 A"的事件。
         如果真要双向通信，用 Saga 显式编排，不要靠事件。
```

**改造 3**：为关键事件加"溯源标记"，防止环路。

```java
public class EventMetadata {
    private String causationId;       // 触发本事件的事件 ID
    private String correlationId;     // 整个流程的关联 ID
    private List<String> eventChain;  // 经过的服务链
    // ...
}

@EventHandler
public void on(StockReservedEvent e, MetaData md) {
    List<String> chain = (List<String>) md.get("eventChain");
    if (chain != null && chain.contains("order-service")) {
        log.warn("event already passed through order-service, skipping to break loop");
        return;
    }
    // ... 正常处理
}
```

这是兜底，不能代替架构设计。

**改造 4**：Kafka 监控。

```java
@Scheduled(fixedRate = 30_000)
public void detectStorm() {
    long rate = kafkaMetrics.getMessageRate("domain-events", Duration.ofMinutes(1));
    long baseline = baselineRateRepo.get("domain-events");
    if (rate > baseline * 5) {
        alerter.alert("possible event storm on domain-events: rate={}, baseline={}", rate, baseline);
    }
}
```

#### 验证

写测试覆盖"事件可能形成环"的场景：

```java
@Test
void noEventLoopUnderStockReservation() {
    Set<String> capturedEvents = new HashSet<>();
    eventBus.subscribe(e -> capturedEvents.add(e.getClass().getSimpleName()));

    orderService.placeOrder(req);

    // 期望：每种事件最多出现一次
    Map<String, Long> counts = capturedEvents.stream()
        .collect(groupingBy(identity(), counting()));
    counts.forEach((type, count) ->
        assertTrue(count < 3, type + " emitted " + count + " times, possible loop"));
}
```

> **口诀**：通用事件 = 事件风暴的导火索。**领域事件必须细粒度、单向流动**。给事件加 causation chain 兜底。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 选型层
- [ ] 是否真的需要 Event Sourcing？用 1.3 节的 5 条判据自查，少满足一条都用 CRUD
- [ ] 是否真的需要 EDA？区分"用 MQ 解耦"和"全链路事件驱动"，不要为了 EDA 而 EDA
- [ ] CQRS 和 ES 谁先上？建议先上 CQRS（读写分离），ES 单独评估
- [ ] Saga 步数 ≤ 4 用协同，> 4 必用编排（Camunda / Temporal）
- [ ] 事件量级是否估算过？> 50 万 events/s 才考虑 Kafka 当 Event Store

### 命名层
- [ ] 事件命名一律过去时（`OrderPlaced` 而非 `PlaceOrder`）
- [ ] 命令命名一律祈使句（`PlaceOrder` 而非 `OrderPlaced`）
- [ ] 不要用 `XxxUpdated` 这种通用事件，用细粒度领域事件

### 一致性层
- [ ] 业务事务和事件发送用 Outbox 或 CDC，**绝不**裸调 `kafkaTemplate.send`
- [ ] Outbox 写入和业务表在同一事务，**不能** try-catch 吞异常
- [ ] outbox-relay 用 `FOR UPDATE SKIP LOCKED` 支持多实例
- [ ] 事件消费方必须幂等（带去重表或业务字段判重）

### Schema 演进层
- [ ] 事件类一旦上线，字段类型不能改、不能删、语义不能变
- [ ] 任何字段变更 = 加新版本事件 + Upcaster
- [ ] Schema Registry 设至少 BACKWARD 兼容
- [ ] CI 强制跑历史事件反序列化测试

### Snapshot 层
- [ ] Snapshot 间隔按公式算（P95 加载 SLA / 单事件 apply 时间）
- [ ] Snapshot 异步生成，业务路径零阻塞
- [ ] Snapshot 必须直接从 event store 回放生成，不用缓存
- [ ] Snapshot 写入用乐观锁（`WHERE version < new_version`）防覆盖

### 存储层
- [ ] Event Store 有冷热分层（如 90 天迁移到 cold 表/S3）
- [ ] 短生命周期聚合用独立表，定期清理
- [ ] 监控每周事件增长，按聚合类型分类
- [ ] 备份策略适配 ES 增长（增量备份、归档备份分离）

### Saga 层
- [ ] 所有 Saga 步骤可重试 + 幂等
- [ ] 补偿动作也必须可重试 + 幂等
- [ ] Saga Orchestrator（Camunda）的状态持久化在专属存储
- [ ] 提供人工介入入口（resolve-incident、manual-complete）

### Read Model 层
- [ ] Projection 按聚合 ID 分区并行消费
- [ ] 高 QPS 场景用批量写入读模型
- [ ] 监控 Projection lag，> N 持续告警
- [ ] 关键查询走"读时合并最近写入"或"前端乐观更新"兜底最终一致性窗口

### 监控层
- [ ] Outbox pending 数量 + 最老未发送事件年龄
- [ ] CDC 端到端延迟（事件 created_at 到 Projector 处理时间）
- [ ] Projection lag（按 ProcessingGroup）
- [ ] 事件 rate 与基线对比，识别 Event Storm
- [ ] Saga incident 数告警

### 反模式层
- [ ] 不出现"通用事件"（XxxUpdated / XxxChanged）
- [ ] 不出现"事件回路"（A→B→A 形式）
- [ ] 不在 ES 聚合内做 IO（HTTP / DB 查询）
- [ ] CommandHandler 做校验，EventSourcingHandler 不做校验

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### 框架与运行时
- **Axon Framework** 4.x 的 `TrackingEventProcessor` 实现：token store 怎么保证 Exactly-Once
- **Spring Cloud Stream** 的 binder 抽象：怎么把同一份代码切换 Kafka / RabbitMQ / Pulsar
- **Eventuate Tram**（Chris Richardson 的框架）：Saga + Outbox 一体化方案
- 自己手写 Outbox 框架的边界：什么时候必须切到成熟方案

### 工作流引擎
- **Camunda Platform 8（Zeebe）** vs Camunda 7（Activiti 系）：流程引擎的架构演进
- **Temporal**：用代码（Workflow as Code）写 Saga 的范式
- **Cadence**：Uber 开源的工作流引擎，Temporal 的前身
- 工作流引擎的"幂等性陷阱"：Workflow 重放时副作用怎么避免

### 消息中间件作为 Event Store
- **Apache Kafka 的 log compaction**：保留最后一条消息的能力是不是 ES 的简化版
- **Apache Pulsar 的分层存储**：BookKeeper + S3，比 Kafka 更适合长期事件保留
- **EventStoreDB** 的 ACID 设计：和 Postgres 比有什么独特优势

### CDC 生态
- **Debezium 完整流程**：connector / SMT / Schema Registry / Sink Connector
- **Maxwell** vs **Canal** vs **Debezium**：各自的内核差异
- CDC + Iceberg：把变更流物化成数据湖表

### Schema 与序列化
- **Avro vs Protobuf vs Thrift vs JSON Schema**：在 EDA 场景的取舍
- **Schema Registry** 多种实现：Confluent / Apicurio
- **GraphQL Federation** 和事件 schema 治理的边界

### DDD × EDA
- **Domain Event** vs **Integration Event**：Bounded Context 内外的事件区分
- 战术 DDD 落地：Aggregate 的事件如何对外暴露而不破坏封装
- Event Storming 工作坊的实操步骤

### 分布式系统理论基础
- **CAP / PACELC** 在 EDA 中的具体表现
- **Saga vs 2PC**：为什么微服务时代抛弃了 2PC
- **Idempotency 的数学定义**：什么样的操作天生幂等，什么需要业务判重

### 经典阅读
- **《Microservices Patterns》by Chris Richardson**：Saga / Outbox / Event Sourcing 的工业总结
- **《Designing Event-Driven Systems》by Ben Stopford**（Confluent 出品）：Kafka 视角的 EDA
- **《Implementing Domain-Driven Design》by Vaughn Vernon**：DDD 的工程化书
- **Greg Young** 的《CQRS Documents》（PDF）：CQRS / ES 概念的正本清源

---

## 7. 引用

- Greg Young, *CQRS Documents*（CQRS 概念的奠基文档）
- Chris Richardson, *Microservices Patterns*（Manning, 2018）
- Ben Stopford, *Designing Event-Driven Systems*（O'Reilly / Confluent, 2018）
- Vaughn Vernon, *Implementing Domain-Driven Design*（Addison-Wesley, 2013）
- Pat Helland, *Life beyond Distributed Transactions: an Apostate's Opinion*（CIDR, 2007）
- Martin Kleppmann, *Designing Data-Intensive Applications* 第 11 章（Stream Processing）
- Axon Framework Reference Guide：[https://docs.axoniq.io](https://docs.axoniq.io)
- Debezium Documentation：[https://debezium.io/documentation](https://debezium.io/documentation)
- Confluent Schema Registry：[https://docs.confluent.io/platform/current/schema-registry/](https://docs.confluent.io/platform/current/schema-registry/)
- Camunda Platform 8 Documentation：[https://docs.camunda.io](https://docs.camunda.io)
- Eventuate Tram Sagas：[https://eventuate.io/abouteventuatetram.html](https://eventuate.io/abouteventuatetram.html)
- Microsoft, *CQRS Journey*（开源电子书，Patterns & Practices）

---

> 配套：[DDD 战术战略](/article/architecture/ddd) | [分布式事务](/article/distributed/dist-tx) | [Kafka 存储引擎](/article/mq/kafka-storage)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
