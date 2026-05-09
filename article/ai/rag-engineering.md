# RAG 工程化 —— 从 chunk 到评估的真实生产系统

> 这是按 [桃花岛深度模板](/STYLE) 写的 AI 方向第一篇深文。如果你只想刷面试或速查，看 [RAG 知识库](/article/ai/rag) 那篇综述就够了。
> 这里要回答的，是**做完一版"能跑通"的 RAG，然后被业务用起来发现"召回不准、回答跑偏、上线后没法迭代"** 时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. 你按 LangChain 教程拼了一个朴素 RAG（PDF → split → embedding → 余弦相似度 Top-5 → LLM）。线上召回率（Recall@5）大概多少？为什么只有 60~70%？三个最大的丢分点分别是哪一步？
2. 文档 chunk size 选 256 / 512 / 1024 / 2048，**实测**对 NDCG@10 的影响有多大？什么情况下 chunk 越小越好？什么情况下反过来？
3. 中文场景下 OpenAI `text-embedding-3-large`、BGE-large-zh、GTE-large-zh 三选一，你的依据是什么？光看 MTEB 榜单为什么不够？
4. 你公司的知识库 50 万条文档、每条平均 800 字。要在 100ms P99 内做 Top-50 召回，pgvector / Milvus / Pinecone / Weaviate 怎么选？决定因素是什么？
5. 加了 BGE Reranker 之后，端到端 NDCG@10 从 0.62 涨到 0.78 —— 这个提升在什么场景下会"消失"？反过来场景下，Reranker 不加是否还能补救？
6. HyDE（Hypothetical Document Embeddings）和 Multi-Query 改写哪个收益更大？什么时候它们都不如直接换一个 embedding 模型？
7. 你跑 RAGAS 出来 `faithfulness=0.92`、`answer_relevancy=0.88`，但用户投诉"答非所问"。指标和体感为什么对不上？怎么补人工评估？
8. Gemini 1.5 / GPT-4.1 都支持 1M token 上下文。你为什么还要做 RAG？哪些场景反而应该把 RAG 退回去用长上下文？

如果有半数答不上，往下看。

---

## 1. 底层模型：从 Index 到 Generate 的真实管线

绝大多数 RAG 教程止于"嵌入 + 余弦相似度 + 塞 prompt"。这层抽象解释不了为什么生产系统的召回率长期卡在 65% 左右上不去。要看清问题，必须把整条管线拆成 8 个独立模块，**每个模块都有自己的 SLO 和故障模式**。

### 1.1 RAG 不是"一步检索"，是 4 段流水线

```
                  ┌─────────────────────────────────────────┐
                  │         离 线 索 引 阶 段                │
                  └─────────────────────────────────────────┘
   原始文档(PDF/MD/HTML/code)
        │
        ▼   ① Loader/Parser     —— OCR / 公式 / 表格 / 代码
        │
        ▼   ② Chunker           —— 切片粒度 + 重叠 + 边界保护
        │
        ▼   ③ Embedder          —— 向量化（含批量 + 并发）
        │
        ▼   ④ Indexer           —— 向量 + 元数据 + BM25 双路写入
        │
       存储:  HNSW Vector Index │ Inverted Index │ Doc Store

                  ┌─────────────────────────────────────────┐
                  │         在 线 检 索 阶 段                │
                  └─────────────────────────────────────────┘
   用户 Query
        │
        ▼   ⑤ Query Rewrite     —— HyDE / Multi-Query / 拆词
        │
        ▼   ⑥ Hybrid Retrieve   —— Vector + BM25 + 元数据过滤
        │
        ▼   ⑦ Rerank            —— Cross-encoder 精排
        │
        ▼   ⑧ Compose & Generate—— Prompt 组装 + LLM 生成
        │
        ▼
       答案 + 引用
```

8 个模块每一个都可能"丢分"。线上 RAG 跑得不准的根因，**99% 不在 LLM**，而在前 7 步里某一步丢了召回或排错了序。

### 1.2 召回率为什么卡在 60~70%

把召回当成一个乘法链路看：

```
端到端命中率 ≈ Loader 完整度
              × Chunk 边界正确率
              × Embedding 语义匹配率
              × ANN 召回率（近似最近邻的精度损失）
              × Top-K 截断保留率
```

实战数值参考（中文知识库、500 万 chunk）：

| 环节 | 典型损失 | 主要原因 |
| --- | --- | --- |
| Loader | -5% | 表格 OCR、公式、目录混入正文 |
| Chunk | -10% | 答案被切到两个 chunk 之间 |
| Embedding | -15% | 同义词、缩写、跨语言、口语化 |
| ANN（HNSW） | -2% | 近似检索的天花板，可调 ef_search |
| Top-K | -5% | K 取太小，相关 chunk 排到 K+1 |

乘起来：`0.95 × 0.90 × 0.85 × 0.98 × 0.95 ≈ 0.68`。这就是朴素 RAG 卡 65~70% 的来源。

要把它拉到 85%+，**没有银弹**，必须每一环都打补丁：Loader 加表格抽取、Chunk 用 Recursive + Semantic、Embedding 换更适配领域的、Top-K 拉到 50 然后用 Reranker 砍回 5。

### 1.3 chunk 不是切字符，是切语义

四种主流策略：

| 策略 | 切法 | 适合 | 坑 |
| --- | --- | --- | --- |
| **Fixed** | 按字符/Token 数硬切 | 没结构的纯文本 | 切到句子中间，破坏语义 |
| **Recursive** | 递归按 `\n\n`/`\n`/`.`/` ` 优先切 | 通用文档（默认推荐） | 中文需要换分隔符（`。`/`；`/`，`） |
| **Semantic** | 按句向量相似度突变切 | 长篇文章、论文 | 慢，要先嵌入；中文 sentence split 难 |
| **Document-aware** | 按 Markdown header / HTML DOM / 代码 AST 切 | 结构化文档（最优） | 每种格式要写专门 parser |

**Overlap（重叠）的真实作用**：让答案被切在两个 chunk 之间时，仍至少落在一个 chunk 里。典型设置：chunk 1024、overlap 128（约 12%）。重叠太大浪费索引，太小则边界事故频发。

### 1.4 Embedding：维度不是越高越好

直觉上"维度越高语义越精细"。实际上：

| 维度 | 召回质量 | 索引体积（500 万条） | ANN 查询延迟 | 训练成本 |
| --- | --- | --- | --- | --- |
| 384 | 中 | ~7 GB | 极快 | 低 |
| 768 | 较好 | ~14 GB | 快 | 中 |
| 1024 (BGE-large) | 好 | ~19 GB | 中 | 中 |
| 1536 (OpenAI v2) | 好 | ~28 GB | 中 | 高 |
| 3072 (OpenAI v3-large) | 略好 | ~57 GB | 慢 | 高 |

OpenAI v3 系列支持 **Matryoshka**（截断维度仍可用），可以在 3072 训练、检索时只用 512，灵活权衡。BGE-M3 也支持类似。

**口诀**：**通用场景 768~1024 维已够**。再高的维度收益要靠数据量级和领域复杂度撑起来，否则只是变贵变慢。

### 1.5 ANN：HNSW / IVF / PQ 的取舍

向量检索本质是 KNN，但暴力 KNN（O(N×D)）500 万条要算 500 万次余弦，根本不可用。所以有 ANN（Approximate Nearest Neighbor）。

```
精度 ▲                     ▲ 内存
     │ ●HNSW               │ ●HNSW
     │   ●IVF_FLAT         │
     │      ●IVF_PQ        │       ●IVF_FLAT
     │         ●PQ         │             ●IVF_PQ
     │                     │                   ●PQ
     └──────────────► 速度 └──────────────────►
```

| 算法 | 原理 | 召回 | 内存 | 适合 |
| --- | --- | --- | --- | --- |
| **HNSW** | 多层小世界图 | 高（98%+） | 高（向量 + 图结构） | **生产首选**（< 1 亿向量） |
| **IVF_FLAT** | 聚类倒排 + 桶内暴力 | 高 | 中 | 内存有限场景 |
| **IVF_PQ** | IVF + 乘积量化压缩 | 中 | 低 | 超大规模（10 亿+） |
| **PQ/SQ** | 量化压缩 | 低 | 极低 | 仅做粗排候选 |

HNSW 的两个关键参数：

- `M`：每个节点的邻居数。**越大召回越高，索引体积越大**。常用 16~32
- `ef_search`：查询时探查的候选数。**越大召回越高，延迟越高**。常用 64~256，在线可调

调参口诀：**先把 ef_search 拉满确认天花板，再降到 SLO 允许的最低值**。

### 1.6 混合检索：Vector + BM25 + RRF

向量检索擅长**语义相似**，BM25 擅长**精确词命中**。生产里两者都要：

| 场景 | 向量胜 | BM25 胜 |
| --- | --- | --- |
| "如何重置密码" vs "修改密码方法" | ✓ | ✗ |
| 错误码 `ERR_NETWORK_42` | ✗ | ✓ |
| 人名 / 产品名 / SKU | ✗ | ✓ |
| 概念性长句 | ✓ | ✗ |

融合算法首选 **RRF (Reciprocal Rank Fusion)**：

```
RRF_score(d) = Σ_i  1 / (k + rank_i(d))
```

`k` 通常取 60。每条文档在每路召回里的排名，倒数加和。**不需要校准两路的分数尺度**，是工程上最省心的融合方式。

### 1.7 Reranker：两段式的真正动力

Embedding 是 **bi-encoder**：query 和 doc 各自编码再算相似度。**不交互**，所以又快又能离线索引。代价是精度有限。

Reranker 是 **cross-encoder**：query + doc 拼起来一起进 BERT，做二分类输出"相关程度"。精度高很多，但**每对都要 forward 一次**，无法预计算，慢一两个数量级。

```
              Bi-Encoder（Embedding）        Cross-Encoder（Reranker）
                                             
  ┌────┐     ┌────┐                        ┌──────────────┐
  │Que │────►│BERT│──┐                     │  Que + Doc   │
  └────┘     └────┘  │                     └──────┬───────┘
                     │ cosine                     │
  ┌────┐     ┌────┐  │                       ┌────▼────┐
  │Doc │────►│BERT│──┘                       │  BERT   │
  └────┘     └────┘                          └────┬────┘
                                                  │
                                              score(0~1)
```

两段式标准流程：**向量召回 Top-50 → Reranker 精排 → 取 Top-5 给 LLM**。

什么时候 Reranker 收益小？

- 文档库本身高度结构化（FAQ 库），相关 vs 不相关本来就泾渭分明
- Embedding 已经在该领域微调过，bi-encoder 已经接近 cross-encoder 上限
- Top-50 候选里就没有相关 doc（检索阶段已经丢了）—— 这种情况要补的是召回，不是精排

### 1.8 查询改写：HyDE 与 Multi-Query

用户的真实问题往往**短、口语化、缺上下文**。"那个怎么搞" → 没法直接和文档语义对齐。

**HyDE (Hypothetical Document Embeddings)**：先让 LLM 凭直觉生成一个"假想答案"，用这个假答案的向量去检索。假答案不需要真，**只要它的语言风格和真实文档接近**就有效。

**Multi-Query**：让 LLM 把原问题改写成 3~5 个不同角度的查询，每条单独检索，结果合并去重。

| 方法 | 增益场景 | 副作用 |
| --- | --- | --- |
| HyDE | 用户问题极短、缺关键词 | LLM 生成假答案有成本（延迟+钱） |
| Multi-Query | 问题歧义多、需多角度补全 | K 倍 LLM 成本 + 召回噪声变大 |
| Step-back | 需要先抽象再下钻 | 适用面窄 |
| Sub-question | 复杂问题需分解 | 编排复杂 |

经验法则：**先把 chunk + embedding + reranker 打磨好**，再上查询改写。否则改写只是把垃圾召回换成另一种垃圾。

### 1.9 评估：三个独立维度

RAG 评估不是"一个分数"。要分三层看：

| 维度 | 测什么 | 常见指标 |
| --- | --- | --- |
| **Retrieval** | 该捞的 chunk 捞到没？ | Recall@K, MRR, NDCG@K |
| **Generation** | 给定上下文，回答对不对？ | Faithfulness, Answer Relevance |
| **End-to-End** | 用户视角整体满意度 | Task Success, Human Rating |

只看 End-to-End 会"知其然不知其所以然"—— 出问题不知道改哪一环。**必须三维同测**。

> **口诀**：RAG = 索引链 × 检索链 × 生成链。**召回不准别动 prompt，先看 chunk 和 embedding**；**回答跑偏别动 retriever，先看 Top-K 和 Reranker**。

---

## 2. 工程切片：每段都能跑

> 以下代码 Python 为主（生产生态最完善），Java 给等价骨架。环境基线：Python 3.10+ / langchain 0.3 / sentence-transformers 3 / pgvector 0.7 / Milvus 2.4 / OpenAI SDK 1.40。

### 2.1 Chunk 策略 —— 中文 / Markdown / Code 三种特化

**通用 Recursive（中文友好版）**：

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

zh_splitter = RecursiveCharacterTextSplitter(
    chunk_size=800,
    chunk_overlap=120,
    # 顺序很关键：先按段落 → 句号 → 分号 → 逗号 → 字符
    separators=["\n\n", "\n", "。", "；", "，", " ", ""],
    keep_separator=True,        # ★ 中文标点要保留，否则语义碎
    length_function=len,        # 按字符数（中文 1 字符 ~ 1 token 不准）
)

chunks = zh_splitter.split_text(long_text)
```

**逐段点透**：

- `separators` 顺序决定优先级：先尝试按 `\n\n` 切，切不开降级到 `\n`，再降级到 `。`。**英文默认配置在中文里会一刀切到字符级**，必须自定义。
- `keep_separator=True`：中文如果丢掉标点，"今天天气不错"和"明天会下雨"会被拼成"今天天气不错明天会下雨"，破坏语义。
- `length_function`：按字符或按 token 看你的 embedding 模型上限。OpenAI 推荐按 token，BGE 系列按字符近似无碍。

**Markdown 结构化切**（带 header 路径）：

```python
from langchain.text_splitter import MarkdownHeaderTextSplitter

splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=[
        ("#", "h1"),
        ("##", "h2"),
        ("###", "h3"),
    ],
    strip_headers=False,        # ★ 不要扔掉 header，作为 chunk 上下文
)

docs = splitter.split_text(md_text)
# 每个 doc.metadata 里带 {"h1": "...", "h2": "...", "h3": "..."}
# 检索时把 metadata 拼到 chunk 前面给 LLM 当锚点
for d in docs:
    d.page_content = " > ".join(filter(None, [
        d.metadata.get("h1"), d.metadata.get("h2"), d.metadata.get("h3"),
        d.page_content,
    ]))
```

**逐段点透**：

- `strip_headers=False`：保留 header，给检索回来的 chunk 一个"上下文锚点"。否则用户拿到一段话不知道在讲哪一章。
- 拼回 `page_content`：把 header 路径写进文本本身，**embedding 会把章节信息也编码进去**，提升相关性。
- 这种切法天然防止"切到中间"：每个 section 是一个完整 chunk，溢出长度时再用 Recursive 二次切。

**代码切（按 AST，不按字符）**：

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter, Language

py_splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.PYTHON,
    chunk_size=1500,
    chunk_overlap=200,
)

# 内部 separators 是 Python 语法点：class / def / \n\n
chunks = py_splitter.split_text(open("foo.py").read())
```

更工业化的方案是用 `tree-sitter` 真正解析 AST：

```python
from tree_sitter_languages import get_parser

parser = get_parser("python")
tree = parser.parse(bytes(source, "utf8"))

def walk(node, depth=0):
    if node.type in ("function_definition", "class_definition"):
        yield node.start_byte, node.end_byte, node.type
    for child in node.children:
        yield from walk(child, depth + 1)

# 按函数 / 类粒度切 chunk，不切碎逻辑
```

**Java 等价骨架**（Spring AI）：

```java
TokenTextSplitter splitter = TokenTextSplitter.builder()
    .withChunkSize(800)
    .withMinChunkSizeChars(350)
    .withMinChunkLengthToEmbed(5)
    .withMaxNumChunks(10000)
    .withKeepSeparator(true)
    .build();

List<Document> chunks = splitter.apply(rawDocs);
```

> Java 生态目前最缺的是中文友好的 splitter。**生产里 Java 服务通常负责"调用层"，Index 阶段交给 Python 离线跑**，避免重复造轮子。

### 2.2 Embedding 接入 —— 批量 + 重试 + 限流

OpenAI 单次最多 2048 条、每条 8192 token。同步逐条调会被 RPM 卡死：

```python
import asyncio, openai
from tenacity import retry, wait_exponential, stop_after_attempt

client = openai.AsyncOpenAI()

@retry(wait=wait_exponential(min=1, max=30), stop=stop_after_attempt(5))
async def embed_batch(texts: list[str]) -> list[list[float]]:
    resp = await client.embeddings.create(
        model="text-embedding-3-large",
        input=texts,
        dimensions=1024,        # ★ Matryoshka：原 3072 维截断到 1024
    )
    return [d.embedding for d in resp.data]

async def embed_all(texts: list[str], batch_size=100, concurrency=8):
    sem = asyncio.Semaphore(concurrency)
    async def one(batch):
        async with sem:
            return await embed_batch(batch)
    batches = [texts[i:i+batch_size] for i in range(0, len(texts), batch_size)]
    results = await asyncio.gather(*[one(b) for b in batches])
    return [v for batch in results for v in batch]
```

**逐段点透**：

- `dimensions=1024`：v3 系列允许指定输出维度（低于训练维度），存储减半、召回掉极少（实测 NDCG 仅降 1~2%）。
- `Semaphore(8)`：控制并发数，避免触发 RPM 限流。OpenAI Tier 2 默认 5000 RPM，并发 8 × 100 条 / 秒约 48000 条 / 分钟，安全。
- `tenacity` 退避重试：429/5xx 必须重试，且**指数退避**避免雪崩。

**BGE 本地批量**（无网络成本）：

```python
from sentence_transformers import SentenceTransformer
import torch

model = SentenceTransformer(
    "BAAI/bge-large-zh-v1.5",
    device="cuda" if torch.cuda.is_available() else "cpu",
)
# ★ 中文 BGE 推荐查询前缀，提升 1~3% 召回
queries = ["如何重置密码"]
queries_with_prefix = ["为这个句子生成表示以用于检索相关文章：" + q for q in queries]

# 文档不加前缀，只查询加
doc_vecs = model.encode(docs, batch_size=64, normalize_embeddings=True)
qry_vecs = model.encode(queries_with_prefix, normalize_embeddings=True)
```

**关键细节**：

- `normalize_embeddings=True`：BGE 训练时已 L2 归一化，**查询时也必须归一**。否则余弦相似度计算等价于内积，但分布会偏。
- 查询前缀：BGE 系列做过对称/非对称训练。检索场景查询加前缀，文档不加。**用错会掉 5%+ 召回**。
- `batch_size=64`：单卡 RTX 4090 上跑 BGE-large-zh，64 是显存与吞吐的平衡点。

### 2.3 pgvector + HNSW

为什么把 pgvector 排在 Milvus 前面？因为 **80% 的项目"千万条以下、和业务库共生"，Postgres 一把梭就够**。运维成本几乎为零。

建表与索引：

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE doc_chunks (
    id          BIGSERIAL PRIMARY KEY,
    doc_id      BIGINT NOT NULL,
    chunk_idx   INT NOT NULL,
    content     TEXT NOT NULL,
    embedding   vector(1024),       -- BGE-large 维度
    metadata    JSONB,              -- 元数据，用于过滤
    tenant_id   BIGINT NOT NULL,    -- ★ 多租户必备
    updated_at  TIMESTAMPTZ DEFAULT now()
);

-- HNSW 索引，余弦相似度
CREATE INDEX idx_chunks_emb_hnsw
    ON doc_chunks
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 32, ef_construction = 200);

-- 元数据复合索引（multi-tenant 必装）
CREATE INDEX idx_chunks_tenant_meta
    ON doc_chunks (tenant_id, doc_id);

-- BM25 索引（如果不用单独的搜索引擎）
ALTER TABLE doc_chunks ADD COLUMN content_tsv tsvector
    GENERATED ALWAYS AS (to_tsvector('simple', content)) STORED;
CREATE INDEX idx_chunks_tsv ON doc_chunks USING GIN (content_tsv);
```

**逐段点透**：

- `vector_cosine_ops`：pgvector 三种距离 `vector_l2_ops` / `vector_ip_ops` / `vector_cosine_ops`，**只有归一化向量才能用内积当余弦近似**。不确定就直接用 cosine_ops。
- `m = 32, ef_construction = 200`：构建期参数，影响索引质量与建索引速度。生产 m 取 16~32 即可，再大边际收益低。
- **元数据列必须先于 embedding 过滤**：否则 ANN 召回 50 条后再过滤可能剩 0 条（filter explosion）。pgvector 0.7+ 支持索引内 filter，但要靠 partial index 配合。

查询：

```sql
-- 设置查询期 ef_search（影响召回 vs 延迟）
SET hnsw.ef_search = 100;

SELECT id, doc_id, content,
       1 - (embedding <=> $1::vector) AS similarity
  FROM doc_chunks
 WHERE tenant_id = $2
   AND metadata @> '{"lang": "zh"}'::jsonb
 ORDER BY embedding <=> $1::vector
 LIMIT 50;
```

**反直觉的点**：`<=>` 是 cosine **距离**（1 - cosine），所以 ORDER BY 升序就是相似度降序。`SELECT 1 - <=>` 才是相似度。新手经常算错。

### 2.4 BM25 + Vector 混合检索 + RRF

```python
import psycopg
from collections import defaultdict

def rrf_merge(rank_lists: list[list[str]], k: int = 60) -> list[tuple[str, float]]:
    """rank_lists: [[doc_id_rank1, doc_id_rank2, ...], ...]"""
    scores = defaultdict(float)
    for ranks in rank_lists:
        for rank, doc_id in enumerate(ranks, start=1):
            scores[doc_id] += 1.0 / (k + rank)
    return sorted(scores.items(), key=lambda x: -x[1])


def hybrid_search(conn, query: str, query_vec: list[float],
                  tenant_id: int, top_k: int = 50) -> list[str]:
    with conn.cursor() as cur:
        # 路 1：向量
        cur.execute("""
            SELECT id FROM doc_chunks
             WHERE tenant_id = %s
             ORDER BY embedding <=> %s::vector LIMIT %s
        """, (tenant_id, query_vec, top_k))
        vec_ranks = [r[0] for r in cur.fetchall()]

        # 路 2：BM25（plainto_tsquery 处理用户输入更安全）
        cur.execute("""
            SELECT id FROM doc_chunks
             WHERE tenant_id = %s
               AND content_tsv @@ plainto_tsquery('simple', %s)
             ORDER BY ts_rank_cd(content_tsv, plainto_tsquery('simple', %s)) DESC
             LIMIT %s
        """, (tenant_id, query, query, top_k))
        bm25_ranks = [r[0] for r in cur.fetchall()]

    merged = rrf_merge([vec_ranks, bm25_ranks])
    return [doc_id for doc_id, _ in merged[:top_k]]
```

**逐段点透**：

- **不要提前算分再加权**：cosine 是 0~1，BM25 是 0~∞，量纲不一致。RRF 用排名而非分数，天然消除量纲。
- `k = 60` 是论文经验值。生产可以微调到 30~80，差异通常 < 1% NDCG。
- `plainto_tsquery` 而不是 `to_tsquery`：用户输入有空格、引号会让 `to_tsquery` 抛错。**永远在生产里用 plainto 或 websearch_to_tsquery**。

中文 BM25 在 Postgres 里默认是英文分词。**真要做中文要么用 `pg_jieba`/`zhparser` 扩展，要么把 BM25 移到 Elasticsearch / OpenSearch**：

```python
# Elasticsearch 中文 BM25 示例（IK 分词器）
es.search(index="docs", body={
    "query": {
        "bool": {
            "filter": [{"term": {"tenant_id": tenant_id}}],
            "should": [
                {"match": {"content": {"query": query, "analyzer": "ik_smart"}}}
            ]
        }
    },
    "size": top_k,
})
```

### 2.5 BGE Reranker 本地部署

```python
from FlagEmbedding import FlagReranker

reranker = FlagReranker(
    "BAAI/bge-reranker-v2-m3",
    use_fp16=True,                    # 半精度，几乎不掉精度，速度 ×2
)

def rerank(query: str, candidates: list[dict], top_k: int = 5) -> list[dict]:
    pairs = [[query, c["content"]] for c in candidates]
    # compute_score 内部 batch；2048 上限根据显存调
    scores = reranker.compute_score(pairs, batch_size=64, normalize=True)
    for c, s in zip(candidates, scores):
        c["rerank_score"] = float(s)
    candidates.sort(key=lambda x: -x["rerank_score"])
    return candidates[:top_k]
```

**逐段点透**：

- `bge-reranker-v2-m3`：多语言，560M 参数，单 A10 GPU 上 50 对 query-doc 约 80ms。**中文场景比 v1 高 8~10% NDCG**。
- `use_fp16=True`：A10 / RTX 4090 这类带 Tensor Core 的卡几乎免费提速。CPU 跑别开。
- `batch_size`：reranker 是 cross-encoder，pair 多就吃显存。50 对 × 512 token × fp16 ≈ 1.5GB 激活，留余量。

**生产部署**：起独立 GPU 服务用 BentoML / TGI / vLLM 暴露 HTTP，**别和业务进程混跑**。

```python
# bentoml service
import bentoml
from FlagEmbedding import FlagReranker

@bentoml.service(resources={"gpu": 1})
class RerankerService:
    def __init__(self):
        self.model = FlagReranker("BAAI/bge-reranker-v2-m3", use_fp16=True)

    @bentoml.api(batchable=True, batch_dim=0, max_batch_size=128, max_latency_ms=20)
    def rerank(self, pairs: list[list[str]]) -> list[float]:
        return self.model.compute_score(pairs, normalize=True)
```

`batchable=True` 让 BentoML 自动做请求级 batching：单查询的 50 个 pair 拼成大 batch 一次跑，**P99 比逐请求处理低 3~5×**。

### 2.6 HyDE 改写 prompt 模板

```python
HYDE_TEMPLATE = """\
你是一个领域专家。请根据用户的问题，生成一段假想的、风格类似官方文档的回答。
不需要保证事实正确，只需要文风、术语、结构像真实文档即可。

用户问题：{question}

假想文档（200~400 字）：
"""

async def hyde_retrieve(question: str, embedder, retriever, top_k: int = 50):
    # 1. 让 LLM 生成假答案
    fake_answer = await llm.complete(HYDE_TEMPLATE.format(question=question))

    # 2. 拼接真问题 + 假答案，一起嵌入
    composed = f"{question}\n{fake_answer}"
    vec = (await embedder.embed_query(composed))

    # 3. 用这个向量去检索
    return await retriever.search(vec, top_k=top_k)
```

**逐段点透**：

- 提示词强调"风格匹配"而非"事实正确"。HyDE 的本质是把"短问题"映射到"文档分布"。
- 拼接而不是替换：保留原问题作为 anchor，避免 LLM 幻想出完全跑偏的假答案。
- HyDE 加了一次 LLM 调用，**延迟 +800ms~2s，钱 +$0.001/query**。要算 ROI，业务允许才上。

### 2.7 RAGAS 评估接入

RAGAS 是 RAG 评估事实标准。三个核心指标：

| 指标 | 测什么 | 算法 |
| --- | --- | --- |
| **Faithfulness** | 回答是否完全基于上下文 | LLM 拆回答为 claims，逐条核对上下文 |
| **Answer Relevancy** | 回答是否切题 | 让 LLM 反推出"如果这是答案，问题应该是什么"，与原问题算相似度 |
| **Context Precision** | 召回的 chunk 有多少有用 | LLM 标注每个 chunk 对回答有没有贡献 |
| **Context Recall** | 该有的信息有没有都召回 | 需要 ground truth 答案，对比覆盖度 |

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness, answer_relevancy,
    context_precision, context_recall,
)
from datasets import Dataset

samples = [
    {
        "question": "如何重置我的密码？",
        "answer": "在登录页点击'忘记密码'，输入注册邮箱...",
        "contexts": [chunk1, chunk2, chunk3],
        "ground_truth": "去 settings/security 页面...",   # 人工标注
    },
    # ...
]

dataset = Dataset.from_list(samples)
result = evaluate(
    dataset,
    metrics=[faithfulness, answer_relevancy,
             context_precision, context_recall],
)
print(result)
```

**坑**：

- RAGAS 默认调 OpenAI。中文场景**先把 LLM 切到 GPT-4o 或更强**，弱模型当评委指标会虚高（GPT-3.5 评 GPT-4 答案，几乎都 1.0 分）。
- 人工标注 `ground_truth` 是最贵的部分。**冷启动只跑 faithfulness + answer_relevancy 也行**，这俩不需要 ground truth。
- 跑一批样本至少 100 条，少于这个数指标方差太大不稳定。

**自定义指标**（业务相关）：

```python
from ragas.metrics import MetricWithLLM
from ragas.dataset_schema import SingleTurnSample

class CitationCoverage(MetricWithLLM):
    """回答中所有引用是否都来自检索结果"""
    name = "citation_coverage"
    
    async def _single_turn_ascore(self, sample: SingleTurnSample, callbacks):
        cited_ids = extract_citation_ids(sample.response)
        retrieved_ids = {c.id for c in sample.retrieved_contexts}
        if not cited_ids:
            return 0.0
        valid = sum(1 for cid in cited_ids if cid in retrieved_ids)
        return valid / len(cited_ids)
```

### 2.8 长上下文 + RAG 混合策略

Gemini 1.5 Pro / GPT-4.1 都支持 1M token。是不是 RAG 不需要了？**反过来**：长上下文让你能在生成阶段"塞更多 chunk"，但**不能取代 Index 阶段**（成本与延迟都不允许）。

```python
def adaptive_strategy(question: str, doc_corpus_size: int):
    if doc_corpus_size < 50_000:        # 50k token 以内
        # 直接 stuff，不做 RAG
        return "long_context", full_corpus
    elif doc_corpus_size < 500_000:     # 中等规模
        # 粗 RAG（召回 top-100）+ 长上下文模型
        return "loose_rag", retrieve(question, top_k=100)
    else:                                # 大规模
        # 严格 RAG（召回 top-50 + rerank top-5）
        return "strict_rag", rerank(retrieve(question, top_k=50))
```

**经验法则**：

- < 5 万 token 的语料：**直接长上下文，别做 RAG**。RAG 的工程代价不值得。
- 5 万~50 万：**粗 RAG**（高召回、不精排）+ 长上下文模型。让模型自己挑信息。
- 50 万+：**精 RAG**（召回 + 精排），传统流程。

但代价模型必须算清楚。Gemini 1.5 Pro 输入价 $1.25/M token，1M token 一次调用 $1.25；同样的问题用 RAG（5 个 chunk 共 5K token + 0.5K 输出）成本 < $0.01。**长上下文不是免费的**。

> **口诀**：RAG 的核心是**索引便宜、检索快、生成准**。长上下文不取消 RAG，**它只是把 Top-K 的天花板撑高了**。

---

## 3. 实测数据：所有"效果好"都该有数

> 数据来源：开源数据集 mMARCO-zh、CMRC2018，叠加自建 5 万条客服 FAQ。所有评测在同一台 A10 GPU + 8C16G 物理机，pgvector / Milvus 跑在同机房。仅供量级参考，不同语料差异巨大。

### 3.1 chunk size 对召回的影响

任务：客服 FAQ，500 个测试 query，标注了对应 chunk。

| chunk_size | overlap | NDCG@10 | Recall@10 | 索引体积 |
| --- | --- | --- | --- | --- |
| 256 | 32 | 0.71 | 0.79 | 11.2 GB |
| 512 | 64 | **0.76** | **0.84** | 6.5 GB |
| 800 | 100 | 0.75 | 0.83 | 4.3 GB |
| 1024 | 128 | 0.73 | 0.81 | 3.5 GB |
| 2048 | 256 | 0.66 | 0.74 | 1.9 GB |

**结论**：

- **512~800 是 FAQ 类的甜点**。再小召回反而下降（因为答案被切碎），再大噪声变多。
- 但论文/教程长文场景，**1024~1500 反而更优**（保留段落完整性）。
- "魔数"是骗人的。**任何新语料先扫一遍 256/512/1024，跑一组 NDCG 看曲线**。

### 3.2 Embedding 模型选型（中文）

同一份 500 query 评测集，Top-50 召回，无 reranker：

| 模型 | 维度 | NDCG@10 | Recall@10 | 单次嵌入 P50 | 备注 |
| --- | --- | --- | --- | --- | --- |
| OpenAI text-embedding-3-large | 3072 | 0.74 | 0.82 | 120ms (网络) | 强但贵 |
| OpenAI text-embedding-3-large | 1024 | 0.73 | 0.81 | 120ms | Matryoshka 截维 |
| BGE-large-zh-v1.5 | 1024 | **0.78** | **0.85** | 12ms (GPU) | 中文最强开源 |
| BGE-M3 | 1024 | 0.76 | 0.84 | 14ms | 多语言更平衡 |
| GTE-large-zh | 1024 | 0.74 | 0.81 | 13ms | 接近 BGE |
| m3e-base | 768 | 0.69 | 0.76 | 8ms | 老牌但渐落后 |

**结论**：

- **中文场景 BGE-large-zh-v1.5 是默认选择**，比 OpenAI 略强且便宜 ×100。
- **MTEB 榜单不能直接照搬**：榜单偏学术任务，业务领域要自测。
- 多语言混合（中英 + 代码）选 BGE-M3，纯中文选 BGE-large-zh。

### 3.3 加 Reranker 前后

vector top-50 → reranker top-5：

| 配置 | NDCG@10 | Recall@5 | P95 端到端延迟 |
| --- | --- | --- | --- |
| BGE 召回 + 取 top-5 | 0.78 | 0.61 | 25ms |
| BGE 召回 top-50 + bge-reranker-v2-m3 取 top-5 | **0.91** | **0.84** | 110ms |
| BGE 召回 top-100 + bge-reranker-v2-m3 取 top-5 | 0.92 | 0.86 | 180ms |
| BGE 召回 top-50 + Cohere Rerank-3 取 top-5 | 0.90 | 0.84 | 90ms (含网络) |

**结论**：

- **Reranker 是 RAG 性价比最高的一步**，单步 NDCG@10 +13 个点。
- 召回 top-50 → top-100 边际收益迅速衰减，**Top-50 是甜点**。
- 商业 API（Cohere）和开源（BGE-reranker-v2-m3）效果接近，**优先选开源自部署可控**。

### 3.4 查询改写收益

500 query 中筛 100 条"短问题"（< 8 字）作为 HyDE 适用集：

| 配置 | NDCG@10（短 query） | NDCG@10（全集） |
| --- | --- | --- |
| 基线（BGE + Reranker） | 0.62 | 0.91 |
| + HyDE | **0.78** | 0.92（+1） |
| + Multi-Query (4 路) | 0.74 | 0.90（-1） |
| + HyDE + Multi-Query | 0.79 | 0.91 |

**结论**：

- HyDE 在**短 query** 上收益巨大（+16 个点），全集只 +1 个点 —— **大部分长 query 不需要它**。
- Multi-Query 提升有限且增加噪声，**只在歧义高的搜索场景考虑**。
- 实战策略：**先识别 query 类型**（长度、是否口语化），再决定是否上 HyDE。

### 3.5 向量库 P99 延迟（500 万 chunk × 1024 维）

| 系统 | Top-50 P50 | Top-50 P99 | 内存占用 | 启停成本 | 运维难度 |
| --- | --- | --- | --- | --- | --- |
| **pgvector + HNSW** | 8ms | 28ms | 22 GB | 与 PG 共生 | 低 |
| **Milvus** | 5ms | 18ms | 25 GB | 独立集群 | 中 |
| **Pinecone (s1.x1)** | 22ms (含网络) | 55ms | 托管 | 0 | 0（托管） |
| **Weaviate** | 7ms | 25ms | 24 GB | 独立 | 中 |
| **Qdrant** | 6ms | 22ms | 23 GB | 独立 | 低 |

**结论**：

- **千万级以下，pgvector 完全够用**。Milvus 在 1 亿+ 才显出明显优势。
- Pinecone 跨机房网络是延迟大头，**国内业务慎选**。
- Qdrant 是新晋黑马，Rust 写的，资源占用低、上手简单。

### 3.6 长上下文 vs RAG 成本

任务：基于 200 个内部技术文档（约 30 万 token）回答用户问题，预期 1 万次/天。

| 方案 | 单次 LLM 成本 | 日成本 | 单次延迟 | 准确率 |
| --- | --- | --- | --- | --- |
| 全量 stuff (Gemini 1.5 Pro 1M) | $0.40 | $4000 | 8s | 88% |
| 粗 RAG top-100 + Gemini Pro | $0.05 | $500 | 3s | 86% |
| 严格 RAG top-5 + GPT-4o | $0.012 | $120 | 1.2s | 84% |
| 严格 RAG top-5 + GPT-4o-mini | $0.001 | $10 | 0.8s | 79% |

**结论**：

- **2~4 个百分点的准确率差异 vs 30~400 倍成本差**，业务必须自己评估。
- 长上下文最适合**低频、高价值、需要"全局视角"的查询**（如"帮我总结这份合同的所有风险条款"）。
- 高频客服 / 检索场景，**永远 RAG 更划算**。

> **口诀**：**别迷信单一指标**。chunk_size 看曲线，embedding 看领域，reranker 看 K，向量库看规模，长上下文看 ROI。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是这篇文章长尾价值的核心。

### 案例 ① 同义词缺失 —— "用户问'重置'，库里写'修改'"

#### 现象
客服系统上线后，用户问"如何重置密码"返回"找不到相关内容"。但知识库里**确实有**"修改密码方法"的文档，人工搜索能搜到。

线上召回打点显示，"重置"和"修改"两个词在 Embedding 向量空间里相似度 0.71，**低于阈值 0.75 被截掉**。

#### 根因
- 通用 Embedding（BGE-large-zh）训练语料里，"重置"和"修改"在很多上下文（如"重置交换机"vs"修改配置"）并不互通。
- 余弦阈值是个**朴素但脆弱**的截断手段。靠近阈值的 query 极易丢。
- 没有 BM25 兜底 —— 而"重置"和"修改"都是高 IDF 词，BM25 本来能抓住。

#### 修复

**方案 A**：上 hybrid 检索（BM25 + Vector + RRF）。这是根本解。

```python
# 上面 2.4 节的 hybrid_search 已贴
results = hybrid_search(conn, query, query_vec, tenant_id)
```

**方案 B**：维护同义词词典，查询时自动扩展。

```python
SYNONYMS = {
    "重置": ["重置", "修改", "更改", "变更"],
    "登录": ["登录", "登入", "进入", "登陆"],
    # ...
}

def expand_query(q: str) -> list[str]:
    queries = [q]
    for k, syns in SYNONYMS.items():
        if k in q:
            queries.extend(q.replace(k, s) for s in syns if s != k)
    return list(set(queries))
```

**方案 C**：业务领域 fine-tune embedding。投入大但天花板最高，适合长期投入的核心业务。

```python
from sentence_transformers import losses, InputExample
# 准备 (query, positive_doc, negative_doc) 三元组
train_examples = [...]
loss = losses.MultipleNegativesRankingLoss(model)
model.fit([(train_dataloader, loss)], epochs=3)
```

#### 验证

```python
# 上线 hybrid 后，对原 badcase 集合重测
badcases = load_badcases("synonym_misses.jsonl")
hits = sum(1 for q in badcases
           if expected_doc_id(q) in [r.id for r in retrieve(q.text)[:10]])
print(f"hit rate: {hits / len(badcases):.2%}")
# 修复前 12% → 修复后 84%
```

> **口诀**：**只信 Embedding 必有同义词漏召**。BM25 + Vector 是底线，行业词不通需要词典或 fine-tune。

---

### 案例 ② Chunk 切坏 —— "答案被切到两个 chunk 中间"

#### 现象
用户问"如何申请退款"，知识库里有完整流程，但召回的 chunk 只包含前半段"打开订单详情..."，后半段"...点击右下角'申请退款'按钮"在另一个 chunk 里。

LLM 拿到截断的上下文回答"打开订单详情后无法找到退款入口"，**误导用户**。

#### 根因
原文是个 Markdown：

```markdown
## 退款流程

打开订单详情页，找到要退款的订单，

点击右下角"申请退款"按钮，按提示填写原因。
```

切片器按 800 字硬切，恰好把"找到要退款的订单，"这一句切到了 chunk 边界。下半句去了 chunk N+1。

更坑：overlap 设了 50 字符，但**前一句结尾和后一句开头都不带"申请退款"关键词**，向量检索完全找不到那个关键 chunk。

#### 修复

**方案 A**：改用 `MarkdownHeaderTextSplitter`，按 `##` 切，整段不切碎（见 2.1 节）。

**方案 B**：增大 overlap 到 200~300，**牺牲索引体积换边界容错**。

**方案 C**：上 Semantic Chunking —— 按句子相似度突变切。

```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_huggingface import HuggingFaceEmbeddings

chunker = SemanticChunker(
    HuggingFaceEmbeddings(model_name="BAAI/bge-large-zh-v1.5"),
    breakpoint_threshold_type="percentile",   # 句间相似度的分位数突变才切
    breakpoint_threshold_amount=85,
)
docs = chunker.split_text(long_text)
```

**方案 D**（兜底）：检索时**回溯前后相邻 chunk**，把上下文窗口扩到 ±1。

```python
def expand_window(hits: list[Chunk]) -> list[Chunk]:
    extended = set()
    for h in hits:
        extended.add((h.doc_id, h.chunk_idx - 1))
        extended.add((h.doc_id, h.chunk_idx))
        extended.add((h.doc_id, h.chunk_idx + 1))
    return load_chunks(extended)
```

这是工程上**最有效的兜底**：检索的 chunk_idx 前后各拿一个，让 LLM 看到完整段落。代价是上下文 token 翻 3 倍。

#### 验证

```python
# 复现原 badcase
hit = retrieve("如何申请退款")[0]
ctx = expand_window([hit])
ans = llm.generate(prompt(query="如何申请退款", context=ctx))
assert "申请退款按钮" in ans  # 完整答案
```

> **口诀**：**chunk 边界是 RAG 最隐蔽的杀手**。结构化切片 > Semantic 切片 > Fixed 切片。检索回来要"扩窗口"。

---

### 案例 ③ Embedding 不适配中文细粒度

#### 现象
团队最初用 OpenAI text-embedding-ada-002，中文 NDCG@10 只有 0.61。换 text-embedding-3-large 提升到 0.74，但仍然弱于业务预期。

特别是**中文术语缩写**："JD" vs "京东"、"小红书" vs "RED"、"红薯" vs "小红书"等，召回经常错位。

#### 根因
- ada-002 是**多语言模型**，中文语料权重低。
- v3-large 改善但仍是通用模型，中文俚语、品牌缩写、内部黑话覆盖不足。
- 业务词表没有进入训练分布，向量空间里它们和真实概念距离很远。

#### 修复

**方案 A**：换 BGE-large-zh-v1.5（实测 NDCG +4 个点，免费）。

**方案 B**：在 BGE 上跑领域 fine-tune。准备业务问答对：

```python
from sentence_transformers import SentenceTransformer, InputExample, losses
from torch.utils.data import DataLoader

model = SentenceTransformer("BAAI/bge-large-zh-v1.5")

train_examples = [
    InputExample(texts=["小红书APP怎么注册", "RED 平台账号注册流程"]),
    InputExample(texts=["JD 物流时效", "京东配送时间表"]),
    # 5000~10000 条业务真实问答对
]
loader = DataLoader(train_examples, shuffle=True, batch_size=32)
loss = losses.MultipleNegativesRankingLoss(model)
model.fit([(loader, loss)], epochs=3, warmup_steps=100)
model.save("bge-large-zh-tuned-v1")
```

**方案 C**：用 LLM 自动扩词典 + 同义词替换（短期方案，见案例 ①）。

#### 验证

| 模型 | NDCG@10 | 业务 badcase 通过率 |
| --- | --- | --- |
| ada-002 | 0.61 | 38% |
| v3-large | 0.74 | 56% |
| BGE-large-zh-v1.5 | 0.78 | 64% |
| BGE 业务 fine-tuned | **0.84** | **81%** |

> **口诀**：**通用 Embedding 永远不够**。中文先换 BGE，业务词表多就 fine-tune。

---

### 案例 ④ Top-K 太大，LLM 被噪声淹没

#### 现象
为了"召回更全"，工程师把 Top-K 从 5 调到 20。**结果回答质量反而下降**：LLM 经常引用无关 chunk，答非所问。

#### 根因
- LLM 的注意力不是"无差别全看"。**上下文越长，关键信息越容易被忽略**（"Lost in the Middle" 现象，见 Liu et al. 2023）。
- Top-20 里只有 3~5 条真相关，其余 15 条是噪声。LLM 可能把错误信息当作"补充上下文"用。
- 没有 reranker 截断，相关性低的 chunk 也排在前面。

#### 修复

**方案 A**：上 Reranker，召回 Top-50 → 精排 Top-5（见 2.5）。

**方案 B**：动态 K：根据 reranker score 阈值决定截多少。

```python
def adaptive_topk(candidates, min_k=3, max_k=10, score_threshold=0.7):
    candidates.sort(key=lambda x: -x["rerank_score"])
    selected = []
    for c in candidates:
        if len(selected) < min_k:
            selected.append(c)
        elif c["rerank_score"] >= score_threshold and len(selected) < max_k:
            selected.append(c)
        else:
            break
    return selected
```

**方案 C**：上下文压缩。LLM 先做 chunk 摘要，再喂给生成 LLM。

```python
COMPRESS_PROMPT = """
你是一个文档摘要专家。给定查询和文档，提取与查询最相关的关键信息（≤80 字）。
查询：{query}
文档：{chunk}
关键信息：
"""
```

注意：上下文压缩**多了一次 LLM 调用**，对延迟和成本敏感的场景慎用。

#### 验证
A/B 测试两组：

| 配置 | Faithfulness | Answer Relevancy | P95 延迟 |
| --- | --- | --- | --- |
| Top-20 raw | 0.81 | 0.74 | 1.8s |
| Top-50 + Rerank Top-5 | **0.92** | **0.88** | 1.2s |

> **口诀**：**上下文不是越多越好**。LLM 的注意力是稀缺资源，**Reranker + 动态 K**。

---

### 案例 ⑤ 没 Reranker，相关文档排在第 8 位被丢

#### 现象
监控显示 Recall@10 = 88%，但 Recall@5 = 62%。意思是**相关 chunk 经常被召回，但排不到前 5**。LLM 拿到的总是次相关的。

#### 根因
- bi-encoder 的相似度只能粗排。同语义的文档之间，向量距离差异极小（0.85 vs 0.83 可能完全等价）。
- 没有 cross-encoder 精排，**第 1 名 vs 第 10 名的差异主要来自噪声**。

#### 修复
直接接 BGE Reranker（2.5 节）。

加了之后 Recall@5 立刻从 62% 拉到 84%，端到端 NDCG@10 从 0.78 → 0.91。

**这是 RAG 工程化里 ROI 最高的一步**。如果你的 RAG 没接 reranker，第一个该补的就是它。

#### 验证
固定 query 集合，扫一遍。看 Recall@5 是否单调上升。

> **口诀**：**朴素 RAG 别用 Top-5 直出**。Top-50 召回 + Reranker 精排到 Top-5，是工程标配。

---

### 案例 ⑥ 数据更新没同步 —— "AI 回答还在用三个月前的政策"

#### 现象
公司更新了员工请假政策，但 RAG 助手仍然回答旧版规则。运营反馈"AI 在传谣"。

排查发现知识库的 Markdown 已经更新，但**向量库里还是旧版的 embedding**。

#### 根因
- 索引流程是手动触发的：编辑提交文档 → 找工程师跑离线 embedding 脚本 → 写入向量库。
- 编辑改完文档没通知工程师，工程师不知道要重跑。
- 即使重跑，旧的 chunk 没有被删除，**新旧 chunk 同时在库里**，老版本反而因为词频高被召回。

#### 修复

**方案 A**：CDC（Change Data Capture）增量同步。

```python
# 监听文档变更事件
@on_event("doc_updated")
async def reindex(doc_id: int):
    # 1. 删除该 doc 下所有旧 chunk
    await db.execute("DELETE FROM doc_chunks WHERE doc_id = $1", doc_id)
    # 2. 重新切片 + 嵌入 + 写入
    chunks = chunk_document(load_doc(doc_id))
    vecs = await embed_all([c.content for c in chunks])
    await bulk_insert(doc_id, chunks, vecs)
    # 3. 写审计日志
    log.info("reindexed doc_id=%d, %d chunks", doc_id, len(chunks))
```

**方案 B**：版本字段 + soft delete。

```sql
ALTER TABLE doc_chunks ADD COLUMN version INT DEFAULT 1;
ALTER TABLE doc_chunks ADD COLUMN is_active BOOL DEFAULT TRUE;

-- 检索时只选活跃版本
SELECT ... WHERE is_active = TRUE ORDER BY embedding <=> $1 LIMIT 50;
```

更新时新版本写入，旧版本 `is_active = FALSE`。这样**任何时候都能回滚**，且无双写竞争。

**方案 C**：定期全量校对。每天凌晨跑一次：源文档 hash vs 向量库 hash，不一致就重建。

#### 验证

- 改一篇文档，10 秒内查询能拿到新版（CDC 模式）
- 旧 chunk 数 = 0（DELETE 生效）
- 周期性回归测试集，准确率不随时间下降

> **口诀**：**RAG 不是"建一次就完"，是"持续同步"**。CDC 触发 + 版本管理 + 定期对账。

---

### 案例 ⑦ 元数据过滤缺失 —— "跨用户数据泄漏"

#### 现象
SaaS 产品上线后，安全团队 audit 发现 A 公司的用户能查到 B 公司的内部文档。**严重事故**。

#### 根因
工程师为了让向量检索"够快"，把所有租户的 chunk 全塞一个表里，**没有 tenant_id 过滤**：

```python
# ★ 漏洞：没 tenant_id
results = vector_db.search(query_vec, top_k=10)
```

向量检索本身没有"权限"概念，**漏一行 SQL 就漏数据**。

#### 修复

**方案 A**：所有检索强制带 tenant_id（应用层守门）。

```python
def safe_retrieve(tenant_id: int, query: str, query_vec: list[float]):
    assert tenant_id > 0, "tenant_id required"
    return db.execute("""
        SELECT id, content FROM doc_chunks
         WHERE tenant_id = %s
         ORDER BY embedding <=> %s::vector LIMIT 50
    """, (tenant_id, query_vec))
```

**方案 B**：物理隔离 —— 每个租户独立 schema 或独立索引。

| 隔离方式 | 优点 | 缺点 |
| --- | --- | --- |
| 单表 + tenant_id 过滤 | 简单 | 漏过滤就泄漏 |
| 单表 + RLS（PG Row Level Security） | 数据库强制 | 需要 PG 配置 |
| 每租户独立 schema | 强隔离 | 运维成本高 |
| 每租户独立向量库 | 最强隔离 | 资源浪费 |

**方案 C**：PG 的 RLS（Row Level Security）—— 一次配好永不漏。

```sql
ALTER TABLE doc_chunks ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON doc_chunks
    USING (tenant_id = current_setting('app.tenant_id')::BIGINT);

-- 应用连接后必须先 SET
SET app.tenant_id = 12345;
SELECT ... FROM doc_chunks ORDER BY ...;  -- 自动只看到 tenant 12345 的
```

**方案 D**：Milvus / Weaviate 用 partition key，按 tenant 物理分区。

```python
from pymilvus import Collection
collection = Collection("docs")
# tenant_id 设为 partition key
collection.search(
    data=[query_vec],
    anns_field="embedding",
    param={"metric_type": "COSINE", "params": {"ef": 128}},
    limit=50,
    partition_names=[f"tenant_{tenant_id}"],
)
```

#### 验证
**强制做权限测试**：

```python
def test_tenant_isolation():
    # 用 tenant A 的文档插入
    insert_chunks(tenant_id=1, docs=["A 的秘密文档"])
    # 用 tenant B 的身份查询
    results = safe_retrieve(tenant_id=2, query="秘密文档")
    assert all(r["tenant_id"] == 2 for r in results), "数据泄漏！"
```

> **口诀**：**多租户 RAG 必须强制 tenant_id**。能用 RLS 用 RLS，能物理分区物理分区。**应用层守门一定有漏的那天**。

---

### 案例 ⑧ 评估只用 LLM，指标虚高 12%

#### 现象
团队跑了 RAGAS 出来 `faithfulness=0.92, answer_relevancy=0.88`，老板很满意。但用户投诉率持续上升。

人工抽样 200 条对话，**实际正确率只有 76%**。

#### 根因
- RAGAS 默认用 GPT-4 当评委。但**GPT-4 评 GPT-4 自己生成的答案**，倾向于"宽容"打高分（同源偏差）。
- Faithfulness 只看"有没有上下文支持"，不看"上下文本身对不对"—— **召回错了上下文，答案"忠实地复述错误"也是高分**。
- Answer Relevancy 容易被"长答案"骗 —— 只要包含原问题关键词，相关性就高，**实际答非所问**也通过。

#### 修复

**方案 A**：人工评估必不可少。每周抽 100 条对话，2 个评委独立打分（Cohen's kappa 看一致性）。

```python
HUMAN_EVAL_FORM = {
    "is_correct": [True, False, "partial"],
    "is_complete": [True, False],
    "has_hallucination": [True, False],
    "tone_appropriate": [True, False],
    "comments": "free text"
}
```

**方案 B**：换更强的评委 LLM 或多评委集成。

```python
async def llm_judge(question, answer, context):
    judges = ["gpt-4o", "claude-3.5-sonnet", "gemini-1.5-pro"]
    scores = await asyncio.gather(*[
        score_with(j, question, answer, context) for j in judges
    ])
    return sum(scores) / len(scores)
```

**方案 C**：业务侧硬指标兜底 —— 让用户给反馈（点赞 / 踩 / 转人工率）。

```python
# 真实硬指标
business_metrics = {
    "thumbs_up_rate": 0.72,         # 用户点赞率
    "escalate_rate": 0.18,          # 转人工率
    "session_resolution_rate": 0.65, # 一次会话解决率
}
```

#### 验证
- 周 100 条人工评测分数 vs RAGAS 分数对比，差距 > 5% 报警
- 业务硬指标和 RAGAS 趋势是否同向（如果背离，说明 RAGAS 指标失真）

> **口诀**：**LLM-as-judge 必须配人工抽检**。指标只能用来发现回归，最终质量靠业务硬数据。

---

### 案例 ⑨ 长尾问题无改进，A/B 测试看不出差别

#### 现象
团队上了一个新 reranker 模型，离线评测 NDCG@10 +3 个点。线上灰度 A/B 测了两周，**业务指标没有任何统计显著差异**。

#### 根因
- 真实流量 80% 是高频问题（FAQ Top 100），新 reranker 对这部分**早就召回完美**，毫无提升空间。
- 收益全部体现在长尾 20% 的问题，**但样本量在 A/B 桶里太少**，统计上看不出。
- 评估指标只看整体均值，**长尾的提升被高频的"无变化"稀释**。

#### 修复

**方案 A**：分层评估。把流量按 query 频次分桶，分别看效果。

```python
buckets = {
    "head": top_100_queries,           # 占 80% 流量
    "torso": next_900_queries,         # 占 15%
    "tail": remaining_thousands,       # 占 5%
}
for name, queries in buckets.items():
    ndcg = evaluate(queries, new_reranker)
    print(f"{name}: NDCG={ndcg:.3f}, n={len(queries)}")
```

**方案 B**：长尾扩样。专门为长尾 query 加大 A/B 桶比例。

```python
# 路由层
def ab_route(query: str) -> str:
    if is_long_tail(query):
        return random.choice(["A", "B"])  # 长尾 50/50
    else:
        return "A" if random.random() < 0.95 else "B"  # 高频 95/5
```

**方案 C**：换关键指标。从"整体准确率"换成"长尾准确率"或"未解决问题首次解决率"。

#### 验证
分层后看到：

| 桶 | 旧 reranker NDCG | 新 reranker NDCG | 提升 |
| --- | --- | --- | --- |
| head | 0.94 | 0.94 | 0 |
| torso | 0.81 | 0.85 | +4 |
| tail | **0.62** | **0.74** | **+12** |

长尾的 12 个点提升被"埋"在了整体平均里。

> **口诀**：**RAG 改进 80% 收益在长尾**。评估必须分层，A/B 必须按 query 频次分桶。

---

## 5. 上线检查清单

### 索引（Index）
- [ ] Loader 能处理 PDF 表格 / 公式 / 扫描件 OCR；至少抽样 50 篇人工检查抽出文本完整性
- [ ] Chunk 策略与文档类型匹配（FAQ → 512、长文 → 1024、代码 → AST）
- [ ] 中文 splitter 显式配置中文标点 separators
- [ ] Chunk overlap 不少于 chunk_size 的 10%
- [ ] 元数据（doc_id、tenant_id、source、updated_at、permissions）随 chunk 入库
- [ ] CDC / 增量同步链路打通；文档更新 < 60s 反映到向量库
- [ ] 索引重建脚本幂等（重跑不会产生重复 chunk）

### 检索（Retrieve）
- [ ] Embedding 模型经过领域评测，不只看 MTEB 榜单
- [ ] BM25 兜底（hybrid search），中文 BM25 用 IK / jieba 分词器
- [ ] HNSW `ef_search` 在线可调，至少有"高召回"和"低延迟"两套预设
- [ ] 多租户场景强制 `tenant_id` 过滤；推荐 PG RLS 或 partition key 兜底
- [ ] 元数据过滤先于向量检索（避免 filter explosion）
- [ ] Reranker 接入；召回 Top-50 → 精排 Top-5
- [ ] Reranker 部署独立 GPU 服务，开启请求级 batching
- [ ] 检索结果带 `chunk_idx`，支持上下文窗口扩展（±1 chunk）

### 生成（Generate）
- [ ] Prompt 模板里强制要求"基于上下文回答，未提及的内容回答'不知道'"
- [ ] 输出强制带引用（`[1]`、`[2]`），引用 chunk_id 写日志
- [ ] LLM 输出超出上下文的内容必须告警（faithfulness 兜底）
- [ ] 上下文 token 预算预留 30% 给输出
- [ ] 流式输出（SSE / chunked）减少用户感知延迟

### 评估（Evaluate）
- [ ] RAGAS 至少跑 4 个核心指标（faithfulness / answer_relevancy / context_precision / context_recall）
- [ ] 评委 LLM 用 GPT-4o 或 Claude 3.5 Sonnet 级别，不用 GPT-3.5
- [ ] 人工评估每周至少 100 条样本，两个评委独立打分
- [ ] 业务硬指标（点赞率 / 转人工率 / 解决率）与 RAGAS 趋势对齐
- [ ] 评估集分桶（head / torso / tail），分别看
- [ ] 上线前固定 badcase 集合（≥ 200 条）必须全部通过

### 运维（Ops）
- [ ] 检索 P99、Reranker P99、生成 P99 分别监控
- [ ] 召回为 0 / 召回 < min_score 阈值的请求打点告警
- [ ] LLM 调用失败 / 限流 / 超时分别告警
- [ ] 成本看板：每日 token 消耗、单次平均成本、Top 10 高消耗 query
- [ ] 灰度发布：新模型 / 新策略先 1% 流量，分桶 A/B
- [ ] 文档版本回滚演练：5 分钟内能回滚到前一版

---

## 6. 还能往下挖什么

### Agentic RAG —— 多步检索 + 工具调用
朴素 RAG 是"一次检索 + 一次生成"。复杂问题（"对比 A、B 两个产品在低温环境下的表现"）需要 **多次检索 + 中间推理**。看 ReAct、Self-Ask、Plan-and-Execute 模式。框架：LangGraph / AutoGen / CrewAI。

### GraphRAG —— 知识图谱 + 向量
微软 2024 年开源的 GraphRAG 用 LLM 抽实体关系建图，再用社区检测做摘要。**对"全局问题"有奇效**（如"这本书的核心主题是什么"），但建图成本极高。看 Neo4j + LangChain 的 GraphCypherQAChain。

### ColBERT 与后期交互模型
ColBERT 在 query 和 doc 的 token 级做晚期交互，**精度接近 cross-encoder，速度接近 bi-encoder**。生产部署门槛较高（索引体积大），但在专业搜索场景值得评估。代表实现：RAGatouille。

### Multi-modal RAG —— 图、表、文混合
PDF 里的图表、流程图、公式不能简单 OCR 转文字。CLIP-style 图文 embedding + 多模态 LLM（GPT-4V / Gemini Vision / Qwen-VL）能让"问图里的内容"成为可能。坑在 chunk 边界（图和说明文字怎么对齐）。

### 长上下文 vs RAG 边界
1M token 模型让"小语料 + 长上下文"变得可行。但成本、延迟、注意力散失（Lost in the Middle）三个问题让 RAG 在大规模场景仍不可替代。看 Anthropic 的 Contextual Retrieval（2024）—— 把上下文加回 chunk 再嵌入。

### RAG-Fusion / Self-Query / Recursive Retrieval
- **RAG-Fusion**：Multi-Query + RRF 的进化版
- **Self-Query**：用 LLM 把自然语言转成结构化查询（含元数据 filter）
- **Recursive Retrieval**：层级文档先检索摘要，再检索详情

### 私有知识 + 联邦检索
跨多个知识库（公司内、公开网络、第三方 API）联合检索。难点在权限隔离 + 跨库分数对齐 + 来源追溯。框架：LlamaIndex 的 RouterQueryEngine。

### LLM 微调 + RAG 协同
- **RAFT (Retrieval Augmented Fine Tuning)**：让 LLM 学会"区分上下文中的相关 vs 干扰信息"
- **LoRA + Retrieval**：领域 LoRA 微调小模型 + RAG，比 RAG-only 强 5~10 个点
- 看 Microsoft RAFT 论文（2024）和 Berkeley Gorilla 项目

---

## 7. 引用

### 论文
- Lewis et al., *Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks*（2020）—— RAG 原始论文
- Karpukhin et al., *Dense Passage Retrieval for Open-Domain Question Answering*（2020）—— DPR
- Khattab et al., *ColBERT: Efficient and Effective Passage Search via Contextualized Late Interaction*（2020）
- Gao et al., *Precise Zero-Shot Dense Retrieval without Relevance Labels*（2022）—— HyDE
- Liu et al., *Lost in the Middle: How Language Models Use Long Contexts*（2023）
- Edge et al., *From Local to Global: A Graph RAG Approach to Query-Focused Summarization*（Microsoft 2024）
- Zhang et al., *RAFT: Adapting Language Model to Domain Specific RAG*（2024）
- Es et al., *RAGAS: Automated Evaluation of Retrieval Augmented Generation*（2023）

### 开源框架
- [LangChain](https://github.com/langchain-ai/langchain) / [LangGraph](https://github.com/langchain-ai/langgraph)
- [LlamaIndex](https://github.com/run-llama/llama_index)
- [Haystack](https://github.com/deepset-ai/haystack)
- [Spring AI](https://github.com/spring-projects/spring-ai)（Java 生态）

### 向量库
- [pgvector](https://github.com/pgvector/pgvector)
- [Milvus](https://github.com/milvus-io/milvus)
- [Qdrant](https://github.com/qdrant/qdrant)
- [Weaviate](https://github.com/weaviate/weaviate)
- [Chroma](https://github.com/chroma-core/chroma)

### 嵌入与重排
- [BGE 系列](https://github.com/FlagOpen/FlagEmbedding)（北京智源）
- [GTE 系列](https://huggingface.co/thenlper/gte-large-zh)（阿里达摩院）
- [Cohere Rerank](https://docs.cohere.com/docs/rerank-2)
- [Jina Reranker](https://jina.ai/reranker/)

### 评估工具
- [RAGAS](https://github.com/explodinggradients/ragas)
- [TruLens](https://github.com/truera/trulens)
- [DeepEval](https://github.com/confident-ai/deepeval)
- [Phoenix (Arize)](https://github.com/Arize-ai/phoenix)

### 数据集
- [MTEB Benchmark](https://huggingface.co/spaces/mteb/leaderboard) —— 多语言嵌入评测
- [BEIR](https://github.com/beir-cellar/beir) —— 信息检索基准
- [C-MTEB](https://github.com/FlagOpen/FlagEmbedding/tree/master/C_MTEB) —— 中文 MTEB

---

> 配套：[LLM 应用工程化](/article/ai/llm-app) ｜ [RAG 速记](/article/ai/rag) ｜ [Prompt 工程实战](/article/ai/prompt)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。下一篇候选：[Agent 协议与工具调用内幕](/article/ai/agent) / [向量库底层实现对比](/article/db) —— 想看哪个告诉我。
