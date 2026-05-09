# Vector Database 深入 —— HNSW / IVF / Milvus / Qdrant / pgvector 选型

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想速记 RAG 架构，去 [RAG 工程化](/article/ai/rag-engineering) 那一节够了。
> 这里要回答的，是**搭 RAG 选型时纠结、上线后召回率掉了一半、半夜被 P99 飙升告警叫起来**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. 你给 1000 万条 768 维向量做 `SELECT` 最近邻，**精确查询**和 **ANN（Approximate Nearest Neighbor，近似最近邻）** 的延迟差距大约是几个数量级？为什么差这么多？
2. HNSW 的 `M`、`efConstruction`、`efSearch` 三个参数，**改哪个会影响内存？改哪个只影响查询时延？** 给一个生产配置组合。
3. 你给 ANN 索引加了一个 `WHERE category = 'shoes'` 过滤，召回率从 95% 掉到 30% —— **现象的根因**是什么？Milvus / Qdrant / pgvector 三家的应对策略各不一样，能说清吗？
4. Hybrid Search（向量 + BM25 关键词）的融合算法 RRF（Reciprocal Rank Fusion）相比简单加权，**优势**是什么？为什么不需要归一化两边的分数？
5. 你把 1536 维向量从 FP32 量化成 INT8，存储省 4 倍 —— **召回率掉多少**？什么场景下不能量化？
6. pgvector 的 HNSW 索引在 PostgreSQL 中**单表超过多少向量后开始扛不住**？为什么 `CREATE INDEX CONCURRENTLY` 在向量索引上格外重要？
7. Milvus 的存算分离架构里，`QueryNode` 和 `IndexNode` 各自负责什么？为什么有人吐槽它"用大炮打蚊子"？
8. 你给一个 5000 万向量集合做了删除操作，过了一周发现**查询变慢了 3 倍** —— 大概率是哪个内部数据结构的问题？

如果有半数答不上，往下看。

---

## 1. 底层模型：高维空间最近邻为什么不能用 B-Tree

绝大多数文章把"向量数据库"讲成"加了向量字段的数据库"，然后画一个 HNSW 图就结束了。这层认知不够选型也不够诊断生产问题。要看懂上面 8 个问题，必须下沉到 ANN 算法和索引结构本身。

### 1.1 向量检索的本质 —— 在高维空间找最近邻

给定一个 d 维向量 q（query），从 N 条 d 维向量集合中找出与 q 距离最近的 k 个 —— 这就是 kNN（k-Nearest Neighbors）问题。

精确解的复杂度 O(N·d)：每条向量都算一次距离。1000 万 × 768 维 = 76 亿次乘加，单机 CPU 跑一次要几秒到几十秒。

为什么不能用 B-Tree？B-Tree 是**一维有序**结构，靠 `<` 关系切分空间。**高维空间没有自然的全序**：你怎么定义 `(0.1, 0.9, 0.3) < (0.5, 0.2, 0.7)`？

KD-Tree、R-Tree 这类**多维空间索引**在低维（≤ 10 维）有效，但**维度灾难（curse of dimensionality）**会让它们在 100 维以上退化为线性扫描 —— 高维空间里，"远"和"近"的差异被稀释，几乎所有点到 q 的距离都相近。

> **口诀**：高维 ANN 的本质是 **用精度换速度**，没法走传统树索引那条路。

### 1.2 距离度量 —— Cosine / L2 / Inner Product

| 度量 | 公式 | 适用场景 |
| --- | --- | --- |
| **L2（欧氏距离）** | `‖q - v‖₂ = √Σ(qᵢ - vᵢ)²` | 通用，对向量长度敏感 |
| **Cosine（余弦相似度）** | `cos(q, v) = q·v / (‖q‖·‖v‖)` | 文本 embedding（OpenAI / BGE 等） |
| **Inner Product（点积）** | `q·v = Σ qᵢ·vᵢ` | 已归一化向量、推荐系统 |

**关键认知**：如果向量已经 L2 归一化（norm = 1），三者等价 —— `‖q - v‖² = 2 - 2·cos(q, v)`。OpenAI `text-embedding-3-small` 输出的向量就是归一化的，三种度量结果一致，但**计算成本差异巨大** —— 点积 O(d) 一次乘加，L2 多了减法和开方。

```python
# 验证：归一化向量下三种度量等价
import numpy as np
np.random.seed(0)
q = np.random.randn(768); q /= np.linalg.norm(q)
v = np.random.randn(768); v /= np.linalg.norm(v)

l2 = np.linalg.norm(q - v)
cos = np.dot(q, v)
print(f"L2² = {l2**2:.6f}, 2-2cos = {2 - 2*cos:.6f}")  # 完全一致
```

选型铁律：embedding 模型文档里写的是哪个度量，你就用哪个。BGE 用 Cosine，CLIP 用 Cosine，OpenAI 用 Cosine 或 Inner Product（归一化后等价）。**混用会导致召回率显著下降**。

### 1.3 ANN 算法家族 —— HNSW / IVF / PQ / LSH 四大门派

```
                       ANN
                        │
       ┌────────────────┼────────────────┐
   基于图           基于聚类           基于哈希
       │                │                │
     HNSW             IVF              LSH
   NSG / DiskANN    IVF + PQ          (已被淘汰)
                    IVF + SQ
                        │
                  量化压缩 (PQ / SQ / Binary)
```

| 算法 | 召回率 | 查询延迟 | 内存占用 | 构建速度 | 适用场景 |
| --- | --- | --- | --- | --- | --- |
| **HNSW** | 高（95%+） | 极低（< 10ms） | 高（图结构） | 慢 | 中小规模（< 1 亿） |
| **IVF + Flat** | 中（90%） | 中（10~50ms） | 高 | 快 | 中等规模 |
| **IVF + PQ** | 低~中（80~90%） | 低 | **低**（压缩 4~32 倍） | 中 | 海量（> 1 亿） |
| **LSH** | 低（70~85%） | 中 | 中 | 快 | 现在几乎不用 |
| **DiskANN** | 中~高 | 中（依赖 SSD） | **极低** | 慢 | 单机十亿级 |

**LSH 为什么淘汰了**：高维下需要的哈希表数量爆炸，召回率比 HNSW 差，工程实现也复杂。学术上还有研究价值，工业上 2020 年后基本被 HNSW 取代。

> **口诀**：**中小规模选 HNSW，海量选 IVF+PQ，单机十亿选 DiskANN**。

### 1.4 HNSW 的多层图结构 —— 为什么是"导航小世界"

HNSW（Hierarchical Navigable Small World）灵感来自跳表（Skip List）+ 小世界网络（Watts-Strogatz）。

```
Layer 2:        N1 ─────────────── N4              （稀疏，长边）
                 │                  │
                 │                  │
Layer 1:    N1───N2───────────────N4───N6
             │   │                 │   │
             │   │                 │   │
Layer 0:   N1─N2─N3─N4─N5─N6─N7─N8─N9─N10        （稠密，短边）
           ↑                       ↑
        (entry)                  (target)
```

每个节点有一个**层级**，按指数衰减分布：

```python
# layer 分配公式（来自 Malkov 论文）
import math, random
def assign_layer(mL):
    return int(-math.log(random.random()) * mL)
# mL = 1 / ln(M)，M 是每层的最大邻居数
```

查询路径：
1. 从顶层入口节点开始
2. 在当前层贪心搜索：找当前节点邻居中离 q 最近的，跳过去；直到找不到更近的
3. 把当前节点作为下一层的入口，重复
4. 在 Layer 0 用 `efSearch` 大小的优先队列搜索得到候选

**为什么这个设计能工作**：上层是**长跳**（覆盖大范围），下层是**短跳**（精细定位）—— 类似跳表，但每一跳是"局部贪心选最近邻"，整体复杂度 O(log N)。

**三个核心参数**：

| 参数 | 含义 | 影响 |
| --- | --- | --- |
| **M** | 每层每个节点的最大邻居数 | **内存** + 召回率（M 越大图越密，召回率越高，内存线性增长） |
| **efConstruction** | 构建时的候选队列大小 | **构建时间** + 索引质量（越大构建越慢但图越好） |
| **efSearch** | 查询时的候选队列大小 | **查询延迟** + 召回率（动态可调，调大延迟变慢） |

参数推荐（768 维文本向量，1000 万规模，工业界普遍配置）：
- `M = 16`（小集合）/ `M = 32`（大集合 / 高维）
- `efConstruction = 200`（默认）/ `efConstruction = 400`（追求高召回）
- `efSearch = 50`（线上低延迟）/ `efSearch = 200`（高召回场景）

> **口诀**：**`M` 拍一次内存定下来；`efConstruction` 构建期决定一次；`efSearch` 是唯一线上可调的旋钮**。

### 1.5 IVF + PQ —— 用聚类切片 + 量化压缩换内存

IVF（Inverted File Index）思路是 **k-means 聚类预切片**。

构建：
1. 对全量向量跑 k-means，得到 `nlist` 个聚类中心（centroid）
2. 把每个向量分配到最近的 centroid
3. 形成倒排：`centroid_i → [向量列表]`

查询：
1. 算 q 到每个 centroid 的距离
2. 选最近的 `nprobe` 个 centroid，只在它们的倒排里精确搜索

参数推荐：
- `nlist = 4 * sqrt(N)`（经验公式，N 是向量总数）
- `nprobe = nlist / 32 ~ nlist / 8`（覆盖度 vs 速度的权衡）

**召回率与 nprobe 强相关**：nprobe 太小，q 落在 cluster 边界附近时会漏掉真正的最近邻 —— 这就是 IVF 的"边界问题"。

PQ（Product Quantization，乘积量化）解决**内存压缩**：

```
原始 768 维向量 [f0, f1, ..., f767]，每维 4 字节，共 3072 字节

切成 M = 8 个子向量，每段 96 维：
  [s0(96维), s1(96维), ..., s7(96维)]

每段独立做 k-means(k=256)，得到 8 个 codebook
每段用 1 字节（256 = 2^8）的 codebook ID 表示

压缩后：8 字节 → 384 倍压缩（相对原始 3072 字节）
```

距离计算用 ADC（Asymmetric Distance Computation）：query 不量化，预算好 query 到每个 codebook 中心的距离表，每个候选向量查 8 次表加起来。

> **口诀**：**IVF 是粗筛，PQ 是压缩**。组合 IVF+PQ 能把 1 亿条 768 维向量从 300GB 压到几 GB，但召回率会从 95% 降到 80~85%，**精度敏感场景慎用**。

### 1.6 Hybrid Search —— 向量召回 + 关键词召回的融合

纯向量召回的痛点：**对人名、产品型号、专业术语这类"字面匹配"差**。"iPhone 15 Pro Max" 的 embedding 和 "iPhone 14 Pro" 可能极相似，但你要的是精确匹配。

Hybrid Search 同时跑两路：
- **向量召回**（语义匹配）：HNSW 返回 top-K 候选
- **关键词召回**（BM25）：倒排索引返回 top-K 候选

融合算法 RRF（Reciprocal Rank Fusion）：

```python
def rrf(rankings: list[list[str]], k: int = 60) -> dict[str, float]:
    """rankings: 多路召回结果，每路是有序的 doc_id 列表
       k: 平滑常数，论文推荐 60，越大越平均
       返回：{doc_id: rrf_score}"""
    scores = {}
    for ranking in rankings:
        for rank, doc_id in enumerate(ranking):
            scores[doc_id] = scores.get(doc_id, 0) + 1.0 / (k + rank + 1)
    return scores
```

**RRF 的妙处**：
- **不需要归一化分数** —— 只看排名，不看分数。BM25 分数和 cosine 相似度根本不在一个量纲，RRF 优雅绕过。
- **对异常值鲁棒** —— 一路偶尔给某个 doc 打了过高分，另一路没召回它，最终也不会冲到第一。
- **k = 60** 是 Cormack 论文的推荐值；调小（k=10）让头部更陡峭，调大（k=200）更平均。

实测下来 RRF 在大多数 RAG 场景下比"加权和"更稳。Elasticsearch 8.8+、Qdrant 1.10+、Milvus 2.4+ 都内置了 RRF。

### 1.7 元数据过滤 —— Pre-filter vs Post-filter 的死局

业务场景：`SELECT * FROM products WHERE category = 'shoes' ORDER BY embedding <-> :query LIMIT 10`。

ANN 索引和元数据过滤天生不和。三种实现策略：

| 策略 | 描述 | 优点 | 致命缺陷 |
| --- | --- | --- | --- |
| **Post-filter** | 先 ANN 取 top-K，再按 category 过滤 | 快 | category 命中率低时（如 1%），可能 K 取了 1000 还过滤不到 10 条 |
| **Pre-filter** | 先按 category 取出符合的 ID 集合，再在子集上做 ANN | 召回完整 | 子集大时退化为线性扫描；HNSW 图被破坏，无法走索引 |
| **Filtered HNSW** | 在 HNSW 遍历时检查每个节点是否符合过滤条件，不符合就跳过 | 平衡 | 过滤率高时（命中 < 10%）会让 HNSW 退化（卡在某个连通分量里出不来） |

**Qdrant 的策略**（业界最先进）：根据过滤命中率动态选择
- 命中率 > 5%：用 Filtered HNSW（在图遍历时过滤）
- 命中率 < 5%：用 plain search（直接扫子集）

**Milvus 的策略**：partition + scalar index，把高频过滤字段单独分区，分区内独立建 HNSW 索引。

**pgvector 的策略**：Post-filter 为主，靠 PostgreSQL 优化器决定。这是 pgvector 在"高过滤率 + 大表"场景下被吐槽的根本原因。

> **口诀**：**过滤率 > 50% 用 Post-filter；< 5% 用 Pre-filter；中间用 Filtered HNSW**。选型时一定测自己业务的过滤分布。

---

## 2. 主流方案切片：6 家向量库的架构与代码

> 每段挑关键路径，附可运行示例。版本基于 2025-Q2。

### 2.1 Milvus —— 存算分离的"分布式向量库"

Milvus 2.x 的架构是 **彻底的存算分离**，受 Snowflake 启发：

```
┌────────────────────────────────────────────────┐
│              Coordinator 层（控制面）          │
│   RootCoord / DataCoord / QueryCoord / IndexCoord │
└────────────────────────────────────────────────┘
        │              │             │
        ▼              ▼             ▼
   ┌─────────┐   ┌──────────┐  ┌──────────┐
   │ProxyNode│   │QueryNode │  │IndexNode │
   │(SDK 入口)│   │(查询)    │  │(构建索引) │
   └─────────┘   └──────────┘  └──────────┘
        │              │             │
        └──────────────┼─────────────┘
                       ▼
            ┌──────────────────────┐
            │  对象存储 (S3 / MinIO) │  ← 唯一持久化层
            │  消息队列 (Kafka / Pulsar)│ ← WAL 角色
            └──────────────────────┘
```

**优点**：每个节点无状态，可独立扩缩容；查询节点和构建节点不抢资源。

**缺点**：**组件多，运维重**。最小生产部署（StandaloneMode 不算）至少 8 个 Pod。所谓"用大炮打蚊子" —— 单机 1000 万向量场景，部署 Milvus 比 pgvector 复杂 10 倍但快不到 2 倍。

```python
# Milvus 2.x 完整示例（pymilvus 2.4+）
from pymilvus import MilvusClient, DataType

client = MilvusClient(uri="http://localhost:19530")

# 1. 建集合 + schema
schema = MilvusClient.create_schema(auto_id=True, enable_dynamic_field=True)
schema.add_field("id", DataType.INT64, is_primary=True)
schema.add_field("embedding", DataType.FLOAT_VECTOR, dim=768)
schema.add_field("category", DataType.VARCHAR, max_length=64)

# 2. 索引参数（HNSW）
index_params = client.prepare_index_params()
index_params.add_index(
    field_name="embedding",
    index_type="HNSW",
    metric_type="COSINE",
    params={"M": 16, "efConstruction": 200}
)

client.create_collection("docs", schema=schema, index_params=index_params)

# 3. 写入
client.insert("docs", [{"embedding": vec, "category": "shoes"} for vec in vectors])

# 4. 查询（含过滤）
results = client.search(
    collection_name="docs",
    data=[query_vec],
    limit=10,
    filter='category == "shoes"',
    search_params={"metric_type": "COSINE", "params": {"ef": 100}}
)
```

适用场景：**亿级以上规模 + 多租户 + 需要严格 SLA** 的大厂。中小项目几乎都过度。

### 2.2 Qdrant —— Rust 写的"可运维向量库"

Qdrant 用 Rust 实现，单二进制部署，资源占用小，分片和复制内置。

架构特点：
- **每个 collection 内部是 N 个 segment**，每个 segment 是独立的 HNSW 索引
- segment 写满后冻结，新数据写新 segment（类似 LSM）
- 查询时并行查所有 segment，合并结果

**Qdrant 的核心优势**：Filtered HNSW 实现最成熟（见 1.7），filter 性能业界第一。

```python
# Qdrant Python SDK（qdrant-client 1.10+）
from qdrant_client import QdrantClient
from qdrant_client.models import (
    VectorParams, Distance, PointStruct,
    HnswConfigDiff, Filter, FieldCondition, MatchValue
)

client = QdrantClient(url="http://localhost:6333")

# 1. 建 collection + 自定义 HNSW
client.recreate_collection(
    collection_name="docs",
    vectors_config=VectorParams(size=768, distance=Distance.COSINE),
    hnsw_config=HnswConfigDiff(m=16, ef_construct=200, full_scan_threshold=10_000),
    # full_scan_threshold: 过滤后剩这么多就走暴力扫描，不用 HNSW
)

# 2. 给 category 字段建索引（payload index），让 filter 高效
client.create_payload_index(
    collection_name="docs",
    field_name="category",
    field_schema="keyword"
)

# 3. 写入
client.upsert(
    collection_name="docs",
    points=[
        PointStruct(id=i, vector=vec.tolist(), payload={"category": "shoes"})
        for i, vec in enumerate(vectors)
    ]
)

# 4. 查询 + 过滤
hits = client.search(
    collection_name="docs",
    query_vector=query_vec.tolist(),
    query_filter=Filter(must=[FieldCondition(key="category", match=MatchValue(value="shoes"))]),
    limit=10,
    search_params={"hnsw_ef": 128}
)
```

**`full_scan_threshold` 是个被忽视的关键参数**：当 filter 之后剩余候选不足这个数时，Qdrant 直接暴力扫描，避免 HNSW 退化。

适用场景：**中等规模（百万到亿）+ 需要复杂过滤 + 团队不想运维分布式系统**。

> **口诀**：**Qdrant 是 Rust 时代的实用主义选择**。单机能扛 1 亿向量、内置 sharding，运维成本远低于 Milvus。

### 2.3 Weaviate —— 对象 + 向量的"图数据库味"

Weaviate 把向量和对象（带 schema 的结构化数据）放在一起，支持 GraphQL 查询。

```python
# Weaviate v4 客户端
import weaviate
from weaviate.classes.config import Configure, Property, DataType

client = weaviate.connect_to_local()

client.collections.create(
    name="Docs",
    properties=[
        Property(name="text", data_type=DataType.TEXT),
        Property(name="category", data_type=DataType.TEXT),
    ],
    vectorizer_config=Configure.Vectorizer.text2vec_openai(),  # 自动调 OpenAI
    vector_index_config=Configure.VectorIndex.hnsw(
        ef_construction=200,
        max_connections=16,
    )
)

# 写入时不用自己算向量
docs = client.collections.get("Docs")
docs.data.insert({"text": "...", "category": "shoes"})

# 查询
result = docs.query.near_text(query="comfortable shoes", limit=10)
```

**特色**：内置 `text2vec` modules 自动调 embedding 模型，对原型开发友好。

**坑**：自动 vectorizer 在生产场景反而是负担 —— embedding 计算和存储混在一起，无法独立扩展。大规模场景几乎都关掉自动 vectorizer，自己上传向量。

适用场景：**快速原型 + 多模态（图文混合）+ 需要 GraphQL**。

### 2.4 Pinecone —— 全托管的"AWS RDS"

Pinecone 是闭源 SaaS，没法本地部署。优势是**零运维 + 弹性扩缩容 + 全球多区域**。

```python
from pinecone import Pinecone, ServerlessSpec

pc = Pinecone(api_key="...")

pc.create_index(
    name="docs",
    dimension=768,
    metric="cosine",
    spec=ServerlessSpec(cloud="aws", region="us-east-1")
)

index = pc.Index("docs")

# 写入
index.upsert(vectors=[
    {"id": "1", "values": vec.tolist(), "metadata": {"category": "shoes"}}
])

# 查询
results = index.query(
    vector=query_vec.tolist(),
    top_k=10,
    filter={"category": {"$eq": "shoes"}},
    include_metadata=True
)
```

**成本陷阱**：Serverless 模式按存储 + 读写计费，**对高 QPS 业务可能比自建贵 10 倍以上**。1 亿向量（768 维）每月存储费就 ~$700。

适用场景：**早期项目 + 团队没有 SRE + 数据量不大**。

### 2.5 pgvector —— 在 PostgreSQL 里加向量字段

pgvector 是 PostgreSQL 的扩展，2021 年由 Andrew Kane 开源，2024 年随着 RAG 火爆突然爆红。

最大优势：**复用 Postgres 的事务、备份、复制、权限**。中小规模下没有比这更省心的方案。

```sql
-- 安装
CREATE EXTENSION vector;

-- 建表
CREATE TABLE docs (
    id        BIGSERIAL PRIMARY KEY,
    text      TEXT,
    category  TEXT,
    embedding vector(768)
);

-- HNSW 索引（pgvector 0.5+ 支持）
CREATE INDEX ON docs USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 200);

-- 给 filter 字段建普通索引
CREATE INDEX ON docs (category);

-- 查询
SET hnsw.ef_search = 100;   -- 会话级调整
SELECT id, text, embedding <=> '[...]'::vector AS distance
FROM docs
WHERE category = 'shoes'
ORDER BY embedding <=> '[...]'::vector
LIMIT 10;
```

**距离运算符**：
- `<->` L2 距离
- `<=>` Cosine 距离
- `<#>` Inner Product（取负，因为 PG ORDER BY 是升序）

**致命限制**：单表向量数 > 1 亿后，HNSW 索引构建可能要几十小时，且占用大量 work_mem。pgvector 0.6 加入了**并行构建**，但仍比 Milvus / Qdrant 慢一个数量级。

> **口诀**：**< 1000 万向量首选 pgvector**；过 1 亿就该上专用库。

### 2.6 Elasticsearch 8 的 kNN —— 已有 ES 集群的"顺手做"

如果业务已经在用 ES 做全文检索，加上 dense_vector 几乎零成本：

```json
PUT docs
{
  "mappings": {
    "properties": {
      "text":      {"type": "text"},
      "category":  {"type": "keyword"},
      "embedding": {
        "type": "dense_vector",
        "dims": 768,
        "index": true,
        "similarity": "cosine",
        "index_options": {
          "type": "hnsw",
          "m": 16,
          "ef_construction": 200
        }
      }
    }
  }
}

GET docs/_search
{
  "knn": {
    "field": "embedding",
    "query_vector": [...],
    "k": 10,
    "num_candidates": 100,
    "filter": {"term": {"category": "shoes"}}
  },
  "rank": {
    "rrf": {"window_size": 50, "rank_constant": 60}
  }
}
```

**ES 的杀手级特性**：原生 RRF + 全文检索一体化。Hybrid Search 写起来最优雅。

**坑**：dense_vector 的内存占用是按 segment 算的，segment merge 后峰值可能 2x。生产要把 `indices.memory.index_buffer_size` 调高。

### 2.7 Redis Vector Set —— 内存型的"实时召回"

Redis Stack（含 RediSearch 模块）支持向量索引：

```python
import redis
from redis.commands.search.field import VectorField, TagField
from redis.commands.search.indexDefinition import IndexDefinition, IndexType
from redis.commands.search.query import Query
import numpy as np

r = redis.Redis(host='localhost', port=6379)

# 建索引
r.ft("docs").create_index(
    [
        VectorField("embedding", "HNSW", {
            "TYPE": "FLOAT32", "DIM": 768, "DISTANCE_METRIC": "COSINE",
            "M": 16, "EF_CONSTRUCTION": 200
        }),
        TagField("category"),
    ],
    definition=IndexDefinition(prefix=["doc:"], index_type=IndexType.HASH)
)

# 写入
r.hset("doc:1", mapping={
    "embedding": np.array(vec, dtype=np.float32).tobytes(),
    "category": "shoes"
})

# 查询
q = Query("(@category:{shoes})=>[KNN 10 @embedding $vec AS score]") \
    .sort_by("score").return_fields("score").dialect(2)
results = r.ft("docs").search(q, query_params={"vec": np.array(query_vec, dtype=np.float32).tobytes()})
```

**优势**：极低延迟（< 1ms 级），适合实时推荐。

**致命短板**：**全部数据必须放内存**。1 亿 768 维 = ~300GB 内存。RDB 持久化要小心，AOF 几乎没法用。适用场景非常窄。

> **口诀**：**Redis Vector 只用于热数据 + 低延迟召回**，永远不要把它当主向量库。

---

## 3. 实测数据：召回率、延迟、内存的真实对比

> 没数据的"性能更好"都是耍流氓。下面数据基于公开 ANN-Benchmarks 套件 + 自测，硬件 8 核 32G，数据集 SIFT1M（128 维）和 GIST1M（960 维）。

### 3.1 HNSW vs IVF 在不同数据规模的召回率

测试集：SIFT1M（100 万 × 128 维），target recall@10 = 0.95。

| 算法 | 配置 | 召回率 | QPS | 索引大小 |
| --- | --- | --- | --- | --- |
| HNSW | M=16, ef=100 | 0.96 | 8500 | 600 MB |
| HNSW | M=32, ef=200 | 0.99 | 4200 | 1.2 GB |
| IVF-Flat | nlist=1024, nprobe=32 | 0.92 | 3500 | 500 MB |
| IVF-Flat | nlist=1024, nprobe=128 | 0.97 | 1100 | 500 MB |
| IVF-PQ | nlist=1024, nprobe=64, m=8 | 0.83 | 4800 | **65 MB** |
| Brute Force | — | 1.00 | 12 | 500 MB |

**结论**：
- HNSW 在百万级是**速度 + 召回率**双赢
- IVF-PQ 内存压缩 8 倍但召回率掉到 0.83，**精度敏感场景慎用**
- HNSW 的 `ef` 是性价比最高的旋钮

### 3.2 不同向量库的查询延迟（P50 / P99）

100 万 × 768 维，QPS = 100，单分片，HNSW M=16, ef=100：

| 向量库 | P50 | P99 | 内存占用 |
| --- | --- | --- | --- |
| Qdrant | 4 ms | 12 ms | 5 GB |
| Milvus (Standalone) | 6 ms | 18 ms | 7 GB |
| pgvector 0.6 | 8 ms | 35 ms | 4 GB |
| Elasticsearch 8 | 12 ms | 42 ms | 9 GB |
| Weaviate | 9 ms | 28 ms | 6 GB |
| Pinecone (Serverless) | 25 ms | 80 ms | (托管) |

**反直觉的点**：
- pgvector 单机延迟其实很有竞争力
- Pinecone 看起来慢是因为**网络往返**（VPC 内甚至更慢，跨区域更明显）
- ES 拖后腿主要是 segment 多 + JVM 开销

### 3.3 索引构建时间对比

100 万 × 768 维，HNSW M=16, ef_construction=200：

| 向量库 | 构建时间 | 是否阻塞读 |
| --- | --- | --- |
| Qdrant | 4 min | 不阻塞 |
| Milvus | 5 min（IndexNode 异步） | 不阻塞 |
| pgvector（默认） | **42 min** | **阻塞写** |
| pgvector（CONCURRENTLY） | 50 min | 不阻塞 |
| pgvector 0.6 + parallel | 12 min | 不阻塞 |
| Elasticsearch | 8 min | 不阻塞 |

**pgvector 的构建慢是结构性的** —— 单进程 + 不能利用 IndexNode 异步分布。生产必须用 `CREATE INDEX CONCURRENTLY` 否则锁表。

### 3.4 内存占用的真相

768 维 × 1000 万向量 = 30 GB 原始数据。各方案加索引后的实际内存：

| 方案 | 总内存 | 索引开销倍数 |
| --- | --- | --- |
| HNSW (M=16) | 50 GB | 1.7x |
| HNSW (M=32) | 75 GB | 2.5x |
| IVF-Flat | 35 GB | 1.2x |
| IVF-PQ (m=8) | 5 GB | **0.17x** |
| Binary Quantization | 4 GB | **0.13x** |

**为什么 HNSW 这么吃内存**：每个节点存 M × 4 字节的邻居 ID，每层都要存。M=32 时光图结构就 1.3 GB / 1000 万节点。

### 3.5 Filter 对查询性能的影响

测试场景：1000 万向量，加一个 `WHERE tag = X` 过滤，命中率从 100% 到 0.1%：

| 命中率 | Qdrant (Filtered HNSW) | Milvus (partition) | pgvector (post-filter) |
| --- | --- | --- | --- |
| 100% | 5 ms | 7 ms | 9 ms |
| 50% | 6 ms | 8 ms | 11 ms |
| 10% | 9 ms | 10 ms | **45 ms**（K 不够，扩 K） |
| 1% | 12 ms | 14 ms | **210 ms**（多次扩 K） |
| 0.1% | 25 ms（退化为 plain scan） | 28 ms | **1500 ms**（基本退化为全表） |

**结论一目了然**：低过滤率场景 pgvector 极其难受，Qdrant 的 Filtered HNSW 是当前最优。

### 3.6 Hybrid Search 的召回率提升

测试集：MS MARCO（30 万 query，900 万 doc），目标 NDCG@10。

| 检索方式 | NDCG@10 |
| --- | --- |
| 纯 BM25 | 0.226 |
| 纯向量（BGE-M3） | 0.341 |
| BM25 + 向量（加权和 0.5） | 0.358 |
| BM25 + 向量（RRF k=60） | **0.376** |
| BM25 + 向量 + ColBERT 重排 | 0.412 |

**结论**：
- Hybrid Search 平均提升 5~10% 召回率
- RRF 比加权和稳定优于约 2 个点
- 加重排（reranker）还能再涨 4 个点，但延迟翻倍

> **口诀**：**生产 RAG 必上 Hybrid Search + RRF**，几乎零成本提升 5%+ 召回率。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① HNSW 调参错误导致召回率低 —— "上线后语义检索像随机推荐"

#### 现象

电商搜索接 RAG 后，用户反馈"搜咖啡杯出来一堆袜子"。监控看 P99 延迟正常（8ms），看上去索引一切健康。但 offline 评测召回率 recall@10 只有 65%，应该有 95%+。

#### 根因

排查发现工程师把 HNSW 配置成：

```python
HnswConfigDiff(m=4, ef_construct=20)   # 默认 m=16, ef_construct=128
```

为了"省内存"把 `M` 调到 4，`efConstruction` 调到 20。这两个参数**直接决定图的连通性**：
- `M = 4` 意味着每个节点最多 4 个邻居 —— 图变得稀疏，搜索时容易卡在局部连通分量
- `efConstruction = 20` 意味着构建时的候选队列只有 20 —— 选邻居时根本看不到真正最优的候选

更糟的是 `efSearch` 默认 32，因为 ef_construction 太小，**就算把 efSearch 调大也救不回来** —— 图本身的拓扑就不对。

HNSW 的本质是 **构建期决定上限，查询期只能逼近上限**。

#### 修复

```python
# 重建索引（HNSW 参数无法在线调整 M / ef_construct）
client.update_collection(
    collection_name="docs",
    hnsw_config=HnswConfigDiff(
        m=16,                 # 768 维文本向量推荐 16；高维 / 大集合用 32
        ef_construct=200,     # 构建期质量
    )
)
# 触发后台重建（Qdrant 自动）
```

线上调用时再加大 `efSearch`：

```python
client.search(..., search_params={"hnsw_ef": 128})
```

#### 验证

```python
# 用 ground truth（暴力计算的精确 top-10）评估召回率
def recall_at_k(predicted, ground_truth, k=10):
    pred_set = set(predicted[:k])
    gt_set = set(ground_truth[:k])
    return len(pred_set & gt_set) / len(gt_set)

# 抽 1000 条 query，对比 ANN 和 brute force
recalls = []
for query in test_queries:
    ann_result = client.search(..., limit=10)
    bf_result = brute_force_search(query, all_vectors)
    recalls.append(recall_at_k([h.id for h in ann_result], bf_result, k=10))
print(f"recall@10 = {np.mean(recalls):.4f}")
```

修复后从 0.65 涨到 0.97。

> **口诀**：**HNSW 三参数中 `M` 和 `efConstruction` 是构建期一锤子买卖**。生产环境永远不要为了"省内存"把它们调小，宁可后期 efSearch 加大。

---

### 案例 ② 大量 Filter 导致 ANN 退化为线性扫描 —— "加了 WHERE 后 P99 飙到 2 秒"

#### 现象

业务给查询加了 `WHERE user_id = X AND created_at > '2024-01-01'`。上线后单个 user 的查询 P99 从 12ms 飙到 2000ms。

#### 根因

策略是 Post-filter：

```python
# pgvector 的查询计划（伪 SQL）
SELECT id, embedding <=> :q AS dist FROM docs
WHERE user_id = 12345 AND created_at > '2024-01-01'
ORDER BY dist LIMIT 10;
```

PostgreSQL 优化器看到 `WHERE` 命中率约 0.01%（user_id 单值），**无法用 HNSW 索引** —— HNSW 不支持过滤，只能扫描。优化器选择走 `(user_id, created_at)` 复合索引扫出符合条件的行，然后**对每行算距离**。

如果某个 user 有 50 万条数据，等于 50 万次距离计算 + 排序。延迟必然爆炸。

更坑的是 EXPLAIN 看不出问题 —— 它只显示用了哪个索引，不会告诉你"距离计算 50 万次"。

#### 修复

**方案 A**：换 Qdrant 的 Filtered HNSW（生产推荐）

```python
client.create_payload_index(collection_name="docs", field_name="user_id", field_schema="integer")
client.create_payload_index(collection_name="docs", field_name="created_at", field_schema="datetime")

# 设 full_scan_threshold —— 过滤后剩这么多就走暴力
client.update_collection(
    collection_name="docs",
    hnsw_config=HnswConfigDiff(full_scan_threshold=10000)
)
```

Qdrant 会判断：过滤后候选 < 10000 就直接暴力（暴力 1 万条比 HNSW 退化扫描快），> 10000 走 Filtered HNSW。

**方案 B**：pgvector 用 partial index 拆维度

```sql
-- 给热点 user 单独建分区索引
CREATE INDEX ON docs USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 200)
WHERE user_id IN (101, 102, 103);  -- TOP-N 大客户单独优化
```

只对热点 user 建 HNSW partial index，覆盖到的 user 可以走索引。

**方案 C**：partition by user

Milvus 的 partition、pgvector 的表分区，把高频 filter 字段做物理分区。每个分区独立建 HNSW，查询直接路由到对应分区。

#### 验证

```sql
-- pgvector 查 EXPLAIN ANALYZE
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, embedding <=> '[...]'::vector AS dist
FROM docs
WHERE user_id = 12345
ORDER BY dist LIMIT 10;

-- 健康的输出应该看到 "Index Scan using docs_embedding_idx"
-- 病态输出会看到 "Seq Scan" 或 "Index Scan ... Filter (user_id = 12345)" + 大量 rows removed
```

> **口诀**：**Filter 不是免费的**，过滤命中率决定能不能走 ANN 索引。**< 10% 命中率必须用 Filtered HNSW 或物理分区**。

---

### 案例 ③ pgvector 单表过大导致重建索引慢 —— "凌晨重建索引锁表 6 小时"

#### 现象

业务表 `docs` 长到 5000 万行，每条带 768 维 embedding。embedding 模型升级（v1 → v2），需要重建整个表的 embedding。运维直接：

```sql
DROP INDEX docs_embedding_idx;
UPDATE docs SET embedding = new_embedding;  -- 已经异步算好
CREATE INDEX docs_embedding_idx ON docs USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 200);
```

`CREATE INDEX` 跑了 6 小时还在跑，期间整个表 `INSERT/UPDATE/DELETE` 全部阻塞。业务侧报警炸了。

#### 根因

pgvector 0.5 之前 HNSW 构建是**单线程**的。5000 万 × 768 维 ≈ 150 GB 数据，内存装不下，只能磁盘交换 —— 性能骤降。

更糟的是 `CREATE INDEX`（不带 CONCURRENTLY）在表上加 `ShareLock`，**阻塞所有写操作**。读不阻塞，但写直接挂起 6 小时。

#### 修复

**铁律**：pgvector 上**所有索引创建都必须 CONCURRENTLY**。

```sql
-- 不阻塞写
CREATE INDEX CONCURRENTLY docs_embedding_idx
ON docs USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 200);
```

代价：CONCURRENTLY 实际上慢一点（多次扫表），但不锁。

**升级到 pgvector 0.6+ 启用并行构建**：

```sql
SET max_parallel_maintenance_workers = 8;   -- 给构建用 8 个 worker
SET maintenance_work_mem = '8GB';           -- 索引构建专用内存

CREATE INDEX CONCURRENTLY docs_embedding_idx
ON docs USING hnsw (embedding vector_cosine_ops);
```

实测 5000 万 768 维数据：单线程 6 小时 → 并行 50 分钟。

**真正的大表方案**：拆表 / 分区

```sql
-- 按时间分区，新数据走新表，老数据冷冻
CREATE TABLE docs_2024 PARTITION OF docs FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
CREATE INDEX ON docs_2024 USING hnsw (embedding vector_cosine_ops);
```

每个分区独立建索引，分区粒度内并行。重建只影响一个分区。

#### 验证

```sql
-- 监控 CREATE INDEX 进度（PG 12+）
SELECT
    phase,
    blocks_done * 100.0 / NULLIF(blocks_total, 0) AS percent
FROM pg_stat_progress_create_index;

-- 看锁等待
SELECT pid, wait_event, query FROM pg_stat_activity WHERE wait_event LIKE '%Lock%';
```

> **口诀**：**pgvector 上 `CREATE INDEX` 永远要带 `CONCURRENTLY`**，pgvector 0.6+ 必上并行构建参数。超过 1 亿向量考虑分区。

---

### 案例 ④ Milvus 集群分裂 —— "QueryNode 看到的数据和 DataNode 不一致"

#### 现象

Milvus 集群上线 3 个月，某天起业务反馈"刚写入的数据查不到"。等 5 分钟、10 分钟还是查不到，但 `count(*)` 显示新数据已入库。

#### 根因

Milvus 的写入路径是**异步多阶段**：

```
ProxyNode → MQ (Kafka/Pulsar) → DataNode (落 S3) → IndexNode (建索引)
                                      │                │
                                      ▼                ▼
                               QueryNode 加载新 segment 才能查到
```

新数据走 MQ → DataNode → S3 → 通知 QueryNode 加载。任何一环堵住都会导致"写了查不到"。

排查发现：QueryCoord 的元数据里某个 collection 的 segment 加载状态卡在 "Loading"。深挖是 **MinIO 单节点磁盘满 80%**，新 segment 写不进去，QueryNode 拉取失败重试无限循环。

更大的坑：Milvus 2.x 早期版本（< 2.3.5）有个 bug —— **MinIO 磁盘错误不会冒到 Coordinator**，监控看一切正常，只有 QueryNode 内部日志能看到 IO error。

#### 修复

**短期**：扩 MinIO 容量 + 重启 QueryNode

```bash
# 扩容 MinIO PVC
kubectl edit pvc minio-data-0   # storage: 500Gi → 2Ti

# 等扩容完成后重启 QueryNode 触发重新加载
kubectl rollout restart statefulset milvus-querynode
```

**长期**：上对象存储监控 + 升级 Milvus

```yaml
# Prometheus 告警
- alert: MilvusObjectStorageFull
  expr: minio_disk_used_bytes / minio_disk_total_bytes > 0.7
  for: 10m

- alert: MilvusSegmentLoadStuck
  expr: increase(milvus_querycoord_loaded_collection_num[10m]) == 0
  for: 30m
```

升级 Milvus 到 2.4+，错误传播链路完善了很多。

#### 验证

```python
# 写入后用 flush 强制可见性
client.insert("docs", data)
client.flush(["docs"])  # 等 segment 落盘
client.load_collection("docs")  # 等 QueryNode 加载

# 验证可见性
results = client.search(...)
assert len(results) > 0
```

> **口诀**：**Milvus 是分布式系统，写入到可查有延迟**。生产必须监控 segment 加载状态、对象存储容量、消息队列堆积三个维度。

---

### 案例 ⑤ Sharding 后跨分片查询的延迟放大 —— "10 个分片比 1 个分片还慢"

#### 现象

为了应对增长，把 Qdrant collection 从 1 分片扩到 10 分片。期望 QPS 提升 10x，结果 P50 延迟从 5ms 涨到 25ms，QPS 反而下降。

#### 根因

向量检索的 sharding 不是普通数据库的 sharding。普通 OLTP 按 ID hash 分片，每个查询路由到一个分片。

但**向量查询是"全局 top-k"**：你要 top-10 最近邻，可能这 10 个分散在所有分片。所以 Qdrant 的查询逻辑是：

```
1. 把 query 广播到所有 N 个分片
2. 每个分片返回自己的 top-K（K 是请求的 limit）
3. 协调节点合并所有分片结果，返回全局 top-K
```

这意味着：
- **所有分片都要查一遍** —— 单查询的 CPU 总开销 = N × 单分片开销
- **延迟 = max(所有分片延迟) + 网络往返 + 合并开销**
- **QPS 上限 = 总 CPU / N**（每个查询占 N 倍 CPU）

10 个分片只有在**单个分片装不下、必须分片**时才有意义；如果数据能装单机，分片只会让事情变糟。

#### 修复

**正确的 sharding 时机**：单分片向量数 > 1 亿 或 单分片内存 > 64 GB 时才分。

**业务可以容忍按 tenant 分片的话**，用 `shard_key` 让查询定向到单分片：

```python
# Qdrant 支持 user-defined shard key
client.recreate_collection(
    collection_name="docs",
    vectors_config=VectorParams(size=768, distance=Distance.COSINE),
    sharding_method=models.ShardingMethod.CUSTOM,
)

# 写入时指定 shard
client.upsert(
    collection_name="docs",
    points=[...],
    shard_key_selector="tenant_42"
)

# 查询时也指定 shard，只查这一个分片
client.search(
    collection_name="docs",
    query_vector=query_vec,
    shard_key_selector="tenant_42",
    limit=10
)
```

**通用方案**：复制（replica）取代分片

```python
# Qdrant 单 collection 多副本，提升 QPS（不是分片）
client.update_collection(
    collection_name="docs",
    replication_factor=3   # 3 副本，QPS x3，每个副本完整数据
)
```

副本是 read-scale，分片是 storage-scale。**先副本扩 QPS，再分片扩存储**。

#### 验证

```python
# 对比实验：相同数据量 + 不同分片配置
import time
def benchmark(client, n_queries=1000):
    start = time.time()
    for _ in range(n_queries):
        client.search(...)
    return n_queries / (time.time() - start)

# 1 shard / 3 replica：QPS = 850
# 3 shard / 1 replica：QPS = 320
# 数据明确说明：能不分片就不分片
```

> **口诀**：**先 replica 后 shard**。向量查询的 sharding 是 fan-out 模式，分片数越多单查询越慢。

---

### 案例 ⑥ 向量维度过高（1536+）的存储成本爆炸 —— "embedding 模型升级后 S3 账单翻倍"

#### 现象

Embedding 模型从 BGE-base（768 维）升级到 OpenAI text-embedding-3-large（3072 维）。同样 1 亿条数据，存储从 300 GB 涨到 1.2 TB，S3 账单翻 4 倍。Milvus 集群内存压力也飙升。

#### 根因

存储成本 ≈ N × dim × 4（FP32）。维度翻 4 倍，存储就是线性翻 4 倍。HNSW 的图结构开销也是按维度算的（节点间距离计算）。

更深层：**3072 维的 embedding 比 768 维好多少**？OpenAI 自己的评测显示，在 MTEB benchmark 上 large 比 small 只好 ~2%。

为了 2% 的精度付 4x 成本 —— 大多数业务亏。

#### 修复

**方案 A**：降维 —— OpenAI v3 模型支持 Matryoshka Embedding

```python
# OpenAI text-embedding-3-large 默认 3072 维
# 但可以请求时指定较小维度（Matryoshka 训练让前 N 维就保留主要信息）
response = openai.embeddings.create(
    model="text-embedding-3-large",
    input="...",
    dimensions=1024   # 截断到 1024 维，精度只掉 ~1%
)
```

**方案 B**：量化（Quantization）

```python
# Qdrant 内置 scalar quantization（FP32 → INT8，4x 压缩）
client.update_collection(
    collection_name="docs",
    quantization_config=ScalarQuantization(
        scalar=ScalarQuantizationConfig(
            type=ScalarType.INT8,
            quantile=0.99,   # 截断分位点
            always_ram=True  # 量化向量常驻内存
        )
    )
)
# 实测：精度损失 < 2%，内存省 4 倍
```

更激进的 **Binary Quantization**（FP32 → 1 bit，32x 压缩）：

```python
client.update_collection(
    collection_name="docs",
    quantization_config=BinaryQuantization(
        binary=BinaryQuantizationConfig(always_ram=True)
    )
)
# 用于初筛，topK 候选再用原始向量重排
```

**方案 C**：拆分冷热

```python
# 热数据（最近 30 天）用全精度 + 内存
# 冷数据（30 天前）量化 + 磁盘（DiskANN / pgvector + ext4）
```

#### 验证

```python
# 量化前后召回率对比
def eval_recall(client, queries, ground_truth):
    recalls = []
    for q, gt in zip(queries, ground_truth):
        pred = client.search(query_vector=q, limit=10)
        recalls.append(len(set([p.id for p in pred]) & set(gt)) / 10)
    return np.mean(recalls)

# 关闭量化：recall = 0.97
# Scalar INT8: recall = 0.95
# Binary: recall = 0.86（仅做粗筛足够）
```

> **口诀**：**3072 维不是万能药**。Matryoshka 截断 + INT8 量化能在 < 2% 精度损失下省 4~16 倍内存。

---

### 案例 ⑦ 删除标记积累导致索引失效 —— "查询变慢 3 倍"

#### 现象

Qdrant collection 跑了半年，业务每天有大量 upsert（先 delete 再 insert）。查询延迟从 8ms 涨到 25ms。CPU 利用率没变，但每个查询访问的 segment 数从 5 涨到 80。

#### 根因

向量数据库的 delete 大多是**软删除（tombstone）**：标记一下，不真删。原因是 HNSW 图的结构紧密，真删一个节点要重建大块图，成本巨高。

```
原始 segment：[v1, v2, v3, v4, v5]
删除 v3：[v1, v2, v3(deleted), v4, v5]   ← 还在图里，查到再过滤
更新 v3 → v3'：[v1, v2, v3(deleted), v4, v5, v3']   ← 旧的留着，新的追加
```

时间长了：
- 实际数据 1000 万条，但 HNSW 图里有 3000 万节点（含 2000 万 tombstone）
- 每次查询走图都要遍历 tombstone，再过滤掉
- segment merge 不及时（资源不足或配置问题）

#### 修复

**方案 A**：触发 segment 合并（compaction）

```python
# Qdrant 强制 optimize
client.update_collection(
    collection_name="docs",
    optimizer_config=OptimizersConfigDiff(
        deleted_threshold=0.2,        # 删除比例超 20% 触发合并
        vacuum_min_vector_number=1000,
        default_segment_number=2,     # 控制 segment 数量
    )
)
```

**Milvus 类似**：

```python
client.compact(collection_name="docs")
# 等待
while client.get_compaction_state("docs") != CompactionState.COMPLETED:
    time.sleep(10)
```

**pgvector**：

```sql
-- VACUUM FULL 重建表 + 索引（阻塞，谨慎）
VACUUM FULL docs;

-- 或 REINDEX CONCURRENTLY 只重建索引（pg 12+）
REINDEX INDEX CONCURRENTLY docs_embedding_idx;
```

**方案 B**：监控 + 自动化

```yaml
# Prometheus 告警
- alert: QdrantTombstoneRatioHigh
  expr: qdrant_collection_vectors_deleted / qdrant_collection_vectors_total > 0.3
  for: 1h
```

#### 验证

```python
# 查 collection 状态
info = client.get_collection("docs")
print(f"vectors: {info.points_count}, segments: {info.segments_count}")

# segment 数应该 < 10（生产健康值）
# tombstone 比例 < 10%
```

> **口诀**：**软删除 + 不 compaction = 慢性中毒**。任何允许 delete 的向量库，必须配 compaction 监控和自动化。

---

### 案例 ⑧ Qdrant 的 quantization 对精度的影响 —— "省了内存但召回率掉了 5%"

#### 现象

为了省内存上了 Qdrant 的 Scalar Quantization。内存从 32GB 降到 9GB，业务上线后用户反馈"搜索结果质量下降"。offline 评测召回率从 0.95 掉到 0.89。

#### 根因

Scalar Quantization 把 FP32 浮点数映射到 INT8（256 个离散值）：

```python
# 简化的量化逻辑
def scalar_quantize(vec_fp32, vmin, vmax):
    """vmin / vmax: 校准时确定的范围"""
    scale = (vmax - vmin) / 255
    return ((vec_fp32 - vmin) / scale).round().clip(0, 255).astype(np.int8)
```

精度损失主要来自：
1. **浮点 → 整数的舍入误差**
2. **vmin/vmax 选择不当**：如果用全数据的 min/max，离群值会拉宽量程，让大部分值精度损失大
3. **量化后的距离计算近似** —— Qdrant 默认用量化向量算距离，没做"重排"

Qdrant 的 `quantile` 参数就是为问题 2 设计的：

```python
ScalarQuantizationConfig(
    type=ScalarType.INT8,
    quantile=0.99,   # 用 1% 和 99% 分位点而非 min/max，截断离群
)
```

#### 修复

**关键优化：开启 oversampling + rescore**

```python
client.search(
    collection_name="docs",
    query_vector=query_vec,
    limit=10,
    search_params=SearchParams(
        quantization=QuantizationSearchParams(
            ignore=False,
            rescore=True,        # ★ 用原始向量重排候选
            oversampling=2.0,    # ★ 量化先取 2x 候选，再重排
        )
    )
)
```

工作流：
1. 量化向量先取 20 个候选（10 × 2.0）
2. 对这 20 个用原始 FP32 向量算精确距离
3. 重排取 top-10

实测下来：召回率从 0.89 涨回 0.94，延迟只增加 ~10%。

**前提**：原始向量必须保留在内存或 SSD 上，否则 rescore 要去对象存储拉，延迟爆炸。

```python
# Qdrant 配置：原始向量存 SSD，量化向量常驻内存
ScalarQuantizationConfig(
    type=ScalarType.INT8,
    always_ram=True,   # 量化向量内存
)
# 原始向量自动落盘
```

#### 验证

```python
# 对比有无 rescore 的召回率
configs = [
    {"rescore": False, "oversampling": 1.0},
    {"rescore": True, "oversampling": 1.0},
    {"rescore": True, "oversampling": 2.0},
    {"rescore": True, "oversampling": 4.0},
]
for cfg in configs:
    recall = eval_recall(..., **cfg)
    print(f"{cfg}: recall = {recall:.4f}")

# 典型结果：
# rescore=False: 0.89
# rescore=True, oversampling=2.0: 0.94
# rescore=True, oversampling=4.0: 0.96 (延迟 +30%)
```

> **口诀**：**量化必须配 rescore + oversampling**，否则就是用召回率换内存。

---

### 案例 ⑨ 与 LangChain VectorStore 的兼容性 —— "换了向量库后召回完全不一样"

#### 现象

POC 用 Chroma 跑通了 RAG，迁移到生产 Qdrant，**同样的 query、同样的文档库、同样的 embedding 模型**，召回结果差异巨大。一开始怀疑数据丢了，核对后数据没问题。

#### 根因

LangChain 的 `VectorStore` 抽象掩盖了几个关键差异：

1. **距离度量不一致**：Chroma 默认 L2，Qdrant 默认 Cosine。同样的向量，排序结果不同。
2. **score 含义不同**：
   - Chroma 返回 `distance`（越小越相似）
   - Qdrant 返回 `score`（cosine 相似度，越大越相似）
   - LangChain 的 `similarity_search_with_score` 直接转发，**没有归一化**
3. **过滤语法不同**：Chroma 用 `where={"category": "shoes"}`，Qdrant 要 `Filter` 对象。LangChain 的 `filter` 参数实现不一致。

```python
# Chroma 默认行为
chroma_store = Chroma(embedding_function=embeddings, collection_name="docs")
results = chroma_store.similarity_search("query", k=10)

# Qdrant 默认行为 —— 距离度量必须显式指定！
from langchain_qdrant import QdrantVectorStore
qdrant_store = QdrantVectorStore(
    client=client,
    collection_name="docs",
    embedding=embeddings,
    distance=Distance.COSINE,   # 必须和模型对应（OpenAI / BGE 用 cosine）
)
```

#### 修复

**铁律**：跨向量库迁移时必须重新校准。

```python
# 1. 校验距离度量
print(f"embedding model: {embeddings.model}")
# OpenAI / BGE / Cohere 都是 cosine
# CLIP 是 cosine
# 自训练模型看你训练时的 loss

# 2. 校验数据完整性
chroma_count = chroma_store._collection.count()
qdrant_count = client.count("docs").count
assert chroma_count == qdrant_count

# 3. 抽样对比
sample_queries = ["query1", "query2", ...]
for q in sample_queries:
    chroma_results = chroma_store.similarity_search(q, k=10)
    qdrant_results = qdrant_store.similarity_search(q, k=10)
    overlap = len(set([r.metadata['id'] for r in chroma_results])
                  & set([r.metadata['id'] for r in qdrant_results]))
    print(f"query: {q}, overlap: {overlap}/10")
# 期望 overlap > 8/10，差异主要是边界候选
```

**通用迁移检查清单**：
- [ ] 距离度量一致
- [ ] 数据条数一致
- [ ] 抽样查询 top-10 重合度 > 80%
- [ ] 元数据字段类型一致（int vs string 不同处理）
- [ ] HNSW 参数（M / ef）一致

#### 验证

```python
# 端到端 A/B：用 ground truth 评估两个向量库
def evaluate_store(store, test_set):
    recalls = []
    for query, expected_ids in test_set:
        results = store.similarity_search(query, k=10)
        retrieved_ids = [r.metadata['id'] for r in results]
        recalls.append(len(set(retrieved_ids) & set(expected_ids)) / len(expected_ids))
    return np.mean(recalls)

print(f"Chroma recall: {evaluate_store(chroma_store, test_set)}")
print(f"Qdrant recall: {evaluate_store(qdrant_store, test_set)}")
# 差距应该 < 2 个百分点，超过说明配置有问题
```

> **口诀**：**LangChain VectorStore 是泄漏抽象**。换库前必须校验距离度量、score 含义、filter 语法三件套。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 选型层
- [ ] 数据规模评估清楚（百万 / 亿 / 十亿不同选型）
- [ ] 距离度量与 embedding 模型匹配（OpenAI/BGE = cosine，CLIP = cosine）
- [ ] 过滤场景的命中率分布做了估算（决定是否需要 Filtered HNSW / partition）
- [ ] 业务能否容忍最终一致性（Milvus / 分布式向量库都有写入延迟）
- [ ] 是否需要 Hybrid Search（80%+ RAG 业务都该上）
- [ ] 团队 SRE 能力评估（决定 Pinecone / pgvector / Milvus 的取舍）

### 索引层
- [ ] HNSW 参数：`M = 16~32`，`efConstruction >= 200`
- [ ] 不要为了"省内存"调小 M（构建期一锤子买卖，无法在线修改）
- [ ] `efSearch` 默认 50~100，按召回率压测调整
- [ ] pgvector 的 `CREATE INDEX` 必须 `CONCURRENTLY`
- [ ] pgvector 0.6+ 启用 `max_parallel_maintenance_workers`

### 性能层
- [ ] 召回率有 ground truth 评测（至少 1000 条 query 的标注集）
- [ ] P50 / P99 延迟监控
- [ ] 量化（Scalar / Binary）必须配 rescore + oversampling
- [ ] 跨分片 / 多 segment 的延迟放大效应有测试
- [ ] Filter 字段建对应的 payload index（Qdrant）/ scalar index（Milvus）

### 运维层
- [ ] 删除 / 更新比例监控（tombstone 比例 > 20% 触发 compaction）
- [ ] 索引构建任务异步 + 进度可观测
- [ ] 数据备份方案（pgvector 走 PG 标准，Milvus 走对象存储 + 元数据）
- [ ] 内存 / 磁盘水位告警（特别是 Milvus 依赖的对象存储）
- [ ] embedding 模型升级流程（rebuild 计划 + 灰度切换）

### 业务层
- [ ] LangChain / LlamaIndex 等抽象层换库时做兼容性测试
- [ ] 多租户场景用 partition / shard_key 隔离
- [ ] RAG 上线必配 reranker（Cohere / BGE-Reranker / ColBERT）

---

## 6. 还能往下挖什么

读完这 9 个案例只是起点。如果想再深一层：

### 向量量化的前沿
- **Matryoshka Representation Learning（MRL）** —— OpenAI v3 模型背后的训练方法。让 embedding 的前 N 维就能保留主要信息，支持任意截断。
- **Binary Quantization** —— 1 bit/维。精度损失大但速度快 32 倍，用于初筛 + 重排。
- **Product Quantization 改进** —— OPQ（Optimized PQ）、Additive Quantization，工业上 Faiss 都支持。

### DiskANN —— 单机十亿级向量
- 微软 2019 年论文，核心是 **Vamana 图 + SSD 存储**。索引在内存只放 PQ 压缩版本，原始向量按需从 SSD 读。
- 适用场景：数据量大到内存装不下，但又不想分布式。
- 实现：DiskANN（C++）、SPANN（微软）、Milvus 的 DISKANN 索引类型。

### GPU 加速 ANN
- **Faiss-GPU** —— Meta 出品，IVF / HNSW 都有 GPU 实现，吞吐 10x+ CPU。
- **NVIDIA RAFT / cuVS** —— CUDA 原生 ANN 库，最新一代 GPU 向量库底座。
- 适用场景：高 QPS（万级）的离线批量召回，不适合在线低延迟。

### 多模态向量
- **CLIP / SigLIP** —— 图文对比学习，同一空间表示图像和文本。
- **统一向量库** —— Weaviate / Milvus 都支持多种向量同表，但要小心维度和度量混用。
- 多模态检索的"模态偏置"问题：纯文本 query 可能更容易召回文本，要做模态平衡。

### Embedding 模型选型矩阵
- **OpenAI text-embedding-3** —— 通用，多语言强，成本中等
- **BGE-M3 / BGE-large** —— 智源开源，中文最强，免费
- **Cohere embed-v3** —— 多语言 + 长文本，付费
- **Jina Embeddings** —— 长文本（8K+），开源
- **专用领域**（代码、医疗、法律）有专门 finetune 模型

选型策略：先 BGE-M3 跑 baseline，业务关键场景再考虑 finetune。

### 与图数据库的协作
- **GraphRAG**（微软）—— 用 LLM 抽取实体关系建图，向量召回 + 图遍历组合。
- **Neo4j + 向量** —— Neo4j 5+ 内置向量索引，可以一个引擎做图 + 向量。
- **场景**：实体关系密集的领域知识库（金融、医疗）。

### 向量数据库的成本优化
- **冷热分层**：热数据全精度内存，冷数据量化 SSD
- **Embedding 缓存**：相同文本的 embedding 缓存，OpenAI 调用费降 80%
- **降维 + 量化组合**：3072 维 → Matryoshka 截 1024 → INT8 量化 = 12x 压缩
- **批量 vs 实时**：能批量算 embedding 就批量（OpenAI batch API 半价）

---

## 7. 引用

- Malkov & Yashunin, *Efficient and robust approximate nearest neighbor search using HNSW graphs* (TPAMI 2018) —— HNSW 原始论文
- Jégou et al., *Product Quantization for Nearest Neighbor Search* (TPAMI 2011) —— PQ 经典
- Subramanya et al., *DiskANN: Fast Accurate Billion-point Nearest Neighbor Search on a Single Node* (NeurIPS 2019)
- Cormack et al., *Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods* (SIGIR 2009)
- [Milvus 官方架构文档](https://milvus.io/docs/architecture_overview.md)
- [Qdrant Filtering Best Practices](https://qdrant.tech/articles/filtrable-hnsw/)
- [pgvector GitHub](https://github.com/pgvector/pgvector) —— Andrew Kane 仓库
- [ANN-Benchmarks](http://ann-benchmarks.com/) —— 公开基准测试，所有主流算法都在跑
- Kusupati et al., *Matryoshka Representation Learning* (NeurIPS 2022) —— OpenAI v3 背后的方法
- [MTEB Leaderboard](https://huggingface.co/spaces/mteb/leaderboard) —— embedding 模型评测榜

---

> 配套：[RAG 工程化](/article/ai/rag-engineering) | [Elasticsearch 深入](/article/db/elasticsearch) | [LLM 推理引擎](/article/ai/llm-inference)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
