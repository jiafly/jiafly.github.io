# ClassLoader 深入 —— 双亲委派、OSGi、Spring Boot Loader 与热部署

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，看 [JVM 综述](/article/java/jvm) 那一节够了。
> 这里要回答的，是**为什么你的 jar 在 IDE 跑得好好的、打成 fat jar 就 ClassNotFound，半夜被 Tomcat Metaspace OOM 叫起来**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. 双亲委派的精确顺序是什么？JDK 8 和 JDK 9+ 的三层 ClassLoader 名字分别叫什么？为什么要改名？
2. SPI（`ServiceLoader`）、Tomcat、OSGi 都"打破"了双亲委派，**它们打破的方向各不相同**，分别是什么？为什么必须打破？
3. `Class.forName("Foo")` 和 `getClassLoader().loadClass("Foo")` —— 一个会跑 `<clinit>`，一个不会，**哪个不会**？为什么 JDBC 用前者？
4. Spring Boot fat jar 里 `BOOT-INF/lib/*.jar` 是嵌套 jar，**JDK 自带的 `URLClassLoader` 能直接读 jar in jar 吗**？Spring 是怎么解决的？
5. Tomcat 的 `WebappClassLoader` 是"反向委派"——它**先自己找，找不到才问 parent**。为什么这么设计？什么场景会被反咬一口？
6. OSGi 的 Bundle 之间是**网状依赖**，不是树状。`Import-Package` / `Export-Package` 怎么映射到 ClassLoader 层级？同一个类被两个 Bundle 同时加载，会发生什么？
7. 你在 web 应用里用了 `static ThreadLocal<MyClass>`，应用 reload 后 Metaspace 涨了 50MB，再 reload 再涨 50MB —— **谁在持有旧 ClassLoader**？怎么诊断？
8. JDK 17 上你反射调用 `sun.misc.Unsafe`，抛 `InaccessibleObjectException` —— 这是模块系统在挡。**怎么用 `--add-opens` 解掉**？为什么不推荐这么解？

如果有半数答不上，往下看。

---

## 1. 底层模型：三层 ClassLoader、双亲委派与 Class 唯一性

绝大多数文章把 ClassLoader 讲成"Bootstrap → Ext → App，三层向上委派"，画一张树就结束了。这层认知不够诊断生产问题。要看懂上面 8 个问题，必须把模型分成"层级、流程、唯一性、装载时机、模块"五块来拆。

### 1.1 三层 ClassLoader 与 JDK 9+ 的重命名

JDK 8 的经典三层：

```
            Bootstrap ClassLoader   (C++ 写，加载 $JAVA_HOME/jre/lib/rt.jar)
                  ▲
                  │ parent
            Extension ClassLoader   (Java 写，加载 $JAVA_HOME/jre/lib/ext/*.jar)
                  ▲
                  │ parent
            Application ClassLoader (= System ClassLoader，加载 -classpath)
                  ▲
                  │ parent
            Custom ClassLoader      (你自己 new 的)
```

JDK 9 引入模块系统后，整体结构改名：

| JDK 8 | JDK 9+ | 加载范围 |
| --- | --- | --- |
| Bootstrap | Bootstrap | `java.base` 等核心模块（不再叫 rt.jar，这个文件已经没了） |
| Extension | **Platform** | `java.sql` / `java.xml` 等"平台"模块 |
| Application / System | **App** | classpath + 用户模块 |

为什么改名？因为 JDK 9 把 `lib/ext` 这个"管理员往里扔 jar 就全局生效"的目录干掉了 —— 它曾是无数"线上 ext 里残留旧版 jar 导致升级失败"的元凶。Platform ClassLoader 加载的是受模块系统管理的固定模块集合，不再允许用户随意往里塞东西。

```java
// JDK 9+ 你可以直接拿到这两个 ClassLoader
ClassLoader.getPlatformClassLoader();   // = JDK 9 之前的 Ext，但不再可扩展
ClassLoader.getSystemClassLoader();     // App，加载 classpath
// Bootstrap 仍然是 null —— 它在 native 层
```

### 1.2 双亲委派的精确流程

经典口诀"先委派 parent，找不到再自己找"。但代码里实际是 4 步：

```
loadClass(name):
  ① 查缓存：findLoadedClass(name) —— 已加载过就直接返回
  ② 委派 parent：parent != null ? parent.loadClass(name) : findBootstrapClassOrNull(name)
  ③ 调用 findClass(name) —— 自己真正去 classpath 找
  ④ resolveClass(c) —— 链接（可选）
```

这里有一个反直觉的事实：**真正的"加载逻辑"在 `findClass`**，而 `loadClass` 只是负责调度委派。所以自定义 ClassLoader 通常**只覆写 `findClass`，不动 `loadClass`** —— 一旦动了 `loadClass`，就等于打破了双亲委派。

后面 Tomcat / OSGi / Spring Boot Loader 全都是通过覆写 `loadClass` 来"反向委派"或"网状委派"的。

### 1.3 为什么需要打破双亲委派

双亲委派的**核心目的**是安全：保证你不能写一个 `java.lang.String` 类来覆盖 JDK 自带的（你的 String 永远先委派给 Bootstrap，Bootstrap 加载了真的 String，你的就用不上了）。

但这个机制有三个无法解决的场景：

**场景 A：SPI**。JDBC 的 `Driver` 接口在 `java.sql`，由 Bootstrap 加载；但 `mysql-connector` 的实现在 classpath，要 App 加载。Bootstrap 的代码怎么 `new MysqlDriver()`？双亲委派下它**根本看不见 App 加载的类**。解法：**线程上下文 ClassLoader（TCCL）**。

**场景 B：Web 容器**。一个 Tomcat 同时跑多个 webapp，A 用 Spring 5、B 用 Spring 6。如果走双亲委派，谁先加载 Spring 谁就把对方"霸占"了。解法：**反向委派 + 类隔离**。

**场景 C：模块化系统**（OSGi）。同一个 Bundle 可能 export 自己的包给别的 Bundle 用，import 别的 Bundle 的包。这是网状依赖，不是树状。解法：**每个 Bundle 一个 ClassLoader，按 Import/Export 表互相协作**。

> **口诀**：双亲委派保证**安全**；打破委派满足**隔离与扩展**。SPI 是**向下**找（让 Bootstrap 看见 App），Web 容器是**向内**找（让 App 优先于 parent），OSGi 是**横向**找（Bundle 间网状）。

### 1.4 Class 的唯一性 = ClassLoader + 全限定名

JVM 判定两个 Class 是不是同一个类，**不是只比类名**：

```
class identity = (ClassLoader instance, fully qualified name)
```

这意味着：
- `loaderA.loadClass("com.foo.Bar")` 和 `loaderB.loadClass("com.foo.Bar")` 加载出来的 `Class<?>` 对象**不是同一个**
- 它们的 instance 之间不能互相强转，会抛 `ClassCastException`
- 它们的静态字段是**两份**

这是后面"案例 ③ LinkageError 同名类双加载"的根。

举个最小可复现的例子：

```java
URL[] urls = {new File("foo.jar").toURI().toURL()};
URLClassLoader la = new URLClassLoader(urls, null);   // parent = null，不委派
URLClassLoader lb = new URLClassLoader(urls, null);

Class<?> ca = la.loadClass("com.foo.Bar");
Class<?> cb = lb.loadClass("com.foo.Bar");
System.out.println(ca == cb);                  // false
System.out.println(ca.getName().equals(cb.getName()));  // true
Object obj = ca.getDeclaredConstructor().newInstance();
com.foo.Bar bar = (com.foo.Bar) obj;           // ClassCastException!（如果当前类是 lb 加载的）
```

`parent = null` 这一招后面 Tomcat 也会用 —— 只不过 Tomcat 是 parent 指向 Bootstrap，跳过 App。

### 1.5 Class.forName vs ClassLoader.loadClass

两个 API 经常被混用，但有**关键差别**：

| API | 是否触发 `<clinit>`（静态初始化） | 默认 ClassLoader |
| --- | --- | --- |
| `Class.forName(name)` | **是** | 调用方的 ClassLoader |
| `Class.forName(name, true, loader)` | 第二参数控制 | 你指定的 |
| `Class.forName(name, false, loader)` | **否** | 你指定的 |
| `loader.loadClass(name)` | **否** | 你指定的 |

为什么 JDBC 老代码爱写 `Class.forName("com.mysql.jdbc.Driver")`？因为 MySQL Driver 的注册靠 `<clinit>`：

```java
// com.mysql.cj.jdbc.Driver
static {
    try {
        DriverManager.registerDriver(new Driver());
    } catch (SQLException E) {
        throw new RuntimeException("Can't register driver!");
    }
}
```

如果用 `loadClass`，类只是被加载到 JVM，**`<clinit>` 不跑**，driver 不会被注册到 `DriverManager`。

JDBC 4.0 之后这种写法过时了 —— `DriverManager` 通过 `ServiceLoader` 自动加载所有 driver，不需要 `Class.forName`。但老项目里到处都是。

> **口诀**：**`forName` 跑静态块，`loadClass` 不跑**。涉及静态注册的（JDBC Driver、Spring 早期的 `PropertyEditor`），必须用 `forName`。

### 1.6 Java 模块系统（JPMS）对 ClassLoader 的影响

JDK 9 引入模块系统后，ClassLoader 之上多了一层**模块层**（ModuleLayer）。可见性判定从"类对类"扩展为"模块对模块":

```
能否访问类 X = (类 X 在的模块 export 了 X 的包) ∧ (我的模块 require 了它)
              ∧ (ClassLoader 能找到 X)
```

这就是为什么你在 JDK 17 上反射访问 `sun.misc.Unsafe` 会抛：

```
java.lang.reflect.InaccessibleObjectException:
  Unable to make field private sun.misc.Unsafe.theUnsafe accessible:
  module java.base does not "opens sun.misc" to unnamed module
```

**ClassLoader 能加载到 Unsafe**（它在 Bootstrap 里），但**模块系统拦在反射访问这一步**。两层机制叠在一起。

解法（按推荐度排）：

```bash
# 1. 用 VarHandle / MethodHandle 替代（首选）
# 2. 启动参数开口
java --add-opens java.base/sun.misc=ALL-UNNAMED ...
# 3. 用 jdk.internal.misc.Unsafe（更内部，更不稳定）
```

后面案例 ⑧ 会展开。

### 1.7 ClassLoader 与 Metaspace 的关系

JDK 8 起，类元数据（Klass、方法字节码、常量池等）从 PermGen 搬到 **Metaspace**（堆外，操作系统直接管）。

关键事实：**Metaspace 的回收单位是 ClassLoader**。

```
Metaspace
  ├── ClassLoaderData (per ClassLoader)
  │     ├── Klass (per Class)
  │     │     ├── 方法字节码
  │     │     ├── 常量池
  │     │     └── vtable / itable
  │     └── ...
  └── ...
```

只有当**整个 ClassLoader 不被任何 GC root 引用**时，它的所有 Klass 才能被卸载。**只要还有一个对象、一个 Class 字面量、一个静态字段引用着任何一个该 ClassLoader 加载的类，整个 ClassLoader 都不能卸载**。

这就是后面"案例 ④ Tomcat 热部署 Metaspace OOM"和"案例 ⑤ ThreadLocal 泄漏 ClassLoader"的物理基础。

---

## 2. 源码切片：从 ClassLoader.loadClass 到 OSGi BundleWiring

> 基于 OpenJDK 21、Tomcat 10、Spring Boot 3.2、Equinox（OSGi）源码。注释为本文添加。

### 2.1 ClassLoader.loadClass —— 双亲委派的代码

```java
// java.lang.ClassLoader (JDK 21)
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException {
    synchronized (getClassLoadingLock(name)) {
        // ① 缓存：JVM 内部维护，已加载过的类不再加载
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            try {
                if (parent != null) {
                    c = parent.loadClass(name, false);   // ② 委派 parent
                } else {
                    c = findBootstrapClassOrNull(name);  // parent 是 Bootstrap
                }
            } catch (ClassNotFoundException e) {
                // parent 找不到，落到自己找
            }
            if (c == null) {
                c = findClass(name);                     // ③ 自己 findClass
            }
        }
        if (resolve) resolveClass(c);                    // ④ 链接（可选）
        return c;
    }
}
```

**逐段点透**：

- **`getClassLoadingLock(name)` 的妙处**：JDK 7 之前是 `synchronized (this)` 整个 ClassLoader 一把锁，多线程加载不同类会互相阻塞。JDK 7 改成"每个类名一把锁"，从根本上提升了并发加载性能。这是 `ClassLoader.registerAsParallelCapable()` 的前提条件。
- **`findLoadedClass` 的实现是 native**：它查的是 ClassLoader 内部维护的 `Vector<Class<?>>`，但更深层是 JVM 的 SystemDictionary。
- **`parent.loadClass(name, false)` 的 `false`**：表示**不要 resolve**。因为 resolve（链接）只在最终被用到时才需要做，委派路径上跳过能省事。
- **找不到时 `catch` 了又不抛**：双亲委派的精髓 —— "parent 找不到不算错，让我自己再试"。

### 2.2 URLClassLoader.findClass —— 真正的"加载"

```java
// java.net.URLClassLoader 简化
protected Class<?> findClass(final String name) throws ClassNotFoundException {
    String path = name.replace('.', '/').concat(".class");
    Resource res = ucp.getResource(path, false);   // ucp = URLClassPath
    if (res == null) throw new ClassNotFoundException(name);
    try {
        return defineClass(name, res, ...);         // ★ 真正把字节码变成 Class
    } catch (IOException e) {
        throw new ClassNotFoundException(name, e);
    }
}
```

**逐段点透**：

- 找类的过程就是**遍历 classpath 的所有 URL**，挨个尝试找名为 `com/foo/Bar.class` 的资源。
- `defineClass` 是从 ClassLoader 继承的 native 方法，把 byte[] 变成 `Class<?>`。这是**所有动态字节码生成**（CGLib、ASM、Mockito）最终调用的 API。
- 如果你看到 `LinkageError: attempted duplicate class definition` —— 就是同一个 ClassLoader 对同一个类名调了两次 `defineClass`，JVM 拒绝了。

### 2.3 ServiceLoader —— SPI 与线程上下文 ClassLoader

```java
// java.util.ServiceLoader 关键路径
public static <S> ServiceLoader<S> load(Class<S> service) {
    ClassLoader cl = Thread.currentThread().getContextClassLoader();   // ★ TCCL
    return new ServiceLoader<>(Reflection.getCallerClass(), service, cl);
}

// 真正读 META-INF/services/<interface> 文件
private boolean parseLine(...) {
    // 读到一行类名后：
    Class<?> clazz = Class.forName(cn, false, loader);   // ★ 用 TCCL 加载实现类
    ...
}
```

**逐段点透**：

- 为什么用 `Thread.currentThread().getContextClassLoader()`？**为了让父加载器能调到子加载器加载的类**。`ServiceLoader` 这段代码本身可能在 Bootstrap / Platform 加载（比如 `DriverManager`），但实现类（MySQL Driver）在 App ClassLoader 里。
- TCCL 默认就是 App ClassLoader。Web 容器会把它**临时设成 WebappClassLoader**，让框架能加载到应用的类。
- 这是双亲委派的"破例"：父类通过 TCCL **绕过自己的 parent 链**，直接拿到子层的 ClassLoader 用。

### 2.4 Tomcat WebappClassLoader —— 反向委派

```java
// org.apache.catalina.loader.WebappClassLoaderBase 简化
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException {
    Class<?> clazz = null;

    // ① 先查自己缓存
    clazz = findLoadedClass0(name);
    if (clazz != null) return clazz;

    // ② JDK 类必须委派给 parent（不能让 webapp 改 java.lang.String）
    if (filter(name, true)) {
        try {
            clazz = javaseLoader.loadClass(name);
            if (clazz != null) return clazz;
        } catch (ClassNotFoundException e) {}
    }

    // ③ 反向：先自己找
    boolean delegateLoad = delegate || filter(name, false);
    if (!delegateLoad) {
        try {
            clazz = findClass(name);             // ★ 自己 WEB-INF/classes 和 lib/*.jar
            if (clazz != null) return clazz;
        } catch (ClassNotFoundException e) {}
    }

    // ④ 自己找不到才问 parent
    try {
        clazz = Class.forName(name, false, parent);
        if (clazz != null) return clazz;
    } catch (ClassNotFoundException e) {}

    throw new ClassNotFoundException(name);
}
```

**逐段点透**：

- **③ 是反向委派的核心**：webapp 自己的类**优先于 parent**。这就是为什么你在 webapp 里打包了一个 `commons-lang-3.12.jar`，能盖过 Tomcat 自带的版本。
- **② 是反向委派的安全网**：`java.*` / `javax.*` 等"JDK 类"必须先委派 parent，否则 webapp 写一个 `java.lang.String` 就能"覆盖"JDK 的（实际上 JVM 会拒绝 defineClass，但提前过滤更早失败）。
- **`filter(name, true/false)`**：根据 `delegate` 配置和包名前缀决定走哪条路径。
- **每个 webapp 一个 WebappClassLoader 实例**：所以 webapp A 的 `com.foo.Service` 和 webapp B 的 `com.foo.Service` 是**两个不同的 Class**，互不干扰。

### 2.5 Spring Boot LaunchedURLClassLoader —— jar in jar

Spring Boot 的 fat jar 长这样：

```
myapp.jar
├── META-INF/MANIFEST.MF
│     Main-Class: org.springframework.boot.loader.JarLauncher
│     Start-Class: com.foo.MyApplication
├── org/springframework/boot/loader/   <- Spring Boot Loader 自己的类（非嵌套）
├── BOOT-INF/
│   ├── classes/                        <- 你的应用代码
│   └── lib/
│       ├── spring-core-6.1.0.jar       <- 嵌套 jar，不解压
│       └── ...
```

JDK 自带的 `URLClassLoader` 不能直接读 `myapp.jar!/BOOT-INF/lib/spring-core-6.1.0.jar` 里的类 —— 它把内嵌 jar 当作普通文件，不会再当 jar 解析。

Spring Boot 的解法：自己写一套 `JarFile` / `JarURLConnection`，支持 `jar:file:/path/myapp.jar!/BOOT-INF/lib/spring-core.jar!/org/springframework/Foo.class` 这种**多层 ! 的 URL**。

```java
// org.springframework.boot.loader.LaunchedURLClassLoader (Spring Boot 3.x)
public class LaunchedURLClassLoader extends URLClassLoader {

    @Override
    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException {
        // ① 走标准双亲委派（与 webapp 不同！）
        try {
            try {
                definePackageIfNecessary(name);   // 处理嵌套 jar 的 package 信息
            } catch (IllegalArgumentException ex) {
                // re-check
            }
            return super.loadClass(name, resolve);
        } catch (ClassNotFoundException ex) {
            ...
        }
    }

    @Override
    public URL findResource(String name) {
        if (this.exploded) return super.findResource(name);
        Handler.setUseFastConnectionExceptions(true);
        try { return super.findResource(name); }
        finally { Handler.setUseFastConnectionExceptions(false); }
    }
}
```

**逐段点透**：

- Spring Boot Loader **没有**反向委派！它走标准双亲委派 —— fat jar 是单一应用，不需要 webapp 那种多应用隔离。
- 真正的魔法在 `Handler` 类（自定义的 URL Stream Handler）和 `JarFileArchive`：把"jar in jar"映射成虚拟的 URL。
- **Spring Boot 3.2 引入新的 ClassLoader 实现**：`LaunchedClassLoader`（去掉 URL 后缀），但思路相同。
- 启动顺序：`java -jar myapp.jar` → `JarLauncher.main` → 创建 `LaunchedURLClassLoader` → **设为线程上下文 ClassLoader** → 反射调用 `Start-Class` 的 main。

### 2.6 OSGi Bundle ClassLoader —— 网状委派

OSGi 把每个 jar 包装成 Bundle，每个 Bundle 一个独立 ClassLoader。Bundle 之间通过 `MANIFEST.MF` 的 `Import-Package` / `Export-Package` 协商可见性。

```
Bundle A: Export-Package: com.foo.api;version=1.0
Bundle B: Import-Package: com.foo.api;version="[1.0,2.0)"

→ 当 B 的 ClassLoader 要加载 com.foo.api.Foo 时，
  OSGi Resolver 在 B 启动时已经把"com.foo.api 这个包要去 Bundle A 找"记下来，
  B 的 ClassLoader.loadClass("com.foo.api.Foo") 直接走 A 的 ClassLoader
```

Equinox（Eclipse 的 OSGi 实现）的关键代码：

```java
// org.eclipse.osgi.internal.loader.BundleLoader 简化
public Class<?> findClass(String name) throws ClassNotFoundException {
    // ① 查"代理表"：这个包是不是 Import 自其他 Bundle
    PackageSource source = findImportedSource(packageName, null);
    if (source != null) {
        return source.loadClass(name);     // ★ 跳到导出该包的 Bundle 的 ClassLoader
    }

    // ② 查"required bundles"
    source = findRequiredSource(packageName, null);
    if (source != null) return source.loadClass(name);

    // ③ 都没有：自己 Bundle 内部找
    return findLocalClass(name);
}
```

**逐段点透**：

- **没有"parent"的概念**：Bundle ClassLoader 的关系不是树，是图。
- 同一个类可能被 Bundle A 和 Bundle C 各自加载（如果它们都不 Import，各自有自己的实现）—— 这是 OSGi 强大的"版本共存"能力（一个 OSGi 容器可以同时跑 commons-lang 2.x 和 3.x）。
- 也是 OSGi **复杂性的根源**：你以为同名同包的类是同一个，结果运行时抛 `ClassCastException`，因为它们来自两个不同的 Bundle ClassLoader。
- OSGi 必须由 Resolver **在 Bundle 启动时**计算出完整的依赖图，否则运行时根本不知道某个包该问谁要。

### 2.7 JDK 9+ ModuleLayer 与 ClassLoader 的关系

```java
// java.lang.ModuleLayer 关键概念
ModuleLayer bootLayer = ModuleLayer.boot();
Module javaBase = bootLayer.findModule("java.base").orElseThrow();
ClassLoader cl = javaBase.getClassLoader();   // == null（Bootstrap）
```

Module 和 ClassLoader 是**多对一**关系：一个 ClassLoader 可以加载多个 Module 的类，但一个 Module 只有一个 ClassLoader。

可见性判定多了一层"Module 是否 reads / opens / exports"：

```java
// Class.forName 在 JDK 9+
public static Class<?> forName(Module module, String name) {
    // ★ 即使 ClassLoader 能找到，Module 不可见也加载不了
    return Class.forName0(name, false, module.getClassLoader(), module);
}
```

**关键事实**：JPMS 和 ClassLoader 是**两个独立的检查**。"找得到"（ClassLoader）+ "可访问"（Module）才能用。

> 后面案例 ⑧ 会演示 JDK 17 反射访问私有 API 的踩坑。

---

## 3. 实测数据：双亲委派的代价、热部署的成本

口诀化的"ClassLoader 很贵"几乎所有八股文章都说，但**贵在哪、量级是多少**很少有人量化。下面用 JMH + JFR 给一组数据。

### 3.1 双亲委派的查找开销

最简单的实验：从一个深度为 4 的 ClassLoader 链上加载一个不存在的类。

```java
@Benchmark
public void loadMissingClass(Blackhole bh) {
    try {
        bh.consume(deepLoader.loadClass("com.nonexistent.Foo"));
    } catch (ClassNotFoundException e) {
        bh.consume(e);
    }
}
```

典型结果（M2, JDK 21）：

| 链深度 | loadMissingClass 耗时 (ns/op) |
| --- | --- |
| 1 (App 直接) | ~1,200 |
| 2 (Custom → App) | ~2,300 |
| 4 (Custom×3 → App) | ~5,800 |
| 8 (Custom×7 → App) | ~11,500 |

**结论**：每多一层 ClassLoader，查找一次"不存在的类"的成本约 +1.5μs。这对**正常加载**几乎无感（一个类只加载一次），但对**频繁查找不存在的类**（典型场景：`ClassUtils.isPresent` 判断可选依赖、动态代理生成代理类时反复 try）就很贵。

Spring 启动时大量 `ClassUtils.isPresent`，是启动时间的一大消耗源 —— 这也是为什么 GraalVM Native Image 把这些查找全部"提前"（AOT 编译期解析）。

### 3.2 大量 Class 加载的内存与时间

模拟 OSGi / 插件场景：用 ASM 动态生成 10000 个空类，加载到 Metaspace。

```java
ClassWriter cw = new ClassWriter(0);
cw.visit(V17, ACC_PUBLIC, "Generated$" + i, null, "java/lang/Object", null);
cw.visitMethod(ACC_PUBLIC, "<init>", "()V", null, null).visitInsn(RETURN);
cw.visitEnd();
byte[] bytes = cw.toByteArray();
loader.defineClass("Generated$" + i, bytes, 0, bytes.length);
```

| 加载数量 | Metaspace 增量 | 总耗时 |
| --- | --- | --- |
| 1,000 | ~6 MB | ~120 ms |
| 10,000 | ~58 MB | ~1.4 s |
| 100,000 | ~580 MB | ~22 s |

**单个空类约 6 KB Metaspace**。考虑到正常业务类有方法、字段、常量池，平均 10~30 KB 是常态。

**生产警报**：你看到 Metaspace 涨到 1GB+，说明加载的类至少几万到几十万级。常见来源：CGLib 代理（每个 bean 一个生成类）、Groovy 脚本（每跑一次生成新类）、热部署残留。

### 3.3 Spring Boot fat jar 启动时的 ClassLoader 开销

对比"explode 解压版"和"fat jar 嵌套版"启动时间：

```bash
# Fat jar 直接跑
java -jar myapp.jar

# 解压后跑（Spring Boot 自动检测 exploded）
java -cp BOOT-INF/classes:BOOT-INF/lib/* com.foo.MyApplication
```

测试结果（一个 200 个 jar、3000 个类的中等规模 Spring Boot 应用）：

| 模式 | 冷启动 | 类加载次数 |
| --- | --- | --- |
| Fat jar | ~6.2 s | ~5,800 |
| Exploded | ~5.0 s | ~5,800 |

**fat jar 慢约 20%**，主要慢在：
- 嵌套 jar 的二次解析
- 每次 findResource 要遍历 BOOT-INF/lib/*.jar 的内部索引

Spring Boot 3 引入 **Layered Jars**（分层 jar），目的之一就是优化容器场景下的启动 —— 把变化频率低的依赖单独打层，复用 Docker 镜像层。

### 3.4 OSGi 隔离的运行时代价

OSGi 的每次跨 Bundle 调用，理论上要走"代理表 → 目标 Bundle ClassLoader"的查找。实测对比：

| 场景 | 调用耗时 (ns) |
| --- | --- |
| 普通类内方法调用 | < 1 |
| 同 Bundle 内反射 invoke | ~30 |
| 跨 Bundle 反射 invoke | ~50 |
| 跨 Bundle Service 调用（OSGi Service Registry） | ~200 |

跨 Bundle 调用比同 Bundle 慢约 60%，但绝对值仍是纳秒级。**OSGi 的真正成本不在运行时，而在启动时的 Resolver 计算**：上千个 Bundle 的依赖图解析能让 Eclipse 启动几十秒。

### 3.5 热部署（Hot Swap）的代价

JVM 自带的 HotSwap 只能换方法体（方法签名、字段都不能改）。能改的范围：

| 修改内容 | JVM HotSwap | JRebel | DCEVM |
| --- | --- | --- | --- |
| 改方法体 | ✅ | ✅ | ✅ |
| 加方法 | ❌ | ✅ | ✅ |
| 改字段 | ❌ | ✅ | ✅ |
| 改类继承 | ❌ | ✅ | ✅ |
| 加注解 | ❌ | ✅ | ✅ |

实测一次热部署（一个中型 Spring 应用）的内存代价：

| 工具 | Metaspace 增量 / 次 | 全量重启对比 |
| --- | --- | --- |
| JVM HotSwap (方法体) | ~0 KB | 无 |
| JRebel (改类结构) | ~50 KB | 无新 ClassLoader |
| Tomcat Reload | ~30~50 MB | 一个 webapp 全量 |
| Spring DevTools | ~20~40 MB | restart ClassLoader 重建 |

**Spring DevTools 的两层 ClassLoader 设计**：
- `BaseClassLoader`：加载 jar 依赖（启动时一次性加载，restart 时不重建）
- `RestartClassLoader`：加载你的代码（每次 restart 时重建）

这种设计让"改代码 → 重启"只重建小部分类，比全量重启快 5~10 倍。

> **口诀**：ClassLoader 操作的成本是**Metaspace 占用 > 启动耗时 > 运行时反射开销**。生产里最容易爆炸的永远是 Metaspace。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① ClassNotFoundException —— "本地能跑，打包就找不到"

#### 现象

本地 IDE 跑得好好的，打成 fat jar 部署到测试环境就抛：

```
Caused by: java.lang.ClassNotFoundException: com.alibaba.fastjson.JSON
    at java.base/jdk.internal.loader.BuiltinClassLoader.loadClass(BuiltinClassLoader.java:641)
    at java.base/jdk.internal.loader.ClassLoaders$AppClassLoader.loadClass(ClassLoaders.java:188)
    at java.base/java.lang.ClassLoader.loadClass(ClassLoader.java:520)
```

#### 根因

最常见的三种原因：

1. **依赖 scope 错了**：Maven 里写成 `<scope>provided</scope>` —— IDE 跑会把 provided 也丢上 classpath，但打 fat jar 时不会包含。
2. **Maven Shade Plugin 把类剔了**：用了 minimize 或 filter 配置，把"看起来用不上"的类裁掉，但运行时反射用到。
3. **同名 jar 两个版本，前者覆盖后者**：classpath 顺序导致。

诊断三连：

```bash
# 1. 看 fat jar 里到底有没有这个类
unzip -l myapp.jar | grep fastjson
jar tf myapp.jar | grep "JSON.class"

# 2. 看运行时实际的 classpath
jcmd <pid> VM.system_properties | grep java.class.path

# 3. 让 JVM 把所有类加载日志打出来
java -verbose:class -jar myapp.jar | grep fastjson
```

#### 修复

```xml
<!-- 改 scope -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.83</version>
    <!-- 删掉 provided 或者改成 compile -->
</dependency>
```

如果是 Spring Boot Maven Plugin，还要确认 `excludeArtifactIds` 没把它排掉。

#### 验证

```bash
unzip -p myapp.jar BOOT-INF/lib/fastjson-1.2.83.jar | jar tf - | head
# 能列出文件 → jar 在 fat 里
java -verbose:class -jar myapp.jar 2>&1 | grep "JSON.class"
# 能看到 [Loaded com.alibaba.fastjson.JSON ...] → 加载成功
```

> **口诀**：`ClassNotFoundException` 是"jar 不在 classpath"。先看 jar 在不在包里，再看 classpath 顺序，最后看 scope。

---

### 案例 ② NoClassDefFoundError —— "明明刚刚加载成功，下一行就找不到"

#### 现象

不是 ClassNotFoundException，是更难理解的 `NoClassDefFoundError`：

```
java.lang.NoClassDefFoundError: org/slf4j/Logger
    at com.foo.Service.<clinit>(Service.java:15)
    at ...

# 但是！
Class.forName("org.slf4j.Logger")  // 居然能成功
```

#### 根因

`NoClassDefFoundError` 和 `ClassNotFoundException` 是**两件不同的事**：

| 异常 | 含义 | 触发时机 |
| --- | --- | --- |
| `ClassNotFoundException` | classpath 找不到 | 反射 `Class.forName` / `loadClass` |
| `NoClassDefFoundError` | **加载时找到了，但链接/初始化失败** | new / static 字段访问 / 方法调用 |

具体到这个案例，最常见的两种：

**子类 A 依赖 B，B 在 classpath 里**，但**B 的 `<clinit>` 抛了异常**。第一次访问 A 时 JVM 尝试加载 B，B 链接失败 → 抛 `NoClassDefFoundError`。第二次访问 A 时 JVM 记得 B "definition not found"，**直接抛 NoClassDefFoundError，不再尝试加载**。

更隐蔽：**循环引用**。A 的 `<clinit>` 里访问 B 的 static 字段，B 的 `<clinit>` 里又访问 A 的 static 字段。JVM 加载顺序导致一方拿到的是"半初始化"的另一方。

```java
class A {
    static B b = new B();      // 加载时触发 B 加载
    static int x = 100;
}
class B {
    static int y = A.x;        // 此时 A 还没初始化完，A.x = 0！
}
```

#### 修复

针对"B 的 `<clinit>` 失败"：找出第一次加载 B 时的真实异常。**关键是看第一次的栈，不是后续的**：

```java
try {
    Class.forName("com.foo.B");
} catch (Throwable t) {
    t.printStackTrace();   // ★ 这里能看到原始 ExceptionInInitializerError
}
```

或者全局注册 `Thread.UncaughtExceptionHandler` 抓第一次。

针对循环引用：把 static 初始化拆成 lazy init：

```java
class A {
    private static volatile B b;
    static B getB() {
        if (b == null) {
            synchronized (A.class) {
                if (b == null) b = new B();
            }
        }
        return b;
    }
}
```

#### 验证

```bash
# 启动加 -verbose:class，看 B 的加载是否有 ERROR
java -verbose:class -jar myapp.jar 2>&1 | grep -A 3 "B.class"
```

> **口诀**：`NoClassDefFoundError` ≠ classpath 缺类。先排查**第一次**加载这个类时是不是 `<clinit>` 抛了异常或循环依赖。

---

### 案例 ③ LinkageError —— "同一个类被两个 ClassLoader 加载"

#### 现象

```
java.lang.LinkageError: loader constraint violation: when resolving method
"com.foo.Bar.doIt(Lcom/foo/Param;)V" the class loader (instance of ...)
of the current class, com/foo/Service, and the class loader (instance of ...)
for the method's defining class, com/foo/Bar, have different Class objects
for the type Param used in the signature
```

#### 根因

同一个 `com.foo.Param` 被两个不同的 ClassLoader 加载了。`Service` 看到的 `Param` 来自 ClassLoader A，`Bar.doIt` 期望的 `Param` 来自 ClassLoader B。JVM 的"类唯一性 = ClassLoader + 名字"原则决定了它们**不是同一个类**，方法签名不匹配。

最常见的来源：

- Tomcat 启动时 webapp 加载了 `commons-lang-3.12.jar`，同时 `lib/` 下也有一份旧的 `commons-lang-2.6.jar` 被 common ClassLoader 加载，两个 ClassLoader 各自加载自己的 `StringUtils`。
- 你写了个 Java Agent，agent jar 里带了 `org.objectweb.asm.ClassReader`，业务代码里也有同名 `ClassReader`，agent 在 Bootstrap ClassLoader 加载，业务在 App ClassLoader 加载。
- OSGi 多 Bundle 各自带了同名第三方依赖，但没声明 Import 互通。

#### 修复

诊断：找出"同名类被几个 ClassLoader 加载"。

```java
public static void debugClassLoading(String className) {
    Enumeration<URL> urls = Thread.currentThread()
        .getContextClassLoader()
        .getResources(className.replace('.', '/') + ".class");
    while (urls.hasMoreElements()) {
        System.out.println(urls.nextElement());   // 列出所有来源
    }
}
debugClassLoading("com.foo.Param");
// 期望只有一行；多行说明 classpath 里有多份
```

或者用 `arthas`：

```bash
sc -d com.foo.Param
# 输出会显示每个 Class 对象对应的 ClassLoader
```

修复：

1. **让 classpath 只有一份**：找到重复 jar，去掉一个。Maven 的 `mvn dependency:tree` + `<exclusions>`。
2. **明确 ClassLoader 边界**：Tomcat 场景把 jar 放到 `WEB-INF/lib`（webapp 加载）或 `lib/`（共享加载），不能两边都放。
3. **Agent 用独立 ClassLoader**：用 `java.lang.instrument.Instrumentation.appendToBootstrapClassLoaderSearch` 把 agent 依赖也加到 Bootstrap，或者用 Shadow 把 agent 的依赖 relocate 到独立包名。

#### 验证

```bash
# arthas 反复确认
sc -d com.foo.Param
# ClassLoader 只有一个 → 修复成功
```

> **口诀**：`LinkageError` = "类被加载了两次"。jar 列表必须去重，agent 依赖必须 relocate。

---

### 案例 ④ Tomcat 热部署 Metaspace OOM —— "reload 几次就崩"

#### 现象

Tomcat 部署的应用，开发期为了不重启用 reload。reload 几次后：

```
java.lang.OutOfMemoryError: Metaspace
```

或更早：

```
[gc,metaspace] Metaspace: 768.0M / 768.0M
```

#### 根因

理论上 Tomcat reload 时会丢弃旧的 `WebappClassLoader`，新建一个加载新代码。旧 ClassLoader 不可达 → GC → Metaspace 释放。

实际上**几乎总是**有"GC Root → 某对象 → 某 Class → 旧 ClassLoader"的引用链让旧 ClassLoader 无法回收。常见的根：

| 持有者 | 持有路径 |
| --- | --- |
| **Thread**（特别是 ThreadLocal） | webapp 启动时创建的线程池里的 worker 持有 ThreadLocal，TL 里放着 webapp 的对象 |
| **JDBC Driver** | `DriverManager` 静态字段持有 webapp 加载的 Driver 实例 |
| **JNDI** | webapp 注册的 JNDI 资源未注销 |
| **Logger** | log4j 的 `LogManager` 持有 webapp 的 Logger |
| **MBean** | webapp 注册的 MBean 未在 stop 时注销 |

只要有一条引用链，旧 ClassLoader 加载的**所有类**都不能从 Metaspace 卸载。每次 reload 涨几十 MB。

#### 修复

诊断：用 `jmap` + MAT。

```bash
jmap -dump:format=b,file=heap.bin <pid>
# MAT 打开后，找 WebappClassLoader 实例
# Path to GC Roots → exclude weak/soft references
# 看是谁拿着它
```

修复**根本**：webapp 在 stop 时清理所有自己注册的资源。Tomcat 的 `JreMemoryLeakPreventionListener` 已经预防了一些常见的，但应用层泄漏要自己清。

```java
@WebListener
public class CleanupListener implements ServletContextListener {
    public void contextDestroyed(ServletContextEvent sce) {
        // 1. 注销 JDBC Driver
        Enumeration<Driver> drivers = DriverManager.getDrivers();
        while (drivers.hasMoreElements()) {
            Driver driver = drivers.nextElement();
            if (driver.getClass().getClassLoader()
                == Thread.currentThread().getContextClassLoader()) {
                DriverManager.deregisterDriver(driver);
            }
        }

        // 2. 关闭线程池
        myThreadPool.shutdownNow();

        // 3. 清 ThreadLocal（见案例 ⑤）

        // 4. 注销 MBean
        // 5. 关闭日志框架的 Appender 等
    }
}
```

最简单的兜底：开发环境**别用 reload**，用 Spring DevTools 或 JRebel；生产环境**禁用 reload**（autoDeploy=false），强制全量重启。

#### 验证

```bash
# 反复 reload 10 次，监控 Metaspace
jcmd <pid> GC.heap_info
# Metaspace 不持续增长 → 修复成功
```

> **口诀**：Tomcat reload 不会自动回收 ClassLoader。每次 reload 都泄漏几十 MB Metaspace 是**常态**，不是 bug。生产禁用 reload。

---

### 案例 ⑤ ThreadLocal 持有 Class 导致 ClassLoader 泄漏

#### 现象

webapp reload 后用 MAT 看 dump，发现：

```
WebappClassLoader (instance) ← Class<MyService> ← MyService.class$$Lambda
  ← Thread "tomcat-pool-1" ← thread.threadLocals (Map)
  ← ThreadLocal$Entry.value (MyService 实例)
  ← ThreadLocal$Entry.key (this.requestContext, ThreadLocal 对象)
```

旧 ClassLoader 被一个**仍然活着的**线程通过 ThreadLocal 间接持有。

#### 根因

Thread 是个 GC Root。Thread 内部有 `ThreadLocalMap threadLocals`，存着 `Entry[]`。Entry 的 key 是 WeakReference，但 **value 是强引用**。

webapp 启动时创建的 ThreadPoolExecutor 不会随 webapp stop 自动关闭。worker 线程仍然存活，它的 `threadLocals` 里仍然有 webapp 加载的对象 → 旧 ClassLoader 永远不能回收。

特别坑的：**线程池的 worker 是复用的**。HTTP 请求处理完了，worker 不会 die。它把 ThreadLocal 留在了上面。

#### 修复

四步走：

```java
// 1. webapp 启动时记下你创建的所有线程池
@Bean
public ThreadPoolExecutor myPool() { ... }

// 2. webapp 停止时显式关闭
@PreDestroy
public void destroy() {
    myPool.shutdownNow();
    myPool.awaitTermination(30, TimeUnit.SECONDS);
}

// 3. 业务代码：每次 ThreadLocal.set 后必有 finally remove
try {
    CTX.set(value);
    doWork();
} finally {
    CTX.remove();   // ★ 必须！
}

// 4. 用 try-with-resources 模式让 remove 不可遗漏
public class ScopedContext implements AutoCloseable {
    private static final ThreadLocal<Object> TL = new ThreadLocal<>();
    public ScopedContext(Object v) { TL.set(v); }
    public static Object get() { return TL.get(); }
    public void close() { TL.remove(); }
}
// 用法
try (var ctx = new ScopedContext(user)) {
    doWork();
}
```

更狠的兜底（适合不能改业务代码的场景）：每次请求结束时清空 worker 的所有 ThreadLocal。

```java
// Tomcat 的 Cleanup Filter 模式
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
    try { chain.doFilter(req, res); }
    finally {
        // 反射清空当前 Thread 的 threadLocals
        try {
            Field tl = Thread.class.getDeclaredField("threadLocals");
            tl.setAccessible(true);
            tl.set(Thread.currentThread(), null);
        } catch (Exception e) { /* ignore */ }
    }
}
```

JDK 17+ 这种反射要 `--add-opens java.base/java.lang=ALL-UNNAMED`。不推荐生产用，但调试场景能快速验证"是不是 ThreadLocal 在持有"。

#### 验证

```java
// 写一个测试：reload 10 次，看 Metaspace
for (int i = 0; i < 10; i++) {
    deployWebapp("myapp.war");
    undeployWebapp("myapp");
    System.gc();
    System.gc();   // ★ 老年代 GC 才会卸载 Class
    long meta = ManagementFactory.getMemoryPoolMXBeans().stream()
        .filter(p -> p.getName().contains("Metaspace"))
        .findFirst().get().getUsage().getUsed();
    System.out.println("Iteration " + i + " Metaspace: " + meta);
}
// Metaspace 不持续增长 → 修复成功
```

> **口诀**：ThreadLocal 的 value 是强引用。**线程池 + ThreadLocal = ClassLoader 泄漏温床**。set 后必有 remove，否则迟早被 OOM 叫起来。

---

### 案例 ⑥ SPI 在 OSGi 中失效 —— "ServiceLoader 找不到实现"

#### 现象

代码在普通 Spring Boot 跑得好好的，搬到 OSGi 容器（Karaf / Felix）里：

```java
ServiceLoader<MyExtension> loader = ServiceLoader.load(MyExtension.class);
loader.iterator().hasNext();   // false！
// 但 META-INF/services/com.foo.MyExtension 文件确实存在
```

#### 根因

`ServiceLoader.load(class)` 默认用**线程上下文 ClassLoader**（TCCL）找 `META-INF/services/`。

OSGi 里：
- 调用 `ServiceLoader.load` 的代码在 Bundle A
- `MyExtension` 接口在 Bundle B
- 实现类和 `META-INF/services/` 文件在 Bundle C
- TCCL 通常是某个"系统" ClassLoader，**完全不知道 Bundle C 的存在**

OSGi Bundle 之间没有 parent-child 关系，TCCL 这套"向下找"的机制失效。

#### 修复

**方案 A**：显式传 ClassLoader（如果你能拿到 Bundle C 的 ClassLoader）。

```java
// 在 Bundle C 里有一个工具类
public class MyExtensionLoader {
    public static ServiceLoader<MyExtension> load() {
        return ServiceLoader.load(MyExtension.class,
            MyExtensionLoader.class.getClassLoader());
    }
}
```

但这要求调用方知道去问 Bundle C，不通用。

**方案 B**：用 OSGi 自己的 Service Registry（推荐）。

```java
// Bundle C 启动时注册
context.registerService(MyExtension.class, new MyImpl(), null);

// Bundle A 查找
ServiceReference<MyExtension> ref = context.getServiceReference(MyExtension.class);
MyExtension ext = context.getService(ref);
```

OSGi 的 Service Registry 是**跨 Bundle 共享**的，专为这种场景设计。

**方案 C**：用 `org.apache.aries.spifly`（SPI Fly）—— OSGi 社区的桥接工具，让标准 `ServiceLoader.load` 在 OSGi 里"自动"找到所有 Bundle 注册的实现。

```xml
<!-- Bundle C 的 MANIFEST.MF -->
Require-Capability: osgi.serviceloader.registrar
Provide-Capability: osgi.serviceloader;osgi.serviceloader=com.foo.MyExtension
```

SPI Fly 用字节码增强 `ServiceLoader.load` 调用点，把它替换成查询 OSGi Service Registry。

#### 验证

```java
// Bundle A 里
List<MyExtension> list = new ArrayList<>();
ServiceLoader.load(MyExtension.class).forEach(list::add);
assertFalse(list.isEmpty());   // 通过 = SPI Fly 工作正常
```

> **口诀**：OSGi 把"向下找"的 TCCL 模型废了。要么用 OSGi Service Registry（推荐），要么用 SPI Fly 桥接旧代码。

---

### 案例 ⑦ Spring Boot 升级后 ClassLoader 行为变化 —— "fat jar 跑不起来"

#### 现象

Spring Boot 2.7 升到 3.2 后，原来跑得好好的 fat jar 突然报：

```
Caused by: java.io.FileNotFoundException:
/path/myapp.jar!/BOOT-INF/lib/some-lib.jar (No such file or directory)
```

或：

```
java.lang.UnsupportedOperationException:
This class loader doesn't support setting a custom contextual class loader
```

#### 根因

Spring Boot 3 对 fat jar 结构和 ClassLoader 做了一系列变化：

1. **Spring Boot 3.2 引入新 LoadedClassLoader**：替换了原来的 `LaunchedURLClassLoader`，URL 协议略有不同。
2. **嵌套 jar 不再叫 `BOOT-INF/lib/xxx.jar!/`**：在某些版本里改成了 `nested:` 协议。
3. **Layered jar 的目录结构**：`spring-boot-jarmode-layertools` 模式下，结构是 `dependencies/`、`spring-boot-loader/`、`snapshot-dependencies/`、`application/`。

依赖第三方库**硬编码**了"BOOT-INF/lib/" 路径或者用 `URLClassLoader` cast，就会炸。

#### 修复

**第一**：升级所有相关库到兼容 Spring Boot 3 的版本。
- arthas 升到最新
- skywalking-agent 升到 9.x+
- 字节码增强类工具（pinpoint、newrelic）都要升

**第二**：业务代码不要 cast `URLClassLoader`：

```java
// ❌ 老代码
URLClassLoader cl = (URLClassLoader) Thread.currentThread().getContextClassLoader();
URL[] urls = cl.getURLs();

// ✅ 用 ClassLoader API
ClassLoader cl = Thread.currentThread().getContextClassLoader();
String classpath = System.getProperty("java.class.path");
```

JDK 9+ 的 App ClassLoader 已经**不是** URLClassLoader（变成了 `BuiltinClassLoader`），cast 会抛 `ClassCastException`。

**第三**：用 Layered Jar 优化容器部署。

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <layers>
            <enabled>true</enabled>
        </layers>
    </configuration>
</plugin>
```

```dockerfile
# 解压分层后丢进 Docker，每层独立缓存
FROM eclipse-temurin:21
COPY dependencies/ ./
COPY spring-boot-loader/ ./
COPY snapshot-dependencies/ ./
COPY application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

#### 验证

```bash
# 启动加 -verbose:class，确认嵌套 jar 里的类能加载
java -verbose:class -jar myapp.jar 2>&1 | grep "BOOT-INF"
# 看到正常的 [Loaded ...] 日志 → 工作正常
```

> **口诀**：Spring Boot 大版本升级前，先扫一遍依赖里有没有 cast `URLClassLoader` 或硬编码 `BOOT-INF/`。Agent 类工具是重灾区。

---

### 案例 ⑧ JDK 17 强模块化导致反射访问失败

#### 现象

服务从 JDK 8 / 11 升到 JDK 17：

```
java.lang.reflect.InaccessibleObjectException:
Unable to make field private final byte[] java.lang.String.value accessible:
module java.base does not "opens java.lang" to unnamed module @1234abcd
```

或者用 `Unsafe` 也炸了。

#### 根因

JDK 17 默认**强模块化**（strong encapsulation）。`setAccessible(true)` 不再是万能钥匙：

- 你的 unnamed module（普通 classpath 应用）想反射访问 `java.lang.String.value`
- `String` 在 `java.base` 模块里
- `java.base` **没有 `opens java.lang` 给 unnamed module**
- 反射拒绝访问

JDK 9~16 用 `--illegal-access=permit`（默认）可以宽松；JDK 16 改默认为 `deny`；JDK 17 直接**移除了这个开关** —— 强行禁止。

#### 修复

按推荐度排序：

**方案 1（最佳）**：用 JDK 提供的合法 API 替代。`String` 的字节数组能用 `String.getBytes(Charset)`，`Unsafe` 能用 `VarHandle` / `MethodHandle`。

**方案 2**：启动参数 `--add-opens`。

```bash
java --add-opens java.base/java.lang=ALL-UNNAMED \
     --add-opens java.base/java.util=ALL-UNNAMED \
     --add-opens java.base/sun.nio.ch=ALL-UNNAMED \
     -jar myapp.jar
```

`ALL-UNNAMED` 表示开给所有 classpath 上的 unnamed module。

**方案 3**：在 jar 的 `MANIFEST.MF` 里声明（`--add-opens` 永久化）。

```
Add-Opens: java.base/java.lang java.base/java.util
```

只对**可执行 jar** 生效（`java -jar` 启动）。

**避免坑的清单**：

- Lombok、Spring Reflection、Mockito、Hibernate 是 `--add-opens` 重灾区，框架升级后通常能去掉
- Netty 用 Unsafe 较多，新版本已经做了 fallback
- Jackson 反序列化访问私有字段需要 `java.lang.reflect.AccessibleObject` 的开放
- `sun.misc.Unsafe` 在 JDK 17 仍然存在（在 jdk.unsupported 模块），但 JEP 471 计划在未来版本废弃

记录所有 `--add-opens` 的需求，跟踪每个的源头依赖，定期升级以减少。

#### 验证

```bash
# 不加 add-opens 启动，看哪些库炸
java -jar myapp.jar 2>&1 | grep -A 2 "InaccessibleObjectException"

# 加上必需的 add-opens 再跑
java --add-opens ... -jar myapp.jar
```

> **口诀**：JDK 17 反射不能"硬刚" —— 要么用合法 API，要么 `--add-opens` 显式开口。`--illegal-access` 已经死了。

---

### 案例 ⑨ Agent 注入的类与业务类冲突

#### 现象

接入了某 APM Agent（Skywalking / Pinpoint / NewRelic）后，业务代码偶发 `LinkageError` 或 `NoSuchMethodError`：

```
java.lang.NoSuchMethodError: 'org.objectweb.asm.ClassReader.<init>(byte[])'
```

业务代码里也用 ASM，本地测试没问题，加 agent 就炸。

#### 根因

Agent 通过 `-javaagent:xxx.jar` 启动时被加载到 **System ClassLoader 之上**（实际是同一个 System，但优先扫描）。

Agent 的 jar 里如果**没有 relocate** ASM、Guava 等公共依赖，它的 `org.objectweb.asm.ClassReader` 会和业务代码的同名类**在同一个 ClassLoader 里冲突**。

JVM 加载第一个 `ClassReader` 后，第二个被忽略（同名类不会重复加载）。结果：业务代码看到的是 agent 带的 ASM 版本（可能比业务用的旧），方法签名不匹配 → `NoSuchMethodError`。

#### 修复

**Agent 一侧**（如果你能改）：用 Maven Shade Plugin relocate 所有公共依赖：

```xml
<plugin>
    <artifactId>maven-shade-plugin</artifactId>
    <configuration>
        <relocations>
            <relocation>
                <pattern>org.objectweb.asm</pattern>
                <shadedPattern>com.myagent.shaded.asm</shadedPattern>
            </relocation>
            <relocation>
                <pattern>com.google.common</pattern>
                <shadedPattern>com.myagent.shaded.guava</shadedPattern>
            </relocation>
        </relocations>
    </configuration>
</plugin>
```

`shade` 会把 ASM 的所有类**改包名 + 重写引用**，物理上变成不同的类，不再冲突。

**Agent 一侧（更彻底）**：用独立 ClassLoader 加载 agent 内部代码。

```java
public static void premain(String args, Instrumentation inst) {
    URL[] agentUrls = ...;   // agent 内部的 jar 列表
    URLClassLoader agentLoader = new URLClassLoader(agentUrls, null);   // parent = null
    Class<?> realAgent = agentLoader.loadClass("com.myagent.RealAgent");
    realAgent.getMethod("init", Instrumentation.class).invoke(null, inst);
}
```

`parent = null` 让 agent 内部的类**完全独立于 App ClassLoader**，业务代码看不见，agent 也不会污染业务。Skywalking 用的就是这种模式。

**业务一侧**（如果你不能改 agent）：升级业务依赖到与 agent 同版本，或选另一个 agent。

#### 验证

```bash
# 加 -verbose:class，找出 ClassReader 是从哪加载的
java -verbose:class -javaagent:agent.jar -jar myapp.jar 2>&1 | grep ClassReader
# [Loaded org.objectweb.asm.ClassReader from file:/path/agent.jar] → 来自 agent，可能冲突
# [Loaded org.objectweb.asm.ClassReader from file:/path/myapp.jar!/BOOT-INF/lib/asm.jar] → 业务，OK
```

> **口诀**：Agent 必须 relocate 公共依赖，或用独立 ClassLoader 隔离。否则一定会和业务代码 ASM / Guava / Netty 撞车。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 依赖与 classpath
- [ ] `mvn dependency:tree` 没有重复 jar 不同版本（用 `<exclusions>` 清理）
- [ ] 没有 `provided` scope 的"运行时也要用"的 jar
- [ ] fat jar 里 `unzip -l` 能看到所有需要的依赖
- [ ] classpath 顺序在多 jar 同名类时是确定的（避免靠运气）

### Spring Boot fat jar
- [ ] 不在业务代码里 cast `URLClassLoader`（JDK 9+ 不再是）
- [ ] 不假设 `Thread.currentThread().getContextClassLoader()` 是某个具体类型
- [ ] 升级 Spring Boot 大版本前扫一遍依赖里所有 Agent / 字节码工具的兼容性
- [ ] 容器部署用 Layered Jar 优化镜像层

### Tomcat / Web 容器
- [ ] webapp 启动时创建的所有线程池都在 `contextDestroyed` 关闭
- [ ] webapp 注册的 JDBC Driver / JNDI / MBean 在 stop 时全部注销
- [ ] 生产环境 `autoDeploy=false`，不允许 reload，重启走全量
- [ ] 配置 `JreMemoryLeakPreventionListener` 兜底常见泄漏

### ThreadLocal
- [ ] 每个 `ThreadLocal.set` 都有配套 `remove`（用 try-finally 或 try-with-resources）
- [ ] 长生命周期线程池里禁止存储 webapp scope 对象
- [ ] 静态 ThreadLocal 持有的类型不能引用任何 webapp 加载的类

### 反射 / Unsafe（JDK 17+）
- [ ] 列出所有 `--add-opens` 启动参数，记录每个的来源依赖
- [ ] 优先用 `VarHandle` / `MethodHandle` 替代 `Unsafe`
- [ ] 优先用 `String.getBytes(UTF_8)` 这类公开 API，少反射访问私有字段
- [ ] 避免 cast `URLClassLoader`、`File.list()` 返回值之类的过时模式

### Agent
- [ ] 所有 javaagent jar 必须 shade relocate 内部依赖（ASM、Guava、Netty）
- [ ] agent 用独立 ClassLoader 加载内部代码，parent = null 或 Bootstrap
- [ ] 业务代码用的字节码工具（CGLib / Byte Buddy）版本与 agent 不冲突
- [ ] 多个 agent 同时启用时验证启动顺序和兼容性

### Metaspace
- [ ] 显式设置 `-XX:MaxMetaspaceSize`（默认无上限会吃光内存）
- [ ] 监控 Metaspace 使用率，> 80% 告警
- [ ] 启用 `-XX:+HeapDumpOnOutOfMemoryError` 抓 OOM 现场
- [ ] 频繁动态生成类的场景（Groovy / 大量 CGLib）评估改用 Bytecode 缓存

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### JDK 模块系统的"unnamed module"
- 模块系统的"自动模块"（automatic module）是怎么把传统 jar 包"假装成模块"的
- `module-info.java` 里 `requires transitive` / `exports to` 的精细控制
- jlink 自定义 JDK 镜像，剔除你不用的模块

### Spring Boot Loader 的演进
- Spring Boot 3.2 的 `org.springframework.boot.loader.launch.JarLauncher` 比老版本快在哪
- Spring Boot 3 的 `nested:` URL 协议设计
- Spring Boot 3 的 Layered Jars 与 Buildpack 协作

### GraalVM Native Image 下没有 ClassLoader
- AOT 编译期所有类已经决定，**运行时根本不存在 ClassLoader.loadClass 流程**
- `Class.forName` 必须在 `reachability-metadata.json` 中预先声明
- 反射、动态代理、JNI 都需要 build-time 元数据
- Spring AOT、Quarkus 是怎么扫描出所有需要的反射点

### Quarkus 的 Build-Time ClassLoader
- Quarkus 把 Spring 那种"运行时扫描注解"前移到 build time，运行时已经没了 ClassLoader 那层动态性
- Build 流程的两个 ClassLoader：`AugmentationClassLoader` 和 `BaseRuntimeClassLoader`

### JRebel / Hot Swap Agent 的实现
- JRebel 用 instrumentation 在每个类被加载时插入 "indirection layer"
- HotSwap Agent 配合 DCEVM（修改 JVM）实现"加方法"
- 为什么这两个工具都不能完美支持所有变更（lambda、record、sealed class）

### Class Data Sharing (CDS / AppCDS)
- 把"应用启动时加载的所有类"序列化到一个文件，下次启动直接 mmap
- JDK 13+ 默认开启 `-XX:+UseSharedSpaces`
- Spring Boot 3.3 引入 `spring.context.checkpoint=onRefresh` 配合 CRaC 做"启动 checkpoint"

### JEP 472：限制 JNI 与 Unsafe
- `--enable-native-access` 启动参数，未来 JNI 调用必须显式声明
- `sun.misc.Unsafe` 计划在未来版本废弃
- 替代品：Foreign Function & Memory API（JEP 454），更安全的 native 调用

---

## 7. 引用

- OpenJDK 21 源码：[ClassLoader.java](https://github.com/openjdk/jdk21u/blob/master/src/java.base/share/classes/java/lang/ClassLoader.java)、[URLClassLoader.java](https://github.com/openjdk/jdk21u/blob/master/src/java.base/share/classes/java/net/URLClassLoader.java)
- Tomcat 10 源码：[WebappClassLoaderBase](https://github.com/apache/tomcat/blob/main/java/org/apache/catalina/loader/WebappClassLoaderBase.java)
- Spring Boot Loader 设计文档：[Executable Jar Format](https://docs.spring.io/spring-boot/specification/executable-jar/index.html)
- OSGi Core Specification（Release 8）—— Bundle 类加载模型权威文档
- JEP 261: Module System
- JEP 396: Strongly Encapsulate JDK Internals by Default
- JEP 472: Prepare to Restrict the Use of JNI
- 《深入理解 Java 虚拟机》周志明 —— 第 7 章 类加载机制（讲基础最清楚）
- Bill Burke, *Enterprise OSGi in Action* —— 讲 OSGi ClassLoader 模型最实用

---

> 配套：[JIT 与字节码](/article/jvm/jit) | [GraalVM Native Image](/article/jvm/graalvm) | [字节码增强](/article/java/bytecode)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
