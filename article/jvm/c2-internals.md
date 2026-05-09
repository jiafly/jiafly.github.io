# HotSpot C2 编译器内核 —— Sea of Nodes、IR 与优化 Pass

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想知道"JIT 是什么、怎么用 `-XX:+PrintCompilation`"，去看 [JIT 与字节码](/article/jvm/jit) 那篇就够了。
> 这里要回答的，是**当你打开 `PrintIdealGraphFile` 看 C2 的 IR 图、研究为什么自己写的循环没有被向量化、对着 OpenJDK 的 `opto/` 目录读源码**时你需要知道的那部分。

---

## 0. 读这篇之前 —— 8 个自检题

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. C2 的 IR 叫 **Sea of Nodes**（节点之海）。它和传统编译器的 CFG（控制流图）+ SSA（静态单赋值）有什么本质差别？为什么 Cliff Click 1995 年要发明它？
2. 你打开 `-XX:+PrintIdealGraphFile` 看 C2 的中间表示，看到 `Region` 节点和 `Phi` 节点 —— 它们各自的角色是什么？为什么 SSA 一定需要 Phi？
3. **GVN（Global Value Numbering，全局值编号）** 是 C2 最朴素也最重要的优化。两个 `AddI(x, 1)` 节点在 GVN 之后会发生什么？为什么 C2 要做 *Iterative* GVN（IGVN）？
4. C2 把 `for (int i = 0; i < n; i++)` 识别成 **Counted Loop**（计数循环）后能做四类优化：Strip Mining、Unrolling、Range Check Elimination、SuperWord 向量化。说出每一种的触发条件和"会被破坏"的写法。
5. **CHA（Class Hierarchy Analysis，类层级分析）** + **Type Profiling（类型剖析）**：JIT 怎么"看出" `list.add(x)` 实际只调用了一种实现？如果运行时突然出现第二种实现，会发生什么？
6. C2 的 **Speculative Optimization（推测优化）** 会把"99% 不走的分支"编译成 `Uncommon Trap`（罕见路径陷阱）。当真的走到了这条路径，控制流如何"安全降级"回解释器？
7. `Math.sqrt(x)` 在 C2 里不是普通的方法调用，而是被替换成一条 `sqrtsd` 机器指令。这种机制叫什么？它和 `@HotSpotIntrinsicCandidate` 注解的关系是？
8. JEP 317 的 Graal JIT 已经废弃，**JVMCI（JVM Compiler Interface）** 还活着 —— 它给 C2 的替代者留了什么口子？Project Leyden 又是什么角色？

如果有半数答不上，往下看。

---

## 1. C2 编译器架构

绝大多数人对 C2 的认知止于"C2 是 JVM 的 server 编译器，比 C1 慢但优化更激进"。但这层抽象解释不了上面 8 个问题。要看懂它们，必须把 C2 的内部架构、IR、优化 Pass 拆开。

### 1.1 C2 在 HotSpot 中的位置

C2 不是孤立的 —— 它和解释器、C1、CodeCache、CompileBroker 一起协同。一段 Java 代码"从字节码到 CPU 流水线"会触碰这些角色：

```
   .class 字节码
       │
       ▼
   Template Interpreter（模板解释器，JVM 启动时一次性生成的汇编模板）
       │ 调用计数 / 回边计数累积 + profile 收集
       ▼
   C1（client 编译器，快速 + 收集 profile，src/hotspot/share/c1/）
       │ profile 攒够、方法持续 hot
       ▼
   C2（server 编译器，慢但激进，src/hotspot/share/opto/）
       │ 假设破裂时
       ▼
   Deoptimization（反优化，回退到解释器）
```

C2 的代码全在 `src/hotspot/share/opto/` 下（"opto" = "optimizer"）。文件超过 200 个，核心模块：

| 模块 | 作用 | 关键文件 |
| --- | --- | --- |
| Parse（字节码 → IR） | 把字节码翻译成 Sea of Nodes | `parse1.cpp` / `parse2.cpp` / `parse3.cpp` |
| Optimization Pass | 各类图变换 | `phaseX.cpp` / `loopopts.cpp` / `escape.cpp` |
| Code Generation | IR → 机器码 | `output.cpp` / `matcher.cpp` |
| Register Allocation | 寄存器分配（Chaitin 染色） | `chaitin.cpp` / `coalesce.cpp` |
| ad files（架构描述） | 平台相关的指令选择规则 | `cpu/x86/x86_64.ad` / `cpu/aarch64/aarch64.ad` |

### 1.2 编译流程：Bytecode → IR → 优化 → 寄存器分配 → 机器码

C2 处理一个方法的全流程，对应到 `src/hotspot/share/opto/compile.cpp` 的 `Compile::Compile` 构造函数：

```
   Bytecode (Method 对象，Metaspace)
       │ Parse（src/hotspot/share/opto/parse*.cpp）
       │   每条字节码翻译成对应的 Node，串成 IR 图
       ▼
   Sea of Nodes IR（"Ideal" graph）
       │ Optimization Phases（按顺序）
       │   ① IGVN（Iterative Global Value Numbering）
       │   ② Inlining + 再次 Parse
       │   ③ CCP（Conditional Constant Propagation）
       │   ④ Loop Opts（Counted Loop、Unrolling、RCE、SuperWord）
       │   ⑤ Escape Analysis + Scalar Replacement
       │   ⑥ Macro Expansion（new、lock 等"宏节点"展开）
       ▼
   优化后的 IR
       │ Matcher（指令选择，opto/matcher.cpp）
       │   把"Ideal" Node 匹配成平台相关的 Mach Node
       ▼
   MachNode 图
       │ Global Code Motion（全局代码移动，opto/gcm.cpp）
       │   把节点放到具体的"基本块"里
       ▼
   有 BB（Basic Block）的图
       │ Register Allocation（Chaitin 寄存器分配，opto/chaitin.cpp）
       ▼
   带具体寄存器的 MachNode
       │ Output（opto/output.cpp）
       │   按基本块顺序输出机器指令
       ▼
   nmethod（native method，存放在 CodeCache）
```

注意一个反直觉点：**C2 的 IR 里"基本块"是后期才出现的概念**。在 Sea of Nodes 阶段，节点是"自由漂浮"的，只有数据 / 控制依赖边，没有"哪个节点在哪个基本块"的概念。直到 Global Code Motion (GCM) 才决定。

> **口诀**：C2 = Parse 字节码 → 在 IR 上做 N 次图变换 → 选指令 → 选基本块 → 分寄存器 → 输出机器码。每个 Pass 都是图到图的变换。

### 1.3 Sea of Nodes IR 的设计

Cliff Click 1995 年的博士论文提出 Sea of Nodes，核心想法是 **"取消基本块"**。

传统编译器的 IR 长这样（CFG + SSA）：

```
   BB1: x1 = 1
        if (cond) goto BB2 else BB3
   BB2: y1 = x1 + 1; goto BB4
   BB3: y2 = x1 + 2; goto BB4
   BB4: y3 = phi(y1, y2)
        return y3
```

它强调"控制流先行" —— 你先有基本块的边，再在块里塞 SSA 指令。Sea of Nodes 的反转：**没有基本块，所有依赖都是节点之间的边**。

```
       Const(1) ─┐
                 ├─→ AddI ─┐
                 │         ├─→ Phi ─→ Return
       Const(2) ─┤         │
                 ├─→ AddI ─┘
                 │
       Cond ─→ If ─→ IfTrue / IfFalse ─→ Region
```

每个节点表示一个"操作"或"控制点"。**没有"代码顺序"的概念** —— 节点能在什么位置放，由依赖边决定，由 GCM 在最后一刻才决定。

### 1.4 IR 中的三种节点（Control / Data / Memory）

C2 的 Node 按"边的语义"分三大类：

| 类别 | 边的含义 | 典型节点 |
| --- | --- | --- |
| **Control** | 控制流依赖 | `Start` / `Region` / `If` / `IfTrue` / `IfFalse` / `Return` |
| **Data** | 数据依赖（值的流动） | `ConI`（int 常量）/ `AddI` / `LoadI` / `Phi` |
| **Memory** | 内存读写顺序依赖 | `MemBar`（内存屏障）/ `LoadI` / `StoreI` 共享的 `Mem Phi` |

**为什么要把 Memory 单独拎出来？** 因为内存读写有"顺序"约束，但又不像控制流那么硬。比如：

```java
a.x = 1;
b.y = 2;     // 和上一行无关，可以重排
return a.x;  // 必须看到第一行的写
```

C2 用一个独立的 **Memory Edge**（内存边）系统跟踪这种关系。每个 `Store` 节点产生一个新的"内存版本"，每个 `Load` 节点指向它依赖的"内存版本"。多个内存版本在 join point 用 `Mem Phi` 合并。

这套机制让 C2 能做激进的"内存重排"优化：只要内存依赖图没破坏，编译器可以自由调整 Load / Store 的顺序。

### 1.5 Phi 节点与 Region

SSA（Static Single Assignment，静态单赋值）的核心约束是"每个变量只被赋值一次"。但分支 join 时多条路径汇合，需要"合并值" —— 这就是 **Phi 节点**。

```
   if (cond) x = 1; else x = 2;
   use(x);
```

在 SSA 里：

```
   BB1: if (cond) goto BB2 else BB3
   BB2: x1 = 1
   BB3: x2 = 2
   BB4: x3 = phi(x1, x2)   // ← 这就是 Phi
        use(x3)
```

C2 的 Sea of Nodes 里：

```
   If ─┬─→ IfTrue  ─┐
       │           ├─→ Region ─→ ...
       └─→ IfFalse ─┘
                                ▲
                                │ control input
   ConI(1) ─┐                   │
            ├─→ Phi ─────────── ┘
   ConI(2) ─┘
```

**Phi 节点的特殊之处**：它的"输入数量"和它绑定的 `Region` 节点的"前驱数量"必须一致 —— 第 i 个数据输入对应第 i 条控制流路径。

`Region` 节点是"控制流汇合点"。Phi 必须依附于 Region 才有意义。

> **口诀**：Region = 控制流的"汇合点"；Phi = 数据值在汇合点的"合并器"。两者一一对应。

### 1.6 优化 Pass 的串联

C2 的 `Compile::Optimize()` 大致按这个顺序跑（参考 `src/hotspot/share/opto/compile.cpp`）：

```cpp
void Compile::Optimize() {
    igvn.optimize();              // ① 第一轮 IGVN
    inline_incrementally(igvn);   // ② 增量内联，内联完后再次 IGVN
    if (do_escape_analysis()) {
        ConnectionGraph::do_analysis(this, &igvn);  // ③ EA
    }
    if (loop_opts_cnt > 0) {
        for (int i = 0; i < loop_opts_cnt; i++) {
            PhaseIdealLoop::optimize(igvn, ...);     // ④ Loop Opts
            igvn.optimize();
        }
    }
    PhaseCCP ccp(&igvn);
    ccp.do_transform();           // ⑤ CCP
    igvn.optimize();              // ⑥ IGVN 收尾
    macro.expand_macro_nodes();   // ⑦ Macro Expansion
}
```

**逐段点透**：

- 几乎每个 Pass 之间都跑一次 IGVN，把上一轮产生的"明显冗余"清理掉。这就是 *Iterative* 的含义
- 内联和 IGVN **交错进行**：内联完后图变大，可能暴露新的常量折叠 / 死代码消除机会
- Loop Opts 是个大循环，`loop_opts_cnt` 默认 2~3 轮，因为一次循环展开可能暴露下一轮 RCE 的机会
- Macro Expansion 在最后：像 `new` / `lock` 这类高层操作，在前期保留为"宏节点"以方便分析（比如 EA 看到 `new` 才能判断对象逃不逃逸），到最后才展开成具体的内存分配 / CAS 等指令

> **口诀**：IGVN 是"消毒水" —— 每个 Pass 跑完都用它清一遍。优化 Pass 不是单向流水，是"螺旋上升"。

---

## 2. 关键优化 Pass

> 基于 OpenJDK 21 `opto/` 源码。每段切片 < 30 行，下面紧跟"逐段点透"。

### 2.1 Iterative GVN（IGVN）

GVN 的本意：**两个表达式如果产生相同的值，就用同一个节点表示**。Iterative 的含义：节点之间的相等关系会"传染" —— 一次替换可能让另一对节点也变成相等。

`src/hotspot/share/opto/phaseX.cpp` 的核心循环：

```cpp
void PhaseIterGVN::optimize() {
    while (_worklist.size() > 0) {
        Node* n = _worklist.pop();

        // ① Identity：如果 n 等价于自己的某个输入，直接用那个输入替换 n
        Node* i = n->Identity(this);
        if (i != n) { replace_node(n, i); continue; }

        // ② Value：如果 n 的所有输入都是常量，算出 n 的常量值
        const Type* t = n->Value(this);
        if (t->singleton()) { replace_node(n, makecon(t)); continue; }

        // ③ Ideal：节点自己提供的"重写规则"
        Node* k = n->Ideal(this, ...);
        if (k != NULL && k != n) { replace_node(n, k); continue; }

        // ④ Hash：在哈希表里找等价节点
        Node* h = hash_find_insert(n);
        if (h != NULL && h != n) { replace_node(n, h); continue; }
    }
}
```

**逐段点透**：

- `Identity`：处理"代数恒等式" —— `x + 0 = x`、`x * 1 = x`、`x AND x = x`。每种 Node 自己重载 `Identity()` 给出规则
- `Value`：常量折叠的入口。如果 `n` 的所有输入都已知是某个具体常量类型，结果也是常量
- `Ideal`：节点的"局部重写"，比如 `(x + c1) + c2` 可以重写成 `x + (c1 + c2)`
- `Hash`：跨节点的"全局值编号"。两个 `AddI(x, 1)` 哈希值相同 → 用同一个节点

**为什么必须 Iterative？** 因为节点替换会让"用了它的下游节点"重新进 worklist。一次 `x + 0 → x` 的化简，可能让下游 `(x + 0) * 0` 化简成 `0`，再让下下游 `... + 0` 化简……

> **口诀**：IGVN = Identity + Value + Ideal + Hash 四种规则迭代到不动点。这是 C2 最朴素也最重要的优化，每个 Pass 之后都要跑一遍。

### 2.2 Conditional Constant Propagation（CCP）

普通常量传播只看"全局常量"。CCP 还利用 **分支条件** 推导出"在某个分支内某变量是常量"。

`src/hotspot/share/opto/phaseX.cpp` 的 `PhaseCCP`：

```cpp
void PhaseCCP::analyze() {
    // 初始所有节点的 Type 设为 Top（最乐观，"未知但不矛盾"）
    for (each node n) set_type(n, Type::TOP);

    // worklist 算法：每次更新一个节点的 Type，传播到下游
    while (!worklist.empty()) {
        Node* n = worklist.pop();
        const Type* t_old = type(n);
        const Type* t_new = n->Value(this);

        if (t_new != t_old) {
            set_type(n, t_new);
            for (each user of n) worklist.push(user);
        }
    }

    // 完成后用 Type 信息重写 IR
    transform();
}
```

**逐段点透**：

- 算法是"乐观的不动点迭代"：从最乐观的 Top 开始，逐步降到"最保守正确"的 Type
- 关键差异：CCP 利用 `If` 节点 —— 在 `if (x == 5)` 的 true 分支里，`x` 的 Type 收窄成 `IntCon(5)`
- 实践效果：`if (x == 0) return x;` —— 普通 GVN 看不出 `x` 是常量；CCP 直接看出 true 分支里 `return x` 等价于 `return 0`

CCP 比 IGVN 强但也更慢，所以只在"循环优化之后"跑一次。

### 2.3 Escape Analysis（标量替换 / 栈分配 / 锁消除）

EA 的入口在 `src/hotspot/share/opto/escape.cpp`。核心是 **ConnectionGraph**（连接图）：

```cpp
class ConnectionGraph : public ResourceObj {
    enum EscapeState {
        NoEscape,       // 不逃出方法
        ArgEscape,      // 作为参数被传出
        GlobalEscape,   // 全局逃逸（存到堆 / 静态字段 / 返回出去）
    };

    GrowableArray<PointsToNode*> _nodes;

    void compute_escape() {
        build_connection_graph();   // 从 IR 构建图
        find_non_escaped_objects(); // 反向传播逃逸状态
        if (has_non_escaping()) split_unique_types();  // 标量替换
    }
};
```

**逐段点透**：

- EA 是 **方法内**（intraprocedural）的：跨方法的逃逸看不到。所以 inline 必须先发生 —— inline 完整个方法变大，EA 才有更多对象可以分析
- `split_unique_types()` 才是真正"省堆"的步骤：把 NoEscape 对象拆成几个标量字段，由寄存器持有
- HotSpot **没有真正的栈上分配** —— 所有"NoEscape 但无法标量替换"的对象，回退到堆分配。Azul Zing 才有真正的栈分配

EA 最常见的"被破坏"原因：

| 原因 | 例子 |
| --- | --- |
| 隐式 toString | `log.debug("got " + obj)` —— `obj.toString()` 被调用 |
| equals / hashCode | `set.contains(obj)` —— equals 把 obj 暴露 |
| ThreadLocal.set | 显然全局逃逸 |
| 反射 | EA 看不进 reflection 调用 |
| Lambda 捕获 | 捕获的对象可能逃出 |

> **口诀**：EA 是 inline 之后做的"二次省事"。不是省堆 —— 是直接让对象"消失"在标量寄存器里。

### 2.4 Loop Strip Mining + Counted Loop

**Counted Loop**（计数循环）是 C2 对"int 步长 + 简单边界"的循环的特殊建模。识别条件极严：

- 计数器是 **int**（不能是 long！）
- 步长是常量
- 终止条件是简单比较 `i < n` / `i <= n`
- 循环体没有"非纯"操作（比如调用未知方法）

识别成功后，C2 在 IR 里把循环建成 `CountedLoopNode`：

```cpp
// src/hotspot/share/opto/loopnode.hpp
class CountedLoopNode : public BaseCountedLoopNode {
    Node* init_trip();    // 起始值
    Node* limit();        // 终止值
    Node* stride();       // 步长
    // ... 还有"是否被展开过"、"是否被剥离过"等元数据
};
```

**Loop Strip Mining** 是 JDK 10 引入的关键改造（`-XX:+UseCountedLoopSafepoints` 默认开），目的是"既保留 Counted Loop 的速度优势，又能让 GC 进得了 Safepoint"。

```
   原始：for (int i = 0; i < n; i++) { body; }
              ↓ Strip Mining
   外循环：for (int outer = 0; outer < n; outer += STRIDE) {
              内循环：for (int i = outer; i < min(outer+STRIDE, n); i++) {
                  body;     // 内循环走 Counted Loop 的速度优化
              }
              safepoint poll;   // 外循环每轮检查一次
          }
```

`STRIDE` 一般是几千。这样既能享受 Counted Loop 的优化，又把 Safepoint 延迟控制在毫秒级。

> **口诀**：Counted Loop = 速度（无 safepoint poll）+ 风险（GC 卡死）。Strip Mining 折中。

### 2.5 Loop Unrolling / Peeling

**Loop Unrolling**（循环展开）：把循环体复制 N 次，减少跳转和 induction variable 更新的开销。

```java
// 原始：100 次迭代
for (int i = 0; i < 100; i++) a[i] = i * 2;

// 展开 4 倍：25 次迭代
for (int i = 0; i < 100; i += 4) {
    a[i]   = i * 2;
    a[i+1] = (i+1) * 2;
    a[i+2] = (i+2) * 2;
    a[i+3] = (i+3) * 2;
}
```

C2 的展开倍数由 `-XX:LoopUnrollLimit`（默认 60）和"循环体大小"动态决定。注意：**展开是 SuperWord 向量化的前置条件** —— 没展开就没办法识别"4 个连续的 a[i+k] 写"是 SIMD 模式。

**Loop Peeling**（循环剥离）：把循环的"第一次"或"最后一次"迭代剥出来单独执行，目的是简化主循环的边界检查。

```java
// 原始
for (int i = 0; i < n; i++) {
    if (i == 0) { /* 特殊处理 */ }
    else        { /* 通用处理 */ }
}

// Peeling
{ /* 特殊处理 */ }   // i = 0 单独
for (int i = 1; i < n; i++) {
    /* 通用处理 */    // 主循环不用每次判 i == 0
}
```

C2 还有 **Loop Versioning**（循环版本化）：根据运行时的某个条件（比如"数组对齐"、"是否 null"）生成两个版本，前面用一次性的检查决定走哪个。

> **口诀**：Unrolling = 一次干 N 次的活；Peeling = 把异常迭代抽走；Versioning = 同一个循环编两份。三者都为后续优化（向量化、RCE）创造干净的循环结构。

### 2.6 Range Check Elimination（RCE）

Java 的数组访问每次都隐式做边界检查（`a[i]` 会先检查 `0 <= i < a.length`），这是 JLS 强制的。但如果编译器能"证明" `i` 永远在合法范围内，就可以删掉这个检查。

C2 的 RCE 在 `src/hotspot/share/opto/loopopts.cpp::do_range_check`。原理：

```java
for (int i = 0; i < a.length; i++) {
    a[i] = ...;     // i 永远 ∈ [0, a.length)，边界检查可删
}
```

C2 的实现策略叫 **Loop Predication**（循环谓词化）：把"循环里每次迭代都检查一次"变成"循环外检查一次"。

```
   if (a == null || a.length < n) {
       throw NPE / AIOBE;        // 把所有"可能失败"的检查提到循环外
   }
   for (int i = 0; i < n; i++) {
       a[i] = ...;                // 循环里完全没检查
   }
```

**会破坏 RCE 的写法**：
- 用 `long` 当索引（不是 Counted Loop）
- 索引依赖未知函数返回值（`a[f(i)]`）
- 循环里改 `a` 的引用（C2 不知道 a 还指向不指向原来的数组）

> **口诀**：RCE = "把每次迭代的检查搬到循环外一次性做完"。Counted Loop 是它的前提。

### 2.7 Inlining 决策（Profile + CHA）

C2 的内联决策综合了字节码大小、调用频率、调用点的 type profile、CHA 等多个因素。入口在 `src/hotspot/share/opto/bytecodeInfo.cpp::InlineTree::try_to_inline`。

**虚方法的内联**靠 **CHA**（Class Hierarchy Analysis，类层级分析）和 **Type Profile**（类型剖析）：

```cpp
// 简化逻辑
ciMethod* try_to_inline_virtual(ciMethod* callee, ciKlass* receiver_type) {
    // ① CHA：扫描全 JVM 已加载的类
    if (CHA::has_unique_implementation(callee)) {
        return inline_with_dependency_on_cha();   // 唯一实现，直接 devirtualize
    }
    // ② Type Profile：调用点观察到的接收类型分布
    TypeProfile profile = get_type_profile(call_site);
    if (profile.is_monomorphic()) {
        return inline_with_type_check();          // 单态，加 type guard 内联
    }
    if (profile.is_bimorphic()) {
        return inline_two_with_dispatch();        // 双态，编两个 + if-else 分发
    }
    return KEEP_VIRTUAL;                          // 多态，老老实实虚调用
}
```

**逐段点透**：

- CHA 是"全局分析"：JIT 编译瞬间，整个 JVM 里 `callee` 只有一个实现，就用那个实现。代价：**这份编译产物对未来加载的类敏感** —— 后续如果加载了新实现，必须 invalidate
- Type Profile 来自 C1 阶段埋的探针。每个虚调用点记录"曾经的接收类型 Top 3"
- Bimorphic 内联很巧妙：**两个版本都内联进来**，前面用 `klass.cmp` 决定走哪个。比纯虚调用快但代码膨胀

CHA 失败的常见场景：**抽象类有多个实现**。即使运行时 99% 走同一个实现，CHA 也无法 devirtualize（因为"未来"可能调到另一个）。这时只能靠 Type Profile + Speculative。

> **口诀**：CHA = "编译时全局看"；Type Profile = "运行时看自己"。两者都失败时虚调用走 vtable / itable，慢一档。

### 2.8 Speculative Optimization + Deoptimization

C2 的"赌博"哲学：根据 profile 假设"这个分支永不走 / 这个类型永不变 / 这个 null 永不出现"。赌赢了直接没有这段代码；赌输了 **Deoptimization**（反优化）回退到解释器。

`src/hotspot/share/opto/callnode.hpp` 里有 `UncommonTrap` 节点：

```cpp
// IR 里的 UncommonTrap 节点
class UncommonTrapBlobNode : public CallNode {
    enum Reason {
        Reason_null_check,       // 假设非 null 错了
        Reason_class_check,      // 假设类型错了
        Reason_range_check,      // 假设索引在范围内错了
        Reason_unstable_if,      // 假设分支走向错了
        Reason_predicate,        // Loop Predication 假设错了
        // ... 还有 30+ 种
    };
};
```

当 IR 里 C2 决定"这条路径不会走"，它生成一个 `UncommonTrap`：执行到这里就调用 `Deoptimization::uncommon_trap()`，把当前栈帧"吐"出来：

```cpp
// src/hotspot/share/runtime/deoptimization.cpp 简化
JRT_BLOCK_ENTRY(void, Deoptimization::uncommon_trap_inner, ...)
    // ① 标记这份编译产物 "made not entrant"（不再被新调用使用）
    nm->make_not_entrant();

    // ② 重建解释器栈帧（基于编译时记录的 OopMap + DebugInfo）
    rebuild_interpreter_frames(thread);

    // ③ 跳到解释器，从对应字节码继续执行
    set_pc_to_interpreter_entry();
JRT_END
```

**逐段点透**：

- Deopt 不是"丢弃运行" —— 它把当前的 locals 还原到解释器栈帧，从对应 BCI 继续解释执行
- DebugInfo 是 C2 在编译时埋下的"映射表"：哪个机器寄存器对应哪个 Java 局部变量、操作数栈什么状态
- 为什么"made not entrant" 而不是直接删掉？因为可能还有其他线程在这份产物里跑（栈上有它的帧），不能立即回收

频繁 deopt 的代价：每次 deopt 都意味着一次"重新编译"。如果一个方法反复被同样的输入打 deopt，C2 会标记 `dont_compile` 不再编译它（永远解释）。

> **口诀**：Speculative = 赌；Uncommon Trap = 赌输的回退入口；Deopt = 把编译态翻译回解释态的运行时机制。三者构成 C2 "激进优化能 work" 的安全网。

---

## 3. Intrinsics 与平台特化

### 3.1 Intrinsic 的注册（vmIntrinsics）

**Intrinsic**（固有方法）是 JVM 对某些"明知道有更优实现"的方法做的特殊处理：识别到这个方法调用，**不走它的 Java 字节码，而是直接发射一段平台优化过的指令序列**。

注册表在 `src/hotspot/share/classfile/vmIntrinsics.hpp`，是一个超长的宏列表：

```cpp
#define VM_INTRINSICS_DO(do_intrinsic, do_class, do_name, ...) \
    do_intrinsic(_dabs,                  java_lang_Math, abs_name,  double_double_signature, F_S) \
    do_intrinsic(_dsqrt,                 java_lang_Math, sqrt_name, double_double_signature, F_S) \
    do_intrinsic(_arraycopy,             java_lang_System, arraycopy_name, ...) \
    do_intrinsic(_compareAndSetInt,      jdk_internal_misc_Unsafe, ...) \
    do_intrinsic(_indexOf_String,        java_lang_String, indexOf_name, ...) \
    /* ... 几百个 */
```

每个 intrinsic 的"机器码生成"在 `src/hotspot/share/opto/library_call.cpp`：

```cpp
bool LibraryCallKit::inline_math_native(vmIntrinsics::ID id) {
    switch (id) {
        case vmIntrinsics::_dsqrt:
            // 直接发射 sqrtsd 指令而不是调用 Java 实现
            return inline_sqrt_double(argument(0));
        case vmIntrinsics::_dabs:
            return inline_math(argument(0), Op_AbsD);
        // ...
    }
}
```

**`@IntrinsicCandidate` 注解**（旧名 `@HotSpotIntrinsicCandidate`）只是给 JVM 的"提示" —— 实际是否被 intrinsic 化由 vmIntrinsics 表决定，跟注解无关。注解的作用是**告诉维护者"这里如果改 Java 实现，要同步改 intrinsic"**。

### 3.2 String.indexOf / Math.sqrt / CRC32 的 SIMD 加速

最经典的 intrinsic 是字符串搜索。`String.indexOf` 的 Java 实现是朴素的"逐字符比较"，但 C2 把它替换成 SIMD 指令：

```
   x86 上的 indexOf intrinsic 用 SSE4.2 的 pcmpestri 指令
   一次比较 16 个字符，找子串位置
```

`src/hotspot/cpu/x86/macroAssembler_x86.cpp::string_indexof` 里写了几百行手工汇编，分若干种情况：

- needle 长度 = 1 → 用 `pcmpeqb` + `pmovmskb` 一次扫 16 字节
- needle 长度 ≤ 16 → 用 `pcmpestri` 一次匹配
- needle 长度 > 16 → 退化到逐段比较

CRC32 的 intrinsic 用 SSE4.2 的 `crc32q` 指令，单条指令算 8 字节的 CRC，比 Java 实现快 10 倍以上。

`Math.sqrt` 在 x86 上直接是一条 `sqrtsd` 指令；在 ARM 上是 `fsqrt`。**没有任何 Java 字节码被执行** —— C2 在 IR 里就把 `CallStaticJava[Math.sqrt]` 节点替换成 `SqrtD` 节点，最终生成单条机器指令。

### 3.3 Vector API（JEP 489）的 C2 实现

JDK 16 引入的 **Vector API**（孵化中，最新到 JEP 489）让程序员显式写 SIMD 代码：

```java
// 计算两个 float 数组的点积（手写 SIMD）
static final VectorSpecies<Float> SPECIES = FloatVector.SPECIES_PREFERRED;

float dot(float[] a, float[] b) {
    int i = 0;
    var sum = FloatVector.zero(SPECIES);
    int upper = SPECIES.loopBound(a.length);
    for (; i < upper; i += SPECIES.length()) {
        var va = FloatVector.fromArray(SPECIES, a, i);
        var vb = FloatVector.fromArray(SPECIES, b, i);
        sum = va.fma(vb, sum);   // fused multiply-add
    }
    float result = sum.reduceLanes(VectorOperators.ADD);
    for (; i < a.length; i++) result += a[i] * b[i];   // 尾巴
    return result;
}
```

C2 对这些 Vector 操作做 intrinsic：每个 `FloatVector.fma` 调用被替换成对应平台的 SIMD 指令（x86 的 `vfmadd231ps`、ARM 的 `fmla`）。

**为什么不直接靠 SuperWord 自动向量化？** SuperWord 只能识别简单的循环模式，复杂的算法（FFT、矩阵乘、加密）需要程序员主动提示。Vector API 提供"显式 SIMD"的入口。

### 3.4 平台相关的 ad 文件（指令选择）

`src/hotspot/cpu/x86/x86_64.ad` 是平台相关的"架构描述文件"（Architecture Description）。C2 用一种 DSL 描述"什么样的 IR 子图匹配什么机器指令"：

```
// 示例：把 IR 的 AddI(reg, reg) 模式匹配成 x86 的 addl 指令
instruct addI_rReg(rRegI dst, rRegI src, rFlagsReg cr) %{
    match(Set dst (AddI dst src));
    effect(KILL cr);
    format %{ "addl    $dst, $src" %}
    ins_encode %{ __ addl($dst$$Register, $src$$Register); %}
    ins_pipe(ialu_reg_reg);
%}
```

`match` 子句告诉 Matcher（`opto/matcher.cpp`）："如果你看到 IR 里有 `Set dst (AddI dst src)` 这个子图，就用我"。

ad 文件里有上千条这样的规则，覆盖各种指令、各种寻址模式、各种立即数情况。**指令选择本质是树匹配 + 动态规划求最优代价**。

> **口诀**：Intrinsic 是"已知更快的实现"的硬编码；ad 文件是"如何把 IR 选成机器指令"的规则库。两者构成 C2 平台特化的两层。

---

## 4. 实测数据

### 4.1 -XX:+PrintCompilation / +PrintInlining 输出解读

最常用的两个参数（第一个不需要解锁，第二个需要 `+UnlockDiagnosticVMOptions`）：

```bash
java -XX:+PrintCompilation \
     -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining \
     YourApp
```

`PrintCompilation` 一行一条编译事件：

```
   234   12       3       com.example.Foo::add (5 bytes)
   245   13       4       com.example.Foo::add (5 bytes)
   250   12       3       com.example.Foo::add (5 bytes)   made not entrant
```

每列含义：

| 列 | 含义 |
| --- | --- |
| `234` | 启动后毫秒 |
| `12` | 编译任务 ID |
| `3` / `4` | 编译级别（C1 level 3 = full profile / C2 level 4） |
| `made not entrant` | 反优化：这份编译产物不再被新调用使用 |

`PrintInlining` 嵌套显示内联决策：

```
@ 5  java.lang.Math::max (11 bytes)            inline (hot)
@ 9  com.example.Foo::tooBig (245 bytes)       too big
@ 14 com.example.Foo::recursive               recursive call
```

常见的"不内联"原因：`too big` / `hot method too big` / `recursive call` / `not inlineable` / `callee uses too much stack` / `executed < MinInliningThreshold times`。

### 4.2 -XX:+PrintIdealGraphFile 可视化 IR

要看 C2 的 IR 图本体，需要 debug build 的 JVM 加上 `-XX:+PrintIdealGraphFile`：

```bash
# 用 fastdebug 版本的 JDK
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+PrintIdealGraphFile \
     -XX:PrintIdealGraphFile=ig.xml \
     YourApp
```

输出的 `ig.xml` 用 **IdealGraphVisualizer**（IGV，Oracle 开发的 GUI 工具）打开，就能看到每个 Pass 之后的 IR 图。

**经验法则**：

- 节点数从 100 → 50 是 IGVN 在做事
- 看到 `CountedLoop` 节点说明循环识别成功
- 看到 `OuterStripMinedLoop` 说明 Strip Mining 成功
- 看到 `UncommonTrap` 说明 Speculative Optimization 起作用了
- 看到 `VectorBox` / `LoadVector` / `StoreVector` 说明向量化成功

如果生产 JDK 没有 fastdebug，可以用 **JITWatch** 离线分析 `-XX:+LogCompilation` 输出（XML 格式）：

```bash
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+LogCompilation \
     -XX:LogFile=hotspot.log \
     YourApp

# 用 JITWatch 打开 hotspot.log
```

JITWatch 能可视化"哪些方法编译了、走了哪些 Pass、内联决策为什么是这个"。

### 4.3 不同优化 Pass 的实际收益

跑一个简单的 benchmark：500 万次 int 数组求和。

```java
@Benchmark
public long sumArray() {
    long s = 0;
    for (int i = 0; i < a.length; i++) s += a[i];
    return s;
}
```

逐个关掉 Pass 测量（在 JMH 里加 `@Fork(jvmArgsAppend = ...)`）：

| 配置 | 时间（μs） | 倍率 |
| --- | --- | --- |
| 默认（全开） | 38 | 1× |
| `-XX:-UseSuperWord`（关向量化） | 110 | 2.9× |
| `-XX:-UseLoopPredicate`（关 RCE） | 145 | 3.8× |
| `-XX:-OptoLoopUnrolling`（关展开） | 165 | 4.3× |
| 全关 + `-XX:-DoEscapeAnalysis` | 230 | 6.0× |
| `-Xint`（解释器） | 5800 | 152× |

**结论**：单个 Pass 的收益看起来不大，**但叠加之后是数量级**。Sea of Nodes 的设计让这些 Pass 能"互相喂"：unrolling 让 SuperWord 能识别模式，RCE 让循环体没有分支，EA 让对象消失……

### 4.4 Deoptimization 的频率与代价

观察 deopt 的最简单方法：

```bash
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+PrintCompilation \
     -XX:+TraceDeoptimization \
     YourApp 2>&1 | grep "Uncommon trap"
```

输出形如：

```
Uncommon trap occurred in com.example.Foo::process compiler=c2 ...
   reason=class_check action=maybe_recompile
```

如果某个方法反复出现这种日志（每秒几次以上），说明 Speculative 假设和真实运行情况不符。**单次 Deopt 的代价**：

- 进入 `Deoptimization::uncommon_trap_inner`：~1 微秒
- 重建解释器栈帧 + 跳转：~5 微秒
- 之后这个方法在解释器里跑（直到下次重新编译）：每次调用慢 50~100 倍

**生产警戒线**：单 JVM 每秒 deopt 总次数 > 100，就要排查。

### 4.5 C2 vs Graal JIT 在不同 benchmark 的对比

GraalVM 的 JIT 用 Java 写（C2 是 C++），优化策略上有几处差异：

| 维度 | C2 | Graal |
| --- | --- | --- |
| EA | 全有或全无 | **Partial Escape Analysis**（按分支区分） |
| 内联启发 | 字节码大小为主 | **基于调用图重要度** |
| Stream/Lambda | 较弱 | 较强 |
| 启动速度 | 快 | 慢（Graal 自己也要 JIT） |
| 纯计算 | 已成熟 | 持平 |

切到 Graal JIT 的方法（JDK 11~16 自带）：

```bash
java -XX:+UnlockExperimentalVMOptions \
     -XX:+EnableJVMCI -XX:+UseJVMCICompiler \
     -jar app.jar
```

JDK 17+ 不再自带 Graal（JEP 410 移除）。要用 Graal 必须装 GraalVM。

某 Stream pipeline benchmark（字符串处理）：

| 编译器 | ops/s |
| --- | --- |
| C2 | 8.2M |
| Graal | 11.6M |

某纯数值计算（矩阵乘）：

| 编译器 | ops/s |
| --- | --- |
| C2 | 12.4M |
| Graal | 12.1M |

**结论**：Stream / Lambda / 反射密集的代码 Graal 占优；纯数值循环两者持平。**生产替换需谨慎**：Graal 启动慢、堆占用大、稳定性不如 C2。

> **口诀**：Stream 用 Graal，纯计算用 C2，启动敏感都不要换 —— C2 仍是 HotSpot 的默认与压舱石。

---

## 5. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① Loop 没有被 vectorize —— "我明明写了简单循环"

#### 现象

业务代码做 float 数组的 element-wise 乘法，预期能用 AVX 指令一次算 8 个 float，但 perf 看到的还是标量 `mulss` 指令。CPU 利用率被打满，但吞吐远低于 SIMD 应有的水平。

```java
void multiply(float[] a, float[] b, float[] out) {
    for (int i = 0; i < a.length; i++) {
        out[i] = a[i] * b[i];
    }
}
```

#### 根因

C2 的 SuperWord 向量化要求循环必须是 **Counted Loop** + **循环体足够"纯"** + **数组访问连续且对齐**。**任何一处不满足，向量化失败**。

最常见的失败原因（按频率）：

1. **没识别成 Counted Loop**：循环计数器是 long
2. **循环体里调了方法**：比如 `out[i] = Math.abs(a[i] * b[i])` —— `Math.abs` 不是 intrinsic（在某些版本上）
3. **访问步长不是 1**：`for (i = 0; i < n; i += 2)`
4. **数组类型不一致**：`out[i] = (int)(a[i] * b[i])` 涉及类型转换

诊断：打开 `-XX:+TraceSuperWord` 看 C2 给的具体失败原因。

```bash
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+TraceSuperWord -XX:+TraceLoopOpts \
     YourApp 2>&1 | grep -i superword
```

输出形如：`SuperWord::transform_loop: failed because <reason>`。

#### 修复

```java
// ✅ 改造后：纯净的 Counted Loop
void multiply(float[] a, float[] b, float[] out, int n) {
    // 把 length 提到循环外，C2 才好分析
    for (int i = 0; i < n; i++) {
        out[i] = a[i] * b[i];
    }
}
```

如果还是不向量化，**直接用 Vector API**（JEP 489）：

```java
static final VectorSpecies<Float> SPECIES = FloatVector.SPECIES_PREFERRED;

void multiply(float[] a, float[] b, float[] out) {
    int i = 0;
    int upper = SPECIES.loopBound(a.length);
    for (; i < upper; i += SPECIES.length()) {
        var va = FloatVector.fromArray(SPECIES, a, i);
        var vb = FloatVector.fromArray(SPECIES, b, i);
        va.mul(vb).intoArray(out, i);
    }
    for (; i < a.length; i++) out[i] = a[i] * b[i];   // 尾巴
}
```

#### 验证

```bash
# 看是否有 SIMD 指令
perf record -e cycles -j any java -jar app.jar
perf report --stdio | grep -E "(mulps|mulpd|vmul)"

# 或用 PrintAssembly
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly \
     -XX:CompileCommand=print,*::multiply YourApp
```

期望看到 `vmulps`（AVX）/ `mulps`（SSE）这种向量指令而不是 `mulss`（标量）。

---

### 案例 ② Inlining 深度太大导致 CodeCache 满

#### 现象

服务跑了几小时后 CPU 突然飙到 100%，但 QPS 跌到正常的 1/3。日志里有 `CodeCache is full. Compiler has been disabled.` 警告。

#### 根因

`CodeCache` 是存储 nmethod（编译产物）的特殊堆区，默认大小 240MB（JDK 9+）。当它满了，**JIT 编译完全停止**，所有方法回退到解释执行。

激进内联会让这个问题加剧：

- `MaxInlineLevel=15` 默认允许 15 层内联
- 复杂业务（比如 Spring + Stream）一个调用链可能有 20+ 层
- 加上"代码膨胀"：bimorphic 内联 = 一个调用点编译两份

CodeCache 满的常见信号：

```bash
# JFR 看 CodeCache 使用率
jcmd <pid> JFR.start duration=60s filename=jfr.jfr
# 用 JMC 打开，看 CodeCache 区段
```

#### 修复

**方案 A：增大 CodeCache**

```bash
-XX:ReservedCodeCacheSize=512m   # 默认 240m
-XX:InitialCodeCacheSize=64m
-XX:CodeCacheExpansionSize=2m
```

注意：CodeCache 在 metaspace 之外，是独立的进程内存。

**方案 B：限制内联深度**

```bash
-XX:MaxInlineLevel=9        # 从 15 降到 9
-XX:MaxInlineSize=20        # 从 35 降到 20
```

代价是性能小幅下降，换来代码量可控。

**方案 C：开启 CodeCache 段化（Segmented CodeCache，JDK 9+ 默认开）**

```
   Non-Profiled (C2 编译产物)：长期不动
   Profiled (C1 编译产物)：会被升级或丢弃
   Non-NMethod (stub / interpreter)：JVM 自身代码
```

段化后每段独立管理，单段满不影响其他段。

#### 验证

```bash
# 启动时打印 CodeCache 配置
java -XX:+PrintFlagsFinal | grep CodeCache

# 运行时观测
jcmd <pid> Compiler.codecache
```

输出形如：

```
CodeCache: size=245760Kb used=180432Kb max_used=200000Kb free=65328Kb
```

`max_used` 持续接近 `size` 就是危险信号。

---

### 案例 ③ Megamorphic Call Site 退化

#### 现象

业务用接口编程（DDD 风格），一个核心接口有 5+ 实现。压测发现某个调用点的 P99 是其他调用点的 3 倍。

```java
public interface OrderProcessor {
    void process(Order order);
}

// 实现：StandardProcessor / VipProcessor / B2BProcessor / ... 共 5 个

void handle(Order o) {
    OrderProcessor p = processorRegistry.get(o.getType());
    p.process(o);   // 这里的 P99 异常高
}
```

#### 根因

C2 看到调用点的 type profile 有 5 种接收类型 → **megamorphic（多态）** → **不内联**。每次调用都走 vtable / itable 查找。

更坏的是：megamorphic 调用是 **inlining 终止点** —— 它后面的代码优化也跟着丧失。

#### 修复

**方案 A：拆分调用点**

```java
void handle(Order o) {
    switch (o.getType()) {
        case STANDARD -> standardProcessor.process(o);  // 单态调用点
        case VIP      -> vipProcessor.process(o);       // 单态调用点
        case B2B      -> b2bProcessor.process(o);
        // ...
    }
}
```

每个 case 是一个独立的调用点，**type profile 各自统计**，都是 monomorphic。

**方案 B：用 sealed interface（JDK 17+）**

```java
public sealed interface OrderProcessor
    permits StandardProcessor, VipProcessor, B2BProcessor {
    void process(Order o);
}
```

CHA 看到 sealed 就能精确知道实现集合。某些场景能 devirtualize。

**方案 C：Strategy Map → Lambda**

如果实现简单（每个就几行），直接用 `Map<Type, Function<Order, Result>>`，让 lambda 在调用点暴露。

#### 验证

```bash
java -XX:+PrintInlining YourApp 2>&1 | grep -A 1 "OrderProcessor::process"
```

期望从 `not inlineable: virtual call` 变成 `inline (hot)`。

---

### 案例 ④ Speculative Optimization 反复 Deopt

#### 现象

服务上线后 P99 一直在抖。监控显示 CPU 利用率稳定，但某些请求耗时是平均的 50 倍。JFR 看到大量 `JIT Optimization Failed` 事件。

#### 根因

某段代码长这样：

```java
boolean isVip(User u) {
    if (u.tier == null) return false;   // 99.9% 走这里
    return u.tier.equals("VIP");
}
```

C2 根据 profile 看到 99.9% `tier == null`，把 `equals` 那条路径标成 `Uncommon Trap`：

```
   if (u.tier != null) {
       UncommonTrap;     // 走到就 deopt
   }
   return false;
```

第一次 `tier != null` 出现时触发 deopt → 重新编译 → 这次 profile 看到了 0.1% 的 VIP → C2 重新做 speculative，但是阈值算法还是把它当作"罕见" → 再次 deopt → 反复循环。

#### 修复

**方案 A：调整 Reason action 阈值**

```bash
# 提高 deopt 容忍度
-XX:PerMethodTrapLimit=500       # 默认 100，超过才 dont_compile
-XX:PerBytecodeTrapLimit=200     # 默认 4
```

**方案 B：业务层面消除"假罕见路径"**

```java
boolean isVip(User u) {
    String tier = u.tier;
    return "VIP".equals(tier);   // null 用 String.equals 安全处理
}
```

把"两条路径"合成"一条 equals 调用"，没有显式的 null 分支。

**方案 C：让 profile 多见点世面**

启动时跑预热脚本，确保各种 user.tier 值都被见过几次。这样 C2 不会把 0.1% 的路径当作罕见。

#### 验证

```bash
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+TraceDeoptimization YourApp 2>&1 | grep "Uncommon trap" | sort | uniq -c
```

某个方法的 deopt 次数应该 < 10。如果上百次，说明根因没修。

---

### 案例 ⑤ 反射 / Lambda 阻碍 inline

#### 现象

业务用反射做"通用调用框架"，简化代码：

```java
public Object dispatch(String methodName, Object[] args) {
    Method m = findMethod(methodName);
    return m.invoke(target, args);
}
```

P99 比直接调用慢 10 倍，且内联日志显示 `not inlineable`。

#### 根因

`Method.invoke` 内部走 `MethodAccessor`，这层是 JVM 生成的 lambda-like 代码。它的接口签名是 `Object invoke(Object obj, Object[] args)` —— **目标方法的参数被强制装箱进 Object[]**。

C2 看到这个调用点：
- `MethodAccessor.invoke` 是 megamorphic（每个目标方法生成一个 accessor，调用点能看到 N 种）
- 即使内联进来，参数还是 `Object[]`，没法做类型特化

类似问题出现在 Lambda：

```java
list.stream()
    .filter(x -> x.value > 100)
    .map(x -> transform(x))   // ← lambda 在不同流水线里被调用，type profile 混了
    .toList();
```

#### 修复

**方案 A：用 MethodHandle 替代反射**

```java
private static final MethodHandle CACHED_HANDLE = lookup.findVirtual(...);

public Object dispatch(Object[] args) {
    return CACHED_HANDLE.invokeExact(target, args);   // ← invokeExact 没装箱
}
```

`invokeExact` 是签名多态（signature polymorphic），C2 能针对具体签名特化。比 `Method.invoke` 快 5~10 倍。

**方案 B：LambdaMetafactory**

JDK 8+ 用 `LambdaMetafactory.metafactory()` 创建函数式接口实例，性能接近直接调用。Spring 的 `MethodHandles` 工具类就是这么做的。

**方案 C：缓存 + 单态化**

把"通过反射调一个目标方法"的调用点拆开 —— 每个目标方法独立的 accessor 字段：

```java
class FastDispatcher {
    private final MethodHandle h_methodA;
    private final MethodHandle h_methodB;
    // ...
    Object call(String name, Object[] args) {
        return switch (name) {
            case "methodA" -> h_methodA.invokeExact(target, args);
            case "methodB" -> h_methodB.invokeExact(target, args);
            // ...
        };
    }
}
```

#### 验证

```bash
java -XX:+PrintInlining YourApp 2>&1 | grep -i "MethodAccessor\|invokeExact"
```

应该看到 `invokeExact` 被内联到具体目标方法。

---

### 案例 ⑥ JIT 编译队列堵塞

#### 现象

启动后 5 分钟，仍然有大量方法没编译完。jstack 看到 C2 线程一直在跑同一个方法。

#### 根因

某个方法字节码超大（几千行 service.process 上帝方法），C2 编译它可能花几秒。C2 编译线程数默认 `ceil(#CPU * 1.5 / 4)` —— 16 核机器只有 6 个 C2 线程。如果这种"巨方法"多了，编译队列堆积。

加上 C2 是 **单线程编译每个方法**：编译完一个方法之前不释放线程。如果一个方法卡住，1/N 的编译能力就丢了。

#### 修复

**方案 A：增加编译线程**

```bash
-XX:CICompilerCount=8        # 总编译线程数
-XX:CICompilerCountPerCPU=1  # 改用按 CPU 算
```

**方案 B：减小巨方法**

```bash
# 看看有哪些方法编译特别慢
java -XX:+CITime YourApp
```

输出会显示每个方法的编译耗时。前 10 名往往是手写的"上帝方法"。重构它们 —— 拆成 50 行内的小方法，C2 编译速度直接提升一个量级，且每个小方法都能被内联。

**方案 C：限制编译队列长度**

```bash
-XX:CompileThreshold=20000   # 提高编译阈值（默认 10000）
```

只编译真正"超热"的方法，避免低收益方法占用编译资源。

#### 验证

```bash
jcmd <pid> Compiler.queue
```

输出形如：

```
C2 CompileQueue:
  ID  level  method
  234   4    com.example.Foo::process
  235   4    com.example.Bar::handle
```

队列长度持续 > 50 是危险信号。

---

### 案例 ⑦ Tiered Compilation 配置错误

#### 现象

服务部署后 P99 翻倍，但 throughput 看起来正常。监控显示编译次数比之前多 3 倍。

#### 根因

运维"为了启动更快"加了 `-XX:-TieredCompilation`，关闭了分级编译。这意味着：

- 所有方法直接走 C2 编译路径
- C2 阈值是 `-XX:CompileThreshold=10000`
- **没有 C1 收 profile**，C2 只能用最保守的优化

结果：方法启动 → 解释 10000 次（慢） → C2 编译（保守优化，质量不高） → 性能不如分级编译。

#### 修复

**方案 A：恢复分级**

```bash
-XX:+TieredCompilation       # 默认开
-XX:TieredStopAtLevel=4      # 默认 4，全级都开
```

**方案 B：极少数情况下，确实想"只 C1"以加速启动**

```bash
-XX:TieredStopAtLevel=1      # 只到 C1 level 1
```

适用于：短命脚本（启动开销 > 长时间收益）。**不要用于长期服务**。

#### 验证

```bash
java -XX:+PrintFlagsFinal | grep -E "(Tier|Compile)"
```

确认 `TieredCompilation = true`、`TieredStopAtLevel = 4`。

---

### 案例 ⑧ JITWatch / async-profiler 的实战诊断

#### 现象

某接口响应慢，profile 看到 30% 时间花在 `org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept()`。但这是 Spring AOP 的标准代理类，不该这么慢。

#### 根因

排查步骤：

**Step 1：用 async-profiler 抓火焰图**

```bash
./profiler.sh -d 30 -e cpu -f flame.html <pid>
```

火焰图看到调用链：`Controller.method` → `CglibProxy.intercept` → `MethodInterceptor.invoke` → 业务方法。

但是 `intercept` 占了非业务时间的 70%。这不正常 —— Spring 的代理调用应该被 JIT 内联掉。

**Step 2：用 PrintInlining 看 intercept 的内联状态**

```bash
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+PrintInlining \
     YourApp 2>&1 | grep -A 5 "intercept"
```

看到：`callee is too large`。CglibAopProxy 的 intercept 方法字节码 800+ 字节，超过了 `FreqInlineSize=325`。

**Step 3：用 JITWatch 离线分析**

```bash
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+LogCompilation -XX:LogFile=hotspot.log \
     YourApp

# 用 JITWatch GUI 打开 hotspot.log
```

JITWatch 显示：`intercept` 没被内联 → 后面的"业务方法"也成了 megamorphic（因为 CglibProxy 把它们都"代理"了，调用点的 type profile 混了几十种）。

#### 修复

**方案 A：换成 JDK Dynamic Proxy + Lambda**

```java
@Configuration
public class AopConfig {
    @Bean
    public AspectJAutoProxyCreator proxyCreator() {
        AspectJAutoProxyCreator creator = new AspectJAutoProxyCreator();
        creator.setProxyTargetClass(false);   // 用 JDK proxy 而非 CGlib
        return creator;
    }
}
```

JDK Proxy 的 invocation handler 调用路径短，更容易被内联。

**方案 B：减少 AOP 嵌套**

业务代码里少用 `@Transactional` + `@Cacheable` + `@PreAuthorize` 三连。每个 AOP advice 都加一层调用。

**方案 C：手写代码替代 AOP**（高频路径才值得）

#### 验证

```bash
# 重新跑 async-profiler
./profiler.sh -d 30 -e cpu -f flame2.html <pid>
```

期望 intercept 的 CPU 占比从 30% 降到 5% 以下。

---

### 案例 ⑨ Intrinsic 在不同 CPU 架构的可用性

#### 现象

同一段代码，在 x86_64 服务器上 200 ns，迁移到 ARM (Graviton2) 上变成 800 ns。

```java
int crc(byte[] data) {
    CRC32 c = new CRC32();
    c.update(data);
    return (int) c.getValue();
}
```

#### 根因

`CRC32.update` 在 x86_64 上是 intrinsic，编译成 SSE4.2 的 `crc32q` 指令。但在 ARM 上：

- ARMv8.0 没有 CRC32 指令
- ARMv8.1+ 加入了 CRC32 instructions
- HotSpot 必须运行时检测 CPU 特性才决定能否用 intrinsic

如果 JVM 检测失败（比如运行在虚拟机上 cpuid 返回的特性不全），intrinsic 不开，回退到 Java 实现 —— 慢 5~10 倍。

#### 修复

**Step 1：确认 CPU 特性**

```bash
cat /proc/cpuinfo | grep -i "crc32\|features"
```

ARM 上看 `Features` 行是否包含 `crc32`。

**Step 2：确认 JVM 能否识别**

```bash
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintIntrinsics YourApp 2>&1 | grep -i crc
```

输出形如 `_updateBytesCRC32: enabled` 才说明 intrinsic 生效。

**Step 3：如果未生效**

- 升级 JDK：JDK 11 / 17 / 21 对 ARM 的 intrinsic 覆盖率逐版本提升
- 用 `-XX:+UseCRC32Intrinsics` 显式开启
- 检查是否被 `-XX:DisableIntrinsic` 误关

#### 验证

```bash
# 跑 micro benchmark 对比
java -jar crc-bench.jar
```

期望 ARM 上 CRC32 性能至少能到 x86 的 60%（不会完全持平，但不该差 4 倍）。

> **总口诀**：Intrinsic 是"和 CPU 指令集挂钩"的优化。换 CPU 必须重新验证。**别假设 ARM 服务器和 x86 服务器跑同样的 Java 代码性能一致**。

---

## 6. 上线检查清单

代码评审 / 上线前可以直接抄：

1. **不写 200 行以上的方法** —— 超过 `FreqInlineSize=325` 字节码就基本告别内联
2. **循环索引用 int 不用 long** —— 否则不是 Counted Loop，无法 SuperWord、无法 RCE
3. **数组大小提前算好放局部变量** —— `for (int i = 0; i < a.length; i++)` 比 `for (int i = 0; i < array.length; i++)` 更利于优化
4. **避免 megamorphic 调用点** —— 接口实现 ≥ 3 种时，考虑 sealed 或 switch 分发
5. **避免 `synchronized(new Object())`** —— 看似优雅，实则浪费（虽然 EA 会消除，但不要依赖）
6. **不要在循环里 new 大对象** —— 即使 EA 也未必能消除（数组 / 复杂结构）
7. **CodeCache 监控告警**（< 80% 用率）—— 否则 JIT 可能停摆
8. **设置合理 `-XX:ReservedCodeCacheSize`** —— 微服务建议 256m，大型应用 512m
9. **保持 `-XX:+TieredCompilation`** —— 不要为了"启动更快"关掉它
10. **定期跑 PrintCompilation 看启动期编译情况** —— 30 秒后还在大量编译说明预热不足
11. **Speculative Deopt 频率监控** —— `Uncommon trap` 每秒 > 100 就要排查
12. **生产 JVM 加 `-XX:+LogCompilation`** —— 出问题时离线用 JITWatch 分析
13. **预热接口跑过所有热点路径** —— 不要假设 JIT 会"自己找到"
14. **避免反射调用热点路径** —— 用 MethodHandle 或 LambdaMetafactory
15. **Stream 在热点路径慎用** —— 长 pipeline 容易 megamorphic / 阻碍内联
16. **JDK 升级要重新跑性能测试** —— C2 优化每个版本都在变，可能带来惊喜也可能踩坑
17. **ARM 部署额外验证 intrinsic** —— `PrintIntrinsics` 确认关键 intrinsic 已生效
18. **谨慎对待 GraalVM 切换** —— Stream 重的应用可能受益，纯计算和短任务可能反而更慢
19. **Vector API 显式向量化代码加 `--add-modules jdk.incubator.vector`** —— 还是孵化模块，编译运行都需要
20. **生产环境 JVM crash 时检查 CodeCache 是否被踩** —— deopt + invalidate 的边界条件可能 trigger JVM bug

> **底线口诀**：写小方法、循环用 int、避免接口爆炸、监控 CodeCache、Stream 谨慎用 —— 这五条覆盖 80% 的 C2 性能问题。

---

## 7. 还能往下挖什么

C2 是 HotSpot 的过去和现在，但 JVM 编译器生态远不止于此：

- **Graal JIT 替代 C2**（JEP 317 已废弃）—— JDK 17 之后 Graal 不再随 OpenJDK 分发，但 GraalVM 本身仍然活跃。读 `compiler/` 目录的 Java 源码（vs C2 的 C++ 实现）能看出"现代编译器开发"的工程差距
- **GraalVM Native Image 的 AOT** —— 完全跳过 JIT，启动时间从秒到毫秒。代价是反射 / 动态加载的限制，要写 reachability metadata
- **Project Leyden（JEP 483 / JEP 514）** —— OpenJDK 自己的"AOT + 启动优化"项目。目标是把 JIT 编译的成果"持久化"到磁盘，下次启动直接加载。比 CDS / AppCDS 更彻底
- **JVMCI（JEP 243）** —— JVM Compiler Interface，让你能用 Java 写 JIT 编译器（Graal 就是这么做的）。即使 Graal 在 OpenJDK 被移除，JVMCI 接口本身保留
- **LLVM / V8 / SpiderMonkey 对比** —— LLVM 是"批编译器"代表（GCC 同类），V8 / SpiderMonkey 是"JIT + 多 tier"代表。HotSpot C2 + C1 的双 Tier 架构和 V8 的 Ignition + TurboFan 极其相似
- **Vector API 的真实工程化** —— 何时用 / 何时不用，对 SIMD 友好的算法重写技巧
- **C2 in HotSpot 25+ 的演进** —— Loom 虚拟线程对 JIT 的挑战（栈不固定）、Valhalla value class 的 C2 支持、ZGC 的 colored pointer 对内存模型 / barrier 的影响

每一个都能再开一篇深文。

---

## 8. 引用

- Cliff Click, *"From Quads to Graphs: An Intermediate Representation's Journey"*, OOPSLA 1995
- Cliff Click, *"Combining Analyses, Combining Optimizations"*, PhD thesis, Rice University 1995
- OpenJDK 21 源码：`src/hotspot/share/opto/`
- OpenJDK 21 源码：`src/hotspot/share/c1/`
- OpenJDK 21 源码：`src/hotspot/share/runtime/deoptimization.cpp`
- JEP 295: Ahead-of-Time Compilation（已废弃）
- JEP 317: Experimental Java-Based JIT Compiler（已废弃）
- JEP 410: Remove the Experimental AOT and JIT Compiler
- JEP 483: Ahead-of-Time Class Loading & Linking（Project Leyden）
- JEP 489: Vector API (Ninth Incubator)
- Aleksey Shipilëv, *"JVM Anatomy Quarks"* 系列
- Chris Newland, *"JITWatch"*: <https://github.com/AdoptOpenJDK/jitwatch>
- Andrei Pangin, *"async-profiler"*: <https://github.com/async-profiler/async-profiler>
- Doug Simon et al., *"Graal IR: An Extensible Declarative Intermediate Representation"*, APPLC 2013
- T. Würthinger et al., *"Practical Partial Evaluation for High-Performance Dynamic Language Runtimes"*, PLDI 2017

---

> 配套：[JIT 与字节码](/article/jvm/jit) | [GraalVM Native Image](/article/jvm/graalvm) | [JMM 深入](/article/jvm/jmm-deep)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
