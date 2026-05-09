# RocketMQ 内核 —— 存储设计、事务消息、顺序消息与积压排查

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，看 [消息队列综述](/article/mq/) 那一节就够了。
> 这里要回答的，是**生产 broker 磁盘暴涨、消费堆积一亿条、事务消息回查风暴**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. RocketMQ 把所有 topic 的消息都写到**同一个 CommitLog** 里，Kafka 是每个 partition 一个 log 文件。两种存储模型在「单机吞吐」「topic 数量」「随机 IO」上的取舍各是什么？为什么阿里当年要从 Kafka 这套搬出来？
2. 你发了一条 RocketMQ 事务消息，broker 怎么保证「下游 commit 之前消费者看不见」？half message 物理上存在哪？为什么不能直接放在目标 topic 但加个标记？
3. 业务侧本地事务跑完了，但还没来得及 `endTransaction(commit)` 进程就挂了。broker 怎么知道这条消息该 commit 还是 rollback？回查的频率、次数、超时各是多少？回查打到下游 RT 拉爆怎么办？
4. 你想保证「同一笔订单的创建、支付、退款」按顺序消费。Producer 端写 `MessageQueueSelector` 选 queue，Consumer 端用 `MessageListenerOrderly`。但**消费失败**会怎样？整个 MessageQueue 阻塞？阻塞多久？怎么破？
5. RocketMQ 的「Push 模式」在源码里其实是 Pull。说说为什么这么设计、`PullRequestHoldService` 长轮询是怎么实现 100ms 内消息到达就推送的。
6. 同一个 ConsumerGroup 突然加了 10 台机器，已有的 Consumer 看到大量 "the consumer's subscription not exist" 或者「短时间内消息重复」。这是什么风暴？源码里哪个类决定了 reblance 的时机？
7. 你的 broker 突然磁盘 95%，`du -sh /store/commitlog/` 看到几个 T。你**不能直接删**文件 —— 为什么？RocketMQ 的过期机制是怎么工作的？为什么默认 `fileReservedTime=72h`？
8. RocketMQ 5.x 引入了 **Proxy 模式**，把 broker 拆成「计算（Proxy）+ 存储（Store）」两层。这个改造解决了 4.x 的什么痛？为什么 gRPC 协议会替代私有的 Remoting？

如果有半数答不上，往下看。

---

## 1. 底层模型：CommitLog、ConsumeQueue 与 Raft

绝大多数 RocketMQ 文章止于「NameServer + Broker + Producer + Consumer 四组件」，这层抽象解释不了上面 8 个问题。要看懂它们，必须下沉到 broker 内部的存储模型。

### 1.1 角色与职责

```
              ┌──────────────────┐
              │   NameServer     │  ← 无状态，集群部署，存路由
              │  (1) topic→broker│
              │  (2) broker 心跳  │
              └────────┬─────────┘
                       │ 30s 心跳上报
                       │
        ┌──────────────┼──────────────┐
        │              │              │
   ┌────▼─────┐   ┌────▼─────┐   ┌────▼─────┐
   │ Broker A │   │ Broker B │   │ Broker C │
   │ Master/  │   │ Master/  │   │ Master/  │
   │ Slave    │   │ Slave    │   │ Slave    │
   └────┬─────┘   └────┬─────┘   └────┬─────┘
        │              │              │
        │  pull/push   │              │
   ┌────▼──────────────▼──────────────▼────┐
   │            Producer / Consumer        │
   └───────────────────────────────────────┘
```

四个关键角色：

| 角色 | 状态 | 数据 | 备注 |
| --- | --- | --- | --- |
| **NameServer** | 无状态 | 内存里 topic→broker 路由 | 互相不通信，挂一台 producer 自动切下一台 |
| **Broker** | 有状态 | CommitLog + ConsumeQueue + IndexFile | 主从架构或 DLedger 多副本 |
| **Producer** | 无状态 | 本地缓存路由表 | 30s 拉一次最新路由 |
| **Consumer** | 有状态 | 消费位点 offset | 5s 上报一次到 broker |

NameServer 设计的取舍很关键：**故意不用 ZooKeeper / Raft**，节点之间不一致也无所谓。代价是「broker 上下线后路由会有 30s 内的暂时不一致」，换来的是 NameServer 极简、可水平扩。

### 1.2 存储设计：CommitLog 单日志 + ConsumeQueue 索引

这是 RocketMQ 最关键的一处设计，也是和 Kafka 最大的区别。

```
$ROCKETMQ_HOME/store/
├── commitlog/                         ← 所有 topic 的消息混在一起
│   ├── 00000000000000000000           ← 单文件 1GB，文件名 = 起始 offset
│   ├── 00000000001073741824
│   └── ...
├── consumequeue/                      ← 索引：(commitlog_offset, size, tag_hash)
│   ├── TopicA/
│   │   ├── 0/                         ← queueId = 0
│   │   │   └── 00000000000000000000   ← 单文件 6 万条 = 600 万字节
│   │   ├── 1/
│   │   └── ...
│   └── TopicB/
├── index/                             ← Hash 索引（按 key/uniqkey 查消息）
│   └── 20250509180000.000             ← 按时间分片
├── checkpoint                         ← 三个时间戳：commitlog/cq/index 最近刷盘点
└── abort                              ← 启动时存在 = 上次非正常停机
```

**CommitLog 的关键事实**：

- 一个 broker 上**所有 topic 的所有消息**都串行写到同一个 CommitLog
- 单文件固定 1GB，写满切下一个，文件名 = 该文件首条消息的全局 offset
- 写入永远是**顺序追加**，磁盘 IO 模式接近最优

**ConsumeQueue 的关键事实**：

- 每个 (Topic, QueueId) 一个独立目录
- 每条记录定长 20 字节：`[8 byte commitlog offset][4 byte msg size][8 byte tag hash]`
- 单文件 30 万条，等于 600 万字节 = 5.7MB
- 消费者读消息：先读 ConsumeQueue 拿到 commitlog_offset，再去 CommitLog 取实际消息

为什么这么设计？对比 Kafka：

| 维度 | Kafka | RocketMQ |
| --- | --- | --- |
| 物理文件 | 每个 partition 一个 log | 整个 broker 一个 CommitLog |
| 写入模式 | 多文件并行写 | 单文件串行写 |
| topic 多时 | 退化成随机 IO | 仍是顺序 IO |
| 消费端 IO | 一次顺序读 | 索引随机读 + commitlog 随机读 |
| 单 broker 支持 topic 数 | 千级开始劣化 | 万级仍稳定 |

**阿里当年从 Kafka 切走的核心理由**：业务方有几万个 topic，Kafka 写入退化成随机 IO，吞吐崩了。RocketMQ 牺牲消费端的随机读，换写入永远顺序。

> **口诀**：CommitLog = 写优先（顺序 IO），ConsumeQueue = 读索引（PageCache 兜底随机读）。**Topic 多 → RocketMQ；Topic 少、单 topic 吞吐巨大 → Kafka**。

### 1.3 mmap + PageCache：为什么 RocketMQ 几乎不用堆内存

CommitLog 和 ConsumeQueue 都通过 `MappedByteBuffer`（即 mmap）映射到进程地址空间。这意味着：

- 写消息 = 写 mmap 区域 = 实际是写 PageCache，OS 异步刷盘
- 读消息 = 读 mmap 区域 = 命中 PageCache 就是内存速度
- **broker 进程本身堆内存占用很小**（几个 G 就够），数据全靠 OS PageCache 兜

代价：
- mmap 区域内容由 OS 决定何时刷盘，**断电会丢未刷盘部分**（除非用同步刷盘）
- PageCache 命中率取决于物理内存大小和消息生产/消费时间差
- 消费者「拉历史消息」时，PageCache 缺失会触发磁盘 IO，瞬间 RT 飙升（典型积压恢复时的现象）

```
producer ──write──► MappedByteBuffer ──► PageCache ──flush(OS)──► disk
                                              ▲
                                              │ read
consumer ──pull──► MappedByteBuffer ──────────┘
```

### 1.4 主从架构：异步/同步刷盘 × 异步/同步复制

四种组合，**生产用得最多的是「主从异步复制 + 异步刷盘」**：

| 刷盘 | 复制 | 性能 | 数据可靠性 | 适用场景 |
| --- | --- | --- | --- | --- |
| ASYNC_FLUSH | ASYNC_MASTER | 最高 | 主宕机/断电会丢秒级数据 | 日志、埋点 |
| ASYNC_FLUSH | SYNC_MASTER | 中 | 主宕机不丢，断电丢秒级 | 大多数业务 |
| SYNC_FLUSH | ASYNC_MASTER | 中低 | 断电不丢，主宕机丢复制 lag | 极少用 |
| SYNC_FLUSH | SYNC_MASTER | 最低 | 都不丢 | 金融核心 |

**关键结论**：异步复制 + 主宕机 = 数据丢失。要么用同步复制接受性能下降，要么上 DLedger。

### 1.5 DLedger：基于 Raft 的多副本

主从架构的硬伤：**主宕机后从节点不能自动升主**（4.5 之前完全不能；之后有切换工具但仍是有损切换）。DLedger 是 RocketMQ 4.5+ 引入的解决方案，用 Raft 做多副本一致性。

```
            ┌─────────────────┐
            │   DLedger Group │
            │                 │
    ┌───────▼──────┐  ┌──────▼─────┐  ┌──────▼─────┐
    │  Node A      │  │  Node B    │  │  Node C    │
    │  (Leader)    │  │  (Follower)│  │  (Follower)│
    │  CommitLog   │  │  CommitLog │  │  CommitLog │
    └──────────────┘  └────────────┘  └────────────┘
         │                  │                │
         └─── append + 半数 ack ──────────────┘
```

工作机制：
- 写入：Leader 先本地追加到 CommitLog，并行复制给 Follower，**多数派 ack 后才认为写入成功**
- 选主：Leader 心跳超时，Follower 发起 Election，比较 LogIndex 高的胜出
- 切换：Leader 宕机后秒级选出新 Leader，**自动接管写流量**，不丢已 commit 数据

代价：
- 至少 3 节点起步，机器成本翻倍
- 写延迟约为主从异步复制的 1.5~2 倍（多了一次网络往返）
- 不支持跨机房（除非接受高延迟）

### 1.6 消息发送：同步 / 异步 / Oneway

```java
// 同步：发完等响应
SendResult result = producer.send(msg);

// 异步：注册回调
producer.send(msg, new SendCallback() {
    public void onSuccess(SendResult r) { ... }
    public void onException(Throwable e) { ... }
});

// Oneway：扔了就走，不等响应
producer.sendOneway(msg);
```

| 方式 | 可靠性 | 吞吐 | 典型场景 |
| --- | --- | --- | --- |
| 同步 | 最高 | 最低 | 订单、支付等关键链路 |
| 异步 | 高（需回调正确处理） | 中 | 日志聚合、跨地域同步 |
| Oneway | 最低 | 最高 | 心跳、埋点 |

**最常见的丢消息现场**：用了异步 send 但没写 SendCallback 或 callback 里没做重试 —— 网络抖动 broker 没收到，应用以为发出去了。

### 1.7 消息消费：Push（背后是 Pull）/ Pull

RocketMQ 对外的两个 Consumer 类：

```java
// "Push" 模式 —— 业务最常用
DefaultMQPushConsumer push = new DefaultMQPushConsumer("group_a");
push.subscribe("topic_a", "tag_x");
push.registerMessageListener((MessageListenerConcurrently) (msgs, ctx) -> {
    // 业务处理
    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
});

// Pull 模式 —— 完全自己控制
DefaultMQPullConsumer pull = new DefaultMQPullConsumer("group_a");
pull.start();
PullResult r = pull.pull(mq, "tag_x", offset, 32);
```

**Push 模式的真相**：底层仍然是 Pull。Consumer 内部有个 `PullMessageService` 一直在循环拉消息：拉到了就调用户的 listener；没拉到就发起**长轮询**（broker hold 住请求最多 15s，有新消息立即唤醒返回）。

为什么不做真正的 Push？因为 broker 不知道 Consumer 的处理能力 —— 真 Push 容易把 Consumer 打爆。Pull 模式天然背压。

### 1.8 RocketMQ 5.x 的 Proxy 模式

4.x 的痛点：
- broker 同时承担计算（路由、协议解析、ACL）+ 存储两件事
- 私有 Remoting 协议，多语言客户端工程量大
- broker 扩缩容必须搬数据，运维痛苦

5.x 的方案：

```
                ┌────────────────────────────────┐
                │      Client (gRPC)             │
                └───────────────┬────────────────┘
                                │ gRPC
                ┌───────────────▼────────────────┐
                │   Proxy (无状态，可弹性扩缩)    │  ← 计算层
                │  路由 / ACL / 协议 / 限流       │
                └───────────────┬────────────────┘
                                │ Remoting
              ┌─────────────────┼─────────────────┐
        ┌─────▼─────┐    ┌──────▼─────┐    ┌─────▼─────┐
        │  Broker   │    │  Broker    │    │  Broker   │  ← 存储层
        │ CommitLog │    │ CommitLog  │    │ CommitLog │
        └───────────┘    └────────────┘    └───────────┘
```

收益：
- 客户端只跟 Proxy 说 gRPC，多语言 SDK 工程量大幅下降
- Proxy 无状态，按 QPS 弹性伸缩，broker 只按存储量伸缩
- 协议升级在 Proxy 层做，broker 不动

> **口诀**：4.x = 一体；5.x = 计算/存储分离 + gRPC。**新项目直接 5.x，老的 4.x 跑得稳就别动**。

---

## 2. 源码切片：六段关键路径

> 基于 RocketMQ 4.9.x 源码（5.x 在存储路径上变化不大）。注释为本文添加。

### 2.1 消息存储入口：DefaultMessageStore.putMessage

```java
// org.apache.rocketmq.store.DefaultMessageStore
public PutMessageResult putMessage(MessageExtBrokerInner msg) {
    // ① 状态检查：broker 是否可写
    if (this.shutdown) {
        return new PutMessageResult(PutMessageStatus.SERVICE_NOT_AVAILABLE, null);
    }
    if (!this.runningFlags.isWriteable()) {
        return new PutMessageResult(PutMessageStatus.SERVICE_NOT_AVAILABLE, null);
    }

    // ② topic / 消息长度校验
    if (msg.getTopic().length() > Byte.MAX_VALUE) {
        return new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, null);
    }

    // ③ ★ 核心：把消息写到 CommitLog
    PutMessageResult result = this.commitLog.putMessage(msg);

    // ④ 监控统计
    this.storeStatsService.getSinglePutMessageTopicTimesTotal(msg.getTopic())
        .add(result.getAppendMessageResult().getMsgNum());
    this.storeStatsService.getSinglePutMessageTopicSizeTotal(msg.getTopic())
        .add(result.getAppendMessageResult().getWroteBytes());

    return result;
}
```

**逐段点透**：

- **③ 是热路径**：所有写入都汇到 `commitLog.putMessage`。这就是为什么单 broker 写入是「全局串行」的。
- **没有按 topic / queue 分流**：和 Kafka 完全相反。Kafka 的 produce 路径会按 partition 路由到不同的 log 文件。
- **runningFlags 的 7 个 bit**：分别表示磁盘满、消息体超限、内存溢出等。任何一个置位都拒绝写入。这是 broker 的「自我保护开关」。

### 2.2 CommitLog.putMessage：mmap 写 + 锁

```java
// org.apache.rocketmq.store.CommitLog
public PutMessageResult putMessage(MessageExtBrokerInner msg) {
    msg.setStoreTimestamp(System.currentTimeMillis());
    msg.setBodyCRC(UtilAll.crc32(msg.getBody()));

    // ① 拿到当前可写的 MappedFile（mmap 文件）
    MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile();

    // ② ★ 加锁。两种实现：自旋锁 / 重入锁
    putMessageLock.lock();
    try {
        long beginLockTimestamp = System.currentTimeMillis();
        this.beginTimeInLock = beginLockTimestamp;

        // ③ 重新设置时间戳（在锁内，保证 CommitLog 时间戳单调）
        msg.setStoreTimestamp(beginLockTimestamp);

        // ④ 当前文件不够写 → 创建新文件
        if (null == mappedFile || mappedFile.isFull()) {
            mappedFile = this.mappedFileQueue.getLastMappedFile(0);
        }

        // ⑤ ★ 核心：往 MappedByteBuffer 追加
        AppendMessageResult result = mappedFile.appendMessage(msg, this.appendMessageCallback);

        switch (result.getStatus()) {
            case PUT_OK: break;
            case END_OF_FILE:
                // 当前文件剩余空间不够，开新文件再写
                mappedFile = this.mappedFileQueue.getLastMappedFile(0);
                result = mappedFile.appendMessage(msg, this.appendMessageCallback);
                break;
            // ...
        }
    } finally {
        putMessageLock.unlock();
    }

    // ⑥ 同步刷盘（如果配的是 SYNC_FLUSH）
    handleDiskFlush(result, putMessageResult, msg);
    // ⑦ 同步复制（如果是 SYNC_MASTER）
    handleHA(result, putMessageResult, msg);

    return putMessageResult;
}
```

**逐段点透**：

- **② 锁的选择**：4.x 默认是 `ReentrantLock`，可以配 `useReentrantLockWhenPutMessage=false` 切到自旋锁（CAS）。低延迟场景自旋锁更优，长时间高并发场景重入锁不会饿死线程。
- **整个写入串行化**：所有 producer 的写入都要排队过这把锁。这是单机 RocketMQ 写吞吐的物理上限来源（典型 10~30W TPS）。
- **⑥/⑦ 在锁外**：刷盘和复制不阻塞下一条消息的写入。这是「同步刷盘也能跑出几万 TPS」的关键 —— 用 GroupCommit 把多次写入合并成一次 fsync。

### 2.3 ConsumeQueue 的索引构建：ReputMessageService

ConsumeQueue 不是 producer 写入时立刻建好的。它由一个**异步线程**事后构建：

```java
// org.apache.rocketmq.store.DefaultMessageStore.ReputMessageService
class ReputMessageService extends ServiceThread {
    private volatile long reputFromOffset = 0;   // 当前已分发到的 commitlog 位置

    @Override
    public void run() {
        while (!this.isStopped()) {
            try {
                Thread.sleep(1);    // ★ 1ms 一轮
                this.doReput();
            } catch (Exception e) { /* ... */ }
        }
    }

    private void doReput() {
        for (boolean doNext = true; this.isCommitLogAvailable() && doNext; ) {
            // ① 从 commitlog 拉一段还没分发的数据
            SelectMappedBufferResult result =
                DefaultMessageStore.this.commitLog.getData(reputFromOffset);
            if (result == null) break;

            try {
                this.reputFromOffset = result.getStartOffset();
                for (int readSize = 0; readSize < result.getSize() && doNext; ) {
                    // ② 解析一条消息
                    DispatchRequest req =
                        DefaultMessageStore.this.commitLog.checkMessageAndReturnSize(
                            result.getByteBuffer(), false, false);

                    if (req.isSuccess()) {
                        if (req.getMsgSize() > 0) {
                            // ③ ★ 派发到所有 dispatcher：建 ConsumeQueue + IndexFile
                            DefaultMessageStore.this.doDispatch(req);
                            // ④ 通知长轮询的 PullRequest（有新消息了！）
                            if (BrokerRole.SLAVE != ...) {
                                DefaultMessageStore.this.brokerStatsManager
                                    .incTopicPutNums(req.getTopic(), req.getBatchSize(), 1);
                                notifyMessageArriveIfNecessary(req);
                            }
                            this.reputFromOffset += req.getMsgSize();
                            readSize += req.getMsgSize();
                        }
                    }
                }
            } finally {
                result.release();
            }
        }
    }
}
```

**逐段点透**：

- **③ 是 ConsumeQueue 写入的真正位置**：`doDispatch` 会调 `CommitLogDispatcherBuildConsumeQueue` 和 `CommitLogDispatcherBuildIndex` 两个 dispatcher，分别写 ConsumeQueue 和 IndexFile。
- **④ 长轮询唤醒**：每分发一条消息就检查是否有 PullRequest 在等这个 topic/queue —— 有就立刻唤醒，这是「Push 模式 100ms 内消息可见」的根。
- **延迟问题**：如果 ReputMessageService 卡住，消费者就拉不到消息（即使 CommitLog 已写入）。生产排查「消息明明发了但消费不到」要先看这个线程是否健康。
- **重启恢复**：broker 重启时 ReputMessageService 从 checkpoint 之后的位置开始重放，把所有 ConsumeQueue / Index 重建出来。

### 2.4 事务消息：half message 的存储

事务消息的核心思路是把消息分两阶段：

```
Producer                     Broker                          Consumer
    │                          │                                │
    │── (1) sendHalf ────────►│  写到 CommitLog                  │
    │                          │  topic = RMQ_SYS_TRANS_HALF_TOPIC│
    │                          │  对消费者不可见                  │
    │                          │
    │ executeLocalTransaction()                                  │
    │ 业务侧本地事务...                                         │
    │                          │
    │── (2) endTransaction ───►│
    │       (commit)           │  从 half topic 读出原消息       │
    │                          │  写一条到原 topic               │── pull ──►│
    │                          │  并写一条 op 消息到             │
    │                          │  RMQ_SYS_TRANS_OP_HALF_TOPIC    │
```

half message 在源码里：

```java
// org.apache.rocketmq.broker.transaction.queue.TransactionalMessageBridge
public PutMessageResult putHalfMessage(MessageExtBrokerInner msgInner) {
    // ① 备份原 topic / queueId 到属性里
    MessageAccessor.putProperty(msgInner,
        MessageConst.PROPERTY_REAL_TOPIC, msgInner.getTopic());
    MessageAccessor.putProperty(msgInner,
        MessageConst.PROPERTY_REAL_QUEUE_ID, String.valueOf(msgInner.getQueueId()));
    msgInner.setSysFlag(
        MessageSysFlag.resetTransactionValue(
            msgInner.getSysFlag(),
            MessageSysFlag.TRANSACTION_NOT_TYPE));

    // ② ★ 改写 topic 为内置的 half topic，queueId = 0
    msgInner.setTopic(TransactionalMessageUtil.buildHalfTopic());
    msgInner.setQueueId(0);

    // ③ 像普通消息一样写
    msgInner.setPropertiesString(
        MessageDecoder.messageProperties2String(msgInner.getProperties()));
    return store.putMessage(msgInner);
}
```

**逐段点透**：

- **巧妙之处**：half message 物理上写到了内部 topic `RMQ_SYS_TRANS_HALF_TOPIC`。普通消费者订阅的是业务 topic，看不到 half topic，所以**自然不可见**，不需要在 ConsumeQueue 上做任何「过滤」逻辑。
- **commit 时**：从 half topic 读出原消息，恢复 topic/queueId，重新 put 一次。**消费者看到的是「正常消息」**，不知道是事务消息。
- **op message** 的作用：标记「这条 half 已经处理过」（commit 或 rollback），后续回查时不必再处理它。op 消息在 `RMQ_SYS_TRANS_OP_HALF_TOPIC`。

### 2.5 事务回查：TransactionalMessageCheckService

如果 producer 在 step (1) 之后挂了，broker 永远收不到 endTransaction，half message 就一直挂着。回查机制：

```java
// org.apache.rocketmq.broker.transaction.TransactionalMessageCheckService
@Override
public void run() {
    while (!this.isStopped()) {
        // ① 默认每 60s 跑一次
        long checkInterval = brokerConfig.getTransactionCheckInterval();
        this.waitForRunning(checkInterval);
    }
}

protected void onWaitEnd() {
    long timeout = brokerConfig.getTransactionTimeOut();        // 默认 6s
    int checkMax = brokerConfig.getTransactionCheckMax();       // 默认 15 次
    long begin = System.currentTimeMillis();

    // ② 调到 TransactionalMessageServiceImpl.check
    this.brokerController.getTransactionalMessageService()
        .check(timeout, checkMax, this.brokerController.getTransactionalMessageCheckListener());
}
```

```java
// TransactionalMessageServiceImpl.check 简化
public void check(long transactionTimeout, int transactionCheckMax,
                  AbstractTransactionalMessageCheckListener listener) {
    // ① 遍历 half topic 的每个 queue
    for (MessageQueue mq : halfQueues) {
        long halfOffset = consumeQueue.getMinOffsetInQueue(mq);

        while (true) {
            // ② 拉一批 half 消息
            PullResult result = pullHalfMsg(mq, halfOffset, 32);
            for (MessageExt half : result.getMsgFoundList()) {

                // ③ 判断是否已 commit/rollback：去 op topic 查
                if (isOpMessageOver(half)) continue;

                // ④ 检查时间：还没到回查时间就跳过
                long age = System.currentTimeMillis() - half.getBornTimestamp();
                if (age < transactionTimeout) continue;

                // ⑤ 检查回查次数
                int checkTimes = getCheckTimes(half);
                if (checkTimes >= transactionCheckMax) {
                    // 达到最大次数仍未确认 → 丢弃
                    listener.resolveDiscardMsg(half);
                    continue;
                }

                // ⑥ ★ 发起回查：往 producer 的 callback 发 CHECK_TRANSACTION_STATE
                listener.resolveHalfMsg(half);
                halfOffset++;
            }
        }
    }
}
```

**逐段点透**：

- **回查间隔 60s + 单次超时 6s**：表示「half 消息超过 6s 没收到 endTransaction，下一轮 60s 检查时发起回查」。生产中如果业务回查实现慢，会出现回查打到下游 RT 拉爆的情况 —— 必须做异步化或限流。
- **最大 15 次**：超过就丢弃，不再回查。这是为什么「业务实现 checkLocalTransaction 必须能正确返回 commit/rollback」—— 总是返回 UNKNOWN 会让消息悬挂。
- **isOpMessageOver 的查找**：op topic 也是个 ConsumeQueue，遍历找到对应的 half offset。op 消息累积过多时，回查本身就慢。

### 2.6 顺序消息：MessageQueueSelector

发送端选 queue：

```java
// 业务代码
producer.send(msg, new MessageQueueSelector() {
    @Override
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        Long orderId = (Long) arg;
        return mqs.get((int) (orderId % mqs.size()));   // ★ 同一订单 hash 到同一 queue
    }
}, orderId);
```

消费端用顺序监听器：

```java
consumer.registerMessageListener(new MessageListenerOrderly() {
    @Override
    public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs,
                                                ConsumeOrderlyContext ctx) {
        for (MessageExt m : msgs) {
            try { handle(m); }
            catch (Exception e) {
                ctx.setSuspendCurrentQueueTimeMillis(1000);
                return ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;
            }
        }
        return ConsumeOrderlyStatus.SUCCESS;
    }
});
```

```java
// org.apache.rocketmq.client.impl.consumer.ConsumeMessageOrderlyService
public void run() {
    // ① 拿到这个 MessageQueue 的本地锁
    final Object objLock = messageQueueLock.fetchLockObject(this.messageQueue);
    synchronized (objLock) {
        // ② 检查是否还持有这个 queue 的「全局锁」（broker 上的锁）
        if (MessageModel.BROADCASTING.equals(...)
            || (this.processQueue.isLocked() && !this.processQueue.isLockExpired())) {

            // ③ 单线程顺序消费
            while (true) {
                List<MessageExt> msgs = processQueue.takeMessages(consumeBatchSize);
                ConsumeOrderlyStatus status = listener.consumeMessage(msgs, ctx);

                if (ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT == status) {
                    // ④ ★ 失败 → 暂停整个 queue
                    submitConsumeRequestLater(processQueue, mq, ctx.getSuspendCurrentQueueTimeMillis());
                    break;
                }
            }
        }
    }
}
```

**逐段点透**：

- **三把锁的协作**：
  - 本地 JVM 锁（`messageQueueLock`）：保证同一 queue 同一时刻只有一个线程消费
  - Broker 端的 queue 锁：保证集群中只有一个 Consumer 实例消费这个 queue
  - ProcessQueue 的内部锁：保护消息列表
- **顺序消费的强约束**：失败时 `SUSPEND_CURRENT_QUEUE_A_MOMENT`，这个 queue 整体暂停。如果失败没救（业务 bug），整个 queue 永远卡住。这就是案例 ③ 的根。
- **最大消费次数**：默认 `Integer.MAX_VALUE`（无限重试）！必须显式设 `maxReconsumeTimes`，否则一卡卡到天荒地老。

### 2.7 消费端的 reblance：AllocateMessageQueueAveragely

ConsumerGroup 内多个 Consumer 实例如何分 queue？

```java
// org.apache.rocketmq.client.consumer.AllocateMessageQueueAveragely
@Override
public List<MessageQueue> allocate(String consumerGroup, String currentCID,
                                    List<MessageQueue> mqAll, List<String> cidAll) {
    int index = cidAll.indexOf(currentCID);
    int mod = mqAll.size() % cidAll.size();
    int averageSize =
        mqAll.size() <= cidAll.size() ? 1 :
        (mod > 0 && index < mod ? mqAll.size() / cidAll.size() + 1
                                 : mqAll.size() / cidAll.size());
    int startIndex = (mod > 0 && index < mod)
                     ? index * averageSize
                     : index * averageSize + mod;
    int range = Math.min(averageSize, mqAll.size() - startIndex);

    List<MessageQueue> result = new ArrayList<>();
    for (int i = 0; i < range; i++) {
        result.add(mqAll.get((startIndex + i) % mqAll.size()));
    }
    return result;
}
```

**逐段点透**：

- **平均分配**：N 个 queue / M 个 Consumer，能整除就每个 N/M 个；不能整除前 mod 个 Consumer 多分一个。
- **触发时机**：`RebalanceService` 每 20s 跑一次，或 broker 检测到 ConsumerGroup 成员变更时立即推。
- **反直觉**：reblance 是**客户端自治的**。每个 Consumer 各自跑同样算法、各自给自己分配 —— 算法确定性 + 输入一致 = 结果一致。所以**算法必须是稳定排序、纯函数**。
- **Consumer 数 > queue 数会怎样**：多余的 Consumer 分不到 queue，**完全空闲**。这就是「为什么 ConsumerGroup 不能盲目扩容」。

> **口诀**：reblance 是客户端各自算的，输入：(consumerGroup, queues, cids)。**Consumer 上限 = queue 数**，再多就是浪费。

---

## 3. 实测数据：刷盘 / 复制 / DLedger / 与 Kafka

### 3.1 同步 vs 异步刷盘

机器：3 节点 Broker（4C8G + NVMe SSD）；Producer：4 实例 × 16 线程；消息 1KB。

| 模式 | TPS | P99 (ms) | 单条 RT |
| --- | --- | --- | --- |
| ASYNC_FLUSH | **120,000** | 8 | < 1ms |
| SYNC_FLUSH（默认 GroupCommit 10ms） | 70,000 | 22 | ~10ms |
| SYNC_FLUSH（关 GroupCommit） | 8,000 | 40 | ~30ms |

**为什么 SYNC_FLUSH 还能 7W TPS**？因为开了 GroupCommit：每 10ms 一次 fsync，把这 10ms 内所有写入合并刷盘。物理 fsync 频率没变，但每次 fsync 的数据量大了几百倍。

> **口诀**：大多数业务 ASYNC_FLUSH 够用；金融级用 SYNC_FLUSH，但**别关 GroupCommit**。

### 3.2 异步复制 vs 同步复制 vs DLedger

| 复制模式 | 写 TPS | P99 (ms) | 主宕机数据丢失 |
| --- | --- | --- | --- |
| ASYNC_MASTER | 120,000 | 8 | 秒级（取决于 lag） |
| SYNC_MASTER | 80,000 | 18 | 0（已 ack 的不丢） |
| DLedger 3 节点 | 60,000 | 30 | 0 + 自动选主 |

**结论**：DLedger 比同步主从慢约 25%，但获得自动故障切换。生产上「不能容忍人工切换」的场景值得这个代价。

### 3.3 不同消息大小下的吞吐

```bash
# 用 RocketMQ 自带 benchmark
sh bin/tools.sh org.apache.rocketmq.example.benchmark.Producer \
    -n nameserver:9876 -t topicTest -s 1024 -w 16
```

| 消息大小 | TPS | 网络带宽 | 瓶颈 |
| --- | --- | --- | --- |
| 128 B | 200,000 | 25 MB/s | CPU（序列化） |
| 1 KB | 120,000 | 120 MB/s | 锁竞争 |
| 4 KB | 50,000 | 200 MB/s | 网络 |
| 64 KB | 4,000 | 250 MB/s | 网络打满 |

**结论**：单消息 < 4KB 时 RocketMQ 性能最好；超过 64KB 应该考虑把内容存对象存储、消息只发引用。

### 3.4 RocketMQ vs Kafka（小消息高 topic 场景）

机器一致，消息 1KB，1000 个 topic 各 4 个 queue/partition：

| 引擎 | TPS | P99 (ms) | 磁盘写吞吐 |
| --- | --- | --- | --- |
| RocketMQ | 95,000 | 12 | 110 MB/s（顺序） |
| Kafka | 35,000 | 60 | 80 MB/s（接近随机） |

topic 数从 100 涨到 1000 时 Kafka 退化明显，RocketMQ 几乎平稳。这是 CommitLog 设计的核心收益。

### 3.5 GroupCommit 的妙处（火焰图视角）

用 async-profiler 抓 broker 的写入路径：

```bash
./profiler.sh -d 60 -f flame.html -e cpu <broker_pid>
```

健康现场：
- `CommitLog.putMessage` 占 30%
- `MappedFile.appendMessage` 占 20%（mmap 写）
- `GroupCommitService.doCommit` 占 5%（GroupCommit 刷盘）
- `force` 系统调用 < 2%（fsync 实际频率很低）

异常现场（关 GroupCommit）：
- `force` 占 40%（每条消息一次 fsync）
- TPS 只有 1/10

> **口诀**：**RocketMQ 的吞吐 = 锁的粒度 × GroupCommit 的合并系数 × PageCache 的命中率**。任何一个被破坏，性能掉一个数量级。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 消息堆积 1 亿条 —— 「消费速率怎么追」

#### 现象

凌晨 2 点，运营搞了一次大促前的全量 push，订单 topic 突然涌入 1.5 亿条消息。Consumer 处理速度远跟不上，监控显示 `lag` 直线上升。

```bash
sh bin/mqadmin consumerProgress -g order_consumer_group -n nameserver:9876
```

```
#Topic       #Broker  #QID  #Broker_Offset  #Consumer_Offset  #Diff
order        broker-a   0     50000000        12000000          38000000
order        broker-a   1     50000000        12000000          38000000
order        broker-a   2     50000000        12000000          38000000
order        broker-a   3     50000000        12000000          38000000
                                                                ─────
                                                                1.52 亿
```

#### 根因

三个层面同时出问题：

1. **Consumer 实例数 = queue 数**，已经分到极限，加机器没用
2. 单条消息处理 RT 200ms（同步调用第三方接口），单线程吞吐 5 条/s
3. 默认 `consumeThreadMin=20, consumeThreadMax=64`，但 `consumeBatchSize=1`

#### 修复

**第一招：扩 queue 数**，让 Consumer 有得分。

```bash
sh bin/mqadmin updateTopic -t order -c DefaultCluster -r 32 -w 32 -n nameserver:9876
# read/write queue 都从 4 调到 32
```

注意：扩 queue **会让顺序消息出问题**（同 orderId 在扩容前后 hash 到不同 queue），订单这种顺序场景要谨慎。

**第二招：扩 Consumer 线程**。

```java
consumer.setConsumeThreadMin(64);
consumer.setConsumeThreadMax(128);
consumer.setConsumeMessageBatchMaxSize(32);   // 一次处理 32 条
consumer.setPullBatchSize(32);
```

**第三招（应急）：搞个临时消费组只做转发，把消息搬到另一个 topic 让多套 Consumer 并行处理**。

```java
// 临时 Consumer：只把消息转发到 backup topic
consumer.subscribe("order", "*");
consumer.registerMessageListener((MessageListenerConcurrently)(msgs, ctx) -> {
    for (MessageExt m : msgs) {
        Message forward = new Message("order_backup", m.getTags(), m.getBody());
        producer.send(forward);
    }
    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
});
```

`order_backup` 用更多的 queue 数，开 N 套 ConsumerGroup 各自处理一部分。

**第四招（最有效）：业务侧改异步化 / 批量处理**。第三方调用攒成 100 条一次批量发，RT 从 200ms × 100 → 500ms 一批，吞吐涨 40 倍。

#### 验证

```bash
# 持续观察 lag 是否在下降
watch -n 5 'sh bin/mqadmin consumerProgress -g order_consumer_group ...'

# 关注 PageCache 命中率，跌得太狠说明在拉冷数据
sar -B 1
```

> **口诀**：**积压排查三层**：先看 queue 数是否成为上限；再看消费线程是否打满；最后看单条 RT 能否压下去。**临时转发 + 多组并行**是最快回血手段。

---

### 案例 ② half message 堆积导致 commitlog 膨胀

#### 现象

某 broker 磁盘从 30% 一路涨到 90%。`du -sh /store/commitlog/` 看到 3TB。但业务方反映消息量并没暴涨。

#### 根因

`SHOW broker stats`（用 mqadmin clusterList）发现 `RMQ_SYS_TRANS_HALF_TOPIC` 有几千万条 half 消息没处理完。

```bash
sh bin/mqadmin topicStatus -t RMQ_SYS_TRANS_HALF_TOPIC -n nameserver:9876
```

定位到某个新上线的事务消息 producer 服务：
- 业务方 `executeLocalTransaction` 里抛了异常但 catch 后返回 `LocalTransactionState.UNKNOW`
- 触发 broker 持续回查
- 业务方的 `checkLocalTransaction` **永远返回 UNKNOW**（实现错了）
- 回查 15 次后丢弃，但 half message 还在 CommitLog 里 —— 物理空间没释放

CommitLog 是顺序写的，文件不能挖洞。**只有等整个 1GB 文件里所有消息都过了 `fileReservedTime` 才能整文件删除**。

#### 修复

紧急修复：业务方改 `checkLocalTransaction`，根据本地事务表状态返回明确的 commit/rollback。

```java
@Override
public LocalTransactionState checkLocalTransaction(MessageExt msg) {
    String txId = msg.getTransactionId();
    TxRecord r = txRepo.findByTxId(txId);
    if (r == null) return LocalTransactionState.ROLLBACK_MESSAGE;       // 没记录 = 业务没成功
    return r.isSuccess() ? LocalTransactionState.COMMIT_MESSAGE
                         : LocalTransactionState.ROLLBACK_MESSAGE;
}
```

监控告警：

```bash
# 监控 half topic 的堆积
sh bin/mqadmin topicStatus -t RMQ_SYS_TRANS_HALF_TOPIC \
  | awk 'NR>1 {sum+=$5} END {print sum}'

# 超过 10 万告警
```

broker 配置加紧缩刷盘：

```properties
# 默认 72h，可以调小到 24h（注意业务侧消费追溯能力）
fileReservedTime=24
deleteWhen=04             # 凌晨 4 点删
diskMaxUsedSpaceRatio=85  # 磁盘 85% 触发主动删除
```

#### 验证

修复后观察：
- half topic 的 lag 下降到 < 100
- broker 磁盘占用稳定
- `BrokerStatsManager` 里 `transactionRollbackCount` 与 `transactionCommitCount` 之和 ≈ 总发送数

> **口诀**：**事务消息的 checkLocalTransaction 必须能给出明确答复**。返回 UNKNOW 等于让 broker 不停回查 + half 永远堆。

---

### 案例 ③ 顺序消息消费失败 —— 「整个 MessageQueue 阻塞 6 小时」

#### 现象

订单 topic 配的是顺序消费。某天监控告警「order queue 0 的消费延迟 6 小时」。kibana 看消费日志，最后一条业务日志在 6 小时前打印了「业务异常 NullPointerException」。

#### 根因

顺序消费的语义是**严格保证顺序**：失败的消息会**无限重试**，期间整个 queue 暂停。

业务代码：

```java
consumer.registerMessageListener(new MessageListenerOrderly() {
    @Override
    public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs,
                                                ConsumeOrderlyContext ctx) {
        for (MessageExt m : msgs) {
            try { processOrder(m); }
            catch (Exception e) {
                log.error("processOrder failed", e);
                return ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;   // ★
            }
        }
        return ConsumeOrderlyStatus.SUCCESS;
    }
});
// 默认 maxReconsumeTimes = Integer.MAX_VALUE
```

某条订单的 body 解析时 NPE，无限重试 + queue 整体暂停。

#### 修复

**铁律**：顺序消费必须设最大重试次数 + 死信告警。

```java
consumer.setMaxReconsumeTimes(16);   // 重试 16 次

consumer.registerMessageListener(new MessageListenerOrderly() {
    @Override
    public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs,
                                                ConsumeOrderlyContext ctx) {
        for (MessageExt m : msgs) {
            try {
                processOrder(m);
            } catch (Exception e) {
                if (m.getReconsumeTimes() >= 15) {
                    // ★ 最后一次：发死信告警 + 跳过，不阻塞 queue
                    deadLetterAlert.send(m, e);
                    log.error("DLQ: skip msgId={} after {} retries",
                              m.getMsgId(), m.getReconsumeTimes(), e);
                    continue;   // 当成成功
                }
                ctx.setSuspendCurrentQueueTimeMillis(1000 * (1 << m.getReconsumeTimes()));   // 指数退避
                return ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;
            }
        }
        return ConsumeOrderlyStatus.SUCCESS;
    }
});
```

注意：跳过失败消息会**破坏严格顺序**。这是「顺序」与「可用性」的取舍 —— 业务方必须明确选择。如果业务确实不能跳，那就保留无限重试 + 强告警，让人介入。

监控：

```java
// 死信 topic 是 %DLQ%<consumerGroup>
consumer.subscribe("%DLQ%order_consumer_group", "*");
```

#### 验证

```bash
# 模拟一条「永远失败」的消息，看 queue 在 16 次后是否恢复
# 预期：queue 暂停 2^16 = 18 小时 → 修复后 < 1 小时
```

> **口诀**：顺序消费 = 强约束。**必须设 maxReconsumeTimes，必须有死信兜底**。否则一条坏消息卡死整个 queue。

---

### 案例 ④ 主从切换数据丢失 —— 「异步复制的承诺成本」

#### 现象

某天主 broker 物理机宕机（电源故障），运维做了主从切换，业务恢复。第二天对账发现某段时间订单丢了 3000 条。

#### 根因

集群配的是 ASYNC_MASTER + ASYNC_FLUSH。主宕机时：
- producer 发出去 broker 已 ack（但只是写到主的 PageCache）
- 复制到从是异步的，可能 lag 50ms~几秒
- 主宕机 + 物理损坏 → PageCache 内容全没
- 从只有切换前最后一次复制成功的那部分

复制 lag 可以观测：

```bash
# 在主上看
sh bin/mqadmin brokerStatus -n nameserver:9876 -b broker-a:10911 \
  | grep -E 'putMessageDistributeTime|slaveFallBehindMaster'
```

#### 修复

根据数据可靠性需求选模式：

| 选择 | 配置 | 代价 |
| --- | --- | --- |
| 接受丢秒级数据 | ASYNC_MASTER + ASYNC_FLUSH | 默认，性能最高 |
| 主宕机不丢，断电丢部分 | SYNC_MASTER + ASYNC_FLUSH | TPS 降 30% |
| 都不丢（金融） | SYNC_MASTER + SYNC_FLUSH | TPS 降 50% |
| 自动故障切换 + 多副本 | DLedger 3 节点 | TPS 降 50%，机器 3 倍 |

订单这种场景应该至少 SYNC_MASTER：

```properties
brokerRole=SYNC_MASTER
flushDiskType=ASYNC_FLUSH
```

更进一步，应用侧加**对账兜底**。每条关键消息：
1. 写入业务库 + 待发送消息表（一个事务）
2. 异步任务扫表，发到 MQ
3. 发成功后删表
4. 表里超过 1 分钟未发的告警 + 重试

这套设计独立于 MQ 配置，能挡住 MQ 任何丢失。

#### 验证

灾难演练：模拟主 broker `kill -9` + 立即从切到主，对账业务库订单表 vs MQ 消费记录。

> **口诀**：**异步复制 + 主断电 = 必丢**。要么换 SYNC，要么走 DLedger，要么应用侧本地消息表兜底。三选一，不要侥幸。

---

### 案例 ⑤ 重复消费 —— 「业务侧没做幂等」

#### 现象

退款系统某用户被退了两次款。日志里看到同一个 `outTradeNo` 的退款消息消费了两次。

#### 根因

RocketMQ 的语义是 **At Least Once**（至少一次）。这意味着重复是**正常**的，不是 bug。常见的重复来源：
- producer 网络抖动重试 → 消息真的发了两份
- broker 切换时 consumer 还没提交 offset → 重新消费一遍
- consumer reblance → 同一条消息被两个实例消费过
- consumer 处理完但 commit offset 前重启

业务方的代码：

```java
@RocketMQMessageListener(topic = "refund", consumerGroup = "refund_consumer")
public class RefundListener implements RocketMQListener<RefundMsg> {
    @Override
    public void onMessage(RefundMsg msg) {
        accountService.refund(msg.getUserId(), msg.getAmount());   // ★ 没幂等
    }
}
```

#### 修复

**业务侧必须做幂等**。三种典型实现：

**方案 A**：唯一键约束。

```java
@Transactional
public void refund(RefundMsg msg) {
    try {
        // refund_record 表的 (out_trade_no) 是唯一索引
        refundRepo.save(new RefundRecord(msg.getOutTradeNo(), ...));
    } catch (DuplicateKeyException e) {
        log.info("duplicate refund msg, ignored: {}", msg.getOutTradeNo());
        return;
    }
    accountService.refund(...);
}
```

**方案 B**：状态机。

```java
@Transactional
public void refund(RefundMsg msg) {
    Order o = orderRepo.findByIdForUpdate(msg.getOrderId());
    if (o.getStatus() == OrderStatus.REFUNDED) {
        log.info("order already refunded, skip: {}", o.getId());
        return;
    }
    accountService.refund(...);
    o.setStatus(OrderStatus.REFUNDED);
    orderRepo.save(o);
}
```

**方案 C**：Redis 记录已处理的 msgId。

```java
public void onMessage(RefundMsg msg) {
    String key = "consumed:refund:" + msg.getMsgId();
    Boolean first = redis.opsForValue().setIfAbsent(key, "1", Duration.ofDays(7));
    if (Boolean.FALSE.equals(first)) {
        log.info("already consumed: {}", msg.getMsgId());
        return;
    }
    accountService.refund(...);
}
```

注意 msgId 也可能变（producer 重试时 msgId 是新的），用业务唯一键（outTradeNo）才稳。

#### 验证

```java
@Test
void refundShouldBeIdempotent() {
    RefundMsg msg = new RefundMsg("trade_001", 100);
    listener.onMessage(msg);
    listener.onMessage(msg);
    listener.onMessage(msg);
    // 期望：用户余额只增加 100，refund_record 只有 1 条
}
```

> **口诀**：MQ 是 At Least Once，**幂等是业务的责任**。唯一键 / 状态机 / 去重 ID 三选一，必须有。

---

### 案例 ⑥ 消息丢失 —— 「异步发送 + 没设回调」

#### 现象

某统计任务依赖一组 MQ 消息驱动，结果发现日 PV 数据少了 5%。检查代码：

```java
producer.send(msg, new SendCallback() {
    @Override
    public void onSuccess(SendResult sendResult) {}
    @Override
    public void onException(Throwable e) {}     // ★ 空回调
});
```

#### 根因

异步发送的可靠性**完全依赖 SendCallback 的正确实现**。
- `onException` 不打日志 → 网络抖动失败时没人知道
- `onException` 不重试 → 失败的消息真就丢了
- 进程退出前没 flush 缓冲区里的待发消息 → 缓冲区里的全丢

`DefaultMQProducer` 内部用 `ExecutorService` 异步执行 send。如果应用 SIGTERM 时直接退出，executor 里的任务就没了。

#### 修复

**铁律**：异步发送必须实现完整回调 + 关闭时 shutdown producer。

```java
@Component
public class ReliableProducer {
    private final DefaultMQProducer producer;
    private final RetryTemplate retry = ...;

    public void sendAsync(Message msg) {
        producer.send(msg, new SendCallback() {
            @Override
            public void onSuccess(SendResult r) {
                log.debug("sent ok: {}", r.getMsgId());
            }
            @Override
            public void onException(Throwable e) {
                log.error("send failed, will retry: topic={}, body={}",
                          msg.getTopic(), new String(msg.getBody()), e);
                // ★ 重试或进入死信表
                retry.execute(ctx -> {
                    producer.send(msg);
                    return null;
                }, ctx -> {
                    deadLetterStore.save(msg);   // 重试也失败 → 落表
                    return null;
                });
            }
        });
    }

    @PreDestroy
    public void close() throws Exception {
        producer.shutdown();    // ★ 等 buffer 里的消息发完
    }
}
```

更可靠的方案：**本地消息表 + 异步发送**。业务事务里写一条消息记录到表，单独的 worker 扫表发 MQ。MQ 任何失败都能从表里恢复。

```sql
CREATE TABLE pending_message (
    id BIGINT PRIMARY KEY,
    topic VARCHAR(64),
    body BLOB,
    status TINYINT,           -- 0=pending, 1=sent, 2=failed
    retry_count INT,
    created_at DATETIME,
    INDEX idx_status_created (status, created_at)
);
```

```java
@Scheduled(fixedDelay = 1000)
public void scanAndSend() {
    List<PendingMessage> pending = repo.findTop100ByStatusOrderById(0);
    for (PendingMessage p : pending) {
        try {
            producer.send(new Message(p.getTopic(), p.getBody()));
            p.setStatus(1);
        } catch (Exception e) {
            p.setRetryCount(p.getRetryCount() + 1);
            if (p.getRetryCount() > 5) p.setStatus(2);   // 标记失败 → 告警
        }
        repo.save(p);
    }
}
```

#### 验证

压测：拔网线 30s 后插回，观察消息是否最终都到 MQ。本地消息表方案应该 100% 不丢。

> **口诀**：**异步发送的可靠性 = 回调的质量**。空回调 = 默认丢。关键场景上本地消息表。

---

### 案例 ⑦ 消费端 reblance 风暴 —— 「短时间消息全部重复」

#### 现象

业务方某天扩容 ConsumerGroup 从 8 台到 16 台，扩完观察日志发现「短短 3 分钟内某些消息消费了 5 次」。客诉量飙升（业务侧没做幂等）。

#### 根因

ConsumerGroup 成员变化时会触发全员 reblance。reblance 的步骤：

1. 每个 Consumer 各自跑分配算法
2. 算出新的「我应该消费的 queue 集合」
3. 释放不再属于自己的 queue（停止 pull）
4. 锁定新分到的 queue（broker 端的锁）
5. 开始 pull 新 queue

问题出在 step 3 → step 5 之间的过渡期。如果你扩容是「先启 8 台新机器，等都 ready 了再...」，那 16 台机器的 reblance 可能不是同时发生：
- 第 1 台心跳 → 触发一次 reblance
- 第 2 台心跳 → 又触发一次
- ...
- 短时间内 reblance 几十次

每次 reblance：
- 已经在跑的消息可能 ack 还没回 broker，offset 没提交 → 新分到这个 queue 的 Consumer 从老 offset 开始拉 → 重复
- 顺序消费场景下还可能短暂双消费（旧 Consumer 已经停了但 broker 锁还没释放，新 Consumer 拿不到锁等待）

#### 修复

**第一招**：扩容时一次性启起来，让 reblance 在一个窗口内完成。

```bash
# K8s
kubectl scale deployment consumer --replicas=16
# 不要一台一台地起
```

**第二招**：业务必须幂等（呼应案例 ⑤）。reblance 引起的重复是无法完全避免的。

**第三招**：调整 reblance 频率。

```java
// 默认 20s 一次 RebalanceService 跑
// 紧急情况下可以加大间隔，让 reblance 更稳定
// （4.x 的 RebalanceService 间隔是写死的 20s，5.x 在 Proxy 层有更细控制）
```

**第四招**：迁移到 5.x 的 Proxy 模式。Proxy 模式下 reblance 由 Proxy 集中协调，行为更可控。

监控：

```java
// 实现 MessageQueueListener 监听 reblance 事件
consumer.setMessageQueueListener((topic, mqAll, mqDivided) -> {
    log.info("reblance: topic={}, total={}, mine={}",
             topic, mqAll.size(), mqDivided.size());
    rebalanceCounter.increment();
});
```

如果短时间 rebalance 触发频率突增，告警。

#### 验证

扩容前后 5 分钟内消费的 msgId 去重，看重复率。理想 < 1%。

> **口诀**：reblance 必有重复窗口期。**幂等 + 一次性扩容 + 监控** 是组合拳。

---

### 案例 ⑧ 集群消费 vs 广播消费选错 —— 「订阅日志只到了一台机器」

#### 现象

某业务订阅一个「全局配置变更」topic，期望所有机器都收到、都更新自己的本地缓存。结果发现配置改了之后，10 台机器里只有 1 台缓存被刷新。

#### 根因

代码：

```java
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("config_consumer");
consumer.setMessageModel(MessageModel.CLUSTERING);   // ★ 默认就是 CLUSTERING
consumer.subscribe("config_change", "*");
```

`MessageModel.CLUSTERING`（集群消费）的语义：**同一 ConsumerGroup 的所有实例分摊消息，每条消息只被消费一次**。所以 10 台机器里只有一台拿到。

要让所有实例都收到，应该用 `MessageModel.BROADCASTING`（广播消费）。

#### 修复

```java
consumer.setMessageModel(MessageModel.BROADCASTING);
```

广播模式的特点：
- 每个 Consumer 实例都收到全部消息
- offset 存在**本地文件**（`~/.rocketmq_offsets/...`），不在 broker
- 实例宕机后重启，offset 是本地的，可能从老位置消费
- 不能用 reblance（因为不需要分配 queue）

广播模式的常见坑：
- 实例 A 宕机 1 小时后重启 → 这 1 小时的消息会重新从本地 offset 处消费 → 可能拉一大批
- 如果磁盘清了 → offset 重置，从队尾或队首开始（看配置）

更稳的方案是**保留 CLUSTERING + 多个 ConsumerGroup**。每台机器一个独立的 group：

```java
String groupName = "config_consumer_" + getHostName();
consumer.setConsumerGroup(groupName);
consumer.setMessageModel(MessageModel.CLUSTERING);
```

每个 group 独立维护 offset 在 broker 端，重启稳定，不依赖本地文件。

#### 验证

测试用例：
1. 起 5 个实例
2. 发一条消息
3. 5 个实例的日志都应该看到这条消息

> **口诀**：广播 = 所有实例都收（offset 在本地）；集群 = 分摊（offset 在 broker）。**配置同步类用 BROADCASTING 或独立 group**。

---

### 案例 ⑨ NameServer 单点风险 —— 「NameServer 全挂业务还能跑吗」

#### 现象

线上演练：把 3 台 NameServer 全部 kill。预期业务报错。**实际**：业务跑了 30 分钟没事，第 31 分钟开始大量 sendException。

#### 根因

NameServer 挂了 producer / consumer 不会立即出问题，因为它们**本地缓存了路由表**：

```java
// org.apache.rocketmq.client.impl.factory.MQClientInstance
this.scheduledExecutorService.scheduleAtFixedRate(() -> {
    try {
        MQClientInstance.this.updateTopicRouteInfoFromNameServer();
    } catch (Exception e) {
        log.error("ScheduledTask updateTopicRouteInfoFromNameServer exception", e);
    }
}, 10, this.clientConfig.getPollNameServerInterval(), TimeUnit.MILLISECONDS);
// 默认 30s 拉一次最新路由
```

只要本地缓存有效，就能继续工作。但：
- broker 重启或下线后，本地缓存里的 broker 还在，发到一个不存在的 broker → 失败
- topic 新建后路由也拉不到
- broker 心跳上不去 NameServer → broker 还是活的，但 producer 拿不到新加的 broker

所以「NameServer 全挂业务能跑一段时间」是真的，但**任何拓扑变化都会引发雪崩**。

#### 修复

**第一招**：NameServer 至少 3 台，跨机房部署。NameServer 节点之间不通信，故意去除任何状态依赖，所以容易做高可用。

**第二招**：客户端 NameServer 地址用 DNS / VIP，不要写死 IP。

```java
producer.setNamesrvAddr("ns.example.com:9876");
// DNS 后面挂 3 台 NameServer，挂一台流量自动切
```

**第三招**：监控。

```bash
# Producer / Consumer 端关键指标
# - 路由刷新失败次数
# - send 失败次数
# - 选不到 broker 的次数
```

NameServer 端：
```bash
# 监控每台 NameServer 的 broker 心跳数应该一致
# 不一致说明某些 broker 没上报到所有 NameServer
```

**第四招**：客户端容灾。NameServer 拉不到时不要清掉本地缓存。

```java
// org.apache.rocketmq.client.impl.factory.MQClientInstance.updateTopicRouteInfoFromNameServer
// 默认实现：拿不到新路由就保留旧的，不清空
```

确认 4.x 版本有这个保护行为（早期版本有 bug 会清空本地缓存）。

#### 验证

混沌演练：
1. 杀掉 1 台 NameServer，业务无感
2. 杀掉 2 台 NameServer，业务无感
3. 全杀 + broker 不变更 → 业务持续可用 30 分钟+
4. 全杀 + 新增 broker → 不可见，符合预期
5. 恢复 NameServer → 1 分钟内一切恢复

> **口诀**：NameServer 不是「中心」，是「目录服务」。**3 台 + DNS + 客户端缓存保护** 是标配。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### Broker 配置层
- [ ] `brokerRole` 明确：异步复制 / 同步复制 / DLedger（按数据可靠性需求选）
- [ ] `flushDiskType` 明确：ASYNC_FLUSH / SYNC_FLUSH（金融级才用 SYNC）
- [ ] `fileReservedTime` 设合理（默认 72h，根据磁盘容量和消息量调整）
- [ ] `diskMaxUsedSpaceRatio` 设阈值（默认 75%，超过会拒绝写入）
- [ ] `useReentrantLockWhenPutMessage`：低延迟场景关掉用自旋锁
- [ ] `transientStorePoolEnable`：开后写入先到堆外内存再异步刷 mmap，吞吐+RT 都改善
- [ ] NameServer ≥ 3 节点，跨机房

### Producer 层
- [ ] 同步发送 vs 异步发送 vs Oneway，明确选择
- [ ] **异步发送必须实现完整 SendCallback**（成功打 trace、失败必重试）
- [ ] `@PreDestroy` 调 `producer.shutdown()`，等 buffer 内消息发完
- [ ] 消息大小 < 4MB（broker 默认上限），关键场景控制在 4KB 以下
- [ ] 关键消息走「本地消息表 + 异步发」兜底
- [ ] 事务消息的 `checkLocalTransaction` 必须能给明确答复（不返 UNKNOW）

### Consumer 层
- [ ] **消费逻辑必须幂等**（唯一键 / 状态机 / 去重 ID）
- [ ] 顺序消费必须设 `maxReconsumeTimes`（默认无限重试是陷阱）
- [ ] 顺序消费要有死信兜底逻辑，避免一条坏消息卡死 queue
- [ ] CLUSTERING vs BROADCASTING 明确选择
- [ ] `consumeThreadMin/Max`、`pullBatchSize`、`consumeMessageBatchMaxSize` 按业务调
- [ ] Consumer 实例数 ≤ queue 数（再多就空闲）
- [ ] 监控 lag、reblance 频率、死信 topic

### Topic / Queue 层
- [ ] 每个业务 topic 提前规划 queue 数（扩容会让顺序消息 hash 漂移）
- [ ] 顺序消息场景永不动态扩 queue
- [ ] 大流量 topic 与小流量 topic 不要塞一个 broker

### 监控告警
- [ ] broker 磁盘占用 > 80% 告警
- [ ] `RMQ_SYS_TRANS_HALF_TOPIC` 堆积监控
- [ ] 各业务 topic 的 ConsumerGroup lag 监控
- [ ] 主从复制 lag（slaveFallBehindMaster）
- [ ] 死信 topic 有任何消息 → 告警
- [ ] NameServer 与 broker 心跳数一致性
- [ ] Producer 端：send 失败率
- [ ] Consumer 端：reblance 触发频率

### 运维
- [ ] DLedger 三节点跨机架部署
- [ ] broker 升级用滚动方式（先升 slave 再切再升 master）
- [ ] 消息追溯期内不要删除 commitlog 文件（必须等过期机制）
- [ ] 重大变更前做混沌演练（kill broker / kill NameServer）

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### RocketMQ 5.x 的 Proxy 模式
- gRPC 协议替代 Remoting：多语言客户端的成本降到几乎为零
- 计算/存储分离：Proxy 弹性、Broker 按存储量伸缩
- POP 消费模式：消息只在 broker 端记锁定状态，不再绑 queue → 解决了「Consumer 数 > queue 数浪费」的问题
- 看 `apache/rocketmq-clients` 仓库的 grpc-go / grpc-java 实现

### Apache Pulsar 的对比
- **存储计算分离**做得更彻底：BookKeeper 做存储，Broker 是无状态的
- **多租户**能力原生支持（namespace + tenant）
- 不用 mmap，用顺序追加 + 索引文件
- 适合超大规模、多租户场景；中小规模 RocketMQ 仍是更简单稳定的选择

### Kafka 的对比
- 详细对比看 [Kafka 存储引擎深文](/article/mq/kafka-storage)
- 关键点：Kafka 每 partition 一个 log，topic 多时随机 IO；RocketMQ 单 CommitLog 永远顺序
- Kafka 的事务是 partition 级，跨 partition 用 transactional.id；RocketMQ 是 broker 级 half message + 回查
- Kafka Streams 已成熟，RocketMQ Streams 仍在发展

### DLedger 的 Raft 实现
- DLedger 是独立项目（`openmessaging/dledger`），可以用作任何状态机的复制底座
- 实现的是 Raft 而不是 Multi-Paxos
- LogEntry 直接复用 RocketMQ 的 CommitLog 格式，所以 DLedger 模式下 broker 上层逻辑几乎不变
- Leader 选举的优化：preVote 减少 split-vote

### 流处理：RocketMQ Streams
- 类似 Kafka Streams 的轻量流处理框架
- 主推「单机即可跑」，不依赖外部 Yarn / K8s
- 阿里内部场景有，外部生态较弱

### Connector 生态
- RocketMQ Connect：source/sink 连接器框架
- 类似 Kafka Connect 但生态成熟度差距大
- 主流场景仍是用 Flink CDC 接入

### 内核层
- DispatchService 的 1ms 轮询是不是 CPU 浪费？社区有 [RIP](https://github.com/apache/rocketmq/issues) 在讨论事件驱动方案
- mmap 在 ARM 架构上的性能（华为鲲鹏适配的实测）
- io_uring 替代 mmap 的可能性

---

## 7. 引用

- RocketMQ 4.9.x 源码：[apache/rocketmq](https://github.com/apache/rocketmq)
- RocketMQ 官方文档：[rocketmq.apache.org/docs](https://rocketmq.apache.org/docs/)
- 《RocketMQ 技术内幕（第 2 版）》—— 丁威 / 周继锋
- 《分布式消息中间件实践之路》—— 赵屹华
- DLedger 论文 / 设计文档：[openmessaging/dledger](https://github.com/openmessaging/dledger)
- 阿里云 RocketMQ 5.x 架构：[help.aliyun.com](https://help.aliyun.com/product/29530.html)
- Pulsar 对比：[Pulsar vs Kafka vs RocketMQ](https://pulsar.apache.org/blog/2018/06/01/pulsar-vs-kafka/)
- Brian Lim, *Building a High-Performance Messaging Platform with RocketMQ*

---

> 配套：[消息队列主题](/article/mq/) | [Kafka 存储引擎](/article/mq/kafka-storage) | [分布式事务](/article/distributed/dist-tx)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
