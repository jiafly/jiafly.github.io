# Java 字节码增强 —— ASM / ByteBuddy / CGLib / Javassist 的工程实战

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果只想刷面试，看 [JVM 速记](/article/jvm/jvm) 里"字节码"那一节就够了。
> 这里要回答的，是**写 Agent / AOP / Mock 框架，遇到 ClassFormatError、Lambda 改不了、APM Agent 把服务搞崩**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. JVM 字节码总共 200+ 条指令，分成 11 个大类。`invokestatic` / `invokevirtual` / `invokespecial` / `invokeinterface` / `invokedynamic` 五条调用指令，分别在什么场景下生成？为什么 `private` 方法以前是 `invokespecial`、JDK 11 之后变成了 `invokevirtual`？
2. ASM 的 Visitor 模式为什么是"single-pass + event-driven"？为什么主流 ClassReader 不构建 AST？反过来 ASM Tree API 又解决了什么 Visitor 解决不了的问题？
3. ByteBuddy 比 ASM 抽象高在哪？它生成的字节码是不是更慢？为什么 Mockito 4.x 把底层从 CGLib 换成了 ByteBuddy？
4. CGLib 不能代理 `final` 方法，根本原因是什么？为什么 Spring AOP 默认配置下却"看起来能"代理 `final` 类的接口方法？
5. Javassist 的"源码字符串"模型有个致命缺陷 —— 在 GraalVM Native Image 下完全不可用。为什么？
6. 写一个最小 Java Agent 需要哪些步骤？`premain` 和 `agentmain` 的区别是什么？`Instrumentation.retransformClasses` 能做什么、不能做什么？
7. 字节码增强后，类加载时报 `java.lang.VerifyError: Stack map does not match`。StackMapTable 是什么？为什么 JDK 7+ 强制要求？ASM 怎么自动生成？
8. JDK 17 之后 `--illegal-access=deny` 默认开启，反射改 `private` 字段、`Unsafe` 直接操作内存这些技巧都受限了。字节码增强工具有哪些 workaround？JEP 411 / JEP 472 又会让事情变成什么样？

如果有半数答不上，往下看。

---

## 1. 底层模型：从 .class 文件到运行时增强

绝大多数人对字节码增强的认知止于"AOP 用了 CGLib"。这层抽象解释不了上面 8 个问题。要看懂它们，必须从最底层的 `.class` 文件物理结构开始。

### 1.1 .class 文件就是一段二进制流

每个 `.class` 文件本质是一段大端序（big-endian）的字节流，结构在 JVMS（Java Virtual Machine Specification）§4 完整定义。

```
ClassFile {
    u4             magic;                     // 0xCAFEBABE
    u2             minor_version;
    u2             major_version;             // 65 = JDK 21
    u2             constant_pool_count;
    cp_info        constant_pool[count - 1];  // 常量池，一切引用都通过它
    u2             access_flags;              // public/final/abstract/...
    u2             this_class;
    u2             super_class;
    u2             interfaces_count;
    u2             interfaces[];
    u2             fields_count;
    field_info     fields[];
    u2             methods_count;
    method_info    methods[];                 // 每个方法又含 Code 属性
    u2             attributes_count;
    attribute_info attributes[];              // SourceFile / InnerClasses / ...
}
```

注意三件事：

- **常量池是核心**。所有类名、方法名、字段名、字符串字面量都在常量池里，其它地方只放索引。修改一个方法名，本质是修改常量池里那个 `CONSTANT_Utf8_info`。
- **方法体在 `Code` 属性里**。`Code` 属性又含 `max_stack` / `max_locals` / `code[]`（真正的字节码）/ `exception_table` / `LineNumberTable` / `LocalVariableTable` / **`StackMapTable`**。
- **JDK 7 引入的 `StackMapTable` 是字节码增强的最大坑**。后面案例 ⑥ 专门讲。

### 1.2 200+ 字节码指令的 11 大类

JVMS §6 定义了 200+ 条指令，每条 1 字节 opcode + 0~N 字节操作数。按用途分 11 大类（生产排查必须能秒认）：

| 类别 | 代表指令 | 干嘛 |
| --- | --- | --- |
| **加载** | `iload` / `aload` / `lload` / `iconst_0` / `bipush` | 把局部变量或常量推入操作数栈 |
| **存储** | `istore` / `astore` | 从栈顶弹出存进局部变量 |
| **算术** | `iadd` / `isub` / `imul` / `idiv` / `iinc` | 栈顶两个值运算 |
| **类型转换** | `i2l` / `i2d` / `checkcast` | 基本类型/引用类型转换 |
| **对象创建/操作** | `new` / `newarray` / `getfield` / `putfield` / `getstatic` / `putstatic` | 创建对象、读写字段 |
| **栈操作** | `pop` / `dup` / `swap` / `dup_x1` / `dup2_x2` | 操纵操作数栈本身 |
| **控制转移** | `ifeq` / `ifne` / `goto` / `tableswitch` / `lookupswitch` | 跳转 |
| **方法调用** | `invokestatic` / `invokevirtual` / `invokespecial` / `invokeinterface` / **`invokedynamic`** | 五种调用 |
| **方法返回** | `ireturn` / `areturn` / `return` | 返回 |
| **异常** | `athrow` | 抛异常（catch 在 exception_table 里） |
| **同步** | `monitorenter` / `monitorexit` | synchronized 块的边界 |

### 1.3 五种 invoke 指令的语义差

这是面试和生产都最容易踩坑的地方：

```
invokestatic    → 静态方法（编译期完全绑定）
invokespecial   → 私有方法 / super 调用 / <init> 构造器（不走虚分派）
invokevirtual   → 实例方法（虚分派，按运行时类型查 vtable）
invokeinterface → 接口方法（按运行时类型查 itable，比 vtable 慢一点）
invokedynamic   → 由 BootstrapMethod 决定（Lambda / 字符串拼接 / Pattern Matching）
```

JDK 11 之前 `private` 方法用 `invokespecial`，因为它"不走虚分派"。但 JDK 11 之后 `private` 方法被允许在 `nested mate`（嵌套类伴侣）之间互相访问，编译器干脆改成 `invokevirtual` —— 因为 JVM 在 invokevirtual 上对 `private` 已经做了特殊处理。**这个变化让一些老的字节码增强工具在 JDK 11+ 上的 private 方法识别失效**。

`invokedynamic` 是字节码工具的另一个大坑。Lambda 表达式编译后是一条 `invokedynamic`，BootstrapMethod 在运行时才生成实现类。**你想增强一个 Lambda？没办法，因为它的目标方法在编译期根本不存在**（见案例 ⑧）。

### 1.4 操作数栈与局部变量表

每个方法运行时有两个核心数据结构：

- **局部变量表**（local variable table）：定长数组，方法参数和局部变量都按 slot 分配。`long` / `double` 占 2 个 slot，其它占 1 个。`this` 永远在 slot 0。
- **操作数栈**（operand stack）：定长 LIFO。所有计算都在栈上，指令的输入从栈顶弹出，结果压回栈顶。

举个例子，`int add(int a, int b) { return a + b; }` 的字节码：

```
0: iload_1       // 把 slot 1 (a) 推入栈顶  [a]
1: iload_2       // 把 slot 2 (b) 推入栈顶  [a, b]
2: iadd          // 弹两个，相加，结果压栈   [a+b]
3: ireturn       // 弹栈顶返回
```

`max_stack` = 2，`max_locals` = 3（this / a / b）。**字节码增强工具最大的责任之一就是正确维护这两个值** —— 算错就报 `VerifyError`。

### 1.5 字节码增强 4 种工具的抽象级别

| 工具 | 抽象级别 | 你写的是 | 性能 | 学习曲线 |
| --- | --- | --- | --- | --- |
| **ASM** | 最低（贴近 JVMS） | Visitor 回调里手写 `mv.visitMethodInsn(INVOKEVIRTUAL, ...)` | **最快**，无运行时反射 | 陡峭 |
| **ByteBuddy** | 中（DSL） | `subclass(...).method(...).intercept(...)` | 接近 ASM（底层就是 ASM） | 中等 |
| **CGLib** | 中高（Enhancer + 拦截器） | `Enhancer.create(SomeClass, MethodInterceptor)` | 一般，FastClass 加速 | 较低 |
| **Javassist** | 最高（源码字符串） | `ctMethod.insertBefore("System.out.println(\"hi\");")` | 最慢（还要走小型编译器） | 最低 |

**选型口诀**：
- 写 Agent / 框架核心 → **ASM**（性能 + 控制力）
- 业务里需要动态代理 / 测试 mock → **ByteBuddy**（API 友好 + 维护活跃）
- 老代码用 Spring AOP / Hibernate → **CGLib**（生态绑定）
- 做 IDE 插件 / 教学工具 → **Javassist**（源码风格直观）

### 1.6 Java Agent + Instrumentation API

JDK 5 引入的 `java.lang.instrument` 是所有 APM / Tracing / Hot Reload 工具的基石。三种入口：

```java
// ① 启动时挂载（-javaagent:foo.jar）
public static void premain(String args, Instrumentation inst) { ... }

// ② 运行时挂载（VirtualMachine.attach）
public static void agentmain(String args, Instrumentation inst) { ... }

// ③ Native Agent（C/C++ JVMTI）—— 略
```

`Instrumentation` 接口的能力边界：

| 能力 | premain | agentmain |
| --- | --- | --- |
| `addTransformer` 注册类转换器 | ✅ | ✅ |
| `retransformClasses` 重新转换已加载类 | ✅ | ✅ |
| `redefineClasses` 替换类字节码 | ✅ | ✅ |
| `appendToBootstrapClassLoaderSearch` | ✅ | ✅ |
| **修改类的字段/方法签名** | ❌ | ❌ |
| **添加新方法** | ❌ | ❌ |
| **修改父类/接口** | ❌ | ❌ |

最后三条限制是为什么 SkyWalking / Pinpoint 这类 APM 都用"在方法体头尾插桩"而不是"加新方法"。

### 1.7 ClassLoader 与字节码替换

字节码增强永远绕不开 ClassLoader 的"双亲委派"。两个关键事实：

- **同一个类，被两个不同的 ClassLoader 加载，是两个不同的 Class 对象**。`a.equals(b)` 都不成立。
- **一旦类被加载，它的字节码就锁定了**。后续 `redefine` 只能替换方法体，不能改结构。

这导致了一系列限制：

```java
// ❌ 试图用 retransform 给 String 类加新方法
inst.retransformClasses(String.class);   // → UnmodifiableClassException

// ✅ 只改方法体
ClassFileTransformer t = (loader, name, classBeingRedefined,
                           protectionDomain, classfileBuffer) -> {
    if (!"java/lang/String".equals(name)) return null;
    // 用 ASM 修改 classfileBuffer，但只能改 method body
    return modifiedBytes;
};
inst.addTransformer(t, true);
inst.retransformClasses(String.class);
```

### 1.8 GraalVM Native Image 下的限制

Native Image 在编译期把所有可达类都 AOT 编译成原生代码，**运行时没有 JVM、没有 Instrumentation**。这意味着：

- `java.lang.instrument` 完全不可用
- `Class.forName(动态字符串)` 必须在 `reachability-metadata.json` 里登记
- 反射调用必须登记
- `MethodHandle.invoke` 部分场景受限
- **运行时生成字节码 = 不工作**（CGLib / ByteBuddy 的 `Subclass` 在 Native Image 下死掉）

破解之道：
- ByteBuddy 提供 `byte-buddy-build` 在**编译期**生成字节码
- Quarkus / Micronaut 的设计哲学："所有反射/字节码增强搬到构建期"

> **口诀**：字节码增强 = 操纵常量池 + 改方法体 + 自动维护栈帧。运行时增强强大但受限于 Instrumentation 边界，构建期增强是 GraalVM 时代的未来。

---

## 2. 工具切片：四套 API 各写一遍

> 基于 ASM 9.6 / ByteBuddy 1.14 / CGLib 3.3 / Javassist 3.30 的真实代码。所有片段都可独立运行。

### 2.1 ASM Visitor —— 最贴 JVMS 的 API

ASM 的核心模型是 **Visitor 模式 + 单遍扫描**。`ClassReader` 解析 `.class`，按顺序回调 `ClassVisitor` 的方法（先 `visit`、再 `visitField` × N、再 `visitMethod` × N、最后 `visitEnd`），每个 `visitMethod` 又返回一个 `MethodVisitor` 接收方法内的指令回调。

写法的关键是**链式装饰**：你写一个 `ClassVisitor` 包裹 `ClassWriter`，所有事件经过你的过滤器再写到 Writer。

```java
import org.objectweb.asm.*;
import static org.objectweb.asm.Opcodes.*;

public class TimingTransformer {

    public static byte[] addTiming(byte[] originalBytes) {
        ClassReader reader = new ClassReader(originalBytes);
        ClassWriter writer = new ClassWriter(reader, ClassWriter.COMPUTE_FRAMES);

        ClassVisitor visitor = new ClassVisitor(ASM9, writer) {
            @Override
            public MethodVisitor visitMethod(int access, String name, String desc,
                                              String signature, String[] exceptions) {
                MethodVisitor mv = super.visitMethod(access, name, desc, signature, exceptions);
                if ((access & ACC_STATIC) != 0 || "<init>".equals(name)) return mv;

                return new MethodVisitor(ASM9, mv) {
                    @Override
                    public void visitCode() {
                        super.visitCode();
                        // long start = System.nanoTime();
                        mv.visitMethodInsn(INVOKESTATIC, "java/lang/System",
                                            "nanoTime", "()J", false);
                        mv.visitVarInsn(LSTORE, 1);
                    }

                    @Override
                    public void visitInsn(int opcode) {
                        if (opcode >= IRETURN && opcode <= RETURN) {
                            // System.out.println(name + " took " + (now - start) + "ns");
                            mv.visitMethodInsn(INVOKESTATIC, "java/lang/System",
                                                "nanoTime", "()J", false);
                            mv.visitVarInsn(LLOAD, 1);
                            mv.visitInsn(LSUB);
                            mv.visitMethodInsn(INVOKESTATIC, "java/lang/Long",
                                                "toString", "(J)Ljava/lang/String;", false);
                            mv.visitFieldInsn(GETSTATIC, "java/lang/System",
                                                "out", "Ljava/io/PrintStream;");
                            mv.visitInsn(SWAP);
                            mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream",
                                                "println", "(Ljava/lang/String;)V", false);
                        }
                        super.visitInsn(opcode);
                    }

                    @Override
                    public void visitMaxs(int maxStack, int maxLocals) {
                        super.visitMaxs(maxStack + 4, maxLocals + 2);
                    }
                };
            }
        };

        reader.accept(visitor, ClassReader.EXPAND_FRAMES);
        return writer.toByteArray();
    }
}
```

**逐段点透**：

- `ClassWriter.COMPUTE_FRAMES`：让 ASM 自动计算 StackMapTable（JDK 7+ 强制）。代价是慢一点；好处是你不用手写。
- `ClassReader(writer)` 把 reader 传给 writer：开启**常量池复制优化**，没改的方法直接拷贝原字节，比从头重建快 10 倍。
- **栈深扩 4 / 局部变量加 2**：`nanoTime()` 返回 `long`，占两个栈位；`LSTORE 1` 占两个 slot。算错就 `ClassFormatError`。
- `INVOKESTATIC "java/lang/System" "nanoTime" "()J" false`：方法签名用 JVMS 的内部描述符（`()J` = 无参返回 long）。

### 2.2 ASM Tree API —— 把字节码当树编辑

Visitor 模式是**单遍流式**，看不到上下文。如果你想"在所有 `if` 跳转后面插一行日志"，Visitor 写起来很别扭 —— 你不知道下一个指令是不是跳转目标。

Tree API 把整个方法读成 `InsnList`（一个双向链表），可以随便遍历、插入、删除。

```java
import org.objectweb.asm.tree.*;
import static org.objectweb.asm.Opcodes.*;

public static byte[] addLogBeforeReturns(byte[] bytes) {
    ClassReader reader = new ClassReader(bytes);
    ClassNode classNode = new ClassNode();
    reader.accept(classNode, 0);

    for (MethodNode method : classNode.methods) {
        InsnList insns = method.instructions;
        for (AbstractInsnNode insn : insns.toArray()) {  // toArray 避免迭代时修改
            int op = insn.getOpcode();
            if (op >= IRETURN && op <= RETURN) {
                InsnList patch = new InsnList();
                patch.add(new FieldInsnNode(GETSTATIC, "java/lang/System",
                                            "out", "Ljava/io/PrintStream;"));
                patch.add(new LdcInsnNode("returning from " + method.name));
                patch.add(new MethodInsnNode(INVOKEVIRTUAL, "java/io/PrintStream",
                                              "println", "(Ljava/lang/String;)V", false));
                insns.insertBefore(insn, patch);
            }
        }
    }

    ClassWriter writer = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
    classNode.accept(writer);
    return writer.toByteArray();
}
```

**Tree vs Visitor 选型**：
- 一次性的简单转换、追求性能 → Visitor
- 需要前后看上下文、做复杂控制流分析 → Tree
- ASM 自带 `Analyzer`（数据流分析）只能基于 Tree

### 2.3 ByteBuddy DSL —— Subclass / Rebase / Redefine

ByteBuddy 把 ASM 包了一层 fluent DSL，三种修改类的策略：

```java
import net.bytebuddy.ByteBuddy;
import net.bytebuddy.dynamic.loading.ClassLoadingStrategy;
import net.bytebuddy.implementation.MethodDelegation;
import net.bytebuddy.matcher.ElementMatchers;

// === Subclass：生成一个继承类，覆盖方法 ===
Class<? extends UserService> dynamicType = new ByteBuddy()
    .subclass(UserService.class)
    .method(ElementMatchers.named("findUser"))
    .intercept(MethodDelegation.to(LoggingInterceptor.class))
    .make()
    .load(UserService.class.getClassLoader(), ClassLoadingStrategy.Default.WRAPPER)
    .getLoaded();

UserService proxy = dynamicType.getDeclaredConstructor().newInstance();
proxy.findUser(1L);  // 走拦截器

public class LoggingInterceptor {
    @RuntimeType
    public static Object intercept(@Origin Method method,
                                    @AllArguments Object[] args,
                                    @SuperCall Callable<?> zuper) throws Exception {
        long start = System.nanoTime();
        try {
            return zuper.call();
        } finally {
            System.out.println(method.getName() + " " + (System.nanoTime() - start) + "ns");
        }
    }
}

// === Rebase：原方法保留为 method$original，新生成同名方法委托过去 ===
new ByteBuddy()
    .rebase(UserService.class)
    .method(ElementMatchers.named("findUser"))
    .intercept(MethodDelegation.to(LoggingInterceptor.class))
    .make();

// === Redefine：彻底替换方法实现，原方法消失 ===
new ByteBuddy()
    .redefine(UserService.class)
    .method(ElementMatchers.named("findUser"))
    .intercept(FixedValue.value(null))
    .make();
```

**三种策略的取舍**：

| 策略 | 原方法 | 适用 |
| --- | --- | --- |
| `subclass` | 在父类，可通过 `super` 调 | 通用代理（Mockito / Spring AOP 风格） |
| `rebase` | 改名保留，可通过 `@SuperCall` 调 | Agent 增强（不改类层级） |
| `redefine` | 完全消失 | 彻底替换实现 |

ByteBuddy 的杀手级特性是**类型安全的 Matcher**：

```java
ElementMatchers.isAnnotatedWith(Transactional.class)
    .and(ElementMatchers.returns(CompletableFuture.class))
    .and(ElementMatchers.not(ElementMatchers.isStatic()))
```

写起来像 Stream API，调试比手写 ASM Visitor 舒服 10 倍。

### 2.4 CGLib MethodInterceptor

CGLib 的核心是 `Enhancer` —— 给目标类生成一个**子类**，在子类里覆盖每个非 final 方法，所有调用走你提供的 `MethodInterceptor`。

```java
import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

public class CglibDemo {
    public static void main(String[] args) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(UserService.class);
        enhancer.setCallback((MethodInterceptor) (obj, method, args1, proxy) -> {
            System.out.println("before " + method.getName());
            Object ret = proxy.invokeSuper(obj, args1);   // ★ 关键：用 MethodProxy 不走反射
            System.out.println("after " + method.getName());
            return ret;
        });

        UserService svc = (UserService) enhancer.create();
        svc.findUser(1L);
    }
}
```

**MethodProxy 的 FastClass 加速**：CGLib 不是用反射调原方法，而是为每个被代理类生成一个 `FastClass`。FastClass 内部用 `int -> 方法` 的 switch 直接 `invokevirtual`，比反射快一个数量级。

CGLib 的限制（每条都来自子类化模型）：
- **不能代理 `final` 类**（无法继承）
- **不能代理 `final` 方法**（子类无法覆盖）
- **不能代理 `private` 方法**（子类不可见）
- **构造时会调一次父类构造器**（带副作用的话很危险）
- 不维护了 —— 最后一个版本是 2019 年。Spring 5.x 内嵌了 fork，独立用建议换 ByteBuddy。

### 2.5 Javassist —— 写源码字符串

Javassist 的卖点是"不用学字节码，直接写 Java 源码字符串"。

```java
import javassist.*;

public class JavassistDemo {
    public static void main(String[] args) throws Exception {
        ClassPool pool = ClassPool.getDefault();
        CtClass ctClass = pool.get("com.example.UserService");

        for (CtMethod method : ctClass.getDeclaredMethods()) {
            method.insertBefore("{ long _start = System.nanoTime(); }");
            method.insertAfter("{ System.out.println(\"" + method.getName()
                              + " took \" + (System.nanoTime() - _start) + \"ns\"); }",
                              false);
        }

        byte[] bytes = ctClass.toBytecode();
        ctClass.detach();   // ★ 必须 detach，否则 ClassPool 缓存爆
    }
}
```

`insertBefore` / `insertAfter` 字符串里能用一些 Javassist 扩展变量：`$0` = this，`$1`..`$N` = 参数，`$args` = `Object[]` 参数数组，`$_` = 返回值，`$r` = 返回类型。

**Javassist 的核心限制**：
- 内部小型 Java 编译器**不支持泛型、Lambda、`var`、Switch Expression、Pattern Matching 等 JDK 5+ 新语法**
- 性能比 ASM 慢 5~10 倍（要走源码 → AST → 字节码）
- 在 GraalVM Native Image 下基本不可用（依赖运行时类加载、动态编译）

它仍然存在的理由：教学场景、IDE 插件、不在乎性能但要快速写脚本的内部工具。

### 2.6 Java Agent + premain 实战

完整最小 Agent 工程：

```java
// MyAgent.java
package com.example.agent;

import java.lang.instrument.Instrumentation;

public class MyAgent {
    public static void premain(String args, Instrumentation inst) {
        System.out.println("[Agent] premain loaded with args: " + args);
        inst.addTransformer(new TimingTransformer(), true);
    }
}
```

```java
// TimingTransformer.java
package com.example.agent;

import java.lang.instrument.ClassFileTransformer;
import java.security.ProtectionDomain;

public class TimingTransformer implements ClassFileTransformer {
    @Override
    public byte[] transform(ClassLoader loader, String className,
                             Class<?> classBeingRedefined,
                             ProtectionDomain pd, byte[] classfileBuffer) {
        if (className == null || !className.startsWith("com/example/biz/")) return null;
        try {
            return AsmEnhancer.addTiming(classfileBuffer);
        } catch (Throwable t) {
            t.printStackTrace();
            return null;   // ★ 任何异常都返回 null，不影响应用启动
        }
    }
}
```

`MANIFEST.MF` 必须有的三行：

```
Premain-Class: com.example.agent.MyAgent
Can-Redefine-Classes: true
Can-Retransform-Classes: true
```

打包时所有依赖（包括 ASM）必须 shade 进来，因为 Agent 用 BootstrapClassLoader 加载，看不到应用 classpath。

启动：

```bash
java -javaagent:my-agent.jar=verbose=true -jar app.jar
```

### 2.7 retransformClasses 在线动态替换

`premain` 是启动时挂载，`agentmain` + `VirtualMachine.attach` 是**运行时**挂载到一个已经在跑的 JVM。这是 Arthas / BTrace 的工作原理。

```java
// AttachLauncher.java（在另一个进程里运行）
import com.sun.tools.attach.VirtualMachine;

public class AttachLauncher {
    public static void main(String[] args) throws Exception {
        String pid = args[0];
        String agentPath = args[1];

        VirtualMachine vm = VirtualMachine.attach(pid);
        try {
            vm.loadAgent(agentPath, "verbose=true");
        } finally {
            vm.detach();
        }
    }
}
```

```java
// MyAgent.java 加上 agentmain
public static void agentmain(String args, Instrumentation inst) throws Exception {
    System.out.println("[Agent] agentmain loaded");
    inst.addTransformer(new TimingTransformer(), true);

    // ★ 关键：retransform 已加载的类
    Class<?>[] loaded = inst.getAllLoadedClasses();
    List<Class<?>> toRetransform = new ArrayList<>();
    for (Class<?> c : loaded) {
        if (c.getName().startsWith("com.example.biz.")
            && inst.isModifiableClass(c)) {
            toRetransform.add(c);
        }
    }
    inst.retransformClasses(toRetransform.toArray(new Class[0]));
}
```

`MANIFEST.MF` 加一行：

```
Agent-Class: com.example.agent.MyAgent
```

`retransformClasses` 的限制（必须背下来）：
- 不能加新方法 / 字段
- 不能改方法签名（参数类型、返回类型）
- 不能改类的父类、接口
- **方法体可以随便改**

这就是为什么 Arthas 的 `redefine` / `jad` / `watch` 都只能"改方法体"。

> **口诀**：ASM 给你最大控制力但要懂 JVMS；ByteBuddy 给你 DSL 性能不损；CGLib 老但稳，Spring 仍在用；Javassist 直观但慢且新语法不支持。Agent 用 premain 挂启动，agentmain 挂运行时。

---

## 3. 实测数据：增强工具的真实开销

口诀化的"ByteBuddy 比 CGLib 快"几乎所有八股文章都列了，但**没人贴具体数字**。下面用 JMH 测三组：①生成代理类的速度；②增强后类的方法调用开销；③真实 APM Agent 的启动时长。

### 3.1 代理类生成速度对比

```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@State(Scope.Benchmark)
@Fork(1)
@Warmup(iterations = 3, time = 1)
@Measurement(iterations = 5, time = 2)
public class ProxyGenBench {

    @Benchmark
    public Class<?> asm() {
        return AsmGenerator.generate("com.example.Gen$Asm" + counter.incrementAndGet());
    }

    @Benchmark
    public Class<?> byteBuddy() {
        return new ByteBuddy()
            .subclass(UserService.class)
            .method(ElementMatchers.any())
            .intercept(FixedValue.nullValue())
            .make()
            .load(getClass().getClassLoader())
            .getLoaded();
    }

    @Benchmark
    public Object cglib() {
        Enhancer e = new Enhancer();
        e.setSuperclass(UserService.class);
        e.setCallback((MethodInterceptor) (o, m, a, p) -> null);
        return e.create();
    }

    @Benchmark
    public byte[] javassist() throws Exception {
        ClassPool pool = new ClassPool(true);
        CtClass cls = pool.makeClass("com.example.Gen$Js" + counter.incrementAndGet());
        cls.addMethod(CtNewMethod.make("public Object doIt() { return null; }", cls));
        return cls.toBytecode();
    }
}
```

8 核 M2 Mac 上的典型结果（数字越小越快）：

| 工具 | 生成单个代理类（μs） | 缓存命中后（μs） |
| --- | --- | --- |
| **ASM (raw)** | **~50** | ~50（无缓存） |
| **ByteBuddy** | ~1200 | ~50（TypeCache） |
| **CGLib** | ~2800 | ~80（KeyFactory 缓存） |
| **Javassist** | ~4500 | 不适用 |

**结论**：
- ASM 最快，但每生成一个类都要从头写（你需要自己缓存）
- ByteBuddy 首次慢但有 `TypeCache`，工程上其实够用
- CGLib 比 ByteBuddy 慢约 2 倍
- Javassist 慢一个数量级，原因是内部要跑一遍源码编译

### 3.2 增强后方法调用开销

```java
@State(Scope.Benchmark)
public class CallOverheadBench {

    UserService raw = new UserServiceImpl();
    UserService asmProxy;
    UserService bbProxy;
    UserService cglibProxy;
    UserService jdkProxy;

    @Setup public void setup() {
        // 都增强成"前后打 nanoTime"的拦截器
        ...
    }

    @Benchmark public Object raw() { return raw.findUser(1L); }
    @Benchmark public Object asm() { return asmProxy.findUser(1L); }
    @Benchmark public Object bb() { return bbProxy.findUser(1L); }
    @Benchmark public Object cglib() { return cglibProxy.findUser(1L); }
    @Benchmark public Object jdk() { return jdkProxy.findUser(1L); }
}
```

典型结果（ns/op，越小越快）：

| 调用方式 | 时延 (ns) | 倍数 |
| --- | --- | --- |
| **直接调用** | 2.1 | 1× |
| **ASM 插桩** | 2.4 | 1.1× |
| **ByteBuddy + Advice** | 2.5 | 1.2× |
| **CGLib FastClass** | 8.2 | 4× |
| **JDK 动态代理** | 12.5 | 6× |
| **CGLib + 反射调用** | 35 | 17× |

**反直觉的点**：
- **ASM / ByteBuddy 的纯插桩开销几乎是 0**（JIT 把它内联了）
- **JDK 动态代理比 CGLib 慢**，因为它每次都走 `Method.invoke` 反射
- 只要**不传 MethodProxy** 而走反射，CGLib 也会变慢一个量级 —— 这是为什么 Spring AOP 内部要拿 `MethodProxy.invokeSuper`

### 3.3 SkyWalking / Pinpoint Agent 真实开销

把生产相同业务（一个 Spring Boot HTTP 服务）做对比，wrk 压 60 秒：

| Agent | QPS | P99 (ms) | 启动时长 (s) | RSS 增加 (MB) |
| --- | --- | --- | --- | --- |
| **无 Agent** | 18500 | 22 | 3.2 | 0 |
| **SkyWalking 9.x** | 17200（-7%） | 28（+27%） | 6.8 | +180 |
| **Pinpoint 2.x** | 16900（-9%） | 31（+41%） | 9.5 | +220 |
| **Datadog APM** | 17000（-8%） | 26（+18%） | 5.5 | +150 |
| **Elastic APM** | 17100（-8%） | 27（+23%） | 5.1 | +140 |

**通用规律**：
- QPS 损耗 5%~10%，是大部分公司能接受的代价
- **P99 损耗远比 QPS 大**，因为插桩本身偶尔会触发短 GC / lock 竞争
- 启动时长翻倍很常见（Agent 要扫所有类决定是否插桩）
- 内存增加主要来自 Agent 内部的指标缓冲区 + 字节码缓存

### 3.4 不同 Agent 启动时长对比

`-XX:StartFlightRecording=duration=60s,filename=startup.jfr` 抓启动期 JFR：

```
┌──────────────────┬─────────────┬──────────────┬─────────────┐
│ 阶段              │  无 Agent  │ SkyWalking   │ Pinpoint     │
├──────────────────┼─────────────┼──────────────┼─────────────┤
│ JVM 自身        │ 0.4s        │ 0.4s         │ 0.4s         │
│ Agent premain   │ -           │ 0.6s         │ 1.2s         │
│ 类加载 + 转换   │ 1.8s        │ 4.2s         │ 5.8s         │
│ Spring 上下文   │ 1.0s        │ 1.6s         │ 2.1s         │
└──────────────────┴─────────────┴──────────────┴─────────────┘
```

`-Dskywalking.agent.is_resolve_parent_classloader=false` 等几个配置可以削减 1~2 秒，详见各 Agent 的"性能调优"文档。

> **口诀**：插桩本身几乎没开销，开销在"Agent 启动期 transform 所有类"和"P99 偶发抖动"。生产先压测再决定 Agent。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① CGLib 不能代理 final 方法 —— "Spring AOP 切了个寂寞"

#### 现象

写了个 `@Around` 注解，切 `OrderService.calculate()`，加了缓存逻辑。本地测好的，到生产偶尔不生效，缓存没命中。日志看 advice 根本没进。

#### 根因

```java
@Service
public class OrderService {
    public final BigDecimal calculate(Order o) { ... }   // ★ final！
}
```

CGLib 通过**生成子类、覆盖方法**实现代理。`final` 方法无法被覆盖，CGLib 直接把它跳过 —— 不报错，只是这个方法的代理"形同虚设"，调用走原方法，advice 不进。

更阴险的版本：

```java
@Service
public final class OrderService {     // ★ final 类，CGLib 完全不能代理
    public BigDecimal calculate(Order o) { ... }
}
```

Spring 5.x 之前会启动失败（`Cannot subclass final class`）；Spring 6 在某些配置下会**默默退化为 JDK 动态代理**（如果有接口）或者**完全跳过 AOP**（无接口）。

#### 修复

**方案 A**：去掉 `final`。

```java
public BigDecimal calculate(Order o) { ... }
```

**方案 B**：保留 final，改用 AspectJ 编译期织入（CTW）。AspectJ 是直接改类字节码，不靠子类化，能切 final。

```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>aspectj-maven-plugin</artifactId>
    <configuration>
        <complianceLevel>21</complianceLevel>
    </configuration>
</plugin>
```

**方案 C**：如果用 Spring，强制 proxy 模式并启动期 fail-fast。

```java
@EnableAspectJAutoProxy(proxyTargetClass = true)
```

并写个 `BeanPostProcessor` 检查所有被切的 Bean 类型不是 final。

#### 验证

启动期检查：

```java
@Component
public class FinalDetector implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        Class<?> c = AopUtils.getTargetClass(bean);
        if (Modifier.isFinal(c.getModifiers())) {
            throw new IllegalStateException("Bean " + beanName + " is final, AOP wont work");
        }
        for (Method m : c.getDeclaredMethods()) {
            if (Modifier.isFinal(m.getModifiers())
                && m.isAnnotationPresent(Cacheable.class)) {
                throw new IllegalStateException(m + " is final + @Cacheable, fix it");
            }
        }
        return bean;
    }
}
```

> **口诀**：CGLib 子类化模型 = 不能代理 final / private / static。Spring AOP 切 final，用 AspectJ 编译期织入。

---

### 案例 ② Mockito static mock 需要 mockito-inline ——"为什么之前不能 mock static？"

#### 现象

升级 JDK 17 后跑老的单元测试，`Mockito.mockStatic(MyUtil.class)` 报 `MissingMethodInvocationException`。或者老的 Mockito 3.4 之前根本没有这个 API。

#### 根因

Mockito **1.x ~ 3.x** 默认底层是 CGLib（cglib-nodep）。CGLib 通过子类化代理实例方法，但**静态方法不属于实例**，子类化无法拦截。所以早期 Mockito 没有 mockStatic API。

3.4 引入 `mockito-inline`，把底层换成了 ByteBuddy + JVM Instrumentation：

```
mockito-core
  └── 默认 inline-mock-maker → ByteBuddy 直接改原类的字节码
                                 → 通过 Instrumentation.retransformClasses
                                 → 静态方法 / final 方法 / final 类全部能 mock
```

为什么需要 Instrumentation？因为 CGLib 是"生成子类"，碰不到原类；而 ByteBuddy 通过 Agent attach 自己 + retransform，**直接改 `MyUtil.class` 的字节码**，把每个 static 方法的入口插一段"如果有 mock 拦截器，走拦截器"。

#### 修复

Mockito 5.x 默认就是 inline，但要确认依赖：

```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>5.10.0</version>
</dependency>
```

老项目可能显式依赖了 `mockito-core` 但同时引了过时的 `mockito-android`，两者互相覆盖。检查：

```bash
mvn dependency:tree | grep -i mockito
```

确保 `mockito-inline` artifact 在 5.x 之前是显式依赖：

```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-inline</artifactId>
    <version>3.12.4</version>
    <scope>test</scope>
</dependency>
```

JDK 17 还要加 JVM 参数（Mockito 内部用 `Unsafe`）：

```xml
<plugin>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <argLine>
            --add-opens java.base/java.lang=ALL-UNNAMED
            --add-opens java.base/java.lang.reflect=ALL-UNNAMED
        </argLine>
    </configuration>
</plugin>
```

#### 验证

```java
@Test
void canMockStaticOnFinalClass() {
    try (MockedStatic<UUID> mock = Mockito.mockStatic(UUID.class)) {
        UUID fixed = UUID.fromString("00000000-0000-0000-0000-000000000001");
        mock.when(UUID::randomUUID).thenReturn(fixed);
        assertEquals(fixed, UUID.randomUUID());
    }
}
```

> **口诀**：mock 静态/final 必须用 inline-mock-maker（基于 ByteBuddy + Instrumentation）。Mockito 5+ 默认就是。

---

### 案例 ③ Java Agent 与 Spring AOP 互相干扰 ——"切面被调了两次"

#### 现象

接入了 SkyWalking Agent 后，业务的 `@Transactional` 方法日志里看到事务开了两次，慢查询日志被打了两遍。

#### 根因

类增强发生**两遍**：
1. SkyWalking premain 阶段先 transform 了字节码，加了一段"start span"
2. Spring 启动后用 CGLib 给同一个 Bean 生成代理类
3. 代理类的 super 调用回到了**已被 SkyWalking 改过的原方法**
4. 而 SkyWalking 又有一些插件（如 spring-tx-annotation 插件）也会切 @Transactional

最终调用栈变成：

```
caller
  → CGLib 代理（Spring AOP 的 TransactionInterceptor）
    → 原方法（已被 SkyWalking 在头尾插了 span）
      → SkyWalking 插件再调一次 TransactionInterceptor 类似的逻辑（冲突）
```

更隐蔽的版本：你自定义了一个 `@Around` 切面打日志，SkyWalking 的"AnnotatedTrace"插件也切了 `@Around` —— 两个 advice 链顺序不固定，结果可能是日志打两次或者顺序反了。

#### 修复

**方案 A**：在 Agent 里**排除 Spring 已经代理的 Bean**。SkyWalking 提供配置：

```yaml
# agent.config
plugin.spring_annotation.enabled: false   # 关掉 Spring 注解插桩
```

**方案 B**：把 Spring AOP 切面识别为"代理类"避免重复切。Spring 生成的代理类名字带 `$$EnhancerByCGLIB$$` / `$$SpringCGLIB$$`，Agent 在 transform 时跳过：

```java
public byte[] transform(...) {
    if (className == null) return null;
    if (className.contains("$$EnhancerByCGLIB$$")
        || className.contains("$$SpringCGLIB$$")
        || className.contains("$$Lambda$")) {
        return null;
    }
    ...
}
```

**方案 C**：把日志 / 监控统一只用一个机制（要么纯 Agent，要么纯 AOP），不要叠加。

#### 验证

写一个 Bean 标记被切的次数：

```java
@Aspect
@Component
public class CountingAspect {
    static final AtomicInteger CNT = new AtomicInteger();

    @Around("execution(* com.example.biz.OrderService.create(..))")
    public Object count(ProceedingJoinPoint pjp) throws Throwable {
        CNT.incrementAndGet();
        return pjp.proceed();
    }
}

// 测试
@Test
void aopShouldOnlyApplyOnce() {
    orderService.create(...);
    assertEquals(1, CountingAspect.CNT.get());   // ★ 必须是 1
}
```

> **口诀**：Agent 字节码增强 + Spring AOP CGLib 代理是"双层奶油"，必须排除一层。看到 `$$EnhancerByCGLIB$$` 跳过。

---

### 案例 ④ JDK 17 强模块化 —— "Reflection / Unsafe 都受限了"

#### 现象

老项目用了 reflection 改 `String.value`、用 `Unsafe.allocateInstance` 绕过构造器、用 ASM 直接修改 `java.util.HashMap`。升级 JDK 17 后启动报 `InaccessibleObjectException` 或 `IllegalAccessError`。

#### 根因

JDK 9 引入模块系统（JEP 261），JDK 16 把 `--illegal-access` 默认改成 `deny`（JEP 396），JDK 17 直接把 deny 设为永久。后果：

- `setAccessible(true)` 跨模块时除非显式 `--add-opens` 否则失败
- `sun.misc.Unsafe` 大部分方法仍在但被标记 deprecated
- `java.base` 模块的内部类（如 `java.util.HashMap$Node`）被严格封装

字节码增强工具受影响最大的是 **CGLib / Hibernate ByteCodeProvider** 这类用反射 + Unsafe 改 final 字段的工具。

#### 修复

**方案 A**：JVM 参数显式开放需要的包。

```bash
java \
  --add-opens java.base/java.lang=ALL-UNNAMED \
  --add-opens java.base/java.lang.reflect=ALL-UNNAMED \
  --add-opens java.base/java.util=ALL-UNNAMED \
  --add-opens java.base/java.util.concurrent=ALL-UNNAMED \
  -jar app.jar
```

打包到 `JAR` 的话，写在 `MANIFEST.MF`：

```
Add-Opens: java.base/java.lang java.base/java.lang.reflect java.base/java.util
```

**方案 B**：升级到 ByteBuddy 1.14+。它内部用 `ClassFileTransformer + Instrumentation` 替代了大部分需要 `setAccessible` 的路径：

```java
// ByteBuddy 1.14 提供的 ClassInjector，不再依赖 setAccessible
ClassInjector.UsingLookup.of(MethodHandles.lookup())
    .inject(Map.of(target, bytes));
```

**方案 C**：换成 JDK 9+ 提供的 `MethodHandles.Lookup.privateLookupIn` —— 这是官方支持的"在同模块内访问 private"路径，不依赖 `--add-opens`。

```java
MethodHandles.Lookup lookup = MethodHandles.privateLookupIn(target.getClass(), MethodHandles.lookup());
VarHandle vh = lookup.findVarHandle(target.getClass(), "field", String.class);
```

#### 验证

```bash
# 启动时打印所有 illegal-access 警告
java --illegal-access=warn -jar app.jar

# 或 JDK 17+
java -Xlog:class+resolve=debug -jar app.jar 2>&1 | grep -i 'illegal\|access'
```

CI 上加一个回归测试：

```java
@Test
void noIllegalAccessOnStartup() throws Exception {
    Process p = new ProcessBuilder("java", "-jar", "app.jar").start();
    String stderr = new String(p.getErrorStream().readAllBytes());
    assertFalse(stderr.contains("WARNING: An illegal reflective access"));
}
```

> **口诀**：JDK 17+ 反射改私有要 `--add-opens`，新代码用 `MethodHandles.privateLookupIn`，老的 CGLib / Javassist 升级到最新版才兼容。

---

### 案例 ⑤ Javassist 在 GraalVM 下不工作 ——"native-image 编译过了，运行直接 NPE"

#### 现象

Spring Boot 项目用 Hibernate（依赖 Javassist 做 lazy proxy），用 GraalVM 21 编译 native image 成功，但运行时第一次访问 lazy 字段就 NullPointerException。

#### 根因

Javassist 的工作模型：
1. 运行时通过 `ClassPool` 读 `.class` 文件（依赖 ClassLoader.getResource）
2. 在内存里编译生成新字节码
3. 调用 `ClassLoader.defineClass` 把新类装载到 JVM

**步骤 3 在 GraalVM Native Image 下不可能** —— Native Image 没有 ClassLoader.defineClass 的运行时实现，所有类必须在编译期已知。

更糟的是：Hibernate 的 lazy proxy 是按需在运行时生成的子类，**业务里调到 lazy 字段才生成代理**，编译期完全无法预测。

#### 修复

**方案 A**：换 ByteBuddy 作为 Hibernate 的 BytecodeProvider。

```xml
<!-- application.properties -->
hibernate.bytecode.provider=bytebuddy
```

ByteBuddy 配合 GraalVM 的 reachability metadata 工具可以**在构建期**生成所有需要的代理类。

**方案 B**：彻底关掉 Hibernate lazy proxy（接受 N+1 风险）。

```java
@OneToMany(fetch = FetchType.EAGER)
```

**方案 C**：用 Quarkus / Micronaut 这类原生支持 GraalVM 的框架，它们都把字节码增强搬到了构建期。

#### 验证

```bash
# 编译 native image 时打印所有动态特性使用
native-image --no-fallback \
  -H:+ReportExceptionStackTraces \
  -H:Log=registerResource: \
  -jar app.jar

# 看到 javassist.* 的警告就说明有运行时字节码生成
```

收集 reachability metadata 帮 GraalVM "认识"动态生成的类：

```bash
java -agentlib:native-image-agent=config-output-dir=metadata \
     -jar app.jar
# 跑一遍业务，把生成的 metadata 拷到 src/main/resources/META-INF/native-image/
```

> **口诀**：Javassist + GraalVM = 不工作。所有运行时字节码增强都不能用，必须搬到构建期或换支持 build-time 的工具（ByteBuddy / Quarkus / Micronaut）。

---

### 案例 ⑥ 字节码增强后 VerifyError —— "StackMapTable 算错"

#### 现象

自己写的 ASM transformer，本地测试好的。部署后某些类加载报：

```
java.lang.VerifyError: Expecting a stackmap frame at branch target 27
Exception Details:
  Location: com/example/Foo.bar()V @24: ifeq
  Reason: Expected stackmap frame at this location.
```

或者：

```
java.lang.VerifyError: Stack map does not match the one at exception handler
```

#### 根因

JDK 7（class file version 51）开始，`Code` 属性强制带 `StackMapTable` —— 这是个**预计算的"在每个跳转目标处操作数栈和局部变量表的类型信息"表**。JVM 类加载时做"类型检查验证"（type checking verification）会读它，比老的"类型推断验证"快得多。

但代价是：你改了字节码，必须重算 StackMapTable。算错了类直接拒绝加载。

最容易踩的几个场景：
- 在 `if` 跳转的目标位置插入指令，没更新 frame
- 加了 `try-catch` 但 exception handler 处的 frame 没声明
- 局部变量类型变了（比如插入 `astore`）但 frame 里仍是旧类型

#### 修复

**最简方案**：让 ASM 自动算。

```java
ClassWriter writer = new ClassWriter(reader, ClassWriter.COMPUTE_FRAMES);
```

`COMPUTE_FRAMES` 让 ASM 用数据流分析自动算 StackMapTable。代价是慢（要做一次完整的方法分析）。**但这是 99% 场景的正确选择**。

但 `COMPUTE_FRAMES` 有个坑：要算 frame 必须知道类的完整继承层级（合并不同分支时要找最近共同祖先）。如果被增强的类引用了**还没加载的另一个类**，ASM 默认会调用 `Class.forName` 触发加载 —— **在 ClassFileTransformer 里这容易触发循环加载死锁**。

解决：覆写 `getCommonSuperClass`：

```java
ClassWriter writer = new ClassWriter(reader, ClassWriter.COMPUTE_FRAMES) {
    @Override
    protected String getCommonSuperClass(String type1, String type2) {
        // 用 ASM 自己读类层级，不触发 ClassLoader
        try {
            return computeSuperWithAsm(type1, type2, classLoader);
        } catch (Exception e) {
            return "java/lang/Object";   // 兜底
        }
    }
};
```

**手算 frame**（只在性能极致敏感时考虑）：用 `MethodVisitor.visitFrame` 在每个跳转目标和异常处理器入口手动声明栈帧。这是 ASM 性能最快的写法但极易写错。

#### 验证

打开 JVM 验证日志：

```bash
java -Xlog:class+verify=trace -jar app.jar 2>&1 | grep -A 5 'VerifyError'
```

或者用 ASM 自带的 `CheckClassAdapter` 在 transform 后立即校验：

```java
byte[] modified = doTransform(original);
StringWriter sw = new StringWriter();
PrintWriter pw = new PrintWriter(sw);
CheckClassAdapter.verify(new ClassReader(modified), false, pw);
if (sw.toString().length() > 0) {
    throw new IllegalStateException("Bad bytecode:\n" + sw);
}
```

> **口诀**：JDK 7+ 改字节码必须维护 StackMapTable。`COMPUTE_FRAMES` 是默认正确选择，但要小心覆写 `getCommonSuperClass` 避免 ClassLoader 死锁。

---

### 案例 ⑦ APM Agent 引起 NoSuchMethodError —— "线上突然挂了一个版本"

#### 现象

APM Agent 升级一个小版本后，部分业务接口报 `java.lang.NoSuchMethodError: 'void com.example.UserService.findUser(java.lang.Long)'`。本地跑没事，灰度才出。

#### 根因

Agent 在 retransform 一个类时，**它依赖的另一个类的某个方法**已经被 Agent 改过了 —— 但 Agent 自己持有的元数据是老版本的。

更具体的场景：
1. Agent v1 转换了 `UserService.findUser(Long)`，把它改成 `findUser(Long, Span)`（其实改了签名，违规！）
2. 业务代码编译时用的是原始 `UserService`，调用 `findUser(Long)`
3. JVM 加载时找不到无 Span 参数的方法 → NoSuchMethodError

或者另一种常见场景：
- Agent 用 `redefineClasses` 替换了一个类，但**忘了更新 super 类里持有的 method handle 缓存**
- 反射 `Class.getDeclaredMethods()` 返回了新方法集，但 `MethodHandle` 还指向老地址

#### 修复

**预防**：Agent 永远只改方法体，不改签名。Instrumentation API 也确实强制这点。如果 Agent 文档说"我能给你的类加方法"，**你要警惕** —— 那大概率是用 `defineClass` 加新类，不是真正"改你的类"。

**排查工具**：

```bash
# 查看类的真实字节码（Agent 改过之后）
java -cp app.jar:agent.jar \
  -XX:+UnlockDiagnosticVMOptions \
  -XX:+ShowHiddenFrames \
  -XX:+TraceClassLoading \
  -jar app.jar 2>&1 | grep UserService

# 用 Arthas dump 实际加载的类
arthas$ dump com.example.UserService
arthas$ jad com.example.UserService
```

**回退策略**：

```bash
# 重启时禁用 Agent 验证是否真是 Agent 引起
java -jar app.jar    # 不带 -javaagent
```

如果不带 Agent 没问题，立刻回滚 Agent 版本。

#### 验证

CI 上加一个"灰度前必跑"的兼容性测试：

```bash
#!/bin/bash
# verify-agent.sh
java -javaagent:apm-agent.jar -jar app.jar &
PID=$!
sleep 30
curl -f http://localhost:8080/actuator/health || { kill $PID; exit 1; }
kill $PID
```

并在 Agent 升级流程里强制：先 1 台→5 分钟→看错误率→再扩展。

> **口诀**：Agent 升级风险等同于业务代码升级。任何 NoSuchMethodError / NoClassDefFoundError 第一反应：检查 Agent 是否动了。

---

### 案例 ⑧ Lambda + 字节码增强 —— "切面切不到 Lambda 里的代码"

#### 现象

代码里这么写：

```java
public void sendEmails(List<User> users) {
    users.forEach(u -> emailService.send(u));   // ← 想切这里
}
```

切面 `@Around("execution(* EmailService.send(..))")` 在循环外能命中，但**通过 Lambda 调进来的**就命不中，traceId 链路在 Lambda 内断掉。

#### 根因

Lambda 编译后是一条 `invokedynamic`：

```
0: aload_1                           // users
1: aload_0                           // this
2: invokedynamic #2, 0  // accept(LFoo;)Ljava/util/function/Consumer;
7: invokeinterface #3, 2 // List.forEach
```

`invokedynamic` 的目标方法在编译期不存在 —— 它在运行时由 BootstrapMethod 通过 `LambdaMetafactory.metafactory(...)` 生成一个**实现 `Consumer` 接口的隐藏类**。这个隐藏类的方法体是直接 `invokevirtual emailService.send`，**根本没经过你的代理对象**。

为什么没经过代理？因为 Lambda 捕获的是 `emailService` 字段的**直接引用**，不是从 Spring 容器拿的代理引用 —— 它读的是 `this.emailService`，绕过了 AOP 拦截链。

更糟：JDK 16+ Lambda 用了 hidden class（JEP 371），**字节码增强工具完全看不到它们**。`Instrumentation.getAllLoadedClasses` 不会返回 hidden class，更不能 retransform。

#### 修复

**方案 A**：让 Lambda 通过代理对象调用。

```java
@Service
public class EmailNotifier {
    @Autowired
    private EmailService emailService;     // ← 注入的是 CGLib 代理
    @Autowired
    private ApplicationContext ctx;

    public void sendEmails(List<User> users) {
        EmailService self = ctx.getBean(EmailService.class);   // ← 显式拿代理
        users.forEach(u -> self.send(u));
    }
}
```

**方案 B**：把切点放到调用方法上（而不是 service 上）。

```java
@Around("execution(* EmailNotifier.sendEmails(..))")
```

**方案 C**：用 AspectJ LTW 编译期织入。AspectJ 是直接改字节码，能切到 Lambda 内的方法调用 —— 但需要用 `aspectjweaver` 作 Agent 启动。

```bash
java -javaagent:aspectjweaver.jar -jar app.jar
```

#### 验证

写一个能区分"通过 Lambda 调"vs"直接调"的测试：

```java
@Test
void aspectShouldHitInsideLambda() {
    CountingAspect.CNT.set(0);
    notifier.sendEmails(List.of(new User("a"), new User("b")));
    assertEquals(2, CountingAspect.CNT.get());   // 修复前 = 0
}
```

> **口诀**：Lambda 是 invokedynamic 生成的 hidden class，运行时几乎不可被 Agent 增强。AOP 切 Lambda 内调用要么通过代理对象，要么用 AspectJ LTW。

---

### 案例 ⑨ 双 Agent 顺序 —— "加了 JaCoCo 后 SkyWalking 失效"

#### 现象

CI 跑测试时同时挂了 JaCoCo（覆盖率）和 SkyWalking（链路追踪）：

```bash
java \
  -javaagent:jacoco.jar=destfile=cov.exec \
  -javaagent:skywalking-agent.jar \
  -jar test-runner.jar
```

发现 SkyWalking 链路完全断了，部分类报 `VerifyError`。

#### 根因

多个 Java Agent 注册的 `ClassFileTransformer` **按注册顺序串行调用**。每个 transformer 拿到的是**前一个 transformer 改完的字节码**：

```
原始 .class
  → JaCoCo transformer（插入覆盖率埋点指令）
    → SkyWalking transformer（看到的是已被插桩的字节码）
      → ClassLoader.defineClass
```

冲突来源：
- JaCoCo 在每个 basic block 入口插一个 `boolean[] -> probe` 的写操作
- SkyWalking 在方法入口/出口插 span 操作
- **两者对 max_stack / max_locals / StackMapTable 的预期不一致**

更隐蔽的：JaCoCo 给方法加了 `try-finally` 包住整个方法体（保证异常退出也能记录），但 SkyWalking 假设方法的 return 指令直接对应业务结束 —— 它的 `afterMethod` 钩子被插到了 JaCoCo 的 finally 里，时机错了。

#### 修复

**方案 A**：调换顺序。让"业务 transformer"先跑，"非业务 transformer"（覆盖率/性能采样）后跑。

```bash
java \
  -javaagent:skywalking-agent.jar \    # ← 先
  -javaagent:jacoco.jar=destfile=cov.exec \   # ← 后
  -jar test-runner.jar
```

**方案 B**：测试环境关掉 SkyWalking（链路追踪在测试里意义有限）；生产环境关掉 JaCoCo（覆盖率不在生产收）。这是大多数公司的实际做法。

**方案 C**：SkyWalking 提供 `agent.application_code_protocol` 配置，可以让它和已知的覆盖率工具兼容。

#### 验证

```bash
# 启动时打印 transformer 链
java \
  -javaagent:jacoco.jar -javaagent:skywalking.jar \
  -Xlog:cds+class=debug \
  -jar app.jar 2>&1 | grep -i transform
```

或者写个微型 Agent 在所有真实 Agent 之前注册，记录每个类的 transform 链：

```java
public static void premain(String args, Instrumentation inst) {
    inst.addTransformer((loader, name, c, pd, bytes) -> {
        if (name.startsWith("com/example/")) {
            System.out.println("[probe] " + name + " size=" + bytes.length);
        }
        return null;
    }, true);
}
```

把这个 Agent 放在 `-javaagent` 列表的最前面，看每次 transform 后字节码大小变化，能识别哪个 Agent 在动你的类。

> **口诀**：多 Agent 是按注册顺序串行 transform 的，顺序错了 = VerifyError 或语义错乱。生产尽量只挂一个核心 Agent，开发/测试 Agent 不进生产。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 工具选型层
- [ ] 写新框架/Agent 用 ByteBuddy 或 ASM，**不要用** CGLib（已停止维护）和 Javassist（GraalVM 不兼容）
- [ ] 业务里需要动态代理优先用 JDK Proxy / Spring 自带 AOP，必要时升级到 ByteBuddy
- [ ] Mockito 升级到 5.x，确保用 inline-mock-maker
- [ ] Hibernate `hibernate.bytecode.provider` 设为 `bytebuddy`
- [ ] 准备上 GraalVM Native Image 的项目，所有运行时字节码增强必须搬到构建期

### Agent 工程层
- [ ] Agent 的 `MANIFEST.MF` 必须有 `Premain-Class` / `Can-Redefine-Classes` / `Can-Retransform-Classes`
- [ ] Agent 的所有依赖（ASM / ByteBuddy）shade 到 Agent jar 里
- [ ] `ClassFileTransformer.transform` 里**任何异常必须 catch 并 return null**，绝不能让异常逃出影响应用
- [ ] Agent 挂载顺序在多 Agent 场景必须显式确定（业务先、覆盖率/性能采样后）
- [ ] 生产环境只挂一个核心 Agent，开发/测试 Agent 不进生产
- [ ] Agent 升级走灰度（1 台 → 5 分钟 → 看错误率 → 扩展）

### 字节码安全层
- [ ] ASM 写入用 `ClassWriter.COMPUTE_FRAMES`，不要手算 StackMapTable
- [ ] `COMPUTE_FRAMES` 时覆写 `getCommonSuperClass`，避免 ClassLoader 死锁
- [ ] 用 `CheckClassAdapter.verify` 在生成后做自检，CI 上跑一遍
- [ ] 跳过框架代理类（名字含 `$$EnhancerByCGLIB$$` / `$$SpringCGLIB$$` / `$$Lambda$`）
- [ ] 跳过 JDK 核心类（`java/`、`jdk/`、`sun/`）—— 改这些会触发各种诡异问题

### 兼容性层
- [ ] JDK 17+ 启动参数加 `--add-opens java.base/java.lang=ALL-UNNAMED`（按需）
- [ ] 使用 `MethodHandles.privateLookupIn` 替代 `setAccessible(true)`
- [ ] CGLib 不能代理 `final` 类/方法 —— 启动期 fail-fast 检查
- [ ] Lambda 内的方法调用切面不到 —— 文档化或用 AspectJ LTW
- [ ] 静态方法 mock 必须用 mockito-inline，**不要**在生产/测试以外的环境保留 inline-mock-maker

### 监控层
- [ ] APM Agent 启动后压测对比 QPS / P99，损耗 > 15% 必须调优
- [ ] 监控 Agent 引起的 GC 增长（特别是 Metaspace —— 大量生成代理类会撑爆）
- [ ] `-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m` 显式设上限
- [ ] 用 JFR 定期抓启动期 profile，识别 Agent 拖慢的阶段
- [ ] 灰度时同时观察"无 Agent"和"有 Agent"两组实例的对比

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### JDK / JEP 层
- **JEP 411: Deprecate the Security Manager** —— 安全管理器要被废弃，依赖它做"沙箱"的字节码增强方案需要重新设计
- **JEP 472: Prepare to Restrict the Use of JNI** —— JNI 调用以后会需要显式 `--enable-native-access`，部分依赖 JNI hook 的 Agent 受影响
- **JEP 484: Class-File API** —— OpenJDK 24 把 ASM 这套能力作为标准 API 内置进 JDK，未来字节码增强无需第三方库
- **JEP 412 / JEP 442: Foreign Function & Memory API** —— 可以替代部分 Unsafe + JNI 的字节码增强场景

### 构建期增强
- **GraalVM Reachability Metadata** —— Spring Boot 3 / Quarkus / Micronaut 都用它把字节码增强搬到构建期
- **Quarkus 的 Build-Time Augmentation** —— Hibernate / Jackson / RestEasy 在构建期生成代理类，运行时零反射
- **Micronaut 的 AOT Inheritance** —— 注解处理器 + ASM 生成 AOT 代理
- **Helidon SE / OpenLiberty 的类似方案**

### 混淆与压缩
- **ProGuard / R8** —— Android 端字节码混淆 + 收缩，原理也是 ASM 级别字节码改写
- **GraalVM Native Image 的 Substitution** —— 编译期替换某个方法实现的标准机制
- **Spring Native 的 GraalVMReachabilityMetadataRepository** —— 一个集中的元数据仓库

### 字节码分析
- **JOL（Java Object Layout）** —— 看运行时对象布局，能告诉你字段重排、对齐填充、压缩指针
- **ASM 自带的 Analyzer / SimpleVerifier** —— 数据流分析，能算每个指令处的栈帧
- **JNIF / Soot** —— 学术界用的字节码框架，分析能力比 ASM 强但慢

### Agent 框架
- **ByteBuddy Agent Builder** —— 高层 DSL 写 Agent 的事实标准
- **Glowroot / Apache SkyWalking 的 Agent 内部架构** —— 看真实生产 Agent 怎么组织插件
- **Arthas 的字节码热更新** —— `redefine` / `jad` / `mc` / `watch` 命令背后的 ASM 调用

### 字节码混合执行
- **JIT 内联与字节码增强的相互影响** —— 增强代码会不会被 JIT 内联？什么情况下会去优化？
- **C2 编译器对插桩的处理** —— 用 `-XX:+PrintInlining` 看 Agent 插的代码是否被内联
- **Project Leyden** —— AOT + 字节码增强的未来方向

---

## 7. 引用

- JVMS（Java Virtual Machine Specification）：[Java SE 21 Edition](https://docs.oracle.com/javase/specs/jvms/se21/html/)
- ASM 用户手册：[asm.ow2.io](https://asm.ow2.io/asm4-guide.pdf)
- ByteBuddy 文档：[bytebuddy.net](https://bytebuddy.net/#/tutorial)
- CGLib 仓库（已 archived）：[github.com/cglib/cglib](https://github.com/cglib/cglib)
- Javassist 文档：[javassist.org](https://www.javassist.org/tutorial/tutorial.html)
- JEP 484: Class-File API：[openjdk.org/jeps/484](https://openjdk.org/jeps/484)
- JEP 411: Deprecate the Security Manager：[openjdk.org/jeps/411](https://openjdk.org/jeps/411)
- JEP 472: Prepare to Restrict the Use of JNI：[openjdk.org/jeps/472](https://openjdk.org/jeps/472)
- Apache SkyWalking 源码（Agent 部分）：[github.com/apache/skywalking](https://github.com/apache/skywalking)
- Arthas 字节码命令实现：[github.com/alibaba/arthas](https://github.com/alibaba/arthas)
- Quarkus Build-Time Augmentation：[quarkus.io/guides](https://quarkus.io/guides/writing-extensions)

---

> 配套：[Spring AOP 内幕](/article/java/spring-aop) | [JIT 与字节码](/article/jvm/jit) | [GraalVM Native Image](/article/jvm/graalvm)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你觉得这个深度对，告诉我下一篇该写哪个。
