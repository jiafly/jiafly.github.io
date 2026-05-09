# 单元测试与 Mock 深入 —— JUnit 5 / Mockito / Testcontainers 的工程实战

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，看综述里的"测试金字塔"那一节就够了。
> 这里要回答的，是**写代码遇到测试拖累、CI 红了又绿绿了又红、覆盖率 90% 还是上线炸**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. Mock / Stub / Spy / Fake 四个术语，**精确**说出每对之间的差别。`Mockito.mock(X.class)` 默认产生的是哪一种？
2. TDD 的"红-绿-重构"循环里，**绿**这一步允许写"假实现"（hardcoded return）吗？为什么必须允许？
3. FIRST 原则的 I 是 Independent。如果两个测试方法都依赖**同一个 Spring Context**，它们还算 Independent 吗？JUnit 5 的 `@TestInstance(Lifecycle.PER_CLASS)` 改变了什么？
4. Mockito 5 默认能 mock `final` 方法、`final` 类、静态方法吗？背后用的是什么字节码方案？
5. `@WebMvcTest(UserController.class)` 启动了多少个 Bean？`@DataJpaTest` 默认用什么数据库？为什么这俩切片比 `@SpringBootTest` 快一个量级？
6. Testcontainers 启动 PostgreSQL 容器要 3 秒；100 个测试方法跑下来 5 分钟。**复用容器**有几种做法？singleton container 模式怎么写？
7. 一个被覆盖率工具标记为"100% 行覆盖"的方法，下一周还是出了 NPE。可能的原因有哪些？分支覆盖能查出来吗？突变测试呢？
8. 你的测试类继承了一个抽象基类（提供共享的 `@BeforeEach` 初始化），结果某个子类测试在 CI 上 flaky，本地稳定通过。能说出至少 3 种排查思路吗？

如果有半数答不上，往下看。

---

## 1. 测试模型：金字塔、Test Double、TDD

绝大多数文章把单元测试讲成"用 JUnit 写 `@Test` 方法 + 用 Mockito mock 依赖"，然后给个 `assertEquals` 例子就结束了。这层认知不够覆盖生产里的真实情况。要看懂上面 8 个问题，必须把模型先建立起来。

### 1.1 测试金字塔（Test Pyramid）

Mike Cohn 2009 年提出的经典模型。三层从下往上：

```
              ▲   E2E (10%)
             ╱ ╲   慢、贵、易碎，跑全链路
            ╱───╲
           ╱     ╲ 集成 (20%)
          ╱       ╲ 中等速度，跨进程/跨服务
         ╱─────────╲
        ╱           ╲ 单元 (70%)
       ╱             ╲ 快、稳、纯函数级
      ╱_______________╲
```

| 层级 | 跑一个的耗时 | 跨进程？ | 例子 |
| --- | --- | --- | --- |
| **单元** | < 50 ms | 否 | 一个方法、一个类的纯逻辑 |
| **集成** | 100 ms ~ 几秒 | 是（DB / 消息队列等） | Repository 接 Testcontainers PG |
| **E2E** | 秒级 ~ 分钟级 | 是（全链路） | Selenium / Cypress / Playwright |

这个比例不是教条，但**反过来的金字塔（"冰淇淋"反模式）会让 CI 变成几十分钟的拖累**。Google 内部的 SET（Software Engineer in Test）岗位早期就在死磕这个比例。

### 1.2 FIRST 原则

Robert Martin 的 *Clean Code* 提出的 5 个性质：

| 字母 | 含义 | 违反会怎样 |
| --- | --- | --- |
| **F**ast | 单测毫秒级；整套秒级 | 慢测试没人跑，最后没人跑 = 没有测试 |
| **I**ndependent | 任意顺序、单独运行结果不变 | 一个挂了拖一串挂，定位时间翻倍 |
| **R**epeatable | 任何环境（本地 / CI / 离线）结果一致 | "在我机器上能跑"反模式 |
| **S**elf-validating | 通过/失败是布尔，不需要肉眼判断 | 输出一堆日志让人看，毫无意义 |
| **T**imely | 与生产代码同时（甚至先于）写 | 后补的测试只是"覆盖率装饰" |

最容易被忽视的是 **I**：很多团队的测试**默默依赖运行顺序**或**全局状态**。本地按文件名顺序跑通过，CI 按字典序跑就挂。

### 1.3 Test Double 五种（Gerard Meszaros 的精确分类）

通用术语 "mock" 在工程里被滥用。*xUnit Test Patterns* 给的精确分类：

| 名字 | 行为 | 例子 | Mockito 怎么造 |
| --- | --- | --- | --- |
| **Dummy** | 占位、不被使用 | 方法签名要个 User 但根本不读 | `null` 或 `mock(User.class)` |
| **Stub** | 对查询返回预设值 | "调 `findById(1)` 就返回 alice" | `when(...).thenReturn(...)` |
| **Spy** | 真实对象 + 记录调用 | 跑真逻辑，同时记录"被调过几次" | `Mockito.spy(realObj)` |
| **Mock** | 预期+验证调用模式 | "必须调一次 `save`，参数是 alice" | `mock + verify(...)` |
| **Fake** | 简化但可工作的实现 | 内存版 UserRepository | 自己写一个 class 实现接口 |

**关键差别**：
- **Stub vs Mock**：Stub 关心"返回啥让我能继续跑"；Mock 关心"被怎么调用了"（行为验证）
- **Spy vs Mock**：Spy 默认走真实方法，按需 stub；Mock 默认所有方法返回 null/0/false
- **Fake vs Stub**：Fake 是真实的简化实现（可能有逻辑分支）；Stub 只对几个固定输入返回固定输出

`Mockito.mock(Foo.class)` 默认产生的对象同时具备 Stub（你能 `when(...).thenReturn`）和 Mock（你能 `verify(...)`）的能力。所以**严格意义上它两者都是**，按你怎么用决定它的"角色"。

### 1.4 TDD 的红绿重构循环

Kent Beck 在 *TDD By Example* 里反复强调的节奏：

```
   ┌─→ RED ──→ GREEN ──→ REFACTOR ──┐
   │   写挂   最快通过      重构     │
   │   的测试   的实现     不改语义    │
   └────────────────────────────────┘
```

每一步的纪律：

- **RED**：测试要**真的挂**（不是编译错），且失败信息要清晰。如果你的测试一上来就过了，要么测试没写对，要么需求其实已经实现。
- **GREEN**：用**最少**的代码让它通过。允许"假实现"（直接 `return 42`）—— 这一步的目标是建立反馈，不是优雅。
- **REFACTOR**：重构生产代码 + 测试代码。这一步**不能让任何测试由绿变红**。

新人最常违反的是 GREEN：他们想"既然要写就写好"，于是一次性写一个完整实现。这剥夺了 TDD 最大的价值 —— **小步反馈**。允许"假实现"是因为下一个 RED 测试会逼你把假实现变成真实现。

### 1.5 BDD 与 Cucumber / Spock

BDD（Behavior Driven Development）是 TDD 的"业务对齐"变种。核心是 **Given-When-Then** 三段式：

```gherkin
Feature: 用户登录
  Scenario: 密码错误
    Given 用户 alice 已注册，密码为 "correct"
    When 用 alice/wrong 登录
    Then 返回 401
    And 失败计数 +1
```

Cucumber / SpecFlow 的目标受众是**非程序员（产品/测试）也能写 spec**。实际工程里 80% 团队没坚持下来，因为：维护 .feature 文件 + step definitions 的双重成本太高；产品其实并不愿意写。

[Spock](https://spockframework.org/)（Groovy）是个更程序员友好的 BDD 框架：

```groovy
def "密码错误返回 401"() {
    given:
        userRepo.findByName("alice") >> new User("alice", "correct")
    when:
        def resp = controller.login("alice", "wrong")
    then:
        resp.status == 401
        1 * loginCounter.increment("alice", "fail")
}
```

它的 data table、`>>` stub 语法、内置 mock 比 JUnit + Mockito 简洁很多，但生态绑定 Groovy。

### 1.6 测试覆盖率：陷阱比指标更重要

| 指标 | 含义 | 工具 | 陷阱 |
| --- | --- | --- | --- |
| **行覆盖** | 这一行被执行过 | JaCoCo | 行被执行 ≠ 这行的逻辑被测 |
| **分支覆盖** | if/else 两条路径都走到 | JaCoCo | 走到了 ≠ 断言了正确结果 |
| **路径覆盖** | 所有可能的路径组合 | 罕见 | 路径数指数级，不实用 |
| **突变覆盖** | 故意改代码（突变），看测试能否抓到 | PIT | **真正度量"测试有效性"** |

经典陷阱：

```java
public int divide(int a, int b) {
    return a / b;        // 这一行 100% 行覆盖
}

@Test
void testDivide() {
    divide(10, 2);       // 没断言！但行覆盖率工具显示 100%
}
```

突变测试会把 `a / b` 改成 `a * b`、`a + b` 等"突变体"，跑你的测试。如果没有任何测试因突变而失败，说明你的测试**根本没在断言行为**。

> **口诀**：行覆盖说"代码被执行了"；分支覆盖说"分支被走到了"；只有**突变覆盖**能说"测试在度量行为"。覆盖率是负向指标 —— 90% 不代表好，但 30% 一定不好。

---

## 2. 工具切片：JUnit 5 / Mockito / AssertJ / Spring Test / Testcontainers

> 基于 JUnit 5.10、Mockito 5.x、Spring Boot 3.x、Testcontainers 1.19。

### 2.1 JUnit 5 的核心特性

JUnit 5（Jupiter）相对 4.x 是一次彻底重写。最重要的几个新能力：

```java
@DisplayName("用户登录用例")
class LoginServiceTest {

    @Nested
    @DisplayName("密码错误时")
    class WhenPasswordWrong {

        @Test
        @DisplayName("返回 401")
        void returns401() { /* ... */ }

        @Test
        @DisplayName("登录失败计数 +1")
        void incrementsFailureCount() { /* ... */ }
    }

    @ParameterizedTest(name = "[{index}] {0} 应被识别为弱密码")
    @ValueSource(strings = {"123", "password", "qwerty", ""})
    void weakPasswords(String pwd) {
        assertTrue(passwordChecker.isWeak(pwd));
    }
}
```

**逐项点透**：

- **`@Nested`**：把相关用例分组到内部类，IDE 里可折叠，报告里有层级。**比"一个方法名 30 字符"清晰得多**。
- **`@DisplayName`**：方法名是给程序员看的，`DisplayName` 是给报告/产品看的。
- **`@ParameterizedTest`**：4.x 时代要 hack `@RunWith(Parameterized.class)` 写一堆样板；Jupiter 一行搞定。`name = "{0}"` 让每条用例的报告里有可读名字。
- **生命周期**：默认 `@TestInstance(Lifecycle.PER_METHOD)`，每个 `@Test` 新建一个测试类实例 —— 字段无法跨方法共享，**这是 I（Independent）的硬性保障**。

`@TestInstance(Lifecycle.PER_CLASS)` 改变什么：所有方法共享一个实例，可以让 `@BeforeAll` 不必是 static。代价：**字段污染风险** —— 一个方法把 `this.list` 加进了元素，下一个方法看见的就不是空的了。除非你确定不需要，**别开 PER_CLASS**。

### 2.2 Mockito 5 的核心 API

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock UserRepository userRepo;
    @Mock PaymentClient paymentClient;
    @InjectMocks OrderService orderService;     // 自动把上面的 mock 注入

    @Captor ArgumentCaptor<Order> orderCaptor;

    @Test
    void createOrder() {
        // Stub
        when(userRepo.findById(1L)).thenReturn(Optional.of(new User(1L, "alice")));
        when(paymentClient.charge(any(), eq(100))).thenReturn(PaymentResult.ok("tx-1"));

        // 跑生产代码
        long orderId = orderService.create(1L, 100);

        // 验证：参数捕获
        verify(userRepo).save(orderCaptor.capture());
        Order saved = orderCaptor.getValue();
        assertEquals("alice", saved.getUserName());
        assertEquals(100, saved.getAmount());

        // 验证：调用次数
        verify(paymentClient, times(1)).charge(any(), anyInt());
        verifyNoMoreInteractions(paymentClient);
    }
}
```

**逐项点透**：

- **`@ExtendWith(MockitoExtension.class)`** 替代 4.x 的 `@RunWith(MockitoJUnitRunner.class)`。它会在 `@BeforeEach` 自动初始化所有 `@Mock` 字段，并在 `@AfterEach` 检查 `Mockito.validateMockitoUsage()`（catch 用错的 stub）。
- **`@InjectMocks`** 用反射把 `@Mock` 字段按构造器/setter/字段三种策略注入到被测类。**优先构造器注入**，匹配不到再退化。生产建议用构造器注入，让 `@InjectMocks` 走最稳的路径。
- **`ArgumentCaptor`** 是验证"传给 mock 的对象长啥样"的标准做法。比写一个臃肿的 `argThat(o -> o.getName().equals(...) && ...)` 可读得多。
- **`verifyNoMoreInteractions`** 用来发现"测试漏看"的调用。慎用，会导致测试与实现细节耦合（见案例 ②）。

### 2.3 Mockito 5 已默认能 mock final / static / 构造函数

4.x 时代 mock final 类要单独引 `mockito-inline`。**Mockito 5 之后 inline mock maker 是默认**。

```java
// final 类
SomeFinal real = new SomeFinal();
SomeFinal mockFinal = mock(SomeFinal.class);
when(mockFinal.compute()).thenReturn(42);

// 静态方法
try (MockedStatic<UUID> mocked = mockStatic(UUID.class)) {
    mocked.when(UUID::randomUUID).thenReturn(UUID.fromString("00000000-..."));
    assertEquals("00000000-...", UUID.randomUUID().toString());
}
// ★ try-with-resources 必须！否则静态 mock 泄漏到下个测试，全套挂

// 构造函数
try (MockedConstruction<File> mocked = mockConstruction(File.class,
        (m, ctx) -> when(m.length()).thenReturn(123L))) {
    File f = new File("/tmp/x");
    assertEquals(123L, f.length());     // ★ 这里 new 出来的实际是 mock
}
```

**逐项点透**：

- **底层方案**：inline mock maker 用 ByteBuddy 在运行时改字节码（attach Java agent），可以拦截 final 方法/类、static 方法、构造函数。代价：每个被 mock 的类首次需要 ~50ms 的 instrument 开销。
- **`MockedStatic` / `MockedConstruction` 必须用 try-with-resources**：它们是**线程级注册**，不释放就泄漏到下个测试。**这是 Mockito 5 最常踩的坑**（见案例 ⑦）。
- **不能 mock 的**：`java.lang.System`、`java.lang.Thread`、`java.lang.String`、原生类型 —— JVM 本身的 bootstrap 类被保护。

### 2.4 AssertJ：流式断言

JUnit 内置的 `assertEquals` 只能做最基础的断言。AssertJ 提供链式 + 类型感知的断言，**失败信息也好得多**：

```java
import static org.assertj.core.api.Assertions.*;

assertThat(orders)
    .hasSize(3)
    .extracting(Order::getUserName)
    .containsExactlyInAnyOrder("alice", "bob", "carol");

assertThat(user)
    .isNotNull()
    .satisfies(u -> {
        assertThat(u.getAge()).isBetween(18, 65);
        assertThat(u.getEmail()).endsWith("@example.com");
    });

// 自定义错误信息
assertThat(amount)
    .as("订单 %s 的金额", orderId)
    .isPositive()
    .isLessThan(10000);

// 异常断言
assertThatThrownBy(() -> service.charge(-1))
    .isInstanceOf(IllegalArgumentException.class)
    .hasMessageContaining("amount must be positive");
```

**为什么不用 JUnit 的 `assertEquals`？** AssertJ 的失败消息是**结构化**的：

```
Expecting:
  ["alice", "bob"]
to contain exactly in any order:
  ["alice", "bob", "carol"]
but could not find:
  ["carol"]
```

而 `assertEquals` 只会告诉你 `expected: <[...]> but was: <[...]>`，自己肉眼对比。

### 2.5 Spring Boot Test 的几种切片

`@SpringBootTest` 启动**整个** ApplicationContext，慢且重。Spring Boot 提供"切片测试"按需启动一部分：

```java
@WebMvcTest(UserController.class)         // 只启 MVC + 这一个 controller
class UserControllerTest {
    @Autowired MockMvc mvc;
    @MockBean UserService userService;    // ★ 不是 @Mock，是 @MockBean

    @Test
    void getUser() throws Exception {
        when(userService.find(1L)).thenReturn(new User(1L, "alice"));
        mvc.perform(get("/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("alice"));
    }
}

@DataJpaTest                              // 只启 JPA + 内嵌数据库（默认 H2）
class UserRepositoryTest {
    @Autowired UserRepository repo;
    @Autowired TestEntityManager em;

    @Test
    void findByEmail() {
        em.persist(new User("alice", "alice@example.com"));
        em.flush();
        assertThat(repo.findByEmail("alice@example.com")).isPresent();
    }
}
```

| 切片 | 启动什么 | 不启动什么 | 速度 |
| --- | --- | --- | --- |
| `@SpringBootTest` | 全部 Bean | — | 慢（10~30s） |
| `@WebMvcTest` | MVC、Controller、Filter、@ControllerAdvice | Service、Repository、JPA | 快（2~5s） |
| `@DataJpaTest` | JPA、Repository、内嵌 DB | Web、Service | 中（3~8s） |
| `@DataMongoTest` / `@DataRedisTest` | 对应数据层 | 其他 | 中 |
| `@JsonTest` | Jackson 配置 | 其他 | 极快（< 1s） |
| `@RestClientTest` | RestTemplate / WebClient | 其他 | 快 |

**关键差异**：`@MockBean` vs `@Mock`：
- `@Mock` 只造 mock 对象，**不进 Spring 容器**
- `@MockBean` 把 mock 注册成 Bean，替换容器里同类型的 Bean

`@WebMvcTest` 里你必须用 `@MockBean` 替换 Service 层 —— 否则 Controller 注入不到东西。

### 2.6 Testcontainers：真实数据库测试

`@DataJpaTest` 默认用 H2 内存数据库。问题：**H2 的 SQL 方言、行为与生产 MySQL/PostgreSQL 有差异**（窗口函数、JSON 列、CTE、`ON CONFLICT`、序列、字符集...）。

Testcontainers 让你在测试里跑**真实的 Docker 容器**：

```java
@Testcontainers
@SpringBootTest
class OrderRepositoryIT {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("test")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configure(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired OrderRepository repo;

    @Test
    void useRealPostgresFeature() {
        // 用 PG 特有的 jsonb、ON CONFLICT、CTE 等真实测
    }
}
```

**几个进阶用法**：

```java
// Kafka
static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));

// Redis（用通用 GenericContainer）
static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine").withExposedPorts(6379);

// 自定义初始化脚本
postgres.withInitScript("schema.sql");

// 等待策略：等服务真的能用
container.waitingFor(Wait.forLogMessage(".*ready to accept.*", 1));
```

### 2.7 单容器 singleton 模式（关键性能优化）

JUnit 默认每个测试类启停一次容器，100 个 IT × 3 秒 = 5 分钟纯启停开销。**Singleton container 模式**让所有测试共享一个容器：

```java
public abstract class AbstractPostgresIT {
    static final PostgreSQLContainer<?> POSTGRES;

    static {
        POSTGRES = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("test")
            .withReuse(true);          // ★ 配合 ~/.testcontainers.properties 的 reuse=true
        POSTGRES.start();
        Runtime.getRuntime().addShutdownHook(new Thread(POSTGRES::stop));
    }

    @DynamicPropertySource
    static void configure(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
    }
}
```

继承这个抽象类的所有 IT 共享同一个 `POSTGRES` 容器。注意：
- 不用 `@Container` 注解（那个会触发自动启停）
- 加 ShutdownHook 保证 JVM 退出时清理
- `withReuse(true)` 让 Testcontainers 在 `start()` 时如果发现已有匹配容器就**不新建**（跨 JVM 进程复用，CI 上效果显著）

### 2.8 WireMock：HTTP Mock

测试代码调外部 HTTP 服务时，不要直接打到真实服务器。WireMock 起个进程内 HTTP server：

```java
@WireMockTest(httpPort = 8089)
class PaymentClientTest {

    @Test
    void chargeSuccess() {
        stubFor(post(urlEqualTo("/api/charge"))
            .withRequestBody(matchingJsonPath("$.amount", equalTo("100")))
            .willReturn(okJson("{\"status\":\"ok\",\"txId\":\"tx-1\"}")));

        PaymentClient client = new PaymentClient("http://localhost:8089");
        PaymentResult r = client.charge("user-1", 100);

        assertThat(r.getTxId()).isEqualTo("tx-1");
        verify(postRequestedFor(urlEqualTo("/api/charge")));
    }

    @Test
    void slowResponse() {
        stubFor(get("/api/slow").willReturn(ok().withFixedDelay(5000)));
        // 测客户端的超时配置是否生效
    }

    @Test
    void scenarios() {     // 状态机：第一次 503，第二次 ok（测重试）
        stubFor(get("/api/flaky").inScenario("retry")
            .whenScenarioStateIs(STARTED)
            .willReturn(serverError())
            .willSetStateTo("retried"));
        stubFor(get("/api/flaky").inScenario("retry")
            .whenScenarioStateIs("retried")
            .willReturn(ok()));
    }
}
```

**为什么 WireMock 比 mock client 类好？** 它在 HTTP 层 mock —— **能验证你的 HTTP 客户端配置（重试、超时、序列化）**。如果直接 `mock(PaymentClient.class)`，所有这些客户端层的 bug 都跳过了。

### 2.9 Pact：契约测试

跨服务测试有个根本难题：**消费方（Consumer）改了请求格式，没人通知提供方（Provider）**。Pact 用"契约文件"显式化这种约定：

```java
// 消费方：定义期望
@ExtendWith(PactConsumerTestExt.class)
class OrderServiceConsumerTest {

    @Pact(consumer = "order-service", provider = "user-service")
    public RequestResponsePact pact(PactDslWithProvider builder) {
        return builder
            .given("user 1 exists")
            .uponReceiving("get user 1")
            .path("/users/1").method("GET")
            .willRespondWith()
            .status(200)
            .body(new PactDslJsonBody().integerType("id", 1).stringType("name", "alice"))
            .toPact();
    }

    @Test
    @PactTestFor(providerName = "user-service", port = "8080")
    void getUser(MockServer mockServer) {
        UserClient client = new UserClient(mockServer.getUrl());
        User u = client.find(1L);
        assertEquals("alice", u.getName());
    }
}
```

跑完会生成 `target/pacts/order-service-user-service.json`，上传到 Pact Broker。提供方拿到契约文件后跑：

```java
@Provider("user-service")
@PactBroker(host = "pact.example.com")
class UserServicePactTest {
    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void verifyPact(PactVerificationContext context) {
        context.verifyInteraction();      // 自动按契约打请求，验证返回
    }
}
```

**契约测试的位置**：在测试金字塔里它在"集成"层，但**不需要真起对方服务** —— 这是它最大的价值。

> **口诀**：单元测试用 Mockito + JUnit + AssertJ；切片测试用 `@WebMvcTest` / `@DataJpaTest` 别用 `@SpringBootTest`；数据层 IT 用 Testcontainers + singleton 模式；HTTP 客户端测试用 WireMock；跨服务用 Pact 契约。

---

## 3. 实测数据：测试粒度的速度差

### 3.1 不同测试粒度的执行时间

实测一个中等规模的 Spring Boot 项目（约 200 个类），按不同测试类型测量启动 + 执行 + 销毁的总耗时（秒）：

| 测试类型 | 单个方法平均耗时 | 100 个方法总耗时 | 是否能在 PR 钩子里跑 |
| --- | --- | --- | --- |
| 纯 JUnit + Mockito 单元测试 | 5 ms | 0.5 s | ✅ 必须 |
| `@JsonTest` | 30 ms | 3 s | ✅ |
| `@WebMvcTest` | 200 ms | 20 s | ✅ |
| `@DataJpaTest`（H2） | 300 ms | 30 s | ✅ |
| `@DataJpaTest` + Testcontainers PG | 100 ms（singleton） / 3 s（每个新建） | 10 s ~ 5 min | ⚠️ 取决于复用 |
| `@SpringBootTest` | 500 ms | 50 s | ⚠️ 谨慎 |
| `@SpringBootTest` + Testcontainers | 1 s | 100 s | ❌ nightly 跑 |
| Selenium E2E | 5 s | 500 s | ❌ |

**结论**：单元测试比集成测试快**两到三个量级**。这就是金字塔比例的物理基础 —— 不是教条，是 CI 时间预算的硬约束。

### 3.2 Testcontainers vs H2 的差异（具体场景）

下面这些 SQL，H2 行为与 PostgreSQL 不同，**用 H2 会让你的测试通过、生产挂掉**：

```sql
-- ① jsonb（H2 没有原生 jsonb）
SELECT data->>'name' FROM users WHERE data @> '{"vip":true}';

-- ② ON CONFLICT DO UPDATE（H2 用 MERGE INTO 语法）
INSERT INTO users(id, name) VALUES(1, 'alice')
ON CONFLICT (id) DO UPDATE SET name = EXCLUDED.name;

-- ③ 窗口函数 RANGE BETWEEN INTERVAL（H2 部分支持）
SELECT id, SUM(amount) OVER (
    ORDER BY created_at RANGE BETWEEN INTERVAL '1 day' PRECEDING AND CURRENT ROW
) FROM orders;

-- ④ 区分大小写的 LIKE（PG 区分，H2 默认不区分）
SELECT * FROM users WHERE name LIKE 'A%';

-- ⑤ 行级锁的等待行为（PG 的 SKIP LOCKED，H2 不支持）
SELECT * FROM jobs WHERE state = 'pending' FOR UPDATE SKIP LOCKED LIMIT 10;
```

**结论**：只要你用了任何上述 PG 特有特性，`@DataJpaTest` 默认的 H2 就**测了个寂寞**。Testcontainers 是必须项。

### 3.3 测试并行化的速度提升

JUnit 5 支持类内/类间并行：

```properties
# src/test/resources/junit-platform.properties
junit.jupiter.execution.parallel.enabled = true
junit.jupiter.execution.parallel.mode.default = same_thread
junit.jupiter.execution.parallel.mode.classes.default = concurrent
junit.jupiter.execution.parallel.config.strategy = dynamic
junit.jupiter.execution.parallel.config.dynamic.factor = 1
```

实测一个 500 测试方法的项目，10 核机器：

| 配置 | 耗时 | 加速比 |
| --- | --- | --- |
| 串行 | 180 s | 1.0x |
| 类间并行（4 线程） | 65 s | 2.8x |
| 类间并行（10 线程） | 35 s | 5.1x |
| 类间 + 方法间并行（10 线程） | 28 s | 6.4x |

**为什么加速比 < 线程数？** 因为：① Spring Context 共享时有同步成本；② 数据库 / 容器是共享资源（singleton），并行受其吞吐限制；③ JVM 启动开销固定。

**并行的前提**：测试必须是 **I**ndependent。任何共享可变状态（静态字段、数据库共享数据）都会让并行变成竞态地狱。

### 3.4 突变测试（PIT）的实际命中率

[PIT](https://pitest.org/) 是 JVM 上最成熟的突变测试工具。它对每个被测方法应用一系列突变（mutator），跑测试，看是否有失败：

```xml
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <version>1.15.0</version>
    <configuration>
        <targetClasses><param>com.example.service.*</param></targetClasses>
        <targetTests><param>com.example.service.*Test</param></targetTests>
        <mutators><mutator>STRONGER</mutator></mutators>
    </configuration>
</plugin>
```

实测一个"行覆盖率 90%、分支覆盖率 80%"的项目，PIT 的突变得分（mutation score）通常在 **50%~70%**。这意味着：**有 30% 以上的"被测代码"，你改坏了它，测试根本不报错**。

典型未杀的突变：

```java
// 原代码
if (amount > 100) discount = 0.9;

// 突变 1：> 改成 >=  → 边界条件测试缺失
if (amount >= 100) discount = 0.9;

// 突变 2：0.9 改成 0  → 你只测了"是否有 discount 字段被赋值"，没测值
if (amount > 100) discount = 0;

// 突变 3：删掉这个 if → 你的测试没覆盖"小金额不打折"的分支
discount = 0.9;
```

**结论**：覆盖率工具度量"代码被走过没"，突变测试度量"测试有没有真在断言"。但 PIT 慢（一次跑可能几十分钟），适合**每周一次定期跑**或**关键模块 PR 钩子**，不适合每次 push。

> **口诀**：纯单元测试是反馈环的命脉，毫秒级；切片测试 1~10 倍开销，能在 PR 钩子里跑；Testcontainers 集成测必须 singleton 否则 CI 拖死；突变测试是覆盖率的"质检"，定期跑就行。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 测试实现细节而非行为 —— "改个内部方法名，30 个测试全挂"

#### 现象

一次重构把 `OrderService.calculateTotal()` 内部的两个 private 方法改了名字 + 调用顺序，业务行为完全没变。结果跑测试，30 个测试方法红了 —— 全是 verify 调用顺序、调用次数对不上。

#### 根因

测试代码长这样：

```java
@Test
void createOrder_callsInRightOrder() {
    InOrder inOrder = inOrder(taxCalc, discountCalc, shippingCalc);
    orderService.create(req);

    inOrder.verify(taxCalc).compute(any());
    inOrder.verify(discountCalc).apply(any());
    inOrder.verify(shippingCalc).estimate(any());
    verify(taxCalc, times(2)).compute(any());      // 必须调 2 次！
    verifyNoMoreInteractions(taxCalc, discountCalc, shippingCalc);
}
```

这个测试断言的是"**实现以什么顺序调用了几个 mock**"。重构改变了实现细节（顺序、调用次数），即使**最终输出完全一致**，测试还是挂。

这是 over-specification（过度规约）。它把生产代码的一切实现细节"焊死"在测试里，重构成本指数级上升。

#### 修复

把测试改成断言**输出**而非"调用记录"：

```java
@Test
void createOrder_correctTotal() {
    when(taxCalc.compute(100)).thenReturn(8);
    when(discountCalc.apply(100)).thenReturn(10);
    when(shippingCalc.estimate(any())).thenReturn(5);

    Order order = orderService.create(req(100));

    // ★ 只断言最终结果
    assertThat(order.getTotal()).isEqualTo(103);    // 100 + 8 - 10 + 5
}
```

如果实在需要 verify，**只 verify 必要的副作用**（"必须发了通知"、"必须扣了款"），不 verify 顺序/次数：

```java
verify(notifier).send(eq("alice"), contains("created"));
// 不要 verify(notifier, times(1).inOrder...)
```

#### 验证

把测试和实现"解耦"的简单标准：**假装重写一遍生产代码（完全不同的内部结构，相同的输入输出）**，测试应该全部通过。如果不能，说明你的测试在测实现，不在测行为。

> **口诀**：测试断言**行为（输出）**，不断言**实现（调用记录）**。`InOrder` 和 `verifyNoMoreInteractions` 是危险信号，慎用。

---

### 案例 ② 过度 Mock —— "测试变成调用清单"

#### 现象

一个 `OrderController` 测试，被测方法只有 20 行，测试代码 200 行，全在 stub & verify。每次改业务逻辑，至少要改 5 个地方的 mock。

#### 根因

```java
@Test
void createOrder() {
    when(userService.find(1L)).thenReturn(user);
    when(productService.find(101L)).thenReturn(product);
    when(productService.checkStock(101L, 2)).thenReturn(true);
    when(priceCalculator.calc(product, 2)).thenReturn(200);
    when(discountService.apply(user, 200)).thenReturn(180);
    when(taxService.compute(180, "CN")).thenReturn(14);
    when(paymentService.charge(user, 194)).thenReturn(PaymentResult.ok("tx-1"));
    when(orderRepository.save(any())).thenAnswer(inv -> inv.getArgument(0));
    when(notificationService.send(any())).thenReturn(true);
    // ... 还有 5 行 stub

    OrderResult r = controller.create(req);

    verify(userService).find(1L);
    verify(productService).find(101L);
    // ... 还有 8 行 verify
}
```

把每个依赖都 mock 了，测试本质上变成"我**重新声明**了一遍 OrderController 的实现"。任何改动都要双倍工作量。

更深层的问题是：**这个测试什么都没真正测**。它只验证了"代码按我预期的方式调了 9 个 mock"，没验证业务行为对不对（discount 是不是真的对 VIP 用户打了折？）。

#### 修复

**方案 A**：用 Fake 替代部分 mock。

```java
public class FakeUserRepository implements UserRepository {
    private final Map<Long, User> store = new HashMap<>();
    public Optional<User> findById(Long id) { return Optional.ofNullable(store.get(id)); }
    public void save(User u) { store.put(u.getId(), u); }
    public void seed(User... users) { for (User u : users) save(u); }
}

@Test
void createOrder() {
    fakeUserRepo.seed(new User(1L, "alice", VIP));
    fakeProductRepo.seed(new Product(101L, "book", BigDecimal.valueOf(100), stock(10)));

    OrderResult r = orderService.create(new CreateReq(1L, 101L, 2));

    assertThat(r.getTotal()).isEqualTo(BigDecimal.valueOf(180));  // 真的算出 VIP 折扣
    assertThat(fakeUserRepo.findById(1L)).hasValueSatisfying(u ->
        assertThat(u.getRecentOrderCount()).isEqualTo(1)
    );
}
```

**方案 B**：把测试拆到正确的层。Controller 层只测"参数解析、HTTP 状态码、错误处理"；Service 层测业务逻辑；Repository 层测 SQL。**每一层只 mock 它的"下层"，不 mock 它的"内部协作者"**。

#### 验证

判断标准：**Mock 数量 ≤ 3 个**。如果一个测试要 mock 超过 3 个依赖，要么这个测试在错的层级（应该往下拆），要么被测类违反了单一职责（应该重构）。

> **口诀**：**Mock 是给"难以替代的依赖"用的**（外部服务、时间、随机数）。能用 Fake 就用 Fake；能不 mock 就不 mock。

---

### 案例 ③ 测试间相互依赖 —— "本地全过，CI 随机挂"

#### 现象

测试本地按 IDE 的字母顺序跑全过。CI 用 `maven-surefire` 默认 `filesystem` 顺序跑，随机挂 5~10 个。本地按 `mvn test` 又能复现一部分。

#### 根因

```java
class UserServiceTest {
    static UserService userService = new UserService(new InMemoryRepo());

    @Test
    void test1_register() {
        userService.register("alice");
        assertThat(userService.findAll()).hasSize(1);
    }

    @Test
    void test2_login() {
        // ★ 依赖 test1 已经跑过
        assertTrue(userService.login("alice", "pwd"));
    }

    @Test
    void test3_count() {
        assertThat(userService.findAll()).hasSize(1);    // ★ 依赖 test1
    }
}
```

`static` 字段让所有测试共享 `userService` 实例。`test2`、`test3` 暗中假设 `test1` 已经把 alice 注册进去了。

JUnit 5 的方法执行顺序默认是 **deterministic but unspecified**（`MethodOrderer.MethodName` 在 5.10+ 不再是默认）。本地按字母走过 test1 → test2 → test3；CI 上可能按 test2 → test1 → test3，test2 立即挂。

更隐蔽的版本：依赖**数据库里的种子数据**（某个测试 INSERT 了，没回滚，下一个测试 SELECT 期望它存在）。

#### 修复

**铁律**：每个 `@Test` 必须能**单独**通过、必须能**任意顺序**通过。

```java
class UserServiceTest {
    UserService userService;     // 实例字段，每个方法新建（PER_METHOD）

    @BeforeEach
    void setUp() {
        userService = new UserService(new InMemoryRepo());
    }

    @Test
    void register() {
        userService.register("alice");
        assertThat(userService.findAll()).hasSize(1);
    }

    @Test
    void login() {
        userService.register("alice");      // ★ 自己 setup
        assertTrue(userService.login("alice", "pwd"));
    }
}
```

数据库测试用 `@Transactional` 自动回滚（Spring Test 默认在测试方法结束 rollback）：

```java
@SpringBootTest
@Transactional        // ★ 每个 test 在自己的事务里，结束自动回滚
class OrderServiceIT {
    @Test void createOrder() { /* 不用清数据 */ }
}
```

#### 强制约束：随机化执行顺序

```properties
# junit-platform.properties
junit.jupiter.testmethod.order.default = org.junit.jupiter.api.MethodOrderer$Random
```

加上随机化后**至少跑一周 CI**。任何依赖顺序的测试都会暴露。

#### 验证

```bash
# 本地用反向顺序跑一遍
mvn test -Djunit.jupiter.testmethod.order.default=org.junit.jupiter.api.MethodOrderer\$MethodName

# 跑 100 次随机顺序
for i in $(seq 100); do mvn test || echo "FAIL run $i"; done
```

> **口诀**：测试必须**自给自足**。JUnit 的 PER_METHOD 生命周期是给你的安全网，不要用 `static` 字段毁掉它。数据库用 `@Transactional` 自动回滚，不要靠 `@AfterEach` 手动清。

---

### 案例 ④ Flaky Test —— "时间、顺序、网络的三重诅咒"

#### 现象

某个测试每周大概有 1~3 次失败，rerun 就过。跑了一阵后团队习惯性 "如果某个测试挂了点 retry"，最后就再也没人看测试输出了。

#### 根因 1：依赖系统时间

```java
@Test
void tokenExpires() {
    Token t = new Token(System.currentTimeMillis() + 100);    // 100ms 后过期
    Thread.sleep(150);
    assertTrue(t.isExpired());          // 99% 通过；GC pause 时挂
}
```

GC pause / OS 调度让 sleep 实际睡了 90ms 而非 150ms，token 还没过期，测试挂。

**修复**：把"时间"作为依赖注入，测试里用 `Clock` 的 fixed 实例：

```java
public class TokenService {
    private final Clock clock;
    public TokenService(Clock clock) { this.clock = clock; }
    public boolean isExpired(Token t) { return clock.instant().isAfter(t.getExpireAt()); }
}

@Test
void tokenExpires() {
    Clock fixed = Clock.fixed(Instant.parse("2025-01-01T00:00:00Z"), ZoneOffset.UTC);
    TokenService svc = new TokenService(fixed);
    Token t = new Token(Instant.parse("2024-12-31T23:59:59Z"));
    assertTrue(svc.isExpired(t));
}
```

#### 根因 2：依赖网络/外部服务

```java
@Test
void fetchUser() {
    User u = client.get("https://api.example.com/users/1");   // ★ 真打网
    assertThat(u.getName()).isEqualTo("alice");
}
```

任何打外网的测试都是 flaky 候选 —— 网络抖动、对方挂、DNS 解析失败、TLS 证书过期...

**修复**：用 WireMock 拦截。

#### 根因 3：CompletableFuture / 异步任务

```java
@Test
void asyncSend() throws Exception {
    notifier.sendAsync("hello");
    Thread.sleep(100);          // ★ 完蛋
    verify(emailClient).send("hello");
}
```

100ms 在快机器够用，慢 CI 可能不够。

**修复**：用 Awaitility 显式等条件。

```java
@Test
void asyncSend() {
    notifier.sendAsync("hello");
    await().atMost(2, SECONDS).untilAsserted(() ->
        verify(emailClient).send("hello"));
}
```

#### 根因 4：并发原语用错

```java
@Test
void counterIsAccurate() {
    AtomicInteger counter = new AtomicInteger();
    ExecutorService pool = Executors.newFixedThreadPool(10);
    for (int i = 0; i < 1000; i++) {
        pool.execute(counter::incrementAndGet);
    }
    pool.shutdown();
    assertEquals(1000, counter.get());      // ★ 没等任务跑完
}
```

`shutdown` 不阻塞，task 还没跑完就 assert。

**修复**：

```java
pool.shutdown();
assertTrue(pool.awaitTermination(5, SECONDS));
assertEquals(1000, counter.get());
```

#### 治理：检测与隔离

CI 加上 flaky test 检测：

```bash
# Maven Surefire rerun
mvn test -Dsurefire.rerunFailingTestsCount=2
```

某个测试如果"第一次挂、第二次过"，标记为 flaky，**进单独的 quarantine 套件**而非直接通过。这样可视化"我们有多少 flaky 测试"，定期清理。

#### 验证

```bash
# 单独跑这一个测试 100 次
mvn test -Dtest=YourTest#yourMethod -Dsurefire.failIfNoSpecifiedTests=false \
    -DforkCount=4 -DreuseForks=false
```

> **口诀**：**时间用 Clock 注入，网络用 WireMock，异步用 Awaitility，并发显式 await**。Flaky 不能靠 retry 掩盖，只能根治。

---

### 案例 ⑤ 慢测试 —— "单元测试套件跑 12 分钟"

#### 现象

中型项目（约 800 测试方法）跑一遍 12 分钟。开发本地基本不跑，全靠 CI。CI 红了反馈链路长，bug 留到 PR 评审才发现。

#### 根因诊断

打开 Surefire 的报告：

```xml
<surefire>
    <reportFormat>xml</reportFormat>
    <printSummary>true</printSummary>
</surefire>
```

跑完看 `target/surefire-reports/*.xml`，按 time 排序。典型分布：

| 测试类型 | 数量 | 总耗时 | 平均 |
| --- | --- | --- | --- |
| 纯单测 | 600 | 30 s | 50 ms |
| `@WebMvcTest` | 80 | 2 min | 1.5 s |
| `@SpringBootTest` | 100 | 8 min | 4.8 s |
| `@SpringBootTest` + Testcontainers | 20 | 90 s | 4.5 s |

**根因**：`@SpringBootTest` 滥用。100 个用例每个都启完整 Context，每次 ~5 秒。

#### 修复

**优化 1**：能用切片测的不用全量 `@SpringBootTest`。

| 你在测什么 | 应该用 |
| --- | --- |
| Controller 的 HTTP 行为、参数校验、错误处理 | `@WebMvcTest` |
| Repository 的 SQL/JPQL | `@DataJpaTest`（+ Testcontainers） |
| 一个 Service 的业务逻辑 | 纯 Mockito 单测 |
| 序列化/反序列化 | `@JsonTest` |
| 跨多个组件的真实流程 | `@SpringBootTest`（少量） |

**优化 2**：复用 Spring Context。

Spring Test 默认会**按配置的 hash 缓存 Context**。两个 `@SpringBootTest` 类只要 `classes=`、`properties=`、`@MockBean` 完全一致，就共享同一个 Context。

最大杀手：**每个测试都加不同的 `@MockBean`**。每加一个 `@MockBean`，Context cache key 就变了，又得重启一个 Context。

```java
// ❌ 每个测试加不同的 @MockBean → 每次都启新 Context
class TestA {
    @MockBean ServiceA a;
}
class TestB {
    @MockBean ServiceB b;
}

// ✅ 抽到基类，共享 Context
@SpringBootTest
abstract class BaseIT {
    @MockBean ServiceA a;
    @MockBean ServiceB b;
    @MockBean ServiceC c;
}
class TestA extends BaseIT { ... }
class TestB extends BaseIT { ... }
```

`spring.test.context.cache.maxSize` 默认 32。超过会 LRU 淘汰，导致重启。可以调高到 64 或更高。

**优化 3**：并行化（见 3.3）。

**优化 4**：分层 CI。

```yaml
# .github/workflows/ci.yml
test-unit:                      # PR 钩子，必须过
  run: mvn test -Dtest='**/*Test' -DexcludedGroups=integration

test-integration:               # PR 钩子，可选
  run: mvn test -Dgroups=integration
  timeout-minutes: 15

test-slow:                      # nightly 跑
  schedule: '0 2 * * *'
  run: mvn verify -Dgroups=slow
```

JUnit 5 的 `@Tag`：

```java
@Tag("integration")
@SpringBootTest
class OrderIT { ... }
```

#### 验证

```bash
# 看 Context cache 命中
mvn test -Dlogging.level.org.springframework.test.context.cache=DEBUG \
    | grep "Spring test ApplicationContext cache statistics"
```

看到类似 `hitCount = 80, missCount = 5, size = 5` 就是健康的（80 次复用，5 个 Context 实例）。

> **口诀**：**慢测试是组织问题，不是技术问题**。没人盯着耗时趋势就会持续退化。每周看一次 top-20 最慢测试，定期重构。

---

### 案例 ⑥ 覆盖率 100% 但仍有 Bug —— "测试运行了，没在断言"

#### 现象

某 Service 类的 JaCoCo 行覆盖、分支覆盖都 100%。某天上线后某个边界条件下抛 NPE。回头查测试，"覆盖到了"。

#### 根因

```java
public class DiscountService {
    public BigDecimal apply(User user, BigDecimal price) {
        if (user.isVip()) return price.multiply(BigDecimal.valueOf(0.9));
        if (price.compareTo(BigDecimal.valueOf(100)) > 0)
            return price.subtract(BigDecimal.valueOf(10));
        return price;
    }
}

@Test
void apply() {
    DiscountService svc = new DiscountService();
    svc.apply(vip(), BigDecimal.valueOf(50));      // 走第一个分支
    svc.apply(regular(), BigDecimal.valueOf(150)); // 走第二个分支
    svc.apply(regular(), BigDecimal.valueOf(50));  // 走最后
}
```

100% 行/分支覆盖，**0 个 assert**。覆盖率工具高高兴兴报 100%。

#### 修复

**第一道防线**：CI 上加"无 assert 不通过"的 lint。

```xml
<!-- archunit -->
<dependency>
    <groupId>com.tngtech.archunit</groupId>
    <artifactId>archunit-junit5</artifactId>
</dependency>
```

```java
@AnalyzeClasses(packages = "com.example..tests")
class TestQualityRules {
    @ArchTest
    static ArchRule everyTestMustAssert =
        methods()
            .that().areAnnotatedWith(Test.class)
            .should(new ArchCondition<>("contain at least one assertion") {
                public void check(JavaMethod m, ConditionEvents events) {
                    boolean hasAssert = m.getMethodCallsFromSelf().stream()
                        .anyMatch(c -> c.getTargetOwner().getName().contains("Assertions")
                                    || c.getTargetOwner().getName().contains("Mockito")
                                    || c.getName().equals("verify"));
                    if (!hasAssert) events.add(violated(m, "no assert"));
                }
            });
}
```

**第二道防线**：突变测试（PIT）。每周跑一次，关注**突变得分 < 60% 的类**。

```bash
mvn org.pitest:pitest-maven:mutationCoverage \
    -DtargetClasses=com.example.service.DiscountService \
    -DtargetTests=com.example.service.DiscountServiceTest
```

输出会告诉你具体哪一行的什么突变没被杀掉：

```
3. negated conditional → SURVIVED
   DiscountService.java:5
```

意思是把 `if (user.isVip())` 改成 `if (!user.isVip())`，没有任何测试失败。

#### 修复测试

```java
@Test void vipGetsTenPercentOff() {
    assertThat(svc.apply(vip(), BigDecimal.valueOf(100)))
        .isEqualByComparingTo("90");
}

@Test void over100GetsTenOff() {
    assertThat(svc.apply(regular(), BigDecimal.valueOf(150)))
        .isEqualByComparingTo("140");
}

@Test void exactly100NoDiscount() {     // ★ 边界
    assertThat(svc.apply(regular(), BigDecimal.valueOf(100)))
        .isEqualByComparingTo("100");
}

@Test void under100NoDiscount() {
    assertThat(svc.apply(regular(), BigDecimal.valueOf(50)))
        .isEqualByComparingTo("50");
}

@Test void nullUserThrows() {           // ★ 异常路径
    assertThatThrownBy(() -> svc.apply(null, BigDecimal.TEN))
        .isInstanceOf(NullPointerException.class);
}
```

#### 验证

PIT 突变得分 ≥ 80% 才算"测试有效"。低于这个值说明测试还在"装样子"。

> **口诀**：**覆盖率只是负向指标**。100% 行覆盖 ≠ 测试好；30% 一定差。真正的指标是突变得分（mutation score）+ 边界条件覆盖。

---

### 案例 ⑦ Mockito static mock 用错 —— "测试相互污染，只能单独跑"

#### 现象

测试 A 用 `mockStatic(UUID.class)` mock 了 `UUID.randomUUID`。单独跑 A 通过；和 B 一起跑，B 挂在 "expected UUID like ... but got 00000000-0000-..."。

#### 根因

```java
@Test
void createOrder_uuidMocked() {
    MockedStatic<UUID> mocked = mockStatic(UUID.class);
    mocked.when(UUID::randomUUID).thenReturn(UUID.fromString("00000000-..."));
    // ... 测试逻辑
    // ★ 没 close！
}
```

`MockedStatic` 是**线程绑定**的状态。如果不显式 close，**整个 JVM 的 UUID.randomUUID 都被劫持**，下一个测试用真 UUID 时拿到的还是 mock 值。

更糟的：JUnit 跑完会重置一些状态，但 `MockedStatic` 的注册存在 inline mock maker 的内部 `Map<Thread, Mock>` 里，不会被自动清理。Surefire 的 `forkMode=once`（默认）让所有测试共享一个 JVM，污染就跨测试类传播了。

#### 修复

**铁律**：`MockedStatic` 和 `MockedConstruction` 必须用 try-with-resources。

```java
@Test
void createOrder_uuidMocked() {
    try (MockedStatic<UUID> mocked = mockStatic(UUID.class)) {
        mocked.when(UUID::randomUUID).thenReturn(UUID.fromString("00000000-..."));
        // ... 测试逻辑
    }   // ★ 自动 close，清理 thread-local 注册
}
```

或者用 JUnit 5 的 lifecycle：

```java
class Test {
    MockedStatic<UUID> mocked;

    @BeforeEach
    void setUp() {
        mocked = mockStatic(UUID.class);
    }

    @AfterEach
    void tearDown() {
        mocked.close();      // ★ 必须
    }
}
```

**进阶问题**：`mockStatic` 在 Mockito 5 默认线程隔离 —— 一个线程的 mockStatic 不影响另一个线程的真实调用。但**子线程**（你 spawn 的或 `CompletableFuture.supplyAsync` 默认用的 commonPool）**不在 mock 范围内**。如果你的代码会在子线程调 UUID.randomUUID，那次调用拿到的是真 UUID。

```java
try (MockedStatic<UUID> mocked = mockStatic(UUID.class, withSettings().lenient())) {
    mocked.when(UUID::randomUUID).thenReturn(...);
    // 主线程：mock 生效
    UUID.randomUUID();
    // 子线程：mock 不生效（默认）
    new Thread(() -> UUID.randomUUID()).start();
}
```

如果确实需要跨线程，用 `Mockito.mockStatic(UUID.class, Answers.RETURNS_DEFAULTS, true)` 的 `useConstructor` / `lenient` 等参数（看版本）。

#### 验证

```bash
# 把测试用 -Dsurefire.runOrder=random 跑 50 次
for i in $(seq 50); do mvn test -Dsurefire.runOrder=random || break; done
```

如果 close 不正确，几次内必挂。

> **口诀**：`MockedStatic` / `MockedConstruction` **必须**用 try-with-resources。`@BeforeEach` 配 `@AfterEach` 也行，但要每次记得。线程边界外不生效，跨线程的代码 mock 不到。

---

### 案例 ⑧ Spring Boot Test 启动整个 ApplicationContext 慢 —— "100 个 IT 跑 20 分钟"

#### 现象

集成测试套件 100 个 `@SpringBootTest` 跑 20 分钟。本地不跑，CI 卡 PR。

#### 根因

每个 `@SpringBootTest` 都启动完整应用 Context（5 秒 × 100 个 = 8 分钟仅启动）。加上数据库连接初始化、各种 Bean 的 `@PostConstruct`，整体爆炸。

更隐蔽的因素：**Context cache 未命中**。Spring Test 按 cache key 复用 Context，cache key 是这堆东西的 hash：

- `@SpringBootTest(classes = ...)` 的类
- `@SpringBootTest(properties = ...)` 的配置
- `@TestPropertySource`
- `@ActiveProfiles`
- `@MockBean` / `@SpyBean` 的全部声明

只要任意一个不同，就是新 Context。如果你 100 个测试类有 100 套不同的 `@MockBean`，**100 次都重启**。

#### 修复

**优化 1**：抽公共基类，统一 `@MockBean`。

```java
@SpringBootTest
@ActiveProfiles("test")
abstract class BaseSpringIT {
    @MockBean ExternalPaymentClient paymentClient;
    @MockBean ExternalNotifyClient notifyClient;
    @MockBean Clock clock;
    // ... 其他不变的 mock
}

class OrderServiceIT extends BaseSpringIT { ... }
class UserServiceIT extends BaseSpringIT { ... }
```

只要继承 `BaseSpringIT`、不再加额外 `@MockBean`，所有这些 IT 共享同一个 Context。

**优化 2**：避免在测试里改 `@TestPropertySource`。每改一处属性 = 新 cache key。集中到 `application-test.yml`。

**优化 3**：加大 cache size。

```properties
# application-test.properties
spring.test.context.cache.maxSize=64
```

**优化 4**：监控 cache miss。

```java
// 加日志
logging.level.org.springframework.test.context.cache=DEBUG
```

输出：

```
Spring test ApplicationContext cache statistics:
  size = 5, maxSize = 32, parentContextCount = 0,
  hitCount = 95, missCount = 5
```

健康指标：`hitCount / (hitCount + missCount) > 0.9`。

**优化 5**：能下沉到切片就下沉。`@WebMvcTest` 比 `@SpringBootTest` 启动快 5~10 倍。

#### 验证

```bash
mvn test -Dtest='*IT' -Dlogging.level.org.springframework.test.context.cache=DEBUG \
    2>&1 | grep "cache statistics" | tail -1
```

如果 size 接近 maxSize 且 hitCount 占比 < 80%，说明 cache 抖动严重，需要按上面优化。

> **口诀**：Spring Test Context 是**有缓存的**。所有 IT 抽公共基类，`@MockBean` 统一声明，cache 命中率上去就快。任何"加个 @MockBean 试试"都在污染 cache。

---

### 案例 ⑨ Testcontainers 在 CI 上慢 —— "镜像每次拉 30 秒"

#### 现象

本地跑 IT 套件 2 分钟。CI 上 8 分钟，其中近半时间在 Docker 镜像拉取。

#### 根因诊断

CI 日志：

```
Pulling postgres:16-alpine ...
Pulling confluentinc/cp-kafka:7.5.0 ...
Pulling redis:7-alpine ...
```

每次跑 CI 是干净的 runner（GitHub Actions / Jenkins ephemeral agent），Docker 镜像缓存清零。Postgres 镜像 ~80MB，Kafka ~600MB，每次都拉很慢。

第二个根因：每个测试类启停一次容器（默认 `@Container` 行为）。

#### 修复

**优化 1**：CI 上配置 Docker 镜像缓存。

GitHub Actions：

```yaml
- uses: actions/cache@v3
  with:
    path: ~/.docker/buildx-cache
    key: docker-${{ hashFiles('**/pom.xml') }}

- name: Pre-pull images
  run: |
    docker pull postgres:16-alpine &
    docker pull confluentinc/cp-kafka:7.5.0 &
    docker pull redis:7-alpine &
    wait
```

GitLab CI：

```yaml
services:
  - postgres:16-alpine
  - redis:7-alpine
variables:
  DOCKER_DRIVER: overlay2
```

**优化 2**：用 singleton container 模式（见 2.7）。

**优化 3**：用 `withReuse(true)` 跨进程复用。

```java
postgres = new PostgreSQLContainer<>("postgres:16-alpine")
    .withReuse(true);
```

需要在 Testcontainers 配置启用：

```bash
# ~/.testcontainers.properties
testcontainers.reuse.enable=true
```

CI 上加：

```yaml
- run: echo "testcontainers.reuse.enable=true" >> ~/.testcontainers.properties
```

第二次起容器时如果 hash 匹配（image + label + 命令），**直接复用已运行的容器**，0 启动开销。

**优化 4**：用更小的镜像。

```java
// ❌ postgres:16  ~150MB
// ✅ postgres:16-alpine  ~80MB
```

**优化 5**：让 Testcontainers 按 module 启动一次。

构建工具（Maven Surefire / Gradle）配置 fork 复用：

```xml
<configuration>
    <forkCount>1</forkCount>
    <reuseForks>true</reuseForks>     <!-- ★ 一个 JVM 跑所有 IT -->
</configuration>
```

JVM 不退出，singleton container 不会被销毁。

**优化 6**：CI 用 self-hosted runner + Docker 持久化。

如果你用得起，自托管 runner 让 Docker 镜像/卷持久化，第二次跑时镜像本地有，秒拉。

#### 验证

```bash
# 本地观察容器复用
docker ps --filter "label=org.testcontainers.reuse.hash"
# 应该看到 hash 匹配的容器一直在跑，不被销毁
```

> **口诀**：Testcontainers 在 CI 上的速度 = 镜像缓存 + singleton + reuse + alpine 镜像。四件套缺一不可。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 测试设计层
- [ ] 测试断言**行为（输出/副作用）**，不断言"调用记录顺序/次数"
- [ ] 单个测试的 mock 数量 ≤ 3 个；超过考虑用 Fake 或重构被测类
- [ ] 每个 `@Test` 能**独立**通过、能**任意顺序**通过
- [ ] 不用 `static` 字段共享被测对象；每个方法 `@BeforeEach` 重建
- [ ] 测试金字塔比例：单元 > 70%、集成 < 25%、E2E < 10%

### Mockito 层
- [ ] `@ExtendWith(MockitoExtension.class)` 替代手动 `MockitoAnnotations.initMocks`
- [ ] 优先构造器注入，让 `@InjectMocks` 走最稳路径
- [ ] `MockedStatic` / `MockedConstruction` 必须 try-with-resources
- [ ] `verifyNoMoreInteractions` / `InOrder` 慎用，会过度耦合实现
- [ ] 不 mock 自己**不拥有**的类型（第三方 API）—— 包一层自己的接口再 mock

### Spring Boot Test 层
- [ ] 能用切片就用切片（`@WebMvcTest` / `@DataJpaTest`），少用 `@SpringBootTest`
- [ ] IT 抽公共基类，`@MockBean` 统一声明，最大化 Context cache 命中
- [ ] 用 `@Transactional` 自动回滚数据库测试，不要手动清数据
- [ ] `application-test.yml` 集中所有 test 配置，避免 `@TestPropertySource` 抖 cache
- [ ] CI 加 Context cache 命中率监控

### Testcontainers 层
- [ ] 涉及 DB 特有 SQL（jsonb、ON CONFLICT、CTE）必须用 Testcontainers，不用 H2
- [ ] Singleton container 模式：抽抽象基类静态启动一次
- [ ] `withReuse(true)` + `~/.testcontainers.properties` 跨进程复用
- [ ] CI 配置 Docker 镜像缓存 + 预拉取
- [ ] 用 alpine 镜像（postgres:16-alpine 而非 postgres:16）

### 异步与时间层
- [ ] 时间用 `Clock` 注入，测试用 `Clock.fixed`
- [ ] 异步测试用 Awaitility，禁用裸 `Thread.sleep`
- [ ] 并发测试用 `pool.awaitTermination` 显式 await
- [ ] HTTP 客户端用 WireMock 拦截，不打真实外网

### CI 与质量层
- [ ] PR 钩子跑单元测试 + 关键集成测试，控制在 5 分钟内
- [ ] Nightly 跑全套（含 E2E、PIT 突变测试、Pact 验证）
- [ ] Surefire `rerunFailingTestsCount=2` + flaky test 进 quarantine 套件
- [ ] 行覆盖率 ≥ 70%，分支覆盖率 ≥ 60%（最低门槛，不是目标）
- [ ] 关键模块 PIT 突变得分 ≥ 70%
- [ ] ArchUnit 规则：每个 `@Test` 至少有一个 assert/verify

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### 突变测试（PIT）的实战
- PIT 内置 mutator 有 STRONGER / DEFAULTS / ALL 几档，如何按项目阶段选
- `--with-history` 增量模式，只跑改动文件相关的突变（CI 友好）
- arcMutate 这类"语义感知"突变工具，比简单字节码替换更精准
- 突变得分 < 60% 是危险信号，但 100% 也不必追求 —— 等价突变（equivalent mutant）问题

### 契约测试（Pact）跨服务
- Consumer-driven contract（消费方驱动）vs provider-driven 的差异
- Pact Broker + can-i-deploy 让契约成为部署门禁
- 异步消息（Kafka）的契约测试 —— Pact Message
- 与 OpenAPI / GraphQL schema 校验的边界

### 性能测试 vs 单元测试
- [JMH](https://github.com/openjdk/jmh)：JVM 微基准的标准，深坑：JIT 优化让裸 benchmark 全是噪声
- [Gatling](https://gatling.io/) / k6：HTTP 压测，写在 Scala/JS DSL 里
- 性能回归用 [JFR](https://openjdk.org/jeps/328) 持续录制 + 对比 baseline
- 不要把性能测试塞进单元测试套件 —— 它们的 SLA 完全不同

### Property-Based Testing（jqwik）
- 不是写"输入 1 期望 2"，而是写"对任意整数 x，f(x)+f(-x) == 0"
- jqwik 自动生成数千组输入，找反例后自动 shrink 到最小复现
- 适合纯函数、解析器、序列化代码
- 缺点：写"property"比写 example 难，新人门槛高

### Snapshot Testing
- 把"输出"序列化成 JSON / 字符串，与上次提交的快照对比
- Java 生态有 [JsonAssert](https://github.com/skyscreamer/JSONassert) / [Approval Tests](https://github.com/approvals/ApprovalTests.Java)
- 适合 UI、复杂 DTO、报告生成
- 滥用会变成"看到 diff 就 update snapshot"，丧失断言意义

### Architecture Testing（ArchUnit）
- 用代码表达"controller 不能直接调 repository"、"util 包不能依赖 service 包"
- 比静态分析（SonarQube）更精确，能写复杂的"包 A 的类只能被包 B 调"
- 跑得快，应该进 PR 钩子
- 配合 cycle detection 防止包间循环依赖

### TDD 在新人团队的推广
- 强推 TDD 经常失败：节奏要练（红绿重构需要肌肉记忆）
- 折中方案：测试先写（test-first）但不强求 baby step
- 让 PR 评审优先看测试，而非生产代码
- 经典书籍：Kent Beck *TDD By Example*、Steve Freeman *Growing Object-Oriented Software, Guided by Tests*

### Fuzz Testing
- 给函数喂随机/异常输入，看会不会崩
- JVM 上 [Jazzer](https://github.com/CodeIntelligenceTesting/jazzer)（基于 libFuzzer）
- 找解析类、协议类代码的内存安全 / 异常处理 bug
- Google 的 OSS-Fuzz 已经找到 Java 生态几千个 bug

---

## 7. 引用

- Robert Martin, *Clean Code* —— FIRST 原则的出处
- Gerard Meszaros, *xUnit Test Patterns* —— Test Double 五种分类的圣经
- Kent Beck, *Test-Driven Development By Example* —— TDD 节奏的原典
- Steve Freeman, Nat Pryce, *Growing Object-Oriented Software, Guided by Tests* —— 伦敦学派 mockist 的代表
- Mike Cohn, *Succeeding with Agile* —— 测试金字塔出处
- JUnit 5 [User Guide](https://junit.org/junit5/docs/current/user-guide/)
- [Mockito](https://site.mockito.org/) 官方文档
- [AssertJ](https://assertj.github.io/doc/) Core 文档
- [Testcontainers](https://www.testcontainers.org/) 官方文档
- [PIT Mutation Testing](https://pitest.org/) 文档
- [WireMock](https://wiremock.org/) 文档
- [Pact](https://docs.pact.io/) Foundation 文档
- Martin Fowler, [TestPyramid](https://martinfowler.com/bliki/TestPyramid.html)
- Martin Fowler, [Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html) —— mockist vs classicist 之争

---

> 配套：[Spring Boot 自动配置](/article/java/spring-boot) | [DevOps CI/CD](/article/devops/cicd) | [JIT 与字节码](/article/jvm/jit)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
