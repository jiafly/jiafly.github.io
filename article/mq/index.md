# 消息队列

> Kafka / RocketMQ / Pulsar 不是"高吞吐 + 削峰填谷"几个口号。这一节要讲的是：**生产里真正会踩的一致性 / 顺序 / 重复 / 积压问题**。

---

## 为什么单独开这块

绝大多数 MQ 文章都是产品介绍或 API 教程。但你在生产里要回答的是：
- 消息到底丢没丢？怎么证明？
- 消费端一致性 vs 性能怎么平衡？
- 顺序消息在分区扩容时怎么办？
- 积压了 1 亿条怎么追？
- 事务消息真的能保证最终一致吗？

这一节按 [桃花岛深度模板](/STYLE) 切，每篇都要走完**协议设计 + 源码切片 + 失败注入实测 + 生产事故复盘**。

---

## 已完成的深文

- ✅ **[Kafka 存储引擎 —— 顺序写、PageCache、零拷贝与 ISR 的真实代价](/article/mq/kafka-storage)** · 1504 行
- ✅ **[RocketMQ 内核 —— 存储设计、事务消息、顺序消息与积压排查](/article/mq/rocketmq-deep)** · 1525 行

---

## 规划中的深文

### 1. Kafka 存储引擎 ✅ 已完成 → [阅读深文](/article/mq/kafka-storage)

**核心问题**：Kafka 真的"顺序写比内存随机写快"吗？为什么大消息会让 Kafka 卡死？`acks=all + min.insync.replicas` 真的不丢消息？

深文涵盖：日志结构 / Segment+Index 布局 / sendfile 零拷贝路径 / ISR/HW/LEO 三水位 / Kafka 3.x KRaft 源码切片 / 9 个生产事故（大消息→OOM、ISR 抖动、min.insync.replicas 算错、Compaction tombstone 陷阱、KRaft Controller 切换、Reblance 风暴等）/ 30+ 条上线清单 / 7 个延伸方向。

### 2. RocketMQ 内核与事务消息 ✅ 已完成 → [阅读深文](/article/mq/rocketmq-deep)

包含原本规划的 5. 积压排查内容 —— "最终一致"的内部机理

**核心问题**：RocketMQ 事务消息号称解决分布式事务，它和本地消息表 / TCC 的本质差别在哪？回查机制能不能保证 100% 一致？

**写作骨架**：
- **原理推导**：half message 的设计；prepare → commit/rollback 的两阶段；回查的兜底
- **源码切片**：`TransactionMessageServiceImpl.prepareMessage`；`EndTransactionProcessor.processRequest`
- **实测数据**：成功率 / 回查命中率 / 端到端延迟；故障注入下的恢复时间
- **生产案例**：half message 堆积导致 commit log 膨胀；回查接口写得不幂等导致重复消费；本地事务长事务卡住整个 producer
- **延伸**：与 Seata MT 模式的对比、Pulsar 的事务消息实现、Kafka 的 EOS 模型

### 3. 顺序消息 —— 分区扩容时怎么不破坏顺序

**核心问题**：Kafka 用同一个 partition 保证顺序，但你想从 4 partition 扩到 8，原本顺序的消息会被打乱吗？怎么不停服扩容？

**写作骨架**：
- **原理推导**：分区即顺序边界；扩容打散 hash 的本质矛盾；"按 key 顺序"vs"全局顺序"的选择
- **源码切片**：Kafka producer 的 `DefaultPartitioner.partition`；consumer 的 reblance 协议
- **实测数据**：扩容期间 hash 漂移的实际影响范围；自定义 partitioner 的稳定性
- **生产案例**：电商订单状态消息错序、IM 群消息错序、扩容期间消费 lag 暴涨
- **延伸**：Pulsar 的 Key_Shared 订阅模式（保 key 顺序又能扩消费）、RocketMQ 的 MessageQueueSelector

### 4. 消息重复与幂等 —— 从 Producer 端到 Consumer 端的全套方案

**核心问题**：消息重复几乎不可避免（生产端重试、消费端 ack 失败）。Kafka 的"恰好一次"（EOS）真的恰好一次吗？业务侧怎么做幂等？

**写作骨架**：
- **原理推导**：At-most-once / At-least-once / Exactly-once 三者的物理边界；幂等的几种粒度
- **源码切片**：Kafka idempotent producer 的 PID + sequence；Kafka transactions 的 marker 机制
- **实测数据**：开 EOS 后吞吐下降百分比；幂等表方案的 DB 压力
- **生产案例**：扣款重复导致用户投诉；幂等 key 设计错误（用消息 ID vs 用业务 ID）；消息 offset 提交时机错误导致重消
- **延伸**：流处理中的幂等（Flink 的两阶段提交 sink）、Outbox 模式

### 5. 积压排查 —— "几亿条消息怎么追"

**核心问题**：业务半夜报警：消费延迟 4 小时，积压 2 亿条。怎么定位瓶颈、怎么扩容、怎么避免下次？

**写作骨架**：
- **原理推导**：消费速率 = 单消费者处理速度 × 消费者数（受限于分区数）；积压的几个根因层级
- **源码切片**：Kafka consumer 的 fetch.max.bytes / max.poll.records 调参依据；并发消费模型
- **实测数据**：单消费者吞吐随业务处理时长的曲线；多线程消费 vs 多 consumer 实例的差异
- **生产案例**：消费端 DB 慢导致积压、分区数太少无法并行、reblance 风暴让积压更严重
- **延伸**：临时扩消费的"影子 topic"模式、批量消费 + 异步处理的工程实践

### 6. MQ 选型 —— Kafka / RocketMQ / Pulsar / RabbitMQ 真实对比

**核心问题**：千篇一律的"Kafka 适合大数据、RabbitMQ 适合业务"是廉价的。在 2026 年的真实场景，每个 MQ 的最佳定位是什么？

**写作骨架**：
- **原理推导**：4 种 MQ 的存储模型 / 主从模型 / 协议 / 客户端语言生态对比矩阵
- **运维成本**：扩缩容难度、监控成熟度、社区活跃度
- **实测数据**：在相同硬件下 4 种 MQ 的吞吐 / 延迟 / 资源消耗
- **生产案例**：从 RabbitMQ 迁到 Kafka 的踩坑、Pulsar 多租户隔离失败的事故、RocketMQ 4 vs 5 的版本陷阱
- **延伸**：Iceberg + Flink 替代 MQ 的趋势、AutoMQ / WarpStream 的 S3 后端模型

---

## 关联现有文章
- [线程池生产实战](/article/java/threadpool) —— 消费端线程模型
- [分布式系统](/article/distributed/) —— 事务消息的对比基础

## 想看哪一篇？

[告诉我](/article/about) 你最想先看哪个主题的深文，我会优先排期。
