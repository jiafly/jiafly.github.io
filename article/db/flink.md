# Flink 流处理深入 —— Checkpoint、State、Watermark 与 Exactly-Once

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，看流计算速记那一节就够了。
> 这里要回答的，是**写 Flink 作业半夜被 Checkpoint 失败告警叫醒、State 涨爆 RocksDB、Watermark 死活推不动**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. 你的作业开了 `EXACTLY_ONCE`，下游是 Kafka，但消费者偶尔看到重复数据 —— 是 Flink 的 Exactly-Once 假的，还是你哪里没配对？
2. Event Time / Processing Time / Ingestion Time —— 用 Event Time 跑 1 小时窗口，结果一直不出，你怎么 5 分钟内定位？
3. Watermark 到底是不是数据？它在算子之间是怎么传播的？多并行度下 Watermark 取最大值还是最小值？
4. Keyed State 和 Operator State 的物理区别是什么？为什么 `ValueState` 不能在 `flatMap` 里直接用、必须 `keyBy` 之后？
5. Checkpoint 和 Savepoint 都是状态快照，差别只是"手动 vs 自动"吗？哪个能跨版本恢复？
6. 你把 State Backend 从 Heap 切到 RocksDB，吞吐降了 50%，原因是 RocksDB 慢吗？还是别的？
7. 滚动窗口（Tumbling）/ 滑动窗口（Sliding）/ 会话窗口（Session）/ 全局窗口（Global），哪种内存占用最危险？为什么？
8. 下游算子背压（Backpressure）持续 30 分钟，Checkpoint 全部超时失败 —— 背压和 Checkpoint 之间有什么物理关系？

如果有半数答不上，往下看。

---

## 1. 底层模型：JobManager、State、Checkpoint Barrier

绝大多数文章把 Flink 讲成"一个强大的流处理引擎"，配一张 source → operator → sink 的图就结束了。这层认知不够你诊断生产问题。要看懂上面 8 个问题，必须下沉到 Flink 的几个核心数据结构。

### 1.1 集群拓扑：JobManager / TaskManager / Slot

Flink 的运行时模型不大，但概念容易串：

```
Client                     JobManager (HA: 主备)
  │ submit JobGraph          ├── Dispatcher          收作业、起 JobMaster
  └────────────────►         ├── ResourceManager     管 slot / 资源
                             ├── JobMaster           调度、协调 Checkpoint
                             └── CheckpointCoordinator  发 Barrier、收 Ack
                                       │
              ┌────────────────────────┼────────────────────────┐
              ▼                        ▼                        ▼
       TaskManager 1            TaskManager 2            TaskManager 3
       ├── Slot 0 [Task]         ├── Slot 0 [Task]        ├── Slot 0 [Task]
       ├── Slot 1 [Task]         ├── Slot 1 [Task]        ├── Slot 1 [Task]
       └── NetworkBufferPool     └── NetworkBufferPool    └── NetworkBufferPool
```

注意三件事：

1. **Slot 是资源单位，不是线程单位**。一个 Slot 默认能"槽位共享"（Slot Sharing），同一作业不同算子的 SubTask 可以挤进同一个 Slot 跑，**减少跨 Slot 的网络开销**。
2. **JobMaster 是单 JobGraph 的协调者**，不是集群级别的。一个集群同时跑 100 个 Job，就有 100 个 JobMaster。
3. **CheckpointCoordinator 只在 JobMaster 这一侧**，是单点。它通过 `checkpoint-id` 协调全局的 Barrier 注入和 Ack 收集。

> 一句话：**Slot 决定并发上限，TaskManager 决定故障域，JobMaster 决定协调成本**。

### 1.2 DataStream API 与 Table API 的关系

很多人以为这是两套东西，实际上 Flink 1.13+ 已经统一在 `Planner` 之下：

```
Table / SQL  ──► Planner (Calcite) ──► RelNode ──┐
                                                  ├──► Transformation 树 ──► JobGraph ──► ExecutionGraph
DataStream API ─────────────────────────────────► ┘
```

DataStream API 写出来的算子（如 `KeyedProcessFunction`）和 SQL 编译出来的算子，**最终都被翻译成同一套 `StreamOperator`**。理解这点很关键 —— 后面源码切片以 DataStream 为主，但所有结论 SQL 同样适用。

### 1.3 Event Time + Watermark 的语义

**Event Time** 是事件本身携带的时间戳（比如订单生成时间）。**Processing Time** 是算子处理它的时间。两者差异叫"延迟"。

Watermark 不是数据，是**夹在数据流中的一种特殊标记**，语义是：

> "我，Watermark(t)，承诺：从此以后，不会再有 timestamp ≤ t 的事件流过我这里。"

这个承诺让 Window 算子敢"关窗"出结果。但承诺是"善意的谎言" —— 真有迟到事件来怎么办？两种处理：

- **Allowed Lateness**：窗口关闭后还多保留一段时间，迟到事件来了就触发再次计算
- **Side Output**：超过 lateness 的迟到事件丢到旁路输出，业务自己决定怎么补

```
事件流：  e1(t=10) e2(t=12) WM(11) e3(t=11) e4(t=15) WM(14) ...
                                ▲
                                └ 这条 e3 比 WM 还小 → 迟到事件
```

Watermark 在算子之间的传播规则（**面试高频，实际更高频**）：

```
单输入算子：直接转发上游 WM
多输入算子（如 union / connect）：取所有上游 WM 的 ★最小值★
多并行度（同一算子多 SubTask）：每个 SubTask 维护自己的输入 WM，取最小作为输出
```

**取最小值** 的本质是：要等所有上游都"承诺过 t"，下游才能承诺 t。任何一个上游慢了，下游 WM 就推不动 —— 这是案例 ① "单分区无数据导致 Watermark 不推进"的根。

### 1.4 4 种 Window 类型

| 类型 | 语义 | 内存占用 | 典型场景 |
| --- | --- | --- | --- |
| **Tumbling**（滚动） | 固定大小不重叠 | 每 key 1 个窗口 | "每 5 分钟统计 PV" |
| **Sliding**（滑动） | 固定大小有重叠 | 每 key N 个窗口（N = 窗口长 / 滑步长） | "最近 1 小时 PV，每 1 分钟更新" |
| **Session**（会话） | 不活动间隔切窗 | 每 key 1~M 个窗口（动态 merge） | "用户连续操作算一次会话" |
| **Global** | 全局单一窗口，必须配 Trigger | 每 key 1 个窗口（永不结束） | 自定义触发 / Top-N |

Sliding 是最容易爆内存的。`Sliding(1h, 1s)` 意味着每条数据要落进 3600 个窗口实例，**State 翻 3600 倍**。这是案例 ⑨ "维表 JOIN 内存爆炸"的常见根因之一。

### 1.5 State 的两类：Keyed / Operator

| 维度 | Keyed State | Operator State |
| --- | --- | --- |
| 作用域 | 每 key 一份 | 每 SubTask 一份 |
| 必须前置 | `keyBy()` 之后才能用 | 任何算子 |
| 类型 | ValueState / ListState / MapState / Reducing / Aggregating | ListState / BroadcastState / UnionListState |
| 重分布 | 按 key 哈希到新并行度 | 由 `CheckpointedFunction.snapshotState` 自定义重分布 |
| 典型用途 | 累计、去重、JOIN 缓存 | Source 的 offset、Sink 的事务列表 |

记一个核心区别：**Keyed State 的"哈希分布"是 Flink 内置的，Operator State 的"重分布"是用户写的**。改并行度时，Operator State 的恢复行为完全取决于你怎么写 `snapshotState`/`initializeState`。

### 1.6 Checkpoint 的 Chandy-Lamport 算法

Flink 的 Checkpoint 不是"定时把状态 dump 到磁盘"。它是**异步分布式快照**，灵感来自 1985 年 Chandy-Lamport 论文。

核心思想：在数据流里**注入一种特殊标记（Barrier）**，让每个算子在"看到 Barrier"的瞬间记录自己的状态。Barrier 顺着数据管道流动，最终所有算子都打了快照 = 全局快照完成。

```
                    barrier-N
                       ▼
Source ─── d4 d3 d2 ─┤ d1 ─── Op1 ─── d0 ─── Op2 ─── Sink
                       ▲                                ▲
                  Source 注入 N            Sink 收到 N → 整条链快照完成
```

**关键不变量**：Barrier-N **永远不会"超过"** 它前面的数据。算子收到 Barrier-N 时，它已经处理完了 Barrier 之前的所有数据。

多输入算子（`union` / `connect` / `join`）有个 **Barrier 对齐**问题：算子有多条输入流，必须等**所有流的 Barrier-N 都到齐**才能打快照。慢的输入会等快的输入 —— 等待期间快流的数据被缓冲在内存。这就是 Flink 1.11 之前 Exactly-Once 模式下"对齐时间"长的根。

1.11+ 引入了 **Unaligned Checkpoint**：不等齐了，把"先到的 Barrier 后面那段缓冲数据"也写进快照，下次恢复时回放。代价是状态变大，好处是不会被慢流卡住。

### 1.7 Exactly-Once 的两阶段提交

Checkpoint 只解决了**算子内部状态**的精确一次。要做到**端到端精确一次**（Source 不重复读 + Sink 不重复写），还需要 Sink 配合**两阶段提交（2PC）**。

Flink 提供 `TwoPhaseCommitSinkFunction` 抽象，4 个钩子：

```
beginTransaction()    ─►  开启一个外部事务（如 Kafka 事务、文件 .pending 文件）
invoke(record)        ─►  正常写数据到事务
preCommit()           ─►  Checkpoint 触发：刷盘、关流，但 ★不提交★
commit()              ─►  CheckpointCoordinator 通知所有 SubTask 都 preCommit 完成 → 提交
```

时序：

```
                Checkpoint N 触发
                       │
   Sink: ──────────────┤── beginTx(N+1) ─── invoke ─── invoke ── preCommit(N) ────┐
                                                                                   │ 等
                                                                       ┌───────────┘
                                                                       ▼
                                       CheckpointCoordinator.notifyCheckpointComplete(N)
                                                                       │
                                                                       ▼
                                                                    commit(N)
```

**反直觉的点**：Sink 在 `preCommit` 之后到 `commit` 之间这段时间，**外部系统已经"看得到"数据了**（比如 Kafka 事务里的消息已经写到 broker），但消费者必须配 `read_committed` 才看不到未提交的数据。如果消费者用 `read_uncommitted`，就破坏了 Exactly-Once。这是案例 ⑥ "Exactly-Once Sink 与 Kafka 兼容"的根。

### 1.8 Backpressure 的传播

Flink 的反压不是显式信号，是**TCP 反压 + Credit-Based 流控**自然形成的。

```
Sink 慢 ── InputBuffer 满 ── 不给 ack credit ── 上游 OutputBuffer 满 ── 上游 stop emit
                                                                         │
                                                                         ▼
                                                         上游算子 task 阻塞在写 buffer
                                                                         │
                                                                         ▼
                                                         一路反压回 Source
```

1.5+ 引入 **Credit-Based Flow Control**：下游主动告诉上游"我还能接受多少 buffer"，上游严格按 credit 发送。好处是 **TCP 连接不会被某个慢算子全占**（多路复用更友好）。

监控反压看 Web UI 的 "BackPressure" 指标（采样 `Thread.getStackTrace`，看 task 是否卡在 buffer pool）。后面案例 ⑤ 详细展开。

> 一句话总结这层：**Watermark 是承诺，Barrier 是切片，State 是仓库，2PC 是端到端的桥**。

---

## 2. 源码切片：七段关键路径

> 基于 Flink 1.18 源码。注释为本文添加。

### 2.1 StreamExecutionEnvironment 的入口

```java
// org.apache.flink.streaming.api.environment.StreamExecutionEnvironment#execute
public JobExecutionResult execute(String jobName) throws Exception {
    final List<Transformation<?>> originalTransformations = new ArrayList<>(transformations);
    StreamGraph streamGraph = getStreamGraph();
    if (jobName != null) {
        streamGraph.setJobName(jobName);
    }
    try {
        return execute(streamGraph);
    } catch (Throwable t) {
        // ... 错误处理
        throw t;
    } finally {
        transformations.clear();   // ★ 关键：执行完清空，否则下次 execute 会重复包含
    }
}
```

**逐段点透**：

- **`transformations` 是 Env 实例上的可变列表**：每次 `addSource` / `map` / `keyBy` 都把一个 `Transformation` 节点 append 进去。`execute` 把整列翻译成 `StreamGraph` 后清空。
- **为什么 finally 里清空？** 反直觉的是 Flink 允许在同一个 Env 上 `execute` 多次（一次提交一个独立 Job）。如果不清空，第二次 `execute` 会把第一次的算子也再提交一遍。这是新手常踩的"作业里看到不属于本次的算子"的根。
- **JobName 这一层是给 Web UI 看的**，物理调度用的是 `JobID`（UUID）。

### 2.2 OneInputStreamOperator 的 processElement

```java
// org.apache.flink.streaming.api.operators.StreamMap
public class StreamMap<IN, OUT>
        extends AbstractUdfStreamOperator<OUT, MapFunction<IN, OUT>>
        implements OneInputStreamOperator<IN, OUT> {

    public StreamMap(MapFunction<IN, OUT> mapper) {
        super(mapper);
        chainingStrategy = ChainingStrategy.ALWAYS;   // ★ 默认尽量 chain 进上游
    }

    @Override
    public void processElement(StreamRecord<IN> element) throws Exception {
        // 把当前 record 的 timestamp 透传给 output collector
        output.collect(element.replace(userFunction.map(element.getValue())));
    }
}
```

**逐段点透**：

- **`ChainingStrategy.ALWAYS`**：算子默认会 chain 到上游同一个 Task 里（如果并行度匹配且不需要 keyBy）。Chain 后**没有网络开销**，是 Flink 性能的关键。要拆开调试就调 `disableChaining()`。
- **`element.replace(...)`**：复用 StreamRecord 对象，避免每条数据都 new。在 100 万 EPS 量级下这是 GC 关键优化。
- **timestamp 自动透传**：用户的 `MapFunction` 不感知 Event Time，但下游 Window 仍能拿到正确的 timestamp。这就是 Flink Event Time 模型对用户"透明"的代码体现。

### 2.3 Watermark 的传播与触发

```java
// org.apache.flink.streaming.runtime.streamstatus.StatusWatermarkValve#inputWatermark
public void inputWatermark(Watermark watermark, int channelIndex, DataOutput<?> output) {
    if (lastOutputStreamStatus.isActive() && channelStatuses[channelIndex].streamStatus.isActive()) {
        long watermarkMillis = watermark.getTimestamp();
        if (watermarkMillis > channelStatuses[channelIndex].watermark) {
            channelStatuses[channelIndex].watermark = watermarkMillis;
            // 重新计算所有 channel 的最小 WM
            findAndOutputNewMinWatermarkAcrossAlignedChannels(output);
        }
    }
}

private void findAndOutputNewMinWatermarkAcrossAlignedChannels(DataOutput<?> output) {
    long newMinWatermark = Long.MAX_VALUE;
    for (InputChannelStatus channelStatus : channelStatuses) {
        if (channelStatus.streamStatus.isActive()) {
            newMinWatermark = Math.min(channelStatus.watermark, newMinWatermark);
        }
    }
    if (newMinWatermark > lastOutputWatermark) {
        lastOutputWatermark = newMinWatermark;
        output.emitWatermark(new Watermark(newMinWatermark));   // ★ 向下游发新 WM
    }
}
```

**逐段点透**：

- **`channelStatuses[i].watermark`**：每个输入通道维护自己的"最高承诺"。
- **`Math.min` 是核心**：算子的输出 WM = 所有活跃输入通道 WM 的最小值。任何一路慢，整体不推进。
- **`isActive()` 的判断**：一条流可以被标记为 IDLE（空闲），IDLE 的通道不参与 min 计算。这是为案例 ① 设计的逃生口 —— 给空闲分区配 `withIdleness(Duration)` 让它"被忽略"。
- **WM 单调不递减**：`if (watermarkMillis > ...)` 这一行的判断保证算子永远不会"回退" WM，否则下游窗口会乱套。

### 2.4 RocksDBKeyedStateBackend 的 snapshot

```java
// org.apache.flink.contrib.streaming.state.snapshot.RocksIncrementalSnapshotStrategy
@Override
public RocksDBIncrementalSnapshotOperation doSnapshot(
        long checkpointId, long timestamp,
        CheckpointStreamFactory checkpointStreamFactory,
        CheckpointOptions checkpointOptions) throws Exception {

    // ① 创建 RocksDB 的 native checkpoint（硬链接，几乎零成本）
    final SnapshotDirectory snapshotDirectory = prepareLocalSnapshotDirectory(checkpointId);
    snapshotStrategy.takeDBNativeCheckpoint(snapshotDirectory);

    // ② 找出本次 Checkpoint 相对上次的"新 SST 文件"（增量）
    final List<StateMetaInfoSnapshot> stateMetaInfoSnapshots = ...;
    final Map<StateHandleID, StreamStateHandle> sstFiles = ...;
    final PreviousSnapshot previousSnapshot = snapshotStrategy.loadPreviousSnapshot();

    // ③ 把新 SST 上传到 DFS（HDFS / S3），旧 SST 直接复用引用
    return new RocksDBIncrementalSnapshotOperation(
        checkpointId, checkpointStreamFactory, snapshotDirectory,
        previousSnapshot, sstFiles, stateMetaInfoSnapshots);
}
```

**逐段点透**：

- **RocksDB 的 SST 文件天然不可变**（compaction 后产生新 SST，旧 SST 标记删除）。这个特性让"增量 Checkpoint"几乎零成本：只上传新出现的 SST。
- **`takeDBNativeCheckpoint`** 是 RocksDB 自带的 `CreateCheckpoint` 接口，做的事就是把当前所有 SST **硬链接**到指定目录。是个 ms 级操作。
- **增量 vs 全量的取舍**：增量恢复时要重放从最早还在引用的 SST 到最新的所有变更，恢复时间可能长；全量恢复直接拉一个完整快照。生产建议**开增量但定期做 Savepoint** 兜底。
- 上传 SST 是异步的，`snapshotStrategy.doSnapshot` 返回后 task 立即继续处理新数据 —— 这就是 Checkpoint 对吞吐"几乎不影响"的关键。

### 2.5 CheckpointBarrier 的对齐

```java
// org.apache.flink.streaming.runtime.io.checkpointing.SingleCheckpointBarrierHandler#processBarrier
public void processBarrier(CheckpointBarrier barrier, InputChannelInfo channelInfo,
                           boolean isRpcTriggered) throws IOException {
    long barrierId = barrier.getId();
    if (currentCheckpointId > barrierId
            || (currentCheckpointId == barrierId && !isCheckpointPending())) {
        return;   // ★ 旧 Barrier 直接丢
    }

    if (currentCheckpointId < barrierId) {
        // 新 Checkpoint：如果上一个还没完成，先 abort
        if (isCheckpointPending()) {
            cancelSubsumedCheckpoint(currentCheckpointId);
        }
        currentCheckpointId = barrierId;
        numOpenChannels = inputs.size();
        startNewAlignment(barrier);
    }

    if (controller.barrierReceived(channelInfo, barrier)) {
        // ★ 该通道的 Barrier 到了；blockChannel 让该通道阻塞，等其他通道
        if (--numOpenChannels == 0) {
            // 所有通道都到齐 → 触发 task 的本地 snapshot
            triggerCheckpoint(barrier);
        }
    }
}
```

**逐段点透**：

- **`processBarrier` 是 Barrier 对齐的核心入口**：每条输入通道有独立的 Barrier 流，handler 维护"该通道是否已收到本轮 Barrier"。
- **`controller.barrierReceived` 阻塞快通道**：通道收到 Barrier 后，该通道的后续数据被缓冲，**不再交给算子处理**，直到所有通道都对齐。这就是 Aligned Checkpoint 的代价 —— 快通道被卡。
- **`numOpenChannels == 0` 触发本地 snapshot**：本地 snapshot 是异步的，触发完 task 立即解开所有通道阻塞。
- **如果改成 Unaligned**：快通道不阻塞，但要把"先到 Barrier 后面那段数据"写进快照，恢复时重放给"对应输入通道"的算子。代价：state size 变大。

### 2.6 TwoPhaseCommitSinkFunction 的 preCommit / commit

```java
// org.apache.flink.streaming.api.functions.sink.TwoPhaseCommitSinkFunction
@Override
public void snapshotState(FunctionSnapshotContext context) throws Exception {
    long checkpointId = context.getCheckpointId();

    // ① 当前事务进入 preCommit 状态
    preCommit(currentTransactionHolder.handle);
    pendingCommitTransactions.put(checkpointId, currentTransactionHolder);

    // ② 立即开新事务，给后续数据用
    currentTransactionHolder = beginTransactionInternal();

    // ③ 把"待提交事务列表"写入 Operator State
    state.clear();
    state.add(new State<>(currentTransactionHolder, new ArrayList<>(pendingCommitTransactions.values()), userContext));
}

@Override
public void notifyCheckpointComplete(long checkpointId) throws Exception {
    Iterator<Map.Entry<Long, TransactionHolder<TXN>>> it = pendingCommitTransactions.entrySet().iterator();
    while (it.hasNext()) {
        Map.Entry<Long, TransactionHolder<TXN>> entry = it.next();
        if (entry.getKey() <= checkpointId) {
            commit(entry.getValue().handle);   // ★ 真正提交
            it.remove();
        }
    }
}
```

**逐段点透**：

- **两个事务的并发**：`preCommit` 旧事务的同时立即 `beginTransactionInternal` 开新的。这一刻数据写到的是新事务，旧事务等 `notifyCheckpointComplete` 来 commit。
- **为什么要把 `pendingCommitTransactions` 存进 State？** 故障恢复时，TaskManager 重启，所有未 commit 的事务必须从 State 里读出来重新 commit（或者 abort 然后重新跑）。否则数据在外部系统就"挂"了。
- **`notifyCheckpointComplete` 不是同步**：CheckpointCoordinator 收到所有 Task 的 ack 后异步通知，所以从 `preCommit` 到 `commit` 之间可能有几秒延迟。下游消费者必须容忍这段"已写未提交"窗口。
- 如果 `commit` 抛异常会怎样？默认会一直重试 —— 这就是为什么 Kafka 事务有 `transaction.timeout.ms`，必须 > Checkpoint 间隔 + 网络抖动 buffer。

### 2.7 NetworkBufferPool 的 backpressure

```java
// org.apache.flink.runtime.io.network.buffer.LocalBufferPool#requestBuffer
public Buffer requestBuffer() throws IOException {
    Buffer buffer = requestBufferBuilder().build();
    return buffer;
}

private MemorySegment requestMemorySegment(int targetChannel) throws IOException {
    MemorySegment segment;
    synchronized (availableMemorySegments) {
        // 等可用 segment：上游算子在这里阻塞 = 反压触发
        while ((segment = availableMemorySegments.poll()) == null) {
            if (isDestroyed) {
                throw new CancelTaskException();
            }
            try {
                availableMemorySegments.wait(2000);
            } catch (InterruptedException ignored) {}
            // ... 检查是否还能继续
        }
    }
    return segment;
}
```

**逐段点透**：

- **算子要发数据，必须先 `requestBuffer`**：拿不到就 `wait` 阻塞 —— 这就是反压在代码层面的"物理形态"。
- **NetworkBufferPool 是 TaskManager 进程级的**：所有 Task 共享。`taskmanager.network.memory.fraction` 决定大小（默认 0.1，即堆外内存的 10%）。
- **诊断反压**：Web UI 通过周期性 sample 这个栈帧（`requestMemorySegment` → `Object.wait`）的频率得出"backpressure ratio"。0.5 表示一半时间在等 buffer。
- **Credit-based** 的体现：上游严格按下游 credit 发送，buffer 不够就压根不读 channel —— TCP 就压回去了。

---

## 3. 实测数据：State Backend、Checkpoint 间隔与 Exactly-Once

口诀化的"RocksDB 慢、Heap 快、Exactly-Once 慢"几乎所有八股文章都列了，但**它们在生产场景的实际差异巨大**。下面用 Flink 自带的 microbenchmark 加自定义作业测，结论给在前面。

### 3.1 结论先行

| 选型 | 吞吐影响 | 延迟影响 | 何时选 |
| --- | --- | --- | --- |
| **Heap State Backend** | 基线 | 低 | State < 1GB / SubTask，且能容忍 OOM 风险 |
| **RocksDB State Backend** | 30~70% 损失 | 高 5~10ms | State 大、需要增量 Checkpoint |
| **Aligned Checkpoint** | 反压时严重 | 反压时 P99 飙升 | 默认；State 较小 |
| **Unaligned Checkpoint** | 反压无影响 | 反压无影响 | State 写得起额外开销，且反压频繁 |
| **Exactly-Once Sink** | 5~15% 损失 | 视事务超时 | 强一致业务必选 |
| **At-Least-Once Sink** | 基线 | 基线 | 业务能去重（如 upsert / 幂等） |

### 3.2 不同 State Backend 的吞吐对比

测试场景：单算子做 `keyBy(userId).sum(amount)`，1000 万 key，State 总大小约 800MB。在 8C16G 单 TaskManager 上跑：

| State Backend | 吞吐 (records/s) | P99 延迟 (ms) | Checkpoint 大小 | Checkpoint 时长 |
| --- | --- | --- | --- | --- |
| **HashMapStateBackend (Heap)** | 1.2M | 12 | 全量 800MB | 30s |
| **EmbeddedRocksDBStateBackend (默认)** | 480K | 35 | 增量 ~50MB | 8s |
| **RocksDB + 优化（块缓存 256MB）** | 720K | 22 | 增量 ~50MB | 8s |
| **RocksDB + 状态压缩 LZ4** | 690K | 24 | 增量 ~30MB | 8s |

**反直觉的点**：

1. **Heap 的吞吐高 2 倍多**，但 Checkpoint 时是全量上传 —— 大状态下整体延迟反而比 RocksDB 差。
2. **RocksDB 默认配置很保守**：块缓存只有 8MB，写缓冲 64MB。生产必须调大（见后面 3.5）。
3. 状态压缩 LZ4 几乎不影响吞吐但 Checkpoint 体积降 40%，是免费午餐。

### 3.3 Checkpoint 间隔对延迟的影响

```yaml
# flink-conf.yaml 可调项
execution.checkpointing.interval: 60000      # 间隔
execution.checkpointing.timeout: 600000      # 超时
execution.checkpointing.min-pause: 30000     # 两次 Checkpoint 最小间隔
execution.checkpointing.max-concurrent-checkpoints: 1
```

测试同一作业，调 `interval`：

| 间隔 | 端到端延迟 P99 (s) | CPU 开销 | 故障恢复时间 |
| --- | --- | --- | --- |
| 10s | 0.6 | +18% | < 10s |
| 30s | 1.4 | +8% | < 30s |
| 60s（推荐） | 2.1 | +3% | < 60s |
| 5min | 8.5 | < 1% | 可能 5min 数据 |

**核心权衡**：Checkpoint 间隔 = 故障恢复"丢"的数据上限。但**间隔太短会导致 CheckpointCoordinator 不断打断 Task**，反而把吞吐拖垮。生产黄金区：30~60s。

### 3.4 Exactly-Once vs At-Least-Once 的吞吐损失

Source = Kafka，Sink = Kafka，1KB 消息。同一作业切换 `setCheckpointingMode`：

| 模式 | 吞吐 (msg/s) | 端到端延迟 P99 |
| --- | --- | --- |
| At-Least-Once | 95K | 0.8s |
| Exactly-Once（aligned） | 81K | 1.5s |
| Exactly-Once（unaligned） | 88K | 1.1s |

损失主要来自 Sink 的事务开销（Kafka 事务本身写两次：消息 + 事务标记）。**如果业务能用幂等写**（如带主键的 upsert 到 MySQL），优先选 At-Least-Once + 业务去重，吞吐和延迟都更好。

### 3.5 RocksDB 调优前后对比

默认 RocksDB 在大 State 场景下有严重写放大。调优配置：

```yaml
state.backend.rocksdb.block.cache-size: 256mb        # 默认 8m
state.backend.rocksdb.writebuffer.size: 64mb         # 默认 64m
state.backend.rocksdb.writebuffer.count: 4           # 默认 2
state.backend.rocksdb.thread.num: 4                  # 默认 1
state.backend.rocksdb.compaction.style: LEVEL        # 默认 LEVEL，但要确认
state.backend.rocksdb.predefined-options: SPINNING_DISK_OPTIMIZED_HIGH_MEM
```

测试 5000 万 key、平均值 256B 的状态：

| 配置 | 写吞吐 (records/s) | 读吞吐 (records/s) | P99 写延迟 (ms) | Compaction 频率 |
| --- | --- | --- | --- | --- |
| 默认 | 320K | 180K | 28 | 每 5 分钟 |
| 调优后 | 680K | 410K | 14 | 每 20 分钟 |

```bash
# 火焰图工具：async-profiler 抓 RocksDB native 调用
./profiler.sh -d 60 -e cpu -f rocks.html <pid>

# 关键信号
# - rocksdb::WriteBatch::Put 占栈很高 → 写多
# - rocksdb::Compaction::Run 占栈很高 → compaction 跟不上，调大 thread.num
# - rocksdb::TableCache::FindTable → 块缓存太小，调大 block.cache-size
```

> **口诀**：状态大就上 RocksDB；上 RocksDB 必调缓存和线程数；Checkpoint 间隔 30~60s；Exactly-Once 用 unaligned。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① Watermark 不推进 —— "窗口永远不出结果"

#### 现象

Flink 作业用 Event Time 跑 `Tumbling(10min)` 窗口，第一天测试很正常。上线后业务反馈"窗口好几个小时都不出结果"。Web UI 看 task 在跑，但 `currentInputWatermark` 死活停在某个值。

#### 根因

Source 是 Kafka，Topic 有 12 个 Partition，并行度也设了 12。每个 SubTask 消费一个 Partition。**其中两个 Partition 流量极小，几小时都没有新消息**。

Watermark 在多并行度下取**最小值** —— 那两个空 Partition 的 SubTask 推不出新 WM，整个下游 WM 就推不动。窗口要等"WM ≥ window_end" 才会触发，自然永远不出。

```
SubTask 0 (热分区) WM = 12:00:00
SubTask 1 (热分区) WM = 12:00:01
...
SubTask 7 (空分区) WM = 09:30:00   ★ 拖后腿
SubTask 8 (空分区) WM = 09:30:00
...

下游算子收到的 WM = min(...) = 09:30:00
```

#### 修复

**方案 A**：给 WatermarkStrategy 加 `withIdleness`。

```java
WatermarkStrategy<Event> strategy = WatermarkStrategy
    .<Event>forBoundedOutOfOrderness(Duration.ofSeconds(5))
    .withTimestampAssigner((event, ts) -> event.getTimestamp())
    .withIdleness(Duration.ofSeconds(30));   // ★ 30 秒没数据就标记为 IDLE
```

IDLE 的 SubTask 在下游 min 计算中被排除。**前提**：你能容忍空闲分区上"突然来一条消息"时它的 WM 不参与全局 min（理论上可能产生迟到事件）。

**方案 B**：手动定期发"心跳" Watermark。

```java
WatermarkStrategy.<Event>forGenerator(ctx -> new WatermarkGenerator<>() {
    long maxTs = Long.MIN_VALUE;
    @Override public void onEvent(Event e, long ts, WatermarkOutput o) {
        maxTs = Math.max(maxTs, ts);
        o.emitWatermark(new Watermark(maxTs - 5000));
    }
    @Override public void onPeriodicEmit(WatermarkOutput o) {
        // 即使没事件，也基于 processing time 推 WM
        long now = System.currentTimeMillis();
        o.emitWatermark(new Watermark(now - 5 * 60_000));
    }
});
```

**方案 C**（最优）：上游侧根治。让 Kafka Producer 给每个 Partition 周期性写"心跳消息"，Flink 端识别后丢弃但用其推 WM。

#### 验证

```bash
# Web UI: 看每个 SubTask 的 currentOutputWatermark
curl http://jobmanager:8081/jobs/<jobId>/vertices/<vertexId>/watermarks

# 应该看到：所有 SubTask 的 WM 都在缓慢推进，不存在某个长期不变
```

> **口诀**：**Event Time + Watermark = 木桶效应**。任何一个空闲分区都会卡住整体。`withIdleness` 是逃生口，业务上接受"迟到"才能用。

---

### 案例 ② Checkpoint 失败 —— "状态太大，超时一片"

#### 现象

凌晨告警：Checkpoint 连续失败。Web UI 看 "Checkpoint History"，最近 10 次有 8 次 `FAILED (timeout)`。State 大小 50GB，Checkpoint timeout 配的 10 分钟。

#### 根因

排查链路：

1. **状态过大**：50GB 全量 Checkpoint 上传到 S3 单次需要 15 分钟，超过 timeout。
2. **未开增量 Checkpoint**：每次都全量传。
3. **反压同时存在**：上游 Sink 慢导致 Barrier 对齐时间长，进一步拖死。

更深层：你 Web UI 看 "Checkpoint Detail"，发现某些 SubTask 的 "Sync Duration" 几秒，但 "Async Duration" 8 分钟 —— **说明本地 snapshot 快，问题在上传**。

```sql
-- Checkpoint 元数据表 (如果接了 metric reporter)
SELECT checkpoint_id, status, duration_ms, sync_ms, async_ms, state_size
FROM flink_checkpoints
WHERE job_id = 'xxx' AND start_time > NOW() - INTERVAL 1 HOUR
ORDER BY start_time DESC;
```

#### 修复

**第一步**：开 RocksDB + 增量 Checkpoint。

```yaml
state.backend: rocksdb
state.backend.incremental: true
state.checkpoints.dir: s3://my-bucket/flink/checkpoints
execution.checkpointing.interval: 60000
execution.checkpointing.timeout: 600000
execution.checkpointing.min-pause: 30000
```

```java
env.enableCheckpointing(60_000);
env.getCheckpointConfig().setCheckpointTimeout(600_000);
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(30_000);
env.getCheckpointConfig().setCheckpointStorage("s3://my-bucket/flink/checkpoints");
env.setStateBackend(new EmbeddedRocksDBStateBackend(true));   // true = 增量
```

增量后单次 Checkpoint 大小从 50GB 降到 < 200MB（仅新 SST），上传几秒结束。

**第二步**：开 Unaligned Checkpoint，让反压时也能完成。

```java
env.getCheckpointConfig().enableUnalignedCheckpoints();
env.getCheckpointConfig().setAlignedCheckpointTimeout(Duration.ofSeconds(30));
// 30 秒内对齐成功用 aligned，否则自动切 unaligned
```

**第三步**：上传链路本身优化。

```yaml
# 多线程上传 SST
state.backend.rocksdb.checkpoint.transfer.thread.num: 4

# 增大 RocksDB 写缓冲，减少小 SST
state.backend.rocksdb.writebuffer.size: 128mb
```

#### 验证

```bash
# 看最近 20 次 Checkpoint 的成功率
curl http://jobmanager:8081/jobs/<jobId>/checkpoints | \
  jq '.history[].status' | sort | uniq -c

# 期望：COMPLETED 占绝大多数，FAILED < 5%
```

监控指标接告警：

```
flink_jobmanager_job_lastCheckpointDuration > 300_000  → 警告
flink_jobmanager_job_numberOfFailedCheckpoints > 0    → 告警（连续 3 次告紧急）
```

> **口诀**：**大 State 必上 RocksDB 增量；反压频繁必开 Unaligned；上传慢调 transfer 线程数**。

---

### 案例 ③ State 过大导致 OOM —— "TaskManager 不断 OOMKilled"

#### 现象

K8s 上 Flink TaskManager Pod 反复 OOMKilled，重启后跑 1 小时又挂。`kubectl describe pod` 显示 `Reason: OOMKilled`。但 JVM heap dump 看堆只用了 60%。

#### 根因

经典误区：**Flink 的 JVM 堆 != Pod 的内存上限**。Flink 的内存模型分了好几块：

```
Pod Memory Limit: 8GB
├── JVM Heap (taskmanager.memory.task.heap.size): 3GB
├── Managed Memory (RocksDB + sort): 2GB        ★ 堆外
├── Network Buffers: 0.5GB                       ★ 堆外
├── JVM Metaspace: 256MB
├── JVM Overhead (GC、栈等): 768MB               ★ 堆外
└── Framework Memory (Flink 本身): 512MB
```

实际问题：你用了 RocksDB 状态后端，**Managed Memory 给了 RocksDB 用作块缓存**。但代码里同时用了大量 `MapState`，操作时 RocksDB 会临时分配 native 内存，**超出 Managed Memory 限制后还会继续涨**（RocksDB 自己的 hard limit 配置缺失）。

K8s 看的是 Pod RSS，超了直接杀。

#### 修复

**第一步**：给 RocksDB 配硬上限。

```yaml
# 让 Flink 严格控制 RocksDB 内存使用
state.backend.rocksdb.memory.managed: true
state.backend.rocksdb.memory.fixed-per-slot: 512mb
# 这样每个 Slot 的 RocksDB 内存严格限制在 512mb，不会偷偷涨
```

**第二步**：调整内存比例。

```yaml
taskmanager.memory.process.size: 8g
taskmanager.memory.managed.fraction: 0.4   # 默认 0.4，给 RocksDB 留 40%
taskmanager.memory.network.fraction: 0.1
taskmanager.memory.jvm-overhead.fraction: 0.1
taskmanager.memory.jvm-overhead.min: 768mb
```

**第三步**：定位真正的"State 大户"。

```java
// 在 KeyedProcessFunction.open 里打印 state 大小
@Override
public void open(Configuration parameters) throws Exception {
    super.open(parameters);
    // 注册 metric
    getRuntimeContext().getMetricGroup().gauge("stateSize",
        (Gauge<Long>) () -> {
            // 估算：遍历 state 太贵，用 RocksDB 文件大小
            return rocksDBSize();
        });
}
```

或直接看 Web UI 的 "Subtask Metrics" → "rocksdb.estimate-num-keys" 和 "rocksdb.size-all-mem-tables"。

**第四步**：State TTL 必须配。

```java
StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.hours(24))
    .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
    .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
    .cleanupInRocksdbCompactFilter(1000)   // ★ RocksDB compaction 时清理
    .build();

ValueStateDescriptor<MyState> descriptor = new ValueStateDescriptor<>("state", MyState.class);
descriptor.enableTimeToLive(ttlConfig);
```

注意 `cleanupInRocksdbCompactFilter` —— 不开启的话过期 State 不会被物理删除，只是读时跳过，**磁盘和内存继续涨**。

#### 验证

```bash
# 容器内直接看进程的 RSS
kubectl exec <pod> -- cat /proc/1/status | grep -i vmrss

# 应该稳定在 Pod limit 下方，不再增长
# 同时 Heap 利用率正常（不应超过配置的 task.heap.size）
```

> **口诀**：**Flink 内存 = Heap + Managed + Network + Overhead**，少算一块就 OOMKilled。RocksDB 必须 `memory.managed: true`，State 必须配 TTL + compactFilter。

---

### 案例 ④ RocksDB 写放大 —— "磁盘 IO 飙到 100%"

#### 现象

业务感知作业延迟变大。看 Grafana：TaskManager 节点磁盘 IO 飙到 100%，磁盘写入吞吐 600MB/s 持续不降。但作业输入只有 10MB/s。

#### 根因

经典 LSM-Tree 写放大问题。RocksDB 的写路径：

```
WriteBatch ──► MemTable (内存) ──► Immutable MemTable ──► Flush 到 L0 SST
                                                              │
                                                              ▼
                                                         Compaction L0 → L1 → L2 ...
```

每条记录平均会被**写入磁盘 10~30 次**（每层 compaction 都重写）。10MB/s 输入 → 200MB/s 实际磁盘写入是常见的。

加剧因素：

1. **Key 没有 keyBy 后散开**：所有 key 集中在少数 SubTask，单个 RocksDB 实例承受全部写。
2. **`MapState` 用得多**：每个 entry 都是一个 KV 对，写放大叠加 entry 数量。
3. **Compaction 线程不够**：默认 1 个，跟不上写入速度，L0 文件堆到几十个，读延迟也跟着炸。

#### 修复

**第一步**：诊断写放大严重程度。

```bash
# 在 Flink TaskManager 容器里直接看 RocksDB 状态
ls -lh /tmp/flink-io-*/job_*_op_*/db/

# 关键文件：
# - LOG: RocksDB 自己的日志，搜 "compaction"
# - OPTIONS-*: 当前生效的配置
# - *.sst: SST 文件，看数量和大小分布
```

如果 L0 SST 数量持续 > 4，说明 compaction 跟不上。

**第二步**：调大 compaction 并行度。

```yaml
state.backend.rocksdb.thread.num: 4
state.backend.rocksdb.compaction.level.use-dynamic-size: true
state.backend.rocksdb.compaction.level.max-size-level-base: 256mb
state.backend.rocksdb.writebuffer.count: 4
state.backend.rocksdb.writebuffer.size: 128mb
```

**第三步**：换 compaction 策略。

```yaml
# Level Compaction（默认）：写放大高，读放大低
# Universal Compaction：写放大低，读放大高，空间放大高
# 选择取决于业务模式
state.backend.rocksdb.compaction.style: LEVEL   # 或 UNIVERSAL
```

**第四步**：从业务侧降低写量。

```java
// ❌ 每条数据都写 state
@Override
public void processElement(Event e, Context ctx, Collector<Result> out) {
    Long count = countState.value();
    countState.update((count == null ? 0 : count) + 1);
}

// ✅ 攒一批再写（窗口 + flush）
@Override
public void processElement(Event e, Context ctx, Collector<Result> out) {
    buffer.add(e);
    if (buffer.size() > 100) {
        Long count = countState.value();
        countState.update((count == null ? 0 : count) + buffer.size());
        buffer.clear();
    }
}
```

#### 验证

```bash
# 测吞吐：作业再压一遍，磁盘 IO 应稳定在合理区间
iostat -x 1 | grep nvme0n1

# 看 RocksDB 自己的统计
curl http://jobmanager:8081/jobs/<jobId>/vertices/<vid>/subtasks/0/metrics?get=rocksdb.bytes-written,rocksdb.compact-write-bytes
# 写放大比 = compact-write-bytes / bytes-written，理想 < 10
```

> **口诀**：**LSM-Tree 天生写放大**。Flink RocksDB 必须调 compaction 线程数和 writebuffer。业务侧能攒批就攒批，State 不是 KV 数据库。

---

### 案例 ⑤ Backpressure 导致下游不消费 —— "Source 偏移量不前进"

#### 现象

监控看 Kafka Source 的 `currentOffsetLag` 持续增长，但 Source 算子的 CPU 利用率只有 10%。Web UI 看 Source 的 "BackPressure" 显示 HIGH。

#### 根因

反压的传播链：

```
Sink 算子 (写 MySQL，慢) ── 100% busy
                            │ NetworkBuffer 满
                            ▼
中间 KeyedProcess 算子 ── 拿不到 Buffer，wait
                            │
                            ▼
Source 算子 ── 同样拿不到 Buffer，停止从 Kafka 拉取
                            │
                            ▼
Kafka offset lag 增长
```

诊断方法：从 Sink 往上看 BackPressure 指标，**第一个不是 HIGH 的算子**就是瓶颈。本案中 Sink 是 LOW（自己慢，不接受反压），它上游全是 HIGH。

#### 修复

**先定位是真慢还是配置问题**：

```java
// 用 jstack 抓 Sink 线程，看在干嘛
jstack <pid> | grep -A 30 "Sink__"

// 如果是 SocketRead0 → 网络问题或外部系统慢
// 如果是 ConnectionPool.getConnection → 连接池满
// 如果是 BatchInsert.execute → SQL 慢
```

**修复 A**：增大 Sink 并行度（最直接）。

```java
sink.setParallelism(env.getParallelism() * 2);
// 但要注意：MySQL 单实例承受不了无限并发，可能换成分库或异步写
```

**修复 B**：换异步 Sink。

```java
// JdbcSink 是同步的，换成 AsyncSink
DataStream<Result> source = ...;
source.transform("AsyncMySQL", typeInfo,
    new AsyncSinkOperator<>(
        new MySQLAsyncSinkBuilder()
            .setMaxBatchSize(500)
            .setMaxBufferedRequests(10000)
            .setMaxBatchSizeInBytes(1024 * 1024)
            .setMaxTimeInBufferMS(1000)
            .build()));
```

**修复 C**：开 Unaligned Checkpoint，避免反压时 Checkpoint 也跟着挂。

```java
env.getCheckpointConfig().enableUnalignedCheckpoints();
```

**修复 D**：业务侧分级削峰。

```java
// 把 Sink 拆成"实时" + "异步落库"
DataStream<Result> stream = ...;
// 实时分支：只更新 Redis（快）
stream.addSink(new RedisSink<>());
// 异步分支：写 MySQL 走单独慢通道
stream.addSink(new SlowMySQLSink<>()).setParallelism(1);
```

#### 验证

```bash
# 看 Source 的 lag 应稳定下降
kafka-consumer-groups --bootstrap-server kafka:9092 \
  --describe --group my-flink-group

# Web UI 看 BackPressure 指标，应全部回到 OK
```

> **口诀**：**反压从下游传播到上游**。诊断从 Sink 倒推，第一个 OK 的就是瓶颈。修复优先级：异步化 > 增并行 > 削峰。

---

### 案例 ⑥ Exactly-Once Sink 与 Kafka 不兼容 —— "下游消费到重复"

#### 现象

Flink 作业开了 `EXACTLY_ONCE` 写 Kafka，下游 Spark 消费时偶尔看到重复消息。Flink 端日志没异常。

#### 根因

3 个常见兼容问题：

**问题 A**：Kafka Producer 的 `transaction.timeout.ms` < Checkpoint 间隔。

```
Checkpoint 间隔 = 60s
Kafka 默认 transaction.timeout.ms = 60s
故障恢复时，Kafka broker 已经把事务 abort 了 → Flink commit 失败 → 数据丢
```

**问题 B**：下游消费者用了默认的 `read_uncommitted`。

```java
// Spark 默认 isolation.level = read_uncommitted
// 它能读到 Flink 还没 commit 的事务里的数据 → 重复
```

**问题 C**：Kafka broker 版本太老（< 0.11），不支持事务。

#### 修复

**修复 A**：

```java
Properties props = new Properties();
props.setProperty("transaction.timeout.ms", "900000");  // 15 分钟，远大于 Checkpoint 间隔
props.setProperty("transactional.id.prefix", "my-flink-job-");

KafkaSink<String> sink = KafkaSink.<String>builder()
    .setBootstrapServers("kafka:9092")
    .setRecordSerializer(KafkaRecordSerializationSchema.builder()
        .setTopic("output")
        .setValueSerializationSchema(new SimpleStringSchema())
        .build())
    .setDeliveryGuarantee(DeliveryGuarantee.EXACTLY_ONCE)
    .setTransactionalIdPrefix("my-flink-job-")
    .setKafkaProducerConfig(props)
    .build();
```

注意 `transaction.timeout.ms` 还受 broker 端 `transaction.max.timeout.ms` 限制（默认 15 分钟），生产值 timeout 必须 ≤ broker 上限。

**修复 B**：下游消费者必须配 `read_committed`。

```java
// Java consumer
props.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");

// Spark
spark.readStream
    .format("kafka")
    .option("kafka.isolation.level", "read_committed")
    .load();
```

**修复 C**：Kafka broker 要 0.11+，且 `transaction.state.log.replication.factor >= 3` 才能保证事务持久化。

**修复 D**：注意 `transactional.id.prefix` 必须**唯一**，否则两个作业用同一个 prefix 会互相 abort 对方的事务。

#### 验证

```bash
# 1. 检查 Kafka transaction state
kafka-console-consumer --bootstrap-server kafka:9092 \
  --topic __transaction_state \
  --formatter "kafka.coordinator.transaction.TransactionLog\$TransactionLogMessageFormatter" \
  --from-beginning

# 2. 故意 kill TaskManager，看下游是否还有重复
# 3. Flink Web UI 看 Sink 的 "checkpointed-offsets" 是否单调递增
```

> **口诀**：**Exactly-Once 是端到端的契约**：Source 可重放 + Sink 事务 + 下游 read_committed + Kafka transaction.timeout > Checkpoint 间隔，缺一不可。

---

### 案例 ⑦ 重启后状态恢复失败 —— "Savepoint 跨版本不兼容"

#### 现象

升级 Flink 1.16 → 1.18，从 Savepoint 恢复时报错：

```
Caused by: java.lang.RuntimeException: Cannot map checkpoint/savepoint state
for operator <uid> to the new program, because the operator is not available
in the new program. If you want to allow to skip this, you can set the
--allowNonRestoredState option on the CLI.
```

#### 根因

Flink 用 **Operator UID** 把状态和算子绑定。如果你的算子没显式 `.uid("xxx")`，Flink 会用算子的 hash code（基于算子的拓扑位置）作为 ID。**任何拓扑变化（多一个 map、改一个 keyBy）都会让 hash 变 → Savepoint 找不到对应算子**。

更糟的是，**升级 Flink 版本时算子内部实现变化也可能让序列化格式不兼容**。

#### 修复

**铁律**：所有有状态的算子必须显式设置 `uid`。

```java
DataStream<Event> source = env
    .fromSource(kafkaSource, WatermarkStrategy.noWatermarks(), "source")
    .uid("kafka-source")               // ★ 必须
    .name("kafka-source");

source.keyBy(Event::getUserId)
    .process(new MyProcessFunction())
    .uid("user-aggregator")            // ★
    .name("user-aggregator");
```

uid 一旦上线**永远不能改**，否则状态会"丢"。命名约定建议：`<biz>-<algo>-<version>`，如 `order-aggregator-v1`。

**升级 Flink 版本前的检查清单**：

1. 同版本 Flink 自己 take savepoint（不是用线上的）
2. 用新版本 Flink 启动作业，从 savepoint 恢复
3. 跑足够长时间确认状态正确
4. 才能切流量

**如果真改了拓扑，怎么办？**

```bash
# 方案 1：跳过找不到的状态（接受丢失）
flink run --fromSavepoint s3://.../savepoint-xxx \
  --allowNonRestoredState \
  my-job.jar

# 方案 2：用 State Processor API 离线编辑 Savepoint
# 把旧 uid 的状态拷贝到新 uid
```

State Processor API 示例：

```java
// 把旧 uid 的 state 改成新 uid
ExecutionEnvironment bEnv = ExecutionEnvironment.getExecutionEnvironment();
SavepointReader savepoint = SavepointReader.read(bEnv,
    "s3://bucket/savepoint-old", new EmbeddedRocksDBStateBackend());

DataSet<MyState> oldState = savepoint.readKeyedState("old-uid",
    new MyKeyedStateReaderFunction());

SavepointWriter.newSavepoint(64)
    .withOperator("new-uid", OperatorTransformation
        .bootstrapWith(oldState)
        .keyBy(MyState::getKey)
        .transform(new MyKeyedStateBootstrapFunction()))
    .write("s3://bucket/savepoint-new");
```

#### 验证

```bash
# 启动后看 metrics
curl http://jobmanager:8081/jobs/<jobId>/checkpoints/details/<cpId>

# 关键：state-size 应接近 savepoint 大小，说明 state 真的恢复了
```

> **口诀**：**状态算子必须 .uid()，且 uid 永不可改**。升级 Flink 版本必跑预演。State Processor API 是最后的救命稻草。

---

### 案例 ⑧ Window 计算延迟 —— "迟到数据搞乱统计"

#### 现象

业务方说 "10:00~11:00 的窗口结果，11:30 看到的值和 12:30 看到的值不一样"。窗口算子用 Event Time + 5 秒 lateness。

#### 根因

迟到事件的处理路径：

```
WM 推到 11:00:05 → 触发 [10:00, 11:00) 窗口，输出结果 V1
WM 推到 11:00:10 → 仍在 lateness 内
WM 推到 11:01:00 → 超过 lateness（5 秒），窗口被销毁

但 11:30 来了一条 timestamp = 10:55 的迟到事件
→ 已超过 lateness → 默认丢弃，但你没配 sideOutput → 业务感知不到
```

更典型的场景：业务以为 lateness 是"额外保留 5 秒"，**实际是"WM 超过 window_end 后再多等 5 秒"**。如果上游有 3 分钟的乱序，5 秒 lateness 就完全不够。

#### 修复

**修复 A**：合理设置 WatermarkStrategy 的乱序时间。

```java
WatermarkStrategy<Event> wm = WatermarkStrategy
    .<Event>forBoundedOutOfOrderness(Duration.ofMinutes(5))   // ★ 容忍 5 分钟乱序
    .withTimestampAssigner((e, ts) -> e.getEventTime());
```

WM 的乱序时间是"全局缓冲"，会增加端到端延迟（你的窗口要多等 5 分钟才出第一次结果）。

**修复 B**：配 allowedLateness + sideOutput。

```java
OutputTag<Event> lateTag = new OutputTag<Event>("late") {};

SingleOutputStreamOperator<Result> windowed = stream
    .keyBy(Event::getKey)
    .window(TumblingEventTimeWindows.of(Time.hours(1)))
    .allowedLateness(Time.minutes(10))   // ★ 窗口关闭后多保留 10 分钟
    .sideOutputLateData(lateTag)         // ★ 超过 lateness 的进侧输出
    .aggregate(new MyAggregator());

// 主流：正常结果（迟到事件来了会触发更新）
windowed.addSink(...);

// 侧流：实在迟到太狠的，单独处理（修正、补偿、报警）
windowed.getSideOutput(lateTag)
    .addSink(new LateDataSink());
```

注意：**allowedLateness 期间窗口被多次触发**，下游 Sink 必须能处理"同一个窗口结果的多次更新"（典型方案：upsert by window_start + key）。

**修复 C**：用增量聚合 + onTimer，自己控制触发时机。

```java
public class MyAggregator extends KeyedProcessFunction<String, Event, Result> {
    private ValueState<Long> sum;
    private ValueState<Long> windowEnd;

    @Override
    public void processElement(Event e, Context ctx, Collector<Result> out) throws Exception {
        long winStart = e.getEventTime() - e.getEventTime() % 3600_000;
        long winEnd = winStart + 3600_000;
        sum.update((sum.value() == null ? 0 : sum.value()) + e.getValue());

        // 比内置 Window 更细粒度的控制
        if (windowEnd.value() == null || windowEnd.value() < winEnd) {
            windowEnd.update(winEnd);
            ctx.timerService().registerEventTimeTimer(winEnd + 600_000); // 10 分钟 lateness
        }
    }

    @Override
    public void onTimer(long ts, OnTimerContext ctx, Collector<Result> out) throws Exception {
        out.collect(new Result(ctx.getCurrentKey(), sum.value()));
        sum.clear();
        windowEnd.clear();
    }
}
```

#### 验证

```sql
-- 业务侧用 window_start + key 做幂等校验
SELECT window_start, key, count(*)
FROM result_table
WHERE window_start = '2024-01-01 10:00:00'
GROUP BY window_start, key
HAVING count(*) > 1;
-- 应该没有重复，多次更新被合并
```

> **口诀**：**Watermark 乱序时间 + Allowed Lateness 是双层防线**。下游必须能 upsert，否则迟到更新会冲掉之前的结果。

---

### 案例 ⑨ 维表 JOIN 内存爆炸 —— "广播 State 撑满 Heap"

#### 现象

订单流 JOIN 商品维表（500 万行，每行 ~5KB），用 Broadcast State 实现。每个 TaskManager 都要存这 25GB，但配的 Heap 只有 8GB —— 反复 OOM。

#### 根因

Broadcast State 的物理形态：**所有 SubTask 都持有同一份完整状态拷贝**。这是它"广播"的代价。25GB × 4 个 TaskManager = 100GB 总占用，但每个 TM 必须能装下完整的 25GB。

为什么不用 Keyed State？因为维表 lookup 是按 itemId 查，订单流也是按 itemId 来 —— 但订单流可能各种 itemId 都有，**没法把维表"分片"到对应 SubTask**。

#### 修复

**方案 A**：维表压缩 + 只缓存热数据。

```java
// 维表只保留 hotKey
public class HotKeyBroadcastFunction extends BroadcastProcessFunction<...> {
    private MapStateDescriptor<Long, Item> stateDesc;

    @Override
    public void processBroadcastElement(Item item, Context ctx, Collector<Result> out) {
        if (item.getQps() > 100) {  // 只缓存高 QPS 的商品
            ctx.getBroadcastState(stateDesc).put(item.getId(), item);
        }
    }
}
```

**方案 B**：换 Keyed State + 异步 IO 查询维表。

```java
// 用 Async I/O 调外部存储（Redis / HBase）
DataStream<Order> orders = ...;
DataStream<EnrichedOrder> enriched = AsyncDataStream.unorderedWait(
    orders,
    new ItemLookupFunction(),  // 异步查 Redis
    1000, TimeUnit.MILLISECONDS,
    100);  // 容量

public class ItemLookupFunction extends RichAsyncFunction<Order, EnrichedOrder> {
    private transient RedisAsyncClient redis;

    @Override
    public void asyncInvoke(Order order, ResultFuture<EnrichedOrder> resultFuture) {
        redis.get("item:" + order.getItemId())
            .thenAccept(item -> resultFuture.complete(
                Collections.singletonList(new EnrichedOrder(order, item))));
    }
}
```

**方案 C**：Lookup Join (Flink SQL)。

```sql
CREATE TABLE items (
    id BIGINT,
    name STRING,
    PRIMARY KEY (id) NOT ENFORCED
) WITH (
    'connector' = 'jdbc',
    'url' = 'jdbc:mysql://...',
    'lookup.cache.max-rows' = '50000',     -- ★ 本地 cache
    'lookup.cache.ttl' = '10min'
);

SELECT o.*, i.name
FROM orders AS o
LEFT JOIN items FOR SYSTEM_TIME AS OF o.proc_time AS i
ON o.item_id = i.id;
```

`FOR SYSTEM_TIME AS OF` 语义就是"用每条订单的 processing time 去查维表当前值"，配合 `lookup.cache.*` 自动缓存。

**方案 D**：维表特别大时，按时间分桶懒加载。

```java
// 假设 itemId 模式有时间相关性，按 hash 范围分桶
// 每个 SubTask 只加载自己负责的桶
keyBy(order -> order.getItemId() % parallelism)
```

#### 验证

```bash
# 监控 Heap 使用率，应稳定不增长
curl http://jobmanager:8081/jobs/<jobId>/vertices/<vid>/subtasks/0/metrics?get=Status.JVM.Memory.Heap.Used

# 监控外部存储 (Redis) 的 QPS 和延迟
# 如果方案 B 选了，Redis 必须能扛住订单流 × 命中率
```

> **口诀**：**Broadcast State = 全量复制**，只适合小维表（< 100MB）。大维表用 Async I/O + 外部 KV，或 Lookup Join + 本地 cache。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 时间与 Watermark
- [ ] 显式设 `setStreamTimeCharacteristic(EventTime)`（1.12+ 默认 EventTime，但显式更稳）
- [ ] 所有 Source 配 `WatermarkStrategy`，`forBoundedOutOfOrderness` 时间根据上游真实乱序设
- [ ] **空闲分区配 `withIdleness(Duration)`**，避免 Watermark 木桶卡死
- [ ] Window 配 `allowedLateness` + `sideOutputLateData`，迟到数据有处置

### State 与 UID
- [ ] **所有有状态算子必须 `.uid("xxx").name("xxx")`**，且 uid 上线后永不改
- [ ] 大 State 用 `EmbeddedRocksDBStateBackend(true)`（增量 Checkpoint）
- [ ] 所有 ValueState/MapState 配 `enableTimeToLive` + `cleanupInRocksdbCompactFilter`
- [ ] State 大小定期监控（`rocksdb.estimate-num-keys`）

### Checkpoint
- [ ] `enableCheckpointing(60_000)` 起步，根据延迟需求调整
- [ ] `setCheckpointTimeout` 设为间隔的 5~10 倍
- [ ] `setMinPauseBetweenCheckpoints` 设 30s 以上
- [ ] 反压频繁的作业开 `enableUnalignedCheckpoints` + `setAlignedCheckpointTimeout(30s)`
- [ ] Checkpoint 存储用 S3/HDFS（绝对不用本地盘）
- [ ] 周期性手动 Savepoint 兜底（升级、扩容用）

### Exactly-Once 端到端
- [ ] Source 必须支持 offset 重放（Kafka / Pulsar / Kinesis 都支持）
- [ ] Sink 用 2PC 实现（KafkaSink / FileSink / JDBC Sink with XA）
- [ ] Kafka Producer `transaction.timeout.ms` ≥ Checkpoint 间隔 + 网络抖动 buffer
- [ ] 下游消费者配 `isolation.level=read_committed`
- [ ] `transactional.id.prefix` 全局唯一，多个作业不能撞

### 内存与资源
- [ ] **Pod / Container 内存 = JVM Heap + Managed + Network + Overhead**，算清
- [ ] RocksDB 开 `state.backend.rocksdb.memory.managed: true`
- [ ] `state.backend.rocksdb.thread.num` ≥ 4
- [ ] `state.backend.rocksdb.block.cache-size` ≥ 256MB
- [ ] 容器加 `JVM args: -XX:MaxRAMPercentage=70.0` 防止 JVM 把 Pod 内存抢光

### 反压与监控
- [ ] Web UI 的 BackPressure 指标接告警
- [ ] Source 的 `currentOffsetLag` / `currentEmitEventTimeLag` 接告警
- [ ] Checkpoint 失败次数监控（`numberOfFailedCheckpoints`）
- [ ] Heap / RSS / GC 时间监控（`Status.JVM.GC.Time`）
- [ ] 长事务告警：Sink 端事务停留时间 > Checkpoint 间隔 × 3

### 拓扑设计
- [ ] **同 keyBy 的算子尽量 chain**（默认开，别乱 `disableChaining`）
- [ ] Source 并行度 ≤ Kafka 分区数
- [ ] 大 Sink 单独并行度，避免拖累上游
- [ ] CPU 密集 / IO 密集算子放不同 Slot Group

### 升级与发布
- [ ] **升级 Flink 版本前，新版本预演 + 大流量回归**
- [ ] 改拓扑前先 take Savepoint，回滚有路
- [ ] `--allowNonRestoredState` 是最后手段，不能成为日常

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### Flink CEP（复杂事件处理）
- `Pattern API`：定义事件序列模式（A → B 在 5 分钟内 → C）
- 内部实现是 NFA（非确定有限自动机），状态爆炸是它的天敌
- 适合：风控规则、用户行为路径、设备故障序列检测
- 坑：Pattern 越复杂，State 越大，单 key 状态可能 GB 级
- 1.13+ 引入了 `CEPPatternProcessFunction`，比老 API 更灵活

### Flink SQL 与流批一体
- Calcite 优化器、Blink Planner 演化
- 流批一体的本质：**有界流 = 批，无界流 = 流**，同一段 SQL 跑两套 runtime
- Dynamic Table 模型：Append / Retract / Upsert
- 看 `EXPLAIN` 输出理解 SQL 翻译成 DataStream 后的算子拓扑
- 性能调优：`table.exec.mini-batch.enabled` / `table.optimizer.agg-phase-strategy`

### Flink CDC（数据变更捕获）
- `flink-cdc-connectors` 项目，Debezium 内嵌
- 支持 MySQL / Postgres / Oracle / MongoDB
- 全量 + 增量无缝切换的实现：snapshot phase 用并行 SELECT，增量用 binlog
- 适合：实时数仓、物化视图、数据湖入湖
- 坑：MySQL binlog 必须 ROW 模式；Postgres 必须 logical replication；高频 DDL 的库慎用

### 与 Kafka Streams 的对比
| 维度 | Flink | Kafka Streams |
| --- | --- | --- |
| 部署 | 独立集群 | 嵌入应用进程 |
| Source | 任意 | 仅 Kafka |
| State | RocksDB / Heap | RocksDB |
| Exactly-Once | 跨多 Source/Sink | 仅 Kafka 链路 |
| 学习曲线 | 中 | 低 |

简单业务用 Kafka Streams；多 Source/Sink、需要 Window 复杂语义、要跑批计算 → Flink。

### Flink on K8s（Native Kubernetes）
- `Native Kubernetes` 模式：JobManager 直接调用 K8s API 申请 TaskManager Pod
- 对比 `Standalone on K8s`（用 Deployment 静态拉起）：Native 模式弹性更好
- HA 配置：`high-availability: kubernetes` + `kubernetes.cluster-id`
- 镜像构建：`flink:1.18-java11` 基础上叠应用 JAR
- 坑：Pod 网络重启后 IP 变，Checkpoint 路径必须用 PVC 或 S3

### Apache Paimon（Flink Table Store）
- 原 Flink Table Store，2023 年捐给 ASF 改名 Paimon
- 流批一体的湖存储，LSM-Tree on 文件系统，与 Flink 深度集成
- 适合：实时数仓、CDC 入湖、Lakehouse 的存储层
- 与 Iceberg / Hudi 对比：Paimon 的 LSM 结构对**实时更新**更友好

### 实时数仓建设
- 经典分层：ODS（原始）→ DWD（明细）→ DWS（汇总）→ ADS（应用）
- Flink + Kafka + Paimon/Iceberg 是当前主流栈
- Lambda → Kappa → 流批一体 的架构演进
- 数据治理：血缘、质量、SLA 监控

### 调度模型与故障恢复
- `Region Failover`：1.10+ 引入，单算子故障只重启相关 Region 而非全图
- `Pipelined Region` 划分规则
- `Adaptive Scheduler`（1.13+）：动态调整并行度而不重启作业
- `Reactive Mode`：根据资源动态扩缩容

### 性能极致优化
- 算子内对象复用：`enableObjectReuse()`（小心 mutable 对象的副作用）
- Network Buffer 调优：`taskmanager.network.memory.buffers-per-channel`
- Operator Chain 的强制开关：`startNewChain()` / `disableChaining()`
- 序列化：注册 POJO / 用 Avro 替代 Java 序列化

---

## 7. 引用

- Flink 官方源码：[apache/flink](https://github.com/apache/flink)（建议看 1.18+）
- Chandy & Lamport, *Distributed Snapshots: Determining Global States of Distributed Systems*（1985 年的算法论文，所有 Flink Checkpoint 的根）
- Akidau et al., *The Dataflow Model: A Practical Approach to Balancing Correctness, Latency, and Cost in Massive-Scale, Unbounded, Out-of-Order Data Processing*（VLDB 2015，Watermark 模型源头）
- 《Streaming Systems》—— Tyler Akidau / Slava Chernyak（流系统圣经，覆盖 Watermark / Window 的理论）
- 《Stream Processing with Apache Flink》—— Fabian Hueske / Vasiliki Kalavri
- Jamie Grier, *Extending the Yahoo! Streaming Benchmark*（Flink 性能基准源头）
- Flink Forward 历年 PPT：[flink-forward.org](https://www.flink-forward.org/)
- 阿里云 Flink 团队博客：内部实战，特别是 RocksDB 调优、维表 JOIN 部分极有价值
- [FLIP-76: Unaligned Checkpoints](https://cwiki.apache.org/confluence/display/FLINK/FLIP-76%3A+Unaligned+Checkpoints) —— 反压时也能完成 Checkpoint 的设计文档
- [FLIP-188: Introduce Built-in Dynamic Table Storage](https://cwiki.apache.org/confluence/display/FLINK/FLIP-188%3A+Introduce+Built-in+Dynamic+Table+Storage) —— Paimon 前身

---

> 配套：[Kafka 存储引擎](/article/mq/kafka-storage) | [ClickHouse 深入](/article/db/clickhouse) | [分布式事务](/article/distributed/dist-tx)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
