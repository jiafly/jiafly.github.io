# Linux 内存模型与 OOM —— 从 PageCache 到 OOM Killer 的全景

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，看 [Linux 性能综述](/article/linux-perf/) 那一节就够了。
> 这里要回答的，是**线上突然 OOMKilled、容器被杀、`free -h` 看着一堆"available"却 malloc 失败**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. `free -h` 显示 `used 12G / free 200M / available 8G`，到底剩多少能用？为什么 free 那么小但 available 那么大？
2. `buff/cache` 里那 5GB 是不是"被占用"了？我跑应用是直接抢得到，还是要等回收？回收过程会不会卡？
3. Slab 是什么？为什么 `dentry / inode_cache` 能涨到几十 GB？怎么主动让它回收？
4. `vm.overcommit_memory = 0 / 1 / 2` 各代表什么？生产建议哪一档？为什么 Java 容器化后大家爱开 `2`？
5. OOM Killer 决定杀谁的依据是什么？`oom_score` 怎么算？怎么让我的关键进程"豁免"被杀？
6. 容器里 `free -h` 看到的是宿主的内存，还是 cgroup 限额内的？容器 OOMKilled 时，宿主上 `dmesg` 能看到什么？
7. JVM 在 Docker 里启动，怎么知道自己只能用 2GB（而不是宿主的 256GB）？`-XX:MaxRAMPercentage` 是怎么读到 cgroup 限额的？
8. 服务器 swap 关了好还是开好？swap 开着了 P99 抖动是怎么回事？`vm.swappiness` 调小到 0 就一定不 swap 吗？

如果有半数答不上，往下看。

---

## 1. 底层模型：物理、虚拟、PageCache、cgroup

绝大多数人对 Linux 内存的认知止于"`free -h` 看一下"。但这层抽象解释不了上面 8 个问题。要看懂它们，必须下沉到 Linux MM (Memory Management) 子系统的核心数据结构。

### 1.1 一张图看清 Linux 内存层级

```
                         用户空间 (User Space)
   ┌──────────────────────────────────────────────────────────┐
   │   进程虚拟内存                                              │
   │   ┌──────────────────────────────────────────────────┐    │
   │   │ Stack ↓ │ Heap (brk) ↑ │ mmap 区 │ .data │ .text │    │
   │   └──────────────────────────────────────────────────┘    │
   │           │                 │                              │
   │           ▼ MMU 翻译         ▼                              │
   │   ┌──────────────────────────────────────────────────┐    │
   │   │ Page Table (per-process)                         │    │
   │   └──────────────────────────────────────────────────┘    │
   └──────────────────────┬───────────────────────────────────┘
                          │
                         ▼
                 内核空间 (Kernel Space)
   ┌──────────────────────────────────────────────────────────┐
   │  Page Cache (文件缓存)  │  Slab (内核对象 dentry/inode)    │
   │  Anonymous Pages       │  Buddy System (物理页分配器)      │
   │  Swap Cache            │  vmalloc 区                       │
   └──────────────────────┬───────────────────────────────────┘
                          │
                          ▼
                     物理内存 (RAM) + Swap
```

注意 4 件事，后面所有讨论都基于这张图：

1. **进程看到的是虚拟内存**，物理页要走 MMU + Page Table 翻译。`top` 里的 VIRT 是虚拟，RSS 才是真正的物理占用。
2. **内核也吃物理内存**，且不可被换出（除了 swap 区）—— Slab、Page Cache、内核栈、页表本身都算。
3. **Page Cache 不是"被进程占用"**，是内核帮你缓存文件 IO，可被回收。这是 `free -h` 里 `buff/cache` 的核心。
4. **物理页的最终分配者只有一个：Buddy System**。Slab、Page Cache、匿名页都是 Buddy 的"客户"。

### 1.2 物理内存的四种用途

| 用途 | 可回收性 | 何时占用 | 何时释放 |
| --- | --- | --- | --- |
| **Anonymous Pages**（匿名页） | 仅能 swap | 进程 malloc / heap 增长 | 进程退出 / swap 出去 |
| **Page Cache**（文件缓存） | **可丢弃**（干净页直接丢；脏页先回写） | read/write 文件、mmap 文件 | 内存压力下被 reclaim |
| **Slab**（内核对象池） | 部分可回收（`SLAB_RECLAIM_ACCOUNT`） | 内核分配 dentry/inode/task_struct 等 | 引用归零或 shrinker 触发 |
| **HugePages / 内核栈** | 一般不可回收 | 显式预留 / 进程创建 | 显式释放 / 进程退出 |

这四类的总和 = `MemTotal - MemFree`。所以"used 12G"包含**所有内核占用 + 进程匿名页 + Page Cache + Slab**，没扣可回收的那部分。

### 1.3 Buddy System：物理页的源头

Linux 把物理内存划成 `2^0, 2^1, 2^2, ..., 2^10` 共 11 个"order"的连续页块（最大 1024 页 = 4MB）。每个 order 维护一个空闲链表。

```
Free pages by order (one zone, simplified):

order  0    1    2    3    4   ...   10
       ▼    ▼    ▼    ▼    ▼          ▼
      ████ ████ ████ ████ ░░░░       ░░░░
       ↑                              ↑
      4KB                            4MB
```

分配 `2^k` 页：找 order=k 的链表，没了找 k+1 拆半，依此向上。释放：尝试和"伙伴"合并回更大的 order。

**反直觉的点**：即使 `free` 显示 8GB 可用，如果全是离散的 4KB 页（碎片化），**你也分不到一块连续 2MB**。这是 HugePage / 网卡 DMA / 显卡显存分配失败的常见原因（案例 ⑧ 会展开）。

```bash
# 看每个 order 还剩多少页
cat /proc/buddyinfo
```

```
Node 0, zone   Normal  12345  6789   234    12     1     0    0    0    0    0    0
                       ^^^^^                              ^^^^
                       order 0 还有 12345 页              order 7+ 已经没了
```

### 1.4 Page Cache：被误解最多的"used"

读文件时内核把文件内容缓存在 Page Cache 里，下次读直接命中。写文件时先进 Page Cache（"脏页"），后台异步回写到磁盘。

```
应用 read(fd) ──► VFS ──► Page Cache ──Cache miss──► 磁盘
                            │
                            └──Cache hit──► 直接返回 (zero-copy 还能省一次拷贝)

应用 write(fd) ──► VFS ──► Page Cache (脏页) ──► 后台 flusher ──► 磁盘
                            ▲
                            └─ 这一刻 write() 就返回了
```

Page Cache **算在 `used` 里**（旧版 free），但**算在 `available` 里**（新版 free）。这是同一份内存被两个口径数了两遍。

**回收时机**：
- 干净页（已回写的）：直接丢弃，**几乎零代价**
- 脏页（还没回写的）：先 fsync 出去，**有 IO 代价**

所以"available 8G"在干净页多时几乎=可立即用；在脏页多时回收会引发 IO 风暴（案例 ② 展开）。

### 1.5 Slab：内核自己的"对象池"

内核频繁分配 / 释放固定大小的对象（dentry、inode、task_struct、TCP socket、struct file 等）。直接走 Buddy 太重（最小 4KB）且会浪费。Slab/Slub 在 Buddy 之上做了一层"对象级"分配器。

```bash
# 按缓存大小排序
slabtop -s c
```

```
 OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
3245k 3245k 100%    0.19K  77520       42    620160K dentry
1245k 1245k 100%    1.07K  43000       29    689000K xfs_inode
 234k  234k 100%    0.12K   3666       64     29328K kernfs_node_cache
 ...
```

`dentry` / `xfs_inode`（或 ext4_inode）是常见的"内核内存膨胀"嫌疑犯。如果你的服务大量打开关闭文件、ls 大目录、stat 海量小文件，dentry 缓存能涨到几十 GB（案例 ⑤）。

主动回收：

```bash
# 1=PageCache, 2=Slab(dentry+inode), 3=both
echo 2 > /proc/sys/vm/drop_caches
```

**生产警告**：drop_caches 只用于诊断 / 临时救急。频繁调会让真正的热数据被反复加载。

### 1.6 虚拟内存：mmap、brk、vmalloc

进程的虚拟地址空间由几类映射组成：

| 区域 | 来源 | 物理内存何时分配 |
| --- | --- | --- |
| `.text / .data / .bss` | execve 加载二进制 | 第一次访问时 page fault |
| **Heap** | `brk()` / `sbrk()` | 第一次访问时 page fault |
| **Anonymous mmap** | `mmap(NULL, ...)` | 第一次访问时 page fault |
| **File-backed mmap** | `mmap(fd, ...)` | 第一次访问时从 Page Cache 拿 |
| **Stack** | execve / pthread_create | 自动按需扩展 |
| `vmalloc` 区 | 内核内部用 | 通常立即分配（但是不连续物理页拼起来的虚拟连续区） |

关键认知：**虚拟分配 ≠ 物理占用**。`malloc(1GB)` 立即返回成功 + VIRT 涨 1GB，但 RSS 一字节没涨 —— 直到你去 memset 它。

这就是 **overcommit** 概念的来源：内核允许你 malloc 比物理内存还多的虚拟空间，赌你不会全用。

### 1.7 overcommit 与 OOM Killer

`vm.overcommit_memory` 三档：

| 值 | 行为 | 适用 |
| --- | --- | --- |
| `0`（默认，启发式） | 内核估算："这次申请会不会明显超？"超了就拒绝 | 通用桌面 / 大多数服务 |
| `1`（无脑允许） | 永不拒绝。完全相信 OOM Killer 会兜底 | Redis / 数据库 / fork 大进程的应用 |
| `2`（严格限制） | 永不超出 `swap + RAM × overcommit_ratio`（默认 50%）| 不能容忍 OOM Kill 的关键服务 |

为什么 Redis 文档要求 `vm.overcommit_memory = 1`？因为 Redis 持久化用 `fork()` —— fork 时**虚拟空间整个复制了一份**（COW，copy on write），看起来内存翻倍。0 档可能拒绝 fork，导致持久化失败。

OOM Killer 何时触发？当物理 + swap 真的用完，再 page fault 就触发：

1. 内核遍历进程，给每个进程算 `oom_score`
2. 选 `oom_score` 最高的进程，发 SIGKILL
3. 释放它的所有物理页，恢复运行

`oom_score` 怎么算（简化版）：

```
oom_score(p) = (p->total_vm × 1000) / total_pages
             + p->oom_score_adj    (用户可调，-1000 到 +1000)
```

简言之**RSS 大的容易被杀**，但你可以用 `oom_score_adj` 硬调：

```bash
# 让 PID 1234 永远不被 OOM 杀（除非内核自己崩溃）
echo -1000 > /proc/1234/oom_score_adj
```

`-1000` 等价于"豁免"。生产中常给关键守护进程（init、监控 agent、SSH）设 `-1000`。

### 1.8 swap 与 swappiness

swap 是把匿名页（heap、stack）换到磁盘上，腾出物理内存。`vm.swappiness` 控制"内核多大程度倾向于 swap"：

- `0`：除非真要 OOM 才 swap
- `60`（默认）：均衡
- `100`：积极 swap，把匿名页和 PageCache 同等对待

**生产铁律**（云原生 / 数据库场景）：
- 数据库节点 swap 关掉或 `swappiness=1`，**绝不能让热数据被换到磁盘**
- 容器节点 K8s 1.22 前必须关 swap（kubelet 强制要求），1.22+ 可选开

为什么 swap 那么"反感"？因为**P99 抖动来自 swap-in**：被换出的页下次访问要从磁盘读回来，延迟从几十纳秒飙到几毫秒（案例 ⑥）。

### 1.9 NUMA：内存的"远近"

多 socket 服务器有多个 CPU socket，每个 socket 直连一部分内存条。CPU 访问"自己 socket"的内存（local node）快，访问"另一个 socket"的内存（remote node）要走 QPI/UPI 总线，延迟翻倍。

```bash
numactl -H
```

```
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3 ... 23
node 0 size: 128000 MB
node 0 free: 5000 MB
node 1 cpus: 24 25 ... 47
node 1 size: 128000 MB
node 1 free: 80000 MB
node distances:
node   0   1
  0:  10  21    ← node 0 → node 1 比本地慢 2.1 倍
  1:  21  10
```

如果你的进程被调度到 node 0 的 CPU，但物理内存大量在 node 1，每次访问内存都跨节点 —— **延迟翻倍**（案例 ⑦）。

### 1.10 cgroup memory controller：容器化的核心

cgroup 是 K8s/Docker 的内存隔离基础。每个容器是一个 cgroup，内核会在分配内存时检查"这个 cgroup 用了多少 / 上限是多少"。

cgroup v1 vs v2 关键差异：

| 维度 | v1 | v2 |
| --- | --- | --- |
| 接口 | `/sys/fs/cgroup/memory/*` | `/sys/fs/cgroup/*` 统一层级 |
| 限额 | `memory.limit_in_bytes` | `memory.max` |
| 软限制 | `memory.soft_limit_in_bytes`（基本没用） | `memory.high`（**真正起作用**，触发限速回收） |
| OOM | 容器级 OOM Killer 杀容器内进程 | 同上，但更细粒度 |
| 统计 | `memory.stat`（字段多） | `memory.stat`（字段更准确）|

**v2 引入的 `memory.high` 是个大改进**：超过 high 但还没到 max 时，进程会被"throttle"（限速回收），而不是直接 OOMKill。给应用一个"喘息"机会。

容器 OOMKilled 时，你在宿主上 `dmesg` 能看到（案例 ①）：

```
Memory cgroup out of memory: Killed process 12345 (java) total-vm:8GB anon-rss:2GB ...
```

注意 "**Memory cgroup** out of memory" 这行 —— 是 cgroup OOM，不是宿主 OOM。

> **口诀**：物理内存四类用途（Anon / PageCache / Slab / 内核固定），分配总入口是 Buddy，进程虚拟分配 ≠ 物理占用，overcommit 让 OOM Killer 兜底。容器内一切以 cgroup 为准，不看宿主。

---

## 2. 数据切片：从 /proc 看到一切

> 这一节是查问题的"瑞士军刀"。每段都是命令 + 字段解读 + 结合源码点透。

### 2.1 /proc/meminfo —— 全局视角

```bash
cat /proc/meminfo
```

```
MemTotal:       32958288 kB     物理内存总量（不含 reserved）
MemFree:          512000 kB     完全空闲（什么都没装）
MemAvailable:    8500000 kB     ★ 可立即使用的"实际可用"（含可回收的 PageCache/Slab）
Buffers:           50000 kB     块设备缓冲（基本可忽略）
Cached:          7800000 kB     ★ 文件 Page Cache
SwapCached:        20000 kB     被读回但还在 swap 里的页
Active:         18000000 kB     最近被访问，回收优先级低
Inactive:        4000000 kB     久未访问，优先被回收
Active(anon):   12000000 kB     匿名页中的"活跃"
Inactive(anon):   500000 kB
Active(file):    6000000 kB     文件 PageCache 中的"活跃"
Inactive(file):  3500000 kB
Unevictable:       30000 kB     mlock 住的不可回收
Mlocked:           30000 kB
SwapTotal:       8000000 kB
SwapFree:        7900000 kB     swap 空闲
Dirty:             80000 kB     ★ 待回写的脏页（高了说明 IO 跟不上）
Writeback:          2000 kB     正在回写中的
AnonPages:      12000000 kB     ★ 匿名页（进程 heap/stack）
Mapped:          1500000 kB     文件映射页
Shmem:            300000 kB     tmpfs / shared memory
KReclaimable:    1200000 kB     ★ 可回收的内核内存（主要是 Slab 中带 reclaim 标记的）
Slab:            2000000 kB     ★ Slab 总量
SReclaimable:    1200000 kB     Slab 中可回收的部分（dentry、inode）
SUnreclaim:       800000 kB     Slab 中不可回收的（task_struct 等）
KernelStack:       80000 kB     所有内核栈
PageTables:       300000 kB     ★ 页表自身占用（大进程可能占很多）
VmallocTotal:   34359738367 kB
VmallocUsed:      400000 kB
HugePages_Total:       0
HugePages_Free:        0
```

**关键字段的"陷阱"**：

- `MemFree` ≠ 你能用的。它只是"完全没人摸的"。绝大多数服务器 MemFree 都很小，因为内核会拿空闲页做 PageCache。
- `MemAvailable`（3.14+ 引入）才是你真正关心的。它的算法在内核里：

```c
// mm/page_alloc.c 简化
long si_mem_available(void) {
    long available = freepages
                    + pagecache (扣掉一部分必须保留的) 
                    + slab_reclaimable (扣掉一部分);
    available -= totalreserve_pages;
    return available;
}
```

- `Cached` 包含了 PageCache + tmpfs（`Shmem`）。如果你 tmpfs 用得多，要扣掉。
- `Dirty` 高于 `vm.dirty_background_bytes` 就会触发后台回写；高于 `vm.dirty_bytes` 会**阻塞所有 write**直到刷下去。

### 2.2 /proc/\<pid\>/status —— 单进程视角

```bash
cat /proc/12345/status | grep -E 'Vm|Rss|Threads'
```

```
VmPeak:    9876000 kB     ★ 峰值虚拟（历史最高 VIRT）
VmSize:    8200000 kB     当前虚拟（= top 的 VIRT）
VmHWM:     2500000 kB     ★ 峰值 RSS（历史最高物理占用）—— 排查"曾经吃过多少内存"必看
VmRSS:     2100000 kB     ★ 当前 RSS（= top 的 RES）
RssAnon:   1900000 kB     RSS 中匿名页（heap/stack）
RssFile:    180000 kB     RSS 中文件映射页
RssShmem:    20000 kB     RSS 中共享内存
VmData:    2000000 kB     数据段大小
VmStk:        260 kB      栈大小
VmExe:       4500 kB      代码段
VmLib:      80000 kB      共享库
VmSwap:    100000 kB      ★ 这个进程被换到 swap 的字节数
Threads:        125
```

实战要点：
- **`VmHWM`** 是排查"OOM 时它到底涨到多大"的核心字段（OOM 后进程退出，VmRSS 归零看不到了）
- **`VmSwap`** 高了说明这个进程在 swap thrashing，应该排到嫌疑名单
- **`RssShmem`** 和别的进程共享，多进程加总会重复计数 —— 用 `smem` 才能算 PSS

### 2.3 /proc/\<pid\>/smaps —— 看每段映射

```bash
cat /proc/12345/smaps | head -50
```

```
00400000-00432000 r-xp 00000000 fd:00 1234   /usr/bin/myapp
Size:                200 kB
Rss:                 200 kB
Pss:                 200 kB     ★ Proportional Set Size，共享内存按比例分摊
Shared_Clean:          0 kB
Shared_Dirty:          0 kB
Private_Clean:       200 kB
Private_Dirty:         0 kB
Referenced:          200 kB
Anonymous:             0 kB
LazyFree:              0 kB
AnonHugePages:         0 kB
Locked:                0 kB
VmFlags: rd ex mr mw me
```

**PSS** 是排查多进程内存共享时的"公平计量"：一个 4KB 页被 N 个进程共享，每个进程分摊 4/N KB。`smem` 工具就是按 PSS 排序的。

### 2.4 /proc/buddyinfo —— 内存碎片

```bash
cat /proc/buddyinfo
```

```
Node 0, zone      DMA      1      1      1      1      0      0      0      0      0      1      3
Node 0, zone    DMA32   2345   1789   1234    567    234    100     45     12      3      1      0
Node 0, zone   Normal  12345   6789   2345    234     12      1      0      0      0      0      0
                       ^^^^^                                                                ^^^^
                       4KB 块多                                                            4MB 块没了
```

读法：每行最右边的数代表 order=10（4MB）的块数。如果右侧 0 了 → 大块没了 → 碎片化严重。

应对（生产很少触发，但必须知道）：

```bash
# 整理碎片（会暂停短时间）
echo 1 > /proc/sys/vm/compact_memory
```

### 2.5 /proc/slabinfo —— 谁在吃内核内存

```bash
sudo cat /proc/slabinfo | head
```

```
slabinfo - version: 2.1
# name             <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> ...
dentry              3245678 3245678   192    21    1
xfs_inode           1245678 1245678  1088    15    4
kmalloc-1024         234567  234567  1024     4    1
task_struct           12345   13000  9920    13   32
sock_inode_cache       3456    4000   704    23    4
```

`slabtop` 是它的友好版本。常见嫌疑：
- `dentry` 暴涨：应用频繁 stat 大目录 / 打开关闭文件
- `xfs_inode / ext4_inode_cache`：同上
- `task_struct`：进程数失控（fork 炸弹）
- `sock_inode_cache / TCP / UDP`：socket 没关

### 2.6 OOM Killer 源码（mm/oom_kill.c）

`oom_score` 的核心计算：

```c
// mm/oom_kill.c (简化，linux 6.x)
long oom_badness(struct task_struct *p, unsigned long totalpages) {
    long points;
    long adj;

    if (oom_unkillable_task(p))         // init / kthread / oom_score_adj=-1000
        return LONG_MIN;                 // 永远不杀

    p = find_lock_task_mm(p);
    if (!p) return LONG_MIN;

    adj = (long)p->signal->oom_score_adj;
    if (adj == OOM_SCORE_ADJ_MIN) {     // -1000
        task_unlock(p);
        return LONG_MIN;
    }

    // ★ 核心计算：(RSS + swap + 页表) / 总页数 × 1000 + adj
    points = get_mm_rss(p->mm) +
             get_mm_counter(p->mm, MM_SWAPENTS) +
             mm_pgtables_bytes(p->mm) / PAGE_SIZE;
    task_unlock(p);

    points = points * 1000 / totalpages;
    points += adj;

    return points;
}
```

**逐段点透**：

- `OOM_SCORE_ADJ_MIN` (-1000) 直接返回 LONG_MIN → **完全免疫**。生产关键进程（监控、SSH、init）必设。
- 计算包含 `MM_SWAPENTS`（已 swap 出去的页）和 `mm_pgtables_bytes`（页表本身）。**页表大的进程也容易被杀** —— 这是为什么大量小进程比少量大进程更容易触发整体 OOM。
- `points × 1000 / totalpages` —— 是相对总内存的比例。所以"占总内存 30% 的进程" oom_score ≈ 300。再加上 `oom_score_adj`（-1000~+1000）。

容器内 OOM Killer：cgroup 内独立计算，只在 cgroup 内挑进程杀。但**杀进程后 cgroup 自己不会消失**，K8s 看到主进程死了才会重启容器。

### 2.7 cgroup memory.high / memory.max 的语义差别

```bash
# v2 路径（K8s 1.25+ 默认）
cat /sys/fs/cgroup/kubepods.slice/.../memory.max
cat /sys/fs/cgroup/kubepods.slice/.../memory.high
cat /sys/fs/cgroup/kubepods.slice/.../memory.current
cat /sys/fs/cgroup/kubepods.slice/.../memory.stat
```

| 字段 | 行为 |
| --- | --- |
| `memory.max` | 硬限。超了就 OOM Kill cgroup 内进程 |
| `memory.high` | 软限。超了进程被 throttle（强制回收），不杀 |
| `memory.current` | 当前用量 |
| `memory.swap.max` | swap 上限（v2 独立控制 swap） |

`memory.stat` 是诊断金矿：

```
anon                   2147483648    匿名页
file                   1073741824    PageCache
kernel_stack             16777216    内核栈
slab                    104857600    Slab
sock                     50331648    socket buffer
shmem                    20971520    shared memory
file_mapped              80000000    mmap 文件
file_dirty                1048576    cgroup 内的脏页
file_writeback              524288    回写中
inactive_anon           500000000
active_anon            1647483648
inactive_file           300000000
active_file             773741824
pgfault                 50000000     总 page fault 数
pgmajfault                  12000    ★ major fault（要从磁盘读，包括 swap-in）
oom                            0     OOM 触发次数
oom_kill                       0     被 OOM 杀死的进程数
```

**`pgmajfault` 持续增长** = swap thrashing 或 PageCache 反复换入换出，应用 P99 必抖。

> **口诀**：`/proc/meminfo` 看全局，`/proc/<pid>/status` 看单进程峰值（VmHWM），`smaps` 看分段，`buddyinfo` 看碎片，`slabinfo` 看内核内存，cgroup `memory.stat` 看容器内每一项的来源。

---

## 3. 实测案例：模拟、观测、对比

### 3.1 用 stress-ng 模拟内存压力，观察各指标变化

```bash
# 先看基准
free -h
cat /proc/meminfo | grep -E 'MemFree|MemAvailable|Cached|Dirty'

# 灌 8GB 匿名内存，持续 60s
stress-ng --vm 4 --vm-bytes 2G --vm-keep --timeout 60s &

# 同时另一窗口持续观察
watch -n 1 'free -h; echo "----"; cat /proc/meminfo | grep -E "MemFree|MemAvailable|Cached|Dirty|SwapFree"'
```

典型变化（32GB 机器，原本 PageCache 8GB）：

| 时刻 | MemFree | MemAvailable | Cached | Dirty | SwapFree |
| --- | --- | --- | --- | --- | --- |
| 0s（基准） | 500M | 9G | 8G | 50M | 8G |
| 10s（灌到 4G） | 400M | 5G | 5G | 50M | 8G |
| 30s（灌到 8G） | 300M | 1G | 2G | 50M | 8G |
| 50s（灌满 + swap） | 100M | 200M | 500M | 50M | 6G |
| 60s（stress 退出） | 8G | 16G | 2G | 50M | 6G ★ |

**两个观察**：
1. **Cached 急剧下降** —— PageCache 被回收换给匿名页用。这就是"available 8G 是真的能用"的实证。
2. **stress 退出后 SwapFree 没立即恢复** —— swap 出去的页要等下次访问才会被换回。

### 3.2 用 slabtop 找谁占内核内存

```bash
slabtop -o -s c | head -20
```

```
 OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
8245k 8245k 100%    0.19K 196314       42   1570520K dentry
2245k 2245k 100%    1.07K  77380       29   2476160K xfs_inode
 845k  845k 100%    0.50K  26562       32    425000K kmalloc-512
 234k  234k 100%    0.06K   3656       64     14624K vm_area_struct
 ...
```

dentry 1.5GB + xfs_inode 2.4GB —— 加起来近 4GB 内核内存被文件系统缓存占用。如果业务确实需要随时打开关闭海量小文件，这是合理的；如果不是，可能是泄漏（案例 ⑤）。

### 3.3 smem：进程内存的"公平视图"

`top` 看 RES 在多进程共享内存场景下会高估。`smem` 用 PSS：

```bash
smem -k -t -r | head -20
```

```
  PID User     Command                    Swap      USS      PSS      RSS
12345 mysql    mysqld --defaults-file=     0     8.5G    8.6G     9.0G
23456 java     java -jar app.jar           0     2.0G    2.1G     2.5G
34567 nginx    nginx: master process       0      50M    100M     500M  ← RSS 远高于 PSS
34568 nginx    nginx: worker               0      50M    100M     500M
...
                                          0    11.6G   11.9G    13.0G
```

nginx master + 4 个 worker 共享了一大块内存，RSS 加起来虚高，PSS 才反映真实占用。

### 3.4 容器内 vs 宿主的内存视图差异

容器内：

```bash
$ free -h
              total        used        free      shared  buff/cache   available
Mem:          252Gi       180Gi       30Gi      2.0Gi       42Gi       65Gi
```

是不是觉得"容器还有 65G 可用，应该不会 OOM"？

**错。这是宿主的视图**。容器实际只能用 cgroup 限的 2GB。`free` 命令读的 `/proc/meminfo` 是宿主的，**没有 cgroup 隔离**。

正确做法：

```bash
# 容器内看自己的限额
cat /sys/fs/cgroup/memory.max         # cgroup v2
cat /sys/fs/cgroup/memory/memory.limit_in_bytes  # v1

# 当前用量
cat /sys/fs/cgroup/memory.current

# 详细分类
cat /sys/fs/cgroup/memory.stat
```

K8s 里更常用：

```bash
kubectl top pod my-pod              # 当前用量
kubectl describe pod my-pod         # spec 里看 limits.memory
```

### 3.5 swap on/off 的性能影响

简单 benchmark：让进程访问刚好超过物理内存的工作集，对比 swap 开/关。

```bash
# 实验：8GB 物理 + 8GB swap，工作集 10GB
stress-ng --vm 1 --vm-bytes 10G --vm-method walk-rev --timeout 60s --metrics
```

| swap 配置 | 完成时间 | 平均 page fault/s | 备注 |
| --- | --- | --- | --- |
| swap 关闭 | OOMKilled | — | 直接被杀 |
| swap 开 + swappiness=60 | 完成，180s | 50000 | 极慢，明显抖动 |
| swap 开 + swappiness=1 | OOMKilled @ 120s | — | swappiness 接近 0 也救不了"工作集大于物理内存" |

**结论**：swap 不是"扩内存"，它只是"把不常用的页存到磁盘上"。**工作集大于物理内存时，swap 帮不了你**，只是把"快速死"换成"慢慢死"。

> **口诀**：测内存指标不能只看 free 输出，要交叉看 meminfo / status / stat。**容器内一切以 cgroup 接口为准**。

---

## 4. 生产案例 9 则

> 每个案例固定四块：现象 / 根因 / 修复 / 验证。这一节是本文长尾价值的核心。

### 案例 ① 容器 OOMKilled 但宿主还有内存 —— "我们机器明明有 200G"

#### 现象
K8s 集群里某 Java 服务频繁被 OOMKilled。`kubectl describe pod` 看到：

```
Last State: Terminated
  Reason: OOMKilled
  Exit Code: 137
```

但登上宿主机 `free -h` 一看：

```
              total        used        free
Mem:          252Gi        80Gi      170Gi
```

宿主有 170G 空闲，容器为什么 OOM？运维一脸懵。

#### 根因
看 deployment yaml：

```yaml
resources:
  limits:
    memory: "2Gi"
  requests:
    memory: "1Gi"
```

容器实际只能用 **2GB**。宿主有多少跟它没关系。

进一步看宿主 `dmesg`：

```
[1234567.890] Memory cgroup out of memory: Killed process 12345 (java) 
              total-vm:8521340kB, anon-rss:2097152kB, file-rss:50000kB, 
              shmem-rss:0kB, UID:0 pgtables:8192kB oom_score_adj:1000
```

注意 "**Memory cgroup** out of memory"。`anon-rss:2097152kB` ≈ 2GB，正好顶到 limit。

为什么 oom_score_adj 是 1000？K8s 给 Burstable / BestEffort QoS 的 Pod 调高 oom_score，就是为了让它们在节点级别 OOM 时优先被杀。

#### 修复
两个独立的问题：

**问题 A：limit 太小** —— 看监控 P99 内存用量。如果常态接近 2GB，调高 limit。

```yaml
resources:
  requests:
    memory: "2Gi"
  limits:
    memory: "4Gi"   # 留 2x 安全余量
```

**问题 B：JVM 没识别 cgroup 限额** —— 老版 JVM（< 8u191）不读 cgroup，按宿主 256GB 算堆，启动时直接申请几十 G，瞬间撞到 limit 被杀。

```bash
# 看 JVM 实际看到多少内存
jcmd <pid> VM.flags | grep MaxHeapSize
```

修复：

```bash
# JDK 8u191+ / 11+ 默认就 cgroup-aware，无需特殊配
# 但要用 RAM percentage 控制堆比例
java -XX:MaxRAMPercentage=75.0 -jar app.jar
# 不要用 -Xmx 写死（容器迁移到不同 limit 的环境会出问题）
```

`75%` 是经验值：剩下 25% 给堆外（DirectByteBuffer、Metaspace、线程栈、JIT code cache）。

#### 验证
```bash
# 容器内确认 JVM 实际堆大小
kubectl exec my-pod -- jcmd 1 VM.flags | grep -E 'MaxHeapSize|InitialRAMPercentage'

# 期望 MaxHeapSize ≈ limit × 0.75
```

> **口诀**：容器 OOM = cgroup OOM，跟宿主无关。`dmesg` 里看到 "Memory cgroup out of memory" 就是 cgroup 限额问题。**JDK 8u191+ 默认 cgroup-aware，但堆比例必须显式 `MaxRAMPercentage` 限制，留 25% 给堆外**。

---

### 案例 ② Page Cache 被惯性回收 —— "凌晨 IO 突然飙升"

#### 现象
线上某服务凌晨 2 点定时跑批量备份脚本（`tar` 全表数据到对象存储）。备份时 IO 正常。但**备份结束后 30 分钟内**，业务的 P99 飙到平时 5 倍，磁盘 IO 持续高位。

#### 根因
`tar` 顺序读了几百 GB 文件 → Page Cache 被 tar 的"一次性数据"塞满，把业务的"热数据 PageCache"全挤掉了。

备份结束后，业务发现"咦我的索引页不在 PageCache 里了"，全量 page fault 重新从磁盘读 → IO 飙升。

观察证据（备份期间）：

```bash
# Cached 急剧上升
watch -n 1 'cat /proc/meminfo | grep -E "Cached|Active\(file\)|Inactive\(file\)"'
```

```
Cached:        180000000 kB      → 业务热数据被冲走
Active(file):    5000000 kB
Inactive(file): 175000000 kB     ← tar 灌进来的"冷"数据
```

#### 修复
**方案 A**：让 tar 不污染 PageCache。

```bash
# nocache 工具，让进程通过 posix_fadvise 告诉内核"不缓存"
nocache tar czf backup.tar.gz /data

# 或者直接在脚本里加 dd ... iflag=direct 走 O_DIRECT 绕过 PageCache
```

如果脚本不易改：

```bash
# 备份完手动让 tar 写入的部分被尽快回收（小心用！）
sync && echo 3 > /proc/sys/vm/drop_caches
```

但 drop_caches 把所有 PageCache 都清了，包括业务热数据。**不推荐**。

**方案 B**：让业务关键文件 mlock 在内存里（适合数据库）。

```c
mlockall(MCL_CURRENT | MCL_FUTURE);
```

或者按文件粒度：

```c
mmap_addr = mmap(...);
mlock(mmap_addr, len);     // 这段范围永不被 swap 或 PageCache 回收
```

MySQL 用 InnoDB Buffer Pool 自己管理热数据，本质就是 mlock 一大块匿名内存做缓存，不依赖 PageCache。

**方案 C**：业务侧"PageCache 预热"。备份后立即跑一次预热查询，把索引重新拉回 PageCache。

#### 验证
```bash
# 备份前后对比 Inactive(file) 占比
cat /proc/meminfo | grep -E 'Active\(file\)|Inactive\(file\)'

# 业务的 pgmajfault 不应明显上升
sar -B 1 10
```

> **口诀**：**一次性大文件 IO 会污染 PageCache**。备份/导出脚本必须用 nocache / O_DIRECT / posix_fadvise(POSIX_FADV_DONTNEED)。生产数据库自己管缓存，别让 OS PageCache "好心办坏事"。

---

### 案例 ③ JVM 堆外内存泄漏 —— "RSS 远超 -Xmx"

#### 现象
JVM 配置 `-Xmx4g`，但容器 RSS 涨到 8GB 后被 OOMKilled。GC 日志显示堆才 2GB，远远没满。

```bash
$ jcmd 1 GC.heap_info
 garbage-first heap   total 4194304K, used 2097152K
```

但 `cat /proc/1/status | grep VmRSS`：

```
VmRSS:    8245000 kB
```

差了 4GB —— **堆外内存**在哪里？

#### 根因
JVM 实际占用 = 堆 + 堆外。堆外包括：

- **Metaspace**（类元数据）：默认无上限
- **Code Cache**（JIT 编译后的机器码）：默认 240MB
- **线程栈**：每线程 1MB（`-Xss`），1000 线程 = 1GB
- **DirectByteBuffer**（NIO Direct Buffer）：上限 `-XX:MaxDirectMemorySize`，默认 = `-Xmx`
- **JNI 分配的 native memory**
- **GC 自己用的内存**（G1 的 region 元数据等）
- **Jemalloc / glibc malloc 的内部碎片**

最常见嫌疑：**DirectByteBuffer 泄漏**。Netty / Kafka client / RocketMQ client 大量用它做零拷贝。

诊断步骤：

```bash
# 1. 开 NMT (Native Memory Tracking)，启动加参数
java -XX:NativeMemoryTracking=detail -jar app.jar

# 2. 运行后获取报告
jcmd 1 VM.native_memory summary
```

```
Total: reserved=10485760KB, committed=8388608KB
-                 Java Heap (reserved=4194304KB, committed=4194304KB)
-                     Class (reserved=1073741824KB, committed=512000KB)
-                    Thread (reserved=1048576KB, committed=1048576KB)
-                      Code (reserved=245760KB, committed=240000KB)
-                        GC (reserved=300000KB, committed=300000KB)
-                  Internal (reserved=2000000KB, committed=2000000KB)  ← ★ 异常大
-                    Symbol (reserved=20000KB, committed=20000KB)
-    Native Memory Tracking (reserved=12000KB, committed=12000KB)
-               Arena Chunk (reserved=200KB, committed=200KB)
```

`Internal` 段异常大，配合 Netty / 业务日志找到 DirectBuffer 泄漏点。

或者直接看 DirectByteBuffer 用量：

```bash
jcmd 1 VM.native_memory baseline
# ... 跑一段时间 ...
jcmd 1 VM.native_memory summary.diff   # 看增量
```

#### 修复
**方案 A**：限制 DirectMemory 上限。

```bash
java -XX:MaxDirectMemorySize=512m -jar app.jar
```

超过会抛 `OutOfMemoryError: Direct buffer memory`，让你早发现。

**方案 B**：Netty 应用切 jemalloc，碎片更小：

```bash
LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2 java ... 
```

碎片可以减少 30%~50%。

**方案 C**：定位泄漏代码 —— Netty 提供 `-Dio.netty.leakDetectionLevel=PARANOID` 帮你抓。

```java
// Netty 应用排查时加
System.setProperty("io.netty.leakDetectionLevel", "PARANOID");
```

会在日志打印没释放的 DirectBuffer 的栈。

**方案 D**：容器内存预算重新计算：

```
container limit = -Xmx + MaxDirectMemorySize + Metaspace 上限 + 线程栈 × 线程数 + 1GB 安全余量
```

例：`-Xmx 4g + MaxDirectMemorySize 512m + MaxMetaspaceSize 256m + 1024 线程 × 1m + 1g 安全 = 6.75g`。limit 设 7Gi。

#### 验证
```bash
# 持续观察 RSS 不应超过预算
while true; do
  cat /proc/1/status | grep -E 'VmRSS|VmHWM'
  sleep 10
done
```

> **口诀**：JVM RSS = 堆 + Metaspace + Code + Thread Stack + Direct + JNI + GC overhead。**`-Xmx` 只管堆，容器 limit 必须算总账**。RSS 远超堆 → 开 NMT + jemalloc 排查。

---

### 案例 ④ JVM 启动时分配过大堆 —— "刚启动就被杀"

#### 现象
某服务上线，K8s 看到 Pod 不停 CrashLoopBackOff。事件：

```
OOMKilled, exit code 137
```

但服务还没来得及打印任何日志。

#### 根因
查启动命令：

```yaml
command: ["java", "-jar", "app.jar"]
```

没设任何堆参数。容器 limit = 2GB。

JDK 8u191 之前 JVM **不识别 cgroup**，直接读宿主总内存（256GB），按"默认堆 = 总内存 / 4 = 64GB"申请。一启动就远超 limit，瞬间 OOMKilled。

JDK 8u191+ / 11+ 默认 cgroup-aware，但**默认 `MaxRAMPercentage=25%`**（保守）。如果你想用更多，必须显式调。另一个坑：**JDK 8 的 cgroup 支持仅在 cgroup v1 上工作**。如果集群升级到 cgroup v2（K8s 1.25+），老 JDK 又"瞎"了。

诊断：

```bash
# 看 JVM 实际用的堆大小
kubectl exec my-pod -- jcmd 1 VM.flags | grep -E 'MaxHeapSize|MaxRAMPercentage'

# 8u191+ 在 cgroup v1 下应输出
# MaxHeapSize = 限额 × 25% (默认) 或你设的 percentage
```

#### 修复
```dockerfile
# Dockerfile
FROM eclipse-temurin:21-jre   # 用支持 cgroup v2 的较新 JDK

ENV JAVA_OPTS="-XX:MaxRAMPercentage=75.0 \
               -XX:MaxDirectMemorySize=256m \
               -XX:MaxMetaspaceSize=256m \
               -XX:+UseG1GC \
               -XX:+ExitOnOutOfMemoryError \
               -XX:+HeapDumpOnOutOfMemoryError \
               -XX:HeapDumpPath=/data/dump"

ENTRYPOINT ["sh", "-c", "exec java $JAVA_OPTS -jar /app.jar"]
```

为什么用 `MaxRAMPercentage` 而不是 `-Xmx`？因为同一个镜像可能跑在不同 limit 的环境（dev 1G、prod 4G）。`MaxRAMPercentage` 自适应。

**`+ExitOnOutOfMemoryError`** 关键：JVM OOM 后立即退出，让 K8s 重启它，而不是继续在不健康状态运行。

**`+HeapDumpOnOutOfMemoryError`** + 挂载 PVC 到 `/data/dump` —— 拿到堆转储用 MAT 分析。

#### 验证
```bash
# 启动后立即看堆设置是否正确
kubectl exec my-pod -- jcmd 1 VM.flags | grep MaxHeap

# 业务跑一段时间，确认 RSS 不撞 limit
kubectl top pod my-pod
```

> **口诀**：**JVM 容器化必加 `MaxRAMPercentage` + `MaxDirectMemorySize` + `MaxMetaspaceSize`**。三个都不限是裸奔。集群 cgroup 升级（v1→v2）前，确认 JDK 版本支持。

---

### 案例 ⑤ Slab 异常膨胀 —— "dentry 涨到 30G"

#### 现象
某机器 `free -h` 看起来 used 高达 60GB，但 `top` 里所有进程 RSS 加起来才 20GB。其余 40GB 在哪？

```bash
$ cat /proc/meminfo | grep -E 'Slab|SReclaimable|SUnreclaim'
Slab:           42000000 kB    ★ 40GB
SReclaimable:   38000000 kB    ★ 36GB 可回收
SUnreclaim:      4000000 kB
```

Slab 占了 40GB。

```bash
$ slabtop -s c | head
 OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
245M  245M  100%   0.19K  5833333    42  29832000K dentry   ← 28GB
12M   12M   100%   1.07K   413793    29  10752000K xfs_inode
```

dentry 单项 28GB。

#### 根因
某离线分析任务 `find / -name '*.log'` 频繁扫描全盘几千万小文件。每个 stat 都创建 dentry 缓存，且 dentry **没有上限**，只在内存压力下才被回收。

但有 PageCache 的话，内核倾向于先回收 PageCache 而不是 Slab。所以 dentry 越积越多。

#### 修复
**临时**：手动回收。

```bash
# 只回收 dentry + inode，保留 PageCache
echo 2 > /proc/sys/vm/drop_caches
```

**长期**：
1. 调 `vm.vfs_cache_pressure`（默认 100，调到 200~500 提高回收倾向）

```bash
echo 200 > /proc/sys/vm/vfs_cache_pressure
# 或永久：
echo "vm.vfs_cache_pressure = 200" >> /etc/sysctl.conf
```

2. 业务侧改造：批量任务避免 `find` 全盘，改用 `mlocate` 数据库查询；或拆成增量扫描。
3. 容器内 cgroup 限制 Slab：cgroup v1 没接口；cgroup v2 的 `memory.max` 已经把 Slab 算进 cgroup 限额了，**容器内不会发生这种"宿主级 Slab 膨胀"**（这也是 v2 的一个进步）。

#### 验证
```bash
# 持续观察 Slab
watch -n 5 'cat /proc/meminfo | grep -E "Slab|SReclaim"'

# 内核内存账户（v2）
cat /sys/fs/cgroup/.../memory.stat | grep slab
```

> **口诀**：`free -h` 里"消失的内存"经常在 Slab。dentry / inode_cache 是头号嫌疑。**`vm.vfs_cache_pressure` 高调 + 业务避免暴力 find**。cgroup v2 把 Slab 计入 memory.max 是底层修复。

---

### 案例 ⑥ Swap 频繁 P99 抖动 —— "凌晨 P99 从 50ms 飙到 2s"

#### 现象
线上 Java 服务，业务 P99 平时 50ms。凌晨 3-5 点固定飙到 2s+，CPU、QPS 都正常，dump 看不出问题。

#### 根因
`sar -B 1` 看 page fault：

```
04:00:01      pgpgin/s pgpgout/s   fault/s majflt/s    pgfree/s pgscank/s pgscand/s pgsteal/s    %vmeff
04:01:01       2400.5    1200.3    8000.2  ★ 250.6     12000.0    8000.0    2000.0   10000.0     50.5
```

`majflt/s = 250` —— 每秒 250 次主 page fault。

```bash
$ cat /proc/meminfo | grep -i swap
SwapTotal:       8388608 kB
SwapFree:        2000000 kB         ← 6GB 已被 swap
SwapCached:       500000 kB
```

`vmstat 1`：

```
procs -----------memory----------    -----swap--    -----io----    ----system---- ----cpu----
 r  b   swpd   free   buff  cache    si    so    bi    bo   in   cs us sy id wa st
 5  2 6291456 200000  10000 800000  ★ 5000  3000  6000  4000 8000 12000 30 10 50 10  0
```

`si` (swap-in) `so` (swap-out) 都几千页/秒 → swap thrashing。

为什么是凌晨？凌晨某定时任务把全表数据加载到内存做对账，把业务的"热堆"挤到 swap 里。业务请求一来，page fault 把堆从 swap 读回 → 几毫秒到几十毫秒延迟，而不是几十纳秒。

#### 修复
**生产铁律**：JVM / 数据库 / 缓存类应用所在节点 **swap 必须关闭或 swappiness=1**。

```bash
# 临时关 swap
sudo swapoff -a

# 永久关：注释掉 /etc/fstab 里的 swap 行

# 不能关的话最低优先级
echo 1 > /proc/sys/vm/swappiness
echo "vm.swappiness = 1" >> /etc/sysctl.conf
```

K8s 节点默认要求关 swap（kubelet 1.22 之前强制）。1.22+ 可选开但要 `--fail-swap-on=false`。

更彻底：JVM 关键页 mlock：

```bash
# JDK 16+ 支持
java -XX:+UseTransparentHugePages -XX:+AlwaysPreTouch -jar app.jar
```

`+AlwaysPreTouch` 启动时把堆 touch 一遍，确保物理分配 + 不会被 swap（如果 swap 关了的话；swap 没关的话堆还是可能被换）。

cgroup v2 可以禁用容器内的 swap：

```yaml
# K8s 容器
resources:
  limits:
    memory: 4Gi
# 通过 kubelet 配置 memorySwap.swapBehavior: LimitedSwap (1.22+)
```

#### 验证
```bash
# si/so 必须长期为 0
vmstat 1

# major fault 应趋近 0
cat /proc/<pid>/stat | awk '{print $12}'   # majflt

# pgmajfault 不再增长
cat /sys/fs/cgroup/.../memory.stat | grep pgmajfault
```

> **口诀**：**P99 抖动看 swap 和 major fault**。生产 JVM/DB 节点 swap 必关。`vmstat` 的 si/so > 0 就是病灶。

---

### 案例 ⑦ NUMA 跨节点访问 —— "同样的代码在不同机器上慢一倍"

#### 现象
两台同型号双 socket 服务器（48 core / 256GB / 2 NUMA node），跑同样的 Redis，A 机器 P99 = 1ms，B 机器 P99 = 2ms。配置看不出任何差异。

#### 根因
B 机器的 Redis 进程被绑定到 node 0 的 CPU，但启动时 node 0 内存正好紧张，内核把它的内存分配到了 node 1。

```bash
# 看 Redis 物理内存分布在哪个 NUMA node
numastat -p $(pidof redis-server)
```

```
                          Node 0          Node 1           Total
                 --------------- --------------- ---------------
Numa_Hit                 200000.0        100000.0       300000.0
Numa_Miss                100000.0          0.00         100000.0    ← ★ 跨节点
Numa_Foreign                  0.00      100000.0       100000.0
Local_Node               150000.0       100000.0       250000.0
Other_Node                50000.0           0.00        50000.0     ← ★
```

`Other_Node` 50000 页 = 200MB 的内存被进程"远程"访问。每次访问比 local 慢 2x。

A 机器没这问题（运气好分配在 local node）。

#### 修复
**方案 A**：显式 NUMA 绑定。

```bash
# 把 Redis 进程及其内存都绑到 node 0
numactl --cpunodebind=0 --membind=0 redis-server /etc/redis/redis.conf
```

**方案 B**：让内核 auto-balance（内核 3.8+）。

```bash
echo 1 > /proc/sys/kernel/numa_balancing
```

内核会监控访问模式，自动迁移热页到正确的 node。代价是有 CPU 开销，且对极致延迟应用不够稳定。

**方案 C**：禁用 NUMA（粗暴但有效）。

```bash
# 启动参数加（GRUB）
numa=off
```

把所有内存当成一个 node，不再有 local/remote 区分，但对真有 NUMA 收益的场景是损失。

**MySQL/MongoDB 的最佳实践**：用 `numactl --interleave=all` 让内存均匀分布在所有 node 上。这样无论哪个 CPU 访问都"差不多"，避免最坏情况。

```bash
numactl --interleave=all mysqld
```

#### 验证
```bash
numastat -p <pid>          # Other_Node 应趋近 0
perf stat -e node-loads,node-load-misses ./your-test
```

> **口诀**：双 socket 服务器必看 NUMA。延迟敏感应用 `numactl --cpunodebind --membind` 强绑；通用数据库 `--interleave=all` 平均化。

---

### 案例 ⑧ available 充足但 malloc 失败 —— "碎片化"

#### 现象
服务突然报 "Cannot allocate memory"，但 `free -h` 显示 available 还有 8GB。

#### 根因
分两种情况：

**A：内核大块连续物理页耗尽（碎片化）**

应用在分配 HugePage、DMA 缓冲、网卡 ring buffer 等需要**物理连续**的内存。

```bash
$ cat /proc/buddyinfo
Node 0, zone   Normal  234567  12345    234     12      1      0      0      0      0      0      0
                                                                            ^^^^^
                                                                  order 4+ 全是 0
```

虽然总量 8GB 可用，但全是离散的小块。需要 1MB 连续（order 8）的请求会失败。

**B：应用层分配上限（vm.max_map_count、ulimit）**

```bash
$ cat /proc/sys/vm/max_map_count
65530
$ wc -l /proc/<pid>/maps
65530    ← 已经达到上限
```

每次 mmap 算一个 vma。Java/Elasticsearch/某些分布式系统会创建大量 mmap 区域，撞这个上限。

或者：

```bash
$ ulimit -v
1048576    ← virtual memory 限制
```

#### 修复
**对 A 碎片化**：

```bash
# 1. 整理碎片
echo 1 > /proc/sys/vm/compact_memory

# 2. 调高 min_free_kbytes，留更多连续内存做 buffer
echo 1048576 > /proc/sys/vm/min_free_kbytes   # 1GB

# 3. 关闭透明大页 THP（数据库强烈推荐）
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

THP 让 2MB 的连续物理页常态分配，碎片化加剧。MongoDB / Redis / MySQL 文档都建议关闭 THP。

**对 B 上限**：

```bash
# Elasticsearch 经典配置
echo "vm.max_map_count = 262144" >> /etc/sysctl.conf
sysctl -p

# ulimit 在 systemd unit 文件里设
[Service]
LimitAS=infinity
LimitNOFILE=1048576
```

#### 验证
```bash
# 碎片整理后 buddyinfo 高 order 块应回升
cat /proc/buddyinfo

# 应用 mmap 区数
wc -l /proc/<pid>/maps
```

> **口诀**：**malloc 失败但 available 充足 = 三选一**：物理碎片化（buddyinfo 高 order 0）、`vm.max_map_count` 撞顶、ulimit 限制。先看 dmesg 有没有 "page allocation failure"。

---

### 案例 ⑨ JNI 内存泄漏定位 —— "RSS 涨但堆不涨"

#### 现象
某服务（用 Netty + 自研 JNI 加密库）跑两周后 RSS 从 4GB 涨到 12GB。GC 日志显示堆稳定 2GB。NMT 看 Internal 段也不大。

#### 根因
怀疑 JNI 调用的 native 库自己 malloc 后没 free。NMT 不追 native 库内的 malloc（它只追 JVM 自己的）。

诊断步骤：

**步骤 1**：jemalloc + 它的 leak detection

```bash
# 启动加
LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2 \
MALLOC_CONF="prof:true,prof_prefix:/tmp/jeprof,lg_prof_interval:30,lg_prof_sample:17" \
java -jar app.jar
```

跑一段时间，jemalloc 会 dump 出 heap profile 到 `/tmp/jeprof.*`。

```bash
# 用 jeprof 工具分析
jeprof --show_bytes --pdf /usr/bin/java /tmp/jeprof.*.heap > leak.pdf
```

PDF 里就能看到调用栈 + 内存归属，找出哪个 native 函数在泄漏。

**步骤 2**：另一个工具 `pmap` 看进程内存映射

```bash
pmap -x <pid> | sort -k 3 -n -r | head -20
```

```
00007f1234567000  524288  524288  524288 rw---   [ anon ]   ← 一大块匿名 mmap
00007f1287654000  262144  262144  262144 rw---   [ anon ]
...
```

如果有大量同样大小的 anon 段，可能是某个 native 库重复 malloc 大块。

#### 修复
找到泄漏代码后：

```c
// Native 代码必须配对
void* buf = malloc(size);
// ... 用完
free(buf);   // ★ 别忘
```

JNI 调用 Java 层时也要正确释放：

```c
jbyteArray arr = (*env)->NewByteArray(env, size);
// ... 用完
(*env)->DeleteLocalRef(env, arr);   // 不释放也会被 JVM 在方法返回时清掉
                                     // 但循环里大量创建 LocalRef 会撑爆 LocalRef 表
```

经验：**循环里大量调 JNI 必须主动 DeleteLocalRef**，否则 LocalRef 表撑爆抛 OOM。

#### 验证
- jemalloc 的 heap profile diff 看泄漏点是否消失
- 长时间压测 RSS 应趋稳，不再线性增长

> **口诀**：**JNI 泄漏不在 NMT 范围内**。jemalloc + heap profile 是金标准。`pmap` 看大块 anon 段是辅助手段。JNI 循环里主动 `DeleteLocalRef`。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 节点配置层
- [ ] swap 关闭或 `vm.swappiness ≤ 1`（数据库 / JVM / 缓存类节点强制）
- [ ] `vm.overcommit_memory` 明确（默认 0；fork 大进程的 Redis 类设 1）
- [ ] `vm.max_map_count ≥ 262144`（Elasticsearch / 大量 mmap 应用）
- [ ] `vm.vfs_cache_pressure = 200`（不缓存大量小文件场景）
- [ ] 关闭透明大页 THP：`echo never > /sys/kernel/mm/transparent_hugepage/enabled`
- [ ] NUMA 应用显式 `numactl` 绑定 / interleave
- [ ] 关键守护进程 `oom_score_adj = -1000`（init / 监控 agent / SSH）

### JVM 容器化层
- [ ] JDK 版本 ≥ 11（cgroup v2 完整支持需要 JDK 15+）
- [ ] **必加** `-XX:MaxRAMPercentage=75`（不要 `-Xmx` 写死）
- [ ] **必加** `-XX:MaxDirectMemorySize`（不限就是裸奔）
- [ ] **必加** `-XX:MaxMetaspaceSize`（防止 Metaspace 无限涨）
- [ ] `-XX:+ExitOnOutOfMemoryError`（OOM 立即退出，让 K8s 重启）
- [ ] `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dump`
- [ ] 必要时启用 NMT：`-XX:NativeMemoryTracking=summary`（开销 < 5%）
- [ ] Netty 应用 `LD_PRELOAD jemalloc`
- [ ] 容器 limit ≥ `Xmx + Direct + Metaspace + 线程栈 × 线程数 + 1G 安全余量`

### cgroup / K8s 层
- [ ] `requests` 和 `limits.memory` 都设（避免 BestEffort QoS）
- [ ] 关键服务用 Guaranteed QoS（requests = limits）
- [ ] livenessProbe / readinessProbe 配置正确，不会因 OOM 后被加速重启
- [ ] cgroup v2 用 `memory.high` 实现"限速回收"而不只是 hard kill

### 监控告警层
- [ ] `MemAvailable / MemTotal < 10%` 持续 5 分钟告警
- [ ] `pgmajfault` 速率 > 100/s 告警（swap thrashing 信号）
- [ ] cgroup `memory.current / memory.max > 80%` 告警
- [ ] cgroup `oom_kill > 0` 即时告警
- [ ] `dmesg` 持续监控 "Out of memory" / "page allocation failure"
- [ ] 单进程 `VmHWM` 趋势监控
- [ ] node_exporter 暴露 `/proc/buddyinfo` 高 order 块剩余

### 应用层
- [ ] 大文件 IO（备份 / 导出）用 `nocache` 或 `O_DIRECT`，不污染 PageCache
- [ ] 数据库自管缓存（Buffer Pool），不依赖 OS PageCache
- [ ] DirectByteBuffer 使用方加 `Cleaner` 正确释放
- [ ] JNI 循环主动 `DeleteLocalRef`
- [ ] 不要无脑用 `mlockall` —— 它会让所有页常驻，撑爆物理内存

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### cgroup v2 与 Memory QoS
- cgroup v2 的 `memory.high` 触发的回收路径（`mem_cgroup_handle_over_high`）
- K8s `MemoryQoS` 特性门控（Alpha → Beta 中），自动给 Burstable Pod 设 `memory.high`
- `memory.swap.max`：v2 才有的容器级 swap 限制
- `memory.events`：精确统计 high / max / oom 触发次数

### JVM 与 cgroup-aware 演进
- JEP 421（移除 SecurityManager）后的 JVM 启动开销变化
- `-XX:+UseContainerSupport`（默认 ON）的细节
- ZGC / Shenandoah 在容器内的内存回收特性
- Project Lilliput（压缩对象头）对内存占用的减少

### 内存回收路径（kswapd / direct reclaim）
- `kswapd` 异步回收 vs direct reclaim 同步回收
- 回收水位 `low / min / high` 的计算（`vm.min_free_kbytes`）
- LRU 链表的"两段式"老化（active / inactive）
- workingset 检测（4.0+ 的 refault detection）

### HugePage / 透明大页
- 显式 HugePage（`vm.nr_hugepages`）vs THP 的差别
- 数据库为什么禁用 THP（碎片化 + 同步整理 STW）
- 显式 HugePage + `mmap(MAP_HUGETLB)` 在 PostgreSQL / Redis 的应用
- 1GB HugePage 在 NUMA 大内存场景的收益

### 容器内存压测工具
- `stress-ng` 的各种 `--vm-method`（walk / flip / write64 等）
- `memtester` 物理内存测试
- `mmtests` 内核 mm 子系统的官方 benchmark suite
- K8s `LimitedSwap` + chaos engineering 模拟 swap 抖动

### Kubernetes Memory Manager
- K8s 1.21+ 的 Memory Manager（Alpha）：NUMA-aware 内存分配
- `kubelet --memory-manager-policy=static`
- 与 CPU Manager 协同（保证 CPU + 内存都在同一 NUMA node）

### eBPF 内存追踪
- BCC 的 `memleak`：自动追踪 malloc/free 配对，找泄漏
- `oomkill.bt`（bpftrace）：OOM 触发时打印完整上下文
- `slabtop` 替代品 `cachestat`、`cachetop`（PageCache 命中率监控）
- 自定义 eBPF：追踪某进程的 page fault 来源

### Linux MM 的演进方向
- Folios（5.16+）：把 page 抽象升级为变长 folio
- DAMON（5.15+）：基于采样的 working set 监测
- Multi-Gen LRU（6.1+）：替代两段 LRU 的多代回收算法
- Memory Tiering（CXL 时代的冷热分层）

---

## 7. 引用

- Linux 内核源码：[mm/](https://github.com/torvalds/linux/tree/master/mm) / [mm/oom_kill.c](https://github.com/torvalds/linux/blob/master/mm/oom_kill.c)
- 《Linux Kernel Development》—— Robert Love
- 《Systems Performance: Enterprise and the Cloud》—— Brendan Gregg（性能圣经）
- [Linux Memory Management Documentation](https://www.kernel.org/doc/html/latest/admin-guide/mm/index.html)
- [cgroup v2 Documentation](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)
- [JEP 7156366: cgroup awareness in JVM](https://bugs.openjdk.org/browse/JDK-8146115)
- [Brendan Gregg's blog: Linux Memory](https://www.brendangregg.com/blog/2017-01-11/cassandra-jit-cpu-flame-graphs.html)
- Netflix Tech Blog: [Linux Page Cache for SRE](https://netflixtechblog.com/linux-performance-analysis-in-60-000-milliseconds-accc10403c55)
- [Java Native Memory Tracking 官方文档](https://docs.oracle.com/en/java/javase/21/troubleshoot/diagnostic-tools.html#GUID-DA31D3F1-91D7-4D8F-B6BB-680BD8E9F0A2)
- [jemalloc Heap Profiling](https://github.com/jemalloc/jemalloc/wiki/Use-Case%3A-Heap-Profiling)

---

> 配套：[Linux 与性能工程主题](/article/linux-perf/) | [60 秒诊断](/article/linux-perf/60s-diagnosis) | [JVM GC 深入](/article/jvm/gc-deep) | [容器与 cgroup](/article/linux-perf/container-cgroup)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
