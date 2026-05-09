# eBPF 与 Cilium —— 内核态可编程与下一代网络数据面

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想速记面试，看 [K8s 速记](/article/k8s/k8s-essentials) 那一节就够了。
> 这里要回答的，是**当 iptables 规则膨胀到 5 万条、kube-proxy CPU 飙红、Cilium Agent OOM**时你需要知道的那部分。

---

## 0. 读这篇之前 —— 8 个自检题

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. eBPF（extended Berkeley Packet Filter）和 cBPF（classic BPF）的本质差别是什么？只是寄存器从 32 位变 64 位、从 2 个变 11 个吗？
2. 你写了一个 eBPF 程序，加载时报 `program rejected by verifier: back-edge from insn X to Y` —— Verifier 在拒绝什么？为什么内核敢加载用户写的代码进内核态？
3. eBPF Maps 是怎么实现"用户态进程和内核态 eBPF 程序共享数据"的？为什么不直接用 mmap 一块共享内存？
4. XDP（eXpress Data Path）跑在网卡驱动层，理论上能做到 24 Mpps（百万包每秒）；iptables 跑在 netfilter 框架里，到了 1 万条规则就开始崩。为什么差距这么大？
5. Cilium 凭什么敢说"取代 kube-proxy"？kube-proxy iptables 模式 5 万条规则的查找是 O(n)，IPVS 模式是 O(1)，那 Cilium 是 O(几)？怎么做到的？
6. Hubble 是 Cilium 自带的"网络可观测平面"，为什么它能在不开 sidecar / 不抓包的情况下看到所有 L4/L7 流量？这个数据是从哪个内核 hook 点拿到的？
7. eBPF 在**可观测性**（bpftrace / Pixie）、**安全**（Tetragon / Falco）、**网络**（Cilium / Katran）三个方向各自的核心 hook 点是什么？为什么同一套技术能撑起三个完全不同的产品形态？
8. 你想自己写 eBPF 程序：libbpf（C）、aya（Rust）、bcc（Python+C）、ebpf-go（Go），分别什么场景该用哪个？为什么 BCC 在生产环境逐渐被 libbpf 替代？

如果有半数答不上，往下看。

---

## 1. 底层模型：从 cBPF 到内核可编程

绝大多数人对 eBPF 的认知止步于"内核里的 JavaScript"或"安全的内核模块"。这层抽象解释不了上面 8 个问题。要看懂它们，必须下沉到 eBPF 虚拟机、Verifier、Maps、程序类型这四块基石。

### 1.1 eBPF 的虚拟机与字节码

eBPF 不是脚本，是**有自己指令集的虚拟机**（或者说"内核里的小型 RISC 处理器"），跑在内核态。

cBPF（1992，Steven McCanne）vs eBPF（2014，Alexei Starovoitov）核心差异：

| 维度 | cBPF | eBPF |
| --- | --- | --- |
| 寄存器宽度 | 32 位 | 64 位（贴近现代 CPU） |
| 通用寄存器 | A、X 两个累加器 | r0~r10 共 11 个 |
| 调用约定 | 无 | r1~r5 传参，r0 返回值，r10 栈底 |
| 函数调用 | 不支持 | 支持调用内核 Helper / 其他 eBPF 程序 |
| Map | 无 | 内置 12+ 种 Map |
| 用途 | 仅 socket filter（tcpdump） | 网络、追踪、安全、调度全栈 |

寄存器的 64 位化是为了**直接映射到 x86_64 / arm64 的物理寄存器**，避免运行时的位宽转换 —— 这是 eBPF 跑得快的硬件基础。

字节码示例（10 字节定长指令）：

```
op:8  dst:4  src:4  off:16  imm:32
```

一条 `r0 = r1 + r2` 在字节码里就是 `0x0f 0x21 0x0000 0x00000000`。

### 1.2 eBPF 程序的生命周期

```
源码 (.c)
  │ clang -O2 -target bpf -c
  ▼
ELF 文件 (.o，含 BPF 字节码 + Map 定义 + Section 名)
  │ 用户态 loader (libbpf / bpf2go / aya)
  ▼ bpf(BPF_PROG_LOAD, ...) syscall
内核 Verifier
  │ 通过
  ▼
JIT 编译器 (x86_64 / arm64 native code)
  │
  ▼
attach 到 hook 点 (kprobe / XDP / TC / cgroup ...)
  │
  ▼
运行时：每次 hook 触发就跑这段 native code
```

关键点：**字节码经过 Verifier 后会被 JIT 成原生机器码**，运行时性能等同于内核模块（几十纳秒级）。

> **口诀**：eBPF 的"快"靠 JIT，"安全"靠 Verifier，"通用"靠 Maps + Helper。三者缺一就回到 1990 年代的 cBPF。

### 1.3 Verifier：内核敢运行用户代码的根本原因

Verifier 是 eBPF 的**护城河**。它在加载时静态分析字节码，必须证明三件事：

1. **程序一定会终止** —— 不能有无限循环（早期完全禁止 loop，5.3 后允许 bounded loop）
2. **程序不会越权访问内存** —— 每次解引用前必须有边界检查
3. **程序不会泄漏内核内存** —— 寄存器里的"内核指针"不能写进 Map（可能被用户态读到）

实现方式：**抽象解释 + 路径模拟**。Verifier 把字节码当作有向图，深度优先遍历每条可能的执行路径，对每个寄存器跟踪它的"类型 + 值范围"（叫 `tnum`，三态数）。

举例：

```c
// 你写
int *p = bpf_map_lookup_elem(&m, &key);
return *p;

// Verifier 拒绝：
// "R1 invalid mem access 'map_value_or_null'"
// 因为 lookup 可能返回 NULL，必须先判空
```

正确写法：

```c
int *p = bpf_map_lookup_elem(&m, &key);
if (!p) return 0;
return *p;
```

Verifier 限制的演进（你写程序时会撞墙）：

| 内核版本 | 复杂度上限 | 备注 |
| --- | --- | --- |
| 4.4 | 4096 条指令 | 早期，写不了复杂逻辑 |
| 5.2 | 100 万指令 | "复杂度"按已遍历路径数算 |
| 5.3 | 支持 bounded loop | 终于能写循环 |
| 5.5 | BTF（BPF Type Format）+ CO-RE | 一次编译跨内核版本 |
| 6.x | 支持 sleepable program、kfunc | 接近"内核内的小程序" |

> **口诀**：Verifier 不是 lint 工具，是**形式化证明**。被拒绝从来不是 Verifier 的 bug，是你的代码真的不安全。

### 1.4 eBPF Maps：用户态 ↔ 内核态的桥

eBPF 程序的执行上下文是**短暂的**（几微秒）。要把状态保留下来，必须用 Map。

主要 Map 类型：

| 类型 | 用途 | 典型场景 |
| --- | --- | --- |
| `BPF_MAP_TYPE_HASH` | 哈希表 | 包计数、连接跟踪 |
| `BPF_MAP_TYPE_ARRAY` | 数组 | 配置、统计 |
| `BPF_MAP_TYPE_PERCPU_HASH` | per-CPU 哈希 | 高并发计数（无锁） |
| `BPF_MAP_TYPE_LRU_HASH` | LRU 哈希 | 容量受限的缓存 |
| `BPF_MAP_TYPE_RINGBUF` | 单生产多消费环 | 事件流（替代 perf buffer） |
| `BPF_MAP_TYPE_LPM_TRIE` | 最长前缀匹配 | 路由表、CIDR 匹配 |
| `BPF_MAP_TYPE_PROG_ARRAY` | 程序数组 | tail call（程序间跳转） |
| `BPF_MAP_TYPE_SOCKMAP` | Socket 哈希 | sockmap redirect（Cilium L7） |

为什么不直接 mmap 共享内存？因为：
- Map 是**内核管理的**，跨命名空间安全
- Map 操作是**有原子性保证的**（内核帮你处理并发）
- Map 可以被**多个 eBPF 程序共享**，做到模块化

用户态访问 Map 通过 `bpf()` syscall：

```c
// 用户态
union bpf_attr attr = { .map_fd = fd, .key = (__u64)&k, .value = (__u64)&v };
syscall(__NR_bpf, BPF_MAP_LOOKUP_ELEM, &attr, sizeof(attr));
```

每次都走 syscall 太慢？所以 5.7 后引入 `BPF_MAP_TYPE_RINGBUF`，用户态可以**直接 mmap** ringbuf 的数据区，零拷贝消费事件。

### 1.5 eBPF 程序类型：决定了能 attach 到哪里

eBPF 程序"长什么样"由 `prog_type` 决定。常见类型：

| 类型 | hook 点 | 上下文 | 典型用途 |
| --- | --- | --- | --- |
| `BPF_PROG_TYPE_KPROBE` | 任意内核函数入口 | `struct pt_regs *` | 性能追踪（bpftrace / Pixie） |
| `BPF_PROG_TYPE_TRACEPOINT` | 内核预定义的 tracepoint | 各 tracepoint 的固定结构 | 稳定追踪 |
| `BPF_PROG_TYPE_XDP` | 网卡驱动收包 | `struct xdp_md *` | 高速包处理（DDoS、L4LB） |
| `BPF_PROG_TYPE_SCHED_CLS` | TC（traffic control）egress/ingress | `struct __sk_buff *` | Cilium 的主战场 |
| `BPF_PROG_TYPE_CGROUP_SKB` | cgroup 入/出口 | `struct __sk_buff *` | NetworkPolicy egress |
| `BPF_PROG_TYPE_SOCK_OPS` | socket 状态变化（bind/connect/accept） | `struct bpf_sock_ops *` | sockmap 加速 |
| `BPF_PROG_TYPE_PERF_EVENT` | perf 采样事件 | `struct bpf_perf_event_data *` | 持续性能剖析（Parca） |
| `BPF_PROG_TYPE_LSM` | LSM hook（5.7+） | LSM 上下文 | 运行时安全（Tetragon） |

每种类型有自己的**入参**（context）、**Helper 子集**、**返回值语义**。XDP 程序返回 `XDP_PASS / XDP_DROP / XDP_TX / XDP_REDIRECT` 决定包的命运；TC 程序返回 `TC_ACT_OK / TC_ACT_SHOT` 等。

### 1.6 XDP 在网卡驱动层的处理

理解 XDP 的位置：

```
应用层 (read/recv)
  ▲
  │
socket 层
  ▲
  │
TCP / UDP
  ▲
  │
IP 层 (路由、netfilter PREROUTING)  ← iptables / nftables 在这里
  ▲
  │
网络设备子系统 (TC ingress)         ← Cilium 主用 TC
  ▲
  │
驱动层 (XDP)                         ← XDP 在这里，包还没分配 sk_buff
  ▲
  │
网卡硬件 (RX queue)
```

XDP 的"快"来自三件事：

1. **包还是 raw frame**，没分配 `sk_buff`（一个 sk_buff 大约 200+ 字节，分配/释放是 Linux 网络栈的主要开销之一）
2. **离驱动最近**，DROP 决策直接告诉驱动"这块 DMA 内存还回 RX 队列"，不进协议栈
3. **JIT 后是裸机器码**，直接在驱动 NAPI poll 上下文里跑

XDP 的三种模式：

| 模式 | 位置 | 性能 | 兼容性 |
| --- | --- | --- | --- |
| **Native** | 驱动直接调用 | 最快（24+ Mpps） | 需驱动支持（mlx5、ixgbe、i40e、virtio_net 等） |
| **Offloaded** | 智能网卡上 | 极致（包不到 CPU） | 仅 Netronome NFP 等极少数 |
| **Generic** | 网络栈通用入口 | 慢（不到 native 的 1/3） | 任何驱动都能跑 |

生产建议：**永远用 Native 模式**。Generic 模式只适合开发调试，性能对比 iptables 没什么优势。

### 1.7 Cilium 架构：Agent + Operator + Hubble

```
                K8s API Server
                  ▲
                  │ watch
                  │
   ┌──────────────┴──────────────┐
   │     Cilium Operator          │
   │  - IPAM 池管理                │
   │  - CRD reconcile              │
   │  - 集群级 GC                   │
   └──────────────────────────────┘

   每个 Node:
   ┌─────────────────────────────────────┐
   │   Cilium Agent (DaemonSet)          │
   │   ┌────────────────────────────┐    │
   │   │ Endpoint Manager            │    │
   │   │ Identity Allocator          │    │
   │   │ Policy Engine               │    │
   │   │ Service LB                  │    │
   │   └────────────────────────────┘    │
   │            │ 编译 + 加载             │
   │            ▼                        │
   │   ┌────────────────────────────┐    │
   │   │ eBPF 程序 (TC/XDP/cgroup)   │    │
   │   │ Maps (policy, lb, conntrack)│    │
   │   └────────────────────────────┘    │
   │            │                        │
   │            ▼                        │
   │   ┌────────────────────────────┐    │
   │   │ Hubble (流量观测)            │    │
   │   │ - 从 ringbuf 读 Flow        │    │
   │   │ - gRPC 暴露 / Relay 聚合    │    │
   │   └────────────────────────────┘    │
   └─────────────────────────────────────┘
```

核心抽象：
- **Endpoint** ≈ 一个 Pod 的网络身份（IP、Identity、附着在 veth 上的 eBPF 程序）
- **Identity** = 一组 Label 的哈希（让 NetworkPolicy 能用 Label 匹配，但内核态 Map 用整数 ID 高效查找）
- **Service** = ClusterIP + Backend，编译进 eBPF Map，**完全绕过 kube-proxy 的 iptables**

> **口诀**：Cilium 的核心创新不是"用了 eBPF"，而是**"把 K8s 的网络抽象（Pod、Service、Policy）一一映射到 eBPF Map + 程序"**，让网络数据面变成可编程的。

### 1.8 Hubble：免插桩的流量可观测

Hubble 的数据从哪来？答案：**Cilium 在每个 Endpoint 的 TC ingress/egress 上挂了 eBPF，每个包都会触发一次 trace 事件，写到 ringbuf**。

```
Pod A → veth → TC ingress eBPF → 解析 L3/L4/L7 → 写 trace event 到 ringbuf
                                                        │
                                                        ▼
                                          Hubble (用户态) 消费 ringbuf
                                                        │
                                                        ▼
                                          gRPC 暴露 / 转发到 Relay
```

为什么 Hubble 不需要 sidecar？因为**Cilium 已经在数据面上了**，它顺手把包的元信息收集起来就行。L7（HTTP/gRPC）的解析靠 Envoy（Cilium 集成）或 eBPF 内的简易解析器。

代价：**每个包多一次 ringbuf 写入**，CPU 开销 ~1-3%（取决于事件采样率）。

---

## 2. 源码切片：从一行 C 到 Cilium 数据面

> 基于 libbpf 1.4 / Cilium 1.16 源码。注释为本文添加。

### 2.1 一个最小 eBPF 程序：drop ICMP

```c
// drop_icmp.bpf.c
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <bpf/bpf_helpers.h>

SEC("xdp")
int drop_icmp(struct xdp_md *ctx) {
    void *data     = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;

    // ★ 必须做边界检查，否则 Verifier 拒绝
    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end) return XDP_PASS;
    if (eth->h_proto != __constant_htons(ETH_P_IP)) return XDP_PASS;

    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end) return XDP_PASS;

    if (ip->protocol == IPPROTO_ICMP) return XDP_DROP;
    return XDP_PASS;
}

char _license[] SEC("license") = "GPL";
```

**逐段点透**：

- **`SEC("xdp")`**：告诉 loader 这个函数是 XDP 程序类型。`SEC` 是宏，本质是把符号放到指定 ELF section，loader 据此推断 prog_type。
- **`ctx->data` / `ctx->data_end`**：包数据的起止指针。Verifier 把它们当作"动态范围"跟踪，每次解引用前必须证明在范围内。
- **`if ((void *)(eth + 1) > data_end) return XDP_PASS;`**：这一行不是健壮性，是 **Verifier 的硬性要求**。漏掉就会被拒绝加载，错误信息：`invalid access to packet, off=0 size=14, R1(id=0,off=0,r=0)`。
- **License 必须是 GPL 兼容的**：因为 eBPF 程序运行时会调用 GPL 导出的内核 Helper，license 不对会被拒绝。

编译加载：

```bash
clang -O2 -g -target bpf -c drop_icmp.bpf.c -o drop_icmp.o
sudo bpftool prog load drop_icmp.o /sys/fs/bpf/drop_icmp
sudo bpftool net attach xdp pinned /sys/fs/bpf/drop_icmp dev eth0
```

加载完，eth0 收到的所有 ICMP 包会被驱动直接丢弃，连 `tcpdump` 都看不到（因为 tcpdump 在更上层）。

### 2.2 BPF 字节码的加载：bpf() syscall

用户态 loader 干的事：

```c
// libbpf 简化版
int prog_fd = syscall(__NR_bpf, BPF_PROG_LOAD, &(union bpf_attr) {
    .prog_type = BPF_PROG_TYPE_XDP,
    .insn_cnt  = n_insns,
    .insns     = (__u64)insns_buf,
    .license   = (__u64)"GPL",
    .log_level = 1,
    .log_buf   = (__u64)log_buf,
    .log_size  = LOG_SIZE,
}, sizeof(union bpf_attr));

if (prog_fd < 0) {
    fprintf(stderr, "load failed: %s\nverifier log:\n%s\n",
            strerror(errno), log_buf);
    return -1;
}
```

**逐段点透**：

- **`bpf()` syscall 是个"多路复用器"**：第一个参数 `cmd` 决定干什么（PROG_LOAD / MAP_CREATE / MAP_LOOKUP_ELEM ...），后面跟一个大 union。这种设计让新增功能不用加 syscall 号。
- **`log_buf`**：Verifier 拒绝时把详细原因写到这里。生产排查 `program rejected` 必须看 verifier log，否则全是猜。
- **`prog_fd` 是个文件描述符**：返回的不是程序对象，而是 fd —— 关闭 fd 就卸载程序（除非被 pin 到 bpffs）。这是 eBPF 资源管理的核心：跟 socket / file 一样的生命周期。

attach 也走 syscall：

```c
syscall(__NR_bpf, BPF_LINK_CREATE, &(union bpf_attr) {
    .link_create = {
        .prog_fd        = prog_fd,
        .target_ifindex = if_nametoindex("eth0"),
        .attach_type    = BPF_XDP,
    },
}, sizeof(union bpf_attr));
```

`BPF_LINK_CREATE` 是 5.7+ 的新 attach 方式，返回一个 link fd，比老的 `BPF_PROG_ATTACH` 多了"显式生命周期"，避免 attach 后无法可靠 detach。

### 2.3 Cilium 的 Endpoint 与 Identity 模型

Cilium 给每个 Pod 分配一个 Endpoint。Endpoint 的核心字段（Go 简化）：

```go
// pkg/endpoint/endpoint.go
type Endpoint struct {
    ID         uint16          // node 内唯一
    ContainerID string
    IPv4        net.IP
    IPv6        net.IP
    Mac         net.HardwareAddr
    NodeMAC     net.HardwareAddr

    // ★ Identity 是 Label 集合的哈希
    SecurityIdentity *identity.Identity

    // 该 Endpoint 的 Policy 规则（已编译成 Map 内容）
    desiredPolicy *policy.EndpointPolicy

    // 挂在 veth 上的 eBPF 程序
    bpfProgram *bpf.Program
}
```

**逐段点透**：

- **`SecurityIdentity` 是 `uint32` 整数**：Label 是 string，匹配很慢；Cilium 把 `{app=foo, env=prod}` 这种 Label 集合算个哈希，分配一个全局唯一的整数 ID。**eBPF Map 的 key 是这个整数**。
- **Identity 是集群级共享的**：通过 etcd / k8s CRD 在所有节点间同步，保证 Pod A（在 node1）的 Identity 和 Pod B（在 node2）的相同 Label 拿到同一个 ID。
- **NetworkPolicy 在 eBPF 里就是"两个 Identity 之间允不允许通信"的查表**：O(1) 的 Map lookup。

Identity 分配关键路径：

```go
// pkg/identity/cache/local.go (简化)
func (m *CachingIdentityAllocator) AllocateIdentity(
    ctx context.Context, lbls labels.Labels,
) (*Identity, bool, error) {

    // 1. 计算 labels 的稳定哈希
    key := globalIdentity{lbls.LabelArray()}

    // 2. 看本地缓存
    if id, ok := m.localCache[key]; ok {
        return id, false, nil   // 已有，引用计数 +1
    }

    // 3. 走 etcd / k8s 分配新 ID（全局原子）
    numericID, err := m.backend.AllocateID(ctx, key)
    if err != nil { return nil, false, err }

    id := identity.NewIdentity(numericID, lbls)
    m.localCache[key] = id
    return id, true, nil
}
```

为什么 Identity 分配要走中心化（etcd）？因为**全局唯一**：node1 上 `{app=foo}` 第一次出现分到 ID=10000，node2 上 `{app=foo}` 必须也是 10000，否则跨节点 Policy 失效。

### 2.4 NetworkPolicy 的 eBPF 实现

NetworkPolicy 在 Cilium 里最终是一个查找：**{src_identity, dst_identity, dst_port} → ALLOW / DENY**。

eBPF 端的核心代码（简化自 `bpf/lib/policy.h`）：

```c
static __always_inline int
policy_can_access(__u32 ctx_identity, __u32 remote_id,
                  __u16 dport, __u8 proto, int dir) {

    struct policy_key key = {
        .sec_label   = remote_id,
        .dport       = dport,
        .protocol    = proto,
        .egress      = (dir == CT_EGRESS),
    };

    // ★ 多次查找：精确 → 通配 dport → 通配 proto → 通配 identity
    struct policy_entry *p = map_lookup_elem(&POLICY_MAP, &key);
    if (p && !p->deny) return CTX_ACT_OK;

    key.dport = 0;   // 通配端口
    p = map_lookup_elem(&POLICY_MAP, &key);
    if (p && !p->deny) return CTX_ACT_OK;

    // ... 几次降级查找
    return DROP_POLICY;
}
```

**逐段点透**：

- **每个 Endpoint 有自己的 POLICY_MAP**：通过 BPF Map-in-Map（`BPF_MAP_TYPE_HASH_OF_MAPS`）实现，外层 key=endpoint_id，value=该 Endpoint 的 policy map fd。
- **多次降级查找**是为了支持通配规则（"允许所有端口"、"允许任意 source"）。每次查找仍然是 O(1)，最多查 4-5 次。
- **`deny` 字段**：Cilium 1.9+ 支持 deny rule，整体语义是"先匹配 deny，再匹配 allow"。
- **整个判定无锁**：Map 的并发安全由内核保证，eBPF 程序里不需要任何同步原语。

数据面调用：

```c
// bpf/bpf_lxc.c (Pod 入口程序)
__section_tail(CILIUM_MAP_CALLS, CILIUM_CALL_IPV4_FROM_LXC)
int tail_handle_ipv4_cont(struct __ctx_buff *ctx) {
    // ... 解析包，识别 src/dst identity
    int ret = policy_can_access(SECCTX, src_id, dport, proto, CT_INGRESS);
    if (ret < 0) return ret;     // DROP
    // ... 转发到目的
}
```

`__section_tail` 是 **Tail Call**：把大的 eBPF 程序拆成多个，通过 `bpf_tail_call` 跳转。原因：单个 eBPF 程序复杂度有上限（Verifier 限制），拆分后每段都能通过验证。

### 2.5 Service Load Balancing：替代 kube-proxy

kube-proxy iptables 模式查找一个 Service 是 O(n)（n = Service 数 × Backend 数）：

```bash
# kube-proxy 生成的 iptables 链
-A KUBE-SERVICES -d 10.96.0.10/32 -p udp --dport 53 -j KUBE-SVC-TCOU7JCQXEZGVUNU
-A KUBE-SVC-TCOU7JCQXEZGVUNU -m statistic --mode random --probability 0.333 -j KUBE-SEP-AAA
-A KUBE-SVC-TCOU7JCQXEZGVUNU -m statistic --mode random --probability 0.500 -j KUBE-SEP-BBB
-A KUBE-SVC-TCOU7JCQXEZGVUNU -j KUBE-SEP-CCC
```

5000 个 Service 就有几万条规则，每个包都要顺序匹配。

Cilium 的做法：把 Service → Backend 关系写进 eBPF Map：

```c
// bpf/lib/lb.h (简化)
struct lb4_key {
    __be32 address;
    __be16 dport;
    __u16  backend_slot;   // 0 = Service 主表项，>0 = 第几个 Backend
};

struct lb4_service {
    __u32 backend_id;
    __u16 count;           // 主表项里：Backend 总数
    __u16 rev_nat_index;
};

static __always_inline int
lb4_local(struct __ctx_buff *ctx, struct ipv4_ct_tuple *tuple) {
    struct lb4_key key = { .address = tuple->daddr, .dport = tuple->dport };
    struct lb4_service *svc = map_lookup_elem(&LB4_SERVICES, &key);
    if (!svc) return CTX_ACT_OK;   // 不是 Service IP，正常转发

    // ★ 一致性哈希挑 Backend
    __u32 slot = (hash_from_tuple(tuple) % svc->count) + 1;
    key.backend_slot = slot;
    struct lb4_service *be = map_lookup_elem(&LB4_SERVICES, &key);
    struct lb4_backend *backend = map_lookup_elem(&LB4_BACKENDS, &be->backend_id);

    // DNAT
    tuple->daddr = backend->address;
    tuple->dport = backend->port;
    return DIR_REPLY;
}
```

**逐段点透**：

- **三次 Map 查找完成 LB**：第一次拿 Service 元信息，第二次按 slot 拿 Backend ID，第三次拿 Backend 详情。每次都是 O(1) 哈希查找。
- **`slot 0` 是元数据，slot 1..N 是 Backend**：把"一个 Service 的所有 Backend"用 (address, dport, slot) 这个组合 key 编码进同一个 Map，避免 Map-in-Map 的两次跳。
- **一致性哈希**：基于五元组哈希挑 Backend，保证同一个连接永远走同一个 Backend（无需 conntrack）。
- **DNAT 在 eBPF 里完成**：直接改包的目的 IP / 端口，回包时通过 conntrack Map 反查做 SNAT。

性能差距：5000 Service 时 kube-proxy iptables 模式新建连接延迟 ~5ms，Cilium ~50μs（差 100 倍）。

### 2.6 Hubble Flow 的收集

Cilium 数据面每处理一个包，会触发一次 `send_trace_notify`：

```c
// bpf/lib/trace.h
static __always_inline void
send_trace_notify(struct __ctx_buff *ctx, enum trace_point obs_point,
                  __u32 src, __u32 dst, __u16 dst_id, __u32 ifindex,
                  enum trace_reason reason, __u32 monitor) {

    if (!emit_trace_notify(obs_point, monitor)) return;   // 采样

    struct trace_notify msg = {
        .type      = CILIUM_NOTIFY_TRACE,
        .obs_point = obs_point,
        .source    = src,
        .hash      = get_hash_recalc(ctx),
        .orig_len  = ctx_full_len(ctx),
        .ifindex   = ifindex,
        .reason    = reason,
    };

    // 写入 perf event ring，用户态 Hubble 消费
    ctx_event_output(ctx, &EVENTS_MAP, BPF_F_CURRENT_CPU,
                     &msg, sizeof(msg));
}
```

**逐段点透**：

- **`emit_trace_notify` 是采样开关**：通过 Map 配置，可以按 monitor level 决定丢多少。生产环境一般开 1% 采样，全量会打爆 CPU。
- **`ctx_event_output` 写 perf event ring**：每个 CPU 一个 ring buffer，无锁写入。
- **用户态 Hubble**：`mmap` 这些 ring，poll 消费，转换成 protobuf 格式的 Flow，通过 gRPC 暴露。
- **Flow 不是"包"是"事件"**：一次 trace 事件就是一行可观测数据（src_ip、dst_ip、verdict、latency 等），不是原始报文。

> **口诀**：Hubble 的低开销靠**采样 + per-CPU ringbuf + mmap 零拷贝**三件套。生产环境关键是把采样率调到业务能接受的水平。

---

## 3. 实测数据：iptables 死亡曲线 vs Cilium 平坦线

### 3.1 XDP vs iptables 的转发性能（DDoS 防护场景）

**测试环境**：
- 客户端：发 64 字节 UDP 包，单核 wrk 压测
- 服务端：Linux 5.15，Mellanox CX-5 25Gbps 网卡
- 任务：丢弃来自指定 CIDR 的包

| 方案 | 单核 PPS | 25G 线速利用率 | CPU 占用 |
| --- | --- | --- | --- |
| iptables（5 条规则） | 1.2 Mpps | 4.8% | 100%（被打爆） |
| iptables（500 条规则） | 0.3 Mpps | 1.2% | 100% |
| nftables（500 条规则） | 0.6 Mpps | 2.4% | 100% |
| XDP（generic） | 4 Mpps | 16% | 100% |
| **XDP（native）** | **24 Mpps** | **96%** | **45%** |
| XDP（offload，NFP） | 50+ Mpps | 100% | 0% |

**反直觉的点**：

- iptables **规则数线性影响性能**（甚至不到线性，因为 cache miss 加剧），XDP 用 LPM trie，5 条 vs 50 万条几乎无差异。
- XDP native 模式 CPU 占用反而**低于** iptables —— 因为不进协议栈，省了 sk_buff 分配/释放/路由查找等大头。

> **口诀**：iptables 规则不是越多越慢一点，是**到了某个临界点直接崩**。生产环境单网卡 iptables 规则数上限大概 5000 条。

### 3.2 Cilium vs Calico 的吞吐对比（Pod 间通信）

**测试环境**：3 节点 K8s 集群，节点 8C16G，10Gbps 网络，Pod 间 iperf3。

| CNI | 数据面 | 吞吐 | P99 延迟 |
| --- | --- | --- | --- |
| Calico | iptables | 7.2 Gbps | 0.42 ms |
| Calico | eBPF dataplane | 9.4 Gbps | 0.18 ms |
| Cilium | eBPF (kernel routing) | 9.5 Gbps | 0.16 ms |
| Cilium | eBPF (host routing) | 9.6 Gbps | 0.12 ms |
| Cilium | eBPF + XDP fast-path | 9.7 Gbps | 0.09 ms |

**结论**：

- **eBPF 数据面普遍比 iptables 快 30%+**，且延迟低 50%
- Cilium 的 host routing（绕过常规路由表）和 XDP fast-path（跨节点流量在 XDP 层直接转发）是它领先的关键
- Calico 的 eBPF 模式追上来了，差距不大

### 3.3 大规模 NetworkPolicy 下的延迟

**测试**：每 Pod 配 1000 条 NetworkPolicy 规则，看新建连接延迟。

| CNI / 模式 | P50 | P99 | 100k 连接/s 下 CPU |
| --- | --- | --- | --- |
| Calico iptables | 1.8 ms | 12 ms | 60% |
| Calico eBPF | 0.2 ms | 1.1 ms | 18% |
| **Cilium eBPF** | **0.15 ms** | **0.7 ms** | **12%** |
| 无 Policy（裸 K8s） | 0.1 ms | 0.5 ms | 8% |

**核心**：Cilium 把 1000 条规则编译成 Map 查找，**新建连接的 Policy 判定 ~5μs**；iptables 模式下规则数线性影响延迟。

### 3.4 eBPF Profiling 的开销（vs perf）

**测试**：在 8C 机器上跑一个 CPU 密集业务，开各种 profiler 看开销。

| Profiler | 采样频率 | 业务 QPS 损失 |
| --- | --- | --- |
| perf record（call-graph dwarf） | 99 Hz | 12% |
| perf record（fp） | 99 Hz | 4% |
| async-profiler | 99 Hz | 2.5% |
| **bpftrace（kprobe）** | 99 Hz | **1.8%** |
| **Parca Agent（eBPF）** | 19 Hz 持续 | **0.5%** |

**为什么 eBPF 这么省**：
- perf 把样本通过 perf event 写到用户态，ring buffer 满了要 wakeup 用户态进程
- eBPF 在内核态直接做**栈聚合**，写到 Map 里，用户态批量读，syscall 量级少几个数量级

> **口诀**：长时间持续剖析必须用 eBPF。短时高频抓栈，async-profiler / perf 还是更精确。

### 3.5 Cilium 替代 kube-proxy 的延迟改善

**测试**：5000 Service / 50000 Endpoint 的集群，从 Pod 访问 ClusterIP 的 P99。

| 模式 | P50 | P99 | 5000 Service 启动时间 |
| --- | --- | --- | --- |
| kube-proxy iptables | 1.4 ms | 11 ms | iptables 刷新 ~45s |
| kube-proxy IPVS | 0.3 ms | 1.8 ms | ipvsadm sync ~5s |
| **Cilium kubeProxyReplacement** | **0.05 ms** | **0.5 ms** | **Map 写入 <1s** |

**额外收益**：
- iptables 模式每次 Endpoint 变化要重新刷整张表，5000 Service 时 kube-proxy 单次 sync 占 1 个 CPU 持续 30 秒
- Cilium 是增量更新 Map，几乎瞬时

> **口诀**：超过 1000 Service 必须考虑 kube-proxy 替代方案；超过 5000 Service 用 iptables 模式就是自杀。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 内核版本不兼容 —— "Cilium 起不来，CrashLoopBackOff"

#### 现象

Cilium DaemonSet 在某个节点反复 CrashLoopBackOff，日志：

```
level=fatal msg="kernel version 4.14 is too old to run Cilium, minimum required is 4.19.57"
```

或者更隐蔽的：

```
level=warning msg="BPF feature CONFIG_BPF_LSM not available"
level=error msg="failed to attach socket lookup program"
```

#### 根因

eBPF 是个**和内核版本强相关**的子系统。Cilium 用到的特性逐版本累积：

| Cilium 特性 | 最低内核 | 关键 commit |
| --- | --- | --- |
| 基础 bpf() syscall | 3.18 | — |
| eBPF maps | 3.19 | — |
| TC eBPF | 4.4 | — |
| XDP | 4.8 | — |
| **Cilium 1.x 最低要求** | **4.19.57** | — |
| sockmap / SO_REUSEPORT BPF | 4.19 | — |
| BTF + CO-RE | 5.5 | — |
| Cilium kubeProxyReplacement | 5.4+ 推荐 | — |
| **Cilium BPF host routing** | **5.10+** | — |
| Bandwidth Manager (EDT) | 5.1+ | — |
| Cilium Service Mesh L7 | 5.10+ 推荐 | — |
| BPF LSM | 5.7 | — |

CentOS 7 默认内核 3.10，CentOS 8 默认 4.18 —— 都跑不了 Cilium 的高级特性。Ubuntu 18.04 4.15 也不行，必须升 HWE。

更坑的是**有些发行版"backport"了部分 eBPF 特性**：你以为 4.18 不行，结果 RHEL 4.18 backport 了一堆，又能跑了。Cilium 启动时会检测特性矩阵：

```bash
cilium status | grep -A 30 "BPF Features"
```

#### 修复

**短期**：升级节点内核到 5.10+。Ubuntu：

```bash
sudo apt install linux-generic-hwe-22.04
sudo reboot
uname -r   # 应该看到 5.x 或 6.x
```

**长期**：基础设施团队应该把"内核版本"写进节点选型规范。生产建议：

| 场景 | 推荐内核 |
| --- | --- |
| Cilium 基础（CNI + Policy） | 5.10+ |
| kubeProxyReplacement | 5.10+ |
| Service Mesh / L7 | 5.15+ |
| Tetragon 运行时安全 | 5.15+ |
| 想用最新特性 | 6.1 LTS+ |

如果就是降不了内核，可以用 Cilium 的 **chained mode**：让 Calico/Flannel 跑底层 IPAM，Cilium 仅负责 Policy 和可观测，对内核要求会低一些（但损失大部分性能优势）。

#### 验证

```bash
# 启动前自检
docker run --rm cilium/cilium-cli kernel-check

# 已部署的看支持矩阵
kubectl exec -n kube-system ds/cilium -- cilium status --verbose | grep -A 50 "Kernel Features"
```

> **口诀**：**eBPF 不是后向兼容的库，是内核的一部分**。生产环境上 Cilium 前先把所有节点内核升到 5.10+，否则后面的坑挖都挖不完。

---

### 案例 ② Verifier 拒绝复杂 eBPF 程序 —— "本地能跑，CI 上挂"

#### 现象

你写了个稍微复杂点的 eBPF 程序（有几层循环、几十次 Map 查找），本地 Ubuntu 22.04（5.15）加载正常，CI 用的 Ubuntu 20.04（5.4）报错：

```
libbpf: prog 'handle_packet': BPF program load failed: Argument list too long
libbpf: prog 'handle_packet': -- BEGIN PROG LOAD LOG --
processed 1048576 insns (limit 1000000) max_states_per_insn 5 ...
```

或者：

```
infinite loop detected at insn 152
```

或者：

```
R3 unbounded memory access, make sure to bounds check any such access
```

#### 根因

Verifier 的限制本身**逐版本放宽**：

| 版本 | 复杂度上限 | Loop 支持 |
| --- | --- | --- |
| 4.4 | 4096 insn | 无 |
| 4.14 | 13.1 万 | 无 |
| 5.2 | 100 万 | 无 |
| 5.3 | 100 万 | bounded loop（`#pragma unroll` 或显式 bounded） |
| 5.4 | 100 万 | bpf_loop helper |
| 5.10 | 100 万 | 更智能的路径剪枝 |

CI 跑 5.4，你写了个内层循环 32 次、外层 16 次的嵌套，Verifier 把每条可能路径都模拟一次，状态空间爆炸 → "Argument list too long"（其实是路径数超限）。

#### 修复

**手段 1：减少分支**。Verifier 路径数 = 各分支组合，减少早期分支能指数级降低复杂度：

```c
// ❌ 每个 case 都是一条路径
switch (proto) {
case IPPROTO_TCP: handle_tcp(); break;
case IPPROTO_UDP: handle_udp(); break;
case IPPROTO_ICMP: handle_icmp(); break;
}

// ✅ 用 tail call 拆分
__u32 idx = proto_to_idx(proto);
bpf_tail_call(ctx, &PROG_ARRAY, idx);
```

`bpf_tail_call` 把控制权完全交给另一个 eBPF 程序，**不返回**，所以 Verifier 不需要把后续路径合并进当前程序。

**手段 2：用 `bpf_loop` 替代手写循环**（5.17+）：

```c
// ❌ Verifier 要展开所有路径
for (int i = 0; i < 64; i++) {
    if (process_one(ctx, i) < 0) break;
}

// ✅ bpf_loop 是个 helper，Verifier 只看一次循环体
bpf_loop(64, process_one_cb, ctx, 0);
```

**手段 3：`__always_inline` 慎用**。看着是优化，实际把所有路径展开 → Verifier 状态爆炸。Cilium 代码里大量函数标 `__always_inline` 是因为他们做了精细的复杂度控制；你写新代码不要无脑加。

**手段 4：升级开发/CI 内核到 5.10+**。这是一劳永逸的方案。

#### 验证

```bash
# 本地测试 Verifier
sudo bpftool prog load your_prog.o /sys/fs/bpf/test type xdp

# 看 verifier log（成功也能看，看复杂度数字）
sudo bpftool prog dump xlated id <id> verbose
```

如果 `processed N insns` 接近上限，提前重构。

> **口诀**：**Verifier 失败永远是你代码的锅，不是 Verifier 的锅**。把"复杂度"当作和 CPU/内存一样的资源管理。

---

### 案例 ③ Map 大小配置错误导致溢出 —— "新连接突然全失败"

#### 现象

线上跑了 6 个月的 Cilium 集群，某天突然出现大量 Pod-to-Pod 连接失败。Cilium 日志：

```
level=warning msg="BPF map full" map=cilium_ct4_global
level=warning msg="conntrack table full"
```

`hubble observe` 看到大量 `verdict=DROPPED reason=CT_MAP_INSERTION_FAILED`。

#### 根因

Cilium 的 Conntrack（连接跟踪）Map 默认 `512000` 条目。每条连接（含 NAT 反向项）占 2 条，所以最大 ~25 万并发连接。

集群升到 100 个 Pod 后，每个 Pod 有几千连接 + 频繁短连接，瞬时连接数超过 25 万，Map 满了 → 新连接的 conntrack 项写不进 → DROP。

更隐蔽的：**Map 是 LRU 哈希**，理论上满了应该淘汰旧的 —— 但 Cilium 的 conntrack 不是 LRU 类型（用的是 `BPF_MAP_TYPE_HASH`，不是 `BPF_MAP_TYPE_LRU_HASH`），溢出就直接失败。

为什么不用 LRU？因为 conntrack 项 LRU 淘汰 = "正在用的连接被淘汰" → 已建立连接突然 DROP，更糟。所以选了"插入失败"这种"硬故障好 debug"的策略。

#### 修复

**短期**：扩大 Map 大小（需要重启 Cilium Agent，会断流秒级）。

```yaml
# values.yaml
bpf:
  ctTcpMax: 1500000         # TCP conntrack 上限
  ctAnyMax: 750000          # UDP/ICMP conntrack 上限
  natMax: 1500000           # NAT 表大小
  policyMapMax: 65536       # 每 Endpoint 的 Policy 条目上限
```

```bash
helm upgrade cilium cilium/cilium -n kube-system -f values.yaml
```

**长期**：

1. **GC 策略调优**：Cilium 默认每 30s 扫一次 conntrack Map 清理过期项。短连接多的场景调短：`bpf-ct-gc-interval=10s`。
2. **缩短 TCP 超时**：Cilium 默认 TCP_FIN 状态保留 10s，对短连接业务太长。`bpf-ct-tcp-fin-timeout=2s`。
3. **业务侧用连接池**：很多 conntrack 爆掉是因为业务每次请求新建 TCP 连接。HTTP 加 Keep-Alive、DB 加连接池。

#### 验证

```bash
# 看实时 conntrack 用量
kubectl exec -n kube-system ds/cilium -- cilium bpf ct list global | wc -l

# 看 Map 占用率
kubectl exec -n kube-system ds/cilium -- cilium metrics list | grep bpf_map

# Prometheus 查询
cilium_bpf_map_pressure{map_name="cilium_ct4_global"} > 0.8
```

告警：Map 利用率 > 80% 持续 5 分钟。

> **口诀**：所有 eBPF Map 都是**有界资源**。上线前必须按业务峰值算 ctMax / natMax / policyMapMax，留 2x buffer。

---

### 案例 ④ Cilium Agent OOM —— "升级后节点上 cilium-agent 反复重启"

#### 现象

把 Cilium 从 1.13 升到 1.15 后，部分大节点（200+ Pod）的 cilium-agent 反复 OOM Killed。`kubectl top pod` 看到内存使用 ~3GB（limit 2GB）。

#### 根因

Cilium Agent 是用户态进程，主要内存消耗：

1. **Endpoint 缓存**：每个 Pod 一个 Endpoint，含完整 Label / Identity / Policy
2. **Identity Cache**：全集群所有 Identity 在每个节点都缓存一份
3. **Policy 编译产物**：每个 Endpoint 的 Policy 编译成内存中的规则集，再写到 eBPF Map
4. **Hubble 流量缓冲**：默认 4096 个 Flow 在内存中

200 Pod × 10 Identity / Pod = 2000 Identity 缓存，加上 Hubble buffer（升 1.15 后默认从 4096 涨到 8192），加上 1.15 引入的几个新组件（如 BGP Control Plane 的路由表）—— 容易吃完 2GB。

更隐蔽的：**Hubble metrics**（如果开了 L7 metrics）会在用户态聚合，HTTP path / gRPC method 是高基数标签，Pod 多 + 路径多 = 标签笛卡儿爆炸。

#### 修复

**Step 1：增加 limit**（治标）：

```yaml
resources:
  limits:
    cpu: "4"
    memory: "4Gi"
  requests:
    cpu: "500m"
    memory: "512Mi"
```

参考值：节点 200 Pod 给 4Gi，500 Pod 给 8Gi。

**Step 2：缩 Hubble buffer**：

```yaml
hubble:
  eventBufferCapacity: 4095     # 默认 8191，缩一半
  eventQueueSize: 1024
```

**Step 3：禁用高基数 metrics**：

```yaml
hubble:
  metrics:
    enabled:
      - dns
      - drop
      # ❌ - "http:destinationContext=workload-name;sourceContext=workload-name"
      #   path 标签会爆
      - "httpV2:exemplars=true;labelsContext=source_namespace,destination_namespace"
```

**Step 4：用 pprof 抓 heap**：

```bash
kubectl exec -n kube-system <cilium-pod> -- \
    curl -s http://localhost:9234/debug/pprof/heap > heap.out
go tool pprof -http=:8080 heap.out
```

通常能看到 `IdentityCache` 或 `endpointmanager.endpoints` 占大头。

#### 验证

```bash
# 持续观察
watch 'kubectl top pod -n kube-system -l k8s-app=cilium'

# 升级后观察一周内是否还有 OOMKilled
kubectl get pod -n kube-system -l k8s-app=cilium -o json | \
  jq '.items[] | {name: .metadata.name, restarts: .status.containerStatuses[0].restartCount}'
```

> **口诀**：Cilium Agent 不是无状态轻进程，**每节点 Pod 数线性影响内存**。容量规划先按 4Gi/200Pod 起步，按需调整。

---

### 案例 ⑤ Hubble 数据量爆炸 —— "ELK 集群被 Hubble 打爆"

#### 现象

接入 Hubble Export 把 Flow 写到 Elasticsearch，一周后 ES 集群存储炸了，单天 5TB Flow 数据。

#### 根因

默认 Hubble 是**全量采集**所有 Flow。一个中等业务节点每秒几万包 = 几万 Flow，按每条 Flow ~500 字节算：

- 单节点：50000 × 500 = 25 MB/s
- 100 节点集群：2.5 GB/s = **216 TB/天**

ES 即使开 ILM 也扛不住，更别说查询慢死。

而且：**Hubble Flow 不是日志，是事件流**。把"每个包"当作日志存，等于把 tcpdump 的输出全量入库。

#### 修复

**Step 1：采样**。Cilium 1.13+ 支持 Flow 采样：

```yaml
hubble:
  flowFilter:
    # 只采集 DROP 和 L7 相关的，正常通过的不要
    - verdict: ["DROPPED", "ERROR", "AUDIT"]
    - eventType: ["policy-verdict", "http", "dns"]
```

**Step 2：分级采集**：

```yaml
# 默认 1% 采样
# DROP / 异常全采
# L7（HTTP/DNS）按业务需要采
hubble:
  exportFileMaxSizeMb: 100
  exportFileMaxBackups: 3
  exportFilters:
    - name: errors
      match:
        - verdict: ["DROPPED", "ERROR"]
    - name: l7-sample
      match:
        - eventType: ["http", "dns"]
      ratio: 0.01    # 1% 采样
```

**Step 3：用 Hubble 自己的 metrics 替代日志**。Flow 入 ES 是为了"事后查"，但绝大多数需求其实是"统计 P99 / 错误率" —— 这些 Hubble 直接 export 成 Prometheus metrics 就够了：

```yaml
hubble:
  metrics:
    enabled:
      - "drop:sourceContext=pod;destinationContext=pod"
      - "tcp:sourceContext=workload-name"
      - "flow:sourceContext=workload-name"
      - "httpV2:exemplars=true"
```

Metrics 在 Cilium 节点本地聚合，**不走网络**，存储成本不到日志的 1%。

**Step 4：长留只留高价值数据**。原始 Flow 只在本地 ringbuf 留 1 小时（用 `hubble observe --since 1h` 查），异常 Flow 和审计相关导出到对象存储归档。

#### 验证

```bash
# 看 Hubble 当前的 Flow 速率
kubectl exec -n kube-system ds/cilium -- hubble status
# 关注 "Flows/s" 指标

# 看 export 文件大小
kubectl exec -n kube-system ds/cilium -- ls -lh /var/run/cilium/hubble/

# Prometheus 查询每秒 Flow 数
sum(rate(hubble_flows_processed_total[1m])) by (node)
```

> **口诀**：**Hubble 不是日志系统，是可观测平面**。Flow 全量入库永远是错的；用 metrics 兜常态，用采样 + 异常过滤兜偶发查询。

---

### 案例 ⑥ NetworkPolicy 配置错误导致 Pod 间不通 —— "Policy 加上去后整个 ns 黑屏"

#### 现象

业务团队给 namespace 加了第一条 NetworkPolicy：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: api
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 8080
```

加完后**整个 backend namespace 的所有服务全部连不上**，包括没有这条 Policy 选中的 Pod。

#### 根因

K8s NetworkPolicy 的语义是 **"selector 选中的 Pod，默认拒绝所有未明示的入站流量"**：

- `podSelector: matchLabels: {app: api}` 选中所有 `app=api` 的 Pod
- 这些 Pod 一旦被任何 NetworkPolicy 选中，就进入"白名单模式"
- Ingress 只允许 `from frontend on port 8080`
- **所有其他来源的流量都被丢**：Prometheus 抓 metrics 端口、健康检查（如果不是来自 frontend）、其他 backend 服务调用、kubelet 的 livenessProbe...

更坑的是 K8s NetworkPolicy **不影响出站**（不写 egress 部分时）—— 但已经够把服务搞挂了。

很多人以为"加一条 allow 是放开"，其实 K8s NetworkPolicy 是 **"被选中即拒绝默认 + allowlist"** 的语义。

#### 修复

**Step 1：先用 Cilium 的 audit 模式预演**：

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-from-frontend
  namespace: backend
spec:
  endpointSelector:
    matchLabels:
      app: api
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "8080"
```

启用 audit 模式（不 enforce，只记录）：

```bash
kubectl annotate ns backend io.cilium.policy-enforcement=audit
# 或全局
helm upgrade cilium ... --set policyEnforcementMode=audit
```

观察 Hubble：

```bash
hubble observe --namespace backend --verdict AUDIT
```

会看到 "如果 enforce 这条 Policy，下面这些流量会被 drop"。验证没有误伤后再切 enforce。

**Step 2：补全所有合法来源**：

```yaml
ingress:
# 业务流量
- fromEndpoints:
  - matchLabels:
      app: frontend
  toPorts:
  - ports: [{port: "8080"}]

# Prometheus
- fromEndpoints:
  - matchLabels:
      "k8s:io.kubernetes.pod.namespace": monitoring
      app: prometheus
  toPorts:
  - ports: [{port: "9090"}]   # metrics 端口

# 健康检查（来自 host network 的 kubelet）
- fromEntities:
  - host
  toPorts:
  - ports: [{port: "8080"}]   # liveness port
```

**Step 3：`fromEntities` 善用**。Cilium 扩展了几个特殊实体：

| 实体 | 含义 |
| --- | --- |
| `host` | 节点本身（kubelet 健康检查） |
| `remote-node` | 其他节点 |
| `world` | 集群外（互联网） |
| `cluster` | 集群内任何 Pod |
| `kube-apiserver` | 自动识别的 API Server |

很多 Policy 漏掉 `host` 是导致 livenessProbe 失败的根因。

#### 验证

```bash
# 看哪些 Pod 被这条 Policy "纳入管辖"
kubectl exec -n kube-system ds/cilium -- \
    cilium endpoint list -o jsonpath='{range .[*]}{.id}{" "}{.policy.realized}{"\n"}{end}'

# 实时看 drop
hubble observe --namespace backend --verdict DROPPED -f

# 跑测试 Pod 主动验证
kubectl run test --rm -ti --image=nicolaka/netshoot -- \
    curl http://api.backend:8080
```

> **口诀**：NetworkPolicy 永远先 audit 再 enforce。每个 Policy 上线前在 Hubble 里看 audit verdict 跑 24h，确认没误伤再切。

---

### 案例 ⑦ Cilium 与传统 CNI 切换的迁移 —— "在线把 Calico 换成 Cilium"

#### 现象

集群跑了 2 年的 Calico，要换 Cilium 拿 kube-proxy 替代和 L7 可观测能力。但业务不接受停机重建集群。

#### 根因（迁移难点）

1. **CNI 不能并存**：每个 Pod 同时只能由一个 CNI 管 Pod 网络。换 CNI = 重建 Pod。
2. **IP 地址段冲突**：Calico 用 192.168.0.0/16，Cilium 默认 10.0.0.0/8 ——可能和 Service CIDR 冲突。
3. **NetworkPolicy 不兼容**：Calico 的 `GlobalNetworkPolicy` 不是标准 K8s 资源，Cilium 不识别。
4. **kube-proxy 替换有顺序**：先装 Cilium 再删 kube-proxy 还是反过来？错了会断流。

#### 迁移方案：节点级灰度

参考 Cilium 官方迁移文档，核心思路是 **"per-node migration"**：

**Step 1：装 Cilium，但配置不接管现有节点**：

```yaml
# values.yaml
cluster:
  name: prod
  id: 1

# 限制 Cilium 只在带特定 label 的节点工作
nodeSelector:
  io.cilium.migration/cilium-default: "true"

cni:
  customConf: true   # 不写默认 CNI 配置文件，等迁移时手动切换

ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4PodCIDRList: ["10.244.0.0/16"]   # 选个不冲突的

policyEnforcementMode: never   # 一开始不 enforce

# 关键：换 kube-proxy 但保留兼容
kubeProxyReplacement: true
k8sServiceHost: <api-server-ip>
k8sServicePort: 6443
```

```bash
helm install cilium cilium/cilium -n kube-system -f values.yaml
```

**Step 2：选一个 canary 节点迁移**：

```bash
# 节点设不可调度并清空
kubectl cordon node-1
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data

# 切换 CNI 配置
kubectl label node node-1 io.cilium.migration/cilium-default=true

# 删除该节点的 Calico Pod
kubectl delete pod -n kube-system -l k8s-app=calico-node \
    --field-selector spec.nodeName=node-1

# 删除 /etc/cni/net.d/10-calico.conflist（让 Cilium 写入新配置）
ssh node-1 'sudo rm /etc/cni/net.d/10-calico.conflist && sudo systemctl restart kubelet'

# 让节点接受新 Pod
kubectl uncordon node-1
```

**Step 3：观察一段时间，再迁移下一个节点**。所有节点完成后，删除 Calico：

```bash
kubectl delete -f calico.yaml
```

**Step 4：迁移 Policy**。用 `hubble policygen` 工具或手写 CiliumNetworkPolicy 重写 Calico Policy。

#### 验证

```bash
# 跨节点连通性测试
kubectl run -it --rm test1 --image=nicolaka/netshoot --overrides='{"spec":{"nodeName":"node-1"}}' -- \
    ping <pod-on-node-2-ip>

# 看 CNI 配置
kubectl exec -n kube-system ds/cilium -- cat /etc/cni/net.d/05-cilium.conf

# Cilium 健康
kubectl exec -n kube-system ds/cilium -- cilium status
```

#### 关键坑

- **kube-proxy 不能在 Cilium kubeProxyReplacement 模式下并存**：装 Cilium 时如果集群还有 kube-proxy，必须先停 kube-proxy DaemonSet，否则 NodePort 双重 DNAT 出问题。
- **混合期 Pod 间通信**：Calico 节点上的 Pod 和 Cilium 节点上的 Pod 通过 underlay 路由（K8s Service）通信没问题；但 Calico 的 Network Policy 不识别 Cilium 节点的 Pod —— 短暂窗口期 Policy 可能误判，业务需要能容忍。

> **口诀**：CNI 迁移没有"无缝热切"，只有"小批量灰度 + 业务能接受短暂中断"。规划期至少留 2 周。

---

### 案例 ⑧ XDP 模式下硬件不支持 —— "开了 XDP，结果 TCP 重传飙升"

#### 现象

为了榨取性能，把 Cilium 配置成 `loadBalancer.acceleration: native`（XDP 加速 LB）：

```yaml
loadBalancer:
  mode: dsr
  acceleration: native
```

部署后流量正常一段时间，业务报告 P99 延迟从 50ms 涨到 800ms，监控看到 TCP 重传率从 0.01% 涨到 5%。

#### 根因

XDP native 模式要求**网卡驱动支持 XDP**。如果驱动不支持，Cilium 会**自动降级到 generic XDP**（在通用网络栈入口跑 XDP 程序）—— 但这种模式比正常路径还慢，因为 XDP 程序在 sk_buff 已分配的位置跑，多了一次 BPF 调用且没省任何东西。

判断网卡是否支持 XDP native：

```bash
ip link show eth0
# 看是否有 "xdp" feature
ethtool -i eth0
# 看 driver 字段，对照支持列表
```

支持 native XDP 的主流驱动：

| 驱动 | 网卡 | 备注 |
| --- | --- | --- |
| `mlx5_core` | Mellanox CX-4/5/6 | 完整支持 |
| `i40e` | Intel X710/X722 | 完整支持 |
| `ixgbe` | Intel X540/X550 | 完整支持 |
| `ice` | Intel E810 | 完整支持 |
| `bnxt_en` | Broadcom | 部分 |
| `virtio_net` | KVM 虚拟机 | 5.5+ 支持 |
| `ena` | AWS Nitro | 需较新内核 |
| `gve` | GCP | 部分支持 |
| **`vmxnet3`** | **VMware** | **❌ 不支持** |
| `e1000` | 老 Intel | ❌ |

特别坑的是云上：阿里云 `virtio_net` 5.10+ 支持，但有些 ECS 实例的弹性网卡是 `iohub`（私有驱动），不支持。

#### 修复

**Step 1：检查实际是 native 还是 generic**：

```bash
kubectl exec -n kube-system ds/cilium -- cilium status | grep XDP
# 期望看到：XDP Acceleration: Native
# 实际看到：XDP Acceleration: Disabled / Generic
```

**Step 2：如果是 generic，关掉 XDP 加速**：

```yaml
loadBalancer:
  acceleration: disabled    # 退回 TC 模式
```

TC 模式比 generic XDP 快得多。

**Step 3：选合适的 LB 模式**：

| 模式 | 描述 | 性能 | 兼容性 |
| --- | --- | --- | --- |
| `snat` | 默认，Cilium 做 SNAT | 中 | 高 |
| `dsr` (Direct Server Return) | 后端直接回包给客户端 | 高 | 需对称路由 |
| `hybrid` | TCP 用 DSR，UDP 用 SNAT | 高 | 折中 |

DSR 模式如果上下游路由不对称（比如客户端经过 NAT），回包丢失会比 SNAT 严重得多。在云上谨慎用。

#### 验证

```bash
# 性能基线测试
kubectl run -it --rm bench --image=nicolaka/netshoot -- \
    iperf3 -c <service-ip> -P 8 -t 30

# 看 XDP 程序实际 attach 状态
ip link show eth0 xdp
bpftool net show
```

> **口诀**：XDP 加速不是"开了就有"。**先确认硬件支持，再灰度开启，对照基线看延迟和重传率**。

---

### 案例 ⑨ eBPF 在容器中的权限问题 —— "DevContainer 里跑 bpftrace 各种报错"

#### 现象

开发想在 K8s Pod 里跑 bpftrace 调试问题：

```bash
$ bpftrace -e 'kprobe:do_sys_openat2 { printf("%s\n", str(arg1)); }'
ERROR: Failed to load program: Operation not permitted
ERROR: Could not open BPF map: Permission denied
```

或者 Falco / Tetragon Pod 报 `kernel.bpf_jit_enable not enabled`、`unable to load BPF program`。

#### 根因

eBPF 操作需要的能力链：

1. **`CAP_BPF`**（5.8+ 引入，之前是 `CAP_SYS_ADMIN`）—— 加载 eBPF 程序
2. **`CAP_PERFMON`** —— 用 kprobe / perf event 类型的 eBPF
3. **`CAP_NET_ADMIN`** —— attach 到网络相关 hook
4. **`/sys/kernel/debug` mount** —— bpftrace / kprobe 需要读 debugfs
5. **`/sys/fs/bpf` mount** —— pin BPF 对象需要 bpffs
6. **`hostPID: true`**（如果要追踪宿主机所有进程）
7. **PodSecurityPolicy / PSA** —— 集群可能直接拒绝 privileged Pod

容器默认 drop 了几乎所有 capability，eBPF 直接挂。

#### 修复

**方案 A：privileged Pod（开发调试）**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bpftrace-debug
spec:
  hostPID: true
  hostNetwork: true
  containers:
  - name: bpftrace
    image: quay.io/iovisor/bpftrace:latest
    securityContext:
      privileged: true     # 开发环境用
    volumeMounts:
    - mountPath: /sys/kernel/debug
      name: sys-debug
    - mountPath: /lib/modules
      name: modules
      readOnly: true
    command: ["sleep", "infinity"]
  volumes:
  - name: sys-debug
    hostPath: { path: /sys/kernel/debug }
  - name: modules
    hostPath: { path: /lib/modules }
```

**方案 B：精确 capability（生产）**

```yaml
spec:
  containers:
  - name: tetragon
    securityContext:
      capabilities:
        add:
          - BPF
          - PERFMON
          - NET_ADMIN
          - SYS_RESOURCE     # 锁定内存（mlock）
        drop: ["ALL"]
      readOnlyRootFilesystem: true
```

**方案 C：Pod Security Admission 放行**

如果集群启用了 `restricted` PSA，需要为 namespace 放行：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: observability
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
```

**方案 D：内核参数**

宿主机可能关了 BPF JIT 或限制了非特权用户：

```bash
# 看是否禁用了 unprivileged BPF
sysctl kernel.unprivileged_bpf_disabled
# 1 = 禁用（生产推荐），0 = 允许

# JIT 必须开
sysctl net.core.bpf_jit_enable
# 应该是 1

# JIT 加固（防 spectre）
sysctl net.core.bpf_jit_harden
# 0 = 关，1 = 仅非特权，2 = 全开（影响性能）
```

#### 验证

```bash
# 测试 capability 是否生效
kubectl exec -ti bpftrace-debug -- \
    bpftrace -e 'kprobe:do_sys_openat2 { @[comm] = count(); } interval:s:5 { exit(); }'

# 看 SELinux / AppArmor 是否拦截
dmesg | grep -i "bpf\|denied"
```

> **口诀**：eBPF 在容器里的"权限烫手"问题，记三组：**capability（CAP_BPF + CAP_PERFMON + CAP_NET_ADMIN）+ 挂载（debugfs + bpffs）+ 内核参数（jit 开、unprivileged_bpf 适配）**。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 内核与节点层
- [ ] 所有节点内核 ≥ 5.10（Cilium 1.13+ 推荐）；高级特性按需求对照支持矩阵
- [ ] `CONFIG_BPF=y`、`CONFIG_BPF_SYSCALL=y`、`CONFIG_BPF_JIT=y`、`CONFIG_HAVE_EBPF_JIT=y`
- [ ] `net.core.bpf_jit_enable=1`、`kernel.unprivileged_bpf_disabled` 按安全策略设置
- [ ] `bpffs` mount 到 `/sys/fs/bpf`（systemd 节点默认开）
- [ ] BTF 文件可读（`/sys/kernel/btf/vmlinux` 存在）

### CNI 部署层
- [ ] Cilium IPAM Pod CIDR 与 Service CIDR、节点 CIDR 不冲突
- [ ] `kubeProxyReplacement` 模式确认（true / false / strict 含义不同）
- [ ] 启用 `kubeProxyReplacement` 时，确认已删除 kube-proxy DaemonSet
- [ ] `tunnel` 模式（vxlan / geneve / disabled）匹配底层网络（云网络 disabled，跨子网 vxlan）
- [ ] 多集群（ClusterMesh）规划 Cluster ID 不冲突

### eBPF Map 容量层
- [ ] `bpf.ctTcpMax` / `bpf.ctAnyMax` 按业务峰值连接数 × 2 配置
- [ ] `bpf.natMax` ≥ ctTcpMax
- [ ] `bpf.policyMapMax` 按 Endpoint 数 × 平均规则数 配置
- [ ] Map 利用率 > 80% 持续 5 分钟告警

### Cilium Agent 资源层
- [ ] 资源 limit：200 Pod/节点 → 4Gi 内存；500 Pod → 8Gi
- [ ] CPU request 至少 500m，limit 4
- [ ] PriorityClass 设为 system-node-critical
- [ ] `nodeSelector` / `tolerations` 配齐，确保所有节点都能跑

### NetworkPolicy 层
- [ ] 上线前 24h `policyEnforcementMode: audit` 看 Hubble verdict
- [ ] Policy 必须放行 `fromEntities: [host]`（livenessProbe）
- [ ] Policy 必须放行 monitoring namespace 的 metrics 抓取
- [ ] DNS（egress to kube-dns）单独写一条，避免 ns 内 deny-all 把 DNS 也封了
- [ ] 用 CiliumNetworkPolicy 而不是 K8s NetworkPolicy（功能更全）

### Hubble 层
- [ ] **不要** Flow 全量 export 到外部存储，用 metrics + 异常采样
- [ ] `hubble.metrics.enabled` 避免高基数标签（HTTP path、URL）
- [ ] Hubble UI 限制访问（不要暴露到公网）
- [ ] Relay 高可用（多副本）
- [ ] `hubble.eventBufferCapacity` 按节点流量调整

### 性能与监控层
- [ ] Prometheus 抓 `cilium_*` 和 `hubble_*` metrics
- [ ] 关键 Grafana 面板：Conntrack 利用率、Policy 拒绝率、Drop 原因分布、L7 P99
- [ ] 启用 cilium-cli 的健康检查 cronjob（每 5 分钟自检集群 connectivity）
- [ ] 节点 perf event 用量监控（避免 ringbuf 溢出）

### 升级层
- [ ] 升级前看 release notes 的 BREAKING CHANGES（Cilium 大版本经常改默认值）
- [ ] preflight check：`cilium-cli` 跑 `cilium upgrade --dry-run`
- [ ] 灰度：先升 1-2 个节点观察 24h
- [ ] 升级期间 `cilium status` 看 Endpoint 是否有 not-ready

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### Pixie —— 基于 eBPF 的 Auto-instrumentation
- **本质**：在每个 K8s 节点跑一个 PEM Agent，用 eBPF uprobe 自动 hook 应用层协议（HTTP、MySQL、PgSQL、Redis、DNS、gRPC）
- **零侵入**：不需要改业务代码，不需要 sidecar，把"应用层调用"当作内核事件抓
- **代价**：高基数数据全在节点本地，跨集群查询要走 Vizier 控制面，复杂度高
- **看什么**：Pixie 的 Stirling 数据采集器源码（`src/stirling/source_connectors/`）

### Parca —— 持续性能剖析
- **本质**：每节点低频（19Hz）抓所有进程的栈，eBPF 在内核态聚合，每 10s 上报一次到 Parca Server
- **省心点**：用 perf event 类型的 eBPF，自动支持任何编译器产物
- **DWARF unwinder**：Parca Agent 用 eBPF 实现 DWARF 栈展开，省掉了应用必须有 frame pointer 的限制
- **看什么**：Parca Agent 的 `bpf/cpu/cpu.bpf.c` —— 在内核里做栈展开是最近两年 eBPF 最复杂的工程之一

### Aya —— 用 Rust 写 eBPF
- **背景**：libbpf（C）和 BCC（Python+C）写 eBPF 都要懂 C 和内核 API，门槛高
- **Aya** 是纯 Rust 实现的 BPF loader + helper，用 Rust 写 eBPF 程序（用 `aya-bpf` crate）
- **优势**：类型安全（Rust 的 trait 包了 BPF Helper）、编译期检查、单一语言栈
- **看什么**：Aya 的 [book.aya-rs.dev](https://aya-rs.dev) 教程；与 libbpf 对比的是开发体验，不是性能

### Tetragon —— 运行时安全
- **本质**：用 eBPF LSM hook + tracepoint 监控容器内的所有"敏感操作"（exec、网络连接、文件访问）
- **能做什么**：检测容器逃逸、未授权 exec、可疑网络连接，并能**实时阻断**（kill 进程）
- **vs Falco**：Falco 在用户态规则匹配（开销大），Tetragon 在内核态决策（μs 级响应）
- **看什么**：TracingPolicy CRD 设计；Tetragon 与 Cilium 共享 Identity 体系

### Calico 的 eBPF 数据面
- **历史**：Calico 一开始全 iptables，后来加了 eBPF dataplane 模式
- **vs Cilium**：Calico eBPF 仍然依赖 conntrack（Linux 原生）做连接跟踪，Cilium 自己实现 conntrack
- **选择**：已经用 Calico 的，eBPF 模式是平滑升级；新集群一般直接选 Cilium

### WASM 在 eBPF 中的探索
- **背景**：eBPF 字节码 ISA 是为内核执行设计的，可移植性差；WASM 是用户态字节码标准
- **方向 1**：把 WASM 编译成 eBPF（`wasm-to-bpf`）—— 实验性
- **方向 2**：在 eBPF 里嵌入 WASM 解释器（`bpfwasm`）—— 用户态扩展点用 WASM
- **看什么**：CNCF Wasm + eBPF SIG 的近期讨论；当前还不成熟，3 年内不要赌生产

### eBPF for Windows
- **背景**：微软 2021 启动，把 Linux eBPF API 移植到 Windows 内核
- **架构**：用 PREVAIL 替代 Linux Verifier，uBPF / WinDivert 替代内核执行
- **现状**：能跑大部分网络相关 eBPF 程序，但和 Linux 不是 ABI 兼容
- **意义**：eBPF 不再绑死 Linux，长期可能成为"跨内核的可编程标准"

### XDP 极致优化方向
- **AF_XDP**：直接把 XDP 处理过的包零拷贝到用户态 socket，跳过内核协议栈，常用于 DPDK 替代
- **XDP Hints**：5.18+ 让网卡硬件元数据（VLAN、checksum、RSS hash）能在 XDP 程序里直接访问
- **eXpress Bridge**：用 XDP 实现 L2 桥接，性能 10x 于 Linux bridge

---

## 7. 引用

- Cilium 官方文档：[docs.cilium.io](https://docs.cilium.io)
- eBPF 入门权威：[ebpf.io](https://ebpf.io) + Brendan Gregg 的 *BPF Performance Tools*
- libbpf：[github.com/libbpf/libbpf](https://github.com/libbpf/libbpf)
- 论文：Alexei Starovoitov, *eBPF: From a Programmable Filter to a Universal In-Kernel Virtual Machine*（LPC 2014）
- 论文：Daniel Borkmann, *On getting tc classifier fully programmable with cls_bpf*（NetDev 1.1）
- Cilium 源码：[github.com/cilium/cilium](https://github.com/cilium/cilium)（Go 用户态 + C 内核态）
- Hubble：[github.com/cilium/hubble](https://github.com/cilium/hubble)
- bpftrace 一行流：[github.com/bpftrace/bpftrace/blob/master/docs/tutorial_one_liners.md](https://github.com/bpftrace/bpftrace/blob/master/docs/tutorial_one_liners.md)
- Liz Rice, *Learning eBPF*（2023）—— 最新的 eBPF 入门书
- Linux Kernel BPF 文档：[www.kernel.org/doc/html/latest/bpf/](https://www.kernel.org/doc/html/latest/bpf/)
- Tetragon：[github.com/cilium/tetragon](https://github.com/cilium/tetragon)
- Aya：[aya-rs.dev](https://aya-rs.dev)

---

> 配套：[Kubernetes 实战](/article/k8s/k8s-essentials) | [Service Mesh](/article/k8s/service-mesh) | [Linux 火焰图](/article/linux-perf/flame-graph) | [TCP 状态机](/article/network/tcp-state)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
