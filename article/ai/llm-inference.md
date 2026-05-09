# LLM 推理引擎深入 —— vLLM / TGI / llama.cpp 与服务化

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试或快速选型，看 [LLM 应用工程化](/article/ai/llm-app) 那一节就够了。
> 这里要回答的，是**自建推理服务、半夜被 OOM 告警叫起来、要给老板解释为什么 P99 飙到 30s** 时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. 同样是"前向传播"，为什么 LLM 推理要拆成 **Prefill** 和 **Decode** 两阶段？哪个阶段是 compute-bound、哪个是 memory-bound？
2. 一个 7B 模型 FP16 权重 14GB 是常识。但跑起来你的 80GB A100 还是 OOM —— **KV Cache** 凭什么能比权重大几倍？给一个 LLaMA-2-7B、batch=32、context=4096 的占用估算。
3. **PagedAttention** 借用了操作系统哪个经典思想？解决了 vLLM 之前的推理引擎里什么具体问题？给一个旧方案下"内存利用率 < 30%"的场景。
4. **Continuous Batching** 和 **Static Batching** 的差别，**不是**"动态调整 batch 大小"那么简单。说出 iteration-level scheduling 的本质：每个 token 步是不是都能让新请求进来？
5. **Speculative Decoding**（投机解码）为什么能"白嫖"加速？小模型猜错了不是浪费算力吗？为什么它**对 throughput 几乎没收益、只对 latency 有收益**？
6. **INT4 量化**（GPTQ / AWQ / GGUF Q4_K_M）相对 FP16 平均掉多少分？什么任务对量化最敏感（数学推理？长上下文？代码生成？），什么任务几乎无感？
7. 你有 4 张 A100-80G，要部署一个 70B 模型。**Tensor Parallel = 4** vs **Pipeline Parallel = 4** vs **2x2** —— 哪种延迟最低、哪种吞吐最高、哪种 PCIe 带宽要求最低？
8. vLLM、TGI、llama.cpp、Triton + TensorRT-LLM 四个引擎，**生产应该怎么选**？给一个判定流程：单机/多机？GPU 型号？是否需要 CPU 推理？是否要复用 ONNX 模型？

如果有半数答不上，往下看。

---

## 1. 底层模型：从 Transformer 推理到内存几何

> 这一层不讲怎么调用 vLLM 的 API，讲为什么 LLM 推理跟传统 ML 推理完全不是一回事。

### 1.1 Prefill 与 Decode：两个阶段、两种性能瓶颈

LLM 推理对一个请求做的事情，本质上分两阶段：

```
用户输入：  "请帮我写一首七律"   (假设 10 个 token)
              │
              ▼
     ┌─────────────────────┐
     │  Prefill  阶段       │  一次性把 10 个 token 送进所有层
     │  (prompt 处理)       │  → 产出第 1 个输出 token + 写满 10 token 的 KV Cache
     └─────────────────────┘
              │
              ▼ (生成第 1 个 token "诗")
     ┌─────────────────────┐
     │  Decode  阶段        │  ★ 每次只送 1 个 token（上一步刚生成的）
     │  (token by token)   │  → 产出第 N+1 个 token，追加 1 个 KV Cache 槽
     └─────────────────────┘
              │ 重复 N 次直到 EOS 或 max_tokens
              ▼
        最终输出
```

**两个阶段的性能特征截然不同**：

| 维度 | Prefill | Decode |
| --- | --- | --- |
| 一次处理的 token 数 | prompt 长度（几十~几千） | **1 个** |
| 算力（FLOPs）/ token | 满载 | 满载 |
| 内存读取 / token | 少（一次读权重，跑一长串 token） | **多**（每个 token 都要再读一遍权重） |
| 瓶颈类型 | **Compute-bound**（算力） | **Memory-bound**（带宽） |
| GPU 利用率 | 接近峰值 | 通常 5%~30% |
| 优化方向 | FlashAttention、tensor parallel | **Batching**、KV Cache 优化、Speculative |

**这个区分是后面所有引擎设计决策的根**。Decode 阶段 GPU 算力浪费严重，所以必须靠**批处理**把多个请求的 decode 合并起来；Prefill 阶段算力打满，多个 prefill 合并反而会被算力顶住。

### 1.2 KV Cache：你 OOM 的真凶

为什么 decode 阶段需要"缓存"？因为 self-attention 的本质：

```
对于每一层 Transformer：
  Q = X · W_q       (当前 token 的 query)
  K = X · W_k       (当前 token 的 key)
  V = X · W_v       (当前 token 的 value)

  Attention(Q, K_all, V_all) = softmax(Q · K_all^T / √d) · V_all

  其中 K_all 和 V_all 是「从第 1 个 token 到当前 token」所有的 K 和 V
```

如果不缓存，每生成 1 个新 token 都要把前面所有 token 的 K、V 重新算一遍 —— **平方复杂度**。所以推理引擎都把每个 token 算出的 K/V 存下来，下一步直接复用。这就是 **KV Cache**。

**单 token 单层的 KV 占用**：

```
KV_per_token_per_layer = 2 × num_kv_heads × head_dim × dtype_bytes
                                                       ↑ FP16=2, INT8=1
```

**一条请求的总 KV 占用**：

```
KV_total = num_layers × seq_len × 2 × num_kv_heads × head_dim × dtype_bytes
```

LLaMA-2-7B 的具体数字：

| 参数 | 数值 |
| --- | --- |
| num_layers | 32 |
| num_kv_heads | 32（MHA） |
| head_dim | 128 |
| dtype | FP16（2 字节） |
| 单 token 占用 | 32 × 2 × 32 × 128 × 2 = **524288 字节 ≈ 0.5MB** |

所以：
- 一个 4096 token 的请求 → 约 **2GB** KV
- batch=32、平均 4096 token → **64GB**

**权重 14GB，KV 64GB**。这就是为什么"7B 模型在 80GB A100 上 OOM"的真相。

GQA（Grouped-Query Attention，LLaMA-3-70B 用 8 个 KV head 取代 64 个）让 KV 直接缩 8 倍 —— 这是 GQA 大行其道的根本原因，不是为了精度，是为了**让 KV Cache 装得下**。

> **口诀**：**模型权重决定起步成本，KV Cache 决定并发上限**。算 GPU 装载能力，永远拿 KV 算预算。

### 1.3 Continuous Batching：从"凑齐再发车"到"随到随上车"

传统 ML 推理的批处理模型（**Static Batching**）：

```
T0:   ┌─[req A: 10 tok]──┐
T1:   ├─[req B: 50 tok]──┤  → 凑齐 batch=4，一起送进模型
T2:   ├─[req C: 100 tok]─┤    Decode 到所有请求完成
T3:   └─[req D: 200 tok]─┘

    ★ 问题：A 第 10 token 就 EOS 了，但要等 D 跑完 200 步
      A 这个槽位空着浪费 190 步算力
      新来的 req E 必须等下一波 batch
```

**Continuous Batching**（也叫 **Iteration-level Scheduling**，Orca 论文 2022 提出，vLLM、TGI 都用它）：

```
T0:   [A-1][B-1][C-1][D-1]      ← batch=4
T1:   [A-2][B-2][C-2][D-2]
...
T10:  [B-10][C-10][D-10][E-1]   ← A 完成、E 立即上车
T11:  [B-11][C-11][D-11][E-2]
T49:  [C-49][D-49][E-39][F-1]   ← B 完成、F 上车
...
```

**核心改变**：调度的最小粒度不是"一个请求"，而是"**一个 token 步**"。每跑完一个 forward pass（一步 decode），调度器立刻：
1. 把已完成的请求踢出 batch
2. 把队列里等待的请求拉进 batch
3. 重新组装下一步的 input

效果：
- GPU 利用率从 ~30% 拉到 ~80%
- 平均延迟下降，因为新请求不用等

**实现难点**：每一步 batch 里的请求"年龄"不同（有的刚 prefill 完只有 1 个 token，有的已经 decode 了 100 步）。怎么把它们的 attention 计算合并到一个 kernel？答案：**variable-length sequences 的注意力计算** —— 这就是 FlashAttention 2/3 的 `varlen` 接口存在的意义。

### 1.4 PagedAttention：把虚拟内存搬进 GPU

vLLM（SOSP 2023 论文 *Efficient Memory Management for Large Language Model Serving with PagedAttention*）的核心创新。

**问题**：传统实现给每条请求预分配一块**连续**的 KV 显存。

```
预分配模型：每个槽 max_len=4096 token

req A 实际只用 50 token：  [█████░░░░░░░░░░░░░░░░░]  浪费 4046 槽
req B 实际只用 100 token： [██████████░░░░░░░░░░░░]  浪费 3996 槽
req C OOM 不能进来          ←(显存被预分配占了，没碎片合并)
```

实测内存利用率：vLLM 之前的引擎只有 **20%~40%**。

**PagedAttention 的方案**：把 KV Cache 切成固定大小的"**块**（block / page）"，**类比操作系统的虚拟内存分页**：

```
物理 KV 池（GPU 显存）：
[blk0][blk1][blk2][blk3][blk4][blk5][blk6][blk7]...

每个 block = 16 个 token 的 KV

req A 用了 50 token → 占 4 个 block：
  逻辑视图：[tok0..15][tok16..31][tok32..47][tok48..50]
  物理映射：blk5      blk2        blk7        blk0
                        ↑ 完全不连续

req B 用了 100 token → 占 7 个 block，物理也是分散的

每条请求维护一张 "block table"（页表）:
  req A: [5, 2, 7, 0]
  req B: [3, 1, 6, 4, ...]
```

**关键收益**：
- **零碎片**：所有 block 一样大，分配/回收即"页表更新"
- **内存利用率 96%+**（vLLM 论文实测）
- **支持 prefix sharing**：多条请求如果 prompt 前缀相同（比如 system prompt），可以**共享同一组 block**，引用计数维护

代价：attention kernel 不能像以前那样假设 KV 连续，需要专门的 `paged_attention_kernel`，从 block table 里拼接出逻辑连续的 K/V。

这个设计直接奠定了 vLLM 的统治地位。后来 SGLang、TensorRT-LLM、TGI 都跟进了。

### 1.5 FlashAttention：把 attention 关进 SRAM

attention 的标准实现：

```python
# naive
S = Q @ K.T / sqrt(d)        # [N, N]，要写到 HBM
P = softmax(S)                # [N, N]，再写一次 HBM
O = P @ V                     # [N, d]
```

问题：**N×N 矩阵在 HBM 上来回搬**。N=8192 时，S 矩阵 = 8192² × 2 = 128MB，反复读写带宽就是瓶颈。

**FlashAttention**（Tri Dao 2022/2023/2024）：把 Q、K、V 切块，**整个计算只在 SRAM（GPU 片上缓存）里跑**，HBM 上不存中间矩阵 S 和 P。

```
对每个 Q 块：
  对每个 K/V 块：
    在 SRAM 中算 S_ij = Q_i @ K_j.T
    online softmax 累积统计量（max、sum）
    用累积结果更新 O_i
  写回 O_i 到 HBM
```

收益：
- 内存占用从 O(N²) 降到 O(N)
- 速度快 2~4 倍（HBM 是瓶颈）
- 长上下文（N>32k）效果更明显

vLLM 0.4 起默认用 FlashAttention v2，0.5+ 在 H100 上用 FlashAttention v3（专用 Hopper 架构）。

### 1.6 Speculative Decoding：让小模型替大模型先猜

**问题**：decode 阶段一次只生成 1 个 token，但 GPU 算力其实有富余 —— **小 batch 下 GPU 是 memory-bound，多算几个 token 几乎不增加耗时**。

**思路**：用一个小的 draft 模型（比如 LLaMA-7B 给 LLaMA-70B 当 draft），先一口气猜 K 个 token，然后大模型**一次 forward**验证这 K 个 token 是不是它自己也会生成的。

```
大模型 (target):  Llama-70B
小模型 (draft):   Llama-7B

step:
  draft 模型连续生成 4 个候选: [t1', t2', t3', t4']
  ↓
  大模型一次 forward 同时计算这 4 个位置的概率分布
  ↓
  从前往后比对：
    t1' 在 target 分布里 → 接受
    t2' 在 target 分布里 → 接受
    t3' 不在 → 拒绝；从此处用 target 的真实 sample 重新生成 t3
    后面 t4' 直接丢弃
```

**为什么有收益**：
- draft 算 4 步只花一点算力（小模型快）
- target 一次 forward 验证 4 步（不是 4 次 forward） —— 这一步是 **memory-bound**，多算几列几乎不变慢
- 接受率 70%+ 时，平均 1 步 target forward = 输出 2~3 个 token

**收益的边界**：
- **Latency 受益巨大**：单条请求 token/s 可以提升 2~3 倍
- **Throughput 几乎无收益**：高并发场景下 GPU 已经被批处理填满了，多消耗的算力反而成本上升
- 接受率取决于 draft 模型质量、温度参数、任务类型

vLLM 0.5+ 支持。生产慎用 —— **只有 latency 敏感的场景才值**。

### 1.7 量化：FP16 → INT8 → INT4 → 1bit

**为什么量化**：把 16-bit 权重改成 4-bit 权重，**显存占用直接 1/4**，且 memory-bound 的 decode 阶段**速度反而变快**（搬数据少了）。

主流量化方案对比：

| 方案 | 位宽 | 量化对象 | 典型质量损失 | 工具 |
| --- | --- | --- | --- | --- |
| **FP16 / BF16** | 16 | — | 0（基线） | 默认 |
| **INT8 weight-only** | 8 | 仅权重 | < 1% | bitsandbytes、SmoothQuant |
| **INT8 W8A8** | 8 | 权重+激活 | 1%~2% | SmoothQuant、TensorRT-LLM |
| **GPTQ INT4** | 4 | 权重 | 1%~3% | AutoGPTQ、ExLlamaV2 |
| **AWQ INT4** | 4 | 权重（保护重要通道） | 1%~2% | AutoAWQ |
| **GGUF Q4_K_M** | ~4.5 | 权重（混合精度） | 2%~4% | llama.cpp |
| **NF4 (QLoRA)** | 4 | 权重（normal float） | 1%~2% | bitsandbytes，主要训练用 |
| **INT2 / 1-bit** | 1~2 | — | 灾难 | 仍是研究阶段 |

**对量化最敏感的任务**：
- 数学推理（GSM8K、MATH）—— 推理链一步错全盘错
- 长上下文（>32k）—— 累积误差
- 代码生成 —— 语法严格，一个 token 错就编译失败

**几乎无感的任务**：
- 短文本对话
- 摘要、翻译
- 简单分类

**生产建议**：
- 服务级别 SLA 严格 → INT8 W8A8（SmoothQuant）
- 显存吃紧但精度容忍 → AWQ INT4
- CPU / Mac 推理 → GGUF Q4_K_M（llama.cpp 标配）

### 1.8 GPU vs CPU 推理：边界在哪

| 场景 | 推荐 |
| --- | --- |
| 7B 以下模型、QPS < 1、个人用 | **CPU + llama.cpp + GGUF Q4** |
| Mac M 系列开发机 | **llama.cpp Metal 后端** |
| 7B 模型、QPS > 5 | **单卡 GPU + vLLM** |
| 70B 模型、低延迟 | **多卡 GPU + Tensor Parallel** |
| 多模型多租户 | **Triton + 多模型混部** |
| 边缘设备（手机） | **llama.cpp / MLC-LLM / MNN-LLM** |

CPU 推理的硬边界是**带宽**：DDR5-4800 约 38GB/s，跑 7B Q4_K_M（~4GB 权重）大概 8~10 token/s；跑 13B 就降到 4~5 token/s；跑 70B 基本不可用。

> **小结口诀**：Prefill 是算力题、Decode 是带宽题。KV Cache 决定显存预算，PagedAttention 解决碎片，Continuous Batching 解决调度。量化省显存换吞吐，Speculative 省延迟不省吞吐。

---

## 2. 工具切片：四大引擎的源码与代码

> 每段切片 < 30 行，标注版本，并讲反直觉点。

### 2.1 vLLM 的 Engine + Scheduler

vLLM 的核心架构（v0.6.x）：

```
┌─────────────────────────────────────────┐
│           AsyncLLMEngine                 │
│  ┌───────────┐         ┌──────────────┐  │
│  │ Scheduler │◄───────►│  BlockMgr    │  │
│  │           │         │  (KV pool)   │  │
│  └─────┬─────┘         └──────────────┘  │
│        │                                  │
│        ▼                                  │
│  ┌──────────────────────────────────┐    │
│  │  Worker (1 per GPU)              │    │
│  │   ├── ModelRunner                │    │
│  │   │     └── PagedAttention kernel│    │
│  │   └── CacheEngine                │    │
│  └──────────────────────────────────┘    │
└─────────────────────────────────────────┘
        │
        ▼
   gRPC / OpenAI-Compatible HTTP
```

Scheduler 的核心循环（简化自 `vllm/core/scheduler.py`）：

```python
def schedule(self) -> SchedulerOutputs:
    # 1. 把已经完成的请求清理掉
    self.running = [s for s in self.running if not s.is_finished()]

    # 2. 试图调度新的 prefill 请求（waiting -> running）
    prefill_seqs = []
    while self.waiting and self.can_allocate(self.waiting[0]):
        seq = self.waiting.popleft()
        self.allocate(seq)            # 分配 block table
        prefill_seqs.append(seq)
        self.running.append(seq)

    # 3. 给已 running 的 decode 请求分配新 block（如果 token 跨 block）
    decode_seqs = []
    for seq in self.running:
        if seq in prefill_seqs:
            continue
        if not self.append_slot(seq):
            # 显存不够 → swap 到 CPU 或抢占
            self.preempt(seq)
            continue
        decode_seqs.append(seq)

    return SchedulerOutputs(prefill=prefill_seqs, decode=decode_seqs)
```

**逐段点透**：
- **Prefill 和 decode 在同一 batch 里**：vLLM 0.6 之前是"chunked prefill"模式，把长 prefill 切片，每片混在 decode batch 里跑，避免长 prefill 阻塞所有 decode（这就是案例 ③ "长生成阻塞批处理"的解法）
- **`append_slot` 失败时抢占**：vLLM 选择**抢占低优先级请求**而不是让新请求等待 —— 这是为什么 vLLM 在过载时延迟分布更平稳
- **抢占两种策略**：`recompute`（丢掉所有 KV，重新跑 prefill）vs `swap`（把 KV 拷贝到 CPU 内存）。recompute 更省 PCIe，swap 更省算力，vLLM 默认 recompute

启动一个最小可跑的 vLLM 服务：

```bash
pip install vllm

# OpenAI 兼容服务
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Llama-2-7b-chat-hf \
  --tensor-parallel-size 1 \
  --gpu-memory-utilization 0.9 \
  --max-model-len 4096 \
  --port 8000
```

调用：

```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8000/v1", api_key="dummy")
resp = client.chat.completions.create(
    model="meta-llama/Llama-2-7b-chat-hf",
    messages=[{"role": "user", "content": "讲个笑话"}],
    stream=True,
)
for chunk in resp:
    print(chunk.choices[0].delta.content, end="", flush=True)
```

### 2.2 PagedAttention 的内存分页

`vllm/core/block_manager.py` 的核心数据结构：

```python
class BlockSpaceManager:
    def __init__(self, block_size: int, num_gpu_blocks: int, num_cpu_blocks: int):
        self.block_size = block_size          # 通常 16
        self.gpu_allocator = BlockAllocator(num_gpu_blocks)
        self.cpu_allocator = BlockAllocator(num_cpu_blocks)
        # seq_id -> List[PhysicalBlock]
        self.block_tables: Dict[int, List[PhysicalBlock]] = {}

    def can_allocate(self, seq: Sequence) -> bool:
        num_required = math.ceil(seq.get_len() / self.block_size)
        return self.gpu_allocator.free_blocks >= num_required

    def allocate(self, seq: Sequence):
        num_blocks = math.ceil(seq.get_len() / self.block_size)
        blocks = [self.gpu_allocator.allocate() for _ in range(num_blocks)]
        self.block_tables[seq.seq_id] = blocks

    def append_slot(self, seq: Sequence) -> bool:
        # decode 一步后是否需要分配新 block
        block_table = self.block_tables[seq.seq_id]
        if seq.get_len() % self.block_size == 1:
            # 跨入新 block
            new_block = self.gpu_allocator.allocate()
            if new_block is None:
                return False
            block_table.append(new_block)
        return True
```

**反直觉点**：
- block 大小 = 16 不是随便选的。太小 → block table 太长、kernel 跳转开销大；太大 → 末尾浪费严重。16 是综合实测的最优值
- `gpu_blocks` 启动时一次性预分配，**整个进程生命周期不再 cudaMalloc** —— 这是 vLLM 启动慢但运行稳定的原因

PagedAttention kernel 的简化签名：

```cuda
__global__ void paged_attention_kernel(
    float* __restrict__ output,           // [num_seqs, num_heads, head_dim]
    float* __restrict__ query,            // [num_seqs, num_heads, head_dim]
    float* __restrict__ key_cache,        // [num_blocks, block_size, num_kv_heads, head_dim]
    float* __restrict__ value_cache,      // 同上
    int* __restrict__ block_tables,       // [num_seqs, max_blocks_per_seq]
    int* __restrict__ context_lens        // [num_seqs]
) {
    int seq_idx  = blockIdx.x;
    int head_idx = blockIdx.y;
    int len      = context_lens[seq_idx];

    // 沿 block_table 扫描每个 block，做 softmax(Q·K) · V
    for (int block_pos = 0; block_pos < ceil(len / BLOCK_SIZE); block_pos++) {
        int physical_block = block_tables[seq_idx * MAX_BLOCKS + block_pos];
        // 从物理块加载 K, V 进 SMEM
        // 在 SMEM 里跑 attention（FlashAttention 风格的 online softmax）
    }
}
```

关键就一行：`physical_block = block_tables[seq_idx * MAX_BLOCKS + block_pos]` —— 通过页表把逻辑位置翻译成物理位置。

### 2.3 TGI（Text Generation Inference）的 Token Streaming

HuggingFace 的官方推理服务，Rust 写的 router + Python 写的 model server。

启动：

```bash
docker run --gpus all --shm-size 1g -p 8080:80 \
  -v $PWD/data:/data \
  ghcr.io/huggingface/text-generation-inference:2.3.0 \
  --model-id meta-llama/Llama-2-7b-chat-hf \
  --max-input-length 3500 \
  --max-total-tokens 4096 \
  --max-batch-prefill-tokens 4096 \
  --max-concurrent-requests 128
```

TGI 的 streaming 协议是 **Server-Sent Events (SSE)**，比 OpenAI 的 chunked JSON 更轻量：

```python
import requests
import json

with requests.post(
    "http://localhost:8080/generate_stream",
    json={
        "inputs": "讲个笑话",
        "parameters": {"max_new_tokens": 100, "temperature": 0.7},
    },
    stream=True,
) as r:
    for line in r.iter_lines():
        if line.startswith(b"data:"):
            data = json.loads(line[5:])
            print(data["token"]["text"], end="", flush=True)
```

**TGI 与 vLLM 的关键差异**：

| 维度 | TGI | vLLM |
| --- | --- | --- |
| 来源 | HuggingFace | UC Berkeley → 商业化 |
| 调度 | 基于 Rust 的 router | Python AsyncEngine |
| KV 管理 | 早期连续，2.0+ 跟进 PagedAttention | PagedAttention 原创 |
| 多模型 | 单进程单模型 | 单进程单模型 |
| OpenAI 兼容 | 部分（Messages API） | 完整 |
| Quantization | bitsandbytes / GPTQ / EETQ | AWQ / GPTQ / FP8 / INT8 |
| 生产依赖 | HuggingFace Inference Endpoints | Anyscale、Mistral、字节大量自建 |

实际选型的**经验法则**：业务方需要稳定、官方背书、不愿意频繁升级 → TGI；研究方/极致吞吐/最新优化 → vLLM。

### 2.4 llama.cpp 的 GGUF 格式 + Metal/CUDA

llama.cpp 是 Georgi Gerganov 发起的 C++ 项目，最早只跑 LLaMA、现在支持几乎所有开源模型。**核心价值是把推理门槛拉到 Mac / 笔记本 / 树莓派**。

**GGUF（GPT-Generated Unified Format）** 是它的模型容器格式：
- 单文件包含权重、tokenizer、超参、量化方案
- 内嵌 metadata（chat template、prompt format）
- 原生 mmap，启动快
- 跨平台（Linux / macOS / Windows / Android）

构建 + 跑：

```bash
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp
make LLAMA_METAL=1            # Mac
# or:  make LLAMA_CUBLAS=1    # Linux+NVIDIA

# 下载量化好的模型
huggingface-cli download \
  TheBloke/Llama-2-7B-Chat-GGUF \
  llama-2-7b-chat.Q4_K_M.gguf \
  --local-dir models

# 跑起来
./llama-cli -m models/llama-2-7b-chat.Q4_K_M.gguf \
  -p "讲个笑话" \
  -n 256 \
  --threads 8 \
  --gpu-layers 35
```

`--gpu-layers 35` 表示**前 35 层放 GPU、剩下放 CPU**。这是 llama.cpp 的杀手锏 —— **CPU+GPU 混合推理**，让你在 24GB 显存上跑 70B 模型（虽然慢）。

启 server 模式（OpenAI 兼容）：

```bash
./llama-server -m models/llama-2-7b-chat.Q4_K_M.gguf \
  --port 8080 \
  --n-gpu-layers 35 \
  --ctx-size 4096
```

调用就用标准 OpenAI SDK 即可。

**Q4_K_M 是什么**：llama.cpp 自创的量化方案，一个 super-block 内不同子块用不同精度。`K` 表示 K-quants 系列，`M`（medium）表示精度档位。常见档位：
- `Q4_0` / `Q4_1`：早期方案，精度差
- `Q4_K_S` / `Q4_K_M` / `Q4_K_L`：现代方案，从小到大、从快到准
- `Q5_K_M`：5-bit 中档，质量好但体积大
- `Q8_0`：8-bit，几乎无损

实测：M2 Max 64GB 上 LLaMA-3-8B Q4_K_M 大概 40 token/s decode、Q8_0 大概 22 token/s。

### 2.5 Triton Inference Server 的 ONNX/TensorRT 后端

NVIDIA 的官方多模型推理服务器。它**不是 LLM 专用**，但可以通过 **TensorRT-LLM 后端**或 **vLLM 后端**接 LLM。

模型仓库结构（`model_repository/`）：

```
model_repository/
└── llama2-7b/
    ├── config.pbtxt
    └── 1/
        └── trt_engine/        ← TensorRT-LLM 编译产出
```

`config.pbtxt` 示例：

```protobuf
name: "llama2-7b"
backend: "tensorrtllm"
max_batch_size: 64

input [
  { name: "input_ids" data_type: TYPE_INT32 dims: [-1] }
]
output [
  { name: "output_ids" data_type: TYPE_INT32 dims: [-1] }
]

parameters: {
  key: "engine_dir"
  value: { string_value: "/models/llama2-7b/1/trt_engine" }
}

dynamic_batching {
  preferred_batch_size: [16, 32, 64]
  max_queue_delay_microseconds: 100
}
```

启动：

```bash
docker run --gpus all -p 8000:8000 -p 8001:8001 -p 8002:8002 \
  -v $PWD/model_repository:/models \
  nvcr.io/nvidia/tritonserver:24.08-trtllm-python-py3 \
  tritonserver --model-repository=/models
```

**Triton + TensorRT-LLM 的优势**：
- TensorRT-LLM 在 H100 / A100 上是**实测最快**的 LLM 推理引擎，比 vLLM 快 10%~30%
- Triton 支持**多模型混部**：一个 LLM、一个 embedding、一个 reranker 同进程
- 工业级监控（Prometheus 内置 metrics）

**劣势**：
- 编译 engine 慢（半小时起步）
- 模型升级 = 重编译
- TensorRT-LLM 对模型架构支持滞后于 vLLM 几周到几个月
- 学习曲线最陡

### 2.6 OpenAI-Compatible API：所有引擎的最大公约数

vLLM、TGI（部分）、llama.cpp、Ollama、LMDeploy、SGLang —— **几乎所有现代推理引擎都支持 OpenAI 兼容 API**。这意味着应用层代码可以"换引擎不换代码"。

最小的 client：

```python
from openai import AsyncOpenAI
import asyncio

client = AsyncOpenAI(base_url="http://localhost:8000/v1", api_key="anything")

async def call(prompt: str):
    resp = await client.chat.completions.create(
        model="any-name-here",  # 推理引擎通常忽略这个字段
        messages=[{"role": "user", "content": prompt}],
        max_tokens=200,
        stream=True,
    )
    async for chunk in resp:
        if chunk.choices[0].delta.content:
            print(chunk.choices[0].delta.content, end="", flush=True)

asyncio.run(call("用一句话解释 PagedAttention"))
```

**注意：兼容 ≠ 等价**。常见的"兼容性陷阱"（见案例 ⑥）：
- `logprobs` / `top_logprobs` 不是所有引擎都返回
- `response_format={"type": "json_object"}` 大多数引擎不支持
- `tools` / function calling 行为差异大
- `seed` 参数 vLLM 支持但 sampling temperature > 0 时仍可能不可复现

### 2.7 Tensor Parallelism：把一层切到多张卡

70B 模型 FP16 约 140GB，单张 80GB 卡装不下。两条路：

**Pipeline Parallel (PP)**：把 80 层 Transformer 按层切，第 1~40 层放卡 A、第 41~80 层放卡 B。Forward 时数据像流水线：A 算完一片就传给 B。
- 优点：实现简单
- 缺点：**有 bubble**（A 算第二批时 B 在等），且**单条请求延迟没改善**

**Tensor Parallel (TP)**：把每层的权重矩阵按列/行切。比如 attention 的 W_q [d, d] 切成 [d, d/2] + [d, d/2] 分别在两张卡上。Forward 时每张卡算一半，最后 all-reduce 合并。
- 优点：**单条请求延迟降低**（每张卡只算一半算力）
- 缺点：**all-reduce 通信量大**，对 NVLink 要求极高

**4 张卡的常见组合**：

| 配置 | 适用 | 延迟 | 吞吐 | 通信 |
| --- | --- | --- | --- | --- |
| TP=4 | 单机 4 卡有 NVLink | 最低 | 中 | 高（每层 all-reduce） |
| PP=4 | 多机、PCIe 互联 | 中 | 高 | 低 |
| TP=2 × PP=2 | 折中 | 中 | 高 | 中 |

vLLM 的 TP 配置：

```bash
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Llama-2-70b-chat-hf \
  --tensor-parallel-size 4 \
  --pipeline-parallel-size 1
```

**铁律**：TP 必须用在**同机**（NVLink 600GB/s+），跨机 TP（走 InfiniBand 200GB/s 或更差的以太网）通信会成为瓶颈。跨机用 PP。

> **代码段口诀**：vLLM 是默认选项；TGI 是 HF 生态；llama.cpp 是 CPU/Mac 救星；Triton+TRT-LLM 是 NVIDIA 极致优化。OpenAI-Compatible 让你换引擎像换车胎。

---

## 3. 实测数据：吞吐、延迟、量化质量

### 3.1 vLLM vs HuggingFace Transformers 的吞吐对比

**实验**：LLaMA-2-7B、A100-80G、ShareGPT 数据集（平均输入 200 tok、输出 200 tok），并发用户从 1 → 64。

```bash
# 用 vLLM 自带的 benchmark
python benchmarks/benchmark_serving.py \
  --backend vllm \
  --model meta-llama/Llama-2-7b-chat-hf \
  --dataset-name sharegpt \
  --dataset-path ShareGPT_V3_unfiltered_cleaned_split.json \
  --num-prompts 1000 \
  --request-rate 8
```

典型结果：

| 引擎 | 并发 1 | 并发 16 | 并发 64 | 显存利用率 |
| --- | --- | --- | --- | --- |
| **HuggingFace + accelerate** | 32 tok/s | 180 tok/s | 220 tok/s（OOM） | ~30% |
| **vLLM 0.6** | 92 tok/s | 1450 tok/s | **3800 tok/s** | ~95% |

**结论**：高并发下 vLLM 是 HF 的 **15~20 倍**。原因：
- HF 用 static batching，批内请求长度不齐时浪费严重
- HF 不用 PagedAttention，预分配浪费显存导致 OOM
- HF 用普通 attention，FlashAttention 也得手动开

### 3.2 PagedAttention 的内存利用率

vLLM 论文（SOSP 2023）数据：

| 引擎 | 内存利用率 | 同显存下能跑的 batch |
| --- | --- | --- |
| FasterTransformer (NVIDIA) | 38.2% | 16 |
| Orca (Continuous batching, no paging) | 60.7% | 28 |
| **vLLM (PagedAttention)** | **96.3%** | **64** |

实际验证方法：

```python
# 跑 vLLM 时观察
from vllm import LLM
llm = LLM(model="...", gpu_memory_utilization=0.9)

# 看 KV pool 用了多少
runner = llm.llm_engine.model_executor.driver_worker.cache_engine
total = runner.num_gpu_blocks
used = total - llm.llm_engine.scheduler.block_manager.gpu_allocator.free_blocks
print(f"KV blocks used: {used}/{total} = {used/total*100:.1f}%")
```

### 3.3 不同量化方案的延迟与质量对比

LLaMA-2-13B、A100-40G、batch=16、输出 200 tok：

| 量化 | 显存 | 单 token 延迟 | MMLU | GSM8K |
| --- | --- | --- | --- | --- |
| FP16 | 26GB | 18ms | 54.8 | 22.5 |
| INT8 (W8A8 SmoothQuant) | 13GB | 14ms | 54.5 | 22.0 |
| AWQ INT4 | 7GB | 11ms | 54.1 | 21.4 |
| GPTQ INT4 | 7GB | 12ms | 53.6 | 20.8 |
| GGUF Q4_K_M | 7.5GB | 35ms (CPU) | 53.2 | 19.6 |

**结论**：
- INT8 几乎无损、延迟更低（带宽减半）
- AWQ > GPTQ：AWQ 保护"重要通道"，质量更稳
- 数学推理（GSM8K）对量化更敏感（22.5 → 19.6 差 13%）
- MMLU 这种通用知识题几乎无感（54.8 → 53.2 差 3%）

### 3.4 不同 GPU 的吞吐

LLaMA-2-7B、vLLM 0.6、batch=64：

| GPU | 显存 | 吞吐 (tok/s) | 价格区间（云） |
| --- | --- | --- | --- |
| RTX 4090 | 24GB | 1800 | $0.4/h |
| L40 | 48GB | 2200 | $1.2/h |
| A10 | 24GB | 1100 | $0.8/h |
| A100-40G | 40GB | 3200 | $2.5/h |
| A100-80G | 80GB | 3600 | $3.5/h |
| H100-80G | 80GB | **6800** | $8/h |

**结论**：
- **H100 是性价比之王**（虽然贵）—— FlashAttention v3 + FP8 让它真的快 2 倍
- 4090 单卡推理性价比奇高（消费级，注意没 NVLink、不能多卡 TP）
- A100 仍是稳定的生产首选

### 3.5 CPU 推理（llama.cpp）的可用边界

M2 Max 64GB（10 核 P-core + Metal）实测：

| 模型 | 量化 | Token/s（decode） | 启动加载时间 |
| --- | --- | --- | --- |
| LLaMA-3-8B | Q4_K_M | 42 | 2s（mmap） |
| LLaMA-3-8B | Q8_0 | 24 | 3s |
| Mistral-7B | Q4_K_M | 48 | 2s |
| Qwen-2.5-14B | Q4_K_M | 22 | 3s |
| LLaMA-3-70B | Q4_K_M | 4.5 | 8s |
| LLaMA-3-70B | Q5_K_M | 3.2 | 10s |

DDR5 PC（i7-13700K + 64GB DDR5-5600，无独显）：

| 模型 | 量化 | Token/s |
| --- | --- | --- |
| LLaMA-3-8B | Q4_K_M | 11 |
| LLaMA-3-8B | Q8_0 | 6 |
| LLaMA-3-70B | Q4_K_M | 1.2（基本不可用） |

**结论口诀**：
- **CPU 推理的极限是带宽**：DDR5 双通道约 90GB/s，FP16 7B 模型 14GB → 理论 6 token/s
- **Mac 优势是统一内存**：M2 Max 200GB/s 带宽、零拷贝、Metal 加速
- **8B 以下个人用、Mac 可用；13B 边缘可用；30B+ CPU 走开**

> **数据口诀**：高并发选 vLLM；A100/H100 是当家花旦；INT8 是质量与速度的甜点；70B 以上 CPU 别碰。

---

## 4. 生产案例：9 则真实事故复盘

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① OOM：长 prompt 撑爆 KV Cache

#### 现象

线上某 7B 服务（vLLM、A100-40G）正常 QPS 跑得好好的，运营拉来一批"文档总结"任务，prompt 平均 6000 token。**第一波请求刚进来 vLLM 就 CUDA OOM 崩了**，全部 502。

#### 根因

启动配置：

```bash
python -m vllm.entrypoints.openai.api_server \
  --model llama2-7b \
  --gpu-memory-utilization 0.95 \
  --max-model-len 8192 \
  --max-num-seqs 256
```

算一下账：
- 模型权重 14GB
- 剩余 40 × 0.95 - 14 = **24GB 给 KV pool**
- 单 token KV = 0.5MB
- KV 容量 = 24 × 1024 / 0.5 = **49152 token**
- max_num_seqs=256 × max_len=8192 = **209万 token** ←★ 远超 KV 容量

vLLM 启动时**只检查 max_model_len 是否合法，不检查 max_num_seqs × max_model_len 是否够**。配置看似没问题，实际遇到长 prompt 一冲就爆。

更微妙的是：vLLM 的 OOM 不是漂亮的"等会再来"，而是**整个进程死掉**。一个长 prompt 进来 vLLM 试图分配 12 个 block 但池里只剩 4 个 → 抢占其他请求 → 还不够 → CUDA malloc → fail → assert → 进程退出。

#### 修复

**正确的容量规划公式**：

```
kv_capacity = (gpu_mem_total × gpu_memory_utilization - model_size) / kv_per_token

可同时跑的总 token 数应该 ≤ kv_capacity × 0.8 (留 20% 抢占缓冲)

→ max_num_seqs × max_model_len ≤ kv_capacity × 0.8
```

对应到上面的配置：
- 24GB / 0.5MB ≈ 49000 token
- 留 20% → 39000 token
- 如果 max_model_len = 8192 → max_num_seqs ≤ **4**！

要支持 max_num_seqs=256 怎么办？三条路：

1. **升 GPU**：换 80GB A100，KV 池涨到 64GB，能装 ~13万 token
2. **限 prompt**：网关层拒绝 input_token > 4000 的请求
3. **开 chunked prefill**：vLLM 0.4+ 提供，把 6000 token 的 prefill 切成 4 片，每片 1500 token，混在 decode batch 里跑

```bash
python -m vllm.entrypoints.openai.api_server \
  --model llama2-7b \
  --gpu-memory-utilization 0.9 \
  --max-model-len 8192 \
  --max-num-seqs 64 \
  --enable-chunked-prefill \
  --max-num-batched-tokens 2048
```

`max-num-batched-tokens = 2048` 表示**每步 forward 处理的总 token 数（prefill+decode 加起来）不超过 2048**。这把"瞬时显存峰值"压住了。

#### 验证

```bash
# 启动后看实际 KV 容量（vLLM 启动日志会打印）
grep "GPU blocks" vllm.log
# # GPU blocks: 1500, # CPU blocks: 256

# 1500 × 16 = 24000 token，与计算吻合

# 压测：用混合长度的 prompt 跑 30 分钟，看是否有 OOM
python benchmarks/benchmark_serving.py \
  --backend vllm \
  --num-prompts 5000 \
  --request-rate 10
```

> **口诀**：vLLM 启动**不会**警告你 `max_num_seqs × max_model_len > KV 池`。算完账再启动；超了开 chunked-prefill。

---

### 案例 ② 吞吐低：还在用 Static Batching

#### 现象

团队自研 LLM 服务，基于 transformers 库 + 自己写的"凑齐 batch=8 再 forward"逻辑。压测显示 QPS 上不去：单卡 A100 跑 7B 模型只有 80 token/s 总吞吐，远低于宣传值。

#### 根因

伪代码：

```python
def serve_loop():
    while True:
        batch = []
        deadline = time.time() + 0.05  # 50ms 凑批
        while time.time() < deadline and len(batch) < 8:
            req = queue.get(timeout=deadline - time.time())
            batch.append(req)

        # 凑齐 8 个或超时 → 一起跑
        outputs = model.generate(
            input_ids=pad_to_same_length([r.input_ids for r in batch]),
            max_new_tokens=200,
        )
        for r, out in zip(batch, outputs):
            r.callback(out)
```

致命问题：
- **同批次最长生成决定整批耗时**：8 个请求里 1 个生成 200 token，其他 50 token 就 EOS 了，但要等到第 200 步才能返回结果给所有人
- **新请求必须等下一批**：第 9 个请求至少等当前批完成（可能 5 秒+）
- **GPU 空闲槽位浪费**：50 步后 batch 实际有效请求只剩 1 个，但仍占 1/8 GPU

实测 GPU 利用率只有 20%~30%，正是 static batching 的典型征兆。

#### 修复

**直接换 vLLM** 或自己实现 continuous batching。

最小切换：

```python
# 删掉自研代码，全部走 vLLM
from vllm import AsyncLLMEngine, SamplingParams
from vllm.engine.arg_utils import AsyncEngineArgs

engine = AsyncLLMEngine.from_engine_args(AsyncEngineArgs(
    model="meta-llama/Llama-2-7b-chat-hf",
    gpu_memory_utilization=0.9,
    max_model_len=4096,
))

async def handle(req):
    sampling = SamplingParams(temperature=0.7, max_tokens=200)
    async for output in engine.generate(req.prompt, sampling, request_id=req.id):
        yield output.outputs[0].text
```

vLLM 内部就是 continuous batching，不用自己写调度。

如果业务必须自研（比如要做 multi-tenant 优先级），核心是 **iteration-level scheduling**：

```python
# 简化伪代码
class IterationScheduler:
    def __init__(self):
        self.running = []   # 已经在跑的 sequence
        self.waiting = []   # 队列里等的

    def step(self):
        # 1. 从 running 里踢掉已完成的
        self.running = [s for s in self.running if not s.is_finished()]

        # 2. 从 waiting 里塞进 running，直到显存满
        while self.waiting and self.has_kv_for(self.waiting[0]):
            self.running.append(self.waiting.popleft())

        # 3. 把所有 running 的下一步合并成一个 batch
        input_ids = build_variable_length_batch(self.running)
        logits = model.forward(input_ids, kv_cache=...)

        # 4. 每个序列独立采样下一个 token
        for s, l in zip(self.running, logits):
            s.append_token(sample(l))
```

#### 验证

```bash
# 压测对比：
# 自研 static batching: 80 tok/s, GPU 25%
# 切 vLLM:              1450 tok/s, GPU 92%
nvidia-smi dmon -s u -c 60   # 监控 GPU 利用率
```

> **口诀**：自己写 batching 是一年内的能力陷阱。**不要自研，用 vLLM/TGI**。除非你团队 ≥ 5 个全职 infra。

---

### 案例 ③ 长尾延迟：单个长生成阻塞批处理

#### 现象

vLLM 服务 P50 延迟很好（500ms），但 **P99 飙到 30s**。日志里没看到错误。

#### 根因

`vllm` 0.4 之前没有 chunked prefill。一个 8000 token 的 prefill 进来时：
- 占据整个 forward pass，所有正在 decode 的请求**等这一步 prefill 完成**
- 一个 prefill 大概 500ms~2s，期间所有 decode 请求"卡住"
- 如果 prefill 队列里堆了 5 个长 prompt → 早期的 decode 请求积累几十次卡顿 → P99 30s

观察方法：

```bash
# vLLM metrics
curl http://localhost:8000/metrics | grep -E "(prefill|decode)"

# vllm:time_to_first_token_seconds_bucket  ← prefill 阶段
# vllm:time_per_output_token_seconds_bucket ← decode 阶段
```

如果 TTFT P99 远高于 P50，且 TPOT（每 token 延迟）的标准差很大 → 长 prefill 阻塞 decode。

#### 修复

vLLM 0.4+ 的 `--enable-chunked-prefill`：

```bash
python -m vllm.entrypoints.openai.api_server \
  --model llama2-7b \
  --enable-chunked-prefill \
  --max-num-batched-tokens 2048
```

工作机制：8000 token 的 prefill 被切成 4 段，每段 2000 token；每段和当前的 decode 请求**混在一个 batch 里**跑。这样：
- prefill 不再独占 forward pass
- decode 请求每步都能前进，TPOT 稳定
- 总 prefill 时间不变（甚至略增），但 TTFT 略升、TPOT 大幅稳定

`max-num-batched-tokens` 选择：
- 太小（如 512）→ 长 prefill 切得太碎，prefill 总耗时显著上升
- 太大（如 8192）→ chunking 失效，又回到老问题
- 经验值：1024~4096，根据 GPU 算力调

另一个手段：**优先级队列**

```python
# vLLM 0.6+ 支持
sampling = SamplingParams(
    priority=1,            # 越大优先级越高
    max_tokens=200,
)
```

业务可以给"实时对话"高优先级、给"批量总结"低优先级，避免长任务挤占交互式请求。

#### 验证

```bash
# Grafana 看 P50/P95/P99
# vllm:e2e_request_latency_seconds_bucket

# 期望：P99 / P50 < 5x （之前可能 60x）
```

> **口诀**：单个长 prefill 是 P99 杀手。**chunked prefill 必开**；混合负载用优先级。

---

### 案例 ④ Quantization 精度崩塌

#### 现象

为了省显存，把生产模型从 FP16 切到 INT4（GPTQ）。**MMLU 基准从 54 掉到 38**，业务主观体验"变笨了一大截"，回滚。

#### 根因

GPTQ 量化的几个常见坑：

1. **校准集（calibration set）质量不够**：GPTQ 需要一组样本来"学习"量化参数。如果校准集与业务分布不匹配（比如用 wikitext 校准但业务是中文客服），量化误差大
2. **group_size 太大**：group_size=128 比 group_size=32 误差大很多
3. **act_order=False**：不开 act order，重要 channel 没有优先保护

更糟的是，**MMLU 这种一次性评测看不出问题**：模型可能在一般知识上还行，但**长上下文 / 数学 / 代码 / 多轮对话**严重退化。

#### 修复

**方案 A**：换 AWQ。AWQ 通过 activation magnitude 自动识别"重要权重通道"并保留高精度：

```python
from awq import AutoAWQForCausalLM
from transformers import AutoTokenizer

model_path = 'meta-llama/Llama-2-7b-chat-hf'
quant_path = 'llama-2-7b-chat-awq'

quant_config = {
    "zero_point": True,
    "q_group_size": 128,
    "w_bit": 4,
    "version": "GEMM"
}

model = AutoAWQForCausalLM.from_pretrained(model_path)
tokenizer = AutoTokenizer.from_pretrained(model_path)
model.quantize(tokenizer, quant_config=quant_config)
model.save_quantized(quant_path)
tokenizer.save_pretrained(quant_path)
```

实测 AWQ 平均比 GPTQ 高 1~2 个 MMLU 点。

**方案 B**：用业务数据做校准集

```python
from transformers import AutoTokenizer
from auto_gptq import AutoGPTQForCausalLM, BaseQuantizeConfig

# ★ 用业务真实样本做校准集，至少 256 条
calibration_samples = load_business_samples(256)
examples = [tokenizer(s, return_tensors="pt") for s in calibration_samples]

quantize_config = BaseQuantizeConfig(
    bits=4,
    group_size=32,            # ★ 不要用 128，用 32
    desc_act=True,            # ★ 开 act order
    sym=True,
)

model = AutoGPTQForCausalLM.from_pretrained(model_path, quantize_config)
model.quantize(examples)
model.save_quantized(quant_path)
```

**方案 C**：保守选择 INT8 而非 INT4

```bash
# vLLM 启动用 fp8 (H100) 或 int8 (A100)
python -m vllm.entrypoints.openai.api_server \
  --model llama2-13b \
  --quantization fp8       # H100 上原生 FP8 加速
```

INT8 几乎无质量损失，且 H100 上 FP8 速度反而比 FP16 快。

#### 验证

**关键：永远不要只看 MMLU**。建议跑：

```bash
# 跑多个评测，看哪个崩了
lm-eval --model vllm \
  --model_args pretrained=$MODEL,tensor_parallel_size=1 \
  --tasks mmlu,gsm8k,humaneval,longbench,ceval \
  --batch_size auto

# 重点看：
# - gsm8k（数学）
# - humaneval（代码）
# - longbench（长上下文）
# - ceval / cmmlu（中文）
```

业务侧再跑**自己的回归集**（500+ 条覆盖所有业务场景的 prompt + 期望输出关键词）。

> **口诀**：**INT4 是质量与显存的赌博**。能用 INT8 就用 INT8；要 INT4 用 AWQ + 业务校准 + 多维度评测。

---

### 案例 ⑤ 多卡推理通信瓶颈

#### 现象

8 张 A100 跑 70B 模型，TP=8，发现单 token 延迟比预想的高 2 倍。`nvidia-smi` 看 GPU 利用率只有 60%，但 PCIe 带宽接近 100%。

#### 根因

机器是 **PCIe 互联**而非 NVLink。TP=8 时每层都要 all-reduce：

- 单层 hidden_size=8192、batch=32、token=1（decode）
- all-reduce 数据量 = 8192 × 32 × 2 (FP16) = 524KB
- 80 层 × 2 次 all-reduce/层 = 160 次/token
- 总通信 = 160 × 524KB = **84MB/token**

PCIe Gen4 x16 = 32GB/s。84MB / 32GB ≈ 2.6ms 通信开销。
NVLink 600GB/s。84MB / 600GB ≈ 0.14ms 通信开销。

**差 18 倍**。这就是为什么 TP 强制要求 NVLink。

#### 修复

**方案 A**：换 PP（Pipeline Parallel）

```bash
python -m vllm.entrypoints.openai.api_server \
  --model llama2-70b \
  --tensor-parallel-size 1 \
  --pipeline-parallel-size 8
```

PP 只在层间传 hidden state（一次/层），通信量小。代价：有 pipeline bubble，吞吐略降但单卡延迟回归正常。

**方案 B**：拆 2 机 4 卡

如果机器内部是 NVLink，跨机是 InfiniBand，组合：
- 单机 4 卡 NVLink → TP=4
- 2 机间 InfiniBand → PP=2

```bash
# vLLM Ray 多机
ray start --head
# 另一台
ray start --address='<head_ip>:6379'

# 启动
python -m vllm.entrypoints.openai.api_server \
  --model llama2-70b \
  --tensor-parallel-size 4 \
  --pipeline-parallel-size 2
```

**方案 C**：监控通信占比

```bash
# Nsight Systems 抓火焰图
nsys profile -o profile.qdrep python serve.py
nsys-ui profile.qdrep
# 看 NCCL all_reduce 占比，正常应该 < 10%
```

#### 验证

```bash
# 改造后单 token 延迟应该降低 30%~50%
# 监控 PCIe 流量
nvidia-smi nvlink -s   # NVLink 状态
nvidia-smi dmon -s pucmt -c 30
```

> **口诀**：**TP 必须 NVLink**，跨机用 PP。买云服务器要确认 GPU 互联类型，PCIe 卡跑大模型直接 PASS。

---

### 案例 ⑥ vLLM 升级后兼容性破坏

#### 现象

把 vLLM 从 0.5.3 升到 0.6.2 后，业务报"function calling 行为完全变了"。具体：原来给 `tools` 参数能拿到结构化的 `tool_calls`，升级后回归字符串里夹 `{"name": "..."}`。

#### 根因

vLLM 0.6 重写了 chat template 处理逻辑，且对 `tool_choice` 参数的兼容做了变更。具体改动通常没有显眼地写在 changelog 里。

更普遍的兼容性陷阱（vLLM 0.4 → 0.5 → 0.6 我都中过）：

| 版本 | 破坏性变更 |
| --- | --- |
| 0.4 → 0.5 | sampling 默认 seed 行为变 |
| 0.5 → 0.6 | chat template 处理重写、tool_choice 行为变 |
| 0.5 → 0.6 | KV cache dtype 默认 auto，A10 上从 fp16 变 fp8 导致质量降 |
| 0.5.x → 0.5.y | OpenAI logprobs 字段格式 |
| 各版本 | 启动参数名时不时改名 |

#### 修复

**铁律**：

1. **生产环境 pin 死版本**：`vllm==0.6.2`，不要写 `vllm>=0.6`
2. **升级走 staging**：先在影子流量上跑 1 周
3. **回归测试集必须包含**：streaming、tool calling、JSON mode、temperature 0/0.7、long context、各种特殊字符 prompt
4. **监控关键指标的同环比**：升级前后对比 token/s、首字延迟、output token 长度分布

回归测试 example：

```python
import pytest
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8000/v1", api_key="x")

REGRESSION_CASES = load_yaml("regression_cases.yaml")

@pytest.mark.parametrize("case", REGRESSION_CASES)
def test_regression(case):
    resp = client.chat.completions.create(
        model="x",
        messages=case["messages"],
        temperature=0,                    # ★ 必须 0，可复现
        seed=42,
        max_tokens=case["max_tokens"],
    )
    output = resp.choices[0].message.content
    for kw in case["must_contain"]:
        assert kw in output, f"missing keyword: {kw}"
    for kw in case["must_not_contain"]:
        assert kw not in output
```

回滚机制：

```bash
# K8s deployment 同时部署两个版本，灰度切流
kubectl apply -f vllm-0.5.3.yaml
kubectl apply -f vllm-0.6.2.yaml
# Istio / Nginx 5% → 50% → 100%
```

#### 验证

```bash
# 升级前后跑同一组 fixtures
pytest test_regression.py --version 0.5.3 > before.log
pytest test_regression.py --version 0.6.2 > after.log
diff before.log after.log
```

> **口诀**：vLLM 是高速发展的开源项目，**没有保证向后兼容**。pin 版本、灰度升级、回归测试三件套缺一不可。

---

### 案例 ⑦ 模型加载慢：safetensors / sharding

#### 现象

vLLM 服务 K8s pod 重启时**模型加载耗时 4 分钟**。期间健康检查失败，Service 摘了流量，扩容跟不上 burst 流量。

#### 根因

模型 70B FP16 = 140GB。常见加载路径：

```
S3 / OSS → 本地磁盘 → 反序列化 → GPU
   ↑           ↑            ↑       ↑
 网络下载    解压缩      Python 反序列化  H2D 拷贝
```

每一步都可能是瓶颈：
- **网络**：单线程下载 140GB ~ 140s（千兆网）
- **磁盘**：HDD 200MB/s → 700s；SSD 2GB/s → 70s；NVMe 5GB/s → 28s
- **反序列化**：传统 PyTorch `.bin` (pickle) 单线程慢；safetensors 快很多
- **H2D**：PCIe Gen4 x16 = 32GB/s → 4.4s

#### 修复

**优化 1：用 safetensors 格式**

PyTorch `.bin` 是 pickle 格式，加载时单线程反序列化。`safetensors` 是 mmap 友好的二进制格式，加载快 3~5 倍。

```bash
# 转换
pip install safetensors
python -c "
from safetensors.torch import save_file
import torch
state = torch.load('pytorch_model.bin')
save_file(state, 'model.safetensors')
"

# vLLM 默认优先 safetensors，无需额外配置
```

**优化 2：模型分片 + 并行加载**

vLLM 支持把模型切成多个 shard，并行加载：

```bash
# 转换时分片
python -m vllm.model_executor.weight_utils \
  --model llama-2-70b \
  --output-dir sharded \
  --tp-size 4
```

**优化 3：本地预热缓存**

```yaml
# K8s init-container 预拉模型到 emptyDir
spec:
  initContainers:
  - name: model-preloader
    image: amazon/aws-cli
    command:
      - sh
      - -c
      - |
        aws s3 sync s3://models/llama-70b /models --no-progress
    volumeMounts:
    - name: models
      mountPath: /models
  containers:
  - name: vllm
    volumeMounts:
    - name: models
      mountPath: /models
  volumes:
  - name: models
    emptyDir:
      sizeLimit: 200Gi
```

更高级：用 **共享 PVC + ReadOnlyMany**，让多个 pod 共享一份模型文件，避免每个 pod 各自下载。

**优化 4：tensorizer 加速 H2D**

CoreWeave 的 [tensorizer](https://github.com/coreweave/tensorizer) 把模型序列化成"GPU 友好"格式，加载时直接 stream 到 GPU：

```python
from tensorizer import TensorSerializer, TensorDeserializer

# 序列化
serializer = TensorSerializer("model.tensors")
serializer.write_module(model)
serializer.close()

# 反序列化（直接到 GPU）
deserializer = TensorDeserializer("model.tensors", device="cuda")
deserializer.load_into_module(model)
```

实测 70B 加载时间从 4 分钟降到 40 秒。

**优化 5：保持长存活 pod，避免频繁重启**

K8s `terminationGracePeriodSeconds: 300`、配 PDB（PodDisruptionBudget）、关闭 ServerlessGPU 的 scale-to-zero。

#### 验证

```bash
# 测加载耗时
time python -c "
from vllm import LLM
llm = LLM(model='/models/llama-70b')
"

# 期望：< 90s
```

> **口诀**：模型加载慢是大模型生产第一痛点。**safetensors + 本地缓存 + tensorizer + 不重启**四步走。

---

### 案例 ⑧ GPU 显存碎片

#### 现象

vLLM 服务跑了 12 小时后开始零星 OOM，重启后又能跑。`nvidia-smi` 显示显存使用率 70%，理论上还有 24GB 富余，为什么 OOM？

#### 根因

PyTorch 默认用 `caching allocator`，会把释放的显存留在自己的池里复用，不还给 driver。但**池是按 chunk size 分桶的**，长时间运行后各桶大小分布与请求模式不匹配 → 内部碎片。

vLLM 的 `gpu-memory-utilization=0.95` 让分配器把 95% 显存吃下来作为 KV pool，但 PyTorch 还会临时分配一些 workspace（attention kernel、layer norm 中间结果），这些**临时分配也走 PyTorch 的池**。如果 workspace 的请求模式变化（比如长 prompt vs 短 prompt 交替），碎片越来越严重。

观察：

```python
import torch
print(torch.cuda.memory_summary())
# 看 "Allocated memory" vs "Reserved memory"
# 差距大说明碎片严重
```

#### 修复

**方案 A**：降低 `gpu-memory-utilization`，给 PyTorch 留余地

```bash
python -m vllm.entrypoints.openai.api_server \
  --gpu-memory-utilization 0.85   # 留 15% 给 workspace
```

**方案 B**：调 PyTorch 分配器

```bash
# 用 expandable_segments 减少碎片
export PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True

# 或者用更激进的回收
export PYTORCH_CUDA_ALLOC_CONF=garbage_collection_threshold:0.6,max_split_size_mb:128
```

`expandable_segments` 是 PyTorch 2.1+ 的功能，让显存分配按需扩展，碎片显著降低。

**方案 C**：定期主动回收

```python
# 在 vLLM 应用中
import torch, gc

@app.middleware("http")
async def gc_periodically(request, call_next):
    response = await call_next(request)
    if random.random() < 0.001:   # 每千次请求一次
        gc.collect()
        torch.cuda.empty_cache()
    return response
```

注意：`empty_cache()` 不能在请求处理路径上调，会大幅卡顿。

**方案 D**：定期重启（最朴素也最有效）

K8s livenessProbe 配合"运行 12 小时自动重启"：

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8000
  periodSeconds: 30

# 用 OOM 限制 + memory leak detection 自动重启
```

#### 验证

```bash
# 长跑 24h，看 OOM 是否消失
# Prometheus 监控
# - vllm:gpu_cache_usage_perc
# - container_oom_events_total
```

> **口诀**：长跑显存碎片是 GPU 服务通病。`expandable_segments` 是首选；不行就降 `gpu-memory-utilization`；最差也要定期重启。

---

### 案例 ⑨ 流式响应被中间代理断开

#### 现象

vLLM 服务返回流式（SSE）响应，本地直连 OK，但走了 Nginx 后**前端只能看到部分内容就被强行截断**。或者超过 60s 的长生成被代理断开。

#### 根因

中间代理常见的"杀手"：

1. **Nginx `proxy_buffering on`**（默认）—— 缓冲整个响应再发给客户端，流式失效
2. **Nginx `proxy_read_timeout 60s`**（默认）—— 60s 没数据就断
3. **CDN（Cloudflare、AWS ALB）默认 60s timeout**
4. **企业代理 / 防火墙**会 buffer 响应做 DLP 扫描
5. **HTTP/1.1 keepalive 超时**

#### 修复

**Nginx 配置**：

```nginx
upstream vllm {
    server localhost:8000;
}

server {
    listen 443 ssl http2;

    location /v1/ {
        proxy_pass http://vllm;
        proxy_http_version 1.1;

        # ★★★ 关闭缓冲，必须
        proxy_buffering off;
        proxy_request_buffering off;
        proxy_cache off;

        # ★★★ 长 timeout
        proxy_read_timeout 600s;
        proxy_send_timeout 600s;

        # 关闭 chunked 重新打包
        chunked_transfer_encoding on;

        # SSE 必须的 header
        proxy_set_header Connection '';
        proxy_set_header Host $host;

        # 关闭 gzip（流式响应压缩会引入额外缓冲）
        gzip off;
    }
}
```

**云厂商 LB 配置**：

```yaml
# AWS ALB
# Idle Timeout: 600s
# 关闭 HTTP/2 server push

# Cloudflare
# 默认 100s timeout 不可调（普通账号）
# Enterprise 才能调，或改用 cf tunnel

# K8s Ingress (nginx-ingress)
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-buffering: "off"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    # SSE 关键
    nginx.ingress.kubernetes.io/server-snippet: |
      proxy_buffering off;
      chunked_transfer_encoding on;
```

**应用层加 keep-alive**：

vLLM 的流式响应会持续发数据，但如果生成卡住（比如 prefill 长），中间会有几秒静默。在客户端代理这一段加 heartbeat：

```python
# 服务端，每 15s 发一个空 SSE comment 当心跳
async def stream_with_heartbeat(generator):
    last_send = time.time()
    async for chunk in generator:
        yield chunk
        last_send = time.time()
        # 每 15s 没产出就发 keep-alive
        while time.time() - last_send > 15:
            yield ":keep-alive\n\n"   # SSE comment，客户端忽略
            last_send = time.time()
```

**HTTP/2 vs HTTP/1.1**：HTTP/2 多路复用对 SSE 更友好，但有些代理（旧 nginx、ALB classic）对 HTTP/2 SSE 兼容性差，这种情况强制 HTTP/1.1：

```python
# Python httpx
import httpx
client = httpx.Client(http2=False)
```

#### 验证

```bash
# 用 curl 测试流式（必须看到逐 token 输出，而不是一次性全部出现）
curl -N -X POST http://your-domain/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "x",
    "messages": [{"role":"user","content":"写一首长诗"}],
    "stream": true,
    "max_tokens": 1000
  }'

# 预期：每行 "data: {...}" 间隔 < 100ms，持续 30s+ 不断
```

> **口诀**：SSE / 流式 LLM 在中间代理上有 5 个杀手 —— **buffering、timeout、压缩、HTTP/2、心跳**。配 Nginx 时一次性把这 5 项全调好。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 容量层
- [ ] 算清楚 KV 池大小：`(GPU总显存 × utilization - 模型权重) / KV单token大小`
- [ ] `max_num_seqs × max_model_len ≤ KV池容量 × 0.8`
- [ ] 网关层强制限 `max_input_tokens` 防止单条长 prompt 撑爆
- [ ] `--enable-chunked-prefill` 默认开
- [ ] `gpu-memory-utilization` 留 10%~15% 缓冲（0.85~0.9）

### 调度层
- [ ] 永远用 continuous batching（vLLM/TGI）；不要自研 static batching
- [ ] 长生成 / 短生成混合负载用 priority 区分
- [ ] 测试有/无 chunked-prefill 下 P99 差异

### 多卡层
- [ ] 单机多卡 TP 必须 NVLink（不是 PCIe）
- [ ] 跨机用 PP 而不是 TP
- [ ] 验证 NCCL 通信占比 < 10%
- [ ] InfiniBand vs 以太网检查（吞吐型至少 200Gb/s）

### 量化层
- [ ] INT8（W8A8 SmoothQuant）作为默认压缩选择
- [ ] 必须 INT4 时用 AWQ + 业务校准集
- [ ] H100 上用 FP8 而非 FP16
- [ ] 量化后跑 MMLU + GSM8K + HumanEval + 业务回归集

### 模型加载
- [ ] 用 safetensors 不用 .bin
- [ ] 大模型用 tensorizer
- [ ] K8s 用共享 PVC ReadOnlyMany 避免重复下载
- [ ] 配 PDB 避免频繁重启

### 流式 / 网关
- [ ] Nginx `proxy_buffering off`、`proxy_read_timeout 600s`
- [ ] 云 LB / CDN idle timeout 调大
- [ ] 关闭 gzip 压缩
- [ ] SSE 服务端发 15s heartbeat

### 监控
- [ ] vLLM /metrics 接 Prometheus
- [ ] 关键指标：TTFT、TPOT、e2e_latency、gpu_cache_usage、num_requests_running、num_requests_waiting
- [ ] GPU 利用率、显存利用率、PCIe / NVLink 流量
- [ ] 业务侧：QPS、P50/P95/P99、错误率

### 兼容性
- [ ] vLLM / TGI 版本 pin 死，不用 latest
- [ ] 升级走灰度（5% → 50% → 100%）
- [ ] 升级前后跑回归测试集（temperature=0 + seed）
- [ ] 同时部署新旧版本，可秒回滚

### 安全
- [ ] OpenAI Compatible API 必须有鉴权（API Key 或 mTLS）
- [ ] Rate limit 防止单用户打爆（每分钟 token 限额）
- [ ] Input filter 防止 prompt injection
- [ ] 日志脱敏（用户输入可能有 PII）

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### 新一代推理引擎
- **SGLang**：UC Berkeley 团队（vLLM 之后），核心是 **RadixAttention**（前缀树共享 KV）+ structured generation。比 vLLM 在 prefix 重复多的场景快 2~5 倍
- **LMDeploy**：上海 AI Lab，对国产模型（InternLM、Qwen）原生支持最好；W4A16 量化（AWQ 派系）实测最优
- **FastChat**：早期项目（2023），核心是 model worker 抽象，现在主要看其 `controller` 多模型路由设计

### 极致性能
- **TensorRT-LLM**：NVIDIA 闭源，H100 上比 vLLM 快 10~30%。生产门槛是编译复杂度
- **MAX (Modular)**：Chris Lattner 团队，号称跨硬件统一推理栈，2024 后逐渐开源
- **DeepSpeed-Inference**：微软，曾经是吞吐之王，现在地位被 vLLM 抢走，但 ZeRO-Inference 模式仍有特殊场景价值

### KV Cache 共享与卸载
- **LMCache**：跨进程 / 跨节点 KV Cache 共享。同 prefix 的请求复用 KV，对 RAG 场景收益巨大
- **Mooncake (Kimi)**：Disaggregated serving 的代表作，把 prefill 和 decode 拆到不同集群
- **CacheGen**：把 KV Cache 压缩后传输

### Disaggregated Serving（拆分式）
- 把 prefill 和 decode 部署到不同 GPU 集群，因为它们的硬件需求不同：prefill 要算力（H100），decode 要带宽（A100/L40）
- 论文：DistServe (2024)、Splitwise (Microsoft)
- 工业落地：Kimi (Mooncake)、Anyscale

### Mixture of Experts (MoE) 推理
- Mixtral、DeepSeek-V2/V3、Qwen-MoE：只激活部分专家，理论快 4~8 倍
- 推理挑战：**专家路由不均衡** → 某些 GPU 闲、某些过载
- 工程优化：expert parallelism、共享 expert pool、动态 routing

### 长上下文（1M+ token）
- LLaMA 3.1 / GPT-4o / Gemini 1.5 / Claude 3.5 都到 1M 级
- 工程挑战：KV Cache 一条请求就 100GB+；attention O(N²) 不能直接跑
- 方案：**Ring Attention**、**StreamingLLM**（attention sink）、**YaRN**（位置编码外推）

### LLM Gateway（推理服务的"前置"）
- **LiteLLM**：Python 库，统一 100+ 个 LLM Provider 接口
- **Portkey**：商业化 Gateway，有 fallback、retry、cost tracking
- **OneAPI** / **NewAPI**：国产开源，统一接口 + key 管理
- 核心价值：业务代码与具体 Provider 解耦，方便多 Provider 容灾、成本路由

### 推理与训练的边界模糊
- **vLLM + SFT**：用 vLLM 做 PPO 推理（RLHF 训练循环里的 sample 阶段）
- **OpenRLHF / TRL** 用 vLLM 做 generation backend
- 推理引擎的 batching / KV 优化反向赋能训练

---

## 7. 引用

- vLLM 论文: Kwon et al., *Efficient Memory Management for Large Language Model Serving with PagedAttention*, SOSP 2023
- Orca 论文: Yu et al., *Orca: A Distributed Serving System for Transformer-Based Generative Models*, OSDI 2022
- FlashAttention: Dao et al., *FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness*, NeurIPS 2022
- FlashAttention-3: Shah et al., 2024
- Speculative Decoding: Leviathan et al., *Fast Inference from Transformers via Speculative Decoding*, ICML 2023
- AWQ: Lin et al., *AWQ: Activation-aware Weight Quantization for LLM Compression*, MLSys 2024
- GPTQ: Frantar et al., *GPTQ: Accurate Post-Training Quantization for GPT*, ICLR 2023
- SmoothQuant: Xiao et al., 2023
- vLLM 源码: https://github.com/vllm-project/vllm
- TGI 源码: https://github.com/huggingface/text-generation-inference
- llama.cpp 源码: https://github.com/ggerganov/llama.cpp
- TensorRT-LLM: https://github.com/NVIDIA/TensorRT-LLM
- SGLang: https://github.com/sgl-project/sglang
- LMDeploy: https://github.com/InternLM/lmdeploy
- DistServe: Zhong et al., *DistServe: Disaggregating Prefill and Decoding for Goodput-optimized Large Language Model Serving*, OSDI 2024
- Mooncake: Qin et al., *Mooncake: A KVCache-centric Disaggregated Architecture for LLM Serving*, FAST 2025

---

> 配套：[RAG 工程化](/article/ai/rag-engineering) | [Agent 设计模式深入](/article/ai/agent-deep) | [LLM 应用工程化](/article/ai/llm-app)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你觉得这个深度对，告诉我下一篇该写哪个。
