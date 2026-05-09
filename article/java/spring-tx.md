# Spring 事务传播机制深入 —— 7 种传播 + 真实数据库连接的物理绑定

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，看 [Spring 速记](/article/java/spring) 那一节就够了。
> 这里要回答的，是**写代码遇到 `@Transactional` 没生效、或者 DB 连接池莫名耗尽**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. `@Transactional` 默认的传播行为是 `REQUIRED`。这 7 种传播（`REQUIRED` / `REQUIRES_NEW` / `NESTED` / `SUPPORTS` / `NOT_SUPPORTED` / `NEVER` / `MANDATORY`）里，哪几种**会真的开新物理事务**？哪几种只是"逻辑层加层"？
2. `NESTED` 在底层用什么实现？它和 `REQUIRES_NEW` 的核心区别是什么？为什么有些数据库不支持它？
3. `@Transactional` 默认只对 `RuntimeException` 和 `Error` 回滚，**checked exception 不回滚**。Spring 为什么这么设计？怎么覆盖？
4. 一个方法体内 `try { someService.doX(); } catch (Exception e) { log.error(...); }` —— 如果 `doX()` 抛了运行时异常被吃了，这个事务还会回滚吗？答案分两种情况，你能说清楚吗？
5. `@Transactional` 自调用为什么会失效？除了"提取到另一个 Bean"之外还有几种修法？
6. 一次 `@Transactional` 方法在执行期间，到底**占用了几条数据库连接**？跨多个嵌套传播时呢？
7. 多数据源（比如订单库 + 风控库）下用 `@Transactional` 会发生什么？`ChainedTransactionManager` 解决了什么问题、又**没**解决什么问题？
8. `@Transactional(timeout = 5)` 配的是"事务的超时"，但它**实际是怎么生效**的？是不是每条 SQL 都会被卡 5 秒？

如果有半数答不上，往下看。

---

## 1. 底层模型：抽象、传播、连接绑定

绝大多数八股文章把 Spring 事务讲成"7 种传播 + 默认 REQUIRED + 自调用失效"，然后就没了。这层认知不够诊断生产问题。要看懂上面 8 个问题，必须下沉到 Spring 事务的三个核心抽象、传播的状态机、以及**事务和数据库连接的物理绑定关系**。

### 1.1 三个核心抽象

Spring 事务的整个体系建立在三个接口上：

| 接口 | 角色 | 关键方法 |
| --- | --- | --- |
| **`PlatformTransactionManager`** | 事务管理器（动作者） | `getTransaction(def)` / `commit(status)` / `rollback(status)` |
| **`TransactionDefinition`** | 事务定义（**静态描述**） | `getPropagationBehavior()` / `getIsolationLevel()` / `getTimeout()` / `isReadOnly()` |
| **`TransactionStatus`** | 事务状态（**运行时句柄**） | `isNewTransaction()` / `hasSavepoint()` / `setRollbackOnly()` |

这三个接口的关系：

```
@Transactional 注解
       │ 解析
       ▼
TransactionDefinition (一个不可变的"事务诉求")
       │ 传给
       ▼
PlatformTransactionManager.getTransaction(def)
       │ 返回
       ▼
TransactionStatus (这次执行的"句柄"，commit/rollback 都拿它)
```

注意：**一个 `@Transactional` 方法不一定对应"一个新事务"**。`getTransaction` 内部根据传播行为决定：开新的、复用外层的、挂起外层的、还是抛异常。`TransactionStatus.isNewTransaction()` 用来回答这件事。

### 1.2 7 种传播的语义

下面这张表是核心，背都要背下来：

| 传播 | 外层有事务 | 外层无事务 | 物理事务数 |
| --- | --- | --- | --- |
| **`REQUIRED`**（默认） | 加入外层 | 自己开新的 | 1 |
| **`REQUIRES_NEW`** | **挂起外层**，开新的 | 自己开新的 | 2 |
| **`NESTED`** | 在外层里建 SAVEPOINT | 自己开新的 | 1（带 SP） |
| **`SUPPORTS`** | 加入外层 | 不开事务 | 0 或 1 |
| **`NOT_SUPPORTED`** | **挂起外层**，无事务执行 | 不开事务 | 0 |
| **`NEVER`** | **抛异常** | 不开事务 | 0 |
| **`MANDATORY`** | 加入外层 | **抛异常** | 1 或异常 |

"挂起外层"具体是什么意思？是 Spring 会把外层事务持有的 `Connection` 暂时从 `ThreadLocal` 里摘下来、压入栈，本次执行完再还回去。**新事务用一条新的物理 Connection**。

### 1.3 传播的状态图

光看表不直观，画一张状态图。横轴是"外层是否在事务中"，纵轴是 7 种传播的反应：

```
外层在事务 ── REQUIRED ─────► 加入外层（共享 Connection、共享 ReadView）
外层在事务 ── REQUIRES_NEW ──► 挂起外层 → 新 Connection → 新事务
外层在事务 ── NESTED ────────► 同一 Connection，建 SAVEPOINT
外层在事务 ── SUPPORTS ─────► 加入外层
外层在事务 ── NOT_SUPPORTED ─► 挂起外层 → 无事务执行
外层在事务 ── NEVER ────────► 抛 IllegalTransactionStateException
外层在事务 ── MANDATORY ────► 加入外层

外层无事务 ── REQUIRED ─────► 自己开（→ 新 Connection、新事务）
外层无事务 ── REQUIRES_NEW ──► 自己开
外层无事务 ── NESTED ────────► 自己开（注意：未必带 SP）
外层无事务 ── SUPPORTS ─────► 不开事务（getConnection 是按需的）
外层无事务 ── NOT_SUPPORTED ─► 不开事务
外层无事务 ── NEVER ────────► 不开事务
外层无事务 ── MANDATORY ────► 抛 IllegalTransactionStateException
```

**反直觉点**：`NESTED` 在"外层无事务"时表现和 `REQUIRED` 完全一样，**不会自己造一个 SAVEPOINT**。SAVEPOINT 只在外层已有事务时才有意义。

### 1.4 ConnectionHolder：事务和连接的物理绑定

这是 Spring 事务最容易被忽略、却最影响生产的细节。

`DataSourceTransactionManager.doBegin` 里的关键动作：

```
1. dataSource.getConnection()  ── 从 HikariCP 拿一条物理连接
2. con.setAutoCommit(false)    ── 关闭自动提交
3. new ConnectionHolder(con)   ── 包成 Holder
4. TransactionSynchronizationManager.bindResource(dataSource, holder)
   └─► 实际是 ThreadLocal<Map<DataSource, ConnectionHolder>>
```

第 4 步是核心。**当前线程所有后续的 JDBC 操作（无论是 JdbcTemplate、MyBatis 还是 JPA）都会从这个 ThreadLocal 拿同一条 Connection**。这就是"同一事务里所有 SQL 走同一连接"的物理基础。

后果是直接的：**事务持续多久，这条物理连接就被占用多久**。事务里调一次 RPC（500ms）、再写一条 SQL，那条连接就被占了至少 500ms —— 即使它在 RPC 期间什么都没做。这是后面案例 ⑥ 的根。

```
事务方法开始
   │
   ├── 取 Connection（连接池 -1）
   ├── setAutoCommit(false)
   ├── 绑定到 ThreadLocal
   │
   ├── 业务代码：
   │     SQL1（用绑定的 Connection）
   │     RPC 调用 200ms（连接闲置但占用）
   │     SQL2（用同一条 Connection）
   │
   ├── commit()
   ├── 解绑 ThreadLocal
   └── Connection 还回连接池（连接池 +1）
```

### 1.5 SAVEPOINT 与 NESTED

SAVEPOINT 是 SQL 标准里的"事务内回滚点"。`NESTED` 在底层就是用 SAVEPOINT 实现的：

```sql
-- 外层事务
BEGIN;
INSERT INTO orders ...

-- NESTED 内层"事务"开始
SAVEPOINT spA;
INSERT INTO order_items ...

-- 内层抛异常 → 回滚到 SAVEPOINT
ROLLBACK TO SAVEPOINT spA;

-- 外层继续
INSERT INTO order_logs ...
COMMIT;
```

关键认知：
- **整个流程只有一条物理 Connection、一个事务**
- 内层"回滚"只回滚到 SAVEPOINT，外层之前的 INSERT 还在
- 如果外层最终回滚，**所有东西（包括内层"已提交"的）都没了**

这是 `NESTED` 与 `REQUIRES_NEW` 的本质区别：

| 维度 | NESTED | REQUIRES_NEW |
| --- | --- | --- |
| 物理事务数 | 1 | 2 |
| 物理连接数 | 1 | 2 |
| 内层失败 | 回滚到 SP，外层可继续 | 内层独立回滚，外层可继续 |
| **外层失败** | **内层一起回滚** | **内层不回滚（已独立提交）** |
| 数据库支持 | 需要支持 SAVEPOINT（MySQL/PG/Oracle 都支持，部分老 DB 不支持） | 任意 |

后面案例 ④ 会演示 `REQUIRES_NEW` 内外层互不影响的具体表现。

### 1.6 逻辑事务 vs 物理事务

Spring 事务体系里，这两个概念必须分清：

- **物理事务**：DB 层面的 `BEGIN ... COMMIT`，对应一条 Connection
- **逻辑事务**：Spring 框架层面的"一次 `getTransaction` 调用"，可能是新的、也可能是加入外层的

`REQUIRED` 在嵌套调用里只有 1 个物理事务、N 个逻辑事务。Spring 用 `TransactionStatus.isNewTransaction()` 区分两者：

```java
TransactionStatus status = txManager.getTransaction(def);
if (status.isNewTransaction()) {
    // 这是物理新事务，commit() 会真的发 COMMIT
} else {
    // 这是加入了外层逻辑事务，commit() 只是"释放本层"，不发 COMMIT
}
```

**这个区分解释了一个特别绕的现象**：内层方法 `setRollbackOnly()` 后，自己 `commit()` 时不报错，但外层 `commit()` 时会抛 `UnexpectedRollbackException`。因为内层是逻辑事务，它"投票回滚"；外层是物理事务，它去提交时发现"被投票回滚了"，于是失败。后面案例 ④ 会精讲。

> **口诀**：Spring 事务 = 一次"逻辑诉求"（@Transactional）+ 一次"传播决策"（PropagationBehavior）+ 一条"物理连接"（ConnectionHolder）。三件事任何一件出问题，事务就跟你想的不一样。

---

## 2. 源码切片：基于 Spring 6.x

> 基于 Spring Framework 6.1 的 `spring-tx` 模块。注释为本文添加。

### 2.1 TransactionInterceptor —— 拦截入口

`@Transactional` 是个"声明"，真正干活的是 `TransactionInterceptor`（一个 AOP MethodInterceptor）。它的核心方法 `invokeWithinTransaction` 简化后是：

```java
// org.springframework.transaction.interceptor.TransactionAspectSupport
protected Object invokeWithinTransaction(
        Method method, Class<?> targetClass,
        InvocationCallback invocation) throws Throwable {

    TransactionAttribute txAttr = getTransactionAttributeSource()
            .getTransactionAttribute(method, targetClass);
    PlatformTransactionManager tm = determineTransactionManager(txAttr);
    String joinpointId = methodIdentification(method, targetClass, txAttr);

    // ① 开启事务（根据 propagation 决策）
    TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointId);
    Object retVal;
    try {
        retVal = invocation.proceedWithInvocation();    // ② 执行业务方法
    } catch (Throwable ex) {
        // ③ 异常分类：是否需要回滚
        completeTransactionAfterThrowing(txInfo, ex);
        throw ex;
    } finally {
        cleanupTransactionInfo(txInfo);                 // ④ ThreadLocal 清理
    }
    commitTransactionAfterReturning(txInfo);            // ⑤ 提交
    return retVal;
}
```

**逐段点透**：

- **① 的 `createTransactionIfNecessary`**：注意"if necessary" —— 不是每次都开新事务。`SUPPORTS` 在外层无事务时根本不开。
- **③ 的 `completeTransactionAfterThrowing` 是默认行为陷阱的根源**：它内部判断 `txAttr.rollbackOn(ex)`，只有命中才回滚。默认实现是 `DefaultTransactionAttribute.rollbackOn`：仅 `ex instanceof RuntimeException || ex instanceof Error`。**checked exception 默认不回滚**，commit 照常走。
- **⑤ 注意位置**：`commit` 在 try-finally **之外**。如果 commit 自己抛异常（比如外层是 RR、底层 DB 死锁），这个异常会从这里冒上去。

### 2.2 异常分类的核心：`rollbackOn`

```java
// org.springframework.transaction.interceptor.DefaultTransactionAttribute
@Override
public boolean rollbackOn(Throwable ex) {
    return (ex instanceof RuntimeException || ex instanceof Error);
}
```

而 `RuleBasedTransactionAttribute`（带 `rollbackFor` / `noRollbackFor` 配置时用）的实现是：

```java
@Override
public boolean rollbackOn(Throwable ex) {
    RollbackRuleAttribute winner = null;
    int deepest = Integer.MAX_VALUE;

    if (this.rollbackRules != null) {
        for (RollbackRuleAttribute rule : this.rollbackRules) {
            int depth = rule.getDepth(ex);
            if (depth >= 0 && depth < deepest) {
                deepest = depth;
                winner = rule;
            }
        }
    }
    if (winner == null) {
        return super.rollbackOn(ex);   // ★ 退回到 RuntimeException/Error 判定
    }
    return !(winner instanceof NoRollbackRuleAttribute);
}
```

**逐段点透**：

- 如果有自定义规则（`@Transactional(rollbackFor = SomeException.class)`），找匹配深度最浅的规则
- 没匹配时回退到默认（仅 RuntimeException / Error 回滚）
- **想让 checked exception 也回滚**：`@Transactional(rollbackFor = Exception.class)` —— 这是生产里几乎应该默认开的

### 2.3 AbstractPlatformTransactionManager.getTransaction —— 传播分支

整个 7 种传播的决策逻辑都在这一个方法里：

```java
// org.springframework.transaction.support.AbstractPlatformTransactionManager
public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
        throws TransactionException {

    TransactionDefinition def = (definition != null ? definition : TransactionDefinition.withDefaults());
    Object transaction = doGetTransaction();    // 子类返回当前 ThreadLocal 里的事务对象

    if (isExistingTransaction(transaction)) {
        // 外层已有事务 → 走传播分支
        return handleExistingTransaction(def, transaction, debugEnabled);
    }

    // 外层无事务
    if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
        throw new IllegalTransactionStateException(
            "No existing transaction found for transaction marked with propagation 'mandatory'");
    } else if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
               def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
               def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        SuspendedResourcesHolder suspendedResources = suspend(null);
        return startTransaction(def, transaction, false, debugEnabled, suspendedResources);
    } else {
        // SUPPORTS / NOT_SUPPORTED / NEVER：返回一个"空事务"
        return prepareTransactionStatus(def, null, true, false, debugEnabled, null);
    }
}
```

**逐段点透**：

- **`isExistingTransaction(transaction)`**：子类实现。`DataSourceTransactionManager` 的版本是看 `ConnectionHolder` 是否存在并且 `isTransactionActive()`。
- **"外层无事务" 时 NESTED 走 `startTransaction`**：这就是为什么 `NESTED` 在外层无事务时**和 `REQUIRED` 等价** —— 没有外层就建不出 SAVEPOINT。
- **`suspend(null)`**：注意外层无事务时也调了 suspend，传 null —— 实际作用是把当前线程上**其他 DataSource** 的事务挂起（比如多数据源场景）。

### 2.4 handleExistingTransaction —— 外层有事务时的核心分支

```java
private TransactionStatus handleExistingTransaction(
        TransactionDefinition def, Object transaction, boolean debug) {

    if (def.getPropagationBehavior() == PROPAGATION_NEVER) {
        throw new IllegalTransactionStateException(...);
    }

    if (def.getPropagationBehavior() == PROPAGATION_NOT_SUPPORTED) {
        SuspendedResourcesHolder suspended = suspend(transaction);   // ★ 挂起外层
        return prepareTransactionStatus(def, null, false, false, debug, suspended);
    }

    if (def.getPropagationBehavior() == PROPAGATION_REQUIRES_NEW) {
        SuspendedResourcesHolder suspended = suspend(transaction);   // ★ 挂起外层
        try {
            return startTransaction(def, transaction, false, debug, suspended);
        } catch (...) {
            resumeAfterBeginException(transaction, suspended, ...);
            throw ...;
        }
    }

    if (def.getPropagationBehavior() == PROPAGATION_NESTED) {
        if (!isNestedTransactionAllowed()) {
            throw new NestedTransactionNotSupportedException(...);
        }
        if (useSavepointForNestedTransaction()) {
            // ★ 走 SAVEPOINT 路径
            DefaultTransactionStatus status = prepareTransactionStatus(
                def, transaction, false, false, debug, null);
            status.createAndHoldSavepoint();
            return status;
        } else {
            // 极少数 TM 不用 SAVEPOINT（如 JTA）→ 退化成嵌套真事务
            return startTransaction(def, transaction, true, debug, null);
        }
    }

    // 默认：REQUIRED / SUPPORTS / MANDATORY → 加入外层
    return prepareTransactionStatus(def, transaction, false, false, debug, null);
}
```

**逐段点透**：

- **`suspend(transaction)`**：把当前线程上**所有事务相关的 ThreadLocal**（ConnectionHolder、同步回调等）解绑，封进一个 `SuspendedResourcesHolder`，**当前事务执行完后再 `resume`**。这是"挂起外层"的物理动作。
- **`useSavepointForNestedTransaction()`**：`AbstractPlatformTransactionManager` 默认 true，`JtaTransactionManager` 返回 false。所以 JTA 里的 `NESTED` 会变成"开新真事务"。
- **加入外层时返回的 status，`isNewTransaction()` 是 false**：commit 时不会真的发 COMMIT。

### 2.5 DataSourceTransactionManager.doBegin —— 物理开启

```java
// org.springframework.jdbc.datasource.DataSourceTransactionManager
@Override
protected void doBegin(Object transaction, TransactionDefinition definition) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
    Connection con = null;
    try {
        if (!txObject.hasConnectionHolder() ||
            txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
            Connection newCon = obtainDataSource().getConnection();    // ① 取连接
            txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
        }

        txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
        con = txObject.getConnectionHolder().getConnection();

        Integer previousIsolationLevel = DataSourceUtils
            .prepareConnectionForTransaction(con, definition);          // ② 设隔离级别 / 只读
        txObject.setPreviousIsolationLevel(previousIsolationLevel);
        txObject.setReadOnly(definition.isReadOnly());

        if (con.getAutoCommit()) {
            txObject.setMustRestoreAutoCommit(true);
            con.setAutoCommit(false);                                   // ③ 关自动提交
        }

        prepareTransactionalConnection(con, definition);
        txObject.getConnectionHolder().setTransactionActive(true);

        int timeout = determineTimeout(definition);
        if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
            txObject.getConnectionHolder().setTimeoutInSeconds(timeout); // ④ 超时
        }

        if (txObject.isNewConnectionHolder()) {
            TransactionSynchronizationManager.bindResource(             // ⑤ 绑 ThreadLocal
                obtainDataSource(), txObject.getConnectionHolder());
        }
    } catch (Throwable ex) {
        if (txObject.isNewConnectionHolder()) {
            DataSourceUtils.releaseConnection(con, obtainDataSource());
            txObject.setConnectionHolder(null, false);
        }
        throw new CannotCreateTransactionException(...);
    }
}
```

**逐段点透**：

- **① `obtainDataSource().getConnection()`**：这一行就是**连接池 -1** 的瞬间。事务开启 = 连接被借走。
- **③ `con.setAutoCommit(false)`**：JDBC 默认每条 SQL 自动提交。Spring 把它关掉，让多条 SQL 在一个事务里。
- **④ 超时不是给整个 SQL 用的**：它存在 ConnectionHolder 上。后续每次执行 SQL（`PreparedStatementCreator.createPreparedStatement`）会调 `applyTransactionTimeout`，给 `Statement` 设 `setQueryTimeout(剩余秒数)`。**所以超时是"给每条 SQL 单独设的剩余时间"**，不是给整个事务的硬上限。这是案例 ⑨ 的根。
- **⑤ `bindResource`**：把 ConnectionHolder 绑定到 ThreadLocal，键是 DataSource。同一线程多个 DataSource 可以并存。

### 2.6 TransactionSynchronizationManager —— 五个 ThreadLocal

```java
// org.springframework.transaction.support.TransactionSynchronizationManager
public abstract class TransactionSynchronizationManager {

    private static final ThreadLocal<Map<Object, Object>> resources =
        new NamedThreadLocal<>("Transactional resources");
        // ↑ DataSource → ConnectionHolder 的映射

    private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations =
        new NamedThreadLocal<>("Transaction synchronizations");
        // ↑ 事务同步回调（commit/rollback 后做的事）

    private static final ThreadLocal<String> currentTransactionName =
        new NamedThreadLocal<>("Current transaction name");

    private static final ThreadLocal<Boolean> currentTransactionReadOnly =
        new NamedThreadLocal<>("Current transaction read-only status");

    private static final ThreadLocal<Integer> currentTransactionIsolationLevel =
        new NamedThreadLocal<>("Current transaction isolation level");

    private static final ThreadLocal<Boolean> actualTransactionActive =
        new NamedThreadLocal<>("Actual transaction active");
}
```

**逐段点透**：

- 全部基于 `ThreadLocal` —— 这是为什么 `@Transactional` **跨线程不传播**（异步任务、`@Async`、CompletableFuture 都不会带事务上下文）。
- `synchronizations` 是钩子集合：`TransactionSynchronization.afterCommit()` / `afterCompletion()` 都注册在这里。**这是事务里发 MQ 消息的正确姿势**（见案例 ⑤）。
- 注意 **JPA / MyBatis 都从这里取 Connection**：`SqlSessionUtils.getSqlSession` → `DataSourceUtils.getConnection` → 看 `resources` 里有没有 → 有就复用。这就是"事务一致性"的物理基础。

### 2.7 SAVEPOINT 创建：DefaultTransactionStatus.createAndHoldSavepoint

```java
// org.springframework.transaction.support.AbstractTransactionStatus
public void createAndHoldSavepoint() throws TransactionException {
    setSavepoint(getSavepointManager().createSavepoint());
}

@Override
protected SavepointManager getSavepointManager() {
    Object resource = doGetSavepointManager();
    if (!(resource instanceof SavepointManager spm)) {
        throw new NestedTransactionNotSupportedException(...);
    }
    return spm;
}
```

而 `ConnectionHolder` 实现了 `SavepointManager`：

```java
// org.springframework.jdbc.datasource.ConnectionHolder
@Override
public Object createSavepoint() throws SQLException {
    this.savepointCounter++;
    return getConnection().setSavepoint(SAVEPOINT_NAME_PREFIX + this.savepointCounter);
                          // ↑ JDBC 标准 API：发 SAVEPOINT 到 DB
}

@Override
public void rollbackToSavepoint(Object savepoint) throws SQLException {
    getConnection().rollback((Savepoint) savepoint);
}

@Override
public void releaseSavepoint(Object savepoint) throws SQLException {
    getConnection().releaseSavepoint((Savepoint) savepoint);
}
```

**逐段点透**：

- **`con.setSavepoint(name)` 是 JDBC 标准 API**：底层发 `SAVEPOINT spA` 给 DB。所以 `NESTED` 能不能用，取决于 JDBC 驱动 + DB 是否支持。MySQL InnoDB 完全支持，但有些 NoSQL JDBC 网关不支持。
- **`releaseSavepoint`**：内层"提交"时调，告诉 DB "这个回滚点不要了"。MySQL 实际上不需要显式 release，但 PostgreSQL 需要释放占用资源。
- 所有动作都在**同一条 Connection** 上，所以也在**同一个事务**里。

> **口诀**：拦截器拿 `TransactionAttribute`、根据 propagation 走分支、底层是 JDBC 的 `setAutoCommit(false)` + `commit()` / `rollback()` / `setSavepoint()` 三件套。Spring 事务再"声明式"，到 DB 那一层永远是这些原始 API。

---

## 3. 实测数据：传播 vs 连接占用

口诀化的"7 种传播"看完不算懂，得看真实环境下它们对**连接池占用**的差异。下面用一组测试，结论给在前面。

### 3.1 结论先行

| 传播组合 | 一个外层调用占用连接数 | 适用场景 |
| --- | --- | --- |
| 外层 REQUIRED → 内层 REQUIRED | **1** | 默认场景，子方法逻辑同事务 |
| 外层 REQUIRED → 内层 REQUIRES_NEW | **2** | 内层独立提交（如审计日志） |
| 外层 REQUIRED → 内层 NESTED | **1** | 局部失败不影响外层（如批量处理) |
| 外层 REQUIRED → 内层 NOT_SUPPORTED | **1**（外层）+ 内层无连接 | 内层是只读、不需要事务的查询 |
| 三层嵌套全 REQUIRES_NEW | **3** | **危险**，连接池容易耗尽 |

**反直觉点**：`REQUIRES_NEW` 嵌套时，外层连接是**被挂起**而不是"释放"。也就是说一次嵌套 REQUIRES_NEW 调用，**真的会同时占用两条物理连接**。如果连接池只有 10 条，5 个并发 + 嵌套 REQUIRES_NEW 就把池占满。

### 3.2 测试代码

```java
@SpringBootTest
public class PropagationConnectionTest {

    @Autowired DataSource dataSource;
    @Autowired OuterService outer;

    @Test
    void measureRequiresNewConnections() {
        HikariPoolMXBean pool = ((HikariDataSource) dataSource).getHikariPoolMXBean();

        int before = pool.getActiveConnections();
        // 内部 outer.callRequiresNew() 的实现：
        //   @Transactional 调用 inner.doRequiresNew()
        //   inner.doRequiresNew() 里 sleep 2s
        //   外层在 sleep 期间观察 pool.getActiveConnections()
        outer.callRequiresNewWithObservation((duringInner) -> {
            int active = pool.getActiveConnections();
            System.out.println("inner running, active = " + active);
            // 期望：外层 1 + 内层 1 = 2
        });
    }
}

@Service
public class OuterService {
    @Autowired InnerService inner;
    @Autowired JdbcTemplate jdbc;

    @Transactional
    public void callRequiresNewWithObservation(Consumer<Void> probe) {
        jdbc.update("INSERT INTO outer_log (msg) VALUES (?)", "outer");
        inner.doRequiresNew(probe);   // 内层 REQUIRES_NEW
    }
}

@Service
public class InnerService {
    @Autowired JdbcTemplate jdbc;

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void doRequiresNew(Consumer<Void> probe) {
        jdbc.update("INSERT INTO inner_log (msg) VALUES (?)", "inner");
        probe.accept(null);   // 在内层事务期间观察连接数
    }
}
```

### 3.3 在 HikariCP（pool size = 10）上的典型结果

| 嵌套层级 | 期望占用 | 实测占用 | 备注 |
| --- | --- | --- | --- |
| 单层 REQUIRED | 1 | 1 | 基线 |
| 双层 REQUIRED | 1 | 1 | 复用外层连接 |
| 双层 REQUIRES_NEW | 2 | 2 | 外层挂起，新连接 |
| 三层 REQUIRES_NEW | 3 | 3 | 累积占用 |
| 双层 NESTED | 1 | 1 | 同 Connection，SAVEPOINT |

**反直觉发现**：`REQUIRES_NEW` 嵌套深度等于占用连接数。生产里如果有"链路 A → B → C，每层都标了 `REQUIRES_NEW`"，**单次调用占用 3 条连接**。在 200 QPS 下，200 × 3 = 600 条连接需求；如果你的池只有 50，必然耗尽。

### 3.4 长事务对连接池的压力

模拟"事务里调 RPC"：

```java
@Transactional
public void buggyOrder(Order order) {
    jdbc.update("INSERT INTO orders ...", order);   // 1ms
    riskClient.check(order);                         // ★ RPC 500ms
    jdbc.update("UPDATE inventory ...", order);     // 1ms
}
```

事务持有连接 = 502ms。在 200 QPS 下：

```
平均连接占用 = 200 × 0.502 = 100.4 条
峰值占用 ≈ 130~150（按 P95 算）
```

**HikariCP 默认 maximumPoolSize = 10**。100+ 条需求 → 大量请求等待 → 连接池耗尽 → `Connection is not available, request timed out`。

**修复**：把 `riskClient.check` 挪到事务外。事务持有时间从 502ms → 2ms，连接需求从 100 → 0.4。

### 3.5 火焰图能告诉你什么

用 [async-profiler](https://github.com/async-profiler/async-profiler) 抓一下：

```bash
./profiler.sh -d 60 -e wall -f profile.html <pid>
```

典型问题信号：
- **`HikariPool.getConnection` 占栈很高** → 连接池经常没空闲，业务在等连接
- **`DataSourceTransactionManager.doSuspend` 频繁出现** → 大量 `REQUIRES_NEW` 嵌套
- **`Connection.commit` 占比异常** → 事务太多太碎（典型场景：循环里每次都开事务）

> **口诀**：传播看似抽象，落地全是"连接占用 + 时间"。事务持有时间 × QPS = 连接需求；嵌套 `REQUIRES_NEW` × 层数 = 单次占用倍数。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① @Transactional 自调用 —— "事务声明了，根本没生效"

#### 现象

代码上线后发现某个本应回滚的场景没回滚。手工排查事务方法上确实有 `@Transactional`，但日志里看不到 BEGIN / ROLLBACK。

```java
@Service
public class OrderService {

    public void createOrderFlow(Order order) {
        validate(order);
        this.saveOrder(order);   // ★ 内部调用，事务"消失"
    }

    @Transactional
    public void saveOrder(Order order) {
        orderMapper.insert(order);
        if (order.getAmount() > 1000) {
            throw new RuntimeException("amount too large");
        }
    }
}
```

`createOrderFlow("amount=2000 的订单")` 被调，**抛了异常但订单已经写入了 DB**。

#### 根因

`@Transactional` 是 AOP 实现的。Spring 在容器里给 `OrderService` 创建了一个**代理对象**（CGLIB 或 JDK 动态代理），代理对象的 `saveOrder` 方法里包了事务逻辑。

但 `this.saveOrder(order)` 是 Java 语法层面的调用，**`this` 指向的是原始对象，不是代理**。所以根本没经过代理的拦截，事务逻辑被绕过了。

```
外部调用：proxy.createOrderFlow()
                │
                ▼
原始对象的方法体：this.saveOrder()  ← this 是原始对象
                │
                ▼
原始对象的 saveOrder：直接执行 SQL，没有事务
```

#### 修复

**方案 A**：把内部方法提取到另一个 Bean。

```java
@Service
public class OrderFacade {
    @Autowired private OrderTxService txService;

    public void createOrderFlow(Order order) {
        validate(order);
        txService.saveOrder(order);   // ★ 跨 Bean，过代理
    }
}

@Service
public class OrderTxService {
    @Transactional
    public void saveOrder(Order order) { ... }
}
```

**方案 B**：注入自己的代理（`@Lazy` 避免循环依赖）。

```java
@Service
public class OrderService {
    @Autowired @Lazy private OrderService self;   // ★ 拿到代理引用

    public void createOrderFlow(Order order) {
        validate(order);
        self.saveOrder(order);
    }

    @Transactional
    public void saveOrder(Order order) { ... }
}
```

**方案 C**：用 `AopContext.currentProxy()`（需要在配置里 `@EnableAspectJAutoProxy(exposeProxy = true)`）。

```java
public void createOrderFlow(Order order) {
    validate(order);
    ((OrderService) AopContext.currentProxy()).saveOrder(order);
}
```

**方案 D**：改用 AspectJ 编译期织入（彻底绕过代理机制）。配 `@EnableTransactionManagement(mode = AdviceMode.ASPECTJ)`。生产里很少用，因为构建复杂。

#### 验证

```java
@Test
void selfInvocationMustBeProxied() {
    Order o = new Order();
    o.setAmount(2000);
    assertThrows(RuntimeException.class, () -> orderService.createOrderFlow(o));

    // ★ 关键断言：DB 里不应该有这条订单
    assertEquals(0, jdbc.queryForObject("SELECT COUNT(*) FROM orders WHERE id = ?",
        Integer.class, o.getId()));
}
```

> **口诀**：`@Transactional` 必须**经过代理**才有效。`this.method()` 永远不过代理。同理：`@Async`、`@Cacheable`、`@Retryable` 全有这个坑。

---

### 案例 ② 抛 checked exception 没回滚 —— "事务方法抛了，钱还是扣了"

#### 现象

转账业务方法：

```java
@Transactional
public void transfer(Long from, Long to, BigDecimal amount) throws BusinessException {
    accountMapper.deduct(from, amount);
    if (!riskCheck(to)) {
        throw new BusinessException("recipient frozen");   // ★ checked exception
    }
    accountMapper.add(to, amount);
}
```

灰度上发现：转账抛 `BusinessException` 时，**from 账户已经被扣了，to 账户没收到**。事务没回滚。

#### 根因

回到 2.2 节看到的代码：

```java
public boolean rollbackOn(Throwable ex) {
    return (ex instanceof RuntimeException || ex instanceof Error);
}
```

**Spring 默认只回滚 `RuntimeException` 和 `Error`，checked exception 不回滚**。这个设计来自 EJB 时代的约定 —— Spring 直接继承了。

为什么这么设计？历史原因：业务异常（checked）被认为是"已知情况"，可能调用方想自己处理；运行时异常被认为是"不可恢复的故障"，应该回滚。但实际生产里这个区分根本对不上业务语义 —— 大部分业务异常**也希望回滚**。

#### 修复

**方案 A**：注解上声明 `rollbackFor`。

```java
@Transactional(rollbackFor = Exception.class)
public void transfer(...) throws BusinessException { ... }
```

**方案 B**：把业务异常改成 RuntimeException。把 `BusinessException extends Exception` 改成 `extends RuntimeException`。我个人推荐这个 —— 业务异常本就不该是 checked。

**方案 C**：项目级默认。自己实现一个 `TransactionAttributeSource` 或者直接用注解元注解：

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Transactional(rollbackFor = Exception.class)
public @interface BusinessTransactional {}
```

然后业务里全用 `@BusinessTransactional`。

#### 验证

```java
@Test
void checkedExceptionMustRollback() {
    BigDecimal initial = accountMapper.getBalance(FROM_ID);
    assertThrows(BusinessException.class,
        () -> transferService.transfer(FROM_ID, TO_ID, BigDecimal.TEN));

    BigDecimal after = accountMapper.getBalance(FROM_ID);
    assertEquals(initial, after);   // ★ 余额必须没变
}
```

> **口诀**：默认行为陷阱。生产建议**所有 `@Transactional` 都加 `rollbackFor = Exception.class`**，或用元注解封装。

---

### 案例 ③ try-catch 把异常吞了 —— "事务方法没抛错，DB 数据却回滚了"

#### 现象

```java
@Service
public class A {
    @Autowired B b;

    @Transactional
    public void outer() {
        a.doX();
        try {
            b.inner();   // 内层抛异常
        } catch (Exception e) {
            log.error("ignore", e);   // 吞了
        }
        a.doY();
    }
}

@Service
public class B {
    @Transactional
    public void inner() {
        bMapper.write();
        throw new RuntimeException("boom");
    }
}
```

外层方法**正常返回**（异常被吞），日志正常打印，但发现外层提交时抛了：

```
org.springframework.transaction.UnexpectedRollbackException:
Transaction rolled back because it has been marked as rollback-only
```

并且 `a.doX` 和 `a.doY` 写的数据都没了。

#### 根因

回想 1.6 节"逻辑事务 vs 物理事务"。`inner()` 默认 `REQUIRED` —— 加入了外层事务。

`B.inner()` 抛异常 → `TransactionInterceptor` 捕获 → `rollbackOn(ex)` 命中 → **但因为是逻辑事务（不是物理新事务），不能真的发 ROLLBACK**，只能 `setRollbackOnly()` —— 在外层物理事务上**打个"必须回滚"的标记**。

外层 `outer()` 方法体捕获并吞掉了异常，自己看起来"正常"。然后到 commit 的时候，Spring 看到外层物理事务的 rollback-only 标记 → 不能 commit，必须 rollback → 同时**抛 `UnexpectedRollbackException` 通知调用方**。

```
inner 抛异常 → 内层逻辑事务 setRollbackOnly()
                       │
                       ▼
                外层物理事务被标记为 rollback-only
                       │
outer try-catch 吞了 inner 的异常 → 继续执行 doY
                       │
                       ▼
outer 方法返回 → TransactionInterceptor 准备 commit
                       │
                       ▼
发现 rollback-only → 实际执行 ROLLBACK + 抛 UnexpectedRollbackException
```

#### 修复

**方案 A**：让 `inner` 用 `REQUIRES_NEW`。这样 inner 抛异常只回滚 inner 自己的事务，不影响外层。

```java
@Service
public class B {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void inner() { ... }
}
```

但这意味着**内层独立物理事务**，多占一条连接。也意味着外层后续如果失败，inner 的写入不会回滚。

**方案 B**：让 inner 用 `NESTED`。SAVEPOINT 路径，同一连接，内层失败回滚到 SAVEPOINT。

```java
@Transactional(propagation = Propagation.NESTED)
public void inner() { ... }
```

外层在 catch 时，需要清理 SAVEPOINT 状态：

```java
try {
    b.inner();
} catch (Exception e) {
    // SAVEPOINT 已自动回滚（Spring 在 completeTransactionAfterThrowing 里处理）
    log.error("inner failed but outer continues", e);
}
```

**方案 C**：明确这是设计错误 —— 如果业务上允许内层失败外层继续，**必须显式标 `REQUIRES_NEW` 或 `NESTED`**。如果不允许，就**不要 try-catch 吞异常**，让它冒上去。

#### 验证

```java
@Test
void innerFailureWithRequiresNewMustNotAffectOuter() {
    a.outer();   // 不应该抛 UnexpectedRollbackException

    assertNotNull(jdbc.queryForObject("SELECT * FROM a_table WHERE id = ?", ...));
    assertNull(jdbc.queryForObject("SELECT * FROM b_table WHERE id = ?", ...));
}
```

> **口诀**：`REQUIRED` 嵌套时**任何一层失败都会拖累整个物理事务**。想隔离 → `REQUIRES_NEW` / `NESTED`，并明白连接占用代价。

---

### 案例 ④ REQUIRES_NEW 内部异常不应回滚外部，却把外部连带回滚了

#### 现象

```java
@Service
public class OrderService {
    @Autowired AuditService audit;

    @Transactional
    public void place(Order o) {
        orderMapper.insert(o);
        try {
            audit.log(o);   // REQUIRES_NEW 写审计日志
        } catch (Exception e) {
            log.warn("audit log failed, ignore", e);
        }
        orderMapper.updateStatus(o, "PAID");
    }
}

@Service
public class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void log(Order o) {
        auditMapper.insert(o);
        throw new RuntimeException("audit DB temporarily down");
    }
}
```

期望：审计失败不影响主流程，订单照样下、状态照样更新。
实际：订单整个不见了。

#### 根因

`REQUIRES_NEW` 在内层抛异常时，确实会回滚**内层事务**（独立物理事务，独立提交/回滚）。

**但**，如果异常冒到外层而没被吃，外层依然会回滚。这个例子里有 `try-catch` 看似吃掉了，但是要看**异常类型** —— 如果内层 `REQUIRES_NEW` 事务回滚时遇到 commit 时出问题（比如内层事务自己 commit 阶段失败），抛的是 `TransactionException` 系列，外层可能误处理。

更常见的实际原因：**把 `REQUIRES_NEW` 写在了内部方法上，但调用是 `this.audit(...)` —— 自调用绕过代理**！于是 propagation 根本没生效，依然是同一个事务的逻辑层加层 → 内层 setRollbackOnly → 外层连带回滚。

```java
// 错误写法（同一 Bean 内自调用）
@Service
public class OrderService {
    @Transactional
    public void place(Order o) {
        ...
        try {
            this.audit(o);   // ★ propagation 设了 REQUIRES_NEW，但 this 调用没过代理
        } catch (Exception e) { ... }
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void audit(Order o) { ... }
}
```

#### 修复

**确保 `REQUIRES_NEW` 方法在另一个 Bean 上**（同案例 ①）。

#### 验证：能在测试里看到差异

```java
@Test
void requiresNewMustIsolateFailure() {
    Order o = new Order();
    orderService.place(o);   // 不应该抛 UnexpectedRollbackException

    Order found = orderMapper.find(o.getId());
    assertNotNull(found);
    assertEquals("PAID", found.getStatus());   // ★ 主流程完成

    assertNull(auditMapper.find(o.getId()));   // ★ 审计回滚
}
```

如果跑出"主流程也没了"，看日志，多半是**自调用 + 传播失效**。

> **口诀**：`REQUIRES_NEW` 必须跨 Bean 才生效。生效后内外**真正独立** —— 内层失败外层不受影响（前提是异常被妥善处理）；外层失败内层不会回滚（已独立提交）。

---

### 案例 ⑤ 长事务卡住 DB 连接池 —— "事务里发 MQ"

#### 现象

```java
@Transactional
public void place(Order o) {
    orderMapper.insert(o);
    mqClient.send("order.created", o);   // ★ 事务里发 MQ
    orderMapper.updateStatus(o, "PAID");
}
```

下午 3 点流量高峰，业务报"获取连接超时"。监控显示 HikariCP `pendingThreads` 持续 > 20，`activeConnections` 满。

#### 根因

事务里**任何阻塞 IO 都会让事务变长，物理连接被占用更久**。

`mqClient.send` 同步发送 MQ：通常 50ms~200ms。事务持有时间从 5ms 拉长到 200ms+。在 200 QPS 下：

```
连接需求 = 200 × 0.205 = 41 条
HikariCP 默认 max = 10 → 残忍
```

更糟：**MQ 发出去了，但事务还没提交**。下游消费者收到消息去查订单，**查不到**（事务未提交，数据对其他事务不可见）—— 这就是经典的"消息和数据库不一致"问题。

#### 修复

**方案 A**：事务里只做 DB 操作，MQ 在事务后发。

```java
@Transactional
public void place(Order o) {
    orderMapper.insert(o);
    orderMapper.updateStatus(o, "PAID");
}

public void placeAndNotify(Order o) {
    place(o);
    mqClient.send("order.created", o);   // ★ 事务外
}
```

但这有可靠性问题：DB 提交了但 MQ 发送失败 → 消息丢。

**方案 B**：用 `TransactionSynchronizationManager` 注册"事务提交后回调"。

```java
@Transactional
public void place(Order o) {
    orderMapper.insert(o);
    orderMapper.updateStatus(o, "PAID");

    TransactionSynchronizationManager.registerSynchronization(
        new TransactionSynchronization() {
            @Override public void afterCommit() {
                mqClient.send("order.created", o);
            }
        });
}
```

Spring 6+ 有更优雅的写法 —— 事件 + `@TransactionalEventListener`：

```java
@Transactional
public void place(Order o) {
    orderMapper.insert(o);
    orderMapper.updateStatus(o, "PAID");
    eventPublisher.publishEvent(new OrderCreatedEvent(o));
}

@Component
public class OrderEventListener {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onOrderCreated(OrderCreatedEvent e) {
        mqClient.send("order.created", e.order());
    }
}
```

`AFTER_COMMIT` 阶段事务已经提交（数据可见），消息发出去下游能查到订单。

**方案 C（强一致需求）**：本地消息表（事务内写消息表，事务外用调度器扫表发 MQ + 重试）。这是分布式事务的另一个话题，参考 [分布式事务](/article/distributed/dist-tx)。

#### 验证

```bash
# 压测前看 HikariCP active
curl http://localhost:8080/actuator/metrics/hikaricp.connections.active

# 修复前：事务里发 MQ → active 接近 max
# 修复后：事务里只做 SQL → active 很少超过 5
```

> **口诀**：事务里**禁止任何阻塞 IO**（RPC / MQ / 大文件读写 / Sleep）。要在事务后做事，用 `@TransactionalEventListener(AFTER_COMMIT)`。

---

### 案例 ⑥ 多数据源 —— "订单写了，风控记录没了"

#### 现象

业务有两个 DataSource：`orderDS` 和 `riskDS`。代码：

```java
@Transactional
public void placeWithRisk(Order o) {
    orderMapper.insert(o);   // 写 orderDS
    riskMapper.insert(...);  // 写 riskDS
    if (someCondition()) {
        throw new RuntimeException("rollback both");
    }
}
```

抛异常后：**orderDS 回滚成功，riskDS 那条记录还在**。

#### 根因

Spring 的 `DataSourceTransactionManager` 是**单数据源**的。一个 `@Transactional` 注解只绑一个事务管理器。即使你的代码同时操作了两个 DataSource，事务管理器只能管它"绑定"的那一个 DataSource。

```java
@Bean
public PlatformTransactionManager txManager(DataSource orderDS) {
    return new DataSourceTransactionManager(orderDS);   // ★ 只管 orderDS
}
```

写 `riskDS` 时，`MyBatis` / `JPA` 走 `DataSourceUtils.getConnection(riskDS)` —— 发现 ThreadLocal 里没有 riskDS 的 ConnectionHolder（只有 orderDS 的）→ **直接从 riskDS 池里拿一条新连接，按 autoCommit=true 模式跑**。

后果是：riskDS 上每条 SQL 都立即提交。事务回滚时只能回滚 orderDS。

#### 修复

**方案 A（推荐）**：业务上避免跨数据源事务。把跨库操作改成"主库事务 + 异步同步到副库"或"本地消息表"。

**方案 B**：`ChainedTransactionManager`（已 deprecated，慎用）。

```java
@Bean
public PlatformTransactionManager txManager(
        @Qualifier("orderTM") PlatformTransactionManager orderTM,
        @Qualifier("riskTM") PlatformTransactionManager riskTM) {
    return new ChainedTransactionManager(orderTM, riskTM);
}
```

它的语义是**"按顺序 begin、按相反顺序 commit"**：

```
begin orderTM
begin riskTM
... 业务 ...
commit riskTM   ← 注意顺序：后开的先提
commit orderTM
```

**关键缺陷**：如果 riskTM commit 成功、orderTM commit 失败，**riskDS 已经提交了，无法回滚**。所以 `ChainedTransactionManager` 只能解决"begin 阶段失败"和"业务执行阶段失败"，**不能解决 commit 阶段的部分失败**。这就是为什么它在 Spring 5.2 后被标 deprecated。

**方案 C**：JTA / XA 分布式事务（Atomikos / Narayana）。两阶段提交，能真正保证跨库一致性，但性能差、运维复杂。生产里很少用。

**方案 D**：Seata / Saga 模式。补偿型事务，把"回滚"变成"反向操作"。适合跨服务、跨库场景。详见 [分布式事务](/article/distributed/dist-tx)。

#### 验证

写一个"故意让两个库都执行写、然后 throw"的测试，期望两边都回滚。

```java
@Test
void crossDataSourceMustBothRollback() {
    assertThrows(RuntimeException.class, () ->
        chainedService.placeWithRisk(new Order()));

    assertEquals(0, jdbc(orderDS).queryForObject("SELECT COUNT(*) ...", Integer.class));
    assertEquals(0, jdbc(riskDS).queryForObject("SELECT COUNT(*) ...", Integer.class));
}
```

如果只挂 `DataSourceTransactionManager(orderDS)`，第二个断言会失败。

> **口诀**：跨数据源**不要指望 `@Transactional` 自己解决**。能避免就避免；非要做就上 Seata 或本地消息表。`ChainedTransactionManager` 解决不了 commit 阶段的部分失败。

---

### 案例 ⑦ NESTED 在某些数据库不支持 —— "上线就抛 NestedTransactionNotSupportedException"

#### 现象

有个批量导入功能用了 `NESTED`，本地 MySQL 跑得好好的。上线到客户环境（用某老牌商业数据库），启动时报：

```
NestedTransactionNotSupportedException:
Transaction manager does not allow nested transactions
```

#### 根因

`AbstractPlatformTransactionManager.handleExistingTransaction` 里：

```java
if (def.getPropagationBehavior() == PROPAGATION_NESTED) {
    if (!isNestedTransactionAllowed()) {
        throw new NestedTransactionNotSupportedException(...);
    }
    ...
}
```

`isNestedTransactionAllowed()` 默认 false，需要显式开启：

```java
@Bean
public DataSourceTransactionManager txManager(DataSource ds) {
    DataSourceTransactionManager tm = new DataSourceTransactionManager(ds);
    tm.setNestedTransactionAllowed(true);
    return tm;
}
```

**等等**，`DataSourceTransactionManager` 的默认其实**是 true**。报错的真实原因是：用的是 `JpaTransactionManager` 或者老版 `HibernateTransactionManager`，它们默认 false 或者不支持。

更深一层：即使 Spring 这边开了，`Connection.setSavepoint(name)` 是 JDBC 调用，最终要 DB 驱动支持。**部分老 DB / NoSQL 网关 / 嵌入式 DB 不支持 SAVEPOINT** —— `JDBC4Connection.setSavepoint` 抛 `SQLFeatureNotSupportedException`。

#### 修复

**第一步**：检查 TM 类型和配置。

```java
DataSourceTransactionManager:  默认 nestedTransactionAllowed = true
JpaTransactionManager:         默认 false（需要 setNestedTransactionAllowed(true)）
JtaTransactionManager:         不用 SAVEPOINT，开新真事务
```

**第二步**：测试 DB 是否支持 SAVEPOINT。

```java
try (Connection con = dataSource.getConnection()) {
    con.setAutoCommit(false);
    Savepoint sp = con.setSavepoint("test");   // 不抛即支持
    con.rollback(sp);
}
```

**第三步**：如果不支持，把 `NESTED` 改成 `REQUIRES_NEW`。语义不完全等价（独立物理事务），但通常业务可接受。

#### 验证

跨环境（MySQL / PG / 客户 DB）跑一组集成测试，每个环境都执行嵌套事务路径。

> **口诀**：`NESTED` 依赖 SAVEPOINT。**MySQL InnoDB / PG / Oracle 都支持，但有些 DB / 中间件不支持**。生产里如果环境不可控，倾向 `REQUIRES_NEW`。

---

### 案例 ⑧ 事务超时配了没生效 —— "@Transactional(timeout=5)，业务跑了 30 秒"

#### 现象

```java
@Transactional(timeout = 5)
public void slowJob() {
    Thread.sleep(30_000);   // 模拟慢业务
    jdbc.update("INSERT ...");   // 30 秒后才执行
}
```

期望 5 秒后超时抛异常。实际 30 秒后 INSERT 成功执行，事务正常 commit，没超时。

#### 根因

回到 2.5 节的 `doBegin`：

```java
int timeout = determineTimeout(definition);
if (timeout != TIMEOUT_DEFAULT) {
    txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
}
```

注意：**Spring 的 timeout 不是给整个事务的硬上限**。它存在 `ConnectionHolder` 上，每次执行 SQL 时**计算"还剩多少秒"，传给 `Statement.setQueryTimeout(剩余秒数)`**。

`DataSourceUtils.applyTransactionTimeout`：

```java
public static void applyTransactionTimeout(Statement stmt, DataSource dataSource) {
    if (stmt != null && dataSource != null) {
        ConnectionHolder holder = (ConnectionHolder)
            TransactionSynchronizationManager.getResource(dataSource);
        if (holder != null && holder.hasTimeout()) {
            stmt.setQueryTimeout(holder.getTimeToLiveInSeconds());
            // ★ 注意：是给"当前 SQL"设的，不是全局
        }
    }
}
```

**所以 `Thread.sleep(30s)` 期间根本没有"超时检查"**，因为这段时间根本没在执行 SQL。事务的"超时"只对**单条 SQL 的执行时间**起作用 —— 准确说是 SQL 的剩余时间不够，会抛 `SQLTimeoutException`。

如果业务是"sleep 30s + 1ms 的 SQL"，sleep 期间不查超时，到 SQL 执行时检查 `getTimeToLiveInSeconds()`：

- 如果剩余时间 <= 0，抛 `TransactionTimedOutException`
- 否则给 SQL 设 `setQueryTimeout(剩余秒数)`

#### 修复

**方案 A**：业务逻辑里主动检查。

```java
@Transactional(timeout = 5)
public void slowJob() {
    long deadline = System.currentTimeMillis() + 5_000;
    for (Item i : items) {
        if (System.currentTimeMillis() > deadline) {
            throw new TransactionTimedOutException("manual timeout");
        }
        process(i);
    }
}
```

**方案 B**：把 RPC / sleep 移出事务（同案例 ⑤）。

**方案 C**：DB 层面用 `MAX_EXECUTION_TIME` hint（MySQL 5.7+）：

```sql
SELECT /*+ MAX_EXECUTION_TIME(5000) */ * FROM big_table WHERE ...
```

#### 验证

```java
@Test
void timeoutDoesNotKillNonSqlSleep() {
    long start = System.currentTimeMillis();
    assertDoesNotThrow(() -> slowService.sleepThenInsert());
    long elapsed = System.currentTimeMillis() - start;
    assertTrue(elapsed > 30_000);  // ★ 验证 timeout 没杀掉 sleep
}

@Test
void timeoutKillsLongSql() {
    assertThrows(TransactionTimedOutException.class,
        () -> slowService.runSlowQuery());   // SQL 本身慢，会被 setQueryTimeout 杀
}
```

> **口诀**：Spring 的事务 timeout 是**给每条 SQL 单独设的剩余时间**，**不是事务总时长上限**。要管业务逻辑的总耗时，自己写 deadline 检查。

---

### 案例 ⑨ 内层 readOnly 不生效 —— "明明标了 readOnly，DB 还是有 BEGIN/COMMIT"

#### 现象

```java
@Transactional   // 默认 readOnly = false
public void update(...) {
    ...
    queryService.find(...);    // 内层 @Transactional(readOnly = true)
    ...
}

@Service
public class QueryService {
    @Transactional(readOnly = true)
    public Item find(Long id) {
        return mapper.select(id);
    }
}
```

期望：内层走只读优化（连接到只读副本、隔离级别可以放宽等）。
实际：内层走的还是写库，并且看不到 readOnly 优化。

#### 根因

`REQUIRED` 默认行为是**加入外层事务**。一旦加入，**外层的所有属性都赢**：isolation、readOnly、timeout —— 内层声明的统统被忽略。

Spring 6.1+ 实际上会日志警告（如果配了 `validateExistingTransaction = true`），但默认是静默继承外层的。

```java
// AbstractPlatformTransactionManager
private boolean validateExistingTransaction = false;

protected void prepareSynchronization(DefaultTransactionStatus status, TransactionDefinition definition) {
    if (status.isNewSynchronization()) {
        TransactionSynchronizationManager.setActualTransactionActive(status.hasTransaction());
        TransactionSynchronizationManager.setCurrentTransactionIsolationLevel(...);
        TransactionSynchronizationManager.setCurrentTransactionReadOnly(definition.isReadOnly());
        // ↑ 注意：这里读的是"内层定义的"，但实际上 Connection 上的 readOnly 是外层 begin 时设的
        ...
    }
}
```

物理 Connection 的 `setReadOnly(true)` 是在 `doBegin` 时调一次，绑死。后续加入的逻辑事务改不了 Connection 的 readOnly 状态。

#### 修复

**方案 A**：内层用 `REQUIRES_NEW`（独立物理事务，独立 Connection，readOnly 真的生效）。代价：多占一条连接。

**方案 B**：业务层显式分离。**只读查询不要嵌套在写事务里**。改造调用关系：

```java
// 错：写事务里嵌套查询
@Transactional
public void update(...) {
    Item i = queryService.find(...);
    ...
}

// 对：先查、再开写事务
public void process(Long id) {
    Item i = queryService.find(id);   // 只读，独立事务
    updateService.doUpdate(i);         // 写，独立事务
}
```

**方案 C**：用 `@Transactional` 的 `value` / `transactionManager` 指定不同的事务管理器（读副本、写主库）。需要配多个 DataSource + TM。

#### 验证

```java
@Test
void readOnlyMustActuallyTakeEffect() {
    // 抓 Connection 的 isReadOnly 状态
    queryService.findWithAssertion(id, conn -> {
        assertTrue(conn.isReadOnly());   // ★ 真的是 read-only
    });
}
```

实际生产里更好的验证：监控读副本的 QPS。如果 readOnly 配置生效，只读查询应该路由到读副本。

> **口诀**：`REQUIRED` 加入外层时**所有属性继承外层**，readOnly / isolation / timeout 全失效。要让属性生效 → `REQUIRES_NEW` 或拆开调用关系。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 注解层
- [ ] 所有 `@Transactional` 都加 `rollbackFor = Exception.class`（或用元注解封装），避免 checked 异常默认不回滚
- [ ] 明确传播行为，不要全用默认 `REQUIRED`；尤其是嵌套场景
- [ ] `@Transactional(readOnly = true)` 用在**真正只读**的方法上，但要注意 `REQUIRED` 嵌套时不生效
- [ ] 不要在 private / final / static 方法上标 `@Transactional`（CGLIB 代理拦截不到）
- [ ] 不要在内部自调用方法上指望事务生效

### 事务体内
- [ ] **事务体内禁止任何阻塞 IO**：RPC、MQ、文件、Sleep、外部系统调用
- [ ] 事务体内不要做复杂业务计算（让事务尽快结束）
- [ ] 事务里发 MQ → 用 `@TransactionalEventListener(AFTER_COMMIT)`
- [ ] 大批量操作分批，单批 < 1000 行；不要在一个事务里写几万行
- [ ] 避免在事务里调用 `@Cacheable` 方法（缓存读到的可能是事务前的旧值）

### 传播
- [ ] 嵌套调用前画清楚每层的传播；明白每次调用占几条物理连接
- [ ] `REQUIRES_NEW` 嵌套层数 ≤ 2，避免连接需求暴增
- [ ] `NESTED` 用前确认 DB / TM 支持；推荐场景：批量处理里"局部失败可丢弃"
- [ ] `MANDATORY` 用在"必须有外层事务"的工具方法（防止误用）
- [ ] `NEVER` / `NOT_SUPPORTED` 用在明确不能在事务里跑的逻辑（比如 DDL）

### 多数据源
- [ ] 单个 `@Transactional` **不要同时操作多个 DataSource**（除非用了 JTA 或明确接受不一致）
- [ ] 多 DataSource 用 `@Transactional("xxxTxManager")` 显式指定
- [ ] 跨库一致性需求 → 本地消息表 / Saga / Seata，不指望 `ChainedTransactionManager`

### 连接池
- [ ] HikariCP `maximumPoolSize` 算过：`QPS × P95 事务时长 × 安全系数 1.5`
- [ ] HikariCP `connectionTimeout` < 业务超时，避免业务等连接超时
- [ ] HikariCP `leakDetectionThreshold` 开（如 60_000ms），及时发现连接泄漏
- [ ] 连接池监控：active、pending、wait time

### 监控告警
- [ ] 监控 `UnexpectedRollbackException` 数量（说明嵌套事务有未处理失败）
- [ ] 监控长事务（DB 层 `SHOW PROCESSLIST` 或 `pg_stat_activity`），> 5s 告警
- [ ] 监控连接池 `pendingThreads`，持续 > 0 告警
- [ ] 监控事务超时异常 `TransactionTimedOutException`

### 测试
- [ ] 关键事务方法必须有"异常路径必回滚"的测试
- [ ] 跨 Bean 调用必须测代理生效
- [ ] 嵌套传播必须有"内层失败、外层处理、最终状态"的端到端验证

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### JTA / XA 与 Spring
- `JtaTransactionManager` 与 Atomikos / Narayana / Bitronix 的集成
- XA 协议两阶段提交（Prepare + Commit）的工作流
- XA 的"协调者单点故障"问题与 in-doubt 事务恢复
- 为什么云原生时代越来越少用 XA：性能差、运维复杂、与 K8s/容器化不友好
- `Last Resource Commit Optimization` 与 `LRC`（一种 XA 退化优化）

### Reactive 事务
- `ReactiveTransactionManager` 与 `R2dbcTransactionManager`
- WebFlux 里 `@Transactional` 的工作机制（不是 ThreadLocal，是 Reactor `Context`）
- Reactor 的 `TransactionalOperator.transactional()` 显式 API
- Reactive 下传播的限制：很多传播行为在响应式模型里没意义
- Spring Data R2DBC 的事务边界

### Spring 事务与分布式事务
- 本地消息表（事务表 + 调度补偿）
- Saga 模式：长事务拆补偿
- Seata 的 AT 模式（自动补偿）/ TCC 模式 / XA 模式
- 最终一致性 vs 强一致性的业务取舍

### HikariCP 与事务
- HikariCP 的 `HouseKeeper` 线程：连接保活、过期回收
- 连接池"借出 / 归还"的具体实现：`ConcurrentBag`
- `ProxyConnection` 怎么拦截 `commit/rollback/close`
- HikariCP 的 leak detection 怎么发现"借了不还"
- 与 Druid 对比：监控、SQL 拦截、慢查询统计

### JPA / Hibernate 与事务
- `Session` / `EntityManager` 与 Spring 事务的绑定（`OpenSessionInViewFilter` 的争议）
- Hibernate 的"脏检查"在 commit 时刷新到 DB（区别于 JdbcTemplate 立即执行）
- Hibernate 二级缓存与事务隔离的相互作用（缓存可能让你看到"已提交但被别人回滚的数据"）
- Hibernate 的 `flush mode`：AUTO / COMMIT / MANUAL 对事务的影响

### MyBatis 与事务
- `SqlSessionTemplate` 怎么和 Spring 事务集成
- `@MapperScan` 注入的 Mapper 是怎么拿到事务的 Connection 的
- `SqlSession.commit()` vs Spring 的 commit 谁先谁后

### 隔离级别与 MVCC
- Spring `@Transactional(isolation = ...)` 实际是 `Connection.setTransactionIsolation`
- RR / RC 在 DB 层的实现差异（对 MVCC 的影响）
- 详见 [MySQL MVCC 与锁](/article/db/mysql-mvcc)

---

## 7. 引用

- Spring Framework 6.1 源码：[spring-tx](https://github.com/spring-projects/spring-framework/tree/main/spring-tx)
- Spring 官方文档：[Data Access · Transaction Management](https://docs.spring.io/spring-framework/reference/data-access/transaction.html)
- Rod Johnson, *Expert One-on-One J2EE Design and Development*（Spring 设计的源头思想）
- Juergen Hoeller 的演讲 *Inside the Spring Transaction Manager*（YouTube）
- HikariCP 文档：[Connection Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
- 美团技术博客：[Spring 事务源码分析](https://tech.meituan.com/)
- 阿里 Seata 文档：[AT / TCC / Saga 模式](https://seata.io/zh-cn/docs/overview/what-is-seata.html)
- 《Spring 揭秘》—— 王福强（讲清 Spring 内部机制的中文好书）

---

> 配套：[Spring 核心原理](/article/java/spring) | [Spring AOP 内幕](/article/java/spring-aop) | [MySQL MVCC 与锁](/article/db/mysql-mvcc) | [分布式事务](/article/distributed/dist-tx)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
