# Spring 核心原理（速记）

> Spring 这个生态太大，本文只挑**面试必问**和**线上必坑**的部分讲：IoC、AOP、Bean 生命周期、循环依赖、事务、Spring Boot 自动装配。
>
> 五个深度方向已经有专文：
> - **[Spring Bean 生命周期与循环依赖（深文）](/article/java/spring-bean)** — 12 步流程 / 三级缓存 / 9 个事故
> - **[Spring AOP 内幕（深文）](/article/java/spring-aop)** — JDK 代理 / CGLib / 自调用陷阱 / 9 个事故
> - **[Spring 事务传播深入（深文）](/article/java/spring-tx)** — 7 种传播 / 物理连接绑定 / 9 个事故
> - **[Spring Boot 自动配置（深文）](/article/java/spring-boot)** — refresh 12 步 / 9 个事故
> - **[Spring Security 6 深入（深文）](/article/security/spring-security)** — 过滤器链 / OAuth2 / 9 个事故

---

## 1. IoC 容器

### 1.1 三大概念
- **IoC**（Inversion of Control，控制反转）：不再 new，由容器创建；
- **DI**（Dependency Injection，依赖注入）：构造器注入、setter 注入、字段注入；
- **BeanFactory vs ApplicationContext**：后者继承前者，多了国际化、事件、AOP 等。

### 1.2 三种注入方式

| 方式 | 写法 | 推荐度 |
| --- | --- | --- |
| 构造器注入 | 构造方法参数 + `@Autowired`（Spring 4.3+ 单构造器可省） | ⭐⭐⭐⭐⭐ |
| Setter 注入 | setter + `@Autowired` | ⭐⭐⭐ |
| 字段注入 | `@Autowired private Foo foo;` | ⭐ |

> **生产强烈推荐构造器注入**：可声明 final、便于单测、易发现循环依赖。

```java
@Service
public class UserService {
    private final UserMapper mapper;
    public UserService(UserMapper mapper) { this.mapper = mapper; }
}
```

---

## 2. Bean 生命周期（必背）

```
1. 实例化（反射 newInstance）
2. 属性填充（依赖注入）
3. Aware 系列回调（BeanNameAware / BeanFactoryAware / ApplicationContextAware）
4. BeanPostProcessor#postProcessBeforeInitialization
5. @PostConstruct / InitializingBean#afterPropertiesSet / init-method
6. BeanPostProcessor#postProcessAfterInitialization   ← AOP 代理在这里生成！
7. 使用
8. @PreDestroy / DisposableBean#destroy / destroy-method
```

> AOP 代理是在 **第 6 步** 由 `AbstractAutoProxyCreator` 注入的 BeanPostProcessor 生成的。

---

## 3. 循环依赖（高频）

### 3.1 三级缓存
```java
DefaultSingletonBeanRegistry：
  singletonObjects        // 一级：完整 Bean
  earlySingletonObjects   // 二级：半成品（已实例化、未填充属性）
  singletonFactories      // 三级：ObjectFactory，按需返回（解决 AOP 循环依赖）
```

### 3.2 解决思路
1. A 实例化（属性还未填充） → 把 A 的 ObjectFactory 放入三级缓存；
2. A 填充属性时发现需要 B → 去创建 B；
3. B 实例化 → 填充属性时发现需要 A → 从三级缓存拿到 A 的 ObjectFactory，调用获得**早期 A 引用**（必要时生成代理对象）→ 放入二级缓存；
4. B 创建完成；
5. A 继续填充属性，完成创建。

### 3.3 哪些场景仍会失败？
- **构造器循环依赖**：第 1 步就需要对方 → 直接报错；
- **`@Async` 等需要 BPP 后才生成代理**：在某些版本会拿到未代理的早期对象（已被 Spring 6 修复）；
- **prototype Bean**：每次都新建，循环互相依赖必死。

> **解法**：解开循环。代码层面把共用逻辑抽到 C，A、B 都依赖 C。

---

## 4. AOP

### 4.1 实现方式
- **JDK 动态代理**：目标类有接口；
- **CGLIB**：目标类没有接口，生成子类。Spring Boot 2.x+ 默认 `proxyTargetClass=true`，统一用 CGLIB。

### 4.2 通知类型
| 注解 | 时机 |
| --- | --- |
| `@Before` | 方法执行前 |
| `@After` | 方法执行后（finally） |
| `@AfterReturning` | 正常返回后 |
| `@AfterThrowing` | 抛异常后 |
| `@Around` | 包围（最强大） |

### 4.3 AOP 同类调用失效（巨坑）
```java
@Service
class A {
    public void foo() { this.bar(); }   // ❌ this 不是代理对象，AOP 失效
    @Transactional public void bar() {}
}
```

**解法**：
- 注入自己：`@Autowired A self; self.bar();`；
- `((A) AopContext.currentProxy()).bar()` + `@EnableAspectJAutoProxy(exposeProxy=true)`；
- 拆到另一个 Bean。

---

## 5. 事务（@Transactional）

### 5.1 7 种传播行为

| 传播 | 说明 |
| --- | --- |
| **REQUIRED**（默认） | 当前有事务就加入，没有就新建 |
| REQUIRES_NEW | 总是新建一个，挂起当前 |
| SUPPORTS | 有就加入，没有就非事务执行 |
| NOT_SUPPORTED | 总是非事务执行，挂起当前 |
| MANDATORY | 必须有事务，否则抛异常 |
| NEVER | 必须没有事务，否则抛异常 |
| NESTED | 嵌套，使用 SAVEPOINT |

### 5.2 隔离级别
对应数据库四级隔离 + DEFAULT：READ_UNCOMMITTED、READ_COMMITTED、REPEATABLE_READ（MySQL 默认）、SERIALIZABLE。

### 5.3 失效的 8 种情况（必背）
1. **方法非 public**（默认 AOP 只代理 public）；
2. **同类调用**（this 调用绕过代理）；
3. **异常被 catch 没抛**；
4. **抛 checked 异常但默认只回滚 RuntimeException** → 设 `rollbackFor=Exception.class`；
5. **类没被 Spring 管理**（new 出来的）；
6. **数据库引擎不支持事务**（MyISAM）；
7. **传播行为是 NOT_SUPPORTED / NEVER**；
8. **多线程**：事务绑定线程，子线程拿不到。

```java
// 推荐写法
@Transactional(rollbackFor = Exception.class, timeout = 10)
public void biz() { ... }
```

### 5.4 为什么大事务危险
- 长时间持锁 → 阻塞其他事务；
- undo log / binlog 撑爆；
- 失败回滚耗时巨大。

> **铁律**：事务尽量小。RPC 调用、外部 IO、长循环 **不要放事务里**。

---

## 6. Spring Boot 自动装配

### 6.1 核心入口
```java
@SpringBootApplication
  = @SpringBootConfiguration
  + @EnableAutoConfiguration   ← 自动装配的核心
  + @ComponentScan
```

### 6.2 装配流程
1. `@EnableAutoConfiguration` 通过 `AutoConfigurationImportSelector` 扫描；
2. 读取所有 jar 的 **`META-INF/spring.factories`**（Spring Boot 2.x）或 **`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`**（Spring Boot 2.7+）；
3. 通过 `@Conditional*` 系列注解（`@ConditionalOnClass`、`@ConditionalOnMissingBean`、`@ConditionalOnProperty`）按需装配；
4. 配置类通过 `@ConfigurationProperties` 把 application.yml 映射到 Bean。

### 6.3 自定义 Starter
```
your-spring-boot-starter/
├── XxxAutoConfiguration.java
├── XxxProperties.java
└── META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
    （内容：com.xx.XxxAutoConfiguration）
```

---

## 8. 常见线上坑速查

| 坑 | 现象 | 解法 |
| --- | --- | --- |
| 同类调用 AOP 失效 | `@Async`、`@Transactional` 不生效 | 注入自己或 `AopContext` |
| `@Async` 默认线程池被打爆 | 线程数无限增长 | 自定义 `TaskExecutor` |
| `@Transactional` 没回滚 | checked 异常 | 加 `rollbackFor=Exception.class` |
| `@Value` 注入为 null | 静态字段、early Bean | 用 setter + `@Value`，避开 static |
| Bean 循环依赖（构造器） | 启动报错 | 用 setter 或抽出公共逻辑 |
| 注入 `List<MyBean>` 顺序异常 | `@Order` 不生效 | 实现 `Ordered` 接口或 `@Order(int)` |
| Spring Boot 启动时间长 | classpath 扫描慢 | 限制 `@ComponentScan` 范围、用懒加载 |

---

## 9. 高频面试题速记

1. **Spring 三级缓存解决了什么？** AOP 代理 + 循环依赖。
2. **构造器循环依赖能解决吗？** 不能（实例化阶段就需要对方）。
3. **`@Autowired` 和 `@Resource` 区别？** 前者按类型，后者按名称。
4. **Spring Bean 是单例吗？** 默认 singleton，可改 prototype/request/session。
5. **AOP 在哪一步生成代理？** BeanPostProcessor 的 postProcessAfterInitialization。
6. **事务失效的场景？** 见 5.3，面试常考 6 个以上。
7. **Spring Boot 自动装配？** SPI + 条件注解。
8. **JDK 代理 vs CGLIB？** 接口 vs 子类；JDK 反射、CGLIB 字节码生成。
9. **`@SpringBootApplication` 等价于什么？** 见 6.1。

---

> 相关：[Java 基础](/article/java/basic) ｜ [并发编程](/article/java/concurrent)
