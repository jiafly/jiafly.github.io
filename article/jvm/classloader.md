# 类加载机制

> Java 类加载是一个**惰性过程**：用到才加载、加载后做初始化。理解类加载，等于理解了 JVM 启动到运行的"前半段生命周期"。

---

## 1. 一句话八股

> **类的生命周期 = 加载 → 验证 → 准备 → 解析 → 初始化 → 使用 → 卸载**。
> 加载器使用 **双亲委派模型**，从 Bootstrap → ExtClassLoader（JDK9+ Platform）→ AppClassLoader → 自定义。

---

## 2. 类加载的 7 个阶段

```
加载 Loading → 验证 Verification → 准备 Preparation
            ↓
       解析 Resolution（可与初始化交换顺序，运行时绑定）
            ↓
       初始化 Initialization → 使用 → 卸载
```

### 2.1 加载（Loading）
- 通过类全限定名获取 `.class` 字节流；
- 字节流转为方法区运行时数据结构；
- 在堆中生成 `java.lang.Class` 对象，作为方法区的访问入口。

> 字节流来源不限于 `.class` 文件：网络、数据库、JAR、ZIP、动态生成（如 CGLIB、JSP 编译产物）都行。

### 2.2 验证（Verification）
确保字节码安全可执行，避免恶意构造的 class：
- 文件格式（魔数 `0xCAFEBABE`、版本号）；
- 元数据验证（语义合理、有父类等）；
- 字节码验证（控制流、操作数栈类型）；
- 符号引用验证。

### 2.3 准备（Preparation）
- 为 **类变量**（`static`）分配方法区内存，并赋 **零值**；
- `static final` 基本类型/字符串字面量则**直接赋初值**（`ConstantValue` 属性）。

```java
public static int value = 123;        // 准备阶段 value=0，<clinit> 时才赋 123
public static final int FINAL = 123;  // 准备阶段直接 = 123
```

### 2.4 解析（Resolution）
把符号引用（如 `Lcom/foo/Bar;`）转为直接引用（指针、偏移量、句柄）。
**可以延迟到运行时**，配合动态绑定。

### 2.5 初始化（Initialization）
执行 **`<clinit>()` 方法**：
- 编译器自动收集类中所有 `static` 变量的赋值动作和 `static {}` 块；
- 父类的 `<clinit>` 先执行；
- **JVM 保证 `<clinit>` 的线程安全**（多线程并发只会执行一次）。

**触发初始化的 6 种情况**（八股精确背）：
1. `new` / `getstatic` / `putstatic` / `invokestatic`；
2. 反射调用（`Class.forName`）；
3. 子类初始化前先初始化父类；
4. 启动类（含 `main`）；
5. JDK7+ MethodHandle 引用；
6. 接口的默认方法被实现类初始化时。

**不会触发初始化**的"被动引用"：
- 通过子类访问父类的静态字段；
- `Class[]` 数组定义；
- 引用类的 `static final` 常量（编译期已放入常量池）。

---

## 3. 双亲委派模型（Parent Delegation）

```
        ┌─────────────────────────┐
        │  Bootstrap ClassLoader  │   ← C++ 实现，加载 lib/rt.jar 等
        └────────────▲────────────┘
                     │
        ┌────────────┴────────────┐
        │  Extension/Platform CL  │   ← 加载 lib/ext 或 jmods
        └────────────▲────────────┘
                     │
        ┌────────────┴────────────┐
        │  Application ClassLoader│   ← 加载 classpath
        └────────────▲────────────┘
                     │
        ┌────────────┴────────────┐
        │  Custom ClassLoader     │   ← 用户自定义
        └─────────────────────────┘
```

**逻辑（背下来）**：
1. 收到加载请求 → 先委托父加载器；
2. 父加载器找不到 → 自己加载；
3. 这样保证 `java.lang.Object` 这种核心类只会被 Bootstrap 加载，避免重复定义。

源码（`ClassLoader.loadClass`）：
```java
protected Class<?> loadClass(String name, boolean resolve) {
    synchronized (getClassLoadingLock(name)) {
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            try {
                if (parent != null) c = parent.loadClass(name, false);
                else c = findBootstrapClassOrNull(name);
            } catch (ClassNotFoundException e) {}
            if (c == null) c = findClass(name);   // 自定义加载器只需重写这个
        }
        if (resolve) resolveClass(c);
        return c;
    }
}
```

### 3.1 为什么要双亲委派？
- **安全**：用户自定义 `java.lang.String` 不会顶替核心库；
- **唯一性**：同一个类在 JVM 中只有一份 Class 对象。

### 3.2 破坏双亲委派的场景
| 场景 | 怎么破坏 |
| --- | --- |
| **JDBC**（SPI） | 父加载器加载的接口 `Driver`，要调子加载器加载的实现 → `Thread.setContextClassLoader` |
| **Tomcat** | 每个 webapp 有独立 `WebappClassLoader`，先自己加载、找不到再委托父 → 隔离 |
| **OSGi** | 网状加载，按 `Bundle` 模块化 |
| **JDK9 模块系统** | Platform 模块化，分层加载 |

---

## 4. 自定义类加载器示例

```java
public class FileClassLoader extends ClassLoader {
    private final String dir;

    public FileClassLoader(String dir, ClassLoader parent) {
        super(parent);
        this.dir = dir;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            byte[] bytes = Files.readAllBytes(
                Paths.get(dir, name.replace('.', '/') + ".class"));
            return defineClass(name, bytes, 0, bytes.length);
        } catch (IOException e) {
            throw new ClassNotFoundException(name, e);
        }
    }
}
```

### 4.1 类的"相等"判定
**两个类相等的条件**：
1. 全限定名相同；
2. **加载它的 ClassLoader 相同**。

`equals()` / `instanceof` / `isAssignableFrom()` 都受此影响。Tomcat 热部署、热更新失效、`ClassCastException: A cannot be cast to A` 多源于此。

---

## 5. JDK 9 模块系统简述

- 引入 `module-info.java`，模块间显式 `requires` / `exports`；
- 类加载器层次变化：
  - Bootstrap CL：加载 `java.base` 等核心模块；
  - **PlatformClassLoader**（替代 ExtClassLoader）：加载平台模块；
  - AppClassLoader：加载应用模块；
- 不再有 `rt.jar`，改为 `lib/modules` 中的 jmod。

---

## 6. 高频面试题速记

1. **类加载流程？** 加载 → 验证 → 准备 → 解析 → 初始化。
2. **双亲委派的逻辑和好处？** 见第 3 节。
3. **怎么破坏双亲委派？** 重写 `loadClass`、Thread Context ClassLoader、SPI、Tomcat、OSGi。
4. **`<clinit>` 和 `<init>` 区别？** `<clinit>` 类初始化（static），`<init>` 实例初始化（构造）。
5. **静态变量初始化时机？** 准备阶段赋零值，`<clinit>` 阶段赋真实值（`static final` 字面量除外）。
6. **数组的类是谁加载的？** 数组本身不是 ClassLoader 加载，而是 JVM 直接生成；其元素类型由对应 ClassLoader 加载。
7. **为什么 Tomcat 要破坏双亲委派？** 多 webapp 隔离 + 同一容器内多版本依赖共存。

---

## 7. 实战：类加载器排查

```java
System.out.println(MyClass.class.getClassLoader());
System.out.println(MyClass.class.getClassLoader().getParent());
System.out.println(MyClass.class.getClassLoader().getParent().getParent());  // null = Bootstrap
```

线上常见 ClassLoader 异常：

| 异常 | 根因 |
| --- | --- |
| `ClassNotFoundException` | 类没在 classpath，由 `Class.forName` 等抛出 |
| `NoClassDefFoundError` | 类编译时存在，运行时找不到（依赖丢失） |
| `LinkageError` / `ClassCastException A→A` | 同一个类被多个 ClassLoader 加载 |
| `UnsatisfiedLinkError` | Native 方法找不到对应的 dll/so |

---

> 下一步：[Java 内存模型 JMM](/article/jvm/jmm) | [JVM 调优与排查](/article/jvm/tuning)
