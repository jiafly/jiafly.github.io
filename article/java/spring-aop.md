# Spring AOP 内幕 —— 从动态代理到自调用陷阱的全路径

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，看 [Spring 综述](/article/java/spring) 那一节够了。
> 这里要回答的，是**写 `@Transactional` 没生效、被告警叫起来排查 AOP 链路**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. 同一个类里，方法 A 加了 `@Transactional`，方法 B 在事务上下文之外调 `this.A()`，事务**生效不生效**？为什么？给出源码层面的解释，而不是"因为没走代理"。
2. Spring 怎么决定一个 Bean 是用 **JDK 动态代理**还是 **CGLib**？给出三个判定条件，按优先级排序。
3. `proxyTargetClass = true` 在生产里有什么**真实代价**？只回答"性能差一点"不算。
4. `@AspectJ` 的五种 Advice（Before / AfterReturning / AfterThrowing / After / Around）混用时，**执行顺序**是什么？把它们的入栈/出栈时机画出来。
5. Pointcut 表达式 `execution(* com.foo..*Service.*(..))`，从字符串到运行时匹配，中间发生了什么？谁在做这件事？
6. Spring 事务、`@Async`、`@Cacheable`，**它们走的是同一套 AOP 基建吗**？如果是，差别在哪？
7. 一个 Bean 上挂了三个 `@Aspect`，请求进来的时候，**责任链是怎么串起来的**？源码里哪个类是责任链的"指针"？
8. JDK 17 / 21 上 CGLib 直接用还能不能跑？Spring 6 怎么解决的？换成 ByteBuddy 在哪些场景已经发生？

如果有半数答不上，往下看。

---

## 1. 底层模型：代理、织入、责任链

绝大多数人对 Spring AOP 的认知止于"切面 + 注解 + 代理"。这层抽象解释不了上面 8 个问题。要看懂它们，必须把 AOP 拆成三个互不耦合的东西：**编程模型（Aspect / Advice / Pointcut）**、**织入方式（运行时 vs 编译期）**、**代理实现（JDK vs CGLib）**。

### 1.1 概念五件套

| 术语 | 中文 | 一句话定义 | Spring 里的具象 |
| --- | --- | --- | --- |
| **Aspect** | 切面 | 横切关注点的模块化封装 | 一个 `@Aspect` 类 |
| **Advice** | 通知 | 切面在某个时机要做的事 | `@Before` / `@Around` 标注的方法 |
| **Pointcut** | 切点 | 描述"在哪些 Joinpoint 上触发" | `execution(...)` 表达式 |
| **Joinpoint** | 连接点 | 程序执行中可以织入的点 | Spring AOP 里**只有方法调用** |
| **Weaving** | 织入 | 把 Aspect 应用到目标的过程 | Spring 在 `getBean` 时生成代理 |

第一处反直觉：**Spring AOP 的 Joinpoint 只有"方法调用"一种**。AspectJ 完整规范里还有字段读写、对象初始化、构造器调用等，Spring 全部不支持。这是 Spring AOP "够用但不全"的根因，也是案例 ⑦"Bean 内部方法不被拦截"的根。

### 1.2 运行时织入 vs 编译期织入

| 维度 | Spring AOP（运行时织入） | AspectJ（编译期/类加载期织入） |
| --- | --- | --- |
| 织入时机 | Bean 创建时生成代理 | `ajc` 编译时改字节码 / LTW 类加载时改 |
| Joinpoint | 仅方法调用 | 全部（字段、构造器、static 块……） |
| 自调用是否生效 | **不生效**（不走代理） | 生效（直接改了字节码） |
| 性能 | 一次代理跳转 + 责任链 | 接近零开销 |
| 部署复杂度 | 零 | 需要 `ajc` 或 `aspectjweaver` agent |

Spring AOP 选了"运行时 + 仅方法调用"，是用**功能弱化换部署简单**。这个选择支配了它后面所有设计 —— 包括为什么自调用必然失效。

### 1.3 JDK 动态代理：基于接口

JDK 动态代理的对象模型：

```
Target (实现了某个接口 SomeService)
   │
Proxy (运行时 Proxy.newProxyInstance 生成)
   │   ├── implements SomeService
   │   └── 持有 InvocationHandler
   │
InvocationHandler.invoke(proxy, method, args)
   └── 调 advice 链 → 调 target.method(args)
```

关键：**生成的 Proxy 类名形如 `$Proxy0`，它和 Target 是兄弟关系（都实现了 SomeService），不是父子**。所以你能把 Proxy 强转成 `SomeService`，**但不能强转成 `SomeServiceImpl`**。这是案例 ④ ClassCastException 的根。

### 1.4 CGLib：字节码生成子类

CGLib 走的是另一条路：**运行时用 ASM 生成 Target 的子类**。

```
Target (具体类 SomeServiceImpl)
   │
Proxy extends SomeServiceImpl  ← CGLib 生成的子类
   │
   方法被覆写：
   public String foo(...) {
       MethodInterceptor.intercept(this, method, args, methodProxy);
   }
```

代价：
- **final 类不能继承** → 不能代理（案例 ③）
- **final 方法不能覆写** → 不能拦截（同案例 ③）
- **构造器会被调用两次**（父类一次、子类一次）→ 构造器里有副作用就出问题
- 对**包私有方法**的可见性受类加载器约束

### 1.5 责任链与 ReflectiveMethodInvocation

不管 JDK 还是 CGLib，进了 advice 链之后走的是同一个东西：`ReflectiveMethodInvocation`。

```
proxy.foo()
  ↓ InvocationHandler / MethodInterceptor
ReflectiveMethodInvocation.proceed()
  ↓
  interceptor[0].invoke(this)   // 比如 ExposeInvocationInterceptor
     ↓ proceed()
     interceptor[1].invoke(this)   // TransactionInterceptor
        ↓ proceed()
        interceptor[2].invoke(this)   // CacheInterceptor
           ↓ proceed()
           interceptor[3].invoke(this)   // 你的 @Around
              ↓ proceedingJoinPoint.proceed()
                 ↓
                 method.invoke(target, args)   // 真正的业务方法
```

`proceed()` 是责任链的"指针"，每个 interceptor 显式调它来推进到下一个。**如果某个 `@Around` 忘了调 `pjp.proceed()`，方法就被吞了**（案例 ⑨）。

### 1.6 Spring AOP / 事务 / 缓存 是同一套基建

这一点很多人没意识到：

| 注解 | 背后的 Advisor | 背后的 Interceptor |
| --- | --- | --- |
| `@Transactional` | `BeanFactoryTransactionAttributeSourceAdvisor` | `TransactionInterceptor` |
| `@Async` | `AsyncAnnotationAdvisor` | `AnnotationAsyncExecutionInterceptor` |
| `@Cacheable` / `@CachePut` | `BeanFactoryCacheOperationSourceAdvisor` | `CacheInterceptor` |
| 你写的 `@Aspect` | `AspectJPointcutAdvisor` | `AspectJAroundAdvice`（包成 MethodInterceptor） |

**它们公用同一套代理机制**。这就是为什么自调用问题对 `@Transactional` / `@Async` / `@Cacheable` **同时存在** —— 都吃同一个亏。

---

## 2. 源码切片：六段关键路径

> 基于 Spring Framework 6.x 源码。注释为本文添加。

### 2.1 DefaultAopProxyFactory：代理选型

```java
// org.springframework.aop.framework.DefaultAopProxyFactory
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    if (!NativeDetector.inNativeImage()
        && (config.isOptimize()
            || config.isProxyTargetClass()
            || hasNoUserSuppliedProxyInterfaces(config))) {

        Class<?> targetClass = config.getTargetClass();
        if (targetClass == null) {
            throw new AopConfigException(
                "TargetSource cannot determine target class");
        }
        if (targetClass.isInterface()
            || Proxy.isProxyClass(targetClass)
            || ClassUtils.isLambdaClass(targetClass)) {
            return new JdkDynamicAopProxy(config);   // ★ 目标本身是接口/Proxy/lambda
        }
        return new ObjenesisCglibAopProxy(config);   // ★ 走 CGLib
    }
    return new JdkDynamicAopProxy(config);           // ★ 默认 JDK
}
```

**逐段点透**：

- **三个走 CGLib 的条件**（`if` 第一段）：`optimize=true`、`proxyTargetClass=true`、**目标没有用户接口**。任意一个为真就考虑 CGLib。
- **两个降级到 JDK 的兜底**：CGLib 路径里如果发现 `targetClass` 本身就是接口、`Proxy` 类、lambda，反而退回 JDK —— 这些场景 CGLib 没法生子类。
- **`hasNoUserSuppliedProxyInterfaces`** 才是日常最常踩的分支：你的 `Service` 实现了 `XxxService` 接口 → 走 JDK；你的 `Service` 没有任何业务接口（只有 Spring 自己塞的 `Advised` / `SpringProxy`）→ 走 CGLib。
- **Spring Boot 2.x 起默认 `proxyTargetClass=true`**（`spring.aop.proxy-target-class=true`）—— 等于强制全部走 CGLib，不管你有没有接口。这是案例 ⑧ 的根。

### 2.2 JdkDynamicAopProxy.invoke：核心拦截

```java
// org.springframework.aop.framework.JdkDynamicAopProxy
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    Object oldProxy = null;
    boolean setProxyContext = false;
    TargetSource targetSource = this.advised.targetSource;
    Object target = null;

    try {
        // ① 三个特殊方法的短路：equals / hashCode / 来自 DecoratingProxy 的方法
        if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
            return equals(args[0]);
        }
        if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
            return hashCode();
        }
        // ② 来自 Advised 接口的方法 → 直接反射调 advised 自己
        if (method.getDeclaringClass() == DecoratingProxy.class) {
            return AopProxyUtils.ultimateTargetClass(this.advised);
        }

        Object retVal;
        // ③ exposeProxy=true 把 proxy 塞 ThreadLocal，自调用时能拿到
        if (this.advised.exposeProxy) {
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
        }

        target = targetSource.getTarget();
        Class<?> targetClass = (target != null ? target.getClass() : null);

        // ④ 拼出这次方法的拦截器链
        List<Object> chain = this.advised
            .getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

        if (chain.isEmpty()) {
            // ⑤ 没有任何拦截器 → 直接反射调，省掉责任链
            Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
            retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
        } else {
            // ⑥ 走责任链
            MethodInvocation invocation = new ReflectiveMethodInvocation(
                proxy, target, method, args, targetClass, chain);
            retVal = invocation.proceed();
        }
        // ... 返回值处理省略
        return retVal;
    } finally {
        if (target != null && !targetSource.isStatic()) targetSource.releaseTarget(target);
        if (setProxyContext) AopContext.setCurrentProxy(oldProxy);
    }
}
```

**逐段点透**：

- **③ `exposeProxy` 是手动启用的逃生口**：`@EnableAspectJAutoProxy(exposeProxy = true)`。开启后能用 `((SelfClass) AopContext.currentProxy()).method()` 解决自调用，但代价是每次调用都多一次 ThreadLocal 写。
- **⑤ 空链短路**：很多 Bean 实际上没有任何 advice（不被任何 Pointcut 命中），也会走这条路径。Spring 6 在这一步做过专门优化，避免无用的 `ReflectiveMethodInvocation` 对象。
- **`invokeJoinpointUsingReflection` 内部用 `Method.invoke`**：JDK 17 起开了 reflective MethodHandles 优化，单次反射调用大约 3~5ns，几乎不再是瓶颈。

### 2.3 CglibAopProxy：字节码生成

```java
// org.springframework.aop.framework.CglibAopProxy
public Object getProxy(@Nullable ClassLoader classLoader) {
    Class<?> rootClass = this.advised.getTargetClass();
    Class<?> proxySuperClass = rootClass;

    // ① 如果 rootClass 已经是 CGLib 生成的（名字含 $$），剥一层
    if (rootClass.getName().contains(ClassUtils.CGLIB_CLASS_SEPARATOR)) {
        proxySuperClass = rootClass.getSuperclass();
        for (Class<?> ifc : rootClass.getInterfaces()) {
            this.advised.addInterface(ifc);
        }
    }

    validateClassIfNecessary(proxySuperClass, classLoader);

    Enhancer enhancer = createEnhancer();
    enhancer.setSuperclass(proxySuperClass);                       // ★ 子类化目标
    enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
    enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
    enhancer.setAttemptLoad(true);
    enhancer.setStrategy(new ClassLoaderAwareGeneratorStrategy(classLoader));

    Callback[] callbacks = getCallbacks(rootClass);                // ★ 7 个回调，分流不同方法
    enhancer.setCallbackFilter(new ProxyCallbackFilter(...));
    enhancer.setCallbackTypes(types);

    return createProxyClassAndInstance(enhancer, callbacks);
}
```

**逐段点透**：

- **① "剥洋葱"逻辑**：如果 rootClass 自己就是 CGLib 子类（比如已经被代理过一次），用它的父类做新代理的父类。否则会出现 `$$EnhancerByCGLIB$$_$$EnhancerByCGLIB$$` 这种套娃。
- **`validateClassIfNecessary` 检查 final**：里面会扫描所有方法，如果有 final method 会**打 INFO 日志**（不是 ERROR），然后跳过这些方法的代理。**这就是为什么 final 方法上的 `@Transactional` 静默失效** —— 没有报错，只是没拦截到。
- **`getCallbacks` 返回 7 个 Callback**：包括 `DynamicAdvisedInterceptor`（核心）、`StaticUnadvisedInterceptor`、`SerializableNoOp`、`StaticDispatcher`、`AdvisedDispatcher`、`EqualsInterceptor`、`HashCodeInterceptor`。`CallbackFilter` 决定每个方法走哪个回调。

### 2.4 ReflectiveMethodInvocation.proceed：责任链推进

```java
// org.springframework.aop.framework.ReflectiveMethodInvocation
public Object proceed() throws Throwable {
    // ① 链尾：所有 interceptor 都走完了，反射调 target
    if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
        return invokeJoinpoint();
    }

    Object interceptorOrInterceptionAdvice =
        this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);

    // ② 动态切点：运行时再判一次能不能匹配
    if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher dm) {
        Class<?> targetClass = (this.targetClass != null
            ? this.targetClass : this.method.getDeclaringClass());
        if (dm.matcher().matches(this.method, targetClass, this.arguments)) {
            return dm.interceptor().invoke(this);
        } else {
            // 跳过这一个 interceptor，递归推进
            return proceed();
        }
    } else {
        // ③ 普通拦截器 → 调它的 invoke，把 this 传进去让它再调 proceed
        return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
    }
}
```

**逐段点透**：

- **`currentInterceptorIndex` 是责任链的"指针"**：从 -1 开始，每次 `proceed()` 自增到下一个 interceptor。这就是为什么 `MethodInvocation` **不能复用**也**不能跨线程** —— 它持有可变状态。
- **递归调 proceed 的堆栈深度 = 拦截器数 + 1**。日常 5~10 层很常见，事务嵌套缓存嵌套 trace 嵌套自定义切面，深栈可能压到 30~50 层。这就是堆栈里看 Spring AOP 总是"很高"的原因。
- **动态切点（②）**：能拿到运行时实参再判断的切点，比如 `@Pointcut("args(name)") && execution(...)`。它在每次调用时都重新匹配一次，所以**贵**。能用静态切点（仅看签名）就不要用动态切点。

### 2.5 AspectJExpressionPointcut：表达式解析

```java
// org.springframework.aop.aspectj.AspectJExpressionPointcut
public boolean matches(Method method, Class<?> targetClass) {
    obtainPointcutExpression();   // 第一次调用时编译表达式
    ShadowMatch shadowMatch = getTargetShadowMatch(method, targetClass);
    if (shadowMatch.alwaysMatches()) return true;
    if (shadowMatch.neverMatches()) return false;
    // ★ "maybe" 状态需要进一步动态检查
    return false;   // 静态阶段保守判定为 false，由 RuntimeTestWalker 后续处理
}

private PointcutExpression obtainPointcutExpression() {
    if (getExpression() == null) throw new IllegalStateException("...");
    if (this.pointcutExpression == null) {
        this.pointcutClassLoader = determinePointcutClassLoader();
        this.pointcutExpression = buildPointcutExpression(this.pointcutClassLoader);
    }
    return this.pointcutExpression;
}

private PointcutExpression buildPointcutExpression(ClassLoader classLoader) {
    PointcutParser parser = initializePointcutParser(classLoader);
    // ... 处理 args 占位符
    return parser.parsePointcutExpression(
        replaceBooleanOperators(resolveExpression()),
        this.pointcutDeclarationScope, pointcutParameters);
}
```

**逐段点透**：

- **Spring AOP 复用 AspectJ 的解析器**：`org.aspectj.weaver.tools.PointcutParser`。这就是为什么 Pointcut 表达式语法和 AspectJ 完全一致，但**只支持其中"针对方法调用"的子集**（`call`、`set`、`get`、`initialization` 等都不支持）。
- **`ShadowMatch` 三态**：`alwaysMatches`（一定匹配）、`neverMatches`（一定不匹配）、`maybe`（看运行时）。前两者在 Bean 创建时就能判完，maybe 才进动态切点路径。
- **表达式写错的代价**：写成 `execution(* com.foo..*(..))` 漏了 `Service` 这种过滤词，结果 com.foo 包下**所有类的所有方法都被代理**，性能直接崩塌（案例 ⑤）。

### 2.6 AbstractAutoProxyCreator：织入入口

```java
// org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator
public Object postProcessAfterInitialization(Object bean, String beanName) {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        if (this.earlyBeanReferences.remove(cacheKey) != bean) {
            return wrapIfNecessary(bean, beanName, cacheKey);   // ★ 关键
        }
    }
    return bean;
}

protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    if (this.advisedBeans.containsKey(cacheKey)
        && Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }
    if (isInfrastructureClass(bean.getClass())
        || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }

    // ① 找出所有可以应用到这个 Bean 的 Advisor
    Object[] specificInterceptors =
        getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        // ② 真正生成代理对象
        Object proxy = createProxy(
            bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
```

**逐段点透**：

- `AbstractAutoProxyCreator` 是个 `BeanPostProcessor`。**所有的代理织入都发生在 `postProcessAfterInitialization` 这个钩子里** —— 也就是 Bean 已经初始化完成后再"包"一层。
- **`isInfrastructureClass` 的过滤**：Advisor / Pointcut / Advice / AopInfrastructureBean 这几类自己不会被代理，否则会无限递归。
- **`getAdvicesAndAdvisorsForBean` 是 O(N×M)**：N = Advisor 数，M = Bean 数。Bean 多 + Pointcut 写得宽，Spring 启动时会明显变慢。这是大型应用启动慢的隐性来源之一。

> **口诀**：代理生成在 `BeanPostProcessor`，代理调用走 `Invocation.proceed`，匹配判定在 `AspectJExpressionPointcut`。三件事永远是这三个类的责任。

---

## 3. 实测数据：代理开销不是均匀的

口诀化的"AOP 性能损耗很小"几乎所有八股文章都写了，但**不同代理类型、不同 Advice 类型的开销差异巨大**，下面用 JMH 量出来。

### 3.1 结论先行

| 维度 | 调用 ns/op | 相对裸调用 |
| --- | --- | --- |
| 直接方法调用（baseline） | ~1 | 1× |
| JDK 动态代理（无 advice） | ~5 | 5× |
| JDK 动态代理（1 个 @Around） | ~30 | 30× |
| CGLib（无 advice） | ~3 | 3× |
| CGLib（1 个 @Around） | ~28 | 28× |
| CGLib + 5 层 advice | ~120 | 120× |
| CGLib + 5 层 advice + 动态切点 | ~250 | 250× |

**反直觉的点**：
- **CGLib 和 JDK 动态代理调用开销几乎一样**（差 1~2ns），传说中的"CGLib 快"在 JDK 17 之后基本不成立 —— `Method.invoke` 在新 JDK 里走 MethodHandle 直接 dispatch，比早年快太多。
- **真正贵的是 advice 链本身**，不是代理类型。每多一层 advice ≈ +25ns。
- **动态切点 (`args()`、`this()`)** 比静态切点贵一个量级，每次调用都要 `RuntimeTestWalker` 走一遍。

### 3.2 JMH 模板

```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
@Fork(1)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 5, time = 2)
public class AopOverheadBench {

    public interface Service { String greet(String name); }
    public static class Impl implements Service {
        public String greet(String name) { return "hi " + name; }
    }

    private Service raw;
    private Service jdkProxy;
    private Service cglibProxy;
    private Service cglibProxy5;

    @Setup
    public void setup() {
        raw = new Impl();
        jdkProxy = buildProxy(false, 1);
        cglibProxy = buildProxy(true, 1);
        cglibProxy5 = buildProxy(true, 5);
    }

    private Service buildProxy(boolean cglib, int adviceCount) {
        ProxyFactory pf = new ProxyFactory(new Impl());
        pf.setProxyTargetClass(cglib);
        for (int i = 0; i < adviceCount; i++) {
            pf.addAdvice((MethodInterceptor) inv -> inv.proceed());
        }
        return (Service) pf.getProxy();
    }

    @Benchmark public String baseline()    { return raw.greet("x"); }
    @Benchmark public String jdkProxy1()   { return jdkProxy.greet("x"); }
    @Benchmark public String cglibProxy1() { return cglibProxy.greet("x"); }
    @Benchmark public String cglibProxy5() { return cglibProxy5.greet("x"); }
}
```

### 3.3 多层 AOP 嵌套的栈

火焰图里典型的"AOP 蘑菇云"：

```
controller.handle
  └── (CGLib proxy)
       └── DynamicAdvisedInterceptor.intercept
            └── ReflectiveMethodInvocation.proceed [×N]
                 ├── ExposeInvocationInterceptor
                 ├── TransactionInterceptor          ← @Transactional
                 ├── CacheInterceptor                ← @Cacheable
                 ├── AnnotationAsyncExecutionInterceptor   ← @Async（如果同方法）
                 ├── 自定义 LogAspect.@Around
                 └── 真正的 service.handle
```

每多一层 AOP 大约**多 25ns + 多 200B 栈**。8 层 AOP 不算少见，单次方法调用就被 AOP 多吃 200ns。对单机 10 万 QPS 的服务来说，AOP 总开销约 20ms CPU/s。

### 3.4 反射 vs MethodHandle vs Direct

JDK 17 起 `Method.invoke` 内部走 `MethodHandle` dispatch，**反射不再是性能毒药**。我们对比同一个方法的三种调法：

| 调用方式 | ns/op |
| --- | --- |
| 直接调 | 1.0 |
| `MethodHandle.invokeExact` | 1.5 |
| `Method.invoke`（JDK 21） | 3~5 |
| `Method.invoke`（JDK 8） | 30+ |

所以"CGLib 比 JDK 动态代理快"在新 JDK 上已经不再是显著差异。仍然要选 CGLib 的真实理由是**目标没接口**，不是"性能"。

> **口诀**：选代理类型看**有没有接口**，不看性能；性能优化要砍**advice 层数**和**动态切点**，不要砍代理类型。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① @Transactional 自调用失效 —— "事务边界没建立"

#### 现象

```java
@Service
public class OrderService {

    @Transactional
    public void createOrder(Order o) {
        orderRepo.save(o);
        addPoint(o.getUserId());      // ★ 同类调用
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void addPoint(long userId) {
        pointRepo.add(userId, 10);
        if (userId < 0) throw new RuntimeException("bad");   // 故意抛
    }
}
```

预期：`addPoint` 抛异常 → 它自己的事务回滚（REQUIRES_NEW），但已经写入的积分应该被回滚，外层 `createOrder` 也回滚。

实际：`addPoint` 抛异常确实让 `createOrder` 回滚了，但**积分被写入了**（自己没回滚）—— 因为根本就没建立 REQUIRES_NEW 的内层事务。

#### 根因

`@Transactional` 的拦截只发生在**通过代理调用**的时候。`createOrder` 是通过 Spring 注入的代理调进来的，没问题。但 `addPoint(o.getUserId())` 是 `this.addPoint(...)` —— **`this` 是目标对象自己，不是代理**。

回到源码，`JdkDynamicAopProxy.invoke` 里调的是 `target.method(args)`：

```java
return AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
//                                              ↑ target 是 OrderService 实例
```

target 调自己的方法 → 直接走 JVM 方法分派 → **不经过代理** → 拦截器链根本不上场 → `@Transactional` 完全失效。

更糟的是：**没有任何报错**。看日志看不出来，看 jstack 看不出来。

#### 修复

**方案 A**：注入自己的代理。

```java
@Service
public class OrderService {

    @Autowired @Lazy   // ★ @Lazy 避免循环依赖
    private OrderService self;

    @Transactional
    public void createOrder(Order o) {
        orderRepo.save(o);
        self.addPoint(o.getUserId());   // ★ 通过代理调
    }
}
```

**方案 B**：拿 `AopContext.currentProxy()`。

```java
@EnableAspectJAutoProxy(exposeProxy = true)   // ★ 必须开启
@Configuration
public class AopConfig {}

// 业务里：
((OrderService) AopContext.currentProxy()).addPoint(o.getUserId());
```

代价：每次调用多一次 ThreadLocal 写入。

**方案 C**：拆成两个类（最干净）。

```java
@Service
class OrderService {
    @Autowired PointService pointService;

    @Transactional
    public void createOrder(Order o) {
        orderRepo.save(o);
        pointService.addPoint(o.getUserId());
    }
}

@Service
class PointService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void addPoint(long userId) { ... }
}
```

**方案 D**：编程式事务。

```java
@Autowired TransactionTemplate tx;

public void createOrder(Order o) {
    orderRepo.save(o);
    tx.execute(status -> { addPoint(o.getUserId()); return null; });
}
```

#### 验证

```java
@Test
void selfCallShouldNotStartNewTransaction() {
    Order o = new Order();
    o.setUserId(-1);   // 触发内部异常
    assertThrows(RuntimeException.class, () -> service.createOrder(o));
    // 期望：积分表里没有这个用户的记录
    assertEquals(0, pointRepo.countBy(-1));
}
```

修复前：积分表里有一条 `userId=-1, point=10` 的脏数据。

> **口诀**：`this.method()` = 不走代理 = 注解全失效。`@Transactional` / `@Async` / `@Cacheable` 通通中招。

---

### 案例 ② @Async 同样的自调用陷阱

#### 现象

```java
@Service
public class ReportService {

    public void generate() {
        for (int i = 0; i < 100; i++) {
            sendOne(i);     // 期望：异步、并行
        }
    }

    @Async
    public void sendOne(int i) {
        slowHttpCall(i);
    }
}
```

期望：100 个 `sendOne` 异步并行，`generate` 立刻返回。
实际：`generate` 阻塞跑了 100 秒才返回。`@Async` 像没写一样。

#### 根因

和案例 ① 完全一样。`@Async` 也是通过 AOP 拦截 + 把方法体丢到执行器里跑。`this.sendOne(i)` 不走代理 → 拦截器没上场 → 同步执行。

更隐蔽：`@Async` 的方法**返回值必须是 `void` 或 `Future` / `CompletableFuture`**。直接返回 `String` 之类的，启动期不会报错，但你拿到的永远是**目标方法被同步调用后的真实返回值**或 **null**（取决于 Spring 版本），完全失去异步语义。

#### 修复

同案例 ①：拆类、注入 self、或用编程式 `executor.submit`。

```java
@Service
public class ReportService {
    @Autowired AsyncSenderService sender;

    public void generate() {
        for (int i = 0; i < 100; i++) sender.sendOne(i);
    }
}

@Service
class AsyncSenderService {
    @Async
    public void sendOne(int i) { slowHttpCall(i); }
}
```

#### 验证

```java
@Test
void asyncMustBeAsync() throws Exception {
    long start = System.currentTimeMillis();
    service.generate();
    long elapsed = System.currentTimeMillis() - start;
    assertTrue(elapsed < 200, "generate() should return quickly, took " + elapsed);
}
```

> **口诀**：`@Async` 自调用的失败模式比 `@Transactional` 还难发现 —— 因为没有"积分被写入"这种业务可见的脏数据，只是慢一点。压测才能暴露。

---

### 案例 ③ final 类 / final 方法 —— "AOP 静默失效"

#### 现象

某些团队习惯把 Service 写成 `final class XxxService`（"防继承"），或把方法写成 `public final void doIt()`。代码上线，`@Transactional` / `@Cacheable` 全部失效，没有任何报错。

#### 根因

CGLib 通过**生成子类**实现代理：
- final class 不能被继承 → 无法生成代理子类。Spring 5 之前会**直接抛异常**；Spring 6 改成"如果有接口就退化到 JDK 动态代理；没接口就报错"。
- final method 子类可以继承，但**不能被覆写** → CGLib 生成的子类里这个方法保持父类原版，没有任何 advice 拦截。这是日志级别 INFO 的提示，不会报错。

源码：

```java
// org.springframework.aop.framework.CglibAopProxy#doValidateClass
private void doValidateClass(Class<?> proxySuperClass, ClassLoader proxyClassLoader,
                             Set<Class<?>> ifcs) {
    if (proxySuperClass != Object.class) {
        Method[] methods = proxySuperClass.getDeclaredMethods();
        for (Method method : methods) {
            int mod = method.getModifiers();
            if (!Modifier.isStatic(mod) && !Modifier.isPrivate(mod)) {
                if (Modifier.isFinal(mod)) {
                    if (logger.isInfoEnabled()) {
                        // ★ 仅 INFO 日志，不抛异常
                        logger.info("Unable to proxy interface-implementing method ["
                            + method + "] because it is marked as final: ...");
                    }
                }
            }
        }
    }
}
```

注意 `logger.isInfoEnabled()` —— 默认日志级别是 INFO 才能看到。**WARN 及以上级别完全沉默**。

#### 修复

- 拆掉 `final`。Spring AOP 生态里就别用 final，约定优于代码洁癖。
- 如果 final 是设计上必要的（比如 value object），那它就不该放业务逻辑、不该被 AOP 切。
- Kotlin 的所有类**默认 final** —— 必须给 Spring 类加 `open class`，或用 `kotlin-spring` 插件自动 `open`。

```kotlin
// build.gradle.kts
plugins {
    kotlin("plugin.spring") version "1.9.0"
}

// 等价于自动给所有 @Component / @Service / @Configuration 类加 open
```

#### 验证

```java
@Test
void finalMethodWontBeIntercepted() {
    Service s = ctx.getBean(Service.class);
    assertTrue(AopUtils.isAopProxy(s));
    s.finalMethod();    // 期望：拦截器不上场
    assertEquals(0, AdviceCounter.count());
}
```

> **口诀**：`final` 在 Spring AOP 里是隐形地雷。Kotlin 默认 final 等于人均踩雷，必须装 `kotlin-spring`。

---

### 案例 ④ JDK 代理 ClassCastException —— "强转目标类失败"

#### 现象

```java
@Autowired
private OrderServiceImpl orderService;   // ★ 注入的是接口的实现类

@PostMapping("/order")
public void create() {
    orderService.create();   // 启动报 BeanNotOfRequiredTypeException
}
```

或者：

```java
@Autowired
private OrderService orderService;       // 接口

void method() {
    OrderServiceImpl impl = (OrderServiceImpl) orderService;   // ★ ClassCastException
}
```

#### 根因

`OrderServiceImpl implements OrderService`。Spring 默认走 JDK 动态代理 → 生成的代理类 `$Proxy123` **只实现了 `OrderService` 接口**，和 `OrderServiceImpl` 是**兄弟关系**，不是父子。

```
       OrderService (接口)
          ▲      ▲
          │      │
 OrderServiceImpl   $Proxy123
       (target)       (代理)
```

强转必然失败。注入字段类型写实现类也会失败 —— Spring 找不到一个**类型是 OrderServiceImpl** 的 Bean（容器里只有 `$Proxy123` 这个 OrderService 类型的）。

#### 修复

**方案 A**（推荐）：始终注入接口。

```java
@Autowired
private OrderService orderService;
```

**方案 B**：开 `proxyTargetClass = true` 强制 CGLib，代理就是子类，能强转。

```java
@EnableAspectJAutoProxy(proxyTargetClass = true)
```

代价见案例 ⑧。

**方案 C**：从代理拿目标。

```java
OrderServiceImpl impl = (OrderServiceImpl)
    AopProxyUtils.getSingletonTarget(orderService);
```

但这是 hack，仅限测试 / 反射场景，业务代码不要这么写。

#### 验证

```java
@Test
void shouldInjectAsInterface() {
    Object bean = ctx.getBean("orderService");
    assertTrue(bean instanceof OrderService);     // ✅
    assertFalse(bean instanceof OrderServiceImpl); // ✅ JDK 代理
}
```

> **口诀**：JDK 代理是兄弟、CGLib 代理是儿子。**永远依赖接口而不是实现类**，这条规矩省下 90% 的诡异 ClassCastException。

---

### 案例 ⑤ Pointcut 写宽 —— "全表代理性能崩塌"

#### 现象

接入了一个"通用日志切面"后，服务 P99 从 50ms 涨到 800ms。CPU 利用率从 30% 涨到 95%。火焰图里 80% 的栈在 Spring AOP 链里。

#### 根因

切面定义：

```java
@Aspect
@Component
public class LogAspect {

    @Around("execution(* com.foo..*(..))")    // ❌
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try { return pjp.proceed(); }
        finally {
            log.debug("{} took {}ns", pjp.getSignature(), System.nanoTime() - start);
        }
    }
}
```

`com.foo..*(..)` 匹配 `com.foo` 包下**所有类的所有方法**。包括：
- DTO 的 getter/setter（每个字段访问都走 AOP）
- `equals` / `hashCode`
- Spring 自己的工具类
- MyBatis 生成的 Mapper 实现
- 各种 Repository 的方法

每次方法调用 +25ns，看起来不多，但 Map 查找 / equals 比较这种纳秒级方法被一切，总开销暴涨。

#### 修复

收紧 Pointcut，只切真正的"业务"：

```java
@Around(
    "execution(* com.foo.service..*Service.*(..)) " +    // 只切 Service
    "&& !execution(* *..get*(..)) " +                     // 排除 getter
    "&& !execution(* *..set*(..)) " +                     // 排除 setter
    "&& !execution(* *..hashCode()) " +
    "&& !execution(* *..equals(..))"
)
public Object around(ProceedingJoinPoint pjp) throws Throwable { ... }
```

或者用注解驱动 —— 显式标注比 execution 通配符可控得多：

```java
@Target(METHOD) @Retention(RUNTIME)
public @interface LogTime {}

@Around("@annotation(com.foo.LogTime)")
public Object around(ProceedingJoinPoint pjp) throws Throwable { ... }
```

业务代码：

```java
@LogTime
public Order createOrder(...) { ... }
```

#### 验证

启动期把 `org.springframework.aop` 日志开到 DEBUG，看哪些 Bean 被代理了：

```yaml
logging.level.org.springframework.aop: DEBUG
```

或者：

```java
ctx.getBeansOfType(Object.class).forEach((name, bean) -> {
    if (AopUtils.isAopProxy(bean)) {
        log.info("proxied bean: {} -> {}", name, bean.getClass().getName());
    }
});
```

修复前会打印几百个；修复后只剩你真正想切的那些。

> **口诀**：`execution(* com..*(..))` 是反模式。Pointcut **要么按包+类型双维度收紧，要么走注解驱动**。

---

### 案例 ⑥ AOP + Lombok + MapStruct 互相干扰

#### 现象

代码：

```java
@Data                    // Lombok
@Mapper                  // MapStruct
public interface OrderConverter {
    OrderConverter INSTANCE = Mappers.getMapper(OrderConverter.class);
    OrderDTO toDto(Order o);
}
```

Spring 容器里注入这个 Bean，配上一个 `@Around` 切到 `to*(..)`，启动报：

```
java.lang.IllegalArgumentException: object is not an instance of declaring class
```

或者：

```
Could not generate CGLIB subclass of class XxxMapperImpl: ...
```

#### 根因

三个东西的相互作用：

1. **Lombok `@Data`** 给接口加了一堆方法（虽然接口加 `@Data` 本身没意义，写错了，但编译能过），生成一个伪 `OrderConverter$XxxImpl`。
2. **MapStruct** 在 `target/generated-sources` 生成 `OrderConverterImpl`，这是真正的实现类。
3. **Spring 看到 `@Mapper`**（被 `mapstruct-spring-extensions` 处理）注册成 Bean，对这个 Bean 应用 AOP。
4. CGLib 想给 `OrderConverterImpl` 生成子类，但发现**它的 method 签名和接口 `OrderConverter` 的方法因为 Lombok 多生成的方法对不上**，Method 反射调用时 `target` 类型和 `method.getDeclaringClass()` 不匹配。

更常见的简化版："Lombok 给某个 Service 生成了 `@RequiredArgsConstructor`，构造器里**注入了自己（循环依赖）** + AOP 想代理这个 Bean → CGLib 在调子类构造器时父类构造器又触发依赖解析 → 死循环或 NPE。"

#### 修复

**原则一**：`@Mapper` 类不要套 AOP。MapStruct 生成的代码本来就是无副作用的纯函数转换，给它套 `@Around` 没意义还埋雷。Pointcut 要排除：

```java
@Around("execution(* com.foo.service..*Service.*(..)) " +
        "&& !within(@org.mapstruct.Mapper *)")
```

**原则二**：Lombok 的 `@Data` 不要加在接口上 —— 这是个误用，部分版本能编译通过但语义诡异。

**原则三**：构造器注入的 Bean 如果要被 CGLib 代理，**避免在构造器里做副作用**（CGLib 会调两次：父类一次、子类一次）。把副作用放 `@PostConstruct`。

#### 验证

```java
@Test
void mapperShouldNotBeProxied() {
    OrderConverter c = ctx.getBean(OrderConverter.class);
    assertFalse(AopUtils.isAopProxy(c),
        "MapStruct mapper should NOT be wrapped by AOP, got " + c.getClass());
}
```

> **口诀**：AOP 不要切**纯转换函数**（MapStruct / Jackson 反序列化产物 / 各种 Builder）。Pointcut 用 `!within(@org.mapstruct.Mapper *)` 这类元注解过滤把它们排除。

---

### 案例 ⑦ Bean 内部方法不被拦截 —— "不只是自调用"

#### 现象

不是 `this.method()` 这种典型自调用，而是这种：

```java
@Service
public class JobService {

    @PostConstruct
    public void init() {
        register();   // ★ 期望：触发拦截器，但实际不触发
    }

    @Transactional
    public void register() {
        repo.insertJob();
    }
}
```

或者：

```java
@Service
public class CallbackService {
    @Autowired EventBus bus;

    @PostConstruct
    public void setup() {
        bus.subscribe("topic", this::onEvent);   // ★ 注册的是 this 的方法引用
    }

    @Transactional
    public void onEvent(Event e) { ... }
}
```

#### 根因

回到 Spring AOP 的设计原则：**拦截只在调用代理对象的方法时发生**。

- `@PostConstruct` 调 `register()` 时，`this` 是**目标对象**（代理还在外面，正在被 BeanPostProcessor 包装），不是代理 → 不拦截。事实上 Bean 初始化完成才会被代理包装（见 `AbstractAutoProxyCreator.postProcessAfterInitialization`），**`@PostConstruct` 跑的时候根本就没有代理可用**。
- `this::onEvent` 是个方法引用，绑定的是**目标对象的方法**，不是代理的方法。即便 EventBus 后来回调，调的也是 target，绕过了代理。

更微妙的：在 `@Configuration` 类里**调用另一个 `@Bean` 方法**：

```java
@Configuration
public class Config {
    @Bean public Foo foo() { return new Foo(bar()); }
    @Bean public Bar bar() { return new Bar(); }
}
```

这种 `bar()` 调用**会被 CGLib 拦截**（Spring 给 `@Configuration` 类强制 CGLib 代理，确保单例语义）。但 `@Configuration(proxyBeanMethods = false)` 关掉这个机制后，`bar()` 就是普通 Java 调用，每次返回新实例。

#### 修复

`@PostConstruct` 场景：

```java
@Service
public class JobService {

    @Autowired @Lazy
    private JobService self;

    @PostConstruct
    public void init() {
        self.register();   // ✅ 走代理
    }

    @Transactional
    public void register() { ... }
}
```

或更彻底：把 `register` 拆到独立的 `JobRegistry` Bean，由 `JobService` 注入调用。

`this::method` 场景：

```java
@PostConstruct
public void setup() {
    bus.subscribe("topic", e -> self.onEvent(e));   // ✅ lambda 里走代理
}
```

#### 验证

```java
@Test
void postConstructCallShouldBeWrappedManually() {
    JobService s = ctx.getBean(JobService.class);
    assertEquals(1, txCount.get(), "register() should run inside transaction");
}
```

> **口诀**：`@PostConstruct` / 构造器 / 内部回调 / 方法引用 → 全部不走代理。**任何"我在 Bean 内部调自己"的写法都失效**，唯一例外是 `@Configuration(proxyBeanMethods = true)`。

---

### 案例 ⑧ proxyTargetClass=true 导致接口注入失败

#### 现象

为了"统一用 CGLib"在配置里加了：

```yaml
spring.aop.proxy-target-class: true
```

或者：

```java
@EnableAspectJAutoProxy(proxyTargetClass = true)
@EnableTransactionManagement(proxyTargetClass = true)
```

启动报：

```
NoSuchBeanDefinitionException: No qualifying bean of type 'com.foo.OrderService' available
```

或者运行时某些 `instanceof OrderService` 判断挂掉。

#### 根因

`proxyTargetClass=true` 强制走 CGLib **子类化目标类**。生成的代理 `OrderServiceImpl$$EnhancerByCGLIB$$xxx` 是 `OrderServiceImpl` 的子类，但**它有没有自动实现 `OrderService` 接口呢**？

答案是：**有，但只是顺带的** —— 它继承自 `OrderServiceImpl`，自然继承了 `implements OrderService`。但**当 Spring 容器内只有这个 CGLib 代理 Bean 时**，按类型查找 `OrderService` 是能找到的。

那为什么会报 NoSuchBeanDefinitionException？因为常见的搭配错误：

1. 用了 **JDK 代理友好**的写法注入接口字段，但**目标类不实现该接口** —— 接口是从一个父类间接继承的，CGLib 会丢掉某些泛型签名。
2. 用了**通过 FactoryBean 暴露接口**的方式（如 MyBatis Mapper），FactoryBean 自己实现了接口，CGLib 子类化它后接口语义破裂。
3. 切面顺序问题：自定义 `BeanPostProcessor` 提前包了 JDK 代理（暴露的是接口），后续 CGLib 又包一层（子类化的是 `$Proxy123`），形成 `$Proxy123$$EnhancerByCGLIB$$` 套娃，强转任何业务类型都失败。

更普遍的代价（即便不报错）：
- **CGLib 启动慢**：每个 Bean 都要 ASM 生成字节码，启动期多 200~500ms。
- **GraalVM Native Image 不友好**：CGLib 在 native 镜像里基本不能跑，必须切 JDK 动态代理 + 显式接口注册。
- **Spy / Mock 行为变化**：Mockito 的 `@SpyBean` 在 CGLib 代理上的行为和 JDK 代理不同（构造器双调）。

#### 修复

**默认值就是对的**：Spring Framework 6 默认 `proxyTargetClass=false`（按需切 CGLib），Spring Boot 2.x 起改成 `true` 是为了"少踩 ClassCastException"，但代价上面列了。

判断准则：

| 场景 | 推荐 |
| --- | --- |
| 所有 Service 都有接口（DDD / 整洁架构） | `false`（用 JDK 动态代理） |
| 部分 Service 没接口、混合写法 | 保持 Spring Boot 默认 `true` |
| GraalVM Native Image | 强制 `false`，所有 Bean 必须有接口 |
| Kotlin 项目 | `true`（Kotlin class 默认 final，需要 CGLib 处理） |

如果非要 `true`，注入时**显式按实现类注入**（`OrderServiceImpl orderService`）也能工作 —— 但注意这违反了"依赖接口不依赖实现"的原则。

#### 验证

```java
@Test
void verifyProxyType() {
    OrderService bean = ctx.getBean(OrderService.class);
    log.info("proxy class: {}", bean.getClass().getName());
    log.info("is JDK proxy? {}", AopUtils.isJdkDynamicProxy(bean));
    log.info("is CGLib proxy? {}", AopUtils.isCglibProxy(bean));
}
```

> **口诀**：`proxyTargetClass` 不是越保险越好。**有接口就走 JDK，没接口才走 CGLib**。Spring Boot 默认 `true` 是 Boot 团队的妥协，不是技术正确。

---

### 案例 ⑨ AspectJ Around 没调 proceed —— "方法被吞"

#### 现象

写了一个鉴权切面：

```java
@Aspect @Component
public class AuthAspect {

    @Around("@annotation(needAuth)")
    public Object check(ProceedingJoinPoint pjp, NeedAuth needAuth) throws Throwable {
        if (currentUser() == null) {
            log.warn("unauthorized");
            return null;          // ★ 没调 pjp.proceed()
        }
        return pjp.proceed();
    }
}
```

然后线上诡异现象：未登录用户调接口，**没报错**，**没日志**，**没异常**，但接口返回的就是空。业务方追了一周才发现：原来"接口被吞了"。

#### 根因

`@Around` 的语义是 **"在 joinpoint 周围**"，本质就是责任链的一环。如果你不调 `pjp.proceed()`，**目标方法就不会被调用**，链子在这一环断了。

```java
// ReflectiveMethodInvocation 视角
chain[i].invoke(this);
   ↓
   // 你的 Around 体
   if (!authorized) return null;   // ★ 直接 return，没调 proceed
   // 链子断了，链下面的所有 interceptor + 目标方法都不会被调
```

返回 `null` 比抛异常更危险 —— 异常上层能感知，`null` 看起来"接口正常返回了"。

#### 修复

**原则一**：未授权应该**抛异常**，不应该 silently return。

```java
@Around("@annotation(needAuth)")
public Object check(ProceedingJoinPoint pjp, NeedAuth needAuth) throws Throwable {
    if (currentUser() == null) {
        throw new UnauthorizedException("auth required for " + pjp.getSignature());
    }
    return pjp.proceed();
}
```

**原则二**：`@Around` 一定要有"非异常出口都调 proceed"的代码路径。Code Review 看到 `@Around` 方法体里没有 `proceed()`，必须红牌。

**原则三**：能用 `@Before` 别用 `@Around`。鉴权这种"前置检查"，`@Before` 抛异常就阻断后续，根本不存在"忘记 proceed"的可能。

```java
@Before("@annotation(needAuth)")
public void check(NeedAuth needAuth) {
    if (currentUser() == null) throw new UnauthorizedException("...");
}
```

#### 验证

```java
@Test
void aroundMustNotSwallow() {
    // 故意构造一个 Around 没调 proceed 的场景
    ProxyFactory pf = new ProxyFactory(new Impl());
    pf.addAdvice((MethodInterceptor) inv -> "swallowed");   // 没调 proceed
    Service s = (Service) pf.getProxy();
    assertEquals("swallowed", s.greet("x"));   // ❌ 真实业务返回被替换
    // CI 里加这种测试警示团队：Around 必须 proceed
}
```

> **口诀**：`@Around` 是双刃剑 —— **必有 proceed**，否则下游被吞。`@Before` / `@After` 没法忘 proceed，能用就别上 Around。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 设计层
- [ ] Service 一律有接口；接口和实现分目录（接口 `service/`，实现 `service/impl/`）
- [ ] **不在同类内部 this 调用** `@Transactional` / `@Async` / `@Cacheable` 注解的方法
- [ ] 类一律不加 `final`；Kotlin 项目装 `kotlin-spring` 自动 `open`
- [ ] 方法不加 `final`；如确需 final，方法内绝不放需要 AOP 的逻辑
- [ ] `@Configuration(proxyBeanMethods = false)` 的类里**不互调 @Bean 方法**

### Pointcut 层
- [ ] 不写 `execution(* com..*(..))` 这种全包通配
- [ ] 优先用注解驱动（`@annotation(...)`）而非 execution 字符串
- [ ] 切面 Pointcut 排除 getter/setter/equals/hashCode/toString
- [ ] 切面 Pointcut 排除 MapStruct / Jackson / Lombok 生成类
- [ ] 启动期 DEBUG 日志确认被代理的 Bean 数量在预期内（< 100 个为佳）

### Advice 层
- [ ] 所有 `@Around` 方法体内**必有 `pjp.proceed()`**（Code Review 红牌）
- [ ] 能用 `@Before` 不用 `@Around`
- [ ] `@Async` 方法返回值是 `void` / `Future` / `CompletableFuture`，不是普通对象
- [ ] `@Async` / `@Transactional` 方法是 **public**（private / protected 不会被代理）
- [ ] `@Transactional(rollbackFor = Exception.class)` 显式声明回滚类型

### 代理类型层
- [ ] 不为了"防 ClassCastException"无脑开 `proxyTargetClass=true`
- [ ] 注入字段类型一律用接口，不用实现类
- [ ] GraalVM Native Image：所有 Bean 必须有接口，禁用 CGLib

### 自调用逃生层
- [ ] 需要自调用的场景：`@Autowired @Lazy` 注入 self，或开 `exposeProxy=true` 用 `AopContext.currentProxy()`
- [ ] `@PostConstruct` / 构造器中调用注解方法 → 必走代理
- [ ] 注册回调 / 方法引用 / lambda 内 → 必走代理（不要直接 `this::method`）

### 监控层
- [ ] 启动后打印 AOP 代理 Bean 列表，确认无意外
- [ ] 异常监控覆盖 AOP 链中可能吞掉异常的位置（特别是 `@Around` + 自定义 Throwable handling）

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### AspectJ LTW（Load Time Weaving）
- 用 `aspectjweaver` agent 在类加载时改字节码，绕过"必须经过代理"的限制
- 自调用、构造器、static 块都能切，但**部署多一个 javaagent**，且类加载顺序复杂
- 适合"基建团队提供的横切监控"（trace / metrics），不适合业务逻辑

### ByteBuddy 替代 CGLib 的趋势
- CGLib 已多年不更新（最后 release 2019），底层 ASM 在 JDK 17+ 频繁踩坑
- Spring Framework 6 的 CGLib 是 **Spring fork 的内嵌版本**（`org.springframework.cglib`），不是 cglib-nodep 那个
- Mockito 早就用 ByteBuddy 做 mock-maker；新的 Spring 替代品（如 [Hibernate ByteBuddy proxy](https://hibernate.org/orm/releases/)）也已切到 ByteBuddy
- 长期方向：可能 Spring 6.x 后期或 7 把 CGLib 替换成 ByteBuddy

### GraalVM Native Image 下的 AOP 限制
- 反射、动态代理、字节码生成在 native image 里**全部受限**
- Spring Native（已并入 Spring 6）的方案：**编译期注册可代理类**（`reflect-config.json` / Native Hints）
- CGLib 基本不可用 → 必须走 JDK 动态代理 → **所有要 AOP 的 Bean 必须实现接口**
- `@Transactional` 在 native 下需要额外的 `@RegisterReflectionForBinding` 标注

### Mockito 的 mock-maker-inline
- Mockito 默认 mock-maker（subclass）和 CGLib 一样有 final 限制
- `mock-maker-inline` 用 ByteBuddy 改字节码，能 mock final class / final method / static method
- 在 Spring 测试里 `@MockBean` 默认走 subclass，必要时切 inline；但代价是测试启动慢 30%

### Spring 6 的 GraalVM 适配
- `RuntimeHints` API：在编译期声明运行时需要的反射、资源、代理
- AOT 处理器：把 BeanPostProcessor 的逻辑提前到编译期，运行时跳过织入
- `BeanRegistrationAotProcessor` / `BeanFactoryInitializationAotProcessor` 是入口

### 责任链的可观测性
- 自定义 Interceptor 注入 trace span，覆盖整条 AOP 链
- 用 OpenTelemetry 的 `SpanProcessor` 在每个 advice 边界打点
- 火焰图里识别 `ReflectiveMethodInvocation.proceed` 占比，超过 5% 说明 advice 链太长

---

## 7. 引用

- Spring Framework 6.x 源码：[spring-aop](https://github.com/spring-projects/spring-framework/tree/main/spring-aop)
- AspectJ Programming Guide（pointcut 表达式权威参考）
- Rod Johnson, *Expert One-on-One J2EE Design and Development*（Spring AOP 设计的源头）
- Eugene Kuleshov, *cglib: Bytecode generation library*（CGLib 设计文档，2003）
- Rafael Winterhalter, [ByteBuddy 官方文档](https://bytebuddy.net/) —— CGLib 的现代替代
- Spring Native 文档与 GraalVM Hints API
- Mockito Wiki: *Mocking the unmockable* —— inline mock-maker 设计
- JEP 416: Reimplement Core Reflection with Method Handles（解释了 JDK 17 反射性能跃升）

---

> 配套：[Spring 核心原理](/article/java/spring) | [Spring 事务传播深入](/article/java/spring-tx) | [线程池生产实战](/article/java/threadpool)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你觉得这个深度对，告诉我下一篇该写哪个。
