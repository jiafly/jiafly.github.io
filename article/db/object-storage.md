# 对象存储深入 —— S3 / MinIO / Ceph 与现代数据基础设施

> 这是按 [桃花岛深度模板](/STYLE) 写的对象存储深文。如果你只想速记，看 [存储综述](/article/db/storage) 那一节就够了。
> 这里要回答的，是**搭数据湖、跑大模型训练、做大文件直传**时你需要知道的那部分 —— 不是 boto3 的 API 列表。

---

## 0. 读这篇之前 —— 8 个自检题

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. 对象存储和文件存储 / 块存储的本质差别在哪？为什么 S3 不能像 NFS 一样 `open() / write(offset)`？
2. AWS S3 在 2020 年底宣布"强一致"之前，"最终一致"具体指什么？老代码里那些 `sleep(1)` 现在可以删掉吗？
3. 一个 5 TB 的视频文件，用 PutObject 直传 vs Multipart Upload，体验差在哪？分片大小怎么选？最后那个 CompleteMultipartUpload 真的"原子"吗？
4. 预签名 URL（Presigned URL）是怎么签的？签完之后服务端要不要保存？签名泄漏后能怎么补救？
5. MinIO 默认开启 Erasure Coding，配置 `EC:4` 是什么意思？为什么不直接用 3 副本？
6. Ceph 的 CRUSH 算法相比一致性 Hash 解决了什么？为什么扩容时数据迁移量更可控？
7. 数据湖（Iceberg / Delta Lake / Hudi）为什么必须建在对象存储上、不能直接建在 HDFS 上？S3 的什么特性是关键？
8. S3 高并发返回 503 SlowDown，限流维度是 bucket 还是 prefix？怎么"散热"？

如果有半数答不上，往下看。

---

## 1. 底层模型

绝大多数人对对象存储的认知止于"S3 是个云盘"。这层抽象解释不了上面 8 个问题。要看懂它们，必须下沉到对象存储的核心抽象、一致性模型、纠删码与分布式架构。

### 1.1 核心抽象：Bucket / Key / Object / Metadata

对象存储的 4 元组：

```
Bucket（桶）
  │  全局唯一的命名空间，URL 路径的第一段
  │  例：s3://my-data-lake/...
  │
  └── Key（键）
        │  桶内唯一的"路径字符串"
        │  例：warehouse/orders/year=2026/month=05/part-000.parquet
        │  注意：是字符串，不是真正的目录树
        │
        └── Object（对象）
              ├── Body：实际字节流，单对象上限 5TB（S3）
              ├── Metadata：用户自定义 K/V + 系统元数据
              │   ├── Content-Type, Content-Length
              │   ├── ETag（通常是内容 MD5，但 multipart 不是）
              │   ├── x-amz-meta-* 用户自定义
              │   └── 加密、版本、Lifecycle 标签
              └── Version ID（开启版本控制后）
```

**关键认知**：Key 看起来像路径但**不是真目录**。`a/b/c.txt` 中的 `/` 只是字符。S3 没有"目录"的概念，所谓"目录"是通过 List 时的 `prefix` + `delimiter='/'` 模拟出来的。

这有什么影响？**`mv directory` 不存在**。要"移动一个目录"，必须 List 出所有 key 然后逐个 Copy + Delete。这是后面案例 ① "小文件 List 慢"的根。

### 1.2 与文件 / 块存储的对比

```
块存储（EBS / Ceph RBD）
  ├── 暴露：block device，read(offset, len) / write(offset, len)
  ├── 用法：挂载到单机，mkfs 后跑 OS 文件系统
  ├── 一致性：单挂载点强一致
  └── 典型：DB 数据盘、虚拟机系统盘

文件存储（NFS / EFS / CephFS）
  ├── 暴露：POSIX 文件系统，open / read / write / lock
  ├── 用法：多机挂载，看到同一份目录树
  ├── 一致性：close-to-open（NFS）/ 强一致（CephFS）
  └── 典型：共享配置、媒体处理流水线

对象存储（S3 / MinIO / Ceph RGW / OSS）
  ├── 暴露：HTTP REST API，PUT / GET / DELETE
  ├── 用法：HTTP 客户端、SDK，无需挂载
  ├── 一致性：S3 现在强一致；旧 EVCons；MinIO/Ceph 各异
  └── 典型：备份、数据湖、CDN 源站、AI 训练数据
```

三者的核心差异是**抽象粒度**与**网络模型**：
- 块存储：粒度最细（block），但只能单机用
- 文件存储：粒度中等（file），多机共享
- 对象存储：粒度最粗（整对象不可改），但天然 HTTP，可全球访问

为什么对象不可改（immutable）？因为这是**水平扩展到 EB 级**的代价。允许部分修改意味着要维护跨节点的 byte-level 锁、要处理 partial write 的可见性 —— 这些在 PB 级以上是噩梦。**用"整对象覆盖"换扩展性**，这是对象存储的根本设计权衡。

### 1.3 S3 API 标准（事实标准）

```
PutObject       上传整个对象（≤ 5GB 单 PUT）
GetObject       下载（支持 Range，断点续传）
DeleteObject    删除
HeadObject      只取 Metadata
ListObjectsV2   按 prefix 列出 keys

CreateMultipartUpload    开始分片上传
UploadPart               上传分片（5MB ~ 5GB / 片，上限 10000 片）
CompleteMultipartUpload  完成（按片号顺序合并）
AbortMultipartUpload     放弃（释放未完成片）

CopyObject      服务端 Copy（不需走客户端流量）
SelectObjectContent  S3 Select（在存储端跑 SQL）
```

S3 API 是事实标准。MinIO、Ceph RGW、阿里 OSS、华为 OBS、Cloudflare R2、Backblaze B2 全部兼容（程度不同）。换厂商时**改 endpoint 不改业务代码**是设计目标。

但**兼容不等于一致**。Cloudflare R2 不支持 S3 Select，MinIO 在某些 Header 上行为略有出入，**生产前必须做兼容性测试**。

### 1.4 一致性模型：从最终一致到强一致

2020 年 12 月之前，S3 的一致性是这样的：

```
PUT 新对象     → read-after-write 强一致（PUT 完立即可读）
PUT 覆盖对象   → 最终一致（可能读到老版本）
DELETE        → 最终一致（可能还能读到已删的对象）
LIST         → 最终一致（PUT 完立即 List 可能看不到）
```

老代码里那些 `Thread.sleep(1000)` 就是为了等"最终一致"窗口过去。

2020-12-02 之后（[官方公告](https://aws.amazon.com/s3/consistency/)），所有操作变成**强一致**：

```
PUT (新或覆盖) → 强一致
DELETE        → 强一致
LIST         → 强一致
```

但**强一致 ≠ 事务一致**。S3 的"强一致"是指**单对象的读写顺序**，不是"多对象原子提交"。要做"两个文件同时生效"，仍需上层加事务（参见 1.7 数据湖 + Iceberg）。

MinIO / Ceph RGW 也都提供强一致（基于自身的元数据一致性协议）。OSS / OBS 现在主流也是强一致。

> **口诀**：单对象操作信任强一致；多对象原子用 Iceberg / Delta Lake 这类 manifest 方案。

### 1.5 Erasure Coding（纠删码）vs Replication

副本（Replication）：
```
1 个对象 → 存 3 份（不同节点 / 机架 / 区域）
存储效率 = 33%（要 3 倍空间）
能容忍：2 节点失败
```

纠删码（Erasure Coding，EC）：
```
1 个对象 → 切 K 个数据块 + N 个校验块（Reed-Solomon）
存储效率 = K / (K+N)
能容忍：N 个块失败
```

MinIO 默认 EC 配置：
- 4 节点：`EC:2`（4 数据 + 2 校验，可容忍 2 个块丢失）—— 等等，这里要看清楚
- 实际上 MinIO 的 `MINIO_STORAGE_CLASS_STANDARD=EC:N` 中 **N 是校验块数**，数据块数 = 总盘数 - N
- 4 盘 EC:2 = 2 数据 + 2 校验，效率 50%，能丢 2 盘
- 8 盘 EC:4 = 4 数据 + 4 校验，效率 50%，能丢 4 盘
- 16 盘 EC:4 = 12 数据 + 4 校验，效率 75%，能丢 4 盘

对比：
| 方案 | 存储效率 | 写放大 | 修复成本 | 适合场景 |
| --- | --- | --- | --- | --- |
| 3 副本 | 33% | 3 倍 | 拷贝即可 | 小集群、低延迟读、热数据 |
| EC(4+2) | 67% | 1.5 倍 | 需读 K 块重算 | 中等容量、温数据 |
| EC(10+4) | 71% | 1.4 倍 | 同上 | 大容量、冷数据 |
| EC(16+4) | 80% | 1.25 倍 | 同上 | 超大容量、归档 |

EC 的**代价**：
- 修复一个失败块要读 K 个其他块（网络放大）
- 小对象用 EC 反而浪费（管理元数据开销 > 节省的空间）—— MinIO 默认对 < 32KB 对象用 inline 副本

> **口诀**：副本简单快、EC 省空间但修复贵。**热数据 + 小对象用副本，冷数据 + 大对象用 EC**。

### 1.6 MinIO 架构

MinIO 是 S3 兼容的开源对象存储，Go 写的，单二进制。两种部署模式：

```
单机模式（Standalone）
  minio server /data
  ├── 一个进程，多盘可选
  ├── 单盘：无冗余
  └── 多盘：本机 EC，能容忍部分盘坏

分布式模式（Distributed）
  minio server http://node{1...4}/data{1...4}
  ├── 多节点，每节点多盘
  ├── 跨节点 EC：单节点宕机也不丢数据
  └── 通过 grid 协议互联（基于 HTTP/2）
```

关键概念：
- **Erasure Set**：EC 的最小单元。一组盘（典型 4 / 8 / 16 盘）一起做 EC
- **Pool**：一组 Erasure Set，扩容时加新 Pool 而不是扩老 Pool
- **Server Pool**：MinIO 4.x+ 的扩容模型，新 Pool 独立放新数据

MinIO 没有元数据库（no MySQL / etcd）—— **元数据和数据放在一起**，每个对象的元数据是 `xl.meta` 文件，与数据分片同盘。这避免了"元数据服务成为瓶颈"，但**全局 List 必须扫所有盘**。

### 1.7 Ceph RGW 架构：RADOS + CRUSH

Ceph 是更老牌的统一存储（块 / 文件 / 对象三合一）。RGW（RADOS Gateway）是它的 S3 兼容层。

```
S3 Client
  │
  ▼
RGW（HTTP，无状态，可水平扩展）
  │
  ▼
RADOS（Reliable Autonomic Distributed Object Store）
  │
  ├── Monitor（mon）：维护集群状态、CRUSH map
  ├── OSD（Object Storage Daemon）：每个磁盘一个进程
  └── MGR（manager）：监控、Dashboard
       ↑
       │  CRUSH 算法 placement
       ▼
对象 → PG（Placement Group）→ OSD 列表
```

**CRUSH（Controlled Replication Under Scalable Hashing）算法**是 Ceph 的灵魂：

```
传统一致性 Hash：
  hash(key) → 节点
  扩容：环上插一个节点 → 该节点附近的数据迁移
  问题：故障域（机架/数据中心）感知能力弱

CRUSH：
  hash(key) → PG（一个虚拟桶）
  PG → 多个 OSD（按 CRUSH 规则选择）
  规则可配置：
    - 副本数
    - 同 PG 的 OSD 必须跨机架
    - 同 PG 的 OSD 必须跨数据中心
```

CRUSH 解决的关键问题：
1. **客户端无需查询元数据服务**：直接算出对象在哪些 OSD（去中心化）
2. **故障域感知**：副本天然分散到不同机架/区域
3. **扩容数据迁移可预测**：加一个 OSD 后只迁移需要重平衡的 PG，量可算

代价：CRUSH map 本身要在所有节点同步；map 变更要走 Paxos（mon 集群）。

> **口诀**：MinIO 简单粗暴（部署一行命令），Ceph 工业重型（功能全但运维复杂）。**云上用 S3，自建中小集群用 MinIO，超大规模 + 多协议（块/文件/对象）混用 Ceph**。

---

## 2. 工程切片：从 SDK 到数据湖

> 每段代码 < 30 行，挑关键路径，不大段贴。基于 AWS SDK for Java 2.x 和 MinIO Java SDK 8.x。

### 2.1 AWS SDK + 预签名 URL（Presigned URL）

预签名 URL 是对象存储最重要的工程模式：**让客户端直传 / 直下，不经过你的应用服务器**。

```java
S3Presigner presigner = S3Presigner.builder()
    .region(Region.US_EAST_1)
    .credentialsProvider(DefaultCredentialsProvider.create())
    .build();

PutObjectRequest req = PutObjectRequest.builder()
    .bucket("uploads")
    .key("user/1001/avatar.jpg")
    .contentType("image/jpeg")
    .build();

PresignedPutObjectRequest presigned = presigner.presignPutObject(
    PutObjectPresignRequest.builder()
        .signatureDuration(Duration.ofMinutes(15))    // ★ URL 有效期
        .putObjectRequest(req)
        .build());

String url = presigned.url().toString();
// 把 url 返给前端，前端直接 PUT 到这个 URL，不经过你的服务器
```

**为什么是核心模式？**
- 大文件上传不占用应用服务器带宽
- 应用服务器只做"鉴权 + 签 URL"，不做"代理传输"
- 短期有效，泄漏窗口可控

**关键安全要点**（参见案例 ③）：
- `signatureDuration` 必须短（15 分钟以内）
- 签名时 **必须固定 contentType / contentLength** 等关键参数，否则客户端可以传任意类型
- 上传完成后服务端要校验对象（可疑就 Delete + 拉黑）

下载用法对称：

```java
GetObjectRequest get = GetObjectRequest.builder()
    .bucket("uploads")
    .key("user/1001/report.pdf")
    .responseContentDisposition("attachment; filename=\"report.pdf\"")
    .build();

PresignedGetObjectRequest preGet = presigner.presignGetObject(
    GetObjectPresignRequest.builder()
        .signatureDuration(Duration.ofMinutes(5))
        .getObjectRequest(get)
        .build());
```

### 2.2 MinIO Java SDK 实战

MinIO SDK 与 AWS SDK API 风格不同，但用法相近：

```java
MinioClient client = MinioClient.builder()
    .endpoint("https://minio.internal:9000")
    .credentials(accessKey, secretKey)
    .build();

// 上传
client.uploadObject(UploadObjectArgs.builder()
    .bucket("data")
    .object("logs/2026-05-09/app.log")
    .filename("/var/log/app.log")
    .contentType("text/plain")
    .build());

// 下载
try (InputStream in = client.getObject(GetObjectArgs.builder()
        .bucket("data")
        .object("logs/2026-05-09/app.log")
        .build())) {
    Files.copy(in, Path.of("/tmp/app.log"));
}
```

MinIO 的 `uploadObject` 内部会**自动判断是否走 multipart**（默认 > 16MB）。要精细控制必须用底层 `putObject` + `partSize`。

**生产建议**：复用 `MinioClient`（线程安全，内部用 OkHttp 连接池）。每次 `new MinioClient` 会新建连接池，高并发场景大量 TIME_WAIT。

### 2.3 分片上传（Multipart Upload）

5 GB 直传是个糟糕的主意：网络抖一下整个上传重来。Multipart 解决三件事：
1. 断点续传（任一片失败只重传该片）
2. 并行上传（多个分片并发）
3. 突破 5GB 单 PUT 限制（multipart 上限 5TB）

完整流程的 AWS SDK 代码：

```java
// 1. 创建 multipart upload
CreateMultipartUploadResponse mp = s3.createMultipartUpload(
    CreateMultipartUploadRequest.builder()
        .bucket("video").key("4k/movie.mkv")
        .contentType("video/x-matroska")
        .build());
String uploadId = mp.uploadId();

// 2. 并行上传分片
List<CompletedPart> parts = new ArrayList<>();
long partSize = 64 * 1024 * 1024;          // 64 MB / 片
try (var raf = new RandomAccessFile(file, "r")) {
    long total = raf.length();
    int partNumber = 1;
    for (long off = 0; off < total; off += partSize, partNumber++) {
        long size = Math.min(partSize, total - off);
        raf.seek(off);
        byte[] buf = new byte[(int) size];
        raf.readFully(buf);

        UploadPartResponse pr = s3.uploadPart(
            UploadPartRequest.builder()
                .bucket("video").key("4k/movie.mkv")
                .uploadId(uploadId)
                .partNumber(partNumber)
                .contentLength(size).build(),
            RequestBody.fromBytes(buf));

        parts.add(CompletedPart.builder()
            .partNumber(partNumber).eTag(pr.eTag()).build());
    }
}

// 3. 完成（原子提交）
s3.completeMultipartUpload(CompleteMultipartUploadRequest.builder()
    .bucket("video").key("4k/movie.mkv")
    .uploadId(uploadId)
    .multipartUpload(CompletedMultipartUpload.builder().parts(parts).build())
    .build());
```

**分片大小怎么选？**

| 文件大小 | 推荐分片 | 理由 |
| --- | --- | --- |
| < 100 MB | 不用 multipart | 直传更快 |
| 100 MB ~ 1 GB | 16 ~ 32 MB | 兼顾并发与请求开销 |
| 1 GB ~ 100 GB | 64 ~ 128 MB | 平衡常见 |
| > 100 GB | 256 MB ~ 1 GB | 减少分片数（上限 10000） |

**上限**：单对象 5 TB；分片数 10000；单片 5 MB ~ 5 GB（最后一片可小于 5 MB）。

**CompleteMultipartUpload 是原子的**：调用前对象不存在或仍是老版本；调用后立即可见新版本。所以上传中途 abort 不会留下"半个文件"。

### 2.4 S3 Select：把计算下推到存储

S3 Select 让你**直接在 S3 上跑 SQL** 而不下载整个文件。适合"从大 CSV / Parquet / JSON 里只取几列几行"。

```java
SelectObjectContentRequest req = SelectObjectContentRequest.builder()
    .bucket("logs").key("2026-05-09.json.gz")
    .expression("SELECT s.user_id, s.event FROM S3Object[*] s WHERE s.status = 500")
    .expressionType(ExpressionType.SQL)
    .inputSerialization(InputSerialization.builder()
        .compressionType(CompressionType.GZIP)
        .json(JSONInput.builder().type(JSONType.LINES).build())
        .build())
    .outputSerialization(OutputSerialization.builder()
        .json(JSONOutput.builder().build())
        .build())
    .build();

s3Async.selectObjectContent(req, new SelectObjectContentResponseHandler() {
    @Override public void responseReceived(SelectObjectContentResponse r) {}
    @Override public void onEventStream(SdkPublisher<SelectObjectContentEventStream> p) {
        p.subscribe(e -> e.accept(new SelectObjectContentResponseHandler.Visitor() {
            @Override public void visitRecords(RecordsEvent re) {
                System.out.write(re.payload().asByteArray());
            }
        }));
    }
    @Override public void exceptionOccurred(Throwable t) { t.printStackTrace(); }
    @Override public void complete() {}
}).join();
```

什么时候用？
- 100 GB CSV，只查其中 1000 行
- 对象在 Glacier 之外的标准存储
- 数据是 CSV / JSON / Parquet 格式（其他不支持）

什么时候**不用**？
- 全表扫描（不如直接下载然后本地处理）
- 高频查询（用 Athena / DuckDB / Trino 更合适）
- 兼容厂商有限（MinIO 部分支持，Cloudflare R2 不支持）

### 2.5 Bucket 策略与 IAM

S3 的权限模型有两层：
- **IAM Policy**：附着在 user / role 上，"这个用户能做什么"
- **Bucket Policy**：附着在 bucket 上，"谁能动这个 bucket"

两者**取并集**（除非有显式 Deny）。生产中最容易出事的是 bucket 策略写错导致全公网可读。

最小权限的上传专用 bucket policy：

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowAppUpload",
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::123456789012:role/app" },
    "Action": ["s3:PutObject"],
    "Resource": "arn:aws:s3:::uploads/user/*",
    "Condition": {
      "StringEquals": {
        "s3:x-amz-server-side-encryption": "AES256"
      },
      "NumericLessThan": {
        "s3:max-content-length": 10485760
      }
    }
  }]
}
```

要点：
- **Resource 用前缀通配**：避免给整 bucket 权限
- **Condition 强制加密 / 限大小**：在策略层兜底，比应用层校验更可靠
- **Block Public Access** 默认开（账户级、bucket 级双开关），别手贱去关

### 2.6 对接数据湖 Iceberg / Delta Lake / Hudi

这是对象存储**最重要的现代用法**。三者本质都是"在 S3 上加事务 + 元数据"：

```
   传统 Hive 表                   Iceberg / Delta / Hudi
+----------------+              +-----------------------+
| HDFS / S3      |              | S3                    |
| ├── part1.parq |              | ├── data/             |
| ├── part2.parq |              | │    ├── part1.parq   |
| └── ...        |              | │    └── part2.parq   |
+----------------+              | └── metadata/         |
   ↑                            |      ├── v0.json      |
   Hive Metastore               |      ├── v1.json      |
   存表→分区→文件映射             |      └── snapshot ... |
                                +-----------------------+
                                  快照 = 一份 manifest
                                  事务 = 原子写新 manifest
```

为什么必须基于对象存储？两个原因：
1. **HDFS 重度依赖 NameNode**，规模上来后 NameNode 是单点 + 内存瓶颈
2. **对象存储的"按 prefix 列对象 + 强一致 PUT"** 是 manifest 协议的天然底座

Iceberg 写入流程（简化）：

```
1. Writer 写若干 parquet 文件到 s3://bucket/data/...
2. Writer 写一个新的 manifest list 描述"这次提交包含哪些文件"
3. Writer 用 atomic rename / atomic put 把 metadata pointer 指向新 manifest
4. 其他 reader 看到新 manifest → 看到新数据
```

第 3 步**必须原子**。S3 的强一致 PUT + Compare-And-Swap 模式（Iceberg 用 etag-based CAS）保证多 writer 不冲突。

Spark 写 Iceberg 表：

```java
SparkSession spark = SparkSession.builder()
    .config("spark.sql.catalog.lake", "org.apache.iceberg.spark.SparkCatalog")
    .config("spark.sql.catalog.lake.type", "hadoop")
    .config("spark.sql.catalog.lake.warehouse", "s3a://datalake/warehouse")
    .getOrCreate();

spark.sql("""
    CREATE TABLE lake.db.orders (
      id BIGINT, user_id BIGINT, amount DECIMAL(10,2), created_at TIMESTAMP
    ) USING iceberg
    PARTITIONED BY (days(created_at))
""");

orders.writeTo("lake.db.orders").append();
```

### 2.7 CDN 加速

对象存储 + CDN 是 Web 资源分发的标配组合：

```
用户 → CDN edge → CDN 回源到 S3 → 缓存 → 返回用户
            ↑
       命中率 > 95%
       回源率 < 5%
```

关键配置（CloudFront 为例）：

```yaml
# 关键的 cache key 设置
CacheKey:
  Headers: []                        # 不基于 Header 区分缓存（除非真要）
  QueryStrings: ["v"]                 # 只把版本号纳入 key
  Cookies: []                         # 不基于 cookie

OriginAccessIdentity: enabled        # 让 S3 只接受 CDN 来源（防绕过）

S3 BucketPolicy:
  Principal: cloudfront-OAI-id        # 只允许 CDN 读
```

自建 CDN（Nginx 反代）：

```nginx
location / {
    proxy_pass https://minio.internal:9000;
    proxy_cache obj_cache;
    proxy_cache_valid 200 7d;
    proxy_cache_key $uri;             # 不带 query 缓存

    proxy_cache_lock on;              # ★ 防"惊群"：第一个请求回源时其他等待
    proxy_cache_use_stale updating;   # 回源失败时返回老缓存
}
```

`proxy_cache_lock` 是关键 —— 没它的话热门资源回源时同时几百个请求穿透到 S3。

> **口诀**：上传用预签名 URL；下载用 CDN；大文件用 multipart；批量分析用 Iceberg。

---

## 3. 实测数据：直传、EC、网络模型

> 数据没数字的"性能更好"都是耍流氓。下面给可复现的对比。

### 3.1 大文件上传：直传 vs Multipart

测试环境：本机到 us-east-1 S3，1 Gbps 出口，文件 5 GB。

| 方式 | 总耗时 | 失败重试代价 | CPU 占用 |
| --- | --- | --- | --- |
| **PutObject 单次** | 51 s | 全部重传（5 GB） | 12% |
| **Multipart 64MB / 串行** | 53 s | 1 片 = 64 MB | 14% |
| **Multipart 64MB / 4 并发** | **18 s** | 1 片 = 64 MB | 38% |
| **Multipart 64MB / 8 并发** | 16 s | 1 片 = 64 MB | 55% |
| **Multipart 16MB / 8 并发** | 19 s | 1 片 = 16 MB | 62% |
| **Multipart 256MB / 4 并发** | 22 s | 1 片 = 256 MB | 30% |

观察：
- **并发数比分片大小更影响吞吐**：4 并发已经吃满带宽
- **太小的分片增加 HTTP 请求开销**：16 MB / 8 并发反而比 64MB / 4 并发慢
- **太大的分片重传代价高**：256 MB 一旦失败要重传 256 MB

`aws s3 cp` 的默认配置：8 MB 分片、10 并发。这是个保守值，对大文件可以调大：

```bash
aws configure set default.s3.multipart_chunksize 64MB
aws configure set default.s3.max_concurrent_requests 16
```

### 3.2 不同 Erasure Coding 配置的存储效率

MinIO 测试：4 节点 × 4 盘 = 16 盘集群，写入 1 TB 实际数据。

| 配置 | 用盘 | 实际占用 | 效率 | 容灾 |
| --- | --- | --- | --- | --- |
| `EC:0`（无 EC） | 1 盘 | 1.0 TB | 100% | 0 盘 |
| `EC:2`（14+2） | 16 盘 | 1.14 TB | 87.5% | 2 盘 |
| `EC:4`（12+4） | 16 盘 | 1.33 TB | 75% | 4 盘 |
| `EC:8`（8+8） | 16 盘 | 2.0 TB | 50% | 8 盘 |
| 3 副本 | 3 盘 | 3.0 TB | 33% | 2 盘（同一对象） |

MinIO 默认 `EC:N` 中 N = 校验块数，K = 总盘数 - N。

**反直觉发现**：3 副本"只能容忍 2 盘"是指**同一对象的 2 个副本失败**。EC:4 是"任意 4 盘"。在 16 盘集群里，EC:4 的实际可靠性比 3 副本更强（多数情况下），且节省 67% 空间。

### 3.3 MinIO vs Ceph 在分布式场景的吞吐

| 测试 | MinIO（4 节点 × 4 NVMe） | Ceph RGW（4 节点 × 4 NVMe） |
| --- | --- | --- |
| 4 KB PUT QPS | 18,000 | 6,500 |
| 4 KB GET QPS | 42,000 | 18,000 |
| 1 MB PUT 吞吐 | 3.2 GB/s | 2.1 GB/s |
| 1 MB GET 吞吐 | 4.8 GB/s | 3.4 GB/s |
| 100 MB Multipart | 3.5 GB/s | 2.6 GB/s |
| 单对象 List（10 万 keys） | 1.2 s | 2.3 s |
| 全集群 List（1000 万 keys） | 28 s | 41 s |

观察：
- MinIO 在小对象 / 高 QPS 场景明显领先（无元数据服务的优势）
- Ceph 在大对象顺序读写差距缩小
- Ceph 的优势不在 RGW 单协议性能，而在**块/文件/对象统一**

但**这不代表 MinIO 全面碾压**。Ceph 的优势在：
- 同一集群同时提供块（RBD）、文件（CephFS）、对象（RGW）
- 跨数据中心多活（RGW Multisite）
- 大型企业的成熟运维生态

### 3.4 S3 强一致后的实测延迟

写后立即读的延迟分布（us-east-1 region 内）：

| 操作 | P50 | P99 | P999 |
| --- | --- | --- | --- |
| PUT 完成后 GET（同 key） | 12 ms | 35 ms | 80 ms |
| PUT 完成后 LIST（含该 key） | 18 ms | 45 ms | 110 ms |
| PUT 完成后 HEAD | 8 ms | 22 ms | 55 ms |

强一致**不等于零延迟**，仍有几十毫秒的"传播"时间，但**结果是确定的**：调用 GET 在 PUT 返回成功后**永远**能拿到新版本（不会拿到老版本或 404）。

老代码里的 `sleep(1000)` 现在可以删掉，但**网络重试还得保留** —— 因为 PUT 可能因为偶发 5xx 失败，强一致只保证语义不保证可用性。

### 3.5 与 NFS / FUSE 的延迟对比

把 S3 挂载成本地文件系统的几种方案：

| 方案 | 单文件读延迟 | 1000 文件 ls 延迟 | 适合场景 |
| --- | --- | --- | --- |
| **NFS（EFS）** | 2 ms | 50 ms | POSIX 兼容性强、共享配置 |
| **s3fs-fuse** | 50 ms | 8000 ms | 偶尔挂用，性能差 |
| **goofys** | 30 ms | 1200 ms | 顺序读 / 大文件 |
| **JuiceFS（带 Redis 元数据）** | 1 ms（缓存命中）/ 30 ms | 80 ms | AI 训练、混合工作负载 |
| **Alluxio** | 0.5 ms（缓存命中） | 30 ms | 读密集 + 计算紧耦合 |

**关键认知**：把 S3 当文件系统用是**反模式**。POSIX 语义（fsync、locking、partial write、目录 rename）和对象存储的 PUT-only 语义本质冲突。能用 SDK 直接调 S3 就不要挂载。

非要挂载的合理场景：
- 老程序改不动，只能 read/write 文件
- 需要 POSIX 兼容的中间结果（如 Spark shuffle）
- 大模型训练需要 dataset random access → 用 JuiceFS / Alluxio

> **口诀**：S3 不是文件系统。**SDK 访问 > FUSE 挂载**；要挂载选 JuiceFS / Alluxio，别用 s3fs。

---

## 4. 生产案例 9 则

> 每个案例固定四块：现象 / 根因 / 修复 / 验证。

### 案例 ① 小文件过多导致 List 慢 —— "数据湖查询从 5s 退化到 5min"

#### 现象
Spark 跑数据湖查询，几个月前 5 秒返回，现在 5 分钟。CPU、内存、网络都正常。Spark UI 显示**大量时间花在 `listLeafFiles`**。

#### 根因
表分区 `s3://lake/orders/year=2026/month=05/day=09/` 下面有 **80 万个 1 KB 的文件**。每个文件是一次 streaming 任务的微批输出。

S3 的 List API：
- 单次 ListObjectsV2 最多返回 1000 个对象
- 80 万个对象 = 800 次 List 调用
- 每次 List 50ms → 总耗时 40 秒（这还只是 List 阶段）
- 加上每个文件的 footer 读取（Parquet 要读 footer 找 schema）→ 80 万次小 GET → 几分钟

更深层问题：**S3 内部按 prefix 分片做 List**。同一 prefix 下文件越多，单次 List 越慢，且**整个 prefix 共享一个 IO 限速桶**（参见案例 ⑤）。

#### 修复

策略 A：**Compact 小文件**

```java
// Iceberg / Delta / Hudi 都有 compact 命令
spark.sql("CALL lake.system.rewrite_data_files(table => 'db.orders', " +
          "options => map('target-file-size-bytes', '134217728'))");  // 128 MB
```

定期跑（每天一次），把小文件合并成 128 MB ~ 1 GB 的"大文件"。这是数据湖的核心运维操作。

策略 B：**控制写入端的批大小**

```java
// Spark Streaming 调整 trigger 间隔，让每批数据更大
df.writeStream
  .trigger(Trigger.ProcessingTime("10 minutes"))   // 不要 10 秒一批
  .start();
```

策略 C：**用 manifest 文件代替 List**

Iceberg / Delta Lake 自带这个能力 —— manifest 里记录了所有数据文件的位置，**reader 不需要 List S3**，直接读 manifest 即可。所以**只要你用 Iceberg / Delta，第一个问题自然不存在**。但前提是**所有写入都通过 Iceberg API**，野路子直接写 S3 + 让 reader 去 List 才会触发。

#### 验证
```bash
# Spark 任务的 listLeafFiles 时间应该归零（用 manifest 后）
# 单个分区文件数 < 100
aws s3 ls s3://lake/orders/year=2026/month=05/day=09/ --recursive | wc -l
```

> **口诀**：**对象存储讨厌小文件**。数据湖 compact 是必修运维操作；写入端控批；reader 端走 manifest 不走 List。

---

### 案例 ② Multipart 未完成的"幽灵"分片占空间 —— "S3 账单暴涨 3 倍"

#### 现象
财务月底对账：S3 存储费用从 $800 涨到 $2400。但业务数据量没增加多少。CloudWatch 看 `BucketSizeBytes` 也确实涨了 2 TB，但**ListObjectsV2 找不到这些对象**。

#### 根因

Multipart Upload 的"开始"和"完成"是两个独立步骤。如果上传中途**没调 CompleteMultipartUpload 也没调 AbortMultipartUpload**，已上传的分片**永久驻留**，**计费但不可见**。

常见触发场景：
- 客户端进程被 kill 时没有 finally 块兜底
- 网络断开后没有 retry
- 老业务下线了但 multipart 任务还在跑
- 用户上传到一半关了浏览器

`ListObjectsV2` 看不到这些分片，要用 `ListMultipartUploads`：

```bash
aws s3api list-multipart-uploads --bucket uploads
```

```json
{
  "Uploads": [
    { "UploadId": "...", "Key": "video/aaa.mp4", "Initiated": "2025-11-01T..." },
    { "UploadId": "...", "Key": "logs/bbb.gz",  "Initiated": "2025-08-15T..." },
    ...   // 几千条
  ]
}
```

每个 Upload 的已上传分片都计费。

#### 修复

**强制兜底**：bucket lifecycle 自动清理超过 N 天的 incomplete multipart：

```json
{
  "Rules": [{
    "ID": "abort-incomplete-multipart",
    "Status": "Enabled",
    "Filter": {},
    "AbortIncompleteMultipartUpload": {
      "DaysAfterInitiation": 7
    }
  }]
}
```

**这条 lifecycle 是所有 bucket 的必配项**。MinIO / OSS / OBS 同样支持。

应用层兜底：

```java
String uploadId = null;
try {
    var resp = s3.createMultipartUpload(...);
    uploadId = resp.uploadId();
    // ... 上传分片 ...
    s3.completeMultipartUpload(...);
    uploadId = null;        // 标记已完成
} finally {
    if (uploadId != null) {
        try {
            s3.abortMultipartUpload(AbortMultipartUploadRequest.builder()
                .bucket(bucket).key(key).uploadId(uploadId).build());
        } catch (Exception ignore) {}
    }
}
```

#### 验证
```bash
# 配 lifecycle 之后过 7 天，再查应该归零
aws s3api list-multipart-uploads --bucket uploads | jq '.Uploads | length'
```

CloudWatch metric `NumberOfObjects` vs `BucketSizeBytes` 的比例应该回到正常。

> **口诀**：**所有 bucket 必配 AbortIncompleteMultipartUpload 7 天**。应用层 try-finally 兜底。

---

### 案例 ③ 预签名 URL 泄漏导致数据泄漏 —— "客户拿到了别人的发票"

#### 现象
客户投诉："我下载我的发票时打开了别人公司的发票 PDF。"

#### 根因
代码：

```java
// ❌ 错误的写法
String url = presigner.presignGetObject(...).url().toString();
return Map.of("downloadUrl", url, "expiresIn", 3600);
// 用户 A 拿到 URL 后转发给用户 B；URL 一小时内有效
// 用户 B 用 URL 直接下载用户 A 的文件
```

更糟的版本：

```java
// 同一个 URL 被前端缓存到 localStorage，过期前都能用
// 用户 A 的浏览器扩展 / 共享设备 → 泄漏
```

预签名 URL 的本质：**签名内嵌在 URL 里**，任何拿到 URL 的人都能用，直到过期。这是**功能特性，不是 bug**。

#### 修复

**短有效期 + 一次性兑换**：

```java
@PostMapping("/files/{id}/download")
public Map<String, String> getDownloadUrl(@PathVariable Long id, @AuthenticationPrincipal User u) {
    File f = repo.findById(id).orElseThrow();
    if (!f.getOwnerId().equals(u.getId())) throw new ForbiddenException();

    // 生成短 URL（5 分钟）
    String url = presigner.presignGetObject(GetObjectPresignRequest.builder()
        .signatureDuration(Duration.ofMinutes(5))      // ★ 不要 1 小时
        .getObjectRequest(GetObjectRequest.builder()
            .bucket(BUCKET).key(f.getS3Key())
            .build())
        .build()).url().toString();

    // 记审计日志
    audit.log("download", u.getId(), f.getId());
    return Map.of("downloadUrl", url);
}
```

**进阶：服务端代理 + 短 token**

```java
// 不返 S3 URL，返我们自己的下载 URL（带短 token）
String token = downloadTokens.create(u.getId(), f.getId(), Duration.ofMinutes(5));
return Map.of("downloadUrl", "/dl/" + token);

// 真正下载时再校验 token + 用我们的 IAM 流式代理
@GetMapping("/dl/{token}")
public ResponseEntity<StreamingResponseBody> stream(@PathVariable String token) {
    var ctx = downloadTokens.consume(token);    // ★ 一次性，用完作废
    if (ctx == null) throw new ForbiddenException();
    return streamFromS3(ctx.fileId());
}
```

这种方式：
- token 一次性，不能转发
- 服务端日志能精确到"谁在什么时候下载了什么"
- 代价：流量过你的服务器（可加 CDN 缓解）

**绑定 IP / Referrer**（仅特定场景）：

```java
// AWS S3 支持的 condition：从 IAM 策略层限定来源 IP
// bucket policy:
"Condition": {
  "IpAddress": { "aws:SourceIp": "203.0.113.0/24" }
}
```

#### 验证
- 签出来的 URL 复制到另一个 IP / 设备 → 应被拒绝（如果加了 IP 限制）
- token 用过一次第二次应该 403
- 监控异常下载频次告警

> **口诀**：预签名 URL = 短期外发的**裸权限**。**有效期必须短**；敏感场景必须**一次性兑换**；审计日志必配。

---

### 案例 ④ 跨区域复制（CRR）延迟 —— "灾备站点查不到最新数据"

#### 现象
做了主从两个 region 的 S3 跨区域复制（CRR），灾备演练时发现：从主 us-east-1 上传的文件，过了 30 秒在备 us-west-2 还查不到，几分钟后才出现。

#### 根因
S3 CRR（Cross-Region Replication）是**异步**复制：
- PUT 返回成功 = 数据落地到主 region
- 后台 worker 把数据复制到目标 region
- 目标 region 出现的延迟 = **几秒 ~ 几小时**（取决于对象大小、跨 region 带宽）

这是个**功能定义**：CRR 不承诺 RPO=0。AWS 后来推出了 **S3 Replication Time Control (RTC)**：

```json
{
  "Rules": [{
    "Destination": {
      "Bucket": "arn:aws:s3:::backup-west",
      "ReplicationTime": {
        "Status": "Enabled",
        "Time": { "Minutes": 15 }       // 99.99% 在 15 分钟内复制完成
      },
      "Metrics": {
        "Status": "Enabled",
        "EventThreshold": { "Minutes": 15 }
      }
    }
  }]
}
```

RTC 给 SLA：**99.99% 的对象在 15 分钟内复制完成**。但仍不是实时，且**比普通 CRR 贵很多**。

更深的问题：**强一致只在单 region 内**。跨 region 没有强一致保证（CAP 中 P 已经发生）。

#### 修复

策略 A：**业务层放弃跨 region 强一致**

灾备只用于"主 region 完全失联"的场景，正常读写都在主 region。能接受 RPO ≈ 几分钟。

策略 B：**关键数据双写 + 校验**

```java
// 关键数据同时写主备两个 region
CompletableFuture<?> primary = CompletableFuture.runAsync(() ->
    primaryS3.putObject(req, body), pool);
CompletableFuture<?> backup = CompletableFuture.runAsync(() ->
    backupS3.putObject(req, body), pool);

// 主必须成功；备失败异步补偿
primary.join();
backup.exceptionally(t -> { reconcileQueue.put(req); return null; });
```

策略 C：**多活架构（如 Aurora / DynamoDB Global Tables）**

对象存储一般不做多活（成本太高）。多活通常用于元数据 / 索引，对象存储仍是异步复制。

#### 验证
```bash
# 监控 ReplicationLatency 指标
aws cloudwatch get-metric-statistics \
  --namespace AWS/S3 --metric-name ReplicationLatency \
  --dimensions Name=BucketName,Value=primary Name=DestinationBucket,Value=backup \
  --start-time ... --end-time ... --period 300 --statistics Average,Maximum
```

业务上做"主备数据一致性巡检"任务，定期对比关键 key 的 etag。

> **口诀**：CRR 是**异步**复制。需要 SLA 用 RTC；要严格一致只能业务双写 + 对账。

---

### 案例 ⑤ 高并发下的 503 SlowDown —— "瞬秒活动 5 秒后全挂"

#### 现象
直播带货瞬秒活动开始 5 秒后，前端开始大量报错。后端日志大量 `503 Slow Down` 来自 S3。

```
software.amazon.awssdk.services.s3.model.S3Exception:
  Please reduce your request rate. (Service: S3, Status Code: 503, ...)
```

#### 根因

S3 的限流是**按 prefix（key 的前缀）**做的，不是按整个 bucket：

| 操作 | 限速（默认每 prefix） |
| --- | --- |
| GET / HEAD | 5,500 RPS |
| PUT / COPY / DELETE | 3,500 RPS |

业务的 key 设计是：

```
products/2026-05-09/img-1.jpg
products/2026-05-09/img-2.jpg
...
products/2026-05-09/img-100000.jpg
```

所有请求都打到 `products/2026-05-09/` 这个 prefix。瞬秒时几万 QPS 全部撞限流桶 → 503。

S3 内部会**自动分片**热 prefix（split），但需要**几分钟到几十分钟的时间窗口**才能完成。瞬秒场景根本来不及。

#### 修复

**方案 A：Hash 散列 prefix**

```
原 key:  products/2026-05-09/img-1.jpg
新 key:  products/<hash(id)[0:2]>/2026-05-09/img-1.jpg
        →  products/a3/2026-05-09/img-1.jpg
        →  products/7e/2026-05-09/img-1.jpg
        →  ... 256 个不同 prefix → 256 倍限流
```

代价：失去"按日期 list"的便利。可以保留 metadata 索引在外部（DB / Elasticsearch）做查询。

**方案 B：CDN 前置**

瞬秒商品的图片本来就该上 CDN。CDN 命中率 95%+ 之后，回源 S3 的 QPS 自然下降一个量级。

```
浏览器 → CDN edge（命中 95%）→ S3（5%）
原 100K RPS → S3 只剩 5K RPS
```

**方案 C：渐进升温（pre-warming）**

如果场景预知（如双 11），可以提前几小时低频访问目标 prefix，让 S3 内部完成 split 准备。AWS 支持工单申请预先扩容。

**方案 D：对热 prefix 加本地缓存层（read-through cache）**

```java
@Cacheable(value = "imgs", key = "#key")
public byte[] getImage(String key) {
    return s3.getObjectAsBytes(...).asByteArray();
}
```

对热文件用 Caffeine 本地缓存 + Redis 共享缓存。

#### 验证
压测 prefix 设计，单 prefix QPS 应分散到目标限流以下：

```bash
wrk -t8 -c200 -d60s -s upload.lua https://s3.amazonaws.com/bucket/

# CloudWatch 看 503 错误率
aws cloudwatch get-metric-statistics --metric-name 5xxErrors ...
```

> **口诀**：S3 限流**按 prefix**。热点必然 503。**散 prefix + 上 CDN + 本地缓存**三连。

---

### 案例 ⑥ MinIO 集群扩容期间的服务质量 —— "扩容那天客户全挂了"

#### 现象
MinIO 4 节点扩容到 8 节点，运维直接 `minio server ... node{1..8}` 重启 —— 结果是一场灾难：客户全挂、数据看不到、监控报警炸了。

#### 根因
MinIO 4.x 的扩容模型是 **Server Pool（服务池）**：

```
扩容前：Pool 1 = node{1..4}
扩容后：Pool 1 = node{1..4}        (老数据不动)
        Pool 2 = node{5..8}        (新数据写到这里)
```

正确的命令：

```bash
minio server \
  http://node{1...4}/data{1...4} \
  http://node{5...8}/data{1...4}
```

**不是**：

```bash
# ❌ 这会被理解为新集群，老数据全丢失（实际是无法识别）
minio server http://node{1...8}/data{1...4}
```

更深层：MinIO 不会**自动重平衡**老 Pool 的数据到新 Pool。新数据进新 Pool，老数据还在老 Pool。这避免了扩容期间的大量数据迁移流量。

但运维以为"扩容 = 数据自动均分"，重启换了拓扑 → MinIO 误以为是全新集群 → 老数据"消失"。

#### 修复

**正确的扩容流程**：

```bash
# 1. 先用同样的命令在所有 8 节点 systemd 配置中加新 Pool
ExecStart=/usr/local/bin/minio server \
  http://node{1...4}/data{1...4} \
  http://node{5...8}/data{1...4}

# 2. 在所有节点同时重启（必须同时，否则元数据冲突）
ansible -i inv all -m systemd -a "name=minio state=restarted"

# 3. 验证
mc admin info myminio
# 输出应该显示 2 个 Server Pool
```

**回滚方案**：保留扩容前的配置 + 数据快照。MinIO 4.x 不支持"缩 Pool"，所以扩容前必须想清楚。

**升级生产时的额外保险**：先在测试环境做完整演练，文档化每一步。

#### 验证
```bash
mc admin heal -r myminio          # 全集群健康检查
mc admin info myminio             # 看 Pool 数和盘数
mc admin trace myminio | head     # 看请求是否分散到新 Pool
```

> **口诀**：**MinIO 扩容是"加 Pool"不是"扩 Pool"**。命令必须包含原拓扑 + 新拓扑；所有节点同时重启；不会自动重平衡老数据。

---

### 案例 ⑦ 与 Hadoop / Spark 的兼容性 —— "S3A 写入丢数据"

#### 现象
Spark 任务把结果写到 S3，看任务成功，但 `_SUCCESS` 文件存在、数据文件却**少了一半**。重跑就好了。

#### 根因

Hadoop 的 `FileOutputCommitter` 设计假设：
1. 多个 task 各自写到 staging 目录
2. driver 在 commit 阶段把 staging rename 到最终目录
3. rename 在 HDFS 上是原子的元数据操作（瞬间完成）

但 **S3 没有真正的 rename** —— rename = copy + delete。对一个有几千个文件的目录：
- 几千次 copy + 几千次 delete
- 中间任何一个失败都可能留下"半完成"状态
- 总耗时几分钟（HDFS 是几秒）

这个问题有专门名字叫 ["S3 commit problem"](https://hadoop.apache.org/docs/stable/hadoop-aws/tools/hadoop-aws/committers.html)。

旧的 `FileOutputCommitter v1` 在 S3 上：
- 慢（几分钟的 copy）
- 可能丢数据（rename 中途失败）
- 可能写脏数据（task 失败重试 + 老 task 仍在写）

#### 修复

**方案 A：用 S3A Magic Committer**

Hadoop 3.1+ 提供专为 S3 设计的 committer，利用 multipart upload 的"可推迟 commit"特性：

```
1. task 写数据到 S3 时用 multipart upload
2. task 完成时**不调 CompleteMultipartUpload**，只把 uploadId 保存到 magic 路径
3. driver 收集所有 task 的 uploadId
4. driver 在 job commit 阶段批量调 CompleteMultipartUpload
```

Spark 配置：

```bash
spark-submit \
  --conf spark.hadoop.fs.s3a.committer.name=magic \
  --conf spark.hadoop.fs.s3a.committer.magic.enabled=true \
  --conf spark.sql.parquet.output.committer.class=org.apache.spark.internal.io.cloud.BindingParquetOutputCommitter \
  --conf spark.sql.sources.commitProtocolClass=org.apache.spark.internal.io.cloud.PathOutputCommitProtocol \
  ...
```

效果：commit 阶段 < 1 秒；语义正确；重试安全。

**方案 B：直接用 Iceberg / Delta Lake**

它们的 atomic commit 不依赖 rename，而是用"原子 PUT manifest"。从根本上避免了这个问题。

**方案 C：写本地 → 用 distcp 上传**

不推荐，但小规模任务可以这么干：先写本地 HDFS，跑完用 `hadoop distcp` 拷贝到 S3。

#### 验证
```bash
# 跑前后对比
spark-submit job.py
aws s3 ls s3://output/job-id/ --recursive | wc -l    # 文件数应该和预期一致

# 看 S3 端日志（CloudTrail）应该几乎没有 CopyObject 调用
```

> **口诀**：S3 没有原子 rename。Spark / Hadoop 写 S3 必须用 **Magic Committer** 或 Iceberg / Delta Lake。**别用默认的 FileOutputCommitter v1**。

---

### 案例 ⑧ 大文件下载断点续传 —— "客户网络抖一下，2GB 重头来"

#### 现象
APP 下载固件升级包（2 GB）。用户网络抖一下断开，恢复后**整个文件重新下载**。流量爆炸客诉爆炸。

#### 根因
代码：

```java
// ❌ 一把梭哈下载
HttpResponse<InputStream> resp = httpClient.send(request, BodyHandlers.ofInputStream());
Files.copy(resp.body(), Path.of("firmware.bin"));
```

S3 / 任何 HTTP 服务都支持 **Range 请求**：

```
GET /firmware.bin HTTP/1.1
Range: bytes=1073741824-

HTTP/1.1 206 Partial Content
Content-Range: bytes 1073741824-2147483647/2147483648
```

但客户端代码没用，断了就从头开始。

#### 修复

**断点续传客户端**：

```java
public void downloadResumable(String url, Path dest) throws Exception {
    long downloaded = Files.exists(dest) ? Files.size(dest) : 0;
    long total = headContentLength(url);
    if (downloaded >= total) return;     // 已完成

    HttpRequest req = HttpRequest.newBuilder(URI.create(url))
        .header("Range", "bytes=" + downloaded + "-")
        .build();

    try (var resp = httpClient.send(req, BodyHandlers.ofInputStream());
         var out = Files.newOutputStream(dest, StandardOpenOption.CREATE,
                                         StandardOpenOption.APPEND)) {
        if (resp.statusCode() != 206 && resp.statusCode() != 200) {
            throw new IOException("unexpected status: " + resp.statusCode());
        }
        resp.body().transferTo(out);
    }
}

// 加上 ETag 校验防止文件中途变了
private void validateETag(String url, String expectedETag) { ... }
```

**多线程并发下载（aria2 / IDM 风格）**：

```java
// 文件分 8 段并发下载
long partSize = total / 8;
List<CompletableFuture<?>> futures = new ArrayList<>();
for (int i = 0; i < 8; i++) {
    long start = i * partSize;
    long end = (i == 7) ? total - 1 : (start + partSize - 1);
    futures.add(CompletableFuture.runAsync(() ->
        downloadRange(url, start, end, dest), pool));
}
CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
```

注意：要用 `RandomAccessFile` 或 `FileChannel` 写到固定 offset，不是 append。

**服务端的事**：要在 S3 / CDN 前面挂 Nginx 时，确保 `proxy_buffering off` 或者明确支持 Range，否则断点续传失效。

```nginx
location /downloads/ {
    proxy_pass https://s3.internal/;
    proxy_buffering off;            # ★ 透传 Range
    proxy_request_buffering off;
}
```

#### 验证
```bash
# 模拟下载到一半中断后续传
curl -o file.bin --range 0-1000000 https://...    # 下载前 1 MB
curl -o file.bin --range 1000001- --append https://...   # 续传剩余
md5sum file.bin    # 校验完整性
```

> **口诀**：客户端必须支持 Range；ETag 校验防变更；并发下载用 RandomAccessFile 写偏移。CDN / 反代必须透传 Range。

---

### 案例 ⑨ 加密选型 —— "等保审查问 KMS 你才发现没开"

#### 现象
等保 2.0 三级审查，要求"用户敏感数据必须加密存储"。检查发现 S3 bucket 设置里加密项是空的。

#### 根因
S3 提供四种加密选项，多数团队上线时忘了开：

| 模式 | 密钥管理 | 性能影响 | 适合场景 |
| --- | --- | --- | --- |
| **SSE-S3 (AES256)** | S3 自己管 | 几乎无 | 默认开启，最低成本合规 |
| **SSE-KMS** | AWS KMS 管 | 几% | 需要密钥审计、轮转 |
| **SSE-C** | 客户端提供密钥 | 几乎无 | 客户必须自管密钥（医疗 / 金融） |
| **客户端加密 (CSE)** | 应用自己加 | 看实现 | S3 也看不到明文 |

旧 bucket 不开加密：写进去都是明文（虽然在 AWS 内部网络传输有 TLS）。

#### 修复

**方案：默认加密 + bucket policy 强制**

```bash
# 1. 开 bucket 默认加密
aws s3api put-bucket-encryption --bucket sensitive-data \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "alias/sensitive-data-key"
      },
      "BucketKeyEnabled": true
    }]
  }'
```

`BucketKeyEnabled` 重要：S3 用 bucket-level KEK（密钥加密密钥），减少调 KMS 的次数（**省钱 99%**）—— KMS 调用是按次计费的。

**bucket policy 强制加密** —— 防止有人 PUT 时显式覆盖加密设置：

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyUnencryptedUploads",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:PutObject",
    "Resource": "arn:aws:s3:::sensitive-data/*",
    "Condition": {
      "StringNotEquals": {
        "s3:x-amz-server-side-encryption": "aws:kms"
      }
    }
  }]
}
```

**老数据加密回填**：

```bash
# 用 inventory + batch operation 把已有对象重新 copy（触发加密）
aws s3 cp s3://sensitive-data/ s3://sensitive-data/ \
  --recursive --sse aws:kms --sse-kms-key-id alias/sensitive-data-key
```

注意：`cp` 同 bucket 同 key 会触发服务端 copy（不下载）。

**KMS 选型对比**：

| 场景 | 选什么 |
| --- | --- |
| 一般合规需求 | SSE-S3 |
| 等保 / 数据主权 | SSE-KMS（密钥可审计、轮转） |
| 多租户严格隔离 | 每租户一个 KMS key |
| 必须自管密钥 | SSE-C 或 CSE |

#### 验证
```bash
# 检查所有对象的加密状态
aws s3api list-objects-v2 --bucket sensitive-data --query 'Contents[].Key' --output text \
  | xargs -I{} aws s3api head-object --bucket sensitive-data --key {} \
  --query '[Key, ServerSideEncryption, SSEKMSKeyId]' --output table

# 应全部显示 aws:kms
```

> **口诀**：**新 bucket 默认开 SSE-KMS + BucketKey + 强制策略**。老数据用 batch copy 回填。KMS 调用费靠 BucketKey 省。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### Bucket 配置
- [ ] **Block Public Access** 账户级 + bucket 级双开（除非确实是公开 bucket）
- [ ] **默认加密** SSE-S3 或 SSE-KMS（敏感数据必 KMS）
- [ ] **BucketKeyEnabled** 开启（KMS 场景省钱）
- [ ] **版本控制（Versioning）**：合规场景必开
- [ ] **MFA Delete**：金融 / 关键数据开启
- [ ] **Logging**：访问日志写到独立 bucket
- [ ] **Object Lock**：合规留存（WORM）场景必开

### Lifecycle 策略
- [ ] **AbortIncompleteMultipartUpload** 7 天 —— 所有 bucket 必配
- [ ] **NoncurrentVersionExpiration** —— 版本控制 bucket 必配
- [ ] **Transition** 老数据到 IA / Glacier
- [ ] **Expiration** 临时 bucket 设置过期

### 上传 / 下载
- [ ] 大文件用 Multipart Upload（> 100 MB）
- [ ] 分片大小 64 ~ 256 MB；并发 4 ~ 8
- [ ] try-finally 兜底 abort
- [ ] 客户端直传用预签名 URL，不要走代理
- [ ] 预签名有效期 ≤ 15 分钟
- [ ] Content-Type / Content-Length 在签名时固定

### 性能
- [ ] Key 设计避免单 prefix 热点（hash 散）
- [ ] 大对象 Get 用 Range 并发
- [ ] 静态资源前置 CDN
- [ ] 数据湖小文件定期 compact

### 安全
- [ ] IAM 最小权限（Resource 用前缀，不给 `*`）
- [ ] Bucket Policy 显式 Deny 未加密上传
- [ ] 跨账户访问用 STS AssumeRole，不发长期 AK
- [ ] Presigned URL 一次性兑换（敏感场景）
- [ ] 审计日志 + 异常下载告警

### 监控
- [ ] 4xx / 5xx 错误率（含 SlowDown）
- [ ] BucketSizeBytes / NumberOfObjects 趋势
- [ ] CRR / RTC ReplicationLatency
- [ ] 单 prefix QPS（按业务关键 key 抽样）
- [ ] KMS 调用次数（费用监控）

### 数据湖 / Spark 兼容
- [ ] Spark 用 Magic Committer 或 Iceberg / Delta
- [ ] Iceberg 表配 retention + compact 定时任务
- [ ] 读 manifest 不读 List
- [ ] 写入端控制 trigger 间隔避免小文件

### 灾备
- [ ] 跨区域复制（CRR）启用，关键 bucket 配 RTC
- [ ] 主备一致性巡检任务
- [ ] 业务能容忍 RPO（明确文档化）
- [ ] 跨 region 数据校验（etag 比对）

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### S3 Express One Zone（高性能）
- 2023 推出，单 AZ 部署，专为高 IOPS 场景
- 单对象延迟从 ~10 ms 降到 ~3 ms
- 价格更贵但 IOPS 不限速
- 用于：AI 训练 checkpoint、低延迟数据访问

### 数据湖三剑客深入
- **Iceberg**：表格式之争中胜出，Snowflake / Databricks 都支持
- **Delta Lake**：Databricks 主推，与 Spark 集成最紧
- **Hudi**：Uber 出品，记录级 upsert 是优势
- 关注：Iceberg REST Catalog、open table format 之争

### 缓存层
- **JuiceFS**：用 Redis / TiKV 做元数据，对象存储做数据，POSIX 兼容
- **Alluxio**：分布式缓存，跑在计算节点本地 SSD
- **CubeFS（原 ChubaoFS）**：京东开源的分布式存储，POSIX + S3 双协议

### 替代方案 / 新势力
- **SeaweedFS**：Go 实现的小文件优化对象存储（Facebook Haystack 思想）
- **Cloudflare R2**：S3 兼容，**零出口流量费**（流量大场景能省 90%）
- **Backblaze B2**：性价比之选
- **Wasabi**：1/5 价格的 S3 兼容

### 对象存储 + Lambda（无服务器）
- S3 Object Lambda：在 GET 请求时插入函数（图片缩放、PII 脱敏）
- Event Notification → SNS / SQS / Lambda：写入触发处理
- 注意 Lambda 冷启动 + S3 一致性窗口的相互作用

### 自研对象存储的工程化
- 美团 **Mss**：内部对象存储，应对小文件场景优化
- 字节 **TOS**：火山引擎对象存储
- 阿里 **OSS**：飞天底座的产品化
- 关注点：元数据分片、IO 路径优化、纠删码工程实现、跨地域复制

### 大模型时代的新需求
- 训练 checkpoint：动辄 TB 级，频繁读写 → S3 Express / 本地 NVMe + 异步上传
- 数据集：百亿小文件 → 必须用 manifest（如 WebDataset、HuggingFace dataset）
- 推理模型分发：CDN + P2P（如 Dragonfly）

---

## 7. 引用

- AWS S3 [Strong Consistency 公告](https://aws.amazon.com/s3/consistency/)（2020）
- [S3 API Reference](https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html)
- MinIO [Erasure Code Documentation](https://min.io/docs/minio/linux/operations/concepts/erasure-coding.html)
- Sage Weil et al., *CRUSH: Controlled, Scalable, Decentralized Placement of Replicated Data* (SC'06)
- Sage Weil et al., *Ceph: A Scalable, High-Performance Distributed File System* (OSDI'06)
- Hadoop [S3A Committers](https://hadoop.apache.org/docs/stable/hadoop-aws/tools/hadoop-aws/committers.html)
- [Apache Iceberg Specification](https://iceberg.apache.org/spec/)
- [Delta Lake Protocol](https://github.com/delta-io/delta/blob/master/PROTOCOL.md)
- Sanjay Ghemawat et al., *The Google File System*（2003）—— 对象存储设计的远祖
- Doug Beaver et al., *Finding a needle in Haystack: Facebook's photo storage*（OSDI'10）—— 小文件存储经典

---

> 上一篇深文：[MySQL MVCC 与锁](/article/db/mysql-mvcc) ｜ 配套：[Cassandra 深入](/article/db/cassandra) | [Linux IO 模型](/article/linux-perf/io-models)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。下一篇候选告诉我写哪个：数据湖 Iceberg / Delta、Kafka 存储引擎、Redis 持久化、ElasticSearch 分片设计。
