# 多租户架构设计 —— 数据、计算、网络的隔离与共享

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只是想了解多租户是个什么概念，看 [架构总览](/article/architecture/) 那一节够了。
> 这里要回答的，是**真正在 SaaS 平台一线踩过坑、被跨租户数据泄漏告警叫起来**的那部分。

---

## 0. 读这篇之前 —— 8 个自检题

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. 共享数据库（Shared DB）/ 共享 Schema（Shared Schema）/ 独立数据库（DB-per-tenant），三种隔离模式在**100 个租户、1000 个租户、10000 个租户**这三个量级下，分别该选哪种？给出量化的取舍依据。
2. 计算隔离从重到轻有 5 个层级：VM / 容器 / 进程 / 线程 / Quota（配额）。同一个 SaaS 应用，**Free 租户、Pro 租户、Enterprise 租户**分别该用哪一档？为什么不能"一刀切"？
3. 租户路由有 4 种主流方式：子域名 / Path 前缀 / Header / JWT。它们在**反向代理层、应用层、消息队列层、定时任务层**的传播能力各是什么？哪种方式最容易在异步场景丢失上下文？
4. "Noisy Neighbor"（吵闹的邻居）问题：一个大租户的批量任务跑爆了 CPU，影响小租户响应。**在数据库层、应用层、网络层**分别有哪些缓解手段？给出 K8s ResourceQuota + LimitRange 的最小配置。
5. 租户级限流和接口级限流有什么本质区别？为什么 Sentinel 默认的 `LimitApp` 不够用，还要加 `tenant` 维度？给一个能在生产用的实现。
6. 一个租户从 Free 升级到 Enterprise，要从共享 Schema 拆到独立 DB —— 数据迁移的**双写、灰度、回切**怎么做？停机窗口能压到多少分钟？
7. PostgreSQL 的 Row-Level Security (RLS) 是怎么实现的？为什么 RLS 配置不当反而会导致**管理员看不到自己创建的数据**？
8. SaaS、PaaS、IaaS 三种云服务模型在多租户上的隔离强度依次是什么？为什么 Snowflake / Databricks 这类"数据 PaaS" 在多租户设计上比传统 SaaS 更复杂？

如果有半数答不上，往下看。

---

## 1. 底层模型：3 个维度、5 个层级、4 种路由

绝大多数人对多租户的认知止于"加个 tenant_id 字段"。这层抽象解释不了上面 8 个问题。多租户不是一个数据库设计问题，而是一个**横跨数据、计算、网络三个维度的全栈隔离问题**。

### 1.1 多租户的 3 个维度

任何 SaaS 系统都必须在这三个维度上同时做决策：

```
                  多租户隔离矩阵
   ┌──────────────────────────────────────────┐
   │           数据隔离        计算隔离        网络隔离  │
   │           (Storage)      (Compute)      (Network) │
   ├──────────────────────────────────────────┤
   │  共享     共享 DB         共享进程        共享 LB    │
   │   ↑       共享 Schema     共享线程池      共享端口    │
   │   │       共享 Cache                                │
   │   │                                                │
   │   │       共享 DB         共享进程        共享 LB    │
   │   │       独立 Schema     独立线程池      Path 路由  │
   │   │                                                │
   │   │       独立 DB         独立容器        独立子域名 │
   │   │       独立 Schema     独立 Pod        独立证书   │
   │   ↓                                                │
   │  独立     独立 DB 实例    独立 K8s NS     独立 VPC   │
   │           独立加密密钥    独立 Node       独立 IP    │
   └──────────────────────────────────────────┘
```

**关键认知**：三个维度可以独立组合。常见组合：

| 组合 | 典型代表 | 适用 |
| --- | --- | --- |
| 共享 DB + 共享进程 + 共享 LB | 钉钉、飞书、Notion 普通租户 | 大量小租户 |
| 共享 DB + 独立线程池 + 共享 LB | 某些 ERP 的 Pro 版 | 中型租户隔离 QoS |
| 独立 DB + 共享进程 + 共享 LB | Salesforce Pod 模型 | 大型客户数据合规 |
| 独立 DB + 独立 Pod + 独立子域名 | Shopify Plus、企业版 SaaS | Enterprise 客户 |

**反直觉的事实**：很多人以为"独立越好"，但**100 个 Enterprise 客户就要 100 套 K8s Namespace + 100 个 DB 实例**，运维成本指数级上升。多租户架构的本质是**在单位成本和隔离强度之间找平衡**，不是追求绝对隔离。

> **口诀**：数据、计算、网络三个维度独立决策；按租户分级动态选择；不要 all-in 一种模式。

### 1.2 数据隔离的 3 种模式

这是多租户最经典的取舍，也是面试最爱问的：

```
模式 A：共享 DB + 共享 Schema （所有租户挤一起）
┌─────────────────────────────────┐
│         users 表                  │
│  id  tenant_id  name  email      │
│  1   t-001      alice  ...        │
│  2   t-001      bob    ...        │
│  3   t-002      carol  ...        │
│  4   t-003      dave   ...        │
└─────────────────────────────────┘

模式 B：共享 DB + 独立 Schema （一个租户一个 schema）
┌─────────┐  ┌─────────┐  ┌─────────┐
│tenant_001│ │tenant_002│ │tenant_003│
│ ├ users  │ │ ├ users  │ │ ├ users  │
│ ├ orders │ │ ├ orders │ │ ├ orders │
└─────────┘  └─────────┘  └─────────┘

模式 C：独立 DB （一个租户一个数据库实例）
┌─────────┐  ┌─────────┐  ┌─────────┐
│  RDS-01  │ │  RDS-02  │ │  RDS-03  │
│tenant001 │ │tenant002 │ │tenant003 │
└─────────┘  └─────────┘  └─────────┘
```

三种模式的全面对比（**这张表是本文最重要的表之一**）：

| 维度 | 模式 A：共享 Schema | 模式 B：独立 Schema | 模式 C：独立 DB |
| --- | --- | --- | --- |
| **每租户成本** | 极低（$0.01/月） | 中（$1/月） | 高（$50~$500/月） |
| **租户上限** | 数十万 | 数千（受 PostgreSQL schema 数限制） | 数百（受 RDS 实例配额） |
| **跨租户查询** | 容易（一条 SQL） | 难（要 UNION） | 极难（跨实例） |
| **租户级备份** | 难（要 WHERE tenant_id 导出） | 中（pg_dump --schema） | 简单（实例级 snapshot） |
| **数据合规（GDPR / 等保）** | 弱（同表共存） | 中 | 强（物理隔离） |
| **大租户 Noisy Neighbor** | 严重 | 中（共享连接池） | 无 |
| **运维复杂度** | 极低 | 中 | 高（N 套备份、监控、版本升级） |
| **Schema 变更（DDL）** | 一次到位 | 要循环每个 schema 跑 | 要循环每个实例跑 |
| **典型代表** | Notion / Slack 普通租户 | Salesforce Sandbox | Shopify Plus / Heroku |

**选择决策树**：

```
租户数 < 100 且每租户数据量 > 100GB？
   ├── 是 → 独立 DB（成本可接受，隔离最强）
   └── 否 ↓

需要满足 SOC2 / HIPAA 等强合规？
   ├── 是 → 至少独立 Schema，最好独立 DB
   └── 否 ↓

最大租户 / 最小租户数据量比 > 1000？
   ├── 是 → 大租户独立 DB，小租户共享 Schema（混合）
   └── 否 → 共享 Schema + tenant_id（最简单也最常用）
```

> **口诀**：默认共享 Schema；大客户独立 DB；不要为了"看起来高级"上独立 Schema —— 它兼具"运维痛苦"和"成本不低"两大缺点。

### 1.3 计算隔离的 5 个层级

数据是"放在哪"，计算是"用谁的 CPU/内存跑"。从重到轻：

```
重 ──────────────────────────────────────────► 轻
│                                              │
VM         容器       进程       线程        Quota
(KVM)    (Docker)   (JVM)    (ThreadPool)  (Sentinel)
│           │          │          │            │
完全隔离   命名空间    JVM 进程   线程组      软限流
内核级     cgroup     堆共享     竞争 CPU     无强制
```

| 层级 | 隔离强度 | 启动成本 | 典型场景 |
| --- | --- | --- | --- |
| **VM** | 内核级 | 分钟 | 金融、医疗等强合规 |
| **容器** | cgroup + namespace | 秒级 | K8s + 大客户独占 Pod |
| **进程** | 操作系统 | 100ms | 一个租户一个 JVM（内存重） |
| **线程** | JVM 内 | 微秒 | 同进程内多租户共享，按线程组路由 |
| **Quota** | 软限制 | 0 | 同一线程池中，按租户 ID 限流 |

**关键认知**：**计算隔离的层级要随租户分级而变**：

- Free 租户：共享线程池 + Quota 限流（千万级租户的成本必须如此）
- Pro 租户：独立线程池（不会被 Free 拖慢）
- Enterprise 租户：独立 K8s Namespace 或独立 Node（QoS 保证）

为什么不一刀切？**100 万 Free 租户每个独占容器 = 100 万个 Pod = K8s 调度直接崩盘**。K8s etcd 在 Pod 数 > 5000 时性能急剧下降，单集群 Pod 上限实际只有 1.5 万。

> **口诀**：低层级隔离适合大量小租户；高层级隔离适合少量大租户。绝不要把 Free 用户上独立 Pod。

### 1.4 租户路由的 4 种方式

请求进来怎么知道属于哪个租户？

```
方式 ①：子域名 (Subdomain)
  alice.app.com → tenant=alice
  bob.app.com   → tenant=bob

方式 ②：Path 前缀
  app.com/t/alice/orders → tenant=alice

方式 ③：HTTP Header
  GET /orders
  X-Tenant-Id: alice

方式 ④：JWT Claim
  Authorization: Bearer <jwt>
  → 解码后 jwt.payload.tenant = alice
```

四种方式的对比（**选错一种就要全栈重构**）：

| 维度 | 子域名 | Path | Header | JWT |
| --- | --- | --- | --- | --- |
| **可见性** | 用户可见，可分享 | 用户可见 | 不可见 | 不可见 |
| **CDN 缓存友好度** | 高（不同域名独立缓存） | 中 | 低 | 低 |
| **HTTPS 证书** | 需要泛域名证书 | 一张证书够用 | 一张 | 一张 |
| **跨域 Cookie** | 难（默认不共享） | 容易 | 容易 | N/A |
| **CORS 复杂度** | 高 | 低 | 低 | 低 |
| **适合 mobile API** | 不适合 | 适合 | 适合 | 最适合 |
| **租户切换成本** | 重定向（改域名） | 改 URL | 改 Header | 重新登录拿 JWT |

**异步场景的传播能力**（这是最容易踩坑的）：

| 场景 | 子域名 | Path | Header | JWT |
| --- | --- | --- | --- | --- |
| 同步调用（HTTP/gRPC） | ✓ | ✓ | ✓（要透传） | ✓（要透传） |
| 消息队列（Kafka/MQ） | ✗ | ✗ | ✗（消费者拿不到） | 必须放 message header |
| 定时任务 | ✗ | ✗ | ✗（无 HTTP 上下文） | 必须存 trigger context |
| 数据库 trigger | ✗ | ✗ | ✗ | ✗（必须用 SET LOCAL） |

最实用的方案是**组合式**：用户面用子域名（看起来专业），内部一律走 JWT + ThreadLocal/MDC。详见 [线程池上下文透传案例](/article/java/threadpool#案例-mdc--traceid-失踪--链路追踪在线程池断了)。

> **口诀**：用户面 = 子域名（品牌 + CDN）；接口面 = JWT（统一可机读）；内部传递 = MDC + TTL（避免异步丢失）。

### 1.5 "Noisy Neighbor" 与租户级 QoS

"吵闹的邻居"是多租户的头号杀手。一个大租户瞬时跑批，把数据库连接池占满、CPU 打到 100%、网络带宽吃干，**所有共享资源的小租户全部受影响**。

可能发生 Noisy Neighbor 的层次：

```
应用层
  ├── 线程池：大租户大量并发请求 → 小租户排队
  ├── 数据库连接池：大租户长查询 → 小租户拿不到连接
  └── 缓存：大租户大 key 把别的 key 挤出 LRU

数据库层
  ├── 大租户全表扫描 → buffer pool 命中率骤降
  ├── 大租户长事务 → undo log 膨胀，所有事务变慢
  └── 大租户大量 INSERT → 锁竞争，行锁等待

网络层
  ├── 大租户大文件上传 → 带宽打满
  └── 大租户 DDoS 行为 → LB 处理不过来
```

**缓解手段（按层次）**：

| 层次 | 手段 | 工具 |
| --- | --- | --- |
| 应用线程 | 租户级线程池 / 信号量 | 自研 Bulkhead |
| 接口限流 | 租户级 QPS / 并发限流 | Sentinel + Tenant Slot |
| DB 连接 | 租户级连接池 | HikariCP 多池 / ProxySQL |
| DB CPU | 资源组 (Resource Group) | MySQL 8.0 / TiDB |
| 容器资源 | ResourceQuota + LimitRange | K8s |
| 网络带宽 | 流量整形 | Istio EnvoyFilter |

**Resource Group（MySQL 8.0+）实战**：

```sql
-- 创建租户专用资源组
CREATE RESOURCE GROUP rg_tenant_alice
  TYPE = USER
  VCPU = 0-3       -- 限定到 4 个核
  THREAD_PRIORITY = 5;

-- 把会话绑定到资源组
SET RESOURCE GROUP rg_tenant_alice;
```

但实测下来 MySQL 的 Resource Group 控制粒度粗（只能 VCPU 绑核，不能限 IOPS），生产更常见的是**应用层做 Bulkhead + 数据库前面套 ProxySQL**。

> **口诀**：Noisy Neighbor 必须在每一层都做隔离。任何一层"漏"了，整个隔离就破功。

### 1.6 租户分级（Free / Pro / Enterprise）

任何成熟的 SaaS 都不会一视同仁对待所有租户。分级是多租户工程化的灵魂：

```
                 Free                Pro             Enterprise
                  │                   │                  │
数据             共享 Schema         共享 Schema       独立 DB
                                   (大租户独立 DB)    (独立加密密钥)

计算             共享线程池          独立线程池        独立 K8s NS
                 共享 Pod            共享 Pod         独立 Node Pool

网络             共享子域名前缀      独立子域名        独立子域名 + 私有连接
                 (alice.app.com)    (alice.app.com)  (alice.corp.com)

限流             100 req/min        1000 req/min     无限制（合同保证 SLA）

监控告警         不告警              邮件             电话 + 专属客户经理

备份             每日全量            每日全量+小时增量  实时复制 + 跨区灾备

合规             基本                ISO 27001         SOC2 + HIPAA + 等保三级

成本/月          $0~$10              $50~$500          $5K+
```

**关键决策**：分级不是"功能差异"，而是**资源 + SLA 差异**。同一份代码，按租户级别走不同的"配置"。

```java
public class TenantTierResolver {
    public TenantTier resolve(String tenantId) {
        TenantConfig cfg = tenantConfigService.get(tenantId);
        return cfg.getTier();   // FREE / PRO / ENTERPRISE
    }
}

public class TenantAwareThreadPool {
    private final Map<TenantTier, ThreadPoolExecutor> pools = Map.of(
        TenantTier.FREE,       newPool(4, 8, 100),     // 共享小池
        TenantTier.PRO,        newPool(8, 16, 200),    // 中等池
        TenantTier.ENTERPRISE, newPool(16, 32, 500)    // 大池
    );

    public void execute(String tenantId, Runnable task) {
        TenantTier tier = tenantTierResolver.resolve(tenantId);
        pools.get(tier).execute(task);
    }
}
```

> **口诀**：分级是多租户的灵魂。同一份代码、不同的配置、按 SLA 收费。

### 1.7 SaaS / PaaS / IaaS 的多租户差别

| 模型 | 共享什么 | 隔离什么 | 多租户难度 |
| --- | --- | --- | --- |
| **SaaS**（Notion / Slack） | 应用 + 数据库 | 数据（按 tenant_id） | 中（应用层做） |
| **PaaS**（Heroku / Vercel） | 平台 + 运行时 | 应用进程 + DB | 高（要管不同语言/框架） |
| **数据 PaaS**（Snowflake / Databricks） | 计算引擎 | 数据 + 计算资源 | 极高（计算和存储都要按需隔离） |
| **IaaS**（AWS EC2 / 阿里云 ECS） | 物理机 | VM | 低（VM 隔离已成熟） |

为什么 Snowflake 这种"数据 PaaS"最复杂？因为它的客户既要**数据完全隔离**（不同公司的数据不能串），又要**共享计算资源池以降本**（按需开计算 warehouse），还要**支持任意 SQL 查询**（不能像 SaaS 那样限制操作）。

Snowflake 的解法：**Storage / Compute / Service 三层分离**。Storage 是 S3 上的多租户对象（按 micro-partition），Compute 是按需启动的 EC2 集群（虚拟仓库），Service 是元数据层。每个租户的查询启动时**临时分配**计算节点，结束就释放。

普通 SaaS 借鉴：**冷数据共享存储、热计算按租户隔离**。

> **口诀**：SaaS 简单粗暴在应用层做租户路由；PaaS 必须支持任意应用的隔离；数据 PaaS 最复杂，要做"按需弹性的计算 + 持久化的多租户存储"。

---

## 2. 工程切片：从 SQL 到 K8s

> 本节代码均可独立运行。基于 Spring Boot 3.2 / PostgreSQL 16 / K8s 1.29。

### 2.1 共享 Schema + tenant_id 的实现（含索引设计）

最常用、最容易做错的一种模式。**死亡之坑**：忘了加 tenant_id 过滤条件。

#### 表结构与索引

```sql
-- 错误示范
CREATE TABLE orders (
    id        BIGSERIAL PRIMARY KEY,
    tenant_id VARCHAR(32) NOT NULL,
    user_id   BIGINT NOT NULL,
    amount    DECIMAL(10,2),
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_user ON orders(user_id);   -- ❌ 没带 tenant_id

-- 正确示范
CREATE TABLE orders (
    id        BIGSERIAL,
    tenant_id VARCHAR(32) NOT NULL,
    user_id   BIGINT NOT NULL,
    amount    DECIMAL(10,2),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (tenant_id, id)               -- ★ 复合主键，tenant 在前
);
CREATE INDEX idx_tenant_user ON orders(tenant_id, user_id);
CREATE INDEX idx_tenant_created ON orders(tenant_id, created_at DESC);
```

**逐段点透**：

- **PRIMARY KEY (tenant_id, id)**：把 tenant_id 放主键最前面，所有按租户的查询都能走聚簇索引扫描。这是 InnoDB / PostgreSQL B-Tree 的物理布局优势 —— 同租户数据物理相邻，buffer pool 命中率高。
- **所有索引必须 tenant_id 打头**：否则跨租户的全表扫描会让大租户拖死小租户。
- **不要用 BIGSERIAL 单独做主键**：跨租户 ID 全局唯一虽然方便联调，但你失去了 tenant_id 在前的物理聚簇优势。

#### 应用层兜底：MyBatis Interceptor 强制注入

光靠 Code Review 防不住忘加 WHERE。最稳的做法是**在 SQL 解析层强制注入 tenant_id 条件**：

```java
@Intercepts({
    @Signature(type = StatementHandler.class, method = "prepare",
               args = {Connection.class, Integer.class})
})
@Component
public class TenantSqlInterceptor implements Interceptor {

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        StatementHandler handler = (StatementHandler) invocation.getTarget();
        BoundSql boundSql = handler.getBoundSql();
        String originalSql = boundSql.getSql();

        String tenantId = TenantContext.getTenantId();
        if (tenantId == null) {
            // ★ 必须显式拒绝，不能默认放行
            throw new IllegalStateException("no tenant in context, refuse to execute SQL");
        }

        String rewritten = rewriteSql(originalSql, tenantId);
        Field sqlField = BoundSql.class.getDeclaredField("sql");
        sqlField.setAccessible(true);
        sqlField.set(boundSql, rewritten);

        return invocation.proceed();
    }

    private String rewriteSql(String sql, String tenantId) {
        // 用 JSqlParser 解析后注入 WHERE tenant_id = ?
        Statement stmt = CCJSqlParserUtil.parse(sql);
        if (stmt instanceof Select select) {
            new TenantWhereVisitor(tenantId).visit((PlainSelect) select.getSelectBody());
        }
        return stmt.toString();
    }
}
```

**逐段点透**：

- **Interceptor 在 prepare 阶段拦截**：拿到 BoundSql 后用反射改 sql 字段。这是 MyBatis 唯一干净改写 SQL 的时机。
- **TenantContext 拿不到必须抛异常**：千万不能"默认拿一个"。生产 99% 的跨租户数据泄漏都是因为某个工具方法没设 tenant 就走了默认。
- **JSqlParser 改写有性能开销**：每条 SQL 解析约 50us。生产可以加缓存（按 SQL 模板缓存改写结果）。

### 2.2 PostgreSQL Row-Level Security

Postgres 9.5+ 提供的内核级行级安全策略。优势：**SQL 层兜底，应用即使忘加过滤也能拦住**。

#### 启用 RLS

```sql
-- 1. 表上启用 RLS
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders FORCE ROW LEVEL SECURITY;   -- ★ 表所有者也受约束

-- 2. 创建策略：当前会话 GUC 变量必须等于行的 tenant_id
CREATE POLICY tenant_isolation ON orders
    FOR ALL
    USING (tenant_id = current_setting('app.current_tenant', true))
    WITH CHECK (tenant_id = current_setting('app.current_tenant', true));
```

**逐段点透**：

- **ENABLE 和 FORCE 的区别**：ENABLE 让普通用户受 RLS 约束，但表所有者绕过；FORCE 让所有人都受约束（包括连接池用的应用账号，如果它正好是表所有者）。
- **USING vs WITH CHECK**：USING 是查询时过滤（只能看到符合条件的行），WITH CHECK 是写入时校验（不能写不符合条件的行）。两者都要写。
- **current_setting('app.current_tenant', true)** 中的 `true` 表示"找不到时返回 NULL 而不是报错"，避免没设变量时直接 500。

#### 应用层设置 GUC 变量

```java
@Component
public class TenantConnectionInitializer implements ConnectionCustomizer {

    @Override
    public void customize(Connection conn) throws SQLException {
        String tenantId = TenantContext.getTenantId();
        if (tenantId == null) {
            throw new SQLException("refuse: no tenant set");
        }
        try (Statement stmt = conn.createStatement()) {
            // ★ SET LOCAL 仅在当前事务内有效，事务结束自动清空
            stmt.execute("SET LOCAL app.current_tenant = '" + escape(tenantId) + "'");
        }
    }

    private String escape(String s) {
        // 防 SQL 注入：tenant_id 必须只允许 [a-zA-Z0-9_-]
        if (!s.matches("[a-zA-Z0-9_-]+"))
            throw new IllegalArgumentException("invalid tenant id");
        return s;
    }
}
```

**逐段点透**：

- **SET LOCAL 的事务作用域**：必须在事务里设置，事务提交/回滚后自动清空。这避免了连接归还连接池后变量"残留"被下一个租户用错。
- **不能用 PreparedStatement 参数化**：PostgreSQL 的 SET 命令不支持参数绑定，只能拼字符串，所以 escape 必须严格。
- **每次 borrow 连接都要设**：如果连接池复用连接，前一次事务留下的 GUC 不能信。

### 2.3 ShardingSphere 多租户分库

当大租户数据量太大要单独分库时，ShardingSphere 是 Java 生态的事实标准方案。

```yaml
# application.yml
spring:
  shardingsphere:
    datasource:
      names: ds_default,ds_alice,ds_bob
      ds_default:
        type: com.zaxxer.hikari.HikariDataSource
        jdbc-url: jdbc:postgresql://default-db:5432/saas
      ds_alice:
        type: com.zaxxer.hikari.HikariDataSource
        jdbc-url: jdbc:postgresql://alice-dedicated-db:5432/saas
      ds_bob:
        type: com.zaxxer.hikari.HikariDataSource
        jdbc-url: jdbc:postgresql://bob-dedicated-db:5432/saas
    rules:
      sharding:
        tables:
          orders:
            actual-data-nodes: ds_$->{['default','alice','bob']}.orders
            database-strategy:
              hint:
                sharding-algorithm-name: tenant_routing
        sharding-algorithms:
          tenant_routing:
            type: CLASS_BASED
            props:
              strategy: HINT
              algorithmClassName: com.app.TenantHintShardingAlgorithm
```

```java
public class TenantHintShardingAlgorithm implements HintShardingAlgorithm<String> {
    @Override
    public Collection<String> doSharding(Collection<String> dsNames,
                                         HintShardingValue<String> hintValue) {
        String tenant = hintValue.getValues().iterator().next();
        return switch (tenant) {
            case "alice" -> List.of("ds_alice");
            case "bob"   -> List.of("ds_bob");
            default      -> List.of("ds_default");
        };
    }
}
```

**逐段点透**：

- **HINT 路由**：不是按字段值分片，而是显式告诉分片框架"这次请求属于谁"。比"按 tenant_id mod N"更灵活，能支持租户动态迁移到独立 DB。
- **租户从 default 升级到独立 DB** 的流程：先在配置里加新数据源 → 双写新老库 → 校验 → 切流 → 老库删数据。
- **要避免跨库 JOIN**：HINT 路由后跨库查询必须由应用层做聚合。

### 2.4 Spring 的 TenantResolver 实现

完整的请求链路：HTTP 请求 → Filter 解析租户 → ThreadLocal 存储 → DAO 自动拼条件。

```java
@Component
public class TenantContext {
    private static final ThreadLocal<String> CURRENT = new ThreadLocal<>();

    public static void set(String tenantId) { CURRENT.set(tenantId); }
    public static String getTenantId() { return CURRENT.get(); }
    public static void clear() { CURRENT.remove(); }
}

@Component
@Order(Ordered.HIGHEST_PRECEDENCE)   // ★ 最先执行
public class TenantResolverFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse resp,
                                    FilterChain chain) throws ServletException, IOException {
        String tenantId = resolveTenant(req);
        if (tenantId == null) {
            resp.setStatus(401);
            resp.getWriter().write("{\"error\":\"missing tenant\"}");
            return;
        }
        try {
            TenantContext.set(tenantId);
            MDC.put("tenant", tenantId);             // ★ 同步进 MDC，日志能看到
            chain.doFilter(req, resp);
        } finally {
            TenantContext.clear();
            MDC.remove("tenant");
        }
    }

    private String resolveTenant(HttpServletRequest req) {
        // 优先级：JWT > Header > 子域名
        String fromJwt = JwtUtil.getClaim(req.getHeader("Authorization"), "tenant");
        if (fromJwt != null) return fromJwt;

        String fromHeader = req.getHeader("X-Tenant-Id");
        if (fromHeader != null) return fromHeader;

        String host = req.getServerName();   // alice.app.com
        int dot = host.indexOf('.');
        if (dot > 0) return host.substring(0, dot);

        return null;
    }
}
```

**异步任务的传播（最容易翻车）**：

```java
@Configuration
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
        exec.setCorePoolSize(8);
        exec.setMaxPoolSize(32);
        exec.setQueueCapacity(200);
        exec.setTaskDecorator(new TenantAwareTaskDecorator());   // ★ 关键
        exec.initialize();
        return exec;
    }
}

public class TenantAwareTaskDecorator implements TaskDecorator {
    @Override
    public Runnable decorate(Runnable runnable) {
        String tenant = TenantContext.getTenantId();
        Map<String, String> mdc = MDC.getCopyOfContextMap();
        return () -> {
            try {
                if (tenant != null) TenantContext.set(tenant);
                if (mdc != null) MDC.setContextMap(mdc);
                runnable.run();
            } finally {
                TenantContext.clear();
                MDC.clear();
            }
        };
    }
}
```

**逐段点透**：

- **TaskDecorator 在 submit 时捕获**：在主线程提交时拿到 tenant，包装到 Runnable 内部。这是 capture-replay-restore 模式，参考 [线程池上下文透传](/article/java/threadpool#案例-mdc--traceid-失踪)。
- **finally 必须 clear**：worker 线程池复用，不清的话下一个任务会"看见"上一次的 tenant，串数据。
- **更通用的方案是 TransmittableThreadLocal**：阿里的 TTL 能自动处理嵌套池场景。

### 2.5 K8s Namespace + ResourceQuota 的计算隔离

Enterprise 租户独占 Pod 时，必须配 ResourceQuota，否则一个租户的 Pod 跑爆所有节点资源。

```yaml
# 1. 给租户创建 Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-alice
  labels:
    tenant: alice
    tier: enterprise

---
# 2. 整个 Namespace 的资源上限
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-alice-quota
  namespace: tenant-alice
spec:
  hard:
    requests.cpu: "16"           # 总申请 16 核
    requests.memory: 32Gi
    limits.cpu: "32"             # 总限制 32 核
    limits.memory: 64Gi
    persistentvolumeclaims: "10"
    pods: "20"
    services.loadbalancers: "2"

---
# 3. 单个 Pod 的默认值（防止开发忘填 limits）
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-alice-defaults
  namespace: tenant-alice
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "4"
        memory: 8Gi

---
# 4. NetworkPolicy 默认拒绝所有入流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: tenant-alice
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              tenant: alice          # 仅允许同租户内 Pod 互访
        - namespaceSelector:
            matchLabels:
              role: ingress          # 允许 ingress controller
  egress:
    - {}                              # 允许所有出流量（实际生产应限制）
```

**逐段点透**：

- **ResourceQuota 不会限制单个 Pod**：必须配 LimitRange 给单 Pod 默认值和上限。否则一个 Pod 申请 16 核就把整个 quota 用光。
- **NetworkPolicy 默认拒绝**：标准做法是"先全拒，再开白名单"。如果不配 NetworkPolicy，租户 Pod 之间默认完全互通，这是个安全大坑。
- **PriorityClass 给 Enterprise 高优先**：

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: tenant-enterprise
value: 1000
preemptionPolicy: PreemptLowerPriority   # 资源紧张时可以抢占低优先 Pod
```

### 2.6 Service Mesh 的多租户流量隔离

Istio + Envoy 在多租户中能做的：基于 JWT claim 的路由、限流、链路追踪。

```yaml
# 1. JWT 校验，提取 tenant claim 到 metadata
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: app
spec:
  selector:
    matchLabels:
      app: api-gateway
  jwtRules:
    - issuer: "https://auth.app.com"
      jwksUri: "https://auth.app.com/.well-known/jwks.json"
      outputClaimToHeaders:
        - header: "x-tenant-id"
          claim: "tenant"

---
# 2. 基于 header 路由到不同后端
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: orders-route
spec:
  hosts: [orders]
  http:
    - match:
        - headers:
            x-tenant-id:
              prefix: "ent-"          # Enterprise 租户走独立后端
      route:
        - destination:
            host: orders
            subset: enterprise
    - route:                          # 其他租户走共享后端
        - destination:
            host: orders
            subset: shared

---
# 3. 租户级限流（EnvoyFilter）
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: tenant-rate-limit
spec:
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: SIDECAR_INBOUND
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.filters.http.local_ratelimit
          typed_config:
            "@type": type.googleapis.com/udpa.type.v1.TypedStruct
            type_url: type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
            value:
              stat_prefix: tenant_rate_limit
              token_bucket:
                max_tokens: 100
                tokens_per_fill: 100
                fill_interval: 60s
              filter_enabled:
                runtime_key: local_rate_limit_enabled
                default_value:
                  numerator: 100
              descriptors:
                - entries:
                    - key: tenant_id
                      value: free                 # Free 租户限流配置
                  token_bucket:
                    max_tokens: 10
                    tokens_per_fill: 10
                    fill_interval: 60s
```

**逐段点透**：

- **outputClaimToHeaders**：Envoy 把 JWT 里的 tenant claim 提取成 HTTP header，下游服务直接读 header，不用每个服务自己解 JWT。
- **subset 路由**：通过 DestinationRule 把 orders 服务的 Pod 按 label 分组，不同租户走不同 subset。
- **EnvoyFilter 是逃生舱**：常规的 Istio CRD 表达不了的复杂 envoy 配置都通过 EnvoyFilter 注入。生产用要小心：Istio 升级时 EnvoyFilter 配置可能不兼容。

### 2.7 租户级限流（Sentinel + 自研）

Sentinel 默认按"资源"（Resource）+ "应用"（Origin）维度。要按租户限流，需要扩展。

```java
// 把租户作为 origin 透传给 Sentinel
@Component
public class TenantSentinelOriginParser implements RequestOriginParser {
    @Override
    public String parseOrigin(HttpServletRequest request) {
        return TenantContext.getTenantId();
    }
}

@Configuration
public class SentinelConfig {

    @PostConstruct
    public void initRules() {
        List<FlowRule> rules = new ArrayList<>();

        // 通用规则：每个租户最多 100 QPS
        FlowRule rule1 = new FlowRule("/api/orders");
        rule1.setCount(100);
        rule1.setGrade(RuleConstant.FLOW_GRADE_QPS);
        rule1.setLimitApp("default");      // 对所有租户生效
        rules.add(rule1);

        // 特殊规则：Free 租户更严
        FlowRule rule2 = new FlowRule("/api/orders");
        rule2.setCount(10);
        rule2.setGrade(RuleConstant.FLOW_GRADE_QPS);
        rule2.setLimitApp("tier_free");    // 仅对 tier_free 类型租户
        rule2.setStrategy(RuleConstant.STRATEGY_DIRECT);
        rules.add(rule2);

        FlowRuleManager.loadRules(rules);
    }
}
```

**自研更灵活的方案：基于 Redis + Lua 的租户级令牌桶**：

```lua
-- token_bucket.lua
local key = KEYS[1]                    -- "ratelimit:tenant:alice:orders"
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])  -- token / second
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(bucket[1]) or capacity
local last = tonumber(bucket[2]) or now

local elapsed = math.max(0, now - last)
tokens = math.min(capacity, tokens + elapsed * refill_rate)

if tokens < requested then
    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
    redis.call('EXPIRE', key, 60)
    return 0
end

tokens = tokens - requested
redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
redis.call('EXPIRE', key, 60)
return 1
```

```java
@Component
public class TenantRateLimiter {
    private final RedisTemplate<String, String> redis;
    private final RedisScript<Long> script;

    public boolean tryAcquire(String tenantId, String resource, int permits) {
        TenantConfig cfg = tenantConfigService.get(tenantId);
        int capacity = cfg.getRateLimitCapacity();
        int refillRate = cfg.getRateLimitRefillRate();

        String key = "rl:" + tenantId + ":" + resource;
        Long allowed = redis.execute(script, List.of(key),
            String.valueOf(capacity),
            String.valueOf(refillRate),
            String.valueOf(System.currentTimeMillis() / 1000),
            String.valueOf(permits));
        return allowed != null && allowed == 1L;
    }
}
```

**逐段点透**：

- **Lua 脚本是原子的**：Redis 单线程执行 Lua，避免"读-改-写"的竞态。
- **令牌桶 vs 漏桶**：令牌桶允许突发（攒下来的 token 可以一次性用），漏桶强制平滑。租户限流通常用令牌桶，让短时高峰能扛住。
- **每租户每资源独立 key**：Redis 内存占用和租户数 × 资源数 成正比，要算清楚。1 万租户 × 10 资源 = 10 万 key，每 key ~100B = 10MB，可接受。

> **口诀**：租户级限流必须按租户分级配置；用 Redis + Lua 做集群级；用 Sentinel 做单机兜底。

---

## 3. 实测数据：三种隔离方案的成本曲线

> 测试环境：AWS RDS PostgreSQL 16 / db.r6g.xlarge / 4 vCPU 32GB / EBS gp3 500GB

### 3.1 三种数据隔离方案的成本对比（10 / 100 / 1000 / 10000 租户）

每租户假设 1GB 数据、1000 QPS 峰值、连接池 10。

| 租户数 | 共享 Schema | 独立 Schema | 独立 DB |
| --- | --- | --- | --- |
| **10** | 1 实例 / $250/月 | 1 实例 / $250/月 | 10 实例 / $2,500/月 |
| **100** | 1 实例（升大）/ $500/月 | 1 实例（升大）/ $500/月 | 100 实例 / $25,000/月 |
| **1,000** | 主从 + 读写分离 / $2,000/月 | 报错（schema 数 > 1000 元数据慢）| 1,000 实例 / $250,000/月 |
| **10,000** | 分库分表（10 库）/ $20,000/月 | 不可行 | 不可行 |

**结论性口诀**：

- **千租户以下**：共享 Schema 一定胜出（成本 1/100）
- **千~万租户**：必须分库分表
- **大租户独立 DB**：仅对 ARR > $50K 的客户合算（成本 + 隔离 + 合规综合）

### 3.2 RLS 对查询性能的影响

测试 `SELECT * FROM orders WHERE user_id = ?` 在 1000 万行表上的 QPS：

| 配置 | QPS | P99 (ms) | CPU% |
| --- | --- | --- | --- |
| 无 RLS，应用拼 WHERE tenant_id | 18,500 | 4.2 | 65% |
| RLS USING + GUC 变量 | 16,800 | 4.6 | 70% |
| RLS USING + 函数返回 tenant | 9,200 | 8.1 | 92% |

**结论**：

- **GUC 变量方案性能损失 ~10%**：可以接受
- **函数方案损失 ~50%**：因为函数对每行都要调用一次，无法被规划器优化
- **RLS USING 子句必须能被索引利用**：例如 `tenant_id = current_setting('...')` 是常量化的，可以走索引；但 `tenant_id IN (SELECT ... FROM ...)` 不行

### 3.3 K8s Namespace 数对集群的影响

测试不同 Namespace 数下的 kube-apiserver 响应时间：

| Namespace 数 | Pod 总数 | api-server P99 (ms) | etcd 大小 |
| --- | --- | --- | --- |
| 100 | 1,000 | 45 | 200MB |
| 500 | 5,000 | 120 | 1.2GB |
| 1,000 | 10,000 | 380 | 3.5GB |
| 2,000 | 20,000 | 1,200 (不稳) | 8GB |

**结论性口诀**：

- **单集群 Namespace 数 ≤ 500**：超过后 kube-apiserver 退化
- **大 SaaS 必须多集群**：每集群 500 个 Enterprise 租户，按地域 / 行业切分集群
- **etcd 容量上限是关键瓶颈**：默认 8GB，超过 6GB 性能急剧下降

### 3.4 多租户限流的精度

对比 Sentinel 单机限流 vs Redis 集群限流在 100 个租户、每租户 100 QPS 的场景：

| 方案 | 实际限流误差 | 增加 P99 (ms) |
| --- | --- | --- |
| Sentinel 单机（10 节点） | ±35%（节点不均） | +0.1 |
| Sentinel 集群（独立 Token Server） | ±5% | +1.5 |
| Redis Lua 令牌桶 | ±2% | +2.0 |
| Envoy local_ratelimit | ±25%（无跨实例同步）| +0.05 |

**结论**：

- **要精度选 Redis Lua**：能容忍 +2ms 延迟时首选
- **要低延迟选 Envoy local + 接受误差**：边缘网关层用
- **不要用 Sentinel 单机做"全局限流"**：误差太大，租户会投诉

### 3.5 大租户对小租户的 Noisy Neighbor 实测

模拟一个 Enterprise 租户跑 5000 QPS 的批量任务，观察 Free 租户的 P99：

| 隔离配置 | Free 租户 P99 (ms) |
| --- | --- |
| 无任何隔离（共享线程池 + 共享 DB 连接） | 1,800 (受影响 100x) |
| 仅租户级线程池 | 280 (受影响 5x) |
| 租户级线程池 + DB 连接池 | 80 (受影响 1.5x) |
| + Resource Group 限 CPU | 45 (基本无影响) |
| + 大租户独立 DB | 22 (完全无影响) |

**结论性口诀**：**Noisy Neighbor 必须分层隔离**。仅在某一层做隔离（哪怕是看似最重要的 DB 连接池），下一层照样会被打穿。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 共享 Schema 漏 tenant_id —— "客户在自己的列表里看到了别家公司的订单"

#### 现象

某 SaaS 上线半年，一个客户在订单列表页看到了另一家公司的订单数据。客户截图发投诉，公司开了 P0 故障会。

#### 根因

代码：

```java
// OrderService.java
public List<Order> findRecentOrders(Long userId) {
    return orderMapper.selectByUserIdOrderByCreatedDesc(userId, 50);
}
```

```xml
<select id="selectByUserIdOrderByCreatedDesc">
    SELECT * FROM orders WHERE user_id = #{userId}
    ORDER BY created_at DESC LIMIT #{limit}
</select>
```

致命问题：**WHERE 子句忘了 tenant_id**。

更糟的是 user_id 在不同租户**可能重复**（自增主键各自递增）。租户 A 的 user_id=42 和租户 B 的 user_id=42 都查到了。

为什么 Code Review 没拦住？因为团队约定"OrderMapper 自动加 tenant_id"，但这个约定只在某个 BaseMapper 里实现，**不走 BaseMapper 的查询就裸奔**。这种"约定式"防护一旦人多就失控。

#### 修复

**短期**（立即上线）：每条 SQL 强制加 `tenant_id`：

```xml
<select id="selectByUserIdOrderByCreatedDesc">
    SELECT * FROM orders
    WHERE tenant_id = #{tenantId}
      AND user_id = #{userId}
    ORDER BY created_at DESC LIMIT #{limit}
</select>
```

**中期**（一周内）：加 MyBatis Interceptor 强制注入（见 2.1 节代码），上线后扫描所有 SQL：

```bash
# 在 CI 里跑
grep -rE "FROM\s+\w+\s+WHERE" src/main/resources/mapper/ | \
  grep -vE "tenant_id\s*=" | \
  grep -vE "tenant_global_table" > sql_without_tenant.txt

if [ -s sql_without_tenant.txt ]; then
    echo "FOUND SQL WITHOUT TENANT_ID:"
    cat sql_without_tenant.txt
    exit 1
fi
```

**长期**（一个月内）：上 PostgreSQL RLS（见 2.2 节）。这是**最后一道防线**：哪怕应用层 SQL 漏了，DB 也会拦住。

#### 验证

```java
@SpringBootTest
class TenantIsolationTest {

    @Test
    void mustNotLeakAcrossTenants() {
        // 准备：租户 A 和 B 各自有 user_id=42 的用户和订单
        TenantContext.set("tenant-a");
        Order orderA = orderService.create(42L, BigDecimal.TEN);

        TenantContext.set("tenant-b");
        Order orderB = orderService.create(42L, BigDecimal.ONE);

        // 验证：B 看不到 A 的订单
        TenantContext.set("tenant-b");
        List<Order> bOrders = orderService.findRecentOrders(42L);
        assertEquals(1, bOrders.size());
        assertEquals(orderB.getId(), bOrders.get(0).getId());

        // 反向验证
        TenantContext.set("tenant-a");
        List<Order> aOrders = orderService.findRecentOrders(42L);
        assertEquals(orderA.getId(), aOrders.get(0).getId());
    }
}
```

把这种测试加到每个核心业务，并在 CI 强制运行。

> **口诀**：跨租户数据泄漏是 SaaS 的死刑判决。**一道防线不够，至少三层**：应用过滤 + Interceptor 注入 + DB RLS 兜底。

---

### 案例 ② RLS 配置错误 —— "管理员看不到自己创建的数据"

#### 现象

启用 RLS 的第二天，租户管理员投诉："我刚创建的订单，刷新一下就消失了。" 但 DB 里 SELECT 直接查能看到数据。

#### 根因

错误配置：

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_select ON orders
    FOR SELECT                       -- ★ 只对 SELECT 生效
    USING (tenant_id = current_setting('app.current_tenant', true));
```

注意 `FOR SELECT` —— 只对查询生效。INSERT 没有 WITH CHECK 限制，**任何 tenant_id 都能写进去**。

写入流程：

```
应用：INSERT INTO orders (tenant_id, user_id, amount)
              VALUES ('tenant-a', 1, 99);

但此时 SET LOCAL app.current_tenant = 'tenant-a-buggy';   -- 拼错了
                                       ^^^^^^^

结果：
  数据写进去了，tenant_id = 'tenant-a'
  但当前 session 的 app.current_tenant = 'tenant-a-buggy'
  下一次 SELECT 就过不了 RLS 过滤
```

为什么管理员特别容易踩？因为管理员后台用了一个"impersonate"（扮演）功能，把自己的 tenant 切到目标租户，但代码里把 setting 的 key 拼错了，写入用的是错误租户上下文。

#### 修复

**修复 1：加 WITH CHECK 子句**（最重要）

```sql
DROP POLICY tenant_select ON orders;

CREATE POLICY tenant_isolation ON orders
    FOR ALL                          -- ★ 对增删改查都生效
    USING (tenant_id = current_setting('app.current_tenant', true))
    WITH CHECK (tenant_id = current_setting('app.current_tenant', true));
```

加上 WITH CHECK 后，任何 INSERT / UPDATE 都会校验：写入的 tenant_id 必须等于当前 GUC。如果不匹配，直接报错。

**修复 2：FORCE RLS** —— 表所有者也受约束：

```sql
ALTER TABLE orders FORCE ROW LEVEL SECURITY;
```

否则用 owner 账号连接的应用会绕过 RLS。

**修复 3：connection borrow 时严格校验**：

```java
public class TenantConnectionInitializer {
    public void onBorrow(Connection conn) throws SQLException {
        String tenant = TenantContext.getTenantId();
        if (tenant == null || !tenant.matches("[a-zA-Z0-9_-]+")) {
            throw new SQLException("invalid tenant: " + tenant);
        }
        try (Statement stmt = conn.createStatement()) {
            stmt.execute("SET LOCAL app.current_tenant = '" + tenant + "'");
        }
    }
}
```

#### 验证

```sql
-- 验证 RLS 生效：换 tenant 后看不到原数据
BEGIN;
SET LOCAL app.current_tenant = 'tenant-a';
INSERT INTO orders (tenant_id, user_id, amount) VALUES ('tenant-a', 1, 99);
SELECT count(*) FROM orders;  -- 1
COMMIT;

BEGIN;
SET LOCAL app.current_tenant = 'tenant-b';
SELECT count(*) FROM orders;  -- 0  ✓
INSERT INTO orders (tenant_id, user_id, amount) VALUES ('tenant-a', 1, 99);
-- 期望：报错 "new row violates row-level security policy"  ✓
COMMIT;
```

> **口诀**：RLS 必须 USING + WITH CHECK 双管齐下，FOR ALL 而不是 FOR SELECT。FORCE RLS 让所有人都受约束。

---

### 案例 ③ Noisy Neighbor —— "大客户跑批，所有小客户接口变慢"

#### 现象

每月 1 号凌晨，某 Enterprise 客户启动月度报表批处理任务，5 分钟内系统所有租户的 P99 从 200ms 飙到 5s+，告警轰炸值班群。

#### 根因

公司 SaaS 用的是**共享 DB + 共享应用集群**。Enterprise 客户的批处理任务：

1. 一次性查询 6 个月订单（300 万行，全表扫描）
2. 占用 80% DB 连接池（30/40 个连接被它占住）
3. 占用 90% Tomcat 线程（200 个工作线程几乎全用来处理它的请求）
4. 大量 buffer pool 替换，普通查询 cache miss 率从 5% 涨到 60%

**所有共享资源同时被打穿**，单一隔离手段救不了。

#### 修复

**修复 1：租户级线程池（Bulkhead）**

```java
@Component
public class TenantBulkhead {
    private final Map<String, ThreadPoolExecutor> pools = new ConcurrentHashMap<>();

    public <T> CompletableFuture<T> submit(String tenantId, Supplier<T> task) {
        ThreadPoolExecutor pool = pools.computeIfAbsent(tenantId, this::createPool);
        return CompletableFuture.supplyAsync(task, pool);
    }

    private ThreadPoolExecutor createPool(String tenantId) {
        TenantTier tier = tenantTierResolver.resolve(tenantId);
        int core = switch (tier) {
            case FREE       -> 2;
            case PRO        -> 8;
            case ENTERPRISE -> 32;
        };
        return new ThreadPoolExecutor(
            core, core * 2, 60, TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(100),
            new ThreadFactoryBuilder()
                .setNameFormat("biz-" + tenantId + "-%d").build(),
            new ThreadPoolExecutor.AbortPolicy());   // 满了直接拒
    }
}
```

注意：每个租户独立线程池意味着**线程数会大幅增长**。100 个租户 × 8 线程 = 800 线程，有 OOM 和上下文切换风险。生产中要：
- 给租户分级（不是每个租户都独立）
- Free 租户共享一个池
- 长时间无活跃的租户池要回收

**修复 2：租户级 DB 连接池**

```java
@Configuration
public class TenantDataSourceRouter extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        TenantTier tier = tenantTierResolver.resolve(TenantContext.getTenantId());
        return tier;   // FREE / PRO / ENTERPRISE → 不同 DataSource
    }
}

@Bean
public DataSource routingDataSource() {
    Map<Object, Object> targets = new HashMap<>();
    targets.put(TenantTier.FREE,       buildPool("free", 5));
    targets.put(TenantTier.PRO,        buildPool("pro", 15));
    targets.put(TenantTier.ENTERPRISE, buildPool("enterprise", 30));

    TenantDataSourceRouter ds = new TenantDataSourceRouter();
    ds.setTargetDataSources(targets);
    return ds;
}
```

**修复 3：批处理任务必须去单独通道**

```java
public interface OrderQueryService {
    // 实时接口，走小数据集
    List<Order> findRecent(String tenantId, int limit);

    // 报表接口，限流 + 走从库
    @TenantRateLimit(qps = 1, burst = 1)
    @ReadOnly(slave = "report-slave")        // ★ 路由到专门的报表从库
    @Async("reportExecutor")
    CompletableFuture<List<Order>> findForReport(String tenantId, LocalDate from, LocalDate to);
}
```

报表从库可以是：
- 一个延迟 5 分钟的从库
- 一个专门的 OLAP（ClickHouse / Doris）副本
- S3 + Athena 等无服务化方案

#### 验证

```bash
# 用 wrk 模拟 Enterprise 大客户的报表请求
wrk -t 4 -c 100 -d 60s -H 'X-Tenant-Id: enterprise-1' \
    http://api.app.com/orders/report?from=2026-01-01&to=2026-05-01 &

# 同时模拟 Free 客户的普通请求
wrk -t 2 -c 20 -d 60s -H 'X-Tenant-Id: free-1' \
    http://api.app.com/orders/recent

# 修复前：Free 客户 P99 > 5000ms
# 修复后：Free 客户 P99 < 100ms
```

> **口诀**：Noisy Neighbor 不能"按一处"防。线程池、DB 连接池、缓存、网络带宽、专用从库 —— 每一层都做隔离才稳。

---

### 案例 ④ 租户路由失败 —— "Header 在异步任务中丢失"

#### 现象

订单创建走 `@Async` 异步发邮件。日志里发现邮件发送时 `tenant_id` 偶尔是 null，导致 RLS 拦截，邮件没发出去。但同步主流程一切正常。

#### 根因

```java
@Service
public class OrderService {

    @Autowired EmailService emailService;

    public Order create(OrderRequest req) {
        Order order = orderRepository.save(...);
        emailService.sendOrderConfirm(order.getId());   // ★ 异步
        return order;
    }
}

@Service
public class EmailService {
    @Async                                              // ← 异步执行
    public void sendOrderConfirm(Long orderId) {
        // 这里 TenantContext.getTenantId() == null
        Order order = orderRepository.findById(orderId);   // RLS 拦截
        ...
    }
}
```

异步任务在新线程跑，TenantContext（ThreadLocal）没有透传过去。前文 2.4 节的 TaskDecorator 是解法，但**很多团队的 @Async 配置没装 decorator**。

更深层问题：项目里有**多个线程池**，有的装了 decorator 有的没装。Spring 默认的 `SimpleAsyncTaskExecutor` 没装。

#### 修复

**修复 1：所有线程池统一走带 decorator 的**

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
        exec.setCorePoolSize(8);
        exec.setMaxPoolSize(32);
        exec.setQueueCapacity(200);
        exec.setTaskDecorator(new TenantAwareTaskDecorator());
        exec.setThreadNamePrefix("async-");
        exec.initialize();
        return exec;
    }

    @Bean("emailExecutor")
    public Executor emailExecutor() {
        ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
        exec.setTaskDecorator(new TenantAwareTaskDecorator());   // ★ 不要忘
        ...
        return exec;
    }
}
```

**修复 2：消息队列场景必须显式塞 tenant**

```java
@Service
public class OrderEventPublisher {
    public void publish(Order order) {
        Message msg = MessageBuilder
            .withPayload(order)
            .setHeader("X-Tenant-Id", TenantContext.getTenantId())
            .setHeader("X-Trace-Id", MDC.get("traceId"))
            .build();
        kafkaTemplate.send("orders", msg);
    }
}

@KafkaListener(topics = "orders")
public void onOrder(Message<Order> msg) {
    String tenantId = msg.getHeaders().get("X-Tenant-Id", String.class);
    if (tenantId == null) {
        log.error("tenant missing in message, dropping: {}", msg);
        return;
    }
    try {
        TenantContext.set(tenantId);
        MDC.put("tenant", tenantId);
        process(msg.getPayload());
    } finally {
        TenantContext.clear();
        MDC.clear();
    }
}
```

**修复 3：定时任务必须显式遍历租户**

```java
@Component
public class TenantAwareScheduler {

    @Scheduled(cron = "0 0 2 * * *")
    public void dailyReport() {
        List<String> tenants = tenantService.allActive();
        for (String tenantId : tenants) {
            try {
                TenantContext.set(tenantId);
                MDC.put("tenant", tenantId);
                reportService.generate();
            } catch (Exception e) {
                log.error("report failed for {}", tenantId, e);
            } finally {
                TenantContext.clear();
                MDC.clear();
            }
        }
    }
}
```

#### 验证

```java
@SpringBootTest
class TenantPropagationTest {

    @Autowired AsyncTestService svc;

    @Test
    void tenantMustPropagateToAsync() throws Exception {
        TenantContext.set("test-tenant");
        CompletableFuture<String> fut = svc.echoTenantAsync();
        assertEquals("test-tenant", fut.get(2, TimeUnit.SECONDS));
    }
}

@Service
class AsyncTestService {
    @Async
    public CompletableFuture<String> echoTenantAsync() {
        return CompletableFuture.completedFuture(TenantContext.getTenantId());
    }
}
```

把这个测试加到 CI 强制运行。

> **口诀**：tenant 上下文跨线程必须显式透传。@Async 装 decorator；MQ 塞 message header；定时任务循环 tenant 列表。三件套缺一不可。

---

### 案例 ⑤ 数据迁移 —— "从共享 Schema 拆到独立 DB"

#### 现象

一个 Enterprise 客户业务暴增，单租户数据从 1GB 涨到 80GB，长查询和锁争用越来越严重。决策：把这个租户从共享 DB 迁出到独立 RDS 实例。客户要求停机时间 < 5 分钟。

#### 根因

不能一次性 dump+restore：
- 80GB 数据 mysqldump 要 30 分钟
- 双写期间数据可能不一致
- 切流期间应用配置怎么动态更新

#### 修复：双写灰度切流的标准模式

```
T0 (准备):
  - 在新 DB 上创建表结构（与共享 DB 完全一致）
  - 应用上线"双写"开关（默认 OFF）

T1 (开双写):
  - 应用代码：写入时同时写共享 DB 和新 DB
  - 读取仍走共享 DB
  - 监控：双写一致性（每 100 条采样比对）

T2 (历史数据迁移):
  - 用 pg_dump --where "tenant_id='alice'" 导出历史数据
  - 在新 DB restore（约 30 分钟，期间双写继续）
  - 用 logical replication 追增量

T3 (校验):
  - 全量比对：两边记录数、关键字段 checksum
  - 至少跑 24 小时无差异

T4 (切读):
  - 应用配置：读流量切到新 DB（按 tenant_id 路由）
  - 持续 6~24 小时，监控异常

T5 (停写双写):
  - 应用配置：只写新 DB
  - 共享 DB 上的数据保留 7 天回退用

T6 (回收):
  - 7 天后从共享 DB DELETE WHERE tenant_id = 'alice'
```

**双写代码**：

```java
@Component
public class DualWriteOrderRepository {

    private final OrderRepository sharedRepo;
    private final OrderRepository dedicatedRepo;
    private final FeatureFlag dualWriteEnabled;
    private final FeatureFlag readFromDedicated;

    public Order save(Order order) {
        Order saved = sharedRepo.save(order);

        if (dualWriteEnabled.isOn(order.getTenantId())) {
            try {
                dedicatedRepo.save(order);
            } catch (Exception e) {
                // 双写失败不能影响主流程，但要记录用于补偿
                log.error("dual write failed for order {}", order.getId(), e);
                dualWriteFailureRecorder.record(order);
            }
        }
        return saved;
    }

    public Order findById(Long id, String tenantId) {
        if (readFromDedicated.isOn(tenantId)) {
            return dedicatedRepo.findById(id);
        }
        return sharedRepo.findById(id);
    }
}
```

**校验脚本**：

```python
# verify_consistency.py
import psycopg2

src = psycopg2.connect("...shared DB...")
dst = psycopg2.connect("...dedicated DB...")

with src.cursor() as cs, dst.cursor() as cd:
    cs.execute("""
        SELECT id, MD5(ROW(tenant_id, user_id, amount, status, updated_at)::text)
        FROM orders WHERE tenant_id = 'alice' ORDER BY id
    """)
    cd.execute("""
        SELECT id, MD5(ROW(tenant_id, user_id, amount, status, updated_at)::text)
        FROM orders ORDER BY id
    """)

    src_rows = {r[0]: r[1] for r in cs.fetchall()}
    dst_rows = {r[0]: r[1] for r in cd.fetchall()}

    only_in_src = set(src_rows) - set(dst_rows)
    only_in_dst = set(dst_rows) - set(src_rows)
    diff = {k: (src_rows[k], dst_rows[k]) for k in src_rows if k in dst_rows and src_rows[k] != dst_rows[k]}

    print(f"only_in_src: {len(only_in_src)}")
    print(f"only_in_dst: {len(only_in_dst)}")
    print(f"diff: {len(diff)}")
```

#### 验证

```bash
# 切流前后用 wrk 对比延迟
wrk -t 4 -c 50 -d 60s -H 'X-Tenant-Id: alice' http://api.app.com/orders

# 监控指标：
# - 切流后该租户 P99 应该下降（独立 DB 资源更多）
# - 共享 DB 上该租户的连接数应该归零
# - 应用错误率不应该升高
```

> **口诀**：大租户拆 DB 必须走"双写 + 灰度 + 回退"。绝不能"周末停机一次性切"，那种方案只在 < 1GB 数据时才适用。

---

### 案例 ⑥ 租户级限流配置错误 —— "Free 用户能跑 Enterprise 的 QPS"

#### 现象

某 SaaS 上线租户限流后，发现监控里 Free 租户的 QPS 偶尔超过了配置上限的 10 倍。重启服务后又恢复正常，但几小时后再次出现。

#### 根因

实现：

```java
public class SimpleTenantRateLimiter {
    private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();

    public boolean tryAcquire(String tenantId) {
        TenantConfig cfg = tenantConfigService.get(tenantId);
        RateLimiter rl = limiters.computeIfAbsent(tenantId,
            t -> RateLimiter.create(cfg.getQps()));   // ★ Bug 在这里
        return rl.tryAcquire();
    }
}
```

bug：`computeIfAbsent` **只在第一次创建时取 cfg.getQps()**。如果租户后来从 Free（QPS=10）升级到 Pro（QPS=100），limiter 仍然按 10 限流。**降级反过来更危险**：从 Pro 降到 Free 后，老 limiter 仍按 100 放行。

加上多实例部署：每个实例的 limiters Map 独立维护，**总 QPS = 单实例 QPS × 实例数**。10 个实例 = 实际 QPS 10 倍。

#### 修复

**修复 1：跨实例统一计数 —— 用 Redis**

```java
@Component
public class RedisRateLimiter {
    private final RedisTemplate<String, String> redis;
    private final RedisScript<Long> tokenBucketScript;

    public boolean tryAcquire(String tenantId, String resource) {
        TenantConfig cfg = tenantConfigService.get(tenantId);   // ★ 每次拿最新
        String key = "rl:" + tenantId + ":" + resource;

        Long allowed = redis.execute(tokenBucketScript,
            List.of(key),
            String.valueOf(cfg.getRateLimitCapacity()),
            String.valueOf(cfg.getRateLimitRefillRate()),
            String.valueOf(System.currentTimeMillis() / 1000),
            "1");
        return allowed != null && allowed == 1L;
    }
}
```

**修复 2：tier 变更后必须主动失效缓存**

```java
@Component
public class TenantConfigService {

    @Cacheable(cacheNames = "tenant-config", key = "#tenantId")
    public TenantConfig get(String tenantId) {
        return repository.findById(tenantId);
    }

    @CacheEvict(cacheNames = "tenant-config", key = "#tenantId")
    public void onTierChanged(String tenantId, TenantTier newTier) {
        repository.updateTier(tenantId, newTier);
        // 广播失效（因为多实例都有本地缓存）
        eventBus.publish(new TenantConfigChangedEvent(tenantId));
    }
}

@EventListener
public void onConfigChanged(TenantConfigChangedEvent evt) {
    cacheManager.getCache("tenant-config").evict(evt.getTenantId());
    // 通知 Redis 刷掉对应的限流器状态
    redis.delete("rl:" + evt.getTenantId() + ":*");
}
```

**修复 3：限流粒度要按 tier 区分配置（避免误改）**

```yaml
# tenant-tier-defaults.yaml
free:
  qps: 10
  burst: 20
  daily_quota: 1000
pro:
  qps: 100
  burst: 200
  daily_quota: 50000
enterprise:
  qps: 1000
  burst: 2000
  daily_quota: -1   # 无限
```

代码读 tier，不读单租户的 qps：

```java
TenantTier tier = tenantTierResolver.resolve(tenantId);
TierConfig tierCfg = tierConfigService.get(tier);
```

避免某次手抖把所有 Free 租户的 QPS 都改成了 1000。

#### 验证

```bash
# 模拟 Free 租户 100 QPS 灌入
ab -n 1000 -c 50 -H 'X-Tenant-Id: free-1' http://api.app.com/orders

# 期望：返回 429 (Too Many Requests) 比例 > 90%
# 实际：grep 'HTTP/1.1 429' access.log | wc -l
```

> **口诀**：限流配置必须**实时拿**而不是缓存死。多实例必须用集中存储（Redis）。tier 变更要广播失效。

---

### 案例 ⑦ 多租户日志混杂 —— "1 万行日志找不出某租户的请求"

#### 现象

凌晨 P0 故障，某 Enterprise 租户报障。值班想找他的请求日志，但应用 1 分钟产生 100 万行日志，grep 也要好几分钟，且没有有效的过滤维度。

#### 根因

日志格式：

```
2026-05-09 02:14:23 INFO  [http-nio-8080-exec-15] OrderService - create order success: id=12345
```

只有时间、级别、类名。**租户信息完全没打印**。jstack 也救不了，只能事后补。

#### 修复

**修复 1：日志 pattern 加 tenant**

```xml
<!-- logback-spring.xml -->
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>
                %d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%thread]
                tenant=%X{tenant} traceId=%X{traceId} userId=%X{userId}
                %logger{36} - %msg%n
            </pattern>
        </encoder>
    </appender>
</configuration>
```

`%X{tenant}` 是从 MDC 取的。前提是 Filter 里设了 `MDC.put("tenant", ...)`。

**修复 2：日志按租户分文件（仅 Enterprise）**

```xml
<appender name="ENTERPRISE-LOG" class="ch.qos.logback.classic.sift.SiftingAppender">
    <discriminator>
        <key>tenant</key>
        <defaultValue>shared</defaultValue>
    </discriminator>
    <sift>
        <appender name="FILE-${tenant}" class="ch.qos.logback.core.rolling.RollingFileAppender">
            <file>logs/tenant-${tenant}.log</file>
            <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
                <fileNamePattern>logs/tenant-${tenant}-%d{yyyy-MM-dd}.log.gz</fileNamePattern>
                <maxHistory>30</maxHistory>
            </rollingPolicy>
            <encoder>
                <pattern>%d{HH:mm:ss.SSS} %-5level [%thread] %logger - %msg%n</pattern>
            </encoder>
        </appender>
    </sift>
</appender>
```

注意：SiftingAppender 给每个 tenant 创建一个文件，**租户多时会大量打开 fd**。生产用必须配上限：

```xml
<sift>
    <maxAppenders>50</maxAppenders>   <!-- 仅缓存最近 50 个活跃租户 -->
</sift>
```

**修复 3：日志聚合到 ELK / Loki，按 tenant 字段索引**

```yaml
# Filebeat 提取 tenant 字段
processors:
  - dissect:
      tokenizer: '... tenant=%{tenant} traceId=%{traceId} ...'
      field: message
      target_prefix: ''
```

```kibana
# Kibana 查询：直接按 tenant 字段
tenant:"alice" AND level:"ERROR"
```

#### 验证

```bash
# 故意打 100 条不同租户的日志，确认能精准过滤
for i in $(seq 1 100); do
  curl -H "X-Tenant-Id: tenant-$((i%10))" http://api.app.com/orders/list &
done

# 期望：grep 'tenant=tenant-3' app.log | wc -l ≈ 10
```

> **口诀**：多租户日志的最低标准是**每行打 tenant 字段**。Enterprise 租户值得单独文件 / 单独索引。Filebeat / Loki 一定要按 tenant 建标签。

---

### 案例 ⑧ 加密密钥的租户级管理 —— "一个租户的数据被另一个租户的密钥解密"

#### 现象

某金融 SaaS 客户合规审计提出："你们所有租户的数据用同一把密钥加密，这不符合 SOC2 要求。一旦密钥泄漏，所有客户全部完蛋。"

#### 根因

旧实现：

```java
@Component
public class FieldEncryptor {
    @Value("${app.encryption.key}")
    private String masterKey;     // ★ 全局共用

    public String encrypt(String plain) {
        return AES.encrypt(plain, masterKey);
    }
}
```

**所有租户的所有敏感字段共用一个密钥**。这种设计：
- 密钥泄漏 = 全租户数据全暴露
- 删除租户时无法"密码学删除"（要真的物理 DELETE 才行）
- 不符合 GDPR 的"被遗忘权"高效实现要求

#### 修复：每租户一把 DEK（Data Encryption Key），用 KMS 管理

行业标准方案：**Envelope Encryption（信封加密）**

```
KMS 存：KEK (Key Encryption Key)，每个租户一把，永远不离开 KMS
       ├── tenant-alice → KEK-alice
       ├── tenant-bob   → KEK-bob

应用本地：DEK (Data Encryption Key)，用于实际加密数据
        每次加密时：
          1. 应用持有 DEK 的明文（仅在内存）
          2. 数据 = AES(plain, DEK)
          3. encrypted_dek = KMS.encrypt(DEK, KEK-tenant)
          4. 存储：[encrypted_data, encrypted_dek, kek_version]

解密时：
  1. 从存储取 encrypted_dek
  2. DEK_plain = KMS.decrypt(encrypted_dek, KEK-tenant)
  3. plain = AES.decrypt(encrypted_data, DEK_plain)
```

```java
@Component
public class EnvelopeEncryptor {

    private final KmsClient kms;
    private final Cache<String, byte[]> dekCache = Caffeine.newBuilder()
        .expireAfterWrite(5, TimeUnit.MINUTES)
        .maximumSize(1000)
        .build();

    public EncryptedField encrypt(String plain, String tenantId) {
        // 1. 生成新 DEK（256-bit）
        byte[] dek = SecureRandom.getInstanceStrong().generateSeed(32);

        // 2. 用 KEK 加密 DEK
        EncryptResponse encResp = kms.encrypt(EncryptRequest.builder()
            .keyId("alias/tenant-" + tenantId)
            .plaintext(SdkBytes.fromByteArray(dek))
            .build());
        byte[] encryptedDek = encResp.ciphertextBlob().asByteArray();

        // 3. 用 DEK 加密数据
        byte[] iv = new byte[12];
        SecureRandom.getInstanceStrong().nextBytes(iv);
        byte[] ciphertext = AES_GCM.encrypt(plain.getBytes(UTF_8), dek, iv);

        // 4. 立即清空 DEK 内存
        Arrays.fill(dek, (byte) 0);

        return new EncryptedField(ciphertext, encryptedDek, iv, encResp.keyId());
    }

    public String decrypt(EncryptedField field, String tenantId) {
        // 1. KMS 解密 DEK（带本地缓存，避免每次都调 KMS）
        String cacheKey = base64(field.getEncryptedDek());
        byte[] dek = dekCache.get(cacheKey, k -> {
            DecryptResponse resp = kms.decrypt(DecryptRequest.builder()
                .ciphertextBlob(SdkBytes.fromByteArray(field.getEncryptedDek()))
                .keyId("alias/tenant-" + tenantId)
                .build());
            return resp.plaintext().asByteArray();
        });

        // 2. 用 DEK 解密数据
        byte[] plain = AES_GCM.decrypt(field.getCiphertext(), dek, field.getIv());
        return new String(plain, UTF_8);
    }
}
```

**密钥销毁 = "密码学删除"**：

```java
public void deleteTenant(String tenantId) {
    // 1. 触发 KMS 删除该租户的 KEK（带 30 天等待期）
    kms.scheduleKeyDeletion(ScheduleKeyDeletionRequest.builder()
        .keyId("alias/tenant-" + tenantId)
        .pendingWindowInDays(30)
        .build());

    // KEK 被删除后，所有 encrypted_dek 都解不出来
    // 等同于该租户的所有数据"逻辑销毁"
    // 物理 DELETE 可以异步慢慢做，30 天后再来收尾
}
```

**逐段点透**：

- **DEK 在内存里要立即清零**：`Arrays.fill(dek, (byte) 0)`，避免 GC 期间被 dump 出来。
- **DEK 缓存的代价权衡**：每次都调 KMS 太慢（~10ms / 调用），缓存太久不安全。5 分钟是常见折中。
- **AES-GCM 而不是 CBC**：GCM 自带认证（AAD），CBC 不带，受 padding oracle 攻击。
- **KEK 别名 (alias/...)**：KMS 实际 keyId 是 UUID，alias 让代码可读且支持 key 轮换。

#### 验证

```bash
# 验证 1：删除一个租户的 KEK 后，其数据无法解密
aws kms schedule-key-deletion --key-id alias/tenant-test --pending-window-in-days 7
aws kms disable-key --key-id alias/tenant-test
# 应用日志：解密 test 租户数据时 KMS 返回错误

# 验证 2：审计 KMS 调用
aws cloudtrail lookup-events --lookup-attributes \
    AttributeKey=ResourceName,AttributeValue=alias/tenant-alice
# 应该看到所有 encrypt/decrypt 调用记录
```

> **口诀**：多租户加密用信封加密 + 每租户 KEK + 本地 DEK 缓存。删租户 = 删 KEK，比物理 DELETE 更快、更安全、更可审计。

---

### 案例 ⑨ 计费数据的精确归属 —— "收费单对不上账"

#### 现象

按用量计费的 SaaS（API 调用次数 / 流量 / 存储）。月底出账时多个租户投诉账单数字与他们自己的统计对不上，差异 5%~30%。

#### 根因

旧实现：

```java
@Aspect
public class BillingAspect {
    @Around("@annotation(Billable)")
    public Object meter(ProceedingJoinPoint pjp) throws Throwable {
        String tenantId = TenantContext.getTenantId();
        long start = System.currentTimeMillis();
        try {
            return pjp.proceed();
        } finally {
            long dur = System.currentTimeMillis() - start;
            asyncBillingPublisher.publish(tenantId, pjp.getSignature(), dur);
        }
    }
}

@Component
public class AsyncBillingPublisher {
    @Async("billingExecutor")
    public void publish(String tenantId, Signature sig, long dur) {
        kafka.send("billing", new BillingEvent(tenantId, sig, dur));
    }
}
```

问题：

1. **@Async 没装 TenantDecorator**：tenantId 在异步执行时被 capture 了一次，但 publish 内部又读 TenantContext 时已经是 null
2. **kafka.send 失败时静默丢弃**：每天丢几千条事件
3. **BillingEvent 没有幂等 key**：consumer 重复消费时数据翻倍
4. **采用本地内存采集 + 批量上报**：服务异常退出时未上报的数据全丢

每一项都是 1%~10% 的误差，叠加起来 5%~30%。

#### 修复：把计费当作金融级数据处理

**修复 1：每条计费事件带唯一幂等 key**

```java
public record BillingEvent(
    String eventId,           // ★ UUID，作为幂等 key
    String tenantId,
    String resource,
    long amount,
    long timestamp
) {}

@Aspect
public class BillingAspect {
    @Around("@annotation(Billable)")
    public Object meter(ProceedingJoinPoint pjp) throws Throwable {
        String tenantId = TenantContext.getTenantId();
        if (tenantId == null) {
            // ★ 计费场景下 tenant 为 null 是严重错误
            metrics.counter("billing.tenant_missing").increment();
            throw new IllegalStateException("billing without tenant");
        }
        long start = System.nanoTime();
        try {
            return pjp.proceed();
        } finally {
            long dur = (System.nanoTime() - start) / 1000;   // 微秒
            BillingEvent evt = new BillingEvent(
                UUID.randomUUID().toString(),
                tenantId,
                pjp.getSignature().toShortString(),
                dur,
                System.currentTimeMillis()
            );
            // 关键：同步写到本地持久化队列
            billingQueue.offer(evt);
        }
    }
}
```

**修复 2：本地持久化队列 + 守护线程批量上报**

```java
@Component
public class BillingQueue {
    private final BlockingQueue<BillingEvent> memQueue = new LinkedBlockingQueue<>(10000);
    private final ChronicleQueue diskQueue;   // ★ 磁盘持久化兜底

    @PostConstruct
    public void init() {
        diskQueue = ChronicleQueue.singleBuilder("/var/lib/app/billing").build();

        // 守护线程：从 memQueue 取 → 写 diskQueue → 批量上报 Kafka → 删
        new Thread(this::flushLoop, "billing-flusher").start();
    }

    public void offer(BillingEvent evt) {
        memQueue.offer(evt);
    }

    private void flushLoop() {
        ExcerptAppender appender = diskQueue.acquireAppender();
        while (!stopped) {
            try {
                List<BillingEvent> batch = drainBatch(memQueue, 1000, Duration.ofSeconds(1));
                if (batch.isEmpty()) continue;

                // 1. 先落盘（保证应用重启不丢）
                for (BillingEvent e : batch) {
                    appender.writeText(toJson(e));
                }

                // 2. 上报 Kafka
                for (BillingEvent e : batch) {
                    kafka.send("billing", e.tenantId(), e).get(5, TimeUnit.SECONDS);
                }

                // 3. 标记 diskQueue 中这批已处理（用 cycle 索引）
                ackToDiskQueue(batch);
            } catch (Exception e) {
                log.error("billing flush failed", e);
            }
        }
    }
}
```

**修复 3：Consumer 端幂等去重**

```java
@KafkaListener(topics = "billing")
public void onBilling(BillingEvent evt) {
    // 用 Redis SETNX 做幂等
    String dedupeKey = "billing:dedupe:" + evt.eventId();
    Boolean firstTime = redis.opsForValue().setIfAbsent(dedupeKey, "1", Duration.ofDays(7));
    if (Boolean.FALSE.equals(firstTime)) {
        log.debug("dedup billing event {}", evt.eventId());
        return;
    }

    billingDao.insert(evt);   // 入库（DB 也建唯一索引兜底）
}
```

**修复 4：每日对账**

```sql
-- 每天凌晨跑：网关层（Nginx 日志）和 billing DB 对账
WITH gateway_count AS (
    SELECT tenant_id, count(*) as cnt
    FROM nginx_log
    WHERE date = CURRENT_DATE - 1
    GROUP BY tenant_id
),
billing_count AS (
    SELECT tenant_id, count(*) as cnt
    FROM billing_event
    WHERE date = CURRENT_DATE - 1
    GROUP BY tenant_id
)
SELECT
    g.tenant_id,
    g.cnt as gateway_cnt,
    b.cnt as billing_cnt,
    ABS(g.cnt - b.cnt) * 1.0 / g.cnt as diff_ratio
FROM gateway_count g
JOIN billing_count b USING (tenant_id)
WHERE ABS(g.cnt - b.cnt) * 1.0 / g.cnt > 0.01;   -- 差异超过 1% 报警
```

#### 验证

```bash
# 模拟服务崩溃，看是否丢数据
# 1. 灌入 10000 条计费请求
ab -n 10000 -c 100 -H 'X-Tenant-Id: test' http://api.app.com/billable_api

# 2. 中途 kill -9 服务
kill -9 $(pgrep -f myapp)

# 3. 重启服务，等待 disk queue 处理完

# 4. 验证 billing DB 中 test 租户的事件数 == 10000（误差 < 0.1%）
psql -c "SELECT count(*) FROM billing_event WHERE tenant_id='test' AND date=CURRENT_DATE"
```

> **口诀**：计费数据按金融级处理。**幂等 ID + 本地持久化队列 + 集中存储幂等去重 + 每日对账**，四件套缺一不可。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 数据隔离层
- [ ] 所有共享 Schema 表的主键以 `tenant_id` 开头
- [ ] 所有索引必须以 `tenant_id` 开头
- [ ] 应用层 SQL 强制经过 Interceptor 注入 `tenant_id` 条件
- [ ] CI 检查：扫描 Mapper XML，发现没带 tenant_id 的 SQL 直接 fail
- [ ] PostgreSQL 启用 RLS：USING + WITH CHECK + FORCE 三件套
- [ ] 大租户独立 DB 时，路由方式必须是 HINT，不是字段值哈希

### 计算隔离层
- [ ] 租户分级（FREE / PRO / ENTERPRISE）配置中心化
- [ ] 不同 tier 用不同线程池 / DB 连接池
- [ ] K8s Namespace 配 ResourceQuota + LimitRange
- [ ] Enterprise 租户配 PriorityClass，资源紧张时不被 Free 抢占
- [ ] 长任务（报表 / 批处理）走独立通道，不混在通用接口

### 网络隔离层
- [ ] HTTPS 证书覆盖所有租户子域名（泛域名或 ACME 自动签发）
- [ ] K8s NetworkPolicy 默认拒绝，按需开白名单
- [ ] Service Mesh 提取 JWT claim 到 header，下游不重复解析
- [ ] CDN 按租户子域名独立缓存

### 上下文传播层
- [ ] TenantContext 用 ThreadLocal，请求开始 set，结束必须 clear
- [ ] @Async 和所有线程池配 TaskDecorator 透传 tenant
- [ ] Kafka / RocketMQ 消息必须带 tenant header
- [ ] 定时任务显式遍历租户列表，不要"假设有 tenant"
- [ ] 日志格式包含 `%X{tenant}`，所有日志能按租户过滤

### 限流隔离层
- [ ] 租户级限流配置按 tier 区分，不按单租户
- [ ] 限流计数用集中存储（Redis）而非本地内存
- [ ] tier 变更立即广播缓存失效
- [ ] 监控：限流命中率、被限流租户排行

### 安全合规层
- [ ] 敏感字段加密用信封加密 + 每租户 KEK
- [ ] DEK 在内存使用后立即清零
- [ ] 删除租户时 KMS scheduleKeyDeletion 而非立即删
- [ ] 审计日志记录所有跨租户访问（impersonate 操作）
- [ ] GDPR / SOC2 / HIPAA 等合规要求提前评估

### 计费层
- [ ] 每条计费事件带唯一 eventId 作为幂等 key
- [ ] 本地持久化队列兜底（Chronicle / 文件 / 本地 DB）
- [ ] Consumer 端 Redis SETNX + DB 唯一索引双层去重
- [ ] 每日对账：网关日志 vs 计费 DB，差异 > 1% 告警

### 迁移与运维层
- [ ] 大租户拆 DB 走"双写 + 灰度 + 回退"，不一次性切
- [ ] Schema 变更（DDL）有按租户灰度的能力
- [ ] 备份：共享 DB 整库 + 大租户额外 per-tenant 增量
- [ ] 演练：删除任意一个租户的全量数据可在 10 分钟内完成

### 监控层
- [ ] Metrics 全部带 tenant 标签
- [ ] Grafana 有租户级 Dashboard，能选 tenant 看其全链路指标
- [ ] 告警规则按 tier 区分：Enterprise 任何异常 → 电话；Free → 邮件聚合

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### 单元化架构与多租户
- 大型 SaaS 演化到全球多 region 时，每个 region 是一个"单元"，租户绑定到某个单元
- 单元内进一步用多租户隔离，单元间用 GSLB 路由
- 案例：Salesforce 的 Pod 模型；Shopify 的 Pod；阿里云的 Region + AZ 双层
- 关键问题：跨单元的租户迁移、单元故障时的自动 failover

### 多租户在云原生（K8s + Mesh）下的工程化
- vCluster / Kamaji：K8s 上跑 K8s 的"虚拟集群"，给每个 Enterprise 租户一个独立 control plane
- Capsule：基于 Namespace 的多租户管控
- Kiosk / Loft：商业方案
- 趋势：从 Namespace 多租户走向 Virtual Cluster，安全边界更强

### 数据库 RLS 高级用法
- PostgreSQL RLS 与 partition 联动：partition by tenant_id，RLS 只扫对应 partition
- Citus / Yugabyte 的"分布式 RLS"
- 多 schema 模式下用 search_path 实现"自动租户切换"
- RLS 与连接池：pgbouncer transaction mode 下 GUC 变量的陷阱

### 租户级 SLA 设计
- 错误预算 (Error Budget) 按租户独立计算
- SLA 履约报告自动化生成
- 违约赔偿与监控联动
- 案例：AWS / GCP 的 SLA 文档结构

### SaaS 平台的租户运营
- 租户生命周期：试用 → 付费 → 增购 → 流失 → 召回
- 数据驱动的 tier 升级建议（"你的用量已经 80%，建议升 Pro"）
- 租户级 A/B 测试（按租户灰度新功能）
- 多租户 SaaS 的指标体系：MRR / ARR / Churn / Expansion

### 大租户拆分专属集群
- 何时拆：单租户占总流量 > 30% / 单租户 ARR > $100K / 合规要求
- 怎么拆：双写 → 校验 → 切流 → 反向迁移可行性
- 拆后的运维：版本同步、监控告警合并、应急切回

### 边缘多租户
- CDN 边缘节点按租户隔离配置（如 Cloudflare Workers 的 isolate 模型）
- WebAssembly 沙箱实现租户级代码执行隔离
- IoT 场景下"设备 - 租户"二级隔离

### Multi-Region Multi-Tenant 的复杂度爆炸
- 数据驻留要求（GDPR：欧洲数据不出境）
- 全球唯一标识（tenant_id 跨 region 唯一性）
- 跨 region 灾备：active-active vs active-standby
- 配置同步：tenant config 变更如何 < 1 秒同步到全球

---

## 7. 引用

- AWS SaaS Lens for Well-Architected Framework
- 微软 [SaaS Tenancy Patterns](https://learn.microsoft.com/azure/azure-sql/database/saas-tenancy-app-design-patterns)
- PostgreSQL [Row Security Policies](https://www.postgresql.org/docs/16/ddl-rowsecurity.html)
- Salesforce [Multi-Tenant Architecture White Paper](https://developer.salesforce.com/page/Multi_Tenant_Architecture)
- Snowflake [Multi-Cluster Shared Data](https://www.snowflake.com/en/data-cloud/architecture/)
- 阿里巴巴 [TransmittableThreadLocal](https://github.com/alibaba/transmittable-thread-local)
- Apache [ShardingSphere 文档](https://shardingsphere.apache.org/document/current/cn/overview/)
- Sentinel [集群限流](https://sentinelguard.io/zh-cn/docs/cluster-flow-control.html)
- Istio [Multi-tenancy](https://istio.io/latest/docs/ops/deployment/deployment-models/#tenancy-models)
- vCluster [Multi-tenancy with Virtual Clusters](https://www.vcluster.com/)
- Tigera [Calico Network Policies for Multi-tenancy](https://www.tigera.io/learn/guides/kubernetes-networking/kubernetes-multi-tenancy/)
- Google [SRE Workbook: Implementing SLOs](https://sre.google/workbook/implementing-slos/) — 多租户 SLA 设计参考

---

> 配套：[DDD 战术战略](/article/architecture/ddd) | [高可用架构演进](/article/system-design/high-availability) | [Spring Security 6](/article/security/spring-security)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
