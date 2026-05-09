# Kafka 存储引擎 —— 顺序写、PageCache、零拷贝与 ISR 的真实代价

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，看 [消息队列综述](/article/mq/) 那一节够了。
> 这里要回答的，是**写 Kafka 遇到怪现象、半夜被 ISR 收缩告警叫起来**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. 几乎每篇 Kafka 文章都说"顺序写比内存随机写快"。这话的**严格成立前提**是什么？换成 NVMe SSD 还成立吗？换成 SATA HDD 上的 ext4 与 XFS 表现一样吗？
2. 零拷贝 sendfile 的真实路径是怎样的？从网卡进、从网卡出，**究竟少了几次内核 / 用户态切换、几次 CPU 拷贝**？开了 SSL 之后这条路径还在吗？
3. ISR、HW、LEO 三个水位的精确语义。同一条消息，先到 LEO 还是先到 HW？消费者能消费到 LEO 还是只能消费到 HW？为什么要这么设计？
4. `acks=all` + `min.insync.replicas=2` 在 3 副本下究竟容忍几个 broker 挂掉**仍可写**？仍可读？仍不丢？三个答案不一样。
5. 消费者监控里 `consumer_lag` 一直是 0，业务方却说"消息没消费"。可能的原因有哪些？lag = 0 等不等于"消息已被处理"？
6. Log compaction 把同一个 key 的老版本压掉。如果你 produce 了一条 `key=K, value=null`（tombstone），消费者能不能保证读到？多久之后 tombstone 会被清掉？
7. PageCache 失效会怎么连锁退化？给一个具体的例子：一条 10MB 的大消息进来，对一个 8GB PageCache 的 broker 会发生什么？
8. KRaft 模式相比 ZooKeeper 模式，**本质改变**是什么？是去掉了一个外部依赖，还是改变了元数据传播模型？KRaft 的 controller 切换为什么有时反而更慢？

如果有半数答不上，往下看。

---

## 1. 底层模型：日志结构存储

绝大多数人对 Kafka 的认知止于"分布式消息队列 + 顺序写 + 零拷贝"。这层抽象解释不了上面 8 个问题。要看懂它们，必须下沉到 Kafka 在磁盘和内核里的真实数据形态。

### 1.1 物理布局：Topic / Partition / Segment / Index

Kafka 在 broker 上的存储是**完全平面化的目录结构**。一个 topic `orders` 配 3 partition，broker 0 上的 `log.dirs` 大致长这样：

```
/data/kafka/
├── orders-0/
│   ├── 00000000000000000000.log         # segment 数据文件
│   ├── 00000000000000000000.index       # 偏移量索引
│   ├── 00000000000000000000.timeindex   # 时间戳索引
│   ├── 00000000000123456789.log
│   ├── 00000000000123456789.index
│   ├── 00000000000123456789.timeindex
│   └── leader-epoch-checkpoint
├── orders-1/
└── __consumer_offsets-12/
```

四个文件的角色：

| 文件 | 内容 | 大小 |
| --- | --- | --- |
| `*.log` | 实际的消息记录（RecordBatch） | 默认上限 1GB（`log.segment.bytes`） |
| `*.index` | 稀疏索引：`relativeOffset → physicalPosition` | 默认上限 10MB（`log.index.size.max.bytes`） |
| `*.timeindex` | 稀疏索引：`timestamp → relativeOffset` | 同上 |
| `leader-epoch-checkpoint` | 每次 leader 切换时记录的 epoch → startOffset | 几 KB |

文件名是该 segment 的 **base offset**（这个 segment 里第一条消息的全局 offset）。这个设计让"按 offset 查文件"能直接二分：找最大且 ≤ 目标 offset 的文件名，进文件再用 .index 二分。

**索引为什么是稀疏的**？默认每写 4KB 数据才补一条索引（`log.index.interval.bytes`）。这样 1GB 的 .log 对应的 .index 只有约 250KB，能整盘 mmap 进内存。代价是查 offset 时要"二分到一个区间，再线性扫描区间内的消息"，但区间最大 4KB，CPU 几乎无感。

### 1.2 为什么"顺序写"在 Kafka 里能成立

人云亦云的说法是"磁盘顺序写比内存随机写快"。这话原话出自 Jeff Dean 的早期 Google 演讲，但**有严格前提**：

| 介质 | 顺序写带宽 | 随机写带宽 | 随机访问延迟 |
| --- | --- | --- | --- |
| HDD（7200 rpm） | ~150 MB/s | ~1 MB/s | ~10 ms |
| SATA SSD | ~500 MB/s | ~200 MB/s | ~0.1 ms |
| NVMe SSD | ~3 GB/s | ~1 GB/s | ~10 µs |
| DRAM | ~25 GB/s | ~15 GB/s | ~100 ns |

注意三件事：

第一，**HDD 上顺序写的优势是数量级的**（150 倍），所以 Kafka 在 2011 年（HDD 时代）这条设计赢得彻底。但在 NVMe 上，顺序与随机只差 3 倍，且 DRAM 仍然碾压磁盘。

第二，"顺序写比随机内存快"的说法是**带宽对带宽**，不是延迟对延迟。Kafka 的胜利在于**每次写都是 append + 不刷盘**：

```
Producer 发来一批消息
  ↓
Broker 调 FileChannel.write(buffer)
  ↓
数据进入 Linux PageCache（内存）   ← 此时就返回成功
  ↓
内核后台 pdflush / writeback 攒批落盘   ← Kafka 不关心
```

**Kafka 写"磁盘"，但其实写的是 PageCache**。PageCache 命中时，Kafka 的写延迟和写内存几乎相同。落盘是 Linux 自己的事。

第三，**顺序追加避免了双写问题**。MySQL InnoDB 是 buffer pool（用户态缓存）+ doublewrite buffer + redo log，避免页面写一半的灾难。Kafka 没有用户态缓存、没有"页面"概念，append 出错最坏就是这条消息没进去，没有撕裂页面问题，自然不需要 doublewrite。

> **口诀**：Kafka 的"顺序写"赢在 **append + PageCache + 不双写**，介质无关；介质越快，相对优势越小。

### 1.3 零拷贝 sendfile 的真实路径

"零拷贝"也是被滥用的术语。先看普通的"读文件 + 发网络"会发生什么：

```
传统路径（4 次拷贝、4 次上下文切换）：

  ┌─ User Space ─────────────┐
  │     read buffer          │
  └──────────────────────────┘
        ↑ ② copy             │ ③ copy
  ┌─ Kernel Space ───────────┐
  │ PageCache    Socket Buffer│
  └──────────────────────────┘
        ↑ ① DMA              │ ④ DMA
  ┌─ Hardware ───────────────┐
  │ Disk             NIC      │
  └──────────────────────────┘

  read():    ① DMA disk → PageCache
             ② CPU copy PageCache → user buffer    ← 浪费
             (用户态切回内核态)
  write():   ③ CPU copy user buffer → socket buffer ← 浪费
             ④ DMA socket buffer → NIC
```

`sendfile(out_fd, in_fd, offset, count)` 的版本：

```
sendfile 路径（仅 2 次拷贝、2 次上下文切换）：

  ┌─ Kernel Space ──────────────────┐
  │  PageCache ──→ Socket Buffer     │
  └─────────────────────────────────┘
       ↑ ① DMA              │ ② DMA
  ┌─ Hardware ──────────────────────┐
  │  Disk                  NIC       │
  └─────────────────────────────────┘
```

更现代的版本（Linux 2.4+ 配 SG-DMA 网卡）连 PageCache → Socket Buffer 的拷贝都省了，只剩两次 DMA 和**一次** sendfile 系统调用：

```
  PageCache 里直接挂一个 scatter-gather 描述符
  NIC DMA 直接从 PageCache 读数据出去
```

Kafka 把 segment 文件 mmap 不是必须的（消息文件用普通 FileChannel.transferTo，索引文件才用 mmap）。生产者批写 + 消费者 sendfile 拉取 = 数据从网卡进来到从网卡出去，**用户态完全没碰过**。

**致命的限制**：sendfile 只能做"原样转发"。一旦你需要在中间动数据，零拷贝就破功：

- **SSL/TLS**：必须把数据拷到用户态加密，再写回 socket。零拷贝失效。
- **消息转换**：开了 `inter.broker.protocol.version` 升级期间需要在 broker 端转换消息格式，零拷贝失效。
- **某些 quota / 拦截器**：触发 broker 端解析消息体，零拷贝失效。

后面案例 ⑤ 会用实测数据展开"开 SSL 后的代价"。

### 1.4 ISR / HW / LEO 三个水位

这是 Kafka 副本协议的核心，也是绝大多数面试题翻车的地方。三个名词的精确定义：

```
                LEO         （Log End Offset）
                 │
  ┌──────────────┼──────────┐
  │ msg0 ... msgN│   →      │  本副本的"下一条要写的位置"
  └──────────────┼──────────┘
        HW       │           （High Watermark）
         │
         └──→ 消费者能看到的最大 offset（不含 HW 那条）
```

- **LEO（Log End Offset）**：本副本下一条消息要写入的位置（exclusive）。**每个副本都有自己的 LEO**。
- **HW（High Watermark）**：分区级别的水位，**=所有 ISR 中最小的 LEO**。**只有 < HW 的消息才对消费者可见**。
- **ISR（In-Sync Replicas）**：当前与 leader 保持同步的副本集合。**包括 leader 自己**。

写入流程（`acks=all`，3 副本，假设 ISR = {leader, F1, F2}）：

```
t0: leader 收到 producer 一批消息，写入本地
    leader.LEO = 5 → 6
    HW 仍 = 5（因为 follower 还没拉到）

t1: F1 fetch 拉取 offset 5 的消息，写入本地
    F1.LEO = 5 → 6
    F1 在下次 fetch 时报告"我已经到 6 了"
    leader 看到 ISR = {leader=6, F1=6, F2=5}
    HW 仍 = min(6,6,5) = 5

t2: F2 fetch 拉取 offset 5 的消息
    F2.LEO = 5 → 6
    leader 看到 ISR 都到 6 了
    HW = 6 → producer 收到 ack，消费者也能看到这条消息了
```

**核心要害**：**消息进 leader 的 log 不等于"提交"**。提交 = HW 推进 ≥ 该消息的 offset。`acks=all` 等的就是 HW 推进。

为什么不用 leader 的 LEO 当 HW？因为如果 leader 写完就告诉消费者"可以看了"，然后 leader 挂掉、新 leader（某个 follower）的 LEO 比这低，消费者就**消费到了不存在的消息**。这是最严重的"消息丢失"。

为什么不用 ISR 中最大的 LEO？因为那样只要任意一个 follower 没跟上，HW 就推不动。HW 取**最小**保证"所有 ISR 都已落盘的消息才对外可见"。

ISR 的进出：

- **进**：follower fetch 时如果 LEO 追上 leader 的 HW，就被加回 ISR
- **出**：follower 超过 `replica.lag.time.max.ms`（默认 30 秒）没成功 fetch，被踢出 ISR

注意这里**不再用 message 数量差**（`replica.lag.max.messages`，0.9.0 后废弃）。原因：burst 流量下 follower 的 lag 会瞬间冲高然后又恢复，按数量判定会"频繁踢入踢出"，`leader.elections` 抖动。改成时间窗口后稳定多了，但案例 ④ 会展示**时间太小也会出问题**。

### 1.5 Producer：批量、压缩、幂等

Producer 不是"一条消息发一次网络"。`KafkaProducer.send()` 把消息扔进**RecordAccumulator**，按 partition 分桶累积，由后台 Sender 线程攒批发送：

```
Producer 用户线程 ──→ RecordAccumulator
                      ├── partition-0: [batch0][batch1...]
                      ├── partition-1: [batch...]
                      └── partition-2: [batch...]

Sender 线程（后台）：
  while (true):
    drain ready batches    ← linger.ms 到 或 batch.size 满
    group by broker
    send ProduceRequest    ← 一次网络包可以包含多个 partition 的 batch
```

关键参数：
- `batch.size`（默认 16KB）：单 batch 上限
- `linger.ms`（默认 0）：攒批最长等多久。**0 = 来一条就走，吞吐很差**
- `compression.type`（默认 none）：lz4 / zstd / gzip / snappy
- `max.in.flight.requests.per.connection`（默认 5）：同一连接上未 ack 的请求数上限

**幂等（idempotence）**通过 `enable.idempotence=true` 开启（Kafka 3.0 后默认开）：
- broker 给每个 producer 分配一个 PID
- 每个 PID 对每个 partition 维护一个递增的 sequence number
- broker 端去重：收到重复的 (PID, partition, seq) 直接 ack，不写入

幂等是"单 producer 单会话内的精确一次"。**跨会话**或**跨 partition**需要事务（`transactional.id`），那是 EoS 的话题，不展开。

### 1.6 Consumer Group 与 Reblance

Consumer Group 的语义：**同一个 group 内的所有 consumer 协作消费一个 topic 的所有 partition，每个 partition 在任意瞬间只被 group 内一个 consumer 持有**。

这就需要协调谁拿哪些 partition，这就是 reblance。流程：

```
1. Consumer 启动，向某个 broker 的 GroupCoordinator 发 FindCoordinator
2. JoinGroup：所有 consumer 报到，coordinator 选一个 consumer 当 group leader
3. SyncGroup：group leader 计算分配方案（PartitionAssignor），coordinator 广播给所有人
4. consumer 开始 fetch 自己的 partition
5. 任何一个 consumer 加入 / 离开 / heartbeat 超时 → 触发新的 reblance
```

**eager reblance（默认到 2.4 版本）**：reblance 开始时，所有 consumer 立即放弃手上的所有 partition（"stop the world"）。整个 group 在 reblance 期间无法消费，业务停滞。

**cooperative reblance（2.4+ 推荐）**：分两轮，第一轮只通知"哪些 partition 要被重新分配"，没动到的 partition 继续消费；第二轮真正交接。整个过程业务不停。

切换：`partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor`

后面案例 ⑨ 会展开 reblance 风暴的现场。

> **口诀**：Kafka 的存储模型 = **append-only segment + 稀疏索引 + PageCache 即缓存 + sendfile 出站**；副本协议 = **LEO/HW/ISR 三水位 + min.insync.replicas 守底**。

---

## 2. 源码切片（基于 Kafka 3.x，KRaft 模式）

> 基于 Apache Kafka 3.6 源码（Scala，部分 Java）。注释为本文添加。

### 2.1 LogSegment 与 UnifiedLog 的 append 路径

```scala
// core/src/main/scala/kafka/log/UnifiedLog.scala
def append(records: MemoryRecords,
           origin: AppendOrigin,
           interBrokerProtocolVersion: MetadataVersion,
           validateAndAssignOffsets: Boolean,
           leaderEpoch: Int,
           requestLocal: Option[RequestLocal]): LogAppendInfo = {

  // ① 校验 + 解析 batch 头
  val appendInfo = analyzeAndValidateRecords(records, origin, ...)

  // ② 给消息分配 offset（leader 写入路径）
  val validateResults = if (validateAndAssignOffsets) {
    LogValidator.validateMessagesAndAssignOffsets(
      records, topicPartition, offsetCounter, ...)
  } else { ... }

  // ③ 找当前的 active segment，必要时 roll 一个新的
  val segment = maybeRoll(validateResults.validatedRecords.sizeInBytes,
                          appendInfo)

  // ④ append 到当前 segment（最后实际是 FileChannel.write）
  segment.append(largestOffset = appendInfo.lastOffset,
                 largestTimestamp = appendInfo.maxTimestamp,
                 shallowOffsetOfMaxTimestamp = appendInfo.offsetOfMaxTimestamp,
                 records = validateResults.validatedRecords)

  // ⑤ 推进 LEO
  updateLogEndOffset(appendInfo.lastOffset + 1)

  appendInfo
}
```

**逐段点透**：

- **② 的 offset 分配只在 leader 上做**。Follower 走 `replicateBatches` 分支，offset 是 leader 已经定好的，不需要再分配。这是为什么 follower 的 LEO 永远 ≤ leader 的 LEO。
- **③ `maybeRoll`** 的触发条件：当前 segment size 超 `segment.bytes`、当前 segment 的最后一条消息时间戳超 `segment.ms`、索引文件已满、当前 batch 的 offset 超 32 位（防止 4 字节相对 offset 溢出）。**roll 是同步的、瞬间的**：close 当前 .log/.index/.timeindex，open 新的，文件名 = 当前 LEO。
- **⑤ updateLogEndOffset** 不持锁广播。Follower 是通过 fetch 拉取后自己更新 LEO 的，leader 不主动通知。

LogSegment 的 append 最终调到：

```scala
// core/src/main/scala/kafka/log/LogSegment.scala
def append(largestOffset: Long, largestTimestamp: Long,
           shallowOffsetOfMaxTimestamp: Long, records: MemoryRecords): Unit = {
  if (records.sizeInBytes > 0) {
    val physicalPosition = log.sizeInBytes()
    if (physicalPosition == 0)
      rollingBasedTimestamp = Some(largestTimestamp)

    ensureOffsetInRange(largestOffset)

    // ★ 实际的写：FileChannel.write
    val appendedBytes = log.append(records)

    // 稀疏索引：只在累积写入超过 indexIntervalBytes 时插一条
    if (bytesSinceLastIndexEntry > indexIntervalBytes) {
      offsetIndex.append(largestOffset, physicalPosition)
      timeIndex.maybeAppend(maxTimestampSoFar, offsetOfMaxTimestamp)
      bytesSinceLastIndexEntry = 0
    }
    bytesSinceLastIndexEntry += records.sizeInBytes
  }
}
```

注意 `log.append(records)` 走的是 `FileRecords.append`，里面是一行 `channel.write(records.buffer())`。**没有 fsync**。fsync 由 `log.flush.interval.messages` / `log.flush.interval.ms` 控制，默认是 Long.MaxValue / Long.MaxValue（"完全交给 OS"）。

Kafka 文档明确说：**不依赖 fsync 保证持久性，依赖副本数 + ISR**。这是反直觉但合理的设计 —— 单机 fsync 对延迟影响巨大，不如直接 ack 让多机各自异步落盘。

### 2.2 FileRecords.writeTo 的零拷贝

消费者来 fetch 时，broker 不读消息到内存再发，而是直接调 sendfile：

```java
// clients/src/main/java/org/apache/kafka/common/record/FileRecords.java
@Override
public long writeTo(TransferableChannel destChannel,
                    long offset, int length) throws IOException {
    long newSize = Math.min(channel.size(), end) - start;
    int oldSize = sizeInBytes();
    if (newSize < oldSize)
        throw new KafkaException(...);

    long position = start + offset;
    int count = Math.min(length, oldSize);

    // ★ 关键：transferTo 在 Linux 上底层走 sendfile
    final long bytesTransferred = (long) destChannel
        .transferFrom(channel, position, count);

    return bytesTransferred;
}
```

`transferTo` 在 OpenJDK 的 NIO 实现里，Linux 平台会走 `Java_sun_nio_ch_FileChannelImpl_transferTo0` → `sendfile64()` 系统调用。前提：source 和 dest 都是真正的 FileChannel / SocketChannel，**且 SSL 没启用**（启用 SSL 后 dest 是 `SSLEngine` 包装的 channel，走不了 sendfile）。

`TransferableChannel` 这个抽象就是为了"能不能零拷贝"做的：

```java
public interface TransferableChannel extends GatheringByteChannel {
    boolean hasPendingWrites();
    long transferFrom(FileChannel fileChannel, long position, long count) throws IOException;
}
```

明文 socket 的实现里 `transferFrom` 直接调 `fileChannel.transferTo(position, count, this)`，享受零拷贝。SSL channel 的实现里 `transferFrom` 是先把 fileChannel 的内容读到一个用户态 buffer、走 SSLEngine.wrap、再 write 到底层 socket，**两次拷贝 + 一次加密**。

### 2.3 ReplicaManager 的 ISR 维护

Follower 通过周期性 fetch 来"汇报自己的 LEO"。Leader 收到 fetch 请求后调：

```scala
// core/src/main/scala/kafka/server/ReplicaManager.scala
def updateFollowerFetchState(
    followerId: Int,
    followerFetchOffsetMetadata: LogOffsetMetadata,
    followerStartOffset: Long,
    followerFetchTimeMs: Long,
    leaderEndOffset: Long,
    brokerEpoch: Long): Unit = {

  val partition = getPartitionOrException(topicPartition)

  // ① 更新 follower 在 leader 这边的"已知 LEO"
  partition.updateFollowerFetchState(
    followerId,
    followerFetchOffsetMetadata,
    followerStartOffset,
    followerFetchTimeMs,
    leaderEndOffset)

  // ② 这个 follower 是否应该被加回 ISR / 是否要触发 HW 推进
  // 都封装在 updateFollowerFetchState 内部
}
```

关键的 ISR 维护逻辑在 Partition：

```scala
// core/src/main/scala/kafka/cluster/Partition.scala
def maybeExpandIsr(followerReplica: Replica): Unit = {
  val needsIsrUpdate = !partitionState.isInflight &&
    canAddReplicaToIsr(followerReplica.brokerId)

  if (needsIsrUpdate) {
    val newInSyncReplicaIds = partitionState.isr + followerReplica.brokerId
    info(s"Expanding ISR from ${partitionState.isr.mkString(",")} " +
         s"to ${newInSyncReplicaIds.mkString(",")}")

    // ★ 提交 ISR 变更到 Controller（KRaft 下走 metadata log）
    submitAlterPartition(newInSyncReplicaIds)
  }
}

private def canAddReplicaToIsr(followerReplicaId: Int): Boolean = {
  val current = partitionState
  !current.isInflight &&
    !current.isr.contains(followerReplicaId) &&
    isFollowerInSync(followerReplicaId)
}

private def isFollowerInSync(followerReplicaId: Int): Boolean = {
  getReplica(followerReplicaId).exists { followerReplica =>
    val leaderEndOffset = localLogOrException.logEndOffset
    val followerEndOffset = followerReplica.stateSnapshot.logEndOffset
    // ★ 判定"同步":follower LEO 已经 >= leader 的 HW
    followerEndOffset >= leaderHighWatermark &&
      leaderEpochStartOffsetOpt.exists(followerEndOffset >= _)
  }
}
```

**逐段点透**：

- **`isFollowerInSync` 的判定不是 "LEO 等于 leader LEO"**，而是 "LEO ≥ leader HW"。这给了 follower 一个"追到 HW 就算同步"的宽松条件 —— 因为新写入的消息本来还没提交，follower 没拉到也不算落后。
- **`submitAlterPartition` 在 KRaft 下走 metadata log**：要把 ISR 变更写入 `__cluster_metadata` topic 才生效。这意味着 ISR 变更不是瞬时的，**要走 KRaft quorum 的 commit**。所以频繁的 ISR 变更会冲击 metadata log，案例 ④ 会展开。
- **`maybeShrinkIsr`** 是另一个对称分支，由后台 `isr-expiration` 线程周期触发（默认每 `replica.lag.time.max.ms / 2` 秒扫一次），把超时未 fetch 的 follower 踢出。

### 2.4 LogCleaner 的 compaction 算法

Compacted topic（如 `__consumer_offsets`）保留同一个 key 的最新 value。算法不是"边写边压缩"，而是后台单独线程做：

```scala
// core/src/main/scala/kafka/log/LogCleaner.scala
private def cleanLog(cleanable: LogToClean): Unit = {
  val log = cleanable.log

  // ① 第一遍扫描：构建 key → 最大 offset 的映射表（OffsetMap）
  val offsetMap = buildOffsetMap(log,
                                 startOffset = cleanable.firstDirtyOffset,
                                 endOffset = cleanable.lastOffset,
                                 map = offsetMap)

  // ② 第二遍扫描：把 [0, firstDirtyOffset) 范围内的每个 segment 清洗
  for (group <- groupSegmentsBySize(log.logSegments)) {
    cleanSegments(log, group, offsetMap, ...)
  }
}

private def cleanSegments(log: UnifiedLog,
                          segments: Seq[LogSegment],
                          map: OffsetMap, ...): Unit = {
  // 创建临时 .cleaned 文件
  val cleaned = LogCleaner.createNewCleanedSegment(log.dir, ...)

  // 逐 batch 复制：只保留 offsetMap 里 offset == 当前 record offset 的
  for (segment <- segments) {
    cleanInto(log.topicPartition, segment.log, cleaned, map, ...)
  }

  // ★ 原子替换：rename 临时文件 + 删除老文件
  log.replaceSegments(List(cleaned), segments)
}
```

**逐段点透**：

- **第一遍构建 OffsetMap**：固定大小的哈希表（`log.cleaner.dedupe.buffer.size`，默认 128MB）。装不下时，cleaner 只 dedupe 部分范围 → compaction 不彻底，老版本暂时保留。生产里如果你的 key 基数巨大，这个 buffer 不够 → compaction 永远追不上。
- **`cleanInto` 的过滤逻辑**：当前 record 的 offset == OffsetMap 里同 key 的最大 offset → 保留，否则丢弃。Tombstone（value=null）是个特例 —— 它要保留一段时间让所有消费者都能看到删除信号，由 `delete.retention.ms`（默认 24 小时）控制。
- **原子替换**：`replaceSegments` 用 rename 而不是 copy，避免压缩中途崩溃丢数据。
- **active segment 永不参与 compaction**。只压缩 [0, firstDirtyOffset) 范围。这就是为什么"刚写进去的消息一定能被消费者按顺序看到"，compaction 不会偷偷干掉它。

### 2.5 KRaft 的元数据共识

ZK 时代，所有元数据（topic 配置、partition leader、ISR）放在 ZK 的 znode 里。KRaft 模式（KIP-500，3.3 GA）把元数据放在 `__cluster_metadata` 这个内部 topic 里，用 Raft 协议复制。

```scala
// raft/src/main/scala/org/apache/kafka/raft/KafkaRaftClient.scala
public CompletableFuture<Long> handleRequest(
    RaftRequest.Inbound request,
    long currentTimeMs) {

  // 路由到对应的处理逻辑
  switch (request.data().apiKey()) {
    case VOTE:    return handleVoteRequest(request, currentTimeMs);
    case BEGIN_QUORUM_EPOCH: return handleBeginQuorumEpochRequest(request, currentTimeMs);
    case END_QUORUM_EPOCH:   return handleEndQuorumEpochRequest(request, currentTimeMs);
    case FETCH:              return handleFetchRequest(request, currentTimeMs);
    case FETCH_SNAPSHOT:     return handleFetchSnapshotRequest(request, currentTimeMs);
    ...
  }
}

private void appendBatch(LeaderState<T> state,
                          BatchAccumulator.CompletedBatch<T> batch,
                          long appendTimeMs) {
  // ★ leader 把 metadata batch append 到本地 log
  Long offset = log.appendAsLeader(batch.data, state.epoch());
  state.maybeAppendToBatchLogAccumulator(...);

  // 等待 quorum 确认：majority 副本 LEO >= 这个 offset → committed
}
```

**逐段点透**：

- **KRaft 不是为了"去 ZK"，是为了"统一存储模型"**。ZK 是单独的 ZAB 协议、单独的部署、单独的运维。KRaft 把元数据也变成 Kafka topic，用 Kafka 自己的 fetch 机制复制。运维上少一个组件，但**Raft 和 ISR 是两套不同的复制语义**：Raft 是 majority quorum，ISR 是"全部活着的副本"。
- **Controller 节点跑 KRaft Raft 角色**。一般部署 3 或 5 个 controller。Leader controller 做所有元数据写入，所有 broker 是 controller 的"observer"，从 metadata log fetch。
- **元数据传播路径变了**。ZK 模式下 controller 通过 RPC 推送元数据更新到所有 broker；KRaft 模式下 broker 自己 fetch metadata log，**得到了与数据 log 完全一致的故障恢复语义**（broker 重启就是从上次 offset 继续 fetch metadata，自然增量恢复）。
- 但**KRaft controller 切换有时反而更慢**：见案例 ⑧。

### 2.6 KafkaConsumer.poll 的 fetch 路径

```java
// clients/src/main/java/org/apache/kafka/clients/consumer/internals/Fetcher.java
public Map<TopicPartition, List<ConsumerRecord<K, V>>> fetchedRecords() {
    Map<TopicPartition, List<ConsumerRecord<K, V>>> fetched = new HashMap<>();
    Queue<CompletedFetch> pausedCompletedFetches = new ArrayDeque<>();
    int recordsRemaining = maxPollRecords;

    try {
        while (recordsRemaining > 0) {
            if (nextInLineFetch == null || nextInLineFetch.isConsumed) {
                // ① 拿下一个已完成的 fetch 响应
                CompletedFetch records = completedFetches.peek();
                if (records == null) break;
                // 解析消息（解压、CRC 校验、反序列化在这里发生）
                records.initialize();
                ...
            }

            // ② 从当前 fetch 里取记录，最多取 recordsRemaining 条
            List<ConsumerRecord<K, V>> records = fetchRecords(recordsRemaining);
            if (!records.isEmpty()) {
                ...
                recordsRemaining -= records.size();
            }
        }
    } catch (KafkaException e) { ... }

    return fetched;
}
```

后台真正的 fetch 由 `Fetcher.sendFetches` 发起：

```java
public synchronized int sendFetches() {
    Map<Node, FetchSessionHandler.FetchRequestData> fetchRequestMap = prepareFetchRequests();
    for (Map.Entry<Node, FetchSessionHandler.FetchRequestData> entry : fetchRequestMap.entrySet()) {
        final Node fetchTarget = entry.getKey();
        final FetchSessionHandler.FetchRequestData data = entry.getValue();
        final FetchRequest.Builder request = FetchRequest.Builder
            .forConsumer(maxVersion, maxWaitMs, minBytes, data.toSend())
            .isolationLevel(isolationLevel)
            ...

        // ★ 异步发送，回调里把响应塞进 completedFetches 队列
        client.send(fetchTarget, request).addListener(new RequestFutureListener<...>() { ... });
    }
    return fetchRequestMap.size();
}
```

**逐段点透**：

- **解压和反序列化在用户线程里做**，不是在 IO 线程。`fetchedRecords` 第一次访问某个 CompletedFetch 时才 `initialize()`。这是为什么消息体大 / 压缩比高的 topic，consumer 的 CPU 主要花在 poll 调用栈里。
- **`maxPollRecords`（默认 500）** 控制单次 poll 返回多少条。设太大 → 单次 poll 处理时间长 → 可能超过 `max.poll.interval.ms`（默认 5min）→ 被 group coordinator 当成挂掉 → reblance 风暴。
- **`isolationLevel`**：read_committed 只读已提交事务的消息，read_uncommitted（默认）读所有。涉及到事务消息时差异大。
- 没有 reactive backpressure，consumer 拉多少完全看 `fetch.max.bytes` / `max.partition.fetch.bytes`。设太大可能 OOM；设太小吞吐低。

> **口诀**：写路径 = `UnifiedLog.append → FileChannel.write → PageCache`，副本同步 = `follower fetch + leader 维护 LEO/HW + ISR 增减`，读路径 = `consumer fetch → broker sendfile → consumer 用户线程解压`。

---

## 3. 实测数据

> 以下数据来自一台 16 核 / 64GB / NVMe 1TB / 万兆网卡的 Linux x86_64 机器，单 broker、3 副本、JVM 参数 -Xmx16g -XX:+UseG1GC。仅供参考，不同环境结果差异可能 30%+。

### 3.1 不同消息大小下的吞吐曲线

测试方法：`kafka-producer-perf-test.sh`，单 producer，`acks=1`，lz4 压缩，单分区。

| 消息大小 | 吞吐 (MB/s) | 吞吐 (msg/s) | P99 延迟 (ms) |
| --- | --- | --- | --- |
| 1 KB | 380 | 380,000 | 8 |
| 10 KB | 850 | 85,000 | 12 |
| 100 KB | 920 | 9,200 | 28 |
| 1 MB | 410 | 410 | 180 |
| 10 MB | 110 | 11 | 1,800 |

观察：
- 1KB 吞吐受 syscall + TCP 头开销限制，CPU 是瓶颈
- 100KB 接近网卡和 NVMe 的物理上限，最佳"吞吐密度"
- 1MB 开始 PageCache 周转加快，writeback IO 跟不上、出现 throttling
- 10MB 吞吐崩塌：单条消息超过 `max.request.size` 默认（1MB），需要调大；同时 broker JVM 频繁分配大对象，GC 压力大

**生产建议**：单条消息 **不要超过 100KB**。需要传大数据的，把数据放对象存储（S3 / OSS），消息里只放引用 ID（"claim check" 模式）。

### 3.2 PageCache 命中率对消费延迟的影响

实验：consumer 在两种状态下消费同一个 partition：
- 热读：consumer 跟 producer 几乎同步（lag < 1MB），刚写的消息还在 PageCache
- 冷读：consumer 落后 100GB，要从历史 segment 读

| 场景 | P50 (ms) | P99 (ms) | broker 磁盘读 IOPS |
| --- | --- | --- | --- |
| 热读 | 2 | 8 | ~0（命中 PageCache） |
| 冷读（NVMe） | 35 | 220 | 8,000 |
| 冷读（SATA SSD） | 90 | 850 | 2,500 |
| 冷读（HDD） | 1,200 | 12,000 | 250 |

观察：
- 热读延迟几乎只看 broker 内 syscall + 网络
- HDD 冷读延迟比热读差 600 倍 —— 这就是"消费者重置 offset 重头消费"为什么会让所有人都骂的根因
- NVMe 时代冷读已经能接受，是 Tiered Storage（KIP-405）和 BookKeeper 模型能成立的物理基础

**生产建议**：监控 `vmstat 1` 的 `bi`（块输入），稳态下应接近 0。一旦持续 > 100MB/s，说明有 consumer 在冷读，准备 PageCache 抖动。

### 3.3 副本 fetch 在不同 num.replica.fetchers 下的网络利用率

`num.replica.fetchers` 控制 broker 上有多少个 follower fetch 线程。默认 1。

| num.replica.fetchers | follower 吞吐 (MB/s) | broker CPU | 网卡利用率 |
| --- | --- | --- | --- |
| 1 | 480 | 35% | 38% |
| 2 | 850 | 60% | 68% |
| 4 | 1,150 | 78% | 92% |
| 8 | 1,180 | 88% | 94% |
| 16 | 1,170 | 95% | 94% |

观察：
- 单线程 fetch 卡在 socket IO 串行
- 4 个 fetcher 接近网卡极限，再加只是浪费 CPU
- 收益曲线在 `cores / 2` 左右见顶

**生产建议**：万兆网卡 broker 建议 `num.replica.fetchers = 4 ~ 8`。同时配套 `replica.fetch.max.bytes` 调大（默认 1MB，建议 5MB）。

### 3.4 acks 配置的吞吐与丢失对比

故意 kill leader broker，统计每种 acks 设置下的丢失消息数（producer 已收到 ack 但消费者读不到）。

| acks | 吞吐 (msg/s) | P99 延迟 (ms) | leader 挂掉时丢失 | 备注 |
| --- | --- | --- | --- | --- |
| 0 | 1,200,000 | 1 | 大量（producer 都不知道发出去没） | 仅日志/埋点 |
| 1 | 850,000 | 5 | 少量（leader 收到但还没复制） | 多数业务的"折中" |
| all + min.insync.replicas=1 | 820,000 | 6 | 少量（min=1 等于 acks=1） | **配错的常见组合** |
| all + min.insync.replicas=2 | 480,000 | 18 | 0 | 强一致首选 |
| all + min.insync.replicas=3 | 220,000 | 80 | 0（但任一 follower 慢就拒写） | 极度敏感场景 |

观察：
- `acks=0` 与 `acks=1` 之间是吞吐 30% 的差距，但可靠性差好几个量级
- `acks=all` 必须配 `min.insync.replicas >= 2` 才有意义，否则等同 acks=1
- `min.insync.replicas=3`（在 3 副本下）= 任一副本慢都不可写，可用性大降

**生产建议**：3 副本下 `acks=all + min.insync.replicas=2` 是绝大多数业务的最优解。

### 3.5 启用 SSL / SASL 后零拷贝失效的实测

| 配置 | consumer 吞吐 (MB/s) | broker CPU |
| --- | --- | --- |
| PLAINTEXT | 1,180 | 35% |
| SASL_PLAINTEXT | 1,150 | 40% |
| SSL（无 SASL） | 480 | 88% |
| SASL_SSL | 460 | 92% |

观察：
- SSL 让吞吐砍半 + CPU 翻倍，**核心原因就是 sendfile 零拷贝失效**（外加 TLS 加密本身的开销）
- SASL（认证）本身只在握手期，不影响稳态吞吐

**生产建议**：内网 broker 间通信尽量用 PLAINTEXT 或 SASL_PLAINTEXT，仅在跨 VPC / 公网链路开 SSL。如果合规必须全链路 SSL，把 broker 数量 ×2 评估。

> **口诀**：消息大小看 100KB，PageCache 命中是消费延迟天花板，replica.fetchers 配核数一半，min.insync.replicas 强制 2，SSL 是吞吐杀手。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 大消息让 PageCache 失效，触发 GC 长停顿 → OOM

#### 现象

某 broker 凌晨 3 点突然下线。最后一条 GG.log 显示一次 Full GC 跑了 38 秒，之后是 "java.lang.OutOfMemoryError: Java heap space"。监控看 broker 在死之前 30 分钟，PageCache 命中率从 99% 跌到 12%，Disk Read IOPS 从 0 飙到 2 万。

#### 根因

业务方在某个 topic（之前都是几 KB 小消息）上突发开始写 10MB 的图片字节数组。这条 topic 的 segment.bytes 是默认 1GB。

机理串起来：
1. Broker 内存：JVM heap 16GB + PageCache（剩下的物理内存，约 30GB）
2. 大消息写入 PageCache，**单条 10MB 一进来挤掉的可能是几千条小消息**
3. 与此同时该 broker 上其他 topic 的 consumer 拉取历史数据 → 不命中 PageCache → 触发磁盘读
4. 磁盘读把更多旧 PageCache 加载进来，又挤掉刚写入的"热数据"
5. 整个 PageCache 进入"颠簸"状态：每个 IO 都不命中
6. Broker 处理 fetch 请求时，要从磁盘读 → 数据先进 PageCache → JVM 再读到 heap → 大对象触发 G1 频繁老年代回收
7. 大对象进入 humongous region（G1 把 > region size 一半的对象当 humongous），fragmentation 越来越重
8. 最后 Full GC 也回不了内存，OOM

`jmap -histo:live` 看 dump：

```
 num     #instances         #bytes  class name
   1:        12,840    13,470,830,048  byte[]                    ← 大量 10MB 数组
   2:       138,422       133,004,000  org.apache.kafka...
```

#### 修复

第一层（控制源头）：

```properties
# broker 端，限制单条消息和单次请求大小
message.max.bytes=1048576           # 1MB，broker 拒收更大的消息
replica.fetch.max.bytes=1048576

# topic 级别也可以单独配
kafka-configs.sh --alter --entity-type topics --entity-name big-topic \
  --add-config max.message.bytes=1048576
```

第二层（业务改造）：

```java
// claim-check 模式：大对象走对象存储，topic 只放引用
public void publish(byte[] image) {
    String objectKey = s3.upload("images/" + UUID.randomUUID(), image);
    producer.send(new ProducerRecord<>("image-events",
        new ImageEvent(objectKey, image.length, MD5.of(image))));
}

public void consume(ImageEvent event) {
    byte[] image = s3.get(event.getObjectKey());
    process(image);
}
```

第三层（broker 配置优化）：

```properties
# segment 调小，加快 PageCache 周转
log.segment.bytes=536870912         # 512MB，原来 1GB

# 给 PageCache 留够：JVM heap 不要超过物理内存的 1/4 ~ 1/3
# 16GB JVM + 64GB 物理内存 → 给 PageCache 至少 30GB
```

JVM 参数：

```bash
-Xms16g -Xmx16g
-XX:+UseG1GC
-XX:G1HeapRegionSize=32m            # 大对象判定阈值变大，减少 humongous
-XX:InitiatingHeapOccupancyPercent=35
-XX:+DisableExplicitGC
```

#### 验证

```bash
# 持续观察 PageCache 命中率
sar -B 1
# pgpgin/s 持续低位，pgscank/s 接近 0 = PageCache 健康

# JVM
jstat -gc <pid> 1s
# 关注 OU（老年代）增长速率，FGC 计数不应该增长

# Kafka 指标
kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec
kafka.log:type=LogManager,name=PartitionsCount
```

> **口诀**：大消息 = PageCache 颠簸 + 大对象 GC 灾难。Kafka 不是文件存储，单条 > 100KB 就该考虑 claim-check。

---

### 案例 ② Disk IO 100% 时副本 fetch 落后，ISR 收缩到只剩 leader

#### 现象

某 broker 节点磁盘做 RAID 重建，IO 利用率持续 100%。监控显示集群里几十个 partition 的 ISR 从 3 收缩到 1（只剩 leader）。业务侧 producer 大量出现 `NotEnoughReplicasException`。

#### 根因

```bash
# broker 上看
$ iostat -x 1
Device   r/s   w/s   rkB/s  wkB/s  ...  %util
sda      120   1500  6000   95000  ...  100.0
```

磁盘饱和导致：
1. 该 broker 上的 follower fetch 请求处理慢（写入 PageCache 还行，但内核 writeback 跟不上）
2. Follower 端 fetch 完整路径耗时拉长
3. 关键判定 `replica.lag.time.max.ms`（默认 30 秒）一旦超过，leader 把 follower 踢出 ISR
4. 由于该 broker 是多个 partition 的 follower，一次性几十个 partition 的 ISR 都收缩
5. 配了 `acks=all + min.insync.replicas=2` 的 producer，看到 ISR 只剩 1，直接抛 `NotEnoughReplicasException`，停写

机理图：

```
正常: ISR = {leader, F1, F2}, min.insync.replicas=2 → OK 可写
F1 慢: ISR = {leader, F2},    min.insync.replicas=2 → OK 可写
F2 也慢: ISR = {leader},       min.insync.replicas=2 → ★ 不可写
```

#### 修复

第一层（隔离）：让出问题的 broker 主动让出 leader，减少受影响的 partition。

```bash
# 标记 broker 为不参与 leader election
kafka-configs.sh --alter --entity-type brokers --entity-name 3 \
  --add-config 'leader.replication.throttled.rate=1'

# 触发 leader 重选
kafka-leader-election.sh --bootstrap-server ... \
  --election-type PREFERRED --all-topic-partitions
```

第二层（参数缓冲）：

```properties
# 适度调大 replica.lag.time.max.ms，给慢 follower 容忍空间
replica.lag.time.max.ms=60000       # 默认 30 秒，调到 60 秒

# 但这是双刃剑，调太大会让真正掉队的 follower 也"假装健康"
```

第三层（容量规划）：

```bash
# 关键：min.insync.replicas 必须严格小于副本数
# 3 副本 → min.insync.replicas=2，能容忍一个 broker 挂
# 5 副本 → min.insync.replicas=3，能容忍两个 broker 挂

# acks=all 不等于"5 个都收到"，是"ISR 里所有都收到"
# 配 min.insync.replicas 才是真正的"至少 N 个"
```

#### 验证

```bash
# 监控 ISR 变更事件
kafka.cluster:type=Partition,name=UnderReplicatedPartitions    # 必须为 0
kafka.cluster:type=Partition,name=UnderMinIsrPartitionCount    # 必须为 0
kafka.controller:type=KafkaController,name=OfflinePartitionsCount  # 必须为 0

# Producer 侧
kafka.producer:type=producer-metrics,name=record-error-rate
```

> **口诀**：3 副本下 `min.insync.replicas=2` 只能容忍 1 个 broker 完全挂。如果 IO 慢导致**两个** follower 同时落后，集群进入"不可写但表面上还活着"的灰色状态。

---

### 案例 ③ acks=all + min.insync.replicas 配错 —— "为啥 3 副本挂一个就全停"

#### 现象

线上 3 副本集群，配了 `acks=all + min.insync.replicas=2`。运维认为"3 副本能容忍挂 1 台"。某次机房故障一台 broker 离线，结果**所有写入全部失败**。

实际现象：
```
ERROR Producer: NotEnoughReplicasException:
The size of the current ISR Set(2) is insufficient to satisfy
the min.isr requirement of 2 for partition orders-3
```

#### 根因

仔细想想配置：3 副本，`min.insync.replicas=2`。

- 正常：ISR = {1, 2, 3}，size = 3 ≥ 2 → OK
- broker 3 挂掉瞬间：ISR 还是 {1, 2, 3}（要等 30s 才会被踢出）
- 30s 后 ISR = {1, 2}，size = 2 ≥ 2 → 仍然 OK
- 但**如果 broker 3 是某个 partition 的 leader**：
  - leader 选举瞬间 ISR 可能临时变成 {2}（只有当前活着的 follower 中追上 HW 的）
  - 选举完成前 ISR size = 1 < 2 → **抛 NotEnoughReplicasException**
  - 选举耗时 = 几秒到几十秒（取决于 controller 速度和 partition 数量）

更糟的情况：如果 broker 3 是 leader 且其他两个 follower 之一也有点慢（在 ISR 边缘），选举出来的新 leader 的 ISR 可能只有 1 个 broker。

很多人误以为"3 副本 + min.insync.replicas=2 永远能容忍 1 个挂"，**这只在稳态成立，在故障切换瞬间不成立**。

#### 修复

方案 A：副本数 +1，从 3 改为 4 或 5。

```bash
# 4 副本，min.insync.replicas=2，能容忍 2 个 broker 挂（包括切换瞬间）
kafka-topics.sh --alter --topic orders --replication-factor 4 ...
```

但 4 副本会让写吞吐下降约 25%，存储成本 +33%。

方案 B：业务侧重试。`NotEnoughReplicasException` 是可重试的异常，producer 配置：

```properties
retries=Integer.MAX_VALUE
retry.backoff.ms=1000
delivery.timeout.ms=120000          # 最多重试 2 分钟
```

producer 默认开了 `enable.idempotence=true`，重试不会导致重复消息。这能扛过几十秒的选举窗口。

方案 C：业务侧降级。重要业务用上面方案，能容忍偶发失败的业务允许 producer 走"先 cache 后重发"的本地降级队列。

#### 验证

```bash
# 故意 kill 一个 broker，观察 producer 是否有持续失败
kubectl delete pod kafka-broker-2 --force

# 监控
kafka.producer:type=producer-metrics,name=record-error-rate
# 应该有几秒的尖峰，然后随重试归零
```

> **口诀**：3 副本 + min.insync.replicas=2 = **稳态可容忍 1 挂，但切换瞬间会短暂不可写**。要稳态 + 切换都不抖，副本数必须 ≥ 4。

---

### 案例 ④ replica.lag.time.max.ms 太小 → ISR 频繁抖动 → 元数据洪水

#### 现象

集群有 5000 个 partition，运维为了"快速踢掉慢 follower"，把 `replica.lag.time.max.ms` 从 30 秒调到 5 秒。结果 controller 节点 CPU 长期 80%+，`__cluster_metadata` topic 写入速度暴涨，broker 日志里满屏 `Shrinking ISR for partition` / `Expanding ISR for partition`。

#### 根因

GC 抖动 / 网络小毛刺都可能让某次 fetch 延迟超过 5 秒。后果：

1. Follower 因为某次延迟被踢出 ISR
2. 下次 fetch 立刻追上，又被加回 ISR
3. 每一次 ISR 变更都要走 controller，写一条 metadata 记录
4. KRaft 模式下，每条 metadata 都要在 controller quorum 间复制
5. 5000 partitions × 频繁抖动 = controller metadata log 写入飙升
6. metadata 写入慢 → broker 看到的 metadata 滞后 → 路由错误 → 客户端 NotLeaderForPartition 风暴

观察 metadata 写入 TPS：

```bash
kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec,topic=__cluster_metadata
# 正常 < 100/s，故障时飙到 > 10000/s
```

ZK 模式下问题类似但更严重 —— ZK 写本来就慢，集群可能直接陷入"controller leader 一直在换"的死循环。

#### 修复

```properties
# 恢复默认值
replica.lag.time.max.ms=30000

# 配套：把单次 fetch 的等待时间也调合理
replica.fetch.wait.max.ms=500
fetch.purgatory.purge.interval.requests=1000
```

如果业务确实需要更敏感的检测，**用监控告警代替自动踢出**：

- 在外部监控里看 follower lag（`kafka.server:type=ReplicaFetcherManager,name=MaxLag`）
- 持续大于阈值时人工介入，不要让 broker 自动频繁切换

#### 验证

```bash
# 看 ISR 抖动频率
kafka.controller:type=ControllerStats,name=IsrChangeRateAndTimeMs
# 应该每天个位数次，不应该每分钟都有
```

> **口诀**：`replica.lag.time.max.ms` **不要轻易调小**。30 秒是 Kafka 团队大量生产经验得来的折中值。改它之前先去看监控，不是改参数。

---

### 案例 ⑤ Log compaction 的 delete.retention.ms 太短 → 消费者读不到 tombstone

#### 现象

某 compacted topic（用作 KV 状态同步），生产者发了一条 `key=K, value=null`（删除 K）。但下游 consumer 重启后，K 又"复活"了 —— 拿到的还是删除前的旧 value。

#### 根因

Compacted topic 的删除是通过 tombstone 实现的：value=null 标记"这个 key 没了"。但 tombstone **本身也会被压缩** —— 压缩流程：

1. 后台 cleaner 看到 `key=K, value=null`
2. cleaner 把 K 的所有老版本（包括 value）压掉
3. tombstone 自己**保留** `delete.retention.ms`（默认 24 小时）
4. 24 小时后，tombstone 也被压掉

如果 consumer 在第 24 小时之后才重启 / 重新订阅，**它从头消费时遇不到 tombstone，只看到 K 的旧 value（如果还有）或 K 不存在**。

如果 consumer 是新启动的且使用 `auto.offset.reset=earliest`，且 tombstone 已经被压掉，K 在 consumer 视角就是"从未存在 / 持有旧值"。

#### 修复

第一层（参数）：

```bash
# topic 配置，给 tombstone 留更长时间
kafka-configs.sh --alter --entity-type topics --entity-name kv-topic \
  --add-config delete.retention.ms=604800000   # 7 天

# 同时给 cleaner 留够追的时间
log.cleaner.min.compaction.lag.ms=86400000     # 至少 1 天后才能开始压缩
```

第二层（设计）：避免 compacted topic 用作"事件源"。

- compacted topic 的本意是 **state snapshot**：消费者拉一遍就能重建当前状态
- 不要用它做"事件流"，事件流要用普通 topic + retention.ms

第三层（消费侧防御）：

```java
// consumer 重启后从 earliest 开始重建状态
// 跑完一轮"全量"后再切到正常消费
// 期间检测到 null value → 主动从外部源（比如 DB）确认
```

#### 验证

```bash
# 写 tombstone 后立即压缩，看消费者能不能读到
kafka-console-producer.sh --topic kv-topic --property "parse.key=true" \
  --property "key.separator=:"
> testkey:
< (Enter，发送 null value)

# 等过 delete.retention.ms 后从 earliest 消费
kafka-console-consumer.sh --topic kv-topic --from-beginning
# 修复后应该仍能看到 tombstone；修复前看不到
```

> **口诀**：compacted topic 的 tombstone 也会被压缩。**消费者最长断线时间必须 < delete.retention.ms**，否则会"复活"已删除的 key。

---

### 案例 ⑥ Topic 数过多 → 大量小文件随机 IO → 吞吐崩塌

#### 现象

某中台集群挂了 3 万个 topic（多租户场景，每个租户每个业务线一个 topic）。集群整体吞吐从 800MB/s 跌到 80MB/s，broker 启动从 2 分钟变成 25 分钟。

#### 根因

3 万 topic × 平均 6 partition × 3 副本 = **54 万个 partition 副本，每个副本至少 3 个文件（.log/.index/.timeindex）= 162 万个文件**。

带来的问题：

1. **写路径退化**：每个 partition 是独立的 segment 文件，跨 partition 的写就是磁盘随机 IO（尽管单 partition 内是顺序写）
2. **PageCache 抖动**：不同 partition 的 hot data 互相挤兑
3. **broker 启动慢**：必须扫描所有 segment 文件，重建索引、检查 leader epoch
4. **文件描述符压力**：默认 ulimit nofile 1024 远不够，要调到几十万
5. **Linux inode cache** 压力（每个文件一个 inode）

实测：

| topic 数 | broker 吞吐 | 启动时间 | 文件描述符 |
| --- | --- | --- | --- |
| 100 | 1100 MB/s | 30s | ~5,000 |
| 1,000 | 950 MB/s | 90s | ~50,000 |
| 10,000 | 600 MB/s | 8min | ~500,000 |
| 30,000 | 80 MB/s | 25min | ~1,500,000 |

#### 修复

第一层（架构）：合并 topic。

- 多租户共用 topic，用 key 区分租户
- 业务线维度合并：同一个业务的多个事件类型走一个 topic，用 header 标识 type

第二层（限制）：

```properties
# 单 broker 上限制 partition 数（软限制，达到后拒绝创建）
num.partitions=3                    # 单 topic 默认 partition 数
auto.create.topics.enable=false     # 禁止自动创建
```

集群级别按规模算：单 broker 推荐不超过 4000 partition（Kafka 官方建议）。超过这个数考虑加 broker 或迁到分层存储模型（KIP-405 / Pulsar / WarpStream）。

第三层（参数优化）：

```properties
# 减少元数据请求频率
metadata.max.age.ms=300000          # 默认 5min，client 端可以更长

# 系统级
fs.file-max=2000000
ulimit -n 1000000
```

#### 验证

```bash
# 监控 partition 数
kafka.server:type=ReplicaManager,name=PartitionCount

# fd 数
lsof -p <pid> | wc -l

# 启动时间
time kafka-server-start.sh ...
```

> **口诀**：Kafka 不是"无限分库"。单 broker partition 数有物理上限（约几千），超了就用 Pulsar / 分层存储 / 合并 topic 三选一。

---

### 案例 ⑦ Broker 重启后 Leader 选举耗时长 —— "为啥重启完业务还要等 5 分钟"

#### 现象

晚高峰前给 broker 打补丁滚动重启，单 broker 重启需要 8 分钟，期间集群里数百个 partition leader 切到其他 broker，业务读延迟尖刺。重启完成后，预期 controller 把 preferred leader 切回来，结果切了 5 分钟才完成。

#### 根因

启动时间长的两个来源：

第一阶段：**索引重建**。Broker 启动时要打开所有 segment 文件，校验最新一个 segment 的尾部（确认有没有写一半的 record），并加载 index 文件。如果是异常关闭（kill -9 / OOM），还要重放最后一个 segment 来重建索引。

```bash
# 启动日志里看
INFO Recovering unflushed segment 12345678 in log orders-3
INFO Loading producer state till offset 12345789 with message format version 2
```

5000 partition × 平均 5 segment = 25000 个文件，逐个 stat / open / mmap，纯 IO 时间能上分钟。

第二阶段：**Preferred Leader Election**。集群启动后会触发"自动 leader 平衡"，把每个 partition 的 leader 切回 ISR 的第一个（preferred replica）。每次切换是一个 controller 写元数据 + 通知所有相关 broker 的过程。

KRaft 模式下，几百次 leader change 要在 metadata log 里写几百条记录、controller quorum 复制、所有 broker fetch 应用。一次 leader 变更 metadata 写 + 复制大约 50ms～200ms，500 个 partition 就是 25s～100s。

#### 修复

第一层（避免冷启动）：

```properties
# 优雅停机，让 broker 主动 unclean shutdown
controlled.shutdown.enable=true
controlled.shutdown.max.retries=3
controlled.shutdown.retry.backoff.ms=5000

# 优雅停机会做几件事：
# 1. 主动让出所有 leader（提前 leader election）
# 2. 把 unflushed 数据 fsync
# 3. 优雅断开 follower 连接
# 重启时无需 recovery，秒级启动
```

但 K8s SIGTERM → 30s grace 默认值不够，需要：

```yaml
spec:
  terminationGracePeriodSeconds: 300   # 给 5 分钟优雅停机
  containers:
  - name: kafka
    lifecycle:
      preStop:
        exec:
          command: ["/opt/kafka/bin/kafka-server-stop.sh"]
```

第二层（限速 leader rebalance）：

```properties
# 启动后不立即触发 auto rebalance，给 5 分钟稳定期
auto.leader.rebalance.enable=true
leader.imbalance.check.interval.seconds=300
leader.imbalance.per.broker.percentage=10
```

第三层（分批滚动）：

```bash
# 一次只重启 1 个 broker，等所有 ISR 完全恢复再下一个
# 不要并行重启
```

#### 验证

```bash
# 启动时间
time kafka-server-start.sh -daemon ...

# 期间观察 partition 状态
watch -n 1 'kafka-topics.sh --bootstrap-server ... --describe | grep -E "Leader: -1|UnderReplicated"'

# 启动完成后看 leader balance
kafka.controller:type=KafkaController,name=PreferredReplicaImbalanceCount
```

> **口诀**：broker 重启慢 = 索引重建 + leader 切换。**优雅停机 + 限速 rebalance + 分批滚动**三件套缺一不可。

---

### 案例 ⑧ KRaft 下 Controller 切换 → __cluster_metadata 主题的 quorum 抖动

#### 现象

3 节点 KRaft 集群（3 个 controller + 共置 broker）。一个 controller 节点偶发网络抖动 30 秒，期间集群整体 metadata 操作（创建 topic、查询元数据）超时，业务侧创建 consumer 失败。

#### 根因

KRaft controller quorum 是 Raft 协议。3 节点 → quorum size = 2，能容忍 1 节点挂。

但**Raft leader election 期间**：
1. Leader 节点失联 30s 后，followers 触发 election timeout
2. 进入 candidate 状态，发起 vote
3. 一旦发起 election，**整个 metadata log 暂时不能 commit 新条目**
4. 所有依赖 metadata 写入的操作（创建 topic、ISR 变更、ALTER 等）等待
5. 期间任何 broker 的 ISR 变更也累积，故障 controller 恢复后还要补一波

更微妙的是：**如果你的 controller 同时也是 broker**（混合部署，KRaft 默认推荐生产分离）：
- 该节点的 broker 角色处理客户端请求时也会被 election 期间的"metadata 不可见"影响
- 网络抖动期间，连接到该节点的 client 既看不到正确的 metadata，又无法获取新 metadata

ZK 模式下不会有这个问题（因为 ZK 自己有 leader election，但和 broker 解耦），但 ZK 的 election 一旦发生本身就更慢（秒级）。

#### 修复

第一层（部署）：**controller 与 broker 分离**。

```properties
# Controller-only 节点
process.roles=controller
node.id=1
controller.quorum.voters=1@c1:9093,2@c2:9093,3@c3:9093

# Broker-only 节点
process.roles=broker
node.id=10
controller.quorum.voters=1@c1:9093,2@c2:9093,3@c3:9093
```

3 个独立 controller 节点（不跑 broker）+ N 个 broker。controller 资源轻、配置少，故障影响小。

第二层（参数调优）：

```properties
# 缩短 election timeout（默认相对保守）
controller.quorum.election.timeout.ms=2000
controller.quorum.fetch.timeout.ms=2000

# 但太小会让网络抖动频繁触发 election，要测
```

第三层（5 节点 controller quorum）：

```properties
# 大集群推荐 5 个 controller，能容忍 2 个挂
# 单个网络抖动不会触发 election（只要 majority 还在）
controller.quorum.voters=1@c1:9093,2@c2:9093,3@c3:9093,4@c4:9093,5@c5:9093
```

#### 验证

```bash
# 监控 KRaft 状态
kafka.server:type=KafkaServer,name=BrokerState
kafka.controller:type=KafkaController,name=ActiveControllerCount   # 必须 = 1

# Raft 状态
metadata-quorum.sh --bootstrap-server ... describe --status
```

> **口诀**：KRaft 不是免费的"少一个组件"。**生产必须 controller-broker 分离 + 5 节点 quorum + 监控 election 频率**。

---

### 案例 ⑨ Consumer Reblance 风暴 —— "一个消费者下线整个 group 停 30 秒"

#### 现象

业务高峰期，某个 consumer group 的一台 consumer pod 滚动重启。整个 group（共 50 个 consumer，消费 200 个 partition）瞬间 lag 飙升，监控显示**所有 consumer 都停止消费 35 秒**。业务侧延迟尖刺。

#### 根因

默认的 reblance 协议（eager）流程：

1. consumer A 发 LeaveGroup
2. coordinator 触发 reblance，向**所有** consumer 发 GroupUpdate
3. 所有 consumer 收到后，**立即停止 fetch、放弃所有 partition、提交 offset**
4. 重新走 JoinGroup → SyncGroup → 拿到新 partition 分配
5. 重新 fetch

整个过程如果 partition 多、网络不快，可能 30 秒+。期间整个 group 不消费。

更糟的是 reblance 风暴：
- 消费者发 LeaveGroup 后再 JoinGroup（自我重启）
- 老 consumer 还在重 join，新 consumer 来加入
- Coordinator 发现"成员变了"，再触发一次 reblance
- 雪崩

#### 修复

第一层（升级 reblance 协议）：

```properties
# Cooperative Sticky Assignor（2.4+）
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor

# 这个 assignor 做两件事：
# 1. 尽量保持原有分配（sticky）：少 partition 移动，少 offset 重提交
# 2. 协作式 reblance：分两轮，第一轮告诉哪些 partition 要走，没走的继续消费
```

切换有兼容性要求：所有 consumer 必须先升级到支持 cooperative 的版本，再切策略。

第二层（避免不必要的 leave / join）：

```properties
# 区分 session timeout（心跳超时）和 max poll interval
session.timeout.ms=10000             # 默认 45000，调小可以更快发现挂掉
heartbeat.interval.ms=3000           # session 的 1/3
max.poll.interval.ms=300000          # 默认 5min，给 poll 处理时间

# 关键：max.poll.records 不要太大
max.poll.records=100                 # 默认 500，太大可能单次 poll 处理超时
```

第三层（K8s 部署）：

```yaml
# 利用 group.instance.id 实现"静态成员"
env:
  - name: KAFKA_GROUP_INSTANCE_ID
    valueFrom:
      fieldRef:
        fieldPath: metadata.name      # 用 pod name 当 instance id

# Static membership: 客户端断线后 session.timeout 内重连不触发 reblance
# 配套 session.timeout.ms 调到 5min 左右
```

第四层（应用层）：消费者优雅停机。

```java
// SIGTERM 处理
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    consumer.wakeup();
    try { mainThread.join(); } catch (InterruptedException ignore) {}
}));

// 主循环
try {
    while (!stopped) {
        ConsumerRecords<K,V> records = consumer.poll(Duration.ofSeconds(1));
        process(records);
        consumer.commitSync();
    }
} catch (WakeupException e) { /* 正常退出 */ }
finally {
    try { consumer.commitSync(); } finally { consumer.close(); }
}
```

#### 验证

```bash
# 监控 reblance 频率
kafka.consumer:type=consumer-coordinator-metrics,client-id=*,name=rebalance-rate-per-hour

# 单次 reblance 时长
kafka.consumer:type=consumer-coordinator-metrics,name=rebalance-latency-avg

# Group 状态
kafka-consumer-groups.sh --describe --group <group> --members --verbose
```

> **口诀**：reblance 是 consumer group 的"全员停摆"。**Cooperative + Static Membership + 优雅停机** 三件套能把停摆窗口从 30 秒压到秒级。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### Producer 层
- [ ] `enable.idempotence=true`（3.0+ 默认）防止重试导致重复
- [ ] `acks=all` + `retries=Integer.MAX_VALUE` + `delivery.timeout.ms` 显式设置
- [ ] `max.in.flight.requests.per.connection=5`（保留默认；改 1 性能下降，改 >5 影响顺序）
- [ ] `compression.type=lz4` 或 `zstd`（吞吐 + 存储双赢）
- [ ] `linger.ms >= 5`，`batch.size >= 32KB`（攒批换吞吐）
- [ ] 单条消息 < 100KB，超大走 claim-check 模式
- [ ] Producer 配 metric 上报：`record-error-rate`、`record-retry-rate`

### Broker 层
- [ ] `num.replica.fetchers = cores / 2`（万兆网卡场景）
- [ ] `log.segment.bytes` 按消息密度调整（高频小消息用 256MB，低频大消息用 1GB）
- [ ] `log.retention.bytes` 按磁盘容量配（不要靠时间，靠空间）
- [ ] `auto.create.topics.enable=false`，topic 必须显式申请
- [ ] `unclean.leader.election.enable=false`（生产强制，避免选出未同步的 leader 丢数据）
- [ ] `min.insync.replicas` 严格小于副本数（3 副本配 2，5 副本配 3）
- [ ] JVM heap ≤ 物理内存 1/4，剩下留给 PageCache
- [ ] G1GC + `-XX:G1HeapRegionSize=32m` + `-XX:MaxGCPauseMillis=20`
- [ ] `ulimit -n` 调到至少 100000，`fs.file-max` 同步调

### KRaft 层
- [ ] Controller 与 Broker 进程分离（生产环境强制）
- [ ] Controller 节点 3 或 5 个（不要 2 或 4）
- [ ] `__cluster_metadata` topic 监控写入速率，异常飙升要排查
- [ ] `controller.quorum.fetch.timeout.ms` 不要轻易调小

### Consumer 层
- [ ] `partition.assignment.strategy=CooperativeStickyAssignor`
- [ ] `enable.auto.commit=false`，手动提交 offset
- [ ] `max.poll.records` 按单条处理时间反推（< max.poll.interval.ms / 处理时长 / 安全系数）
- [ ] `group.instance.id` 设静态成员（K8s 用 pod name）
- [ ] `session.timeout.ms` 与 K8s `terminationGracePeriodSeconds` 协调
- [ ] 优雅停机：`consumer.wakeup() → commitSync → close`
- [ ] 消费幂等设计（基于 message key + 业务唯一性）

### 监控层
- [ ] `UnderReplicatedPartitions` = 0
- [ ] `UnderMinIsrPartitionCount` = 0
- [ ] `OfflinePartitionsCount` = 0
- [ ] `ActiveControllerCount` = 1
- [ ] Consumer lag 监控（按 partition 维度）
- [ ] `BytesInPerSec` / `BytesOutPerSec` 趋势告警
- [ ] PageCache 命中率（`vmstat 1` 的 `bi` 字段）
- [ ] Disk %util、磁盘空间剩余 > 30%
- [ ] FD 占用 / 物理内存余量 / GC 时间

---

## 6. 还能往下挖什么

### 替代存储模型
- **Apache Pulsar** 的分层架构：BookKeeper 做 segment 存储 + BookKeeper 自己也是分布式日志。Topic 分配到 broker 是动态的（无状态 broker），扩缩容比 Kafka 弹性。配 Tiered Storage 可以把冷数据下沉 S3。
- **AutoMQ / WarpStream**：把 Kafka 的存储层替换成 S3。计算无状态、按用量付费、跨 AZ 自动复制。代价是 S3 的延迟（写入 100ms+），适合"大吞吐 + 容忍 P99 延迟"的场景，不适合实时交易。
- **Pravega**：CNCF 项目，类似 Pulsar 但更专注流处理语义（exactly-once、自动 scale）。

### Iceberg / Lake 替代 MQ
- **Iceberg + Flink**：把"流"变成"对 Iceberg 表的微批 append"。下游消费方式从 Kafka consumer 变成 Iceberg snapshot reader。优点：和数仓打通；缺点：实时性下降到分钟级。
- **WarpStream + Iceberg**：S3 后端的 MQ 直接以 Iceberg 格式落数据，省去 ETL 一步。

### Kafka 自身演进
- **KIP-405 Tiered Storage**（3.6 GA）：本地存最近 X 小时数据，老数据下沉 S3 / HDFS。Broker 层透明，consumer 不感知。
  - 陷阱：冷读延迟高，consumer 落后到 tiered 段时延迟可能从 ms 跳到秒。
  - 配套要重新评估 PageCache 大小（变小后热数据占比变高）。
- **KIP-848 New Consumer Group Protocol**：把 group 协议从客户端推算 → 服务端集中分配。彻底解决 reblance 风暴。3.7 引入，3.9 GA。
- **Kafka Streams**：本地状态用 RocksDB，changelog topic 做容错。和 Flink 比起来更轻量但状态管理弱（RocksDB 单机），适合中小规模实时处理。

### EoS（exactly-once semantics）的两阶段提交
- Producer 开 `transactional.id` + `transaction.timeout.ms`
- Broker 维护 transaction coordinator，事务状态存 `__transaction_state` topic
- 事务提交：`addPartitionsToTxn → produce → commitTxn`，commitTxn 写 commit marker
- Consumer 配 `isolation.level=read_committed` 才能看到完整事务
- 性能代价：单事务 30%+ 吞吐损耗，事务越大代价越高

### ZK → KRaft 迁移实操
- 3.4 引入 dual-write 模式：ZK 与 KRaft 并存
- 迁移流程：ZK → ZK+KRaft 双写 → KRaft only → 摘掉 ZK
- 核心风险：迁移期间 broker 重启不能交叉版本，operator 工具（如 Strimzi）支持比手动稳妥
- 业界经验：先在测试集群跑一遍完整流程，生产建议等到 4.0 后再做（届时 ZK 模式删除）

### 性能调优深水区
- `socket.send.buffer.bytes` / `socket.receive.buffer.bytes` = -1（用 OS 默认 + auto-tune）
- `num.network.threads` / `num.io.threads` 单独调
- `queued.max.requests` 防止 overload
- TCP keepalive、quic 探索

---

## 7. 引用

- Apache Kafka 官方文档：[https://kafka.apache.org/documentation/](https://kafka.apache.org/documentation/)
- Kafka 源码（Apache 3.6）：[github.com/apache/kafka](https://github.com/apache/kafka)
- KIP-500: Replace ZooKeeper with a Self-Managed Metadata Quorum
- KIP-405: Kafka Tiered Storage
- KIP-848: New Consumer Group Protocol
- Jay Kreps, *I Heart Logs*（O'Reilly, 2014）—— Kafka 设计哲学的源头
- 《Kafka 权威指南》（第 2 版）—— Confluent 官方
- LinkedIn Engineering Blog: *Kafka at scale*
- Confluent Blog: *Hands-free Kafka Replication*（详解 ISR 设计）
- Cloudflare Engineering: *Squeezing the firehose: getting the most from Kafka compression*
- 阿里云消息队列 Kafka 版最佳实践
- AutoMQ 论文与博客：S3 后端 Kafka 的设计权衡

---

> 配套：[消息队列主题导览](/article/mq/) ｜ [线程池生产实战](/article/java/threadpool) ｜ [Linux 与性能工程](/article/linux-perf/)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
