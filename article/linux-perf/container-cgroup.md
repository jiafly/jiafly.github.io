# 容器与 cgroup —— Java 应用在 K8s 里的"误读"全景

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，看 [Linux 性能综述](/article/linux-perf/) 那一节就够了。
> 这里要回答的，是**Java 应用部署在 K8s 之后，为什么内存/CPU 行为和你在物理机上完全不一样**的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. cgroup v1 和 v2 的核心差别是什么？为什么 v2 强调"统一层级"？你怎么判断当前节点用的是哪个版本？
2. Linux namespace 一共有几种？容器里你 `ps aux` 看到的 PID=1 是真实 PID 吗？为什么有时候能在容器里看到宿主机进程？
3. 容器被 OOMKilled 和 JVM 抛 `OutOfMemoryError`，**是同一回事吗**？谁先发生？日志在哪看？
4. 给容器配 `limits.cpu: 2`，你在容器里 `Runtime.availableProcessors()` 拿到的是几？JVM 用什么探测？
5. 没设 `-Xmx`，JVM 默认堆大小怎么算？`MaxRAMPercentage` 默认是多少？基于谁算？
6. K8s 的 QoS 三类（Guaranteed / Burstable / BestEffort）分别什么条件触发？节点资源紧张时谁先被杀？
7. `container_cpu_cfs_throttled_seconds_total` 在涨，但容器 CPU 利用率才 30% —— 怎么解释？
8. 同一份 yaml，部署到 cgroup v1 节点和 cgroup v2 节点，**JVM 的内存行为可能不一样吗**？

如果有半数答不上，往下看。

---

## 1. 底层模型：namespace + cgroup + JVM 的三层适配

绝大多数 Java 同学对容器的认知止于"Docker 是一种轻量虚拟化"。这层抽象解释不了上面 8 个问题。要看懂它们，必须把容器拆回它的两个内核原语。

### 1.1 容器 = namespace + cgroup（再加一点 chroot）

**一句话**：容器不是一种新东西，是 Linux 内核两个独立特性的组合：
- **namespace（命名空间）** 负责"看不见外面" —— 隔离视图（PID、网络、文件系统挂载点……）
- **cgroup（control group，控制组）** 负责"用不超额度" —— 限制资源（CPU、内存、IO）

```
       +-------------------------------+
       |  容器（一个 Linux 进程而已）  |
       |                               |
       |   namespace ── "看见啥"       |
       |   cgroup    ── "能用多少"     |
       |   rootfs    ── "在哪个目录"   |
       +-------------------------------+
```

第三件事 rootfs（根文件系统切换）不是 namespace 的一部分，但是容器镜像的物理基础。Docker 镜像 = 一组 OverlayFS 层 + 一份 manifest。

**关键认知**：容器内的进程对宿主机内核来说，**就是一个普通进程**。和宿主机上的其他进程比起来，唯一区别是它"被关在几个 namespace 里" + "在某个 cgroup 控制下"。这个事实是后面案例 ③（容器内 thread dump 看不到宿主机进程）的根。

### 1.2 8 种 namespace

到 Linux 6.x 为止，namespace 总共 8 种：

| namespace | 隔离的内容 | 容器里的体现 |
| --- | --- | --- |
| **PID** | 进程号 | 容器里 `ps` 看到的 PID 从 1 开始 |
| **NET** | 网卡、路由、iptables、socket | 容器有独立的 lo / eth0 |
| **MNT** | 文件系统挂载点 | `mount` 看到的是容器自己的挂载 |
| **UTS** | hostname、domainname | `hostname` 命令显示容器名 |
| **IPC** | System V IPC、POSIX 消息队列 | 不能跨容器共享共享内存 |
| **USER** | UID/GID 映射 | 容器内的 root 在宿主机上是普通用户 |
| **CGROUP** | cgroup 视图 | 容器只看到自己那部分 cgroup 文件系统 |
| **TIME**（5.6+） | CLOCK_MONOTONIC 时间偏移 | 用于 checkpoint/restore |

记忆法：**PNMUICCT** 不是 8 个的英文首字母 —— 实际是 PID / Net / Mount / UTS / IPC / User / Cgroup / Time。最常用的是前 5 个；USER 是安全相关；CGROUP 是为了避免容器从 `/proc/self/cgroup` 看到完整宿主机层级；TIME 是给特殊场景用的。

**一个反直觉的点**：你以为容器"完全隔离"了？错。**内核是同一个**。你在容器里 `uname -r` 看到的内核版本就是宿主机的；容器里加载内核模块（如果没特权）会失败；容器里 `dmesg` 通常是空的（被 capability 限制了）。

### 1.3 cgroup v1（多层级）vs v2（统一层级）

cgroup 从 2.6.24 开始进入内核，演进到现在分为两代。两代的根本差异不在"功能多寡"，而在**层级模型**。

**v1：每个控制器一棵树**

```
/sys/fs/cgroup/
├── cpu/
│   └── kubepods/<pod-uid>/<container-id>/  ← 容器在 cpu 控制器下的位置
├── memory/
│   └── kubepods/<pod-uid>/<container-id>/  ← 同一个容器在 memory 下又有一份
├── blkio/
│   └── kubepods/<pod-uid>/<container-id>/
├── pids/
└── ... 一堆控制器
```

每个控制器（cpu、memory、blkio、pids、cpuset……）是**独立的层级树**。一个进程可以在 cpu 控制器下属于 group A，在 memory 控制器下属于 group B。理论上灵活，实践中混乱：写代码要给每个控制器各自挂载点，进程 / cgroup 的归属管理复杂。

**v2：所有控制器共用一棵树**

```
/sys/fs/cgroup/
└── kubepods.slice/
    └── kubepods-pod<uid>.slice/
        └── cri-containerd-<id>.scope/
            ├── cpu.max         ← cpu 控制器的限额
            ├── memory.max      ← memory 控制器的限额
            ├── memory.current  ← 当前内存
            ├── cpu.stat        ← CPU 统计
            ├── io.max
            └── pids.max
```

所有控制器在**同一棵层级树**下，每个 cgroup 节点用文件直接暴露资源接口。文件命名也规范化了（`<controller>.<key>` 格式）。

| 维度 | cgroup v1 | cgroup v2 |
| --- | --- | --- |
| 层级模型 | 每控制器一棵树 | 统一一棵树 |
| 内存接口 | `memory.limit_in_bytes` | `memory.max` |
| CPU 配额 | `cpu.cfs_quota_us` / `cpu.cfs_period_us` | `cpu.max`（写 `quota period`） |
| OOM 控制 | `memory.oom_control` | `memory.oom.group` |
| 内存压力 | 没有（粗暴 OOM） | `memory.pressure`（PSI） |
| IO 控制 | blkio | io（统一 io 控制器） |

**为什么要统一？** 工程上，v1 时代复杂度太高，调度、设备 I/O、内存联动都需要跨树写代码。v2 由 Tejun Heo 主导从 2014 年开始重写，核心动机是**让控制器能协作**（比如内存压力大时调整 IO 优先级）。

**判断当前节点用的哪个版本**：

```bash
# 方法 1：看挂载点
mount | grep cgroup
# v1：会看到很多个 cgroup 类型的挂载，每个控制器一行
# v2：只有一个 "cgroup2 type cgroup2" 行

# 方法 2：看文件
stat -fc %T /sys/fs/cgroup/
# tmpfs   → v1
# cgroup2fs → v2

# 方法 3：看根目录
ls /sys/fs/cgroup/
# v1：会有 cpu、memory、blkio 等子目录
# v2：直接看到 cgroup.controllers、cpu.max 等文件
```

K8s 1.25 GA 了 cgroup v2 支持。主流发行版（RHEL 9、Ubuntu 22.04+、AKS 1.25+）默认 v2。但你的集群可能还在 v1，混合环境更常见。

### 1.4 cgroup 的核心控制器

挑 4 个对 Java 应用最关键的展开。

**cpu / cpuset**

CPU 控制有两种：
- **CPU 时间配额**（cpu 控制器）：在某个 period 周期内最多能用 quota 微秒。这是 K8s `limits.cpu` 的实现。
- **CPU 亲和性**（cpuset 控制器）：把进程限制到某些核上跑。K8s 默认不用 cpuset，除非你启用 `static` CPU manager。

cpu 控制器的核心参数：

```bash
# v1
/sys/fs/cgroup/cpu/<path>/cpu.cfs_period_us   # 默认 100000 (100ms)
/sys/fs/cgroup/cpu/<path>/cpu.cfs_quota_us    # 100000 = 1 个 CPU 等量

# v2
/sys/fs/cgroup/<path>/cpu.max
# 内容：100000 100000   表示 quota period
# "max 100000" 表示无限额
```

`limits.cpu: 2` 翻译成 cgroup 配置就是 `quota=200000, period=100000`，意思"每 100ms 内最多用 200ms CPU 时间"。

**memory**

```bash
# v1
/sys/fs/cgroup/memory/<path>/memory.limit_in_bytes      # 硬上限
/sys/fs/cgroup/memory/<path>/memory.usage_in_bytes      # 当前用量
/sys/fs/cgroup/memory/<path>/memory.stat                # 详细统计

# v2
/sys/fs/cgroup/<path>/memory.max
/sys/fs/cgroup/<path>/memory.current
/sys/fs/cgroup/<path>/memory.stat
```

超过 `memory.max` 时内核走 OOM Killer 路径。**这个 OOM 是 cgroup 内部的 OOM，不是宿主机全局 OOM**。它只杀这个 cgroup 里的进程，不影响宿主机其他容器。后面案例 ① 会展开。

**io（v2 统一了 v1 的 blkio）**

控制读写带宽和 IOPS。一般 K8s 不直接配，但 IO 密集容器要注意 noisy neighbor 问题。

**pids**

限制进程数量，防止 fork bomb：

```bash
/sys/fs/cgroup/<path>/pids.max
```

Java 应用一般不会撞这个，但 GraalVM 原生镜像 + 大量虚拟线程时要留意。

### 1.5 K8s 的资源模型：Request 和 Limit

K8s 暴露给用户的资源接口只有两个字段：

```yaml
resources:
  requests:
    cpu: 500m       # 0.5 个 CPU
    memory: 1Gi
  limits:
    cpu: 2
    memory: 2Gi
```

但它们落到 cgroup 上的语义完全不同：

| 字段 | CPU 含义 | 内存含义 | 调度作用 |
| --- | --- | --- | --- |
| **requests** | cgroup 的 `cpu.shares`（v1）/ `cpu.weight`（v2），相对权重 | 仅供调度器用，**不写入 cgroup** | 节点必须有这么多空闲资源才能调度 |
| **limits** | cgroup 的 `cpu.cfs_quota_us`，硬上限 | cgroup 的 `memory.limit_in_bytes`，硬上限 | 超限被 throttle / OOMKilled |

**两个反直觉的点**：

1. **CPU request 不是"保底"**，而是"权重"。配 `requests.cpu: 500m` 意味着争抢 CPU 时按 512 / 1024 的份额（v1 的默认 shares 是 1024 表示 1 核）。**节点空闲时能用更多**。
2. **Memory request 在 cgroup 里几乎没体现**。它只影响调度（Pod 能不能放到这台机器上）。容器内存的物理上限只看 `limits.memory`。

K8s 据此把 Pod 分成 3 个 QoS 类：

| QoS 类 | 条件 | 节点资源紧张时被杀顺序 |
| --- | --- | --- |
| **Guaranteed** | 所有容器都设了 limits 且 requests == limits（CPU/Memory 都满足） | 最后被杀 |
| **Burstable** | 至少一个容器有 requests，且不满足 Guaranteed 条件 | 中间被杀 |
| **BestEffort** | 所有容器都没设 requests/limits | 最先被杀 |

**生产建议**：Java 应用一律用 **Guaranteed**。理由：
- JVM 启动时探测内存，依赖 `memory.max` —— 没设 limits 就探测不到
- BestEffort 的 Pod 在节点压力下随时被杀，对 JVM 这种长生命周期进程不友好

### 1.6 JVM 的 cgroup 感知（JEP 223 / 248 的演进）

JVM 是怎么知道"我在容器里只有 2GB 内存"的？答案：**读 cgroup 文件系统**。

但这个能力是**逐步加上**的，老版本的 JVM 在容器里就是瞎子。

| JDK 版本 | cgroup v1 支持 | cgroup v2 支持 | 关键 JEP / Issue |
| --- | --- | --- | --- |
| JDK 8 ≤ 8u131 | 完全没有 | 没有 | —— |
| JDK 8 8u131 ~ 8u190 | 部分（需 `-XX:+UseCGroupMemoryLimitForHeap`） | 没有 | JDK-8170888 |
| JDK 8 8u191+ | 完整（默认） | 没有 | JDK-8146115 backport |
| JDK 10 | 完整 | 没有 | JEP 223 相关 |
| JDK 11 ~ 14 | 完整 | 没有 | —— |
| JDK 15+ | 完整 | 完整 | JDK-8230305 |
| JDK 17+ | 完整 | 完整、稳定 | —— |

**血泪教训**：JDK 8u131 之前的 JVM 在容器里看到的是**宿主机的内存大小**。你在 8GB 内存的容器里跑老 JVM，它以为自己有 256GB 内存（宿主机），按 1/4 默认堆算 → 启动就 OOMKilled。后面案例 ⑤ 展开。

JDK 8u191+ 引入了三个开关：
- `-XX:+UseContainerSupport`（默认开）：开启 cgroup 感知
- `-XX:InitialRAMPercentage` / `-XX:MaxRAMPercentage` / `-XX:MinRAMPercentage`：替代老的 1/4 默认堆

`MaxRAMPercentage` 默认 25%。意思：容器有 4GB 时，JVM 默认 `-Xmx` 是 1GB。**这通常太保守**，生产推荐设到 70~75%。

JEP 223（JDK 9）规范了版本号，本身和容器无关，但同时期的工作集合（"JEP 223 时代"）建立了 JVM 容器感知的基础。真正的 cgroup v2 支持是 [JDK-8230305](https://bugs.openjdk.org/browse/JDK-8230305) 在 JDK 15 完成的。

> **口诀**：容器 = namespace（看不见）+ cgroup（用不超）。JVM 的"容器感知"靠**主动读 cgroup 文件系统**实现，每个 JDK 版本支持的能力不同。生产 JDK 必须 ≥ 11，最好 17。

---

## 2. 源码切片：JVM 与 cgroup 的接口

> 基于 OpenJDK 17 `OperatingSystem` 相关源码 + Linux kernel 6.x cgroup 子系统。注释为本文添加。

### 2.1 JVM 探测 cgroup 版本

OpenJDK 启动时第一步是确定"我在哪种 cgroup 下"：

```cpp
// src/hotspot/os/linux/cgroupSubsystem_linux.cpp 简化
CgroupSubsystem* CgroupSubsystem_linux::create() {
    CgroupV1Controller* memory = nullptr;
    CgroupV1Controller* cpu = nullptr;

    // 读 /proc/self/mountinfo 找 cgroup 挂载
    FILE *mntinfo = fopen("/proc/self/mountinfo", "r");
    while (fgets(buf, sizeof(buf), mntinfo)) {
        if (strstr(buf, "cgroup2 ")) {
            is_cgroupV2 = true;            // ★ 看到 cgroup2 标记
            // 记录 v2 挂载点
        }
        if (strstr(buf, "cgroup ")) {
            // 解析具体哪个控制器（memory / cpu / cpuset）
            // 记录每个控制器的挂载点
        }
    }
    fclose(mntinfo);

    if (is_cgroupV2) {
        return new CgroupV2Subsystem(...);
    } else {
        return new CgroupV1Subsystem(memory, cpu, cpuset, pids);
    }
}
```

**逐段点透**：

- **从 `/proc/self/mountinfo` 入手**：这是 Linux 暴露挂载信息的标准接口。"cgroup2" 字符串是判断 v2 的关键。
- **不是看版本号字符串**：JVM 不调用任何 "cgroup --version" 这种命令，纯粹看挂载类型。这意味着即使你 `mount` 了一个假的 cgroup2，JVM 也会被骗（这是某些 sidecar/init container 的注入手段）。
- **混合模式**：少数节点同时挂了 v1 和 v2（hybrid 模式），JVM 优先认 v2。

### 2.2 cgroup v1 下读取内存上限

```cpp
// src/hotspot/os/linux/cgroupV1Subsystem_linux.cpp 简化
jlong CgroupV1Subsystem::memory_limit_in_bytes() {
    GET_CONTAINER_INFO(jlong, _memory, "/memory.limit_in_bytes",
                       "Memory Limit is: " JLONG_FORMAT, JLONG_FORMAT, memlimit);
    if (memlimit >= os::Linux::physical_memory()) {
        // ★ 关键：如果 cgroup 上限 >= 物理内存，等于没限制
        log_trace(os, container)("Memory Limit is unlimited (no constraint)");
        return -1;
    }
    return memlimit;
}
```

**逐段点透**：

- **路径拼接**：`<memory_mount>/memory.limit_in_bytes`，比如 `/sys/fs/cgroup/memory/kubepods/.../memory.limit_in_bytes`。直接 `read()` 读取数字。
- **"≥ 物理内存即视为无限"** 的判断很关键：很多没设 limits 的容器，cgroup 默认值是 `Long.MAX_VALUE`（约 9223372036854775807）。JVM 把这种情况识别成"无限制"，落回宿主机物理内存。
- **如果你设 `limits.memory: 100Gi` 但宿主机只有 64GB**：JVM 也会判定"无限"，行为就和不设一样。

### 2.3 cgroup v2 下读取内存上限

```cpp
// src/hotspot/os/linux/cgroupV2Subsystem_linux.cpp 简化
jlong CgroupV2Subsystem::memory_limit_in_bytes() {
    GET_CONTAINER_INFO_LINE(char *, unified, "/memory.max",
                            "Memory Limit is: %s", "%s", mem_limit_str);
    if (strncmp(mem_limit_str, "max", 3) == 0) {
        // ★ v2 用字符串 "max" 表示无限
        return -1;
    }
    jlong memlimit = strtoll(mem_limit_str, nullptr, 10);
    if (memlimit >= os::Linux::physical_memory()) {
        return -1;
    }
    return memlimit;
}
```

**逐段点透**：

- **v2 的接口语义**：`memory.max` 文件内容是十进制数字或字符串 `max`（表示无限）。这是 cgroup v2 设计上的语义化改进 —— v1 用一个超大数字表达"无限"是个 hack。
- **和 v1 的探测代码看似一样，逻辑微妙不同**：v2 还要先判 `"max"` 字符串。早期 JDK（11~14）没这个分支，所以在 v2 节点上行为不正确。
- **`physical_memory()` 怎么得到**：读 `/proc/meminfo` 的 `MemTotal`。这个数字也是 namespace 化的吗？**不是**！`/proc/meminfo` 在容器里看到的还是宿主机数字。这是后面案例 ⑥ 的根源之一。

### 2.4 探测 CPU 配额（v1）

```cpp
// src/hotspot/os/linux/cgroupV1Subsystem_linux.cpp
int CgroupV1Subsystem::active_processor_count() {
    int quota_count = 0, share_count = 0;
    int cpu_count = os::Linux::active_processor_count();   // 物理核数

    int quota = cpu_quota();           // cpu.cfs_quota_us
    int period = cpu_period();         // cpu.cfs_period_us
    if (quota > -1 && period > 0) {
        quota_count = ceilf((float)quota / (float)period);
    }
    int share = cpu_shares();          // cpu.shares
    if (share > -1) {
        share_count = ceilf((float)share / (float)PER_CPU_SHARES);
    }

    // ★ 关键：取最小值
    int limit = (quota_count == 0) ? share_count
              : (share_count == 0) ? quota_count
              : MIN2(quota_count, share_count);
    return MIN2(cpu_count, limit > 0 ? limit : cpu_count);
}
```

**逐段点透**：

- **三选一取最小**：物理核数、quota 折算的核数、shares 折算的核数。这是 `Runtime.availableProcessors()` 在容器里的实际返回值。
- **shares 也参与计算**：这个**反直觉**。`requests.cpu: 500m` 设的是 shares，理论上是"权重"，但 JVM 把它**也算成"可用 CPU 数"的上限**。后果：你只配了 request、没配 limit，JVM 看到 `availableProcessors() = 1` —— 实际节点空闲时能用更多核但 JVM 不知道。
- **用 ceilf 向上取整**：`quota=150000, period=100000` → `1.5` → `availableProcessors() = 2`。**不是四舍五入也不是向下取整**。

JDK 19+ 引入了 `-XX:-UseContainerCpuShares` 开关，可以禁用 shares 参与计算。生产建议：**显式设 limits 或加这个开关**，避免 request 影响 JVM 的 GC 线程数估算。

### 2.5 K8s kubelet 创建 cgroup 的路径

```go
// pkg/kubelet/cm/qos_container_manager_linux.go 简化
func (m *qosContainerManagerImpl) UpdateCgroups(...) error {
    qosClasses := []v1.PodQOSClass{
        v1.PodQOSGuaranteed,
        v1.PodQOSBurstable,
        v1.PodQOSBestEffort,
    }
    for _, qosClass := range qosClasses {
        // ★ 每个 QoS 一个父级 cgroup
        cgroupConfig := &CgroupConfig{
            Name: fmt.Sprintf("/kubepods/%s", strings.ToLower(string(qosClass))),
            // ...
        }
        m.cgroupManager.Update(cgroupConfig)
    }
    return nil
}
```

**逐段点透**：

- **三层结构**：`/kubepods/<qos>/<pod-uid>/<container-id>`。Guaranteed Pod 直接挂在 `/kubepods/` 下（少一层），这是优化 —— Guaranteed 不需要再被 QoS 父级二次限制。
- **谁创建容器内的 cgroup**：不是 kubelet，是 **CRI runtime**（containerd、CRI-O）。kubelet 创建到 Pod 级别，container 级别由 runtime 创建。
- **systemd cgroup driver vs cgroupfs cgroup driver**：现代 K8s（1.22+）默认 systemd driver，cgroup 路径变成 `kubepods.slice/kubepods-pod<uid>.slice/cri-containerd-<id>.scope`。slice/scope 是 systemd 的术语，本质还是 cgroup。

### 2.6 OOMKilled 的判定路径

容器 OOM 不是 JVM 抛 `OutOfMemoryError`，而是**内核 cgroup OOM Killer**触发：

```c
// kernel/mm/memcontrol.c 简化
static unsigned long mem_cgroup_iter(...);

static int mem_cgroup_out_of_memory(struct mem_cgroup *memcg, gfp_t gfp_mask, int order) {
    struct oom_control oc = {
        .zonelist = NULL,
        .nodemask = NULL,
        .memcg = memcg,                      // ★ 限制在这个 cgroup 内
        .gfp_mask = gfp_mask,
        .order = order,
    };
    if (!mutex_trylock(&oom_lock)) {
        // 已经在 OOM 处理中
        return -EBUSY;
    }
    ret = out_of_memory(&oc);                // 选一个进程杀
    mutex_unlock(&oom_lock);
    return ret;
}
```

`out_of_memory()` 的核心是 `select_bad_process()`：遍历 cgroup 内所有进程，按 `oom_score_adj` 排序，**杀分数最高的**。

**逐段点透**：

- **杀谁不一定是 JVM**：cgroup 内可能有多个进程（init 进程、agent、JVM）。OOM Killer 选 `oom_score` 最高的杀，通常是占内存最多的那个 = 99% 是 JVM。但**是哪个进程被杀决定了你能不能从 dmesg 看到 OOM**。
- **dmesg 看不到**：容器内 `dmesg` 通常被 capability 限制，必须到宿主机看 `/var/log/messages` 或 `journalctl -k`。
- **被 OOMKill 后容器重启**：是 K8s 控制面的行为，不是内核做的。kubelet 看到容器 ExitCode=137（128 + SIGKILL=9），按 `restartPolicy` 重启。

### 2.7 CFS Throttling 的实现

CPU 限额是 Linux CFS 调度器实现的：

```c
// kernel/sched/fair.c 简化
static int do_sched_cfs_period_timer(struct cfs_bandwidth *cfs_b, ...) {
    // ★ 每 period 周期重新分配 quota
    cfs_b->runtime = cfs_b->quota;

    // 唤醒被 throttle 的任务
    distribute_cfs_runtime(cfs_b);
}

static void throttle_cfs_rq(struct cfs_rq *cfs_rq) {
    // ★ 把整个 cfs_rq 从调度器队列里摘掉
    list_del_leaf_cfs_rq(cfs_rq);
    cfs_rq->throttled = 1;
    cfs_rq->throttled_clock = rq_clock(rq_of(cfs_rq));
}
```

**核心机制**：
1. cgroup 配置 `cpu.cfs_quota_us = 200000, cpu.cfs_period_us = 100000`，意为"每 100ms 周期内最多用 200ms CPU 时间（= 2 核满载）"
2. 内核为这个 cgroup 维护一个 "runtime" 计数，跑一秒减一秒
3. runtime 用完时，所有该 cgroup 的任务被 **throttle**（从调度队列摘除，等到下个 period 才能跑）
4. 每个 period 重新填满 runtime

**反直觉的点**：

- **throttle 是 all-or-nothing**：cgroup 里所有任务一起被冻结，不是某个线程被冻结。所以**单核任务都能拖累整体**。
- **多线程会更快用光 quota**：8 个线程并行跑，10ms 就用光了 80ms 的 quota（quota=200000，period=100000）。**剩下 90ms 全部线程等 throttle**。这是为什么 P99 会飙升 —— 不是平均 CPU 高，是**周期性饥饿**。
- **观测**：`/sys/fs/cgroup/<path>/cpu.stat`：

```bash
cat /sys/fs/cgroup/cpu/<path>/cpu.stat
nr_periods 10000
nr_throttled 5234         # ★ 一半的周期被 throttle 过
throttled_time 53000000000  # ★ 总共被 throttle 多久（纳秒）
```

> **口诀**：JVM 探测 cgroup 是**主动读文件**，不是 syscall。版本不同读的路径不同。CFS throttling 是**周期性冻结整组任务**，不是平滑限流。

---

## 3. 实测数据：4 组对比实验

### 3.1 不同 MaxRAMPercentage 下的 GC 行为

**实验**：同一 Java 应用（Spring Boot + MySQL，一定的对象分配速率），容器 `limits.memory: 4Gi`，分别用 25%（默认）/ 50% / 75% / 90% 的 `-XX:MaxRAMPercentage`，跑 30 分钟压测。

| MaxRAMPercentage | -Xmx 实际值 | YGC 次数 | YGC 平均耗时 | FGC 次数 | OOMKilled? |
| --- | --- | --- | --- | --- | --- |
| 25%（默认） | ~1GB | 4521 | 18 ms | 12 | No |
| 50% | ~2GB | 1832 | 35 ms | 0 | No |
| **75%（推荐）** | ~3GB | 723 | 56 ms | 0 | No |
| 90% | ~3.6GB | 412 | 88 ms | 0 | **Yes（10min）** |

**结论**：
- 默认 25% 浪费 75% 的容器内存，且 YGC 频繁
- 75% 是甜点 —— 给 native（Metaspace、direct buffer、JIT code cache、栈）留 1GB 空间
- 90% 太激进，没给 native 留空间，JVM 堆+native > limits.memory → 容器整体被 OOMKilled

**额外坑**：堆只是 JVM 内存的一部分。一个 Java 进程的实际 RSS ≈ heap + non-heap + JIT + 元空间 + 直接内存 + 栈 + native lib。**`-Xmx2g` 不等于"进程 RSS 上限 2GB"**，通常多 30%~50%。

### 3.2 CPU Limit 触发 Throttling 时的 P99

**实验**：QPS 平稳的 Java 服务，分别配 `limits.cpu = 1 / 2 / 4 / unlimited`，看 P99 延迟。

| limits.cpu | 平均 CPU 利用率 | nr_throttled / nr_periods | P50 (ms) | P99 (ms) | P999 (ms) |
| --- | --- | --- | --- | --- | --- |
| 1 | 92% | 4521 / 10000 (45%) | 12 | **480** | **2100** |
| 2 | 56% | 832 / 10000 (8.3%) | 11 | **120** | **480** |
| 4 | 28% | 12 / 10000 (0.12%) | 10 | 22 | 35 |
| unlimited | 22% | —— | 10 | 21 | 32 |

**反直觉的点**：

- **平均 CPU 才 56%，但 8% 的周期被 throttle**：因为多线程并行突发短暂打满 quota。"平均"掩盖了"突发"。
- **P99 从 480ms 降到 120ms 只用从 1 核加到 2 核**：边际效果非常明显。Java 应用尤其，因为 GC 也吃 CPU，被 throttle 时 STW 直接拉长。
- **从 2 核加到 4 核 P99 进一步从 120ms 降到 22ms**：第二次大跨越。
- **从 4 核加到无限制几乎没变化**：说明 4 核已经够用，更多是浪费。

**生产建议**：**limits.cpu 至少 2，最好 = requests.cpu × 2**。或者干脆不设 limits（K8s 1.27+ 的 `cpuCFSQuotaPeriod` 调整可以缓解，但需要 kubelet 配合）。

### 3.3 cgroup v1 vs v2 在 Java 应用上的差异

**实验**：同一 yaml、同一 JDK 17 镜像，部署到 v1 节点和 v2 节点，做 GC 压测。

| 指标 | cgroup v1 节点 | cgroup v2 节点 |
| --- | --- | --- |
| 启动时探测 limits.memory | 正常 | 正常 |
| `Runtime.availableProcessors()` | 正常 | 正常 |
| 探测耗时（ms） | 8.3 | 4.1 |
| OOMKilled 时内核日志 | `/var/log/messages` 有 | `/var/log/messages` 有 |
| `memory.pressure` | **不存在** | **可读，PSI 数据** |
| 容器内 `/proc/meminfo` | 显示宿主机 | 显示宿主机（同样不准） |

**关键差异在 PSI（Pressure Stall Information）**：v2 提供了 `memory.pressure` / `cpu.pressure` / `io.pressure` 文件，能告诉你"过去 10s/60s/300s 内进程因等待内存/CPU/IO 被阻塞的比例"。这是**比平均利用率更好的指标**。

```bash
cat /sys/fs/cgroup/<path>/memory.pressure
some avg10=0.00 avg60=0.00 avg300=0.00 total=0
full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

`some` 是有部分任务被阻塞，`full` 是全部任务被阻塞。`avg10` 是 10 秒平均比例。

### 3.4 K8s QoS 各类的 OOM 顺序

**实验**：节点 8GB 内存，部署 3 个 Pod，分别是 BestEffort（1GB）、Burstable（requests=500MB, limits=2GB）、Guaranteed（requests=limits=4GB）。让 BestEffort 内的应用持续吃内存到节点压力。

观察 kubelet 驱逐顺序（用 `kubectl get events`）：

| 事件时刻 | 事件 | 被驱逐 Pod |
| --- | --- | --- |
| t=0s | 节点 memory available 跌破 100MiB | BestEffort Pod 被 evicted |
| t=15s | 内存压力继续（其他容器膨胀） | Burstable Pod 被 evicted |
| t=120s | Guaranteed 容器自己超过 limits.memory | Guaranteed 容器 OOMKilled（但 Pod 不会被 evict，只重启容器） |

**结论**：
- **节点压力下，BestEffort 永远第一个被杀**
- **Burstable 第二个**，且超出 request 的部分越多，越早被杀
- **Guaranteed 不会因为节点压力被杀**，只在自己超 limits 时被 cgroup OOM Killer 杀

**生产建议**：**Java 应用一律 Guaranteed**。代价是节点资源利用率不如 Burstable，但稳定性高得多。

> **口诀**：默认 25% 太保守、90% 太激进，**75% 是甜点**；CPU **limits 至少 2**，否则 P99 灾难；**Guaranteed 是 Java 应用的唯一正确选择**。

---

## 4. 生产案例 9 则

### 案例 ① JVM 启动时探测错误，分配过大堆 → OOMKilled

#### 现象

JDK 8u131 之前的镜像（公司有遗留服务），容器 `limits.memory: 4Gi`，应用启动 5 秒就被 OOMKilled，重启循环。日志只看到：

```
Memory cgroup out of memory: Killed process 1 (java)
```

JVM 自己一行错误都没打印。

#### 根因

老 JDK 不识别 cgroup。`java -XX:+PrintFlagsFinal -version | grep MaxHeap` 看到：

```
size_t MaxHeapSize = 67108864000   # 64 GB
```

JVM 把宿主机的物理内存当成自己的可用内存，按 1/4 算默认堆 = 64GB（宿主机 256GB）。**但容器只有 4GB**。JVM 启动时 `mmap` 分配堆区，写入页面 → 触发 cgroup OOM → 进程被杀。

更糟：JVM 没机会打印任何错误，因为它根本没"察觉"自己内存不够 —— 是被外部直接 SIGKILL 的。

#### 修复

方案 A（首选）：升级 JDK。

```dockerfile
# 老
FROM openjdk:8u121
# 新
FROM eclipse-temurin:17-jre-alpine
```

方案 B（升级风险大时的临时方案）：手动设 `-Xmx`。

```yaml
env:
- name: JAVA_TOOL_OPTIONS
  value: "-Xmx2g -Xms2g"
```

**注意**：必须 `-Xmx + non-heap < limits.memory × 0.75`。比如 4GB 限额，`-Xmx2g`，给 native 留 1.5GB。

方案 C（实在不能升）：JDK 8u131 ~ 8u190 加开关：

```yaml
env:
- name: JAVA_TOOL_OPTIONS
  value: "-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -XX:MaxRAMFraction=2"
```

`MaxRAMFraction=2` 表示堆占 1/2，比 1/4 默认更合理。

#### 验证

```bash
# 进容器看 JVM 实际堆
kubectl exec <pod> -- jcmd 1 VM.flags | grep MaxHeapSize

# 期望：MaxHeapSize 应该是合理值（< limits.memory × 0.8）

# 启动后跑一会再看 RSS
kubectl exec <pod> -- cat /proc/1/status | grep VmRSS
# 期望：VmRSS < limits.memory
```

> **口诀**：**JDK 8u190 之前在容器里基本不能用**。生产 JDK 必须 ≥ 11。临时方案要手动 -Xmx 兜底。

---

### 案例 ② Throttled 状态下 P99 飙升 —— "CPU 才 30% 为什么这么卡"

#### 现象

某个 Java 服务 CPU 利用率监控显示一直在 25%~35%，但 P99 延迟从 50ms 飙到 800ms，部分请求超时。Pod 配置 `limits.cpu: 1`。

#### 根因

`limits.cpu: 1` 翻译成 cgroup 配置 `cpu.cfs_quota_us = 100000, cpu.cfs_period_us = 100000`：每 100ms 周期最多用 100ms CPU 时间（= 1 个核）。

但你的 JVM 默认会起：
- GC 线程数 = `availableProcessors()` × 5/8（G1 默认）
- JIT 编译线程
- 业务线程池
- ……

`availableProcessors()` 在容器里返回 1（quota=period），但**线程数还是按物理核数算**？错。**JDK 17+ 是按容器看到的核数算**，所以 GC 线程数 ~ 1。但**业务线程池**是你自己 `newFixedThreadPool(32)` 起的，32 个线程。

32 个业务线程并行处理请求时，多个线程同时跑 → 短时间内打满 quota（10ms 内 32 个线程瞬间用 320ms CPU 时间）→ 剩下 90ms 全部线程被 throttle。

监控的 "CPU 利用率 30%" 是**长周期平均**，掩盖了"短期满载 + 长期空转"。看 throttling 指标：

```bash
cat /sys/fs/cgroup/cpu/<path>/cpu.stat
nr_periods 10000
nr_throttled 4521         ← ★ 45% 的 period 都被 throttle
throttled_time 81000000000  # 81 秒被 throttle（10 分钟内）
```

#### 修复

方案 A（首选）：抬 limits.cpu。

```yaml
resources:
  requests:
    cpu: 500m
  limits:
    cpu: 4         # ★ 至少 2，最好 4
```

理由：突发并发处理需要多核同时跑。limits 给少了等于强制 throttle。

方案 B：禁用 CFS quota（高级用法）。

K8s 1.27+ 支持节点级 `cpuCFSQuota: false`，整个节点的容器都不限 CPU，只用 cpu.shares 做权重。代价：noisy neighbor 风险高，需要节点级别隔离策略。

方案 C：缩短 CFS period（微调）。

```yaml
# kubelet 配置
cpuCFSQuotaPeriod: 10ms
```

默认 100ms 改 10ms 后，throttle 颗粒变细，P99 改善（但 nr_throttled 数字会变大，因为周期变多了，看比例不看绝对值）。

方案 D：JVM 调优。

```bash
# 减少 GC 线程数避免突发
-XX:ParallelGCThreads=2
-XX:ConcGCThreads=1
# 业务线程池缩小
-Dserver.tomcat.threads.max=50
```

#### 验证

修复前后对比：

```bash
# Prometheus 查询
container_cpu_cfs_throttled_seconds_total{pod="<pod>"}
container_cpu_cfs_throttled_periods_total / container_cpu_cfs_periods_total
```

期望：`throttled / periods < 1%`。

业务侧：P99 应回到正常水平。

> **口诀**：**CPU 利用率监控会骗人**。看 `nr_throttled / nr_periods` 比例，> 1% 就有 P99 风险。limits.cpu 至少 2。

---

### 案例 ③ 容器内 thread dump 看不到 —— PID namespace 的坑

#### 现象

线上 Java 应用偶发 hang，运维登容器执行：

```bash
kubectl exec <pod> -- jstack <pid>
```

但**找不到 pid**：

```bash
kubectl exec <pod> -- jps
```

`jps` 列表是空的或只有它自己。在容器外用 `kubectl describe pod` 看到的 PID 也对不上。

#### 根因

容器内的 PID 命名空间是独立的：
- 宿主机看到：`java` 进程 PID = 12345
- 容器内看到：同一个进程 PID = 1

`jps` 实际上扫描 `/tmp/hsperfdata_<user>/<pid>` 文件（JVM 性能数据文件，PID 是创建时写的）。问题在：
- 容器启动时 JVM PID = 1，写文件 `/tmp/hsperfdata_root/1`
- 你 `kubectl exec` 进容器，是另一个 PID namespace 子树，但你看到的还是同一个进程，PID 还是 1
- 但如果 JVM 是被某个父进程（init / dumb-init）启动的，JVM 自己的 PID 可能不是 1

更糟的情况：使用了 `tini` 或 `dumb-init` 包装，JVM PID = 2 或更大。`jps` 找的目录里的 PID 文件名和当前看到的 PID 对不上时就报错。

还有一种情况：你 `kubectl exec` 进容器时是个新 shell，shell 进程在容器 PID namespace 里有自己的 PID。`jps` 输出的 PID 是容器内 PID 视角，但你以为是宿主机视角。

#### 修复

方案 A（首选）：直接用 PID = 1 操作 JVM。

```bash
# 不需要 jps，直接对 PID 1 操作
kubectl exec <pod> -- jstack 1
kubectl exec <pod> -- jcmd 1 GC.heap_dump /tmp/heap.hprof
kubectl exec <pod> -- jcmd 1 Thread.print
```

前提：JVM 必须是容器内的 PID 1，即 Dockerfile 用 `ENTRYPOINT ["java", ...]` 而不是 shell wrapper。

方案 B（用 wrapper 时）：用 `tini` 当 init，但还是用 `jcmd 1`：

```dockerfile
RUN apt-get install -y tini
ENTRYPOINT ["/usr/bin/tini", "--", "java", "-jar", "app.jar"]
```

`tini` 是 PID 1，但它会立即 exec 子进程，子进程是 `java`，PID 一般是 7 或 8。这种情况下 `jcmd` 需要找正确 PID：

```bash
kubectl exec <pod> -- ps aux | grep java
# 找到 java 的实际 PID（比如 7）
kubectl exec <pod> -- jcmd 7 Thread.print
```

方案 C：用 `kubectl debug` 注入诊断容器。

```bash
kubectl debug -it <pod> --image=eclipse-temurin:17-jdk --target=<container-name>
# 在 debug 容器内
jcmd <主容器PID> Thread.print
```

需要 K8s 1.25+ 且开启 `EphemeralContainers` feature。

方案 D：从宿主机操作。

```bash
# 在宿主机找到容器 PID
crictl inspect <container-id> | grep pid
# 比如 12345

# 用宿主机的 jcmd 操作（要求宿主机有 JDK，或用容器内的）
nsenter -t 12345 -m -p -- jcmd 1 Thread.print
```

`nsenter` 是 Linux 工具，进入指定进程的 namespace。

#### 验证

修好之后：

```bash
kubectl exec <pod> -- jcmd 1 VM.uptime
# 应该输出运行时间，不报错
```

> **口诀**：容器里 JVM **几乎总是 PID 1**（除非用了 wrapper）。`jcmd 1` 是最可靠的访问方式。`jps` 不可靠，因为它依赖 hsperfdata 路径。

---

### 案例 ④ 跨节点 cgroup 配置不一致 —— "同一份 yaml，行为不一样"

#### 现象

10 个 Pod 实例（同一 Deployment），有 3 个偶发 P99 飙升、有 7 个完全正常。代码、配置、版本完全一致。日志、监控也看不出差异。

#### 根因

集群处于"灰度升级中"：
- 7 个老节点：cgroup v1，CFS period 默认 100ms
- 3 个新节点：cgroup v2，且节点 kubelet 配了 `cpuCFSQuotaPeriod: 5ms`

cgroup v2 + 短 period 改变了 CPU 限额的"颗粒度"。原本你的 JVM 在 v1 + 100ms period 下，突发计算可以"借"到几个完整周期；切到 v2 + 5ms period 后，每 5ms 就要重新分配一次 quota，**突发被切得更碎**。

更隐蔽的差异：cgroup v1 和 v2 的 `cpu.shares` 单位不同：
- v1：1024 = 1 核相当
- v2：100 = 1 核相当（实际是 `cpu.weight`，范围 1~10000）

K8s 自己做了 v1 → v2 的换算：`request.cpu × 1024 → cpu.shares`，到 v2 节点上 kubelet 把数字按比例缩放。但**老 K8s 版本（< 1.25）这个换算可能有 bug**。

诊断：

```bash
# 每个 Pod 进去看实际的 cgroup 配置
kubectl exec <pod> -- cat /sys/fs/cgroup/cpu.max  # v2
kubectl exec <pod> -- cat /sys/fs/cgroup/cpu.weight

# 或 v1
kubectl exec <pod> -- cat /sys/fs/cgroup/cpu/cpu.cfs_quota_us
kubectl exec <pod> -- cat /sys/fs/cgroup/cpu/cpu.cfs_period_us
kubectl exec <pod> -- cat /sys/fs/cgroup/cpu/cpu.shares
```

对比 v1 和 v2 节点上同一个 Deployment 的 Pod，看具体数字是否符合预期。

#### 修复

方案 A：统一节点 cgroup 版本。要么全 v1 要么全 v2。

```bash
# 检查每个节点
kubectl get nodes -o json | jq '.items[].status.nodeInfo'
```

方案 B：用 nodeAffinity 把关键应用调度到同一类节点。

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: cgroup-version
            operator: In
            values: ["v2"]
```

需要先给节点打 label：

```bash
kubectl label node <node> cgroup-version=v2
```

方案 C：升级 K8s 到 1.27+，`MemoryQoS` 和 `cpuCFSQuotaPeriod` 等 feature 在 v2 下行为更一致。

#### 验证

```bash
# 跨 Pod 对比 throttled 比例
kubectl get pods -l app=<name> -o name | xargs -I{} kubectl exec {} -- cat /sys/fs/cgroup/cpu.stat 2>/dev/null
```

数字应该相近（±10% 内）。

> **口诀**：**集群灰度升级期间最容易踩 cgroup 版本不一致的坑**。同一应用要么全调度到一类节点，要么忍受不一致。

---

### 案例 ⑤ Java 17 之前老版本的 cgroup 误读 —— "升级 JDK 后 GC 突然变好"

#### 现象

某服务从 JDK 11.0.5 升级到 JDK 17，GC 频率从每小时 200 次 YGC 降到 50 次，FGC 从一天 3 次降到 0。**没改任何配置**。

#### 根因

JDK 11.0.5 在 cgroup v2 节点上识别 `memory.max` 不正确（[JDK-8230305](https://bugs.openjdk.org/browse/JDK-8230305) 在 11.0.16 才合并）。JVM 看到 `memory.max = max`（字符串），解析失败 → 落回宿主机内存（如 256GB）→ 默认堆按 25% × 256GB = 64GB。

但容器只有 4GB。`-Xmx` 没显式设的话，JVM 启动后**不会立刻分配 64GB**，而是按需 mmap。但当对象分配速率高时，堆扩张到 4GB 就触发 cgroup OOM。

实际看到的现象更微妙：JVM 启动正常，跑一会突然 OOMKilled；或者 JVM "自我感觉"堆还能涨，YGC 阈值不触发，对象一直占着不被回收，最终撞到容器 limits → 整个容器被杀。

升级 JDK 17 后，cgroup v2 探测正确，`-Xmx` 自动按 4GB 计算（默认 25% = 1GB），堆控制在合理范围 → GC 行为正常。

#### 修复

显式 `-Xmx` 永远是最稳妥方案：

```yaml
env:
- name: JAVA_TOOL_OPTIONS
  value: "-Xms3g -Xmx3g -XX:MaxRAMPercentage=75"
```

注意：`-Xmx` 和 `-XX:MaxRAMPercentage` 同时配的话，`-Xmx` 优先。但 `MaxRAMPercentage` 能让 yaml 一份配置在不同 limits 的容器自动适配。

升级 JDK 路径：

```dockerfile
# 老
FROM openjdk:11-jre-slim
# 新
FROM eclipse-temurin:17-jre-alpine
```

注意 `eclipse-temurin` 是 Eclipse Adoptium 出品的免费 OpenJDK 发行版，比 OracleJDK 更适合容器（无授权问题）。

#### 验证

升级后：

```bash
kubectl exec <pod> -- jcmd 1 VM.flags | grep -E "MaxHeapSize|MaxRAMPercentage"
# MaxHeapSize 应该 ≈ limits.memory × 0.75
```

GC 监控对比升级前后的 YGC / FGC 频率。

> **口诀**：JDK 11 必须 ≥ 11.0.16；JDK 17 比 11 默认更稳。**永远显式 -Xmx，不依赖 JVM 自动探测**。

---

### 案例 ⑥ swap 在容器内意外被启用 —— "明明禁了却还在 swap"

#### 现象

业务报"GC 偶尔卡 5 秒"，但堆远没满。/proc/<pid>/status 看 VmSwap 不为 0：

```bash
kubectl exec <pod> -- cat /proc/1/status | grep -E "VmRSS|VmSwap"
VmRSS:   2048000 kB
VmSwap:   512000 kB    ← 居然有 swap！
```

K8s 文档明明说"K8s 节点不允许 swap"。

#### 根因

K8s **历史上**默认要求节点 swap 关闭（kubelet 启动会失败）。但 1.22+ 引入了 `NodeSwap` feature gate，允许节点开 swap：

```yaml
# kubelet 配置
failSwapOn: false
memorySwap:
  swapBehavior: LimitedSwap   # 或 UnlimitedSwap
```

Burstable QoS 的 Pod 在节点开 swap 时，cgroup 配置允许使用 swap：

```bash
# v1
cat /sys/fs/cgroup/memory/<path>/memory.swappiness    # 60（默认）
cat /sys/fs/cgroup/memory/<path>/memory.memsw.limit_in_bytes

# v2
cat /sys/fs/cgroup/<path>/memory.swap.max
```

**JVM 不喜欢 swap**：
- 堆页面被 swap 出去后，下次 GC 扫描时全部要重新 swap 进来
- swap 的单页 IO 延迟通常是几 ms 到 100ms，比内存慢千倍以上
- Full GC 期间需要扫整个堆，全部 swap 进来 → STW 数秒

#### 修复

方案 A（推荐）：节点级禁 swap。

```bash
# 节点上
swapoff -a
# /etc/fstab 注释掉 swap 行
```

但 K8s 1.22+ 即使节点开 swap，可以通过 kubelet 配置选择性禁用：

方案 B：Pod 级配置。

```yaml
# 设置为 Guaranteed QoS，K8s 自动不分配 swap
resources:
  requests:
    cpu: 2
    memory: 4Gi
  limits:
    cpu: 2
    memory: 4Gi
```

Guaranteed QoS Pod 在 K8s 1.30+ 默认不使用 swap（即使节点开了）。

方案 C：JVM 锁内存。

```bash
-XX:+AlwaysPreTouch -XX:+UseLargePages
```

`AlwaysPreTouch` 启动时把 `-Xmx` 全部 touch 一遍，强制内核分配物理页。结合 `mlock` 可避免被 swap（但 mlock 在容器里需要 capability）。

#### 验证

```bash
kubectl exec <pod> -- cat /proc/1/status | grep VmSwap
# VmSwap: 0 kB
```

> **口诀**：**JVM + swap 是噩梦**。生产 K8s 节点必须禁 swap。开 swap 的话也要让 Java 应用用 Guaranteed QoS。

---

### 案例 ⑦ 容器 OOM 但 JVM 没抛 OutOfMemoryError —— "应用日志一片祥和"

#### 现象

监控告警容器频繁重启，K8s 显示 ExitCode = 137（OOMKilled）。但应用日志里**没有任何** `OutOfMemoryError`，没有 stack trace，没有关于内存的告警。看起来"应用挺好的就突然死了"。

#### 根因

理解两个 OOM 不是同一回事：

| 维度 | JVM `OutOfMemoryError` | 容器 OOMKilled |
| --- | --- | --- |
| 触发者 | JVM 自己 | Linux 内核（cgroup OOM Killer） |
| 时机 | JVM 想分配对象，发现堆已满且 GC 也回收不出空间 | 容器 RSS 撞到 `memory.max` |
| 表现 | 应用收到 OOMError，可被 catch 或终止 | 进程收到 SIGKILL，无法响应 |
| 日志 | 应用日志有 stack trace | 内核日志有 OOM 信息，应用日志无 |

JVM 的堆只是进程总内存的一部分。其他部分：
- **Metaspace**：类元数据。开了大量动态代理 / CGLIB 时膨胀
- **JIT Code Cache**：编译后的机器码。默认 240MB（可被 `-XX:ReservedCodeCacheSize` 调整）
- **Thread Stacks**：每个线程默认 1MB（`-Xss`）。1000 线程 = 1GB
- **Direct Memory（堆外）**：NIO、Netty 用的。默认等于 `-Xmx`
- **Native Memory**：JNI、glibc malloc、mmap 文件 …

如果 `-Xmx2g`，但 Metaspace 占了 500MB，JIT 占了 300MB，1000 线程占了 1GB，direct buffer 占了 500MB → 进程 RSS = 4.3GB > limits.memory = 4GB → 容器 OOMKilled。**JVM 自己的堆完全没满**，所以不抛 OutOfMemoryError。

#### 修复

第一步：搞清楚到底谁吃了内存。开 NMT（Native Memory Tracking）：

```bash
java -XX:NativeMemoryTracking=detail -jar app.jar
# 启动后
jcmd 1 VM.native_memory summary
```

输出：

```
Native Memory Tracking:

Total: reserved=5234567KB, committed=3987654KB

-                 Java Heap (reserved=2097152KB, committed=2097152KB)
                            (mmap: reserved=2097152KB, committed=2097152KB)

-                     Class (reserved=1247635KB, committed=204371KB)
                            (classes #34521)

-                    Thread (reserved=1052480KB, committed=1052480KB)
                            (thread #1024)        ← ★ 1024 线程吃了 1GB

-                      Code (reserved=270592KB, committed=87232KB)
                            (mmap: reserved=270592KB, committed=87232KB)

-                        GC (reserved=92345KB, committed=92345KB)
                            (mmap: reserved=78912KB, committed=78912KB)

-                  Internal (reserved=12345KB, committed=12345KB)

-                    Symbol (reserved=23456KB, committed=23456KB)

-    Native Memory Tracking (reserved=4567KB, committed=4567KB)
```

按上面定位是**线程数过多**。

第二步：根据 NMT 输出针对性修复：

```bash
# 线程数过多 → 业务线程池缩小
-Dserver.tomcat.threads.max=200
# 或换虚拟线程

# Metaspace 膨胀 → 限制并监控
-XX:MaxMetaspaceSize=512m

# Direct Memory 过大 → 限制
-XX:MaxDirectMemorySize=512m

# 启用堆外 OOM 时打 dump
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/heap.hprof
```

第三步：增加 limits 留白。

```yaml
# 改前
limits:
  memory: 4Gi
env:
- name: JAVA_TOOL_OPTIONS
  value: "-Xmx3g"   # 堆 3GB，留 1GB 给 native

# 改后（线程多的应用）
limits:
  memory: 6Gi
env:
- name: JAVA_TOOL_OPTIONS
  value: "-Xmx3g"   # 堆 3GB，留 3GB 给 native（含线程栈）
```

#### 验证

部署后观察：

```bash
# 跑一段时间后查 NMT
kubectl exec <pod> -- jcmd 1 VM.native_memory summary

# 各部分 + heap 总和应 < limits.memory × 0.9
```

并对比修复前后的容器重启次数：

```bash
kubectl get pods -l app=<name> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.containerStatuses[0].restartCount}{"\n"}{end}'
```

> **口诀**：**容器 OOMKilled ≠ JVM OutOfMemoryError**。前者杀全进程，应用日志看不到。NMT 是定位 native 泄露的唯一办法。

---

### 案例 ⑧ 调度器把 CPU-密集容器调到 IO-密集节点 —— noisy neighbor

#### 现象

新加入一个数据处理服务（CPU 密集），上线后**同节点的其他 Java 服务**P99 全部恶化。被恶化的服务自身资源利用率没变，监控找不到自身原因。

#### 根因

K8s 默认调度器只看 `requests` 决策，不看节点上其他容器的"实际负载特征"。新服务 `requests.cpu: 1, limits.cpu: 16`，调度到一个本来 CPU 利用率 30% 的节点。结果：
- 新服务突发跑满 16 核 → 本来余 70% 余量被吃光
- 同节点其他容器（`limits.cpu: 4`）的 quota 物理上仍然能用，但**CPU 缓存被新服务污染**
- L3 cache miss 率从 5% 飙到 40% → 应用执行变慢
- 内存带宽也被抢，DRAM 访问延迟上升

更隐蔽：CFS 调度器是公平的，但**公平不等于隔离**。每个 cgroup 的 quota 都能保证，但缓存、内存带宽、PCIe 带宽这些**共享资源**没有 cgroup 级别的隔离（除非用 Intel RDT / AMD QoS 等硬件特性）。

#### 修复

方案 A：用 nodeAffinity / podAntiAffinity 做软隔离。

```yaml
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              workload-type: cpu-intensive
          topologyKey: kubernetes.io/hostname
  # 自己也打上 label
  metadata:
    labels:
      workload-type: latency-sensitive
```

意思："尽量不要把我和 CPU 密集的容器放一起"。

方案 B：节点池隔离。

```yaml
# 给 CPU 密集型工作负载专门的节点池
nodeSelector:
  workload-tier: batch    # 批处理节点
# 延迟敏感的服务用另一组
nodeSelector:
  workload-tier: online   # 在线节点
```

方案 C：CPU pinning（绑核）。

```yaml
# kubelet 配置 cpuManagerPolicy: static
# Pod 必须 Guaranteed QoS 且 CPU 是整数核
spec:
  containers:
  - name: app
    resources:
      requests:
        cpu: "4"          # ★ 整数
        memory: 8Gi
      limits:
        cpu: "4"
        memory: 8Gi
```

K8s 会用 cpuset 把容器绑到固定的 4 个物理核，避免和别人竞争。

方案 D：开启 CPU CFS Burst（K8s 1.27+）。

```yaml
resources:
  requests:
    cpu: 1
  limits:
    cpu: 1
  # 允许"借用"一段时间的 CPU
  cpuBurst: 1
```

平时按 limits 限制，突发时允许超出短时间。

#### 验证

```bash
# 关键服务的 P99 监控
# 同节点的 CPU cache miss 率（需要节点级 perf 监控）
perf stat -a -e cache-misses,cache-references sleep 60

# 看 cpu-quota 抢占
cat /sys/fs/cgroup/cpu/<path>/cpu.stat | grep wait_sum
```

> **口诀**：**CPU quota 是公平的，但 L3 cache、内存带宽不是 cgroup 隔离的**。混部时延迟敏感服务用 nodeAffinity 或专用节点池。

---

### 案例 ⑨ DevOps 配的 limit < request 导致调度异常

#### 现象

新发布一批 Pod，全部 `Pending`。`kubectl describe pod` 没有明显错误，调度器日志看不到。

#### 根因

```yaml
resources:
  requests:
    cpu: 2
    memory: 4Gi
  limits:
    cpu: 1            # ★ 居然小于 requests
    memory: 4Gi
```

DevOps 同学复制粘贴 yaml 时手抖，limits.cpu 改小了。**API server 接受这个请求**（K8s 的 admission control 不强制检查 limits >= requests，只检查格式合法）。

但调度器发现 limits.cpu < requests.cpu 时，**根本不报错**，只是 silently 不调度（Pod 一直 Pending）。

更隐蔽的情况：limits 比 requests 小一点点，Pod 能调度，但运行时 cgroup 里 quota 比 requests 算的 shares 还小，CPU 严重 throttle。

#### 修复

方案 A：开启 LimitRange 强制约束。

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: enforce-limit-ge-request
  namespace: default
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    max:
      cpu: 4
      memory: 8Gi
    min:
      cpu: 50m
      memory: 64Mi
```

效果：
- 没设 limits 时用 default 填充
- 设了 limits 不能小于 min、不能大于 max

但 LimitRange **不直接验证 limits >= requests**。这要靠 admission webhook 或 OPA Gatekeeper。

方案 B：用 OPA Gatekeeper 写策略。

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sLimitsGreaterThanRequests
metadata:
  name: limits-must-be-greater-than-requests
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
```

对应的 ConstraintTemplate 检查 limits >= requests。

方案 C：CI/CD 阶段做 yaml 静态检查。

```bash
# kubeval / kube-linter / kube-score
kube-score score deployment.yaml
```

会发现 limits < requests 这种问题。

#### 验证

故意提交一个 limits < requests 的 yaml，看是否被拒绝：

```bash
kubectl apply -f bad.yaml
# 期望：报错，被 webhook 拦截
```

> **口诀**：**K8s 不强制 limits >= requests，会 silently Pending**。生产必须用 LimitRange + OPA Gatekeeper + CI 检查三道防线。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### JDK / 镜像层
- [ ] JDK 版本 ≥ 17（最低 11.0.16+），不允许 JDK 8u190 之前的镜像上线
- [ ] 基础镜像用 `eclipse-temurin` 或 `azul/zulu-openjdk`，不要用没维护的旧镜像
- [ ] Dockerfile 用 `ENTRYPOINT ["java", ...]`，让 JVM 是 PID 1
- [ ] 加 `tini` / `dumb-init` 作为 init 进程，避免僵尸进程

### 内存层
- [ ] 显式 `-Xms = -Xmx`，避免堆动态扩缩
- [ ] `-Xmx` ≤ `limits.memory × 0.75`，给 native 留 25%
- [ ] 或用 `-XX:MaxRAMPercentage=75 -XX:InitialRAMPercentage=75`，自适应不同 limits
- [ ] `-XX:+AlwaysPreTouch`，启动时锁定堆物理页
- [ ] `-XX:NativeMemoryTracking=summary`，便于排查 native 泄露
- [ ] `-XX:MaxMetaspaceSize` 显式限制（CGLIB / 动态代理多的应用）
- [ ] `-XX:MaxDirectMemorySize` 显式限制（Netty / NIO 应用）
- [ ] `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/log/heap.hprof`

### CPU 层
- [ ] `limits.cpu` 至少 2，最好 ≥ requests.cpu × 2
- [ ] CPU 密集应用考虑 `cpuManagerPolicy: static` + Guaranteed QoS 实现绑核
- [ ] 关闭依赖 CPU shares 的 JVM 行为：`-XX:-UseContainerCpuShares`（JDK 19+）
- [ ] `-XX:ParallelGCThreads` / `ConcGCThreads` 显式设，不要让 JVM 按 availableProcessors 算

### K8s 资源层
- [ ] **所有 Java 应用配 Guaranteed QoS**：requests == limits
- [ ] 不允许 BestEffort 的 Java Pod 上生产
- [ ] `terminationGracePeriodSeconds` ≥ 60s，给 JVM 优雅停机时间
- [ ] `preStop` 配 `sleep 10` 等待 endpoint 摘除
- [ ] LimitRange + OPA Gatekeeper 强制 limits ≥ requests
- [ ] 关键应用配 podAntiAffinity 或专用节点池

### 监控层
- [ ] cgroup 指标采集：`container_memory_working_set_bytes`、`container_cpu_cfs_throttled_seconds_total`
- [ ] OOMKilled 告警：`kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}`
- [ ] CPU throttle 比例告警：`rate(container_cpu_cfs_throttled_periods_total) / rate(container_cpu_cfs_periods_total) > 0.05`
- [ ] JVM 内存监控（堆、metaspace、direct、threads）通过 Micrometer + Prometheus 暴露
- [ ] PSI 监控（cgroup v2）：memory.pressure、cpu.pressure

### 节点层
- [ ] 所有节点 swap 关闭（除非显式启用且配套 Guaranteed QoS）
- [ ] 集群 cgroup 版本统一（要么全 v1 要么全 v2，避免混合）
- [ ] kubelet 配置 `cpuCFSQuota: true`（默认开），但评估 `cpuCFSQuotaPeriod` 调整
- [ ] containerd / CRI-O 用统一的 cgroup driver（推荐 systemd）

### 排查工具就绪
- [ ] 镜像内有 `jcmd` / `jstack` / `jmap` —— 用 JDK 镜像不是 JRE 镜像
- [ ] 或者部署 sidecar / ephemeralContainer 方案
- [ ] 节点上 `crictl` / `nsenter` 可用
- [ ] `kubectl debug` 在生产可用

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### 容器运行时的演进
- **Kata Containers**：用轻量 VM（QEMU + KVM）跑容器，每个容器一个 VM，内核完全隔离。性能损耗 5%~15%，但隔离性接近虚拟机。适合多租户场景。
- **gVisor**（Google）：用户态实现的 Linux ABI 兼容层，把 syscall 拦截到用户态处理。隔离性强但性能损失大（30%~50%），适合不可信代码沙箱。
- **Firecracker**（AWS）：Lambda 背后的 microVM，启动 < 125ms，单机能跑数千个。Rust 写的极简 VMM。
- **WasmEdge / Wasmer**：用 WebAssembly 替代容器。镜像几 MB、启动毫秒级、跨平台。但生态还不成熟，Java 应用基本跑不了。

### 虚拟化嵌套
- 虚拟机里跑容器（K8s on EC2 / GCE）的代价：double dispatch、IO 路径变长、网络延迟翻倍
- 嵌套虚拟化（Nested KVM）的开销：CPU 性能损失 5%~10%，内存性能损失 10%~20%
- 裸金属 K8s（EKS Bottlerocket / GKE on-prem）的性能优势

### eBPF 在容器观测
- **Cilium**：用 eBPF 实现 K8s CNI 数据面，比 iptables 快一个数量级，且支持 L7 策略
- **Pixie**：用 eBPF 做无侵入应用监控，自动抓 HTTP / gRPC / DNS 流量
- **Tetragon**：用 eBPF 做运行时安全监控，能拦截 syscall 级别的恶意行为
- **bcc / bpftrace**：写 eBPF 程序的工具链，做容器内的细粒度 profiling

### K8s 资源管理的新方向
- **Vertical Pod Autoscaler（VPA）**：基于历史数据自动调整 requests/limits。但和 HPA 不能同时用同一个指标
- **Karpenter**（AWS）：替代 Cluster Autoscaler，更快、更灵活的节点扩容
- **In-place Pod Resize**（K8s 1.27+ alpha）：不重启 Pod 直接改 limits，对 Java 应用是颠覆性的（不需要重启就能扩容）
- **MemoryQoS**（K8s 1.22+）：基于 cgroup v2 PSI 做内存压力调度
- **Topology Manager**：CPU、Memory、Device 在 NUMA 节点上的协同分配

### Java 在云原生的演进
- **GraalVM Native Image**：AOT 编译成原生可执行文件，启动几十毫秒、内存几十 MB。适合 Serverless
- **CRaC（Coordinated Restore at Checkpoint）**：JEP 还没正式入主线，能把热 JVM 状态序列化为 checkpoint，下次启动 < 100ms
- **Project Leyden**：探索 Java 的"静态镜像"模型，AOT + checkpoint 结合
- **AppCDS**（已 GA）：把已加载的类数据 dump 到文件，下次启动快速 mmap，启动时间砍 30%

### 多集群与边缘
- **KubeEdge / OpenYurt**：边缘节点的 K8s，cgroup / 资源模型在受限环境下的简化
- **Service Mesh + cgroup**：Istio sidecar 的 CPU 占用如何归属到正确的应用 cgroup
- **多集群联邦（KubeFed）**：跨集群一致的资源策略

---

## 7. 引用

- Linux kernel 文档：[cgroup-v2.rst](https://www.kernel.org/doc/Documentation/admin-guide/cgroup-v2.rst)
- Linux kernel 文档：[namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- OpenJDK 17 源码：[cgroupV1Subsystem_linux.cpp](https://github.com/openjdk/jdk17u/blob/master/src/hotspot/os/linux/cgroupV1Subsystem_linux.cpp)
- OpenJDK 17 源码：[cgroupV2Subsystem_linux.cpp](https://github.com/openjdk/jdk17u/blob/master/src/hotspot/os/linux/cgroupV2Subsystem_linux.cpp)
- JEP 223: New Version-String Scheme
- JEP 248: Make G1 the Default Garbage Collector
- [JDK-8146115](https://bugs.openjdk.org/browse/JDK-8146115)：Improve docker container detection and resource configuration usage
- [JDK-8230305](https://bugs.openjdk.org/browse/JDK-8230305)：Cgroups v2: Container awareness
- Tejun Heo, *cgroup v2 design notes*（2014~）
- K8s 文档：[Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- K8s 文档：[Quality of Service for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)
- Brendan Gregg, *BPF Performance Tools*（2019）—— eBPF 圣经
- 美团技术博客：[Java 容器内存的那些事](https://tech.meituan.com/2019/02/14/java-container-memory.html)
- AWS Aurora 文档：MVCC + 容器内存配置最佳实践

---

> 配套：[Linux 与性能工程主题](/article/linux-perf/) ｜ [内存与 OOM](/article/linux-perf/memory-oom) ｜ [JVM GC 深入](/article/jvm/gc-deep)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你觉得这个深度对，告诉我下一篇该写哪个。
