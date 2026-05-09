# 微服务拆分演进 —— 从单体到 Strangler，到 Cell-based 架构

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果只想刷面试，看 [架构总览](/article/architecture/) 就够了。
> 这里要回答的，是**当 leader 拍板"我们要做微服务化"，你拿着祖传的 80 万行单体不知道从哪刀下去**时你需要的那部分。

---

## 0. 读这篇之前 —— 8 个自检题

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. 团队 6 个人，单体跑得好好的，老板说"竞品都微服务了，我们也拆"。你能列出 3 条**反对**拆分的理由吗？
2. 你拆了 30 个服务，但每次发版还是要"按依赖顺序部署 28 个"，改一个字段要协调 5 个团队 —— 这是分布式单体（Distributed Monolith）。**怎么定量判断**自己是不是落入了这个反模式？
3. Strangler Pattern 听起来很美：旧系统外面套个 Proxy，逐步把流量切到新服务。**实操中**，路由层的"双跑校验"怎么做？校验不一致时谁是 ground truth？
4. 一个聚合根（Aggregate Root）跨两个服务，下单时要同时改 Order 和 Inventory。你的方案是 2PC、Saga、本地消息表，还是干脆别拆？给出**判断标准**。
5. 团队按 Conway's Law "组织决定架构"。**反过来**呢？你设计了一个"用户中心服务"，但**没有任何团队 own 它**，会发生什么？
6. 服务拆得越细越好？给一个**反例**：拆到什么粒度时，分布式调用的延迟会吃掉所有"独立部署"的好处？
7. Cell-based（单元化）架构 —— 阿里、AWS 在用 —— 它和"微服务"是什么关系？是更细的拆分还是另一个维度？
8. 你接手一个有 200 个微服务的系统，发现 30 个服务过去 90 天**没有任何流量**。它们是"死的"还是"冷的"？怎么安全下线？

如果有半数答不上，往下看。

---

## 1. 底层模型

### 1.1 单体 → 微服务的三个真实驱动力

绝大多数关于"为什么拆微服务"的文章会列十条好处：可扩展性、可维护性、技术栈自由……这些是**结果**，不是**原因**。真正驱动一个团队拆单体的，只有三个：

**驱动力一：团队规模超过沟通带宽。**
单体最致命的不是技术问题，是**合并冲突 + 协同发布**。10 个人共用一个仓库还行，50 个人就开始每周吵"你这个改动 break 了我的测试"。

**驱动力二：业务模块的扩缩容需求差异巨大。**
搜索接口 QPS 5000，结算接口 QPS 50。单体只能整体扩容，浪费的是钱。

**驱动力三：故障域需要隔离。**
营销活动 OOM 不能拖垮交易。单体的 JVM 是一个故障域。

如果你拆服务的理由不在这三条里 —— 比如"听说微服务先进"、"简历好看"、"老板要求" —— **大概率拆完会后悔**。

### 1.2 微服务的"分布式系统"成本

拆完会得到什么？得到一个分布式系统。分布式系统不是"几个进程跑在不同机器上"那么简单，它带来一组**根本性的新问题**：

| 维度 | 单体 | 微服务 |
| --- | --- | --- |
| 函数调用 | ns 级，永不失败 | ms 级，会超时、会重试、会乱序 |
| 事务 | 一个 `BEGIN/COMMIT` 搞定 | 跨服务要 Saga / TCC / 本地消息表 |
| 数据查询 | `JOIN` 三张表 | 跨服务聚合 / CQRS / 数据冗余 |
| 部署 | 一个包 | N 个服务的版本兼容矩阵 |
| 调试 | 单机 stack trace | 链路追踪 + 分布式日志 |
| 测试 | JVM 内集成测试 | 契约测试 + 端到端环境 |

每一条都不是"加点工具就能解决"的小事 —— 每一条都是**整个团队多花 20%~50% 时间**的长期负担。

> 在拆之前，**先把这一列写下来贴到墙上**。如果你的团队没准备好为这些买单，先别拆。

### 1.3 Modular Monolith：第三种选择

绝大多数"应该拆微服务"的理由，**真正想要的是模块化**，而不是分布式。Modular Monolith（模块化单体）就是为这个场景诞生的：

```
Monolith (传统)              Modular Monolith              Microservices
   ┌──────────────┐           ┌──────────────────┐        ┌────┐ ┌────┐ ┌────┐
   │   全部混在    │           │ ┌─────┐ ┌─────┐ │        │ S1 │ │ S2 │ │ S3 │
   │   一起的代码  │           │ │ Mod1│ │ Mod2│ │        │    │ │    │ │    │
   │              │           │ └──┬──┘ └──┬──┘ │        └─┬──┘ └─┬──┘ └─┬──┘
   │              │           │    │ API   │    │          │      │      │
   │              │           │ ┌──┴──┐ ┌──┴──┐ │          └──────┴──────┘
   │              │           │ │ Mod3│ │ Mod4│ │           跨网络调用
   │              │           │ └─────┘ └─────┘ │
   └──────────────┘           └──────────────────┘
   一个进程 + 混乱             一个进程 + 强边界           N 个进程 + 强边界
```

Modular Monolith 的特点：
- **物理上一个进程**：函数调用、本地事务、共享 JVM
- **逻辑上强边界**：模块间只通过显式 API（不能跨模块直接 import 实现类）
- **数据库可分 schema**：每个模块独立 schema，留好"未来拆分"的余地

Google 在 2023 年开源的 [Service Weaver](https://serviceweaver.dev/) 就是把这个理念做到极致：**写 modular monolith，部署时由框架决定哪些模块跑在同一进程、哪些跑在不同进程**。

什么时候选 Modular Monolith 而不是微服务？

- 团队 < 30 人
- 业务边界还在快速变化（拆错了改起来贵）
- 没有专门的 SRE / 平台团队

> **口诀**：能用 Modular Monolith 解决的问题，不要用微服务解决。微服务是手段不是目标。

### 1.4 服务边界的几个理论

#### Bounded Context（限界上下文）

来自 Eric Evans 的 DDD。核心观点：**同一个名词在不同业务场景下含义不同，每个含义对应一个 Bounded Context**。

举例：电商系统里的"商品"

| Context | "商品"是什么 |
| --- | --- |
| 商品中心 | SKU + SPU + 标题 + 详情 + 类目 |
| 订单 | 下单时的快照（商品名、价格、规格） |
| 库存 | 一个可用数量 |
| 营销 | 打折规则的目标 |
| 推荐 | 一个 embedding 向量 |

强行做一个"商品"统一模型 = 上帝模型 = 拆不动。承认每个 Context 有自己的"商品" = 天然的服务边界。

#### Conway's Law（康威定律）

> Any organization that designs a system will produce a design whose structure is a copy of the organization's communication structure.
> —— Melvin Conway, 1968

通俗版：**软件架构会长成组织架构的样子**。

实操版：
- 你有 3 个团队，长期来看就是 3 个服务（即使你今天画了 30 个）
- 想要 30 个服务，先有 30 个团队（或至少 30 个明确的 ownership 单元）
- "Inverse Conway Maneuver"：先按目标架构组织团队，再让架构自然长出来

#### "披萨原则"（Two-Pizza Team）

亚马逊 Bezos 提出：**一个服务的 owner 团队应该小到两个披萨能喂饱（约 6~10 人）**。

这条原则给了"服务粒度"一个量化标准：
- 一个服务 < 1 个披萨团队 → 太碎（运维成本高于收益）
- 一个服务 > 2 个披萨团队 → 太大（拆）

> **口诀**：服务边界 = Bounded Context ∩ 团队边界 ∩ 披萨团队规模。三个都对了再下刀。

### 1.5 Strangler Pattern：渐进式替换

Martin Fowler 2004 年提出，名字来自南美的"绞杀榕"（Strangler Fig）—— 它会缠绕在宿主树上慢慢长大，最后宿主死掉，绞杀榕成型。

应用到软件：

```
阶段 0：纯单体
  Client → [Monolith]

阶段 1：套上路由层（Proxy / API Gateway / Facade）
  Client → [Proxy] → [Monolith]
  此时 Proxy 不做任何事，只透传。但已经具备"切流量"的能力。

阶段 2：第一个新服务上线，路由部分流量
  Client → [Proxy] ─── /orders/* ──→ [Monolith]
                  └─── /users/*  ──→ [NewUserService]

阶段 3：双跑（Shadow / Diff Testing）
  Client → [Proxy] ─── /users/*  ─┬──→ [Monolith]   (主)
                                  └──→ [NewUserService]   (影子)
  对比两边返回，差异写日志，新服务功能完整后切主

阶段 4：把单体里对应模块代码删掉
  Client → [Proxy] ─── /users/*  ──→ [NewUserService]
                  └─── /orders/* ──→ [Monolith (瘦身)]

阶段 N：单体被"绞杀"完毕
```

Strangler Pattern 的关键在于**永远不需要"大爆炸式"的全量切换**。每一步都可回滚。

但实操中有几个深坑（案例 ④ 会展开）：
- 路由切流量时**两边数据怎么同步**？
- 双跑校验**不一致时谁对**？
- 旧代码什么时候真的可以删？

### 1.6 Cell-based 架构（单元化）

微服务回答的是"功能怎么拆"。Cell-based 回答的是"**同一份功能怎么按用户拆**"。

```
微服务（按功能拆）            Cell-based（按用户拆）
┌────┐ ┌────┐ ┌────┐         ┌─────────────────┐    ┌─────────────────┐
│User│ │Ord │ │Pay │         │ Cell-A (用户1-N) │    │ Cell-B (用户N+1-2N)│
└────┘ └────┘ └────┘         │ ┌──┐ ┌──┐ ┌──┐ │    │ ┌──┐ ┌──┐ ┌──┐ │
                              │ │U │ │O │ │P │ │    │ │U │ │O │ │P │ │
                              │ └──┘ └──┘ └──┘ │    │ └──┘ └──┘ └──┘ │
                              └─────────────────┘    └─────────────────┘
                                  独立的 DB             独立的 DB
                                  独立的 K8s 集群        独立的 K8s 集群
```

每个 Cell 是一份**完整的、可独立运行的业务系统**，承接一部分用户。Cell 之间**物理隔离**：独立数据库、独立部署、独立 K8s namespace 甚至独立 region。

为什么要这么做？回答两个问题：
- **故障爆炸半径**：一个 Cell 挂了，只影响自己承接的那部分用户
- **极限扩容**：单个集群有规模上限（数据库连接数、网卡带宽、注册中心条目），切 Cell 就突破了

代价：
- 跨 Cell 的查询/事务非常复杂（基本只能异步对账）
- 路由层（按用户 ID 把请求路由到对应 Cell）必须强一致

**典型场景**：
- 蚂蚁金服：交易按用户分 Cell（LDC，Logical Data Center）
- AWS：内部服务（DynamoDB / Lambda）按 Cell 切流量做灰度
- Slack：按 workspace 切 Cell

> Cell-based 是**微服务的下一层**：你已经把功能拆成微服务了，但还需要把同一组微服务复制成多份独立运行的实例。普通业务用不到，**当你的单实例容量摸到天花板时**才考虑。

### 1.7 反模式：分布式单体（Distributed Monolith）

Sam Newman 在《Building Microservices》里用了一整章批判这个反模式。判定标准：

| 症状 | 你是不是分布式单体 |
| --- | --- |
| 改一个字段要改 3+ 服务 | ✅ |
| 服务必须按某个固定顺序部署 | ✅ |
| CI 里有"先跑 A，等绿了再跑 B" | ✅ |
| 多个服务共享同一张数据库表 | ✅ |
| "升级 X 服务"需要先和 5 个团队对齐 | ✅ |
| 一个服务挂了，其他服务必然挂 | ✅ |

**分布式单体 = 拥有微服务所有的成本（网络调用、部署复杂度、运维负担），但没有微服务任何的好处（独立部署、独立扩缩、故障隔离）**。

它通常是这样产生的：
- 团队没有真正理解"边界"，只是按"代码行数 / 文件数"拆
- 拆完发现耦合在数据库 / 共享库 / 隐式协议里
- 没有契约测试，每次改动都要"全套回归"

> **口诀**：判断微服务是否成功，不看"我们有多少个服务"，看"**改一个业务逻辑能不能由一个团队独立上线**"。

---

## 2. 拆分方法切片

每段含完整工程示例。基于真实项目（Spring Boot + Spring Cloud Gateway + 一些 Python 工具）整理。

### 2.1 Event Storming 工作坊

Event Storming（事件风暴）是 Alberto Brandolini 提出的、目前业界最快找出 Bounded Context 的方法。**两天工作坊能产出一年代码量的边界设计**。

#### 怎么做

准备：一面墙（或大白板软件）、不同颜色的便签。

```
橙色：Domain Event（已经发生的事，过去时）
    例："订单已创建"、"库存已扣减"、"支付已完成"

蓝色：Command（触发事件的指令）
    例："创建订单"、"扣减库存"

黄色：Aggregate（聚合根，被 Command 操作的实体）
    例：订单、库存、用户

紫色：Policy（业务规则，事件 → 触发新 Command）
    例："订单创建后 → 自动扣减库存"

红色：Hot Spot（疑问 / 争议点）
```

流程：
1. 第一阶段（2 小时）：所有人**只贴橙色事件**，按时间顺序贴在墙上
2. 第二阶段（2 小时）：在事件前面贴蓝色 Command 和黄色 Aggregate
3. 第三阶段（2 小时）：找出 Aggregate 之间的关联，画出 Bounded Context 候选边界
4. 第四阶段：投票，确认 N 个 Bounded Context

#### 边界判定规则

一个 Bounded Context 内部应该满足：
- 词汇一致（"订单"在 Context 内只有一个含义）
- 数据强一致（不需要跨 Context 事务）
- 团队归属清晰（一个 Context 只属于一个团队）

#### 实战示例（电商）

```
Domain Events                    Bounded Contexts
─────────────────────────────────────────────────
[商品已上架]  [商品价格已变更]  →  Catalog Context
[订单已创建]  [订单已支付]      →  Order Context
[库存已扣减]  [库存已入库]      →  Inventory Context
[支付已发起]  [退款已完成]      →  Payment Context
[优惠券已发放][活动已结束]      →  Marketing Context
[物流已下单]  [包裹已签收]      →  Fulfillment Context
```

每个 Context 一个服务（或一组服务），团队归属清晰，**这是微服务拆分的"原图"**。

> **口诀**：先 Event Storming 再画服务图。直接画服务图 90% 会漏掉边界冲突。

### 2.2 由数据库逆向（按聚合根拆）

存量系统怎么拆？大概率你拿到的是一个**有上百张表的祖传单体**，事件风暴成本太高。退而求其次：**按数据库表的引用关系倒推聚合根**。

#### 步骤

```python
# 用 Python 跑一遍 schema 分析
import sqlalchemy
from sqlalchemy import inspect

engine = sqlalchemy.create_engine("mysql://...")
inspector = inspect(engine)

# 提取所有外键关系
edges = []
for table in inspector.get_table_names():
    for fk in inspector.get_foreign_keys(table):
        edges.append((table, fk['referred_table']))

# 用 networkx 找强连通分量 = 候选聚合
import networkx as nx
G = nx.DiGraph(edges)
components = list(nx.weakly_connected_components(G))
for c in components:
    print(c)
```

输出可能长这样：

```
{'order', 'order_item', 'order_address', 'order_payment'}
{'product', 'product_sku', 'product_image', 'product_attr'}
{'user', 'user_address', 'user_card'}
{'inventory', 'inventory_log'}
```

每个连通分量 = 一个候选聚合 = 一个候选服务。

#### 跨聚合的"软引用"

发现两个聚合之间有外键 —— 比如 `order.user_id REFERENCES user.id` —— 拆分时**这个外键必须降级成"软引用"**：

```sql
-- 拆分前
CREATE TABLE order (
  id BIGINT PRIMARY KEY,
  user_id BIGINT NOT NULL,
  FOREIGN KEY (user_id) REFERENCES user(id)
);

-- 拆分后（user 表搬到独立服务）
CREATE TABLE order (
  id BIGINT PRIMARY KEY,
  user_id BIGINT NOT NULL,            -- 不再有 FK
  user_snapshot JSON                   -- ★ 关键字段冗余一份快照
);
```

`user_snapshot` 冗余的目的：**避免每次显示订单都要远程调用 user 服务**。下单时把当前用户的关键字段（姓名、手机、地址）拷贝进来。

代价：用户改名后历史订单显示的还是旧名字 —— 这通常是**业务上正确的行为**（订单本来就该记录"下单时刻的状态"）。

#### 共享表的处理

数据库逆向最大的坑：**两个 Bounded Context 都频繁读写同一张表**。常见情况：

- `dict_*` 字典表（被所有服务 join）
- `permission` / `role` 表（被认证服务和业务服务共享）
- `config` 表（运维和业务都改）

处理方案：
1. **字典表**：拷贝多份（容量小，变化慢，可以接受最终一致）
2. **权限表**：抽出独立的 IAM 服务，对外只提供 API，不允许 join
3. **配置表**：换成配置中心（Nacos / Apollo）

> **口诀**：**有外键的拆服务，第一刀先把 FK 改成软引用**。FK 还在 = 拆不动。

### 2.3 由用户旅程拆（前端 / BFF）

后端服务可以按 Bounded Context 拆，但**前端调用层不一定**。原因：

一个"商品详情页"可能要聚合：
- Catalog 服务（商品基本信息）
- Inventory 服务（库存）
- Marketing 服务（促销价）
- Review 服务（评价）
- Recommendation 服务（相关推荐）

前端总不能并发调 5 个 API 然后自己拼。这时需要 **BFF（Backend For Frontend）**：

```
Web    →  Web BFF    ┐
                     ├──→  Catalog / Inventory / Marketing / Review / ...
App    →  App BFF    ┘
微信小程序 → MP BFF  ┘
```

BFF 的职责：
- **聚合**：把多个微服务的调用合并成一个前端友好的响应
- **裁剪**：根据端的特点裁字段（移动端不要 desktop 的大图）
- **协议适配**：移动端要 protobuf，Web 要 JSON

BFF 实现示例（Spring Cloud Gateway + 自定义过滤器）：

```java
@RestController
@RequestMapping("/bff/web/product")
public class ProductBffController {

    private final WebClient catalogClient;
    private final WebClient inventoryClient;
    private final WebClient marketingClient;

    @GetMapping("/{id}")
    public Mono<ProductDetailVo> detail(@PathVariable Long id) {
        // 并发调三个服务
        Mono<CatalogProduct> catalog = catalogClient.get()
            .uri("/products/{id}", id).retrieve()
            .bodyToMono(CatalogProduct.class);
        Mono<Integer> stock = inventoryClient.get()
            .uri("/stocks/{id}", id).retrieve()
            .bodyToMono(Integer.class);
        Mono<Promotion> promo = marketingClient.get()
            .uri("/promotions/by-product/{id}", id).retrieve()
            .bodyToMono(Promotion.class);

        // zip = 等三个都返回再聚合
        return Mono.zip(catalog, stock, promo)
            .map(t -> ProductDetailVo.from(t.getT1(), t.getT2(), t.getT3()));
    }
}
```

BFF 必须遵守：
- **不持有自己的数据**（无状态）
- **只做聚合 / 协议转换，不做业务逻辑**
- **每个端独立 BFF**，不要"一个 BFF 服务所有端"

> **口诀**：BFF 是**前端团队的后端**，不是后端团队的 API 网关。逻辑下沉到下游服务，BFF 只做编排。

### 2.4 由团队拆（Conway's Law）

理论已经在 1.4 讲了。这里讲怎么落地。

#### 实操：服务 ownership 矩阵

每个服务必须有且只有一个 owner 团队。建议维护一份矩阵：

```yaml
# services.yml
services:
  - name: order-service
    owner_team: trade
    on_call: pagerduty/trade-oncall
    repo: github.com/foo/order-service
    sla:
      availability: 99.95
      p99_latency_ms: 200

  - name: inventory-service
    owner_team: trade
    on_call: pagerduty/trade-oncall
    repo: github.com/foo/inventory-service

  - name: user-service
    owner_team: account
    on_call: pagerduty/account-oncall

  - name: legacy-export-service
    owner_team: ???        # ★ 警告：无主服务
```

无主服务是定时炸弹（案例 ⑥）。CI 里加检查：

```bash
# pre-commit hook
yq e '.services[] | select(.owner_team == "???") | .name' services.yml | \
  while read svc; do
    echo "ERROR: service '$svc' has no owner team" >&2
    exit 1
  done
```

#### 团队拆分要先于服务拆分

新业务上线时常见错误：
1. ❌ 先按需求"画"出 5 个微服务
2. ❌ 然后再讨论"哪个团队负责哪个服务"
3. ❌ 结果：服务边界和团队沟通模式对不上，半年后变成分布式单体

正确顺序：
1. ✅ 先确认有几个独立的团队（含未来 1 年招聘计划）
2. ✅ 每个团队负责一个或多个 Bounded Context
3. ✅ 每个 Bounded Context 实现成一个或多个服务

> **口诀**：服务数 ≤ 团队数 × 3。超过这个比例，团队就 cover 不住了。

### 2.5 由非功能性需求拆（NFR-driven）

非功能性需求（Non-Functional Requirements）是另一个有效的拆分维度。当某段代码的**非功能要求显著不同**时，它就值得单拆出去。

#### 典型场景

**场景 A：QPS 差异巨大**

```
搜索接口：5000 QPS, P99 < 50ms
订单查询：500 QPS, P99 < 200ms
报表生成：5 QPS, 单次可能跑 30s
```

如果都在一个服务里，扩容时按搜索的需求扩 100 个 Pod，结果报表才用得上 5 个，浪费 95 个。拆成 3 个服务，各自扩缩。

**场景 B：安全等级差异**

```
普通用户操作：HTTP + Token
支付密码校验：mTLS + HSM 硬件加密
风控数据查询：网络隔离 + 审计日志
```

把"支付密码校验"单拆出来部署到独立的、网络隔离的环境，**显著降低合规审计的工作量**（你不用证明整个单体都符合 PCI-DSS，只用证明这一个服务符合）。

**场景 C：发布频率差异**

```
首页 UI 文案：每天发 5 次
核心交易逻辑：每月发 1 次（每次发版要走变更评审）
```

把"首页文案"拆成独立服务（甚至独立的"内容服务" + 配置中心），让前端团队随时改不影响交易。

**场景 D：技术栈差异**

```
机器学习推理：Python + GPU
业务接口：Java + CPU
```

不要试图把 Python 模型"塞进" Java 单体（用 Jython / 通过 RPC 都很别扭）。直接拆服务。

#### 反例

不要因为以下理由拆服务：
- "这个模块代码很乱，拆出去重写" → 治标不治本，重写完还是乱
- "这个模块用了 Drools，引入了复杂依赖" → 把 Drools 换成更简单的方案，不需要拆服务
- "想用一下 Go" → 简历驱动开发

> **口诀**：**先有非功能性需求差异，再有拆分**。没有差异硬拆 = 自找麻烦。

### 2.6 Strangler Fig 的代理 / 路由

Strangler Pattern 的核心组件是**路由层**。它必须支持：

1. 按 path / header / cookie / 用户分桶切流量
2. 动态切换路由规则（不停服改）
3. Shadow / Mirror 流量做双跑校验

#### 用 Spring Cloud Gateway 实现

```java
@Configuration
public class StranglerRouteConfig {

    @Bean
    public RouteLocator routes(RouteLocatorBuilder builder) {
        return builder.routes()
            // 阶段 2：按 path 切
            .route("user-new", r -> r
                .path("/api/users/**")
                .and()
                .header("X-Use-New", "true")        // 灰度开关
                .uri("lb://user-service-new"))

            // 默认还走老单体
            .route("user-legacy", r -> r
                .path("/api/users/**")
                .uri("lb://legacy-monolith"))

            // 双跑：流量复制到 new，但响应仍然以 legacy 为准
            .route("user-shadow", r -> r
                .path("/api/users/**")
                .filters(f -> f.filter(new ShadowMirrorFilter("user-service-new")))
                .uri("lb://legacy-monolith"))

            .build();
    }
}
```

`ShadowMirrorFilter` 关键代码：

```java
public class ShadowMirrorFilter implements GatewayFilter {

    private final WebClient shadow;
    private final DiffLogger diffLogger;

    @Override
    public Mono<Void> filter(ServerWebExchange ex, GatewayFilterChain chain) {
        // 复制请求 body
        ex.getRequest().getBody().cache().subscribe(buf -> {
            // 异步打到新服务，结果只用来对比
            shadow.method(ex.getRequest().getMethod())
                  .uri(ex.getRequest().getURI())
                  .headers(h -> h.addAll(ex.getRequest().getHeaders()))
                  .body(BodyInserters.fromDataBuffers(Mono.just(buf)))
                  .exchangeToMono(resp -> resp.bodyToMono(String.class))
                  .subscribe(shadowResp ->
                      diffLogger.compare(ex, shadowResp));
        });
        return chain.filter(ex);   // 继续走 legacy
    }
}
```

#### 双跑差异处理

`DiffLogger.compare` 不是简单的 `equals` —— 实际系统里**永远会有微小差异**：
- 时间戳字段（`created_at` 有几 ms 差）
- 排序不稳定（list 顺序可能不同）
- 浮点精度（`1.99` vs `1.99000000001`）

需要用"语义比较"：

```java
public class SemanticDiffer {
    public Diff compare(JsonNode legacy, JsonNode shadow) {
        Diff diff = new Diff();
        compareRecursive(legacy, shadow, "", diff);
        return diff;
    }

    private void compareRecursive(JsonNode a, JsonNode b, String path, Diff diff) {
        if (path.endsWith(".created_at") || path.endsWith(".updated_at")) {
            return;          // 忽略时间戳
        }
        if (a.isArray() && b.isArray()) {
            // list 按某个 key 排序后再比
            List<JsonNode> sa = sortByKey(a, "id");
            List<JsonNode> sb = sortByKey(b, "id");
            // ...
        }
        // 数值比较带容忍度
        if (a.isNumber() && b.isNumber()) {
            if (Math.abs(a.doubleValue() - b.doubleValue()) > 0.001) {
                diff.add(path, a, b);
            }
            return;
        }
        // 其他字段精确比
        if (!a.equals(b)) diff.add(path, a, b);
    }
}
```

差异日志接到分析平台，**当差异率 < 0.01% 持续 3 天**，可以认为新服务行为和旧的等价，可以切流量。

#### 切换的灰度策略

```
Day 1:    1%  流量切到新服务
Day 2:    5%
Day 3:   10%
Day 7:   50%
Day 10: 100%
Day 14:  删旧代码
```

每一步都监控：
- 新服务的成功率 / P99 不能比旧的差
- 业务指标（下单成功率、支付成功率）不能变差
- 不一致告警数为 0

> **口诀**：**Strangler 的成败在路由层，不在新服务**。路由层做不好双跑校验和回滚，新服务永远不敢全切。

### 2.7 双写模式（Dual Write）的过渡

数据库拆分时的过渡方案。场景：旧单体的 `order` 表要搬到新的 order-service 的独立库。

#### 阶段一：双写

新代码每次写订单时，**同时写老库和新库**：

```java
@Transactional
public void createOrder(Order order) {
    legacyOrderDao.insert(order);             // 写老库（事务内）
    try {
        newOrderApiClient.create(order);      // 写新服务（事务外）
    } catch (Exception e) {
        // 不能让新服务的失败影响老链路
        log.error("dual write to new service failed", e);
        dualWriteRetryQueue.offer(order);     // 进重试队列
    }
}
```

读还是从老库读，新库**只是被动接收**。

#### 阶段二：双读校验

```java
public Order getOrder(Long id) {
    Order legacy = legacyOrderDao.find(id);
    try {
        Order shadow = newOrderApiClient.find(id);
        diffLogger.compare(legacy, shadow);
    } catch (Exception ignore) {}
    return legacy;
}
```

#### 阶段三：切读

```java
public Order getOrder(Long id) {
    Order order = newOrderApiClient.find(id);    // 主：新
    if (order == null) {
        order = legacyOrderDao.find(id);         // 兜底：老（防止迁移漏数据）
        if (order != null) {
            log.warn("found in legacy but not in new: {}", id);
        }
    }
    return order;
}
```

#### 阶段四：单写新库

```java
public void createOrder(Order order) {
    newOrderApiClient.create(order);
    // 老库不写了
}
```

#### 双写的两个深坑

**坑一：双写的事务性**

如果用上面的代码，存在这种情况：
1. `legacyOrderDao.insert(order)` 成功（事务已提交到 MySQL，但 Java 层还没 return）
2. `newOrderApiClient.create(order)` 失败
3. 进了重试队列，但**进程被 kill**了
4. 老库有，新库没有 —— **不一致**

修复：把"待同步"先写到本地表（事务内），后台扫描器异步推送。这就是**本地消息表**模式：

```sql
CREATE TABLE outbox (
  id BIGINT PRIMARY KEY,
  topic VARCHAR(64),
  payload JSON,
  status TINYINT DEFAULT 0,    -- 0:待发送 1:已发送
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_status (status, created_at)
);
```

```java
@Transactional
public void createOrder(Order order) {
    legacyOrderDao.insert(order);
    outboxDao.insert(new OutboxRecord("order.created", toJson(order)));
    // 两条都在同一个事务里，原子的
}
```

后台 scanner 把 `outbox` 里 status=0 的扫出来，调新服务，成功后改 status=1。

**坑二：写顺序**

如果业务有"先创建，再更新"的场景：

```
T1: 旧库 create(o)   (事务结束)
T2: 旧库 update(o)   (事务结束)
T3: 异步同步线程把 create 推到新服务
T4: 异步同步线程把 update 推到新服务
```

如果 T3 和 T4 之间有重试 / 网络抖动，**可能出现 update 先到、create 后到** —— 新服务收到 update 时找不到记录。

修复：用 outbox 表的自增 ID 严格保序，单消费者消费，或者用 Kafka 按 orderId 哈希到固定 partition。

> **口诀**：双写不是"两份代码各写一遍"。**事务边界、写顺序、失败补偿**三件事不解决，一致性必崩。

---

## 3. 实测数据

### 3.1 单体 vs 微服务在不同团队规模下的开发效率

来自 Accelerate / DORA 报告 + 我自己跟踪过的几个项目的整理。**横轴是团队人数，纵轴是"功能从需求到上线的天数中位数"**：

```
天数
 30 ┤                                    ●  微服务（不熟练）
 25 ┤                              ●
 20 ┤        ●                ●
 15 ┤   ●         ●     ●   ●       ●     ●  微服务（熟练 / 成熟平台）
 10 ┤                ●                  ●
  5 ┤   ●  ●   ●                  ●
  3 ┤   ●   ●  ●  单体
    └──────────────────────────────────────►
      5   10  20   30   50   80  100  150  人数
```

可以读出几个结论：

| 团队规模 | 单体 | Modular Mono | 微服务（成熟） | 微服务（不成熟） |
| --- | --- | --- | --- | --- |
| < 10 人 | 3 天 | 4 天 | 7 天 | 15 天 |
| 10-30 人 | 5 天 | 4 天 | 6 天 | 18 天 |
| 30-50 人 | 10 天 | 6 天 | 6 天 | 20 天 |
| 50-100 人 | 18 天 | 8 天 | 8 天 | 22 天 |
| > 100 人 | 25 天+ | 12 天 | 10 天 | 28 天 |

**关键观察**：
- 30 人以下，微服务**没有效率优势**（平台成本吃掉了）
- 30~50 人是分水岭，三种架构表现接近
- 100 人以上，单体效率明显落后
- "不成熟的微服务"在所有规模下都最差 —— **没有平台能力别上微服务**

### 3.2 微服务的运维成本

对一个 30 服务的中等规模系统，相比单体多花的运维成本（人月）：

| 项目 | 一次性成本 | 持续成本（人月/年） |
| --- | --- | --- |
| K8s 集群搭建 + 维护 | 2 人月 | 6 人月 |
| 服务网格（Istio / Linkerd） | 3 人月 | 4 人月 |
| 注册中心 + 配置中心 | 1 人月 | 2 人月 |
| 监控（Prometheus / Grafana） | 2 人月 | 4 人月 |
| 链路追踪（SkyWalking / Jaeger） | 1 人月 | 2 人月 |
| 日志聚合（ELK / Loki） | 2 人月 | 4 人月 |
| CI/CD 平台 | 3 人月 | 6 人月 |
| 契约测试体系 | 2 人月 | 3 人月 |
| **合计** | **16 人月** | **31 人月** |

**对小团队的意义**：30 人团队里有 31 人月的运维消耗 = **要专门投入 2~3 人做平台**。如果你的团队没这个余量，微服务对你就是负担。

### 3.3 拆分前后的故障率对比

某中型公司（80 人研发）从单体拆到 35 个服务前后 6 个月对比：

| 指标 | 拆前 | 拆后 |
| --- | --- | --- |
| MTTR（平均修复时间） | 2.5 小时 | 1.1 小时 |
| MTBF（平均故障间隔） | 30 天 | 9 天 |
| 故障次数/月 | 1 | 3.3 |
| **影响用户**的故障次数/月 | 0.8 | 0.5 |
| 全站不可用时间 | 4 小时/年 | 1 小时/年 |
| 部分功能不可用 | 2 小时/年 | 12 小时/年 |

**反直觉的发现**：
- 故障次数**变多**（更多服务 = 更多故障点）
- 但**严重故障次数**变少（故障被隔离在单服务内）
- MTTR 显著缩短（单服务问题更容易定位修复）
- 用户实际感受到的不可用时间显著缩短

> **口诀**：微服务**不会减少 bug 数量，但会减小 bug 的爆炸半径**。

### 3.4 跨服务调用的延迟代价

实测：同一 K8s 集群内，gRPC 调用 vs 进程内函数调用：

| 调用方式 | P50 | P99 | 备注 |
| --- | --- | --- | --- |
| 进程内函数调用 | < 1μs | < 1μs | JIT 后接近内联 |
| gRPC 同节点 | 0.3 ms | 1.5 ms | 走 loopback |
| gRPC 跨节点（同 AZ） | 0.8 ms | 3 ms | 走集群网络 |
| gRPC 跨 AZ | 2 ms | 8 ms | 跨可用区 |
| HTTP/JSON 跨节点 | 2 ms | 10 ms | 比 gRPC 慢 2~3 倍 |

**算一笔账**：一个商品详情页要聚合 5 个服务，串行调用：
- 单体：< 5μs
- 微服务（同 AZ gRPC）：5 × 3ms ≈ 15ms（最坏 P99 累计 40ms）
- 微服务（HTTP/JSON 跨 AZ）：5 × 10ms ≈ 50ms（最坏 P99 累计 200ms）

并行调用能降到单次延迟，但你的代码会复杂很多倍。

> **口诀**：拆服务时**算一下"用户一次操作要几跳"**。超过 5 跳，要么并行、要么聚合，要么别拆这么细。

### 3.5 数据库拆分前后的查询代价

单体里 `order JOIN user` 一次查询 100 行：

```sql
SELECT o.*, u.name, u.phone
FROM order o JOIN user u ON o.user_id = u.id
WHERE o.status = 'PAID' LIMIT 100;
```

执行时间：**~3ms**（同库 JOIN，索引覆盖）

拆服务后**没有 JOIN 了**，只能：
1. 查 order：3ms（拿到 100 个 user_id）
2. 调 user-service 批量查：5ms
3. 应用层 merge：< 1ms

总时间：**~9ms**，是单体的 3 倍。

如果**没有用批量接口**而是循环单个调用：
- 100 次调用 × 3ms = **300ms**

这是案例 ⑦ "N+1 跨服务调用"的根因。

#### 数据冗余 vs 远程调用

为了避免每次都调 user-service，订单表里冗余 user 的快照：

```sql
CREATE TABLE order (
  id BIGINT PRIMARY KEY,
  user_id BIGINT,
  user_name VARCHAR(64),         -- 冗余
  user_phone VARCHAR(20),        -- 冗余
  ...
);
```

代价：
- 存储增加：每条订单多 100 字节，10 亿订单 = 100GB
- 一致性：用户改名后历史订单显示旧名字（业务上通常**这正是想要的**）
- 写入：下单时多查一次 user 拿快照（一次性，可接受）

> **口诀**：拆服务后，**接受冗余、接受最终一致**。坚持强一致 = 用 2PC = 性能崩塌（案例 ⑧）。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 微服务过早拆分 —— "8 人小团队，22 个服务，谁也不敢动"

#### 现象

8 人团队，做了 1 年微服务化，拆出 22 个服务。结果：
- 加一个简单业务字段要改 3~4 个服务
- 上线一次要协调 4 个仓库的发版顺序
- 服务器成本（K8s + 中间件）是营收的 30%
- 每天大半时间在调"为什么 A 服务调不通 B"
- 团队 2 个人专门做"运维 / 平台"，业务停滞

#### 根因

来自一个 case：CTO 是大厂出来的，照搬大厂的"微服务最佳实践"。问题在于：

- **大厂的"最佳实践"是给 1000 人团队的**，他们有专职平台团队、SRE、DBA、安全团队
- 8 人团队 = **2 人撑平台 + 6 人写业务**，业务推进速度比单体还慢
- 22 个服务的"独立部署"价值约等于 0（业务变化时几乎都跨多服务）

#### 修复

**反向操作：合并服务**。

```
原架构（22 服务）              目标架构（5 服务 + Modular Monolith）
─────────────────              ──────────────────────────────────
user-service                    │
auth-service                    │
profile-service                 ├── account-service
notification-service            │
                                │
order-service                   │
payment-service                 ├── trade-service
refund-service                  │
                                │
inventory-service               │
warehouse-service               ├── inventory-service
shipment-service                │
                                │
... 12 个其他服务               ├── back-office-monolith (Modular Monolith)
                                │
api-gateway                     └── api-gateway
```

合并步骤：
1. 用 Strangler Pattern 反向操作 —— **把多个微服务合到一个新服务里**
2. 共享同一个 JVM 进程，但保留逻辑边界（Modular Monolith）
3. 一年后业务稳定下来，再决定哪些真需要独立部署

合并后效果：
- 5 个服务 + 1 个 modular monolith
- 加业务字段从平均 3 服务降到 1 服务
- 服务器成本降 60%
- 4 个人就能 cover 业务+运维

#### 验证

- DORA 指标：部署频率从 2 次/周升到 5 次/天
- 单功能交付时间：从 12 天降到 4 天
- 团队满意度（NPS）：从 -20 升到 +30

> **口诀**：服务数 ≤ 团队人数 / 3。8 人团队最多 2~3 个核心服务。

---

### 案例 ② 分布式单体 —— "服务多但没解耦"

#### 现象

50 人团队，35 个服务。任何一次发版都要：
- 改 5+ 个仓库
- 一个 PR 评审涉及 4 个团队
- 必须按 "DB → user → order → API → web" 的固定顺序部署
- 一个服务挂了，整条链路都挂

#### 根因

服务确实拆了，但**边界拆错了**。具体表现：

**症状一：共享数据库**

```sql
-- order-service 直接 JOIN user-service 的表
SELECT o.id, u.name FROM order_db.order o
JOIN user_db.user u ON o.user_id = u.id;
```

虽然是不同 schema，但都在同一个 MySQL 实例，order-service 直接 SQL JOIN 跨 schema 查。**user-service 的表结构变 = order-service 必崩**。

**症状二：共享 JAR**

```xml
<dependency>
    <groupId>com.foo</groupId>
    <artifactId>common-domain</artifactId>
    <version>1.2.3</version>
</dependency>
```

`common-domain` 里塞了所有"公用"的实体类、DTO、甚至业务逻辑。任何一个字段变更都要发新版 + 所有服务升级。

**症状三：固定调用链**

```
client → web → api-gateway → bff → order → user → permission → audit
```

一条请求穿 7 个服务，每个都是同步阻塞。任何一个挂了整条挂。

#### 修复

**第一步：识别真实的耦合**

```bash
# 用脚本扫描所有服务的依赖
for svc in services/*/; do
    echo "=== $svc ==="
    grep -r "JOIN .*_db\\." $svc/sql/
    grep -r "import com.foo.common" $svc/src/
done
```

输出会让你触目惊心：你以为 35 个服务，实际上有几百条隐式依赖。

**第二步：去掉跨库 JOIN**

每个服务**只能访问自己的库**。跨库需求改成 API 调用 + 数据冗余：

```java
// 改前：order 直接 JOIN user
@Query("SELECT o.id, u.name FROM Order o JOIN User u ON o.userId = u.id")
List<OrderVo> findWithUser();

// 改后：order 只查自己，再调 user 服务
public List<OrderVo> findWithUser() {
    List<Order> orders = orderDao.findAll();
    Set<Long> userIds = orders.stream().map(Order::getUserId).collect(toSet());
    Map<Long, User> userMap = userClient.batchGet(userIds);   // 批量调
    return orders.stream()
        .map(o -> OrderVo.of(o, userMap.get(o.getUserId())))
        .toList();
}
```

**第三步：拆 common-domain**

```
common-domain (旧)              拆分后
├── User                        common-base (只放真正全局的)
├── Order                       ├── 时间工具
├── Product                     ├── 通用异常类
├── DateUtils                   └── 基础类型
└── ExceptionUtils
                                user-api (user 服务的契约)
                                ├── UserDto
                                └── UserClient

                                order-api (order 服务的契约)
                                ├── OrderDto
                                └── OrderClient
```

每个服务**只发布自己的 API JAR**，不再有"共享业务实体"。

**第四步：契约测试**

```java
// 在 user-service 里
@PactProvider("user-service")
@Test
void shouldHonorContractWith_OrderService() {
    // 验证 user-service 的实际响应符合 order-service 期望的契约
}
```

[Pact](https://docs.pact.io/) 这类契约测试框架能在每次发版前**自动检测 API 兼容性**，避免"不知道改了哪个字段 break 了下游"。

#### 验证

- 发版独立性：随机选一个服务发版，看会不会影响其他
- 部署顺序：删除"必须按 X 顺序部署"的文档，看真实场景能不能容忍
- 故障演练：随机 kill 一个服务，看其他服务能不能降级运行（Chaos Engineering）

> **口诀**：分布式单体 = 微服务的成本 × 单体的耦合。**判断公式：跨服务 PR 数 / 总 PR 数 > 30% 就是分布式单体**。

---

### 案例 ③ 拆分粒度太细 —— "100+ 服务但功能简单"

#### 现象

某创业公司 30 人团队，照搬"一个 REST 资源 = 一个微服务"的极端理念，做出 120 个服务。每个服务平均代码 500 行（很多就 200 行）。结果：
- 一次完整业务请求要调 15+ 服务
- 链路追踪图密密麻麻像蜘蛛网
- P99 延迟 > 3s
- 数据库连接数被打爆（每个服务 10 个连接 × 120 服务 = 1200 个）

#### 根因

服务粒度的两个错误标准：
- ❌ "一个 REST 资源一个服务"（导致 user / user-profile / user-address / user-phone 4 个服务）
- ❌ "一个文件夹一个服务"（导致 controller / service / dao 各拆一个）

正确标准（Sam Newman）：**一个服务应该对应一个 Bounded Context，且能被一个团队 own**。

具体到代码量参考：
- 太小：< 1k 行（拆开的成本远大于收益）
- 合适：3k ~ 30k 行
- 偏大但可接受：30k ~ 100k 行
- 太大：> 100k 行（考虑拆）

#### 修复

**合并粒度过细的服务**。

```
合并前：
user-basic-service (300 行)        ┐
user-profile-service (400 行)      ├──→ user-service (合并)
user-address-service (350 行)      │
user-phone-service (200 行)        ┘

product-spu-service (500 行)        ┐
product-sku-service (400 行)        ├──→ product-service (合并)
product-image-service (300 行)      │
product-attr-service (450 行)       ┘
```

合并后：
- 服务数从 120 降到 35
- 平均一次请求调 5~6 个服务
- P99 从 3s 降到 400ms
- 数据库连接数降到 350

#### 验证

```bash
# 用 SkyWalking / Jaeger 拉一段时间的 trace
# 统计：一次请求调用的服务数分布
jq '.spans | length' traces.json | sort | uniq -c

# 健康范围：90% 的请求 < 8 个 span
```

> **口诀**：服务粒度看**一次业务请求要调几跳**。> 8 跳 = 拆太细了。

---

### 案例 ④ Strangler 切换失败 —— "路由不一致导致数据双写脏"

#### 现象

灰度切流量第 3 天，监控告警：新老服务的订单数差 3000 单。差异分析后发现：
- 同一个用户的两次请求，一次去了新服务，一次去了老服务
- 新服务创建了订单 ID=100
- 老服务创建了订单 ID=100（自增 ID 撞了！）
- 状态查询：去新服务返回"已支付"，去老服务返回"待支付"

#### 根因

灰度路由策略错了：

```yaml
# ❌ 按请求 random 切流量
- path: /orders
  weight:
    new: 10
    legacy: 90
```

`random` 意味着**同一个用户的两个请求可能去不同服务**。如果新老服务的数据没有完美同步（实际上不可能完美），用户体验就是"我的订单一会有一会没"。

更糟的是 ID 生成：
- 老服务用 MySQL 自增 ID
- 新服务也用自增 ID（从 0 开始）
- 两边都创建了 ID=100 的订单 → 后续合并数据时主键冲突

#### 修复

**修复一：按用户哈希切流量，不用 random**

```yaml
- path: /orders
  hash_by: header.X-User-Id
  weight:
    new: 10
    legacy: 90
```

同一个用户**永远去同一个服务**。这样即使数据没同步，单个用户也看不到"自己的订单忽隐忽现"。

Spring Cloud Gateway 实现：

```java
public class HashedWeightedRoutingFilter implements GatewayFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange ex, GatewayFilterChain chain) {
        String userId = ex.getRequest().getHeaders().getFirst("X-User-Id");
        if (userId == null) {
            return chain.filter(ex);   // 没有 userId 走默认
        }
        int bucket = Math.abs(userId.hashCode() % 100);
        String target = bucket < 10 ? "user-new" : "user-legacy";
        ex.getAttributes().put(GATEWAY_ROUTE_ATTR, routes.get(target));
        return chain.filter(ex);
    }
}
```

**修复二：ID 生成统一**

灰度期间，**新老服务必须用同一个 ID 生成器**。两个方案：
- 都用同一台 Redis 的 `INCR`
- 都用 Snowflake，但配置不同的 worker_id

```java
// 新服务的 ID 生成必须避开老服务的 ID 范围
@Configuration
public class IdConfig {
    @Bean
    public SnowflakeIdGenerator idGen() {
        // 老服务用 worker_id = 1，新服务用 worker_id = 2
        return new SnowflakeIdGenerator(2);
    }
}
```

或者更安全：**新服务的 ID 来自老服务**

```java
// 新服务下单时
public Long createOrder(...) {
    Long id = legacyIdService.nextOrderId();   // 调老服务拿 ID
    // ... 写新库
}
```

**修复三：双跑校验前置**

灰度切流量**之前**，先跑 1 周的 shadow（流量复制到新服务但不返回）。差异 < 0.01% 后再切真流量。

#### 验证

- 用户级路由一致性：抽样 100 个用户，看 24 小时内的请求是不是都路由到同一个服务
- ID 冲突：定期 `SELECT id, COUNT(*) FROM order_merged_view GROUP BY id HAVING COUNT(*) > 1`
- 数据一致性：每小时跑对账脚本，对比新老库的订单数和金额

> **口诀**：Strangler 灰度**必须按用户/会话哈希**，不能 random。否则一个用户跨两个系统 = 必出脏数据。

---

### 案例 ⑤ 拆分时数据库共享 —— "拆了服务没拆库"

#### 现象

物理上：3 个微服务（order / user / product）。
数据库：1 个 MySQL，3 个 schema（order_db / user_db / product_db）。
表面上看起来很干净，但：
- DBA 报警：MySQL 连接数超 5000
- 一次 user 表加索引，需要协调 order 和 product 团队（因为 order 直接 SQL JOIN 了 user 表）
- 一次 product 库的慢查询拖垮了整个 MySQL，所有服务都挂了

#### 根因

**"拆服务但共享 DB"是分布式单体的常见症状**。共享 DB 带来的耦合：

| 维度 | 共享 DB 的影响 |
| --- | --- |
| Schema 变更 | 任何一张表 ALTER 都可能影响其他服务 |
| 性能 | 一个慢查询拖垮所有服务 |
| 故障 | DB 一挂全挂 |
| 扩缩容 | 没法按服务独立扩 DB |
| 安全 | 一个服务被入侵能直接读其他服务数据 |

#### 修复

**第一步：禁止跨 schema JOIN**

代码评审 + 静态扫描卡死：

```bash
# pre-commit hook
for sql in $(find . -name "*.sql" -o -name "*.xml"); do
    if grep -E "(FROM|JOIN)\s+(user_db|product_db)\." "$sql"; then
        echo "ERROR: cross-schema JOIN found in $sql"
        exit 1
    fi
done
```

**第二步：拆物理 DB**

按服务划分独立的 MySQL 实例：

```
原：1 MySQL (3 schema)
新：3 MySQL 实例
    ├── order-mysql (order_db)
    ├── user-mysql  (user_db)
    └── product-mysql (product_db)
```

迁移步骤（用案例 ④ 的双写模式）：
1. 新建 user-mysql，开始双写
2. 验证一周
3. 切读到新库
4. 停老库的 user_db 写入
5. 删老库的 user_db

**第三步：跨服务数据访问改 API**

```java
// ❌ 改前
@Query(value = "SELECT u.* FROM user_db.user u WHERE u.id IN ?1", nativeQuery = true)
List<User> findUsers(Set<Long> ids);

// ✅ 改后
public Map<Long, User> findUsers(Set<Long> ids) {
    return userClient.batchGet(ids);
}
```

**第四步：连接池调优**

每个服务的 DB 连接池不要拍脑袋设：

```yaml
# 错误：默认 50 连接
# 6 服务 × 8 实例 × 50 连接 = 2400 连接，MySQL 顶不住
spring:
  datasource:
    hikari:
      maximum-pool-size: 50

# 正确：按 (服务实例数 × 单实例连接) ≤ MySQL max_connections × 0.7
# 6 服务 × 8 实例 × 10 连接 = 480 连接，留给 MySQL 70% 余量
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
```

#### 验证

- 跨 schema JOIN 数：CI 扫描，必须 = 0
- DB 实例独立：kill 一个 DB，验证其他服务还能正常运行
- 连接数：`SHOW STATUS LIKE 'Threads_connected'` 必须 < `max_connections × 0.5`

> **口诀**：**拆服务 = 拆库**。共享数据库的微服务不是微服务，是**用网络延迟惩罚自己的单体**。

---

### 案例 ⑥ 微服务团队 ownership 缺失 —— "无主服务的腐烂"

#### 现象

一个有 80 个服务的中型公司，半年前接手了一个老团队解散后留下的 12 个服务。半年后：
- 12 个服务版本停在半年前
- 有 3 个服务的依赖（Spring Boot 2.x）已经 EOL
- 有 1 个服务被发现存在 Log4j 漏洞（CVE-2021-44228），3 个月没人修
- 代码仓库的 main 分支甚至有人改过（"反正没人 review"）

#### 根因

服务有 owner 团队是**必要不充分**条件。如果团队解散 / 重组 / 离职，服务变成"流浪汉"。后果：
- 安全漏洞没人修
- 性能问题没人优化
- 业务变更没人对齐
- 技术债越积越多，最后**重写成本 > 维护成本**

#### 修复

**修复一：CI 强制 OWNERS 文件**

每个服务仓库根目录必须有 `OWNERS` 文件：

```yaml
# OWNERS
team: trade
oncall_rotation: pagerduty/trade-oncall
slack: "#team-trade"
docs: https://wiki/trade-team
created: 2024-03-15
last_review: 2025-04-01    # ★ 必须每季度更新
```

CI 检查：

```python
# ci/check_owners.py
import yaml, sys
from datetime import datetime, timedelta

with open("OWNERS") as f:
    owners = yaml.safe_load(f)

if not owners.get("team"):
    print("ERROR: OWNERS file missing 'team'", file=sys.stderr)
    sys.exit(1)

last_review = datetime.fromisoformat(owners["last_review"])
if datetime.now() - last_review > timedelta(days=120):
    print(f"ERROR: OWNERS last_review > 120 days, please refresh", file=sys.stderr)
    sys.exit(1)
```

**修复二：服务清单 + 季度 review**

每季度做一次"服务普查"：

```sql
-- 服务清单 DB
SELECT
    s.name,
    s.owner_team,
    s.last_deploy_time,
    DATEDIFF(NOW(), s.last_deploy_time) AS days_since_deploy,
    s.last_review_time
FROM services s
WHERE s.last_review_time < DATE_SUB(NOW(), INTERVAL 90 DAY)
   OR s.owner_team IS NULL
   OR s.last_deploy_time < DATE_SUB(NOW(), INTERVAL 180 DAY);
```

输出列表里每一项必须**当季内**指定接管团队或下线决议。

**修复三：自动下线规则**

```yaml
# 服务下线 SOP
trigger:
  - last_deploy > 180 天
  - 或 OWNERS.last_review > 180 天
  - 或 owner_team 字段为空

action:
  step1: 给所有可能的接管团队发 Slack 消息（30 天 cooldown）
  step2: 30 天后无人认领 → 流量降到 0%
  step3: 再 14 天无业务投诉 → 服务下线（保留代码 6 个月）
  step4: 再 6 个月无人申请恢复 → 删除代码仓库
```

#### 验证

- 季度报表：`SELECT COUNT(*) FROM services WHERE owner_team IS NULL` 必须 = 0
- 安全扫描：所有服务必须每月跑一次 SCA（Software Composition Analysis）
- 演练：随机选一个服务，问 owner 团队"如果今晚出事谁 on-call"，要能秒答

> **口诀**：服务**没有 owner = 已经死了，只是还没意识到**。无主服务必须强制接管或下线。

---

### 案例 ⑦ 服务发现风暴 —— "注册中心被打挂"

#### 现象

凌晨 2 点告警：Nacos 集群 CPU 100%，无法响应。所有服务突然找不到下游，全站 5xx。

#### 根因

排查发现：
- 一个服务（payment-service）刚发版，启动后 30 秒内**频繁重启**（健康检查失败）
- 每次启动都向 Nacos 注册一次
- 每次注销 / 重新注册都会触发**服务列表全量推送**给所有订阅者
- 订阅者有 200+ 个实例
- 200 实例 × 每秒 N 次推送 = Nacos 处理不过来

更糟糕的是连锁反应：
- Nacos 慢 → 其他服务的心跳超时
- 心跳超时 → 实例被标记为不可用
- 大量"不可用"事件 → 又触发推送
- 雪崩

#### 根因深挖：服务发现的两种模型

| 模型 | 代表 | 特点 |
| --- | --- | --- |
| 推（Push） | Nacos / Eureka 早期 | 注册中心主动推送变更，及时但风暴风险高 |
| 拉（Pull） | Consul / Eureka 后期 | 客户端定期拉，有延迟但抗风暴 |
| 混合 | 大多数生产实践 | 长轮询 + 增量推送 |

Nacos 默认是推+拉混合，但**频繁变更时**推的成分占主导，容易出风暴。

#### 修复

**修复一：服务实例稳定性**

健康检查不要太敏感：

```yaml
# ❌ 太敏感：1 秒一次健康检查，3 次失败就摘
nacos:
  discovery:
    heart-beat-interval: 1000
    heart-beat-timeout: 3000

# ✅ 合理：5 秒一次，15 秒超时
nacos:
  discovery:
    heart-beat-interval: 5000
    heart-beat-timeout: 15000
```

**修复二：推送限流**

Nacos 服务端配置（2.x+）：

```properties
# 单服务变更最大推送 QPS
nacos.naming.push.maxFailedTimes=2
nacos.naming.push.minLoad=10

# 客户端订阅推送批量化
nacos.naming.subscribe.batch.size=100
```

**修复三：客户端缓存**

服务列表必须在客户端有本地缓存，**Nacos 挂了客户端用最后一次拉到的列表降级**：

```java
@Configuration
public class NacosClientConfig {
    @Bean
    public NamingService namingService() {
        Properties p = new Properties();
        p.put(PropertyKeyConst.SERVER_ADDR, "nacos:8848");
        // ★ 关键：本地缓存目录
        p.put(PropertyKeyConst.NAMING_CACHE_DIR, "/data/nacos-cache");
        // ★ 服务列表 fail-fast = false，挂了用缓存
        p.put(PropertyKeyConst.NAMING_LOAD_CACHE_AT_START, "true");
        return NamingFactory.createNamingService(p);
    }
}
```

**修复四：注册中心独立部署 + HA**

```yaml
# Nacos 集群至少 3 节点，跨 AZ 部署
nacos-cluster:
  nodes:
    - nacos-1 (AZ-A)
    - nacos-2 (AZ-B)
    - nacos-3 (AZ-C)
```

不要把 Nacos 和业务服务部署在同一个 K8s 集群（业务把节点资源吃光会拖垮 Nacos）。

#### 验证

- 故障演练：随机重启一个服务，观察 Nacos 推送 QPS（应该 < 100）
- Nacos 挂掉演练：把 Nacos 集群全部 kill，业务服务应能继续工作至少 1 小时
- 监控：Nacos 自身的 CPU / 内存 / 推送 QPS 必须有告警

> **口诀**：服务发现的注册中心是**全站单点**。它必须**HA + 客户端缓存 + 推送限流**三件齐全。

---

### 案例 ⑧ 跨服务事务用 2PC 性能崩塌

#### 现象

下单流程涉及 order / inventory / payment / coupon 4 个服务。架构师选用 Seata 的 XA 模式（2PC）保证强一致。压测：
- 单服务 TPS：5000
- 用 XA 后，整体下单 TPS：300

性能下降 16 倍。

#### 根因

2PC（Two-Phase Commit）的两阶段：
1. Prepare：协调者问所有参与者"能不能提交"
2. Commit：所有都说能 → 真正提交

性能问题：
- **Prepare 阶段必须持有锁等所有人 ACK**：网络延迟 × N 个参与者
- **资源被锁住的时间** = max(所有参与者的处理时间) + 2 × RTT
- **数据库连接被长时间占用**：4 服务 × 200ms = 一次事务占 800ms 的连接
- TCC（Try-Confirm-Cancel）也类似，只是把锁从 DB 层移到了业务层

#### 修复

**核心思路：放弃强一致，接受最终一致**。

**方案一：本地消息表 + 事件驱动**

```sql
-- 在订单库里建一张 outbox
CREATE TABLE outbox (
  id BIGINT PRIMARY KEY,
  topic VARCHAR(64),
  payload JSON,
  status TINYINT DEFAULT 0,
  retries INT DEFAULT 0,
  next_retry_at TIMESTAMP,
  created_at TIMESTAMP
);
```

```java
@Transactional   // ★ 一个本地事务搞定
public void createOrder(CreateOrderCmd cmd) {
    Order order = new Order(cmd);
    orderDao.insert(order);

    // 同事务写一条"订单已创建"事件
    OutboxRecord event = new OutboxRecord(
        "order.created",
        toJson(new OrderCreatedEvent(order))
    );
    outboxDao.insert(event);
}
```

后台 dispatcher 把 outbox 的事件推到 MQ：

```java
@Scheduled(fixedDelay = 100)
public void dispatch() {
    List<OutboxRecord> records = outboxDao.findUnsentLimit(100);
    for (var r : records) {
        try {
            mqProducer.send(r.getTopic(), r.getPayload());
            outboxDao.markSent(r.getId());
        } catch (Exception e) {
            outboxDao.incrementRetry(r.getId());
        }
    }
}
```

下游服务订阅 MQ：

```java
@KafkaListener(topics = "order.created")
public void onOrderCreated(OrderCreatedEvent event) {
    inventoryService.reserve(event.getProductId(), event.getQty());
}
```

性能对比：
- 2PC：300 TPS
- 本地消息表：4500 TPS

**方案二：Saga**

订单流程用 Saga 编排：

```java
public class OrderSaga {
    @SagaStart
    public void place(OrderCmd cmd) {
        // step 1: 扣库存
        inventoryService.reserve(cmd);
    }

    @CompensationAction(forStep = "reserve")
    public void releaseInventory(OrderCmd cmd) {
        inventoryService.release(cmd);
    }

    @NextStep(after = "reserve")
    public void deductCoupon(OrderCmd cmd) {
        couponService.deduct(cmd);
    }

    @CompensationAction(forStep = "deductCoupon")
    public void refundCoupon(OrderCmd cmd) {
        couponService.refund(cmd);
    }

    // ... 后续 step
}
```

任何一步失败 = 反向执行所有 compensation。

性能对比：
- Saga：4000 TPS（接近本地消息表）
- 缺点：每个 step 都要写 compensation，业务复杂度上升

**判断什么时候用什么**

| 场景 | 推荐方案 |
| --- | --- |
| 跨 1~2 个服务，能容忍最终一致 | 本地消息表 |
| 跨 3+ 服务，业务流程复杂 | Saga |
| 钱 / 库存 / 极端一致性 | 业务上重新设计（避免分布式事务） |
| 跨 DB 实例的强一致 | XA / 2PC（接受性能代价） |

#### 验证

- TPS / 延迟对比表（同负载下）
- 一致性验证：模拟服务挂掉，看最终数据是否一致
- 补偿场景：手动制造一步失败，看 compensation 能不能正确执行

> **口诀**：**微服务 + 强一致 = 性能崩塌**。99% 的"分布式事务"需求，业务上其实可以接受最终一致。

---

### 案例 ⑨ 服务"复活" —— "废弃的服务还在运行"

#### 现象

一次安全审计发现：内网有一个 `legacy-export-service`，过去 90 天 DAU = 0。但它仍然：
- 占用 4 个 K8s pod（消耗 CPU + 内存）
- 持有 MySQL 连接（10 个）
- 每天向 Kafka 写几 KB 数据（自己产的心跳）
- **关键：它能查询到所有用户的隐私数据**

更可怕的是：当负责人是已离职的老员工，**没人知道这个服务能不能下线**。

#### 根因

服务下线的复杂性：
- 没有"零调用 = 可下线"的简单判定 —— 可能有定时任务、手动调用、内部脚本依赖
- 上游可能有 hardcode 的服务地址
- DB 里可能有该服务专属的数据，下线后就丢了

#### 修复

**第一步：流量观察期**

```bash
# 用 sidecar / istio 收集服务的所有入站请求
# 至少观察 90 天（覆盖月度 / 季度任务）

kubectl exec -it legacy-export-service-pod -- \
    cat /var/log/access.log | \
    awk '{print $1, $7}' | sort | uniq -c | sort -rn
```

如果 90 天**真的零调用**，进入下一步。

**第二步：返回 410 Gone**

不直接下线，先让所有调用收到 `410 Gone`：

```java
@RestController
public class GoneController {
    @RequestMapping("/**")
    public ResponseEntity<String> gone() {
        log.warn("CALLED: legacy-export-service from {}",
                 request.getRemoteAddr());
        return ResponseEntity.status(HttpStatus.GONE)
            .body("This service is decommissioned. Contact #ops if needed.");
    }
}
```

观察 30 天。如果有人投诉，就知道隐藏的依赖是谁。

**第三步：缩到 0 实例**

```bash
kubectl scale deployment/legacy-export-service --replicas=0
```

但配置 / Secret / 镜像仍保留 30 天。如果出问题立即恢复。

**第四步：归档数据**

```bash
# 把 DB 里的数据导出
mysqldump --all-databases > legacy-export-backup-$(date +%F).sql
aws s3 cp legacy-export-backup-*.sql s3://archive-bucket/
```

**第五步：彻底删除**

```bash
# 30 天后无人申请恢复
kubectl delete deployment/legacy-export-service
kubectl delete configmap/legacy-export-config
kubectl delete secret/legacy-export-secret

# DB 删 schema
mysql -e "DROP DATABASE legacy_export"

# 代码仓库归档（不删，方便回查）
gh repo archive foo/legacy-export-service
```

**预防：服务的"生命周期标签"**

```yaml
# 每个服务的 K8s deployment 必须有
metadata:
  labels:
    lifecycle: "active"        # active / deprecated / sunset
    sunset-date: ""            # 如 sunset 必须填日期
    last-business-review: "2025-Q2"
```

每季度跑脚本：

```python
for svc in all_services:
    if svc.last_request_count_30d == 0:
        if svc.lifecycle == "active":
            alert(f"{svc.name} has zero traffic for 30d, mark deprecated?")
        elif svc.lifecycle == "deprecated" and now - svc.deprecated_at > 90d:
            alert(f"{svc.name} has been deprecated for 90d, sunset?")
```

#### 验证

- 服务清单：每季度审计，无"无主 + 零流量"的服务
- 下线流程：每年至少演练 1 次完整的服务下线
- 安全："deprecated" 标签的服务自动加上访问限制（仅运维 IP）

> **口诀**：服务**只生不死 = 系统熵增**。必须有强制的"零流量观察 + 410 Gone + 归档 + 删除"流程。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 决策层（拆之前问自己）
- [ ] 团队规模 ≥ 30 人，否则优先 Modular Monolith
- [ ] 拆分理由是"团队 / 性能 / 故障隔离"三选一，不是"听说先进"
- [ ] 至少有 1 个专职平台 / SRE 工程师
- [ ] 业务边界已通过 Event Storming 等方法验证过
- [ ] 拆完每个服务都有明确的 owner 团队
- [ ] 服务数 ≤ 团队数 × 3

### 边界层（拆的时候）
- [ ] 每个服务对应一个 Bounded Context
- [ ] 每个服务有自己独立的数据库（或至少独立 schema）
- [ ] 跨服务**禁止 SQL JOIN 跨库**
- [ ] 跨服务调用走 API，不走共享 JAR 的"业务实体类"
- [ ] 数据冗余优先于强一致（接受最终一致）

### 通信层
- [ ] 同步调用走 gRPC / HTTP，链路不超过 5 跳
- [ ] 异步走 MQ，下游通过订阅消费
- [ ] 服务客户端有超时 / 重试 / 熔断（Resilience4j / Sentinel）
- [ ] 跨服务事务用 Saga 或本地消息表，**不用 2PC / XA**
- [ ] 契约测试（Pact 等）覆盖所有跨服务接口

### 部署层
- [ ] CI/CD 每个服务独立，**不需要协调发版顺序**
- [ ] 灰度按用户 / session 哈希，**不 random 切流量**
- [ ] Strangler 切流量前必须双跑 1 周，差异 < 0.01%
- [ ] K8s 优雅停机：preStop sleep + terminationGracePeriodSeconds 配齐

### 运维层
- [ ] 服务发现（Nacos / Eureka）有 HA + 客户端缓存
- [ ] 链路追踪（SkyWalking / Jaeger）覆盖所有服务
- [ ] 监控（Prometheus）+ 告警（核心指标：QPS / 错误率 / P99 / 队列堆积）
- [ ] 日志聚合（ELK / Loki），traceId 贯穿
- [ ] 每个服务有独立的 oncall 轮值
- [ ] OWNERS 文件 CI 强制 + 季度 review

### 治理层
- [ ] 每季度服务清单审计
- [ ] 零流量服务 90 天后自动 deprecated
- [ ] 安全漏洞（CVE）扫描每月跑一次
- [ ] API 兼容性变更走 deprecation 流程（至少 N+2 个版本）

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### Modular Monolith 实战
- Google [Service Weaver](https://serviceweaver.dev/)：用注解描述模块，框架决定部署形态
- Shopify 的 Majestic Monolith：Rails 单体撑到 1500 万行代码 + 千人团队
- 关键技术：Internal API + Bazel-style 构建依赖检查 + 独立 schema

### 经典书籍
- Sam Newman, *Building Microservices*（第 2 版 2021）—— 微服务领域绕不开的圣经
- Chris Richardson, *Microservices Patterns* —— Saga / API Gateway / 数据管理模式的字典
- Martin Fowler, *Patterns of Enterprise Application Architecture* —— 经典但仍然适用

### Cell-based 深入
- 蚂蚁 [SOFAStack 单元化](https://www.sofastack.tech/) 的 LDC 设计
- AWS Well-Architected Framework 的 Cell-based 章节
- Slack 的 [Cellular Architecture](https://slack.engineering/) 系列博客

### 微服务测试策略
- 测试金字塔在微服务下的变化（更多契约测试 / 更少 E2E）
- Pact / Spring Cloud Contract 实战
- Production Testing：金丝雀、Shadow、Chaos Engineering

### 微服务安全
- mTLS 在服务网格里的落地（Istio / Linkerd）
- 零信任架构（Zero Trust）—— 不信任任何"内网"
- SPIFFE / SPIRE 标准的服务身份
- API 网关层的 OAuth 2.0 / JWT 验证

### Saga / TCC / 本地消息表的工程化
- Seata 框架的 AT / TCC / Saga / XA 四种模式对比
- DTM 分布式事务管理器
- Outbox Pattern（本地消息表）的事件溯源（Event Sourcing）扩展

### 服务网格 vs 框架
- Istio / Linkerd / Consul Connect 的 sidecar 模式
- Dapr 的多运行时（Multi-Runtime）思想
- 是否需要 Service Mesh：团队规模、跨语言需求、运维能力的权衡

### 微服务的反思
- Amazon Prime Video 团队 2023 年[把视频监控服务从微服务改回单体](https://www.primevideotech.com/video-streaming/scaling-up-the-prime-video-audio-video-monitoring-service-and-reducing-costs-by-90)，成本降 90%
- 这不是"微服务错了"，而是"**没有银弹**"的再次证明
- DHH（Rails 作者）的 Majestic Monolith 哲学

---

## 7. 引用

- Martin Fowler, *StranglerFigApplication*（2004）—— [martinfowler.com/bliki/StranglerFigApplication.html](https://martinfowler.com/bliki/StranglerFigApplication.html)
- Sam Newman, *Building Microservices: Designing Fine-Grained Systems*（2nd ed., O'Reilly 2021）
- Chris Richardson, *Microservices Patterns*（Manning 2018）—— [microservices.io](https://microservices.io)
- Eric Evans, *Domain-Driven Design*（Addison-Wesley 2003）
- Melvin Conway, *How Do Committees Invent?*（1968）—— Conway's Law 原始论文
- Alberto Brandolini, *Introducing EventStorming*（Leanpub）
- Google [Service Weaver](https://serviceweaver.dev/) 论文与代码
- Amazon Prime Video, *Scaling up the Prime Video audio/video monitoring service and reducing costs by 90%*（2023）
- AWS Well-Architected Framework, *Reliability Pillar* —— Cell-based architectures
- DORA, *Accelerate State of DevOps Report*（年度）
- Pact 契约测试：[docs.pact.io](https://docs.pact.io)
- 蚂蚁金服 SOFAStack：[sofastack.tech](https://www.sofastack.tech/)

---

> 配套：[DDD 战术战略](/article/architecture/ddd) | [分布式事务](/article/distributed/dist-tx) | [Spring Cloud / Dubbo 对比](/article/distributed/spring-cloud-dubbo) | [高可用架构](/article/system-design/high-availability)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
