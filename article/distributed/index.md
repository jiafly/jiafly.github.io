# 分布式系统

> 不讲三句话八股（CAP / BASE / AP/CP）。这一节要讲的是：写代码遇到分布式问题时**为什么这么设计、错过哪些 trade-off、生产事故怎么定位**。

---

## 为什么单独开这块

99% 的「分布式系统」博客停留在概念背诵 —— CAP 三选二、BASE 软状态、AP/CP 选型。但你团队选 etcd 而不选 ZK 时、做秒杀加分布式锁时、迁移到 TCC 时，这些概念帮不上忙。

这一节按 [桃花岛深度模板](/STYLE) 切，每篇深文都包含：
- 设计的真实推导（Lamport / Ongaro / Brewer 论文里的关键决策）
- 主流开源库（etcd / TiKV / Seata / DTM）的源码切片
- 实测的可用性 / 延迟 / 吞吐数据
- 生产事故复盘：脑裂、活锁、TCC 悬挂、Saga 补偿失败

---

## 已完成的深文

- ✅ **[共识算法 —— 从 Paxos 到 Raft 的工程实现与生产事故](/article/distributed/consensus)** · 1271 行
- ✅ **[分布式锁 —— Redis / ZooKeeper / etcd 全景与脑裂复盘](/article/distributed/dist-lock)** · 1598 行
- ✅ **[分布式事务 —— 2PC / TCC / Saga / 本地消息表的真实选型与事故](/article/distributed/dist-tx)** · 1579 行
- ✅ **[服务注册与发现 —— Eureka / Nacos / Consul / etcd / ZK 全对比](/article/distributed/service-discovery)** · 1396 行
- ✅ **[Spring Cloud vs Dubbo —— 微服务全家桶选型与生产事故](/article/distributed/spring-cloud-dubbo)** · 1547 行

---

## 规划中的深文

### 1. 共识算法 ✅ 已完成 → [阅读深文](/article/distributed/consensus)

**核心问题**：你团队选用 etcd / Consul / TiKV，发生网络分区时谁来选主？term 跳变意味着什么？为什么 Raft 的工程实现比 Diego Ongaro 论文复杂十倍？

**写作骨架（5 层）**：
- **原理推导**：Paxos 的根本难点在哪 → Raft 怎么把它拆成 Leader Election / Log Replication / Safety / Membership Change 四块
- **源码切片**：etcd Raft 库的 `becomeFollower / becomeCandidate / becomeLeader` 状态机；TiKV 的 PreVote 优化
- **实测数据**：3 节点 / 5 节点 / 7 节点集群的写延迟与可用性曲线；网络抖动下的 term 跳变频率
- **生产案例**：磁盘满导致的活锁、PreVote 缺失导致的频繁切主、Snapshot 期间的脑裂、混合时钟漂移
- **延伸**：Multi-Raft / Raft 工程化（PolarDB-X / TiKV）、Paxos 变种（EPaxos / Mencius）

### 2. 分布式锁 ✅ 已完成 → [阅读深文](/article/distributed/dist-lock)

**核心问题**：Redis SETNX 真能做分布式锁？Redlock 论文之争（Martin Kleppmann vs antirez）谁对？什么场景必须用强一致存储？

**写作骨架**：
- **原理推导**：互斥锁的"安全性 vs 活性"、租约（lease）的本质、fencing token 是什么以及为什么必须有
- **源码切片**：Redisson `RLock` 的 lua 脚本与 watchdog 续期；Curator `InterProcessMutex` 的 ZK ephemeral + sequential 模式；etcd `concurrency.NewMutex` 的 lease + revision 机制
- **实测数据**：三种实现在 GC pause、网络分区、节点宕机下的"丢锁率"与"重复持锁率"
- **生产案例**：Redis 主从切换导致重复持锁、Zookeeper session 超时假死、etcd revision 错位
- **延伸**：Chubby 论文、PG advisory lock、悲观锁 vs 乐观锁的边界

### 3. 分布式事务 ✅ 已完成 → [阅读深文](/article/distributed/dist-tx)

**核心问题**：跨服务下单要扣库存 + 扣余额 + 加积分。XA 性能太差，TCC 改造成本高，Saga 补偿写不对，本地消息表又有顺序问题 —— 怎么选？

**写作骨架**：
- **原理推导**：CAP 在事务上的体现 → 强一致（XA / Percolator）vs 最终一致（TCC / Saga / 消息表）的本质差别
- **源码切片**：Seata AT / TCC / Saga / XA 四种模式的核心代码路径；DTM 的子事务屏障实现
- **实测数据**：四种模式在 1KQPS / 10KQPS 下的吞吐与延迟；失败注入下的恢复时间
- **生产案例**：TCC 悬挂（Try 后 Cancel 先到）、空回滚、Saga 反向补偿失败的人工兜底、消息表的"漏发"vs"重发"权衡
- **延伸**：Percolator（Google 2010）、Calvin、Spanner 的 TrueTime + Paxos 事务

### 4. 注册中心与服务发现 ✅ 已完成 → [阅读深文](/article/distributed/service-discovery)

**核心问题**：注册中心 CP 还是 AP？为什么 Netflix 弃用 ZK 改 Eureka，又被 Spring Cloud 弃用？多机房注册中心怎么部署不脑裂？

**写作骨架**：
- **原理推导**：服务发现的本质 = 一个高可用的 KV 存储 + 健康检查 + 推/拉模型；CP（强一致）和 AP（高可用）的真实代价
- **源码切片**：Nacos 的 AP（Distro 协议）+ CP（JRaft）双模实现；Eureka 的自我保护机制
- **实测数据**：万级实例规模下的推送延迟、注册风暴下的雪崩
- **生产案例**：跨机房注册中心脑裂、心跳丢失误下线、客户端缓存陈旧导致的请求黑洞
- **延伸**：Service Mesh（Envoy + Istio）下注册中心的演变、xDS 协议

### 5. 分布式 ID —— 雪花、号段、UUID 的真实成本

**核心问题**：分库分表下要全局唯一 ID。雪花算法时钟回拨怎么办？号段方案的水位怎么设？为什么主流互联网公司都不用 UUID？

**写作骨架**：
- **原理推导**：单调性 / 唯一性 / 趋势递增 三者不可兼得的几何
- **源码切片**：百度 UidGenerator 的 RingBuffer 设计；美团 Leaf 的 segment + snowflake 双模式
- **实测数据**：四种方案在 IO / CPU / 内存上的开销
- **生产案例**：时钟回拨事故（含 NTP 同步失败、容器迁移）、号段耗尽、UUID 索引退化
- **延伸**：Twitter 原版 Snowflake、Sony Sonyflake、Instagram 的 PG 方案

### 6. 一致性哈希与数据分片

**核心问题**：缓存集群扩容时为什么不用普通 hash？一致性哈希的"虚拟节点数"怎么定？rebalance 期间的请求路由怎么办？

**写作骨架**：
- **原理推导**：普通 hash 扩容 → 全失效；一致性哈希 → 仅 1/N 失效；带虚拟节点 → 负载均衡度
- **源码切片**：Memcached 客户端的 ketama；Redis Cluster 的 16384 slot 模型；TiKV 的 Region 分裂
- **实测数据**：物理节点 vs 虚拟节点不同倍数下的负载方差
- **生产案例**：扩容期间缓存击穿、热点 key 倾斜、数据迁移半截宕机
- **延伸**：Jump Hash（Google）、Maglev（Google LB）、CRUSH（Ceph）

---

## 关联现有文章
- [Redis 核心](/article/db/redis) —— 分布式缓存基础
- [MySQL MVCC 与锁深文](/article/db/mysql-mvcc) —— 单机事务，对比理解分布式事务的复杂度

## 想看哪一篇？

[告诉我](/article/about) 你最想先看哪个主题的深文，我会优先排期。
