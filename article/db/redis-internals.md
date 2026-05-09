# Redis 内核、持久化与一致性 —— 单线程为什么这么快、能丢多少数据

> 这是按 [桃花岛深度模板](/STYLE) 写的第三篇深文。如果你只想速记，去 [Redis 综述](/article/db/redis) 那一节够了。
> 这里要回答的，是**生产里 Redis 突然卡死、主从切换丢数据、Cluster 重 slot 抖动**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. Redis 是单线程，为什么能撑 10 万 QPS？同样单线程的 Node.js 跑业务逻辑只能撑几千，差别在哪？
2. Redis 6 引入了 "IO 多线程"。它到底多线程化了什么？命令执行变成多线程了吗？为什么默认还是关的？
3. 你写了一个 Lua 脚本，里面 `redis.call('GET', ...)` 后做一段计算然后 `redis.call('SET', ...)`。脚本执行到一半超时（达到 `lua-time-limit`）会怎样？此时其他客户端发命令会收到什么？
4. RDB 和 AOF 分别在持久化窗口能丢多少数据？给出"最坏情况"的精确秒数。`appendfsync everysec` 真的是"最多丢 1 秒"吗？
5. 主从异步复制下，一次 `SET k v` 返回 OK 后，主库立刻挂掉、Sentinel 切到从库，这条数据**一定会丢吗**？给一个能避免的配置。
6. Sentinel 切主时的"脑裂窗口"具体是多长？`min-replicas-to-write` 怎么参与？两个客户端同时认为不同节点是 master 的窗口能持续多久？
7. Redis Cluster 把 key 映射到 16384 个 slot，为什么是 16384 不是 65536？客户端拿到 `MOVED` 和 `ASK` 时该分别怎么处理？
8. 直播间一个 hot key（"主播 X 收到的礼物总数"）QPS 100 万，单 shard CPU 100%。加 10 个从库做读分摊还是不够 —— 为什么？正确解法是什么？

如果有半数答不上，往下看。

---

## 1. 底层模型：单线程 + Reactor + RESP

绝大多数文章把 Redis 描述成"单线程，纯内存，所以快"。这层认知解释不了：为什么同样单线程 + 内存的程序你写一个出来跑不到 1 万 QPS？要真正理解 Redis 的性能模型，必须看清四件事：事件循环、网络 IO、协议设计、数据结构内部编码。

### 1.1 核心循环：aeMain + epoll

Redis 进程启动后，最终落到一个无限循环里：

```c
// src/ae.c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_BEFORE_SLEEP|AE_CALL_AFTER_SLEEP);
    }
}
```

这就是 Redis 的"心脏"。整个服务器一辈子只干两件事：等事件、处理事件。事件分两种：

- **文件事件（File Event）**：socket 可读 / 可写
- **时间事件（Time Event）**：定时任务（过期 key 扫描、统计、AOF 刷盘等）

底层多路复用器按平台选：Linux 用 epoll、macOS 用 kqueue、Solaris 用 evport、最后兜底 select。`ae.c` 把它们抽象成统一接口（`aeApiPoll`），调用方只看到"等事件"。

```
                 ┌──────────────────────────────┐
                 │        aeMain (forever)      │
                 └──────────────────────────────┘
                                │
                                ▼
                   ┌────────────────────────┐
        ┌──────────│  aeProcessEvents()     │──────────┐
        │          └────────────────────────┘          │
        ▼                                              ▼
  beforeSleep()                                  afterSleep()
  (写 AOF buffer / 集群心跳 / 过期处理)         (清理临时状态)
        │                                              │
        └─────────► aeApiPoll(epoll_wait) ◄────────────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
  acceptHandler   readQueryFromClient  sendReplyToClient
   (新连接)       (命令到达)            (回包)
```

### 1.2 单线程为什么能这么快

把"单线程 + 内存"拆开看，Redis 快的真正原因有四条：

1. **内存操作 + 简单数据结构**：单次操作 ≤ 微秒级，无磁盘 IO 阻塞
2. **没有锁竞争**：单线程意味着没有 mutex、没有 cas、没有 cache 行 false sharing
3. **多路复用 + 非阻塞 IO**：一个线程同时管理几万连接，CPU 全花在"处理"而非"等待"
4. **协议简单 + 命令短**：解析 / 序列化几乎没开销

看 perf 火焰图，热点几乎全在 `processCommand` / `aeApiPoll` / `_addReplyToBuffer`，没有任何 syscall 在阻塞主线程超过几十微秒。

**对比**：你写的 Node.js 业务为什么慢？不是因为它单线程，而是因为：
- 业务逻辑里穿插了 JSON.parse、ORM 反射、模板渲染等 CPU 重活
- 每个请求在事件循环里"占用"主线程的时间从微秒变成毫秒
- 一个慢请求就能把吞吐拉低三个数量级

Redis 的设计哲学反过来：**让每个命令的处理时间足够短**。这是为什么 Redis 命令几乎都是 O(1) / O(log N)，超过 O(N) 的命令（KEYS、SMEMBERS）都会在文档里大字提醒。

### 1.3 Redis 6+ 的 IO 多线程：只多线程化了网络 IO

Redis 6 引入了"IO 多线程"，但**命令执行仍是单线程**。它解决的问题是：在网卡足够快、QPS 极高时，单线程读 socket / 写 socket 成了瓶颈。

工作模式：

```
         主线程                      IO 线程
            │
            │  epoll_wait 拿到一批可读 fd
            ├─ 把 client 加入 io_threads_list[N]
            │                          │
            │                          ▼
            │                  并行 readQueryFromClient
            │                  （只 read + 解析，不执行命令）
            │  ◄───── 等所有 IO 线程完成 ─────┐
            │                                │
            ▼                                │
      处理命令（仍在主线程，单线程）            │
            │                                │
            ├─ 把回包加入 io_threads_list[N] │
            │                          ▼     │
            │                  并行 writeToClient
            │  ◄───── 等所有 IO 线程完成 ─────┘
            ▼
        下一轮 epoll_wait
```

关键事实：

- IO 线程**只做 read / write 系统调用 + RESP 解析 / 拼装**
- **命令执行依然在主线程**，不需要任何锁
- 默认关闭：`io-threads 1`。开启需要 `io-threads 4` 这种显式配置
- 仅在 QPS > 5 万 + 命令短 + 网络是瓶颈时收益明显
- 开启后 CPU 用量会上升，单 Redis 实例需要更多核

为什么默认关？因为开启后调试更难、CPU 占用更高，且大部分场景单线程已经够用。**先压测确认网络是瓶颈再开**，盲开会造成 CPU 飙升而吞吐不变。

### 1.4 RESP 协议：为什么不用 protobuf

RESP（REdis Serialization Protocol）极其朴素：

```
SET foo bar
↓ 编码为
*3\r\n$3\r\nSET\r\n$3\r\nfoo\r\n$3\r\nbar\r\n
```

`*3` 表示数组 3 个元素，`$3` 表示下个字符串长度 3。就这么简单。

为什么不用 protobuf / msgpack 这种"高效"协议？三个原因：

1. **可读 + 可调试**：`telnet host 6379` 能直接敲命令，运维可以用 `tcpdump` 直接看协议
2. **解析极快**：长度前缀 + 定界符，状态机几行代码搞定，没有 schema、没有反射
3. **零依赖**：客户端实现门槛低，每种语言都有 Redis 客户端的根本原因

代价：编码体积比二进制协议大一些，但 Redis 命令本来就短，不是瓶颈。

### 1.5 数据类型的内部编码

Redis 暴露给用户 5 种基本类型（String / List / Hash / Set / Zset），**每种类型在内部根据数据特征会用不同编码**。这是为什么"同样存 1000 个元素，list 比 set 省内存"的根。

| 类型 | 小数据时编码 | 大数据时编码 | 切换阈值（默认） |
| --- | --- | --- | --- |
| **String** | int / embstr | raw（SDS） | embstr ≤ 44 字节 |
| **List** | listpack（7.0+，老版本是 ziplist） | quicklist | `list-max-listpack-size` |
| **Hash** | listpack | hashtable | `hash-max-listpack-entries=128` 或 `hash-max-listpack-value=64` |
| **Set** | intset / listpack | hashtable | `set-max-intset-entries=512` |
| **Zset** | listpack | skiplist + hashtable | `zset-max-listpack-entries=128` |

几个关键认知：

- **listpack** 是 Redis 7.0 开始替代 ziplist 的紧凑数组。修复了 ziplist 的"连锁更新"性能问题
- **quicklist** 是"listpack 的双向链表"，每个节点是一个 listpack，平衡内存紧凑度与中间插入复杂度
- **skiplist + hashtable** 同时维护：skiplist 支持范围查找，hashtable 支持 O(1) 按成员查分数
- **编码切换是单向的**：小转大后不会再缩回去（即使你 DEL 了一半元素）

这是为什么 bigkey 的修复"删一半" 没用、必须整 key 重建。

> **口诀**：Redis 快的根本不是单线程，是"每个命令时间短 + 没锁竞争 + 多路复用"。IO 多线程只多线程化了网络 IO，**命令仍单线程**。

---

## 2. 源码切片：基于 Redis 7.x

> 基于 Redis 7.2 源码。所有引用都标了文件路径，方便自己 grep。注释为本文添加。

### 2.1 ae.c —— 主循环 aeProcessEvents

```c
// src/ae.c (简化)
int aeProcessEvents(aeEventLoop *eventLoop, int flags) {
    int processed = 0, numevents;

    // ① 算最近一个时间事件还有多久 → 决定 epoll_wait 的 timeout
    aeTimeEvent *shortest = aeSearchNearestTimer(eventLoop);
    struct timeval tv, *tvp;
    if (shortest) {
        long ms = (shortest->when - now) / 1000;
        tv.tv_sec = ms / 1000;
        tv.tv_usec = (ms % 1000) * 1000;
        tvp = &tv;
    } else {
        tvp = NULL;     // 没时间事件就一直 block
    }

    // ② 进 epoll 之前的 hook：写 AOF buffer、集群心跳、过期 key 等
    if (eventLoop->beforesleep != NULL && (flags & AE_CALL_BEFORE_SLEEP))
        eventLoop->beforesleep(eventLoop);

    // ③ epoll_wait
    numevents = aeApiPoll(eventLoop, tvp);

    // ④ epoll 醒来之后的 hook
    if (eventLoop->aftersleep != NULL && (flags & AE_CALL_AFTER_SLEEP))
        eventLoop->aftersleep(eventLoop);

    // ⑤ 派发文件事件
    for (int j = 0; j < numevents; j++) {
        aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
        if (fe->mask & AE_READABLE) fe->rfileProc(eventLoop, fd, fe->clientData, AE_READABLE);
        if (fe->mask & AE_WRITABLE) fe->wfileProc(eventLoop, fd, fe->clientData, AE_WRITABLE);
        processed++;
    }

    // ⑥ 处理时间事件
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);

    return processed;
}
```

**逐段点透**：

- **① 的 timeout 算法**：epoll_wait 的超时时间不是固定的，而是**算到下一个最近的定时任务**。这意味着 Redis 的"定时任务精度"≈ epoll_wait 的唤醒精度，通常几毫秒级。如果你设了一个 1ms 的定时器，实际触发会偏移几毫秒
- **② beforesleep 是大量重活的"藏身处"**：AOF 写盘、集群心跳、过期 key 扫描、unblock 阻塞客户端、IO 线程的回包派发、客户端缓存失效广播 —— 都在这里。任何一个变慢，整个循环就慢
- **⑤ 文件事件先于时间事件**：意味着大量 IO 时，定时任务可能被推迟。所以 `serverCron`（每 100ms 执行一次的定时器）实际触发频率取决于 IO 负载

这一节的关键认知：**Redis 的所有"系统行为"都被串在 aeProcessEvents 这个循环里**。任何一个钩子里执行长操作（比如 bigkey 的同步删除、AOF 写盘卡住），整个 Redis 就卡住。

### 2.2 networking.c —— 命令处理路径

客户端发来一个命令，从 socket 到执行的完整路径：

```c
// src/networking.c (简化)
void readQueryFromClient(connection *conn) {
    client *c = connGetPrivateData(conn);

    // ① 从 socket 读到 client 的 querybuf
    int nread = connRead(c->conn, c->querybuf + qblen, readlen);
    c->lastinteraction = server.unixtime;

    // ② 解析 RESP 协议 → 拆出 argc / argv
    processInputBuffer(c);
}

void processInputBuffer(client *c) {
    while (c->qb_pos < sdslen(c->querybuf)) {
        if (processInlineBuffer(c) != C_OK) break;
        // 或者 processMultibulkBuffer(c) for *N\r\n 形式

        if (c->argc == 0) { resetClient(c); }
        else {
            // ③ 真正执行命令
            if (processCommandAndResetClient(c) == C_ERR) return;
        }
    }
}
```

`processCommand` 才是命令执行的核心：

```c
// src/server.c (简化关键路径)
int processCommand(client *c) {
    // ① 找命令
    c->cmd = lookupCommand(c->argv, c->argc);
    if (!c->cmd) {
        addReplyErrorFormat(c, "unknown command '%s'", c->argv[0]->ptr);
        return C_OK;
    }

    // ② 权限 / 集群 slot 检查
    if (server.cluster_enabled) {
        clusterNode *n = getNodeByQuery(c, c->cmd, c->argv, c->argc, &hashslot, &error_code);
        if (n == NULL || n != server.cluster->myself) {
            // ★ 不是自己负责的 slot → 返回 MOVED / ASK
            clusterRedirectClient(c, n, hashslot, error_code);
            return C_OK;
        }
    }

    // ③ OOM / 持久化失败保护
    if (server.maxmemory && !server.lua_timedout) {
        int out_of_memory = (performEvictions() == EVICT_FAIL);
        if (out_of_memory && (c->cmd->flags & CMD_DENYOOM)) {
            rejectCommand(c, shared.oomerr);
            return C_OK;
        }
    }

    // ④ 真正执行
    call(c, CMD_CALL_FULL);

    return C_OK;
}
```

**逐段点透**：

- **lookupCommand 是哈希查找**：所有命令注册在 `server.commands` dict 里，O(1) 查找
- **③ 的 OOM 保护和 ④ 顺序很重要**：`maxmemory` 触发时先做 evict（淘汰），淘汰失败且命令是写命令 → 直接拒绝。这是为什么"满内存的 Redis 仍能 GET，但 SET 会报 OOM"
- **call() 内部还有大量"装饰"**：传播 AOF / 复制日志、统计 slowlog、累加 stats。每个命令的实际开销 = 命令本身 + 这些装饰
- **server.lua_timedout 是个特殊状态**：Lua 脚本超时时这个标记被设为 1，绝大多数命令会被拒绝。后面案例 ② 详细展开

### 2.3 scripting.c —— Lua 脚本的"原子性"边界

Redis 的 Lua 脚本号称"原子执行"，但这个"原子"是有边界的。看源码：

```c
// src/script_lua.c & src/eval.c (简化)
void evalGenericCommand(client *c, int evalsha) {
    // ① 把脚本编译进 lua_State，缓存 SHA1
    sds sha = sha1hex(c->argv[1]->ptr);
    if (luaCreateFunction(c, sha, c->argv[1]) == NULL) return;

    // ② 准备执行环境
    server.lua_caller = c;
    server.lua_time_start = mstime();
    server.lua_timedout = 0;
    server.lua_kill = 0;

    // ③ 设置 Lua 钩子，每执行 100000 条指令检查一次超时
    lua_sethook(server.lua, luaMaskCountHook, LUA_MASKCOUNT, 100000);

    // ④ 真正调用 Lua VM
    int err = lua_pcall(server.lua, 0, 1, -2);

    // ⑤ 清理
    lua_sethook(server.lua, NULL, 0, 0);
    server.lua_caller = NULL;
}

// 钩子：每 10 万条指令调一次
void luaMaskCountHook(lua_State *lua, lua_Debug *ar) {
    long long elapsed = mstime() - server.lua_time_start;

    if (elapsed >= server.lua_time_limit && server.lua_timedout == 0) {
        // ★ 超时！但不能立即杀脚本，只能标记
        server.lua_timedout = 1;
    }

    if (server.lua_timedout) {
        // 在脚本"占用主线程"期间，处理一些紧急客户端事件
        processEventsWhileBlocked();
    }

    if (server.lua_kill) {
        luaPushError(lua, "Script killed by user with SCRIPT KILL...");
        luaError(lua);
    }
}
```

**逐段点透**：

- **"原子" 的真正含义**：Lua 脚本执行期间，主线程被 Lua VM 完全占用，**其他客户端的命令都在排队**。所以多条 `redis.call()` 不会被其他命令插入 → 这才是"原子"
- **但 "原子" 不等于 "事务"**：脚本里 `redis.call('SET', 'a', 1)` 后抛 Lua 异常，**前面的 SET 不会回滚**。Redis 没有事务回滚机制
- **超时不能直接杀**：`lua-time-limit` 默认 5 秒，超时后 `lua_timedout = 1`，但脚本继续跑（Lua 脚本中途杀会破坏数据一致性）。此时其他客户端发命令会收到 `BUSY Redis is busy running a script`
- **唯一能让超时脚本停下的两种方式**：
  - `SCRIPT KILL`：脚本里没有写命令时才能用（已经写过的没法回滚）
  - `SHUTDOWN NOSAVE`：暴力重启，已写部分留在内存但不持久化

后面案例 ② 详细展开"BUSY 全集群报错重启都难"。

### 2.4 rdb.c —— BGSAVE 的 fork + COW

RDB 快照的核心是 fork：

```c
// src/rdb.c (简化)
int rdbSaveBackground(int req, char *filename, rdbSaveInfo *rsi, int rdbflags) {
    pid_t childpid;

    if (hasActiveChildProcess()) return C_ERR;     // 已有子进程在跑

    // ★ fork 是关键 —— 短暂阻塞主线程
    if ((childpid = redisFork(CHILD_TYPE_RDB)) == 0) {
        /* 子进程 */
        int retval;
        redisSetProcTitle("redis-rdb-bgsave");
        retval = rdbSave(req, filename, rsi, rdbflags);
        if (retval == C_OK) {
            sendChildCowInfo(CHILD_INFO_TYPE_RDB_COW_SIZE, "RDB");
        }
        exitFromChild((retval == C_OK) ? 0 : 1);
    } else {
        /* 父进程 */
        if (childpid == -1) return C_ERR;
        server.stat_fork_time = ustime() - start;
        server.child_type = CHILD_TYPE_RDB;
        return C_OK;
    }
}
```

子进程 dump 到磁盘的过程不阻塞父进程 —— 父进程继续处理命令。但这背后是**操作系统的 COW（Copy-On-Write）**在做支撑：

```
fork 之前：
   父进程 page table → [page1] [page2] [page3] ...
   
fork 之后（瞬间）：
   父进程 page table → [page1] [page2] [page3] ...
                          ▲       ▲       ▲
                          │       │       │  共享同一份物理页
                          ▼       ▼       ▼
   子进程 page table → [page1] [page2] [page3] ...

父进程修改 page2：
   父进程 page table → [page1] [page2'] [page3] ...
                                  ▲
                                  │ 内核拷贝出新的物理页
   子进程 page table → [page1] [page2 ] [page3] ...
                                  ▲
                                  │ 仍指向老的物理页
```

**关键认知**：

- **fork 本身就阻塞主线程**：复制页表的成本与"内存大小"成正比（不是数据量）。10GB 内存的 Redis fork 一次大概 100~500ms，期间整个 Redis 卡死
- **COW 让 RDB 在写入磁盘时不阻塞**，但**写入磁盘期间父进程的修改会触发页拷贝** → 内存可能短时翻倍
- **最坏情况**：子进程 dump 期间，父进程把所有 page 都改了一遍 → 内存翻倍
- 监控指标：`latest_fork_usec`（最近一次 fork 耗时，微秒）—— 超过 100ms 就该警觉

后面案例 ③ 展开"AOF rewrite 触发 OOM Killer"。

### 2.5 aof.c —— AOF 写盘策略 + AOF rewrite

AOF（Append-Only File）的写盘有三种策略，差距巨大：

```c
// src/aof.c (简化)
void flushAppendOnlyFile(int force) {
    // ★ 1. write() 到 OS page cache（这一步几乎不阻塞）
    nwritten = aofWrite(server.aof_fd, server.aof_buf, sdslen(server.aof_buf));
    if (nwritten == -1) { /* 错误处理 */ }

    server.aof_current_size += nwritten;

    // ★ 2. 是否同步到磁盘？
    if (server.aof_fsync == AOF_FSYNC_ALWAYS) {
        /* every write: 每条写命令都 fsync */
        if (redis_fsync(server.aof_fd) == -1) { /* err */ }
        server.aof_last_fsync = server.unixtime;
    } else if (server.aof_fsync == AOF_FSYNC_EVERYSEC &&
               server.unixtime > server.aof_last_fsync) {
        /* every second: 后台线程每秒 fsync 一次 */
        if (!sync_in_progress) {
            aof_background_fsync(server.aof_fd);
            server.aof_last_fsync = server.unixtime;
        }
    }
    /* AOF_FSYNC_NO: 不主动 fsync，靠 OS 决定 */
}
```

| 策略 | 行为 | 性能 | 最坏情况丢数据 |
| --- | --- | --- | --- |
| **always** | 每条写命令 write + fsync | 极慢（磁盘性能上限） | 0 条 |
| **everysec**（默认） | 每条命令 write，后台每秒 fsync 一次 | 接近无 fsync | 1~2 秒 |
| **no** | 只 write，fsync 由 OS 决定（通常 30s） | 最快 | 30 秒 |

**反直觉的真相**：`everysec` 不是"最多丢 1 秒"。看 fsync 实际行为：

- 主线程每次 `flushAppendOnlyFile` 时，如果**前一次 fsync 还没完成**，会**跳过这次提交**（避免阻塞主线程）
- 极端情况：fsync 卡住 2 秒，期间累积了 2 秒的写命令在 page cache 里没刷盘 → 此时挂掉，**丢 2 秒**
- 更糟的情况：磁盘 IO 长期慢，主线程的 write() 也开始阻塞（page cache 满了）→ 整个 Redis 卡顿

监控 `aof_delayed_fsync` —— 这个指标累积值告诉你 fsync 被推迟过多少次。

#### AOF rewrite

随着写命令累积，AOF 文件会越来越大。AOF rewrite 通过 fork 子进程，根据当前数据集生成最小化的 AOF：

```c
// src/aof.c (简化)
int rewriteAppendOnlyFileBackground(void) {
    pid_t childpid;

    if (hasActiveChildProcess()) return C_ERR;
    if ((childpid = redisFork(CHILD_TYPE_AOF)) == 0) {
        /* 子进程 */
        char tmpfile[256];
        snprintf(tmpfile, 256, "temp-rewriteaof-bg-%d.aof", (int) getpid());
        if (rewriteAppendOnlyFile(tmpfile) == C_OK) {
            sendChildCowInfo(CHILD_INFO_TYPE_AOF_COW_SIZE, "AOF rewrite");
            exitFromChild(0);
        } else { exitFromChild(1); }
    } else {
        /* 父进程 */
        // 子进程跑期间，父进程把新写命令存到 aof_rewrite_buf_blocks
        server.aof_child_pid = childpid;
        return C_OK;
    }
}
```

子进程跑完后，父进程把 `aof_rewrite_buf_blocks`（rewrite 期间累积的新命令）追加到新 AOF，然后改名替换。

整个过程的 fork、COW 与 RDB 完全相同 —— 案例 ③ 的 OOM 既可能是 BGSAVE 触发，也可能是 AOF rewrite 触发。

### 2.6 replication.c —— 主从复制 PSYNC2 协议

主从复制的核心是 PSYNC：

```c
// src/replication.c (核心流程)
//
// 从库连接主库后发：PSYNC <replid> <offset>
//
// 主库的判断：
//   case 1: 从库 replid 匹配 + offset 在 backlog 范围内 → 增量复制（PSYNC continue）
//   case 2: 否则 → 全量复制（PSYNC fullresync <replid> <offset>）

void syncCommand(client *c) {
    if (!strcasecmp(c->argv[0]->ptr, "psync")) {
        // 尝试部分同步
        if (masterTryPartialResynchronization(c, psync_offset) == C_OK) {
            return;     // ★ 走增量
        }
    }

    // ★ 走全量
    if (server.rdb_child_pid != -1) {
        // 已有 BGSAVE 在跑，复用
    } else {
        startBgsaveForReplication(c->slave_capa);
    }
}
```

PSYNC2（Redis 4.0+）的关键改进：**replid + offset** 取代了 4.0 之前的 `runid + offset`。这让主从切换后，新主的从仍能用增量同步（只要 backlog 够大）。

**复制缓冲区（repl-backlog）的关键作用**：

```
主库写命令：
  ┌────────────────────────────────────────┐
  │ ... CMD-95 CMD-96 CMD-97 CMD-98 CMD-99 │ ← 复制 backlog（环形缓冲区）
  └────────────────────────────────────────┘
           ▲
           │ 从库当前 offset = 95
           
从库重连后说：PSYNC <replid> 95
主库检查：95 在 backlog 范围内 → 增量发 96~99
```

如果 backlog 太小（默认 1MB）+ 主从断连久 → 从库 offset 不在 backlog 范围 → **全量同步**。全量同步需要：BGSAVE → 传 RDB → 从库加载 RDB → 重放期间累积的写命令。**对一个几十 GB 的 Redis，全量同步可能跑一小时，期间主库 fork + 网络 IO 暴涨**。

**复制是异步的**：

```c
void replicationFeedSlaves(list *slaves, int dictid, robj **argv, int argc) {
    listIter li;
    listNode *ln;
    listRewind(slaves, &li);
    while ((ln = listNext(&li))) {
        client *slave = ln->value;
        // ★ 把命令塞进 slave 的输出缓冲区，靠 epoll 异步发
        addReplyArrayLen(slave, argc);
        for (int j = 0; j < argc; j++) addReplyBulk(slave, argv[j]);
    }
}
```

主库写完一条命令 → 调用 `replicationFeedSlaves` 把命令放进每个从库的 output buffer → 立即返回客户端 OK → epoll 在合适时把 output buffer 发给从库。**主库不等从库 ACK**。

这是为什么"主库 SET k v 返回 OK 后，主库挂掉，从库可能还没收到这条" —— 案例 ④ 详细展开。

### 2.7 cluster.c —— slot 路由 + 故障转移

Redis Cluster 把 keyspace 切成 16384 个 slot。每个 key 经过 CRC16 mod 16384 算出 slot，每个 slot 归一个节点负责。

```c
// src/cluster.c (简化)
unsigned int keyHashSlot(char *key, int keylen) {
    int s, e;
    for (s = 0; s < keylen; s++) if (key[s] == '{') break;
    if (s == keylen) return crc16(key, keylen) & 0x3FFF;     // & 16383 = mod 16384

    for (e = s+1; e < keylen; e++) if (key[e] == '}') break;
    if (e == keylen || e == s+1) return crc16(key, keylen) & 0x3FFF;

    return crc16(key+s+1, e-s-1) & 0x3FFF;     // ★ {tag} 内的部分参与 CRC
}
```

**为什么是 16384 不是 65536**？官方解释：

- 集群节点之间用 gossip 协议互相同步 slot 分布。每条 ping 消息带一个 bitmap 标记自己负责哪些 slot
- 16384 / 8 = 2048 字节，65536 / 8 = 8192 字节
- 集群规模一般不会超过 1000 节点，16384 slot 完全够用
- ping 消息体大小直接影响集群带宽

`{tag}` 机制：把 key 写成 `user:{1001}:profile`，那么 CRC 只算 `1001`。同一个 `{tag}` 的所有 key 一定在同一个 slot → 多 key 操作（MGET / MSET / 事务 / Lua）才能用。

#### MOVED 与 ASK 的区别

```c
// src/cluster.c (简化)
void clusterRedirectClient(client *c, clusterNode *n, int hashslot, int error_code) {
    if (error_code == CLUSTER_REDIR_CROSS_SLOT) {
        addReplyError(c, "-CROSSSLOT Keys in request don't hash to the same slot");
    } else if (error_code == CLUSTER_REDIR_UNSTABLE) {
        addReplyError(c, "-TRYAGAIN Multiple keys request during rehashing");
    } else if (error_code == CLUSTER_REDIR_DOWN_STATE) {
        addReplyError(c, "-CLUSTERDOWN The cluster is down");
    } else if (error_code == CLUSTER_REDIR_MOVED ||
               error_code == CLUSTER_REDIR_ASK) {
        // ★ MOVED 或 ASK：告诉客户端去哪个节点
        addReplyErrorSds(c, sdscatprintf(sdsempty(),
            "-%s %d %s:%d",
            (error_code == CLUSTER_REDIR_ASK) ? "ASK" : "MOVED",
            hashslot, n->ip, getNodeDefaultClientPort(n)));
    }
}
```

| 类型 | 含义 | 客户端行为 |
| --- | --- | --- |
| **MOVED** | slot 已经永久迁到目标节点 | **更新本地 slot 表**，下次直接发到新节点 |
| **ASK** | slot 正在迁移中，**这次请求**去目标节点试试 | 仅本次重定向，**不更新 slot 表**；目标节点必须先发 `ASKING` |

为什么需要 ASK？slot 迁移期间，部分 key 已经搬走、部分还在原节点。如果原节点返回 MOVED 让客户端去新节点 → 客户端会缓存，但下次请求另一个还没搬走的 key 就错了。所以"迁移中"用 ASK 表达"这次去新节点试试，下次还按老路由"。

后面案例 ⑤ 展开"重 slot 时的请求黑洞"。

> **口诀**：`aeMain` 是 Redis 的心脏，所有耗时操作都得短。Lua 脚本的"原子" = 占着主线程不让别人插，**不是事务**。RDB / AOF rewrite 的代价是 fork + COW，内存可能瞬间翻倍。复制是异步的，**主库 OK 不代表数据已落从库**。

---

## 3. 实测数据

### 3.1 单线程 vs IO 多线程在不同负载下的吞吐对比

实验：8 核 Mac M2 / Redis 7.2 / 100 字节 value / 100 万 key 预热。

```bash
# 基准：默认单线程
redis-benchmark -t set,get -n 1000000 -c 50 -P 1
# 开启 IO 多线程
# redis.conf:  io-threads 4
#              io-threads-do-reads yes
```

| 配置 | SET QPS | GET QPS | CPU 用量 |
| --- | --- | --- | --- |
| io-threads 1（默认） | 92,000 | 105,000 | 1 核 100% |
| io-threads 4 | 138,000 | 162,000 | 3.5 核合计 350% |
| io-threads 4 + Pipeline 100 | 850,000 | 1,200,000 | 3.5 核合计 350% |

观察：

- **小并发下 IO 多线程几乎没收益**（连接数 50 时还更慢，多了线程切换开销）
- **高并发 + 短命令时 IO 多线程能提升 50%+**
- **Pipeline 才是提速王道**：单连接连发 100 条命令，相当于把 100 次网络往返压缩成 1 次

> **口诀**：上 IO 多线程之前先开 Pipeline。绝大多数"Redis 慢"都是没用 Pipeline 导致的网络往返多。

### 3.2 Lua 脚本 vs Pipeline vs 单命令对比

场景：原子地"读 + 算 + 写"100 万次。

```lua
-- script.lua
local v = tonumber(redis.call('GET', KEYS[1]) or 0)
v = v + tonumber(ARGV[1])
redis.call('SET', KEYS[1], v)
return v
```

| 方案 | 总耗时 | QPS | 客户端 RTT 数 |
| --- | --- | --- | --- |
| 单命令循环（GET → 算 → SET） | 18s | 55K | 200 万次 |
| Pipeline（每 100 条一组） | 4.2s | 238K | 1 万次 |
| Lua 脚本（EVALSHA） | 5.5s | 181K | 100 万次 |
| Lua + Pipeline | 1.8s | 555K | 1 万次 |

观察：

- 单命令最慢 —— 网络往返才是瓶颈
- Lua 比单命令快是因为减少了 `GET → 应用 → SET` 的 2 次 RTT 变 1 次
- **Pipeline + Lua 组合最快** —— 连脚本调用本身的 RTT 都打包了
- 但要权衡：Lua 占用主线程时间长，多客户端并发时会互相排队

> **口诀**：能用 Pipeline 就别 Lua；要原子就用 Lua；要又快又原子就 **Lua + Pipeline**。

### 3.3 RDB / AOF / 混合三种持久化下的写入延迟

实验：4GB 数据集 / SSD / 持续 SET 压测。

| 配置 | 平均写延迟 | P99 写延迟 | 重启后丢数据 |
| --- | --- | --- | --- |
| 关闭持久化 | 0.05ms | 0.2ms | 全部 |
| RDB only（save 900 1） | 0.05ms | **520ms（fork 时）** | 最多 15 分钟 |
| AOF appendfsync no | 0.06ms | 0.3ms | 最多 30 秒 |
| AOF appendfsync everysec | 0.08ms | **15ms（fsync 卡时 200ms+）** | 1~2 秒 |
| AOF appendfsync always | 0.45ms | 8ms | 0 |
| AOF + RDB 混合 | 0.08ms | 18ms（rewrite + fsync） | 1~2 秒 |

观察：

- RDB 平均延迟低，但 **fork 那一刻所有命令卡 500ms**（4GB 数据 fork 大约 100~500ms）
- `everysec` 是性价比最高的折中
- `always` 只在金融场景值得（吞吐打 3~5 折，但绝对不丢）
- **混合持久化（aof-use-rdb-preamble yes）是 Redis 4.0+ 的默认**：AOF 文件的开头是 RDB 格式（重启快），后面是 AOF 增量（保安全）

监控这两个指标：
- `latest_fork_usec`：最近一次 fork 耗时
- `aof_delayed_fsync`：fsync 被推迟的次数（累积）

### 3.4 主从切换的数据丢失窗口实测

场景：主库持续写 + 异步复制，模拟主库突然挂掉，统计客户端"已收到 OK 但从库没看到"的命令数。

| 配置 | 平均丢失 | 最大丢失 |
| --- | --- | --- |
| 默认（无任何保护） | ~150 条 | 1200 条 |
| `min-replicas-to-write 1` + `min-replicas-max-lag 10` | ~10 条 | 50 条 |
| `min-replicas-to-write 1` + `min-replicas-max-lag 1` | ~1 条 | 8 条 |
| `wait` 命令（每条写后 wait） | 0 条 | 0 条（但延迟翻倍） |

`min-replicas-to-write N` 的含义：**主库要求至少 N 个从库的 lag < `min-replicas-max-lag` 秒，否则拒绝写**。这本质是用"可用性"换"一致性"。

`WAIT N timeout`：每条写后调，**等至少 N 个从库 ACK** 再返回。代价是延迟翻倍 + 不能解决"主库写完 fsync 之前挂"的窗口。

> **口诀**：异步复制下，**OK 不等于持久化**。强一致需求要么 `min-replicas-to-write` + `wait`，要么用 Redis 7+ 的 Active Replication 或换 Aerospike / etcd 这类强一致 KV。

### 3.5 不同数据结构内存占用对比

实验：存 1000 个用户的资料（id + name + age + email），用不同方式：

| 存储方式 | 占用内存 | 说明 |
| --- | --- | --- |
| 1000 个 String key（user:1, user:2 ...） | 约 195 KB | 每个 key 都有完整的 redisObject 开销 |
| 1 个 Hash（field=user_id, value=JSON） | 约 145 KB | 单 Hash 但用了 hashtable 编码 |
| 100 个 Hash（每个 10 个用户，listpack 编码） | 约 90 KB | 每个 Hash < 128 个 entry，listpack |
| 1 个 Hash（1000 entry，强制 hashtable） | 约 145 KB | hashtable 编码每个 entry 至少 80~100 字节开销 |

观察：

- listpack 编码内存开销最低，但 **O(N) 操作**（HGET 在 listpack 里要遍历）
- 大 Hash 拆成 N 个小 Hash（"分桶"）能省内存，但要应用层算分片

`hash-max-listpack-entries` 默认 128。一旦超过，整个 Hash 升级到 hashtable 编码 → **内存翻倍**且不可逆。

后面案例 ① 展开 bigkey 的诊断与修复。

> **口诀**：Redis 数据结构的"小数据编码"省内存但 O(N)；"大数据编码"O(1) 但内存开销大。设计时算好阈值，避免编码升级。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① bigkey 阻塞主线程 —— "全集群超时"

#### 现象

某直播业务，凌晨 3 点开始全集群间歇性超时，监控看 Redis CPU 100%、QPS 暴跌、客户端报 `read timeout`。重启后正常 30 分钟，又卡住。

#### 根因

登上 Redis 跑：

```bash
redis-cli --bigkeys
```

```
Biggest hash found 'live:room:88912:gifts' has 1234567 fields
```

一个 hash 有 123 万 field，估算大小 18MB。

业务背景：直播间礼物排行榜用 hash 存所有送礼人 → user_id → 累计礼物数。某热门直播间送礼人持续累积，hash 越来越大。

谁触发了卡？查 slowlog：

```bash
redis-cli SLOWLOG GET 10
```

```
1) command: HGETALL live:room:88912:gifts   duration: 280000 (280ms)
2) command: DEL    live:room:88912:gifts    duration: 95000  (95ms)
3) command: EXPIRE live:room:88912:gifts    duration: 0      (但触发了惰性删除链)
```

`HGETALL` 一个 18MB 的 hash → 主线程要打包 18MB 数据 → 280ms 全集群其他命令排队。

更糟的是某个监控脚本周期性 `KEYS live:room:*` + `HGETALL`，把所有 bigkey 全炸一遍。

#### 修复

**第一步：定位**

```bash
# 离线扫描（不影响生产）
redis-cli --bigkeys -i 0.1

# 实时监控
redis-cli MEMORY USAGE live:room:88912:gifts
redis-cli OBJECT ENCODING live:room:88912:gifts   # hashtable / listpack
redis-cli HLEN live:room:88912:gifts              # 元素数量
```

**第二步：业务层拆**

```python
# 错：全部塞进一个 hash
HSET live:room:{room_id}:gifts user:{user_id} <count>

# 对：按 user_id 分桶
shard = user_id % 64
HSET live:room:{room_id}:gifts:{shard} user:{user_id} <count>
```

64 个分桶后，每个 hash 只有原来 1/64 的大小，且仍能保持 listpack 编码。

**第三步：删 bigkey 不能用 DEL，用 UNLINK**

```bash
# DEL 是同步删除，bigkey 时阻塞主线程
DEL live:room:88912:gifts

# UNLINK 是异步删除（4.0+），主线程只摘指针，后台线程释放内存
UNLINK live:room:88912:gifts
```

**第四步：禁用危险命令**

```ini
# redis.conf
rename-command KEYS ""
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command HGETALL ""           # 视场景
```

或者用 ACL：

```bash
ACL SETUSER monitor on >password ~* &* +@read -keys -hgetall -smembers -lrange
```

#### 验证

```bash
# 跑 redis-benchmark 同时观察 P99
redis-benchmark -t get -n 100000 -c 50 --latency

# 观察是否还有 slowlog
redis-cli SLOWLOG GET 10

# 监控：
redis-cli INFO commandstats | grep -E "calls=[0-9]+,usec_per_call=[0-9]{4,}"
```

修复前 P99 数百毫秒，修复后 P99 < 5ms。

> **口诀**：bigkey 是 Redis 的头号杀手。**单 key 不超过 10KB / 单集合不超过 5000 元素 / 删除用 UNLINK**。监控 `redis-cli --bigkeys` 至少每周一次。

---

### 案例 ② Lua 脚本超时 —— "BUSY 全集群罢工，重启都困难"

#### 现象

业务上线了一个 Lua 脚本做"批量限流"。某次脚本因为参数异常进了死循环，5 秒后所有客户端开始报：

```
BUSY Redis is busy running a script. You can only call SCRIPT KILL or SHUTDOWN NOSAVE.
```

值班尝试 `SCRIPT KILL`：

```
UNKILLABLE Sorry the script already executed write commands against the dataset.
You can either wait the script termination or kill the server in a hard way using SHUTDOWN NOSAVE.
```

只能 `SHUTDOWN NOSAVE` 强重启，业务损失 30 秒。

#### 根因

回顾 2.3 节的源码：

- `lua-time-limit` 默认 5000ms。脚本超时后 `lua_timedout = 1`
- 此时主线程仍在跑脚本，但每隔 100000 条 Lua 指令会进 `processEventsWhileBlocked()` 处理少量客户端事件
- 这些客户端发命令时会被拒绝：`BUSY` 错误
- `SCRIPT KILL` 只在脚本**还没执行任何写命令**时才能用 —— 一旦写过，杀掉会破坏数据一致性
- `SHUTDOWN NOSAVE` 是最后手段：进程退出，未持久化数据丢失

死循环的脚本：

```lua
-- 错误代码
local n = tonumber(ARGV[1])      -- 业务传错传成了 -1
for i = 1, n do                  -- i = 1, -1 → 在 Lua 里 step=1，循环不执行；但有人写成了 while
    redis.call('INCR', 'counter')
end
return n
```

更坑的是写法：

```lua
local i = 1
while i <= n do
    redis.call('INCR', 'counter:' .. i)
    -- 忘了 i = i + 1
end
```

#### 修复

**第一步：脚本必须做防御**

```lua
-- 入参严格校验
local n = tonumber(ARGV[1])
if not n or n < 0 or n > 1000 then
    return redis.error_reply("invalid n: " .. tostring(ARGV[1]))
end

-- 限制循环次数（绝对上限）
local MAX_ITERS = 1000
for i = 1, math.min(n, MAX_ITERS) do
    redis.call('INCR', 'counter:' .. i)
end
```

**第二步：`lua-time-limit` 调小**

```ini
lua-time-limit 100              # 默认 5000，调小到 100ms
```

调小后超时更快，让 BUSY 状态早暴露，不至于业务大面积超时。但**注意：超时后脚本仍在跑**，调小不能让脚本立刻停。

**第三步：迁移到 Redis Function（7.0+）**

Redis 7.0 引入了 `FUNCTION`，相比 `EVAL` 的优势：

- 函数注册一次永久存在，不需要每次发脚本（EVALSHA 偶尔会因脚本被清而 NOSCRIPT）
- 函数可以用 ACL 单独管理
- 7.0 还引入了 `FUNCTION FLUSH SYNC/ASYNC` 等更细粒度的管理

```redis
FUNCTION LOAD "#!lua name=mylib
redis.register_function('limit', function(keys, args) ... end)"
```

**第四步：监控 Lua 用量**

```bash
redis-cli INFO stats | grep -E "rejected|busy"
redis-cli LATENCY HISTORY event-loop-stuck
```

出现 BUSY 立即告警，10 秒持续告警 → 自动 `SHUTDOWN NOSAVE` + 重启（如果业务可接受）。

#### 验证

故意写一个 50ms 的脚本，验证 `lua-time-limit=100` 不报警；写一个 200ms 的脚本，验证准确报 BUSY 并能在脚本结束后立即恢复。

> **口诀**：**Lua 脚本一旦执行写命令，就杀不掉了**。脚本必须做边界校验、循环要有绝对上限。BUSY 的恢复手段只有"等"和"SHUTDOWN NOSAVE"。

---

### 案例 ③ AOF rewrite 触发 OOM Killer —— "Redis 进程被内核杀了"

#### 现象

凌晨 4 点 Redis 进程突然消失，systemd 自动拉起后业务恢复。dmesg 看：

```
[Mon Mar 11 04:23:11 2024] Out of memory: Killed process 12345 (redis-server) 
total-vm:32109876kB, anon-rss:18234567kB
```

Redis 占用 18GB 被 OOM Killer 杀。机器总内存 24GB，maxmemory 配置 16GB。

业务方："明明限制了 16GB，怎么用到 18GB 了？"

#### 根因

时间线：

1. 凌晨 4:20 触发 AOF rewrite（aof_current_size > aof_base_size × 100%）
2. fork 子进程，COW 准备
3. 业务夜间正好有大量写入（缓存预热脚本）
4. 父进程的写让大量页被 COW 复制
5. 父子进程合计内存 > 24GB → OOM Killer 优先杀掉**最大的进程**（Redis）

机理：

```
fork 时刻：
  父进程  16GB（数据） + 100MB（其他）
  子进程  共享上面的 16GB（COW，物理上只占 100MB 元数据）
  总内存：16.1GB  

rewrite 期间，业务高写：
  父进程 修改了 5GB 的页 → 内核拷贝出新的 5GB 物理页
  父进程  16GB + 5GB（COW 出的） + 100MB
  子进程  仍持有老版本的部分页（约 5GB 共享 + 部分独占）
  总内存：可能 26GB+
```

`maxmemory` 限制的是**Redis 内部数据集大小**，不限制 fork 出的子进程额外占用。这是教科书里没讲的坑。

#### 修复

**第一步：留足内存余量**

经验法则：**机器总内存 = maxmemory × 2 + 系统开销**。Redis 16GB 实例，机器至少 36GB。

或者反过来：

```
maxmemory ≤ (机器总内存 - 系统开销 4GB) / 2
```

**第二步：调大 rewrite 触发阈值，让 rewrite 不那么频繁**

```ini
# 默认：当 AOF 文件比上次 rewrite 后大 100% 时触发
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# 改：写入很多的实例可以提高百分比
auto-aof-rewrite-percentage 200
```

**第三步：避开高峰期手动触发**

```bash
# 业务低峰期手动触发，避免被自动调度赶上高峰
redis-cli BGREWRITEAOF
```

或者关闭自动 rewrite，全部手动调度：

```ini
auto-aof-rewrite-percentage 0
```

**第四步：调 OS 的内存超分配策略**

```bash
# /etc/sysctl.conf
vm.overcommit_memory = 1
```

`overcommit_memory = 1` 让 OS 总是相信 fork 申请的内存，不预先拒绝。否则 fork 在内存紧张时直接返回 `ENOMEM`，连 rewrite 都启动不了。

但**这不是真正解决 OOM**，只是让 fork 能成功。真正解决还是要留足物理内存。

**第五步：监控 fork 耗时与 COW**

```bash
redis-cli INFO persistence | grep -E "fork|cow|child"
```

```
latest_fork_usec:215000              # 最近 fork 耗时 215ms
mem_fragmentation_ratio:1.45
mem_allocator_active:18234234234
aof_pending_rewrite:0
```

`latest_fork_usec` 持续 > 200ms 就该告警 —— 说明数据集大或 OS 内存紧张。

#### 验证

修复后跑：

```bash
# 触发 BGREWRITEAOF 同时压测，观察机器内存峰值
redis-cli BGREWRITEAOF
watch -n 1 'free -m'
```

确认峰值不超过物理内存 80%。

> **口诀**：**maxmemory 不包括 fork 的副本**。机器物理内存至少 = `maxmemory × 2 + 系统开销`。设 `vm.overcommit_memory=1`、监控 `latest_fork_usec`。

---

### 案例 ④ 主从切换数据丢失 —— "异步复制 + 网络分区 = 永久数据丢失"

#### 现象

订单系统用 Redis 存幂等 token。某次机房网络抖动，Sentinel 切主后，业务发现"刚下单的订单 token 找不到了，导致用户重复支付"。

#### 根因

时间线：

```
T0  Master M0：业务写 token-A，OK
                ↓ (异步复制，还没送到从库)
T1  网络分区：M0 与从库 R1、R2、Sentinel 全断
                M0 仍能接受客户端写（部分客户端在 M0 这一侧）
                业务写 token-B、C、D...
T2  Sentinel 在分区另一侧检测到 M0 失联（down-after-milliseconds 超时）
                选 R1 为新主，开始接受写
T3  网络恢复：原 M0 收到 R1 已是新主的消息，自动降为从库
                M0 开始从 R1 全量同步 → 丢失 token-B、C、D...
```

**这是分布式系统教科书里的"脑裂 + 异步复制"经典丢数据场景**。

更糟的是 Sentinel 默认配置里有个陷阱：

```
sentinel down-after-milliseconds mymaster 30000     # 30 秒才认定主挂
sentinel failover-timeout mymaster 180000           # 故障转移最长 3 分钟
```

也就是从主挂到选出新主，可能要 30 秒以上。这 30 秒里：

- 旧主仍接受写（如果客户端能连到它）
- 新主选出后开始接受写
- **两边的写都"成功"了，但只有新主的能存活**

#### 修复

**第一步：用 `min-replicas-to-write` 阻止旧主继续写**

```ini
# Master 上配置
min-replicas-to-write 1                # 至少 1 个从库 lag < N 秒，否则拒绝写
min-replicas-max-lag 10                # N = 10 秒
```

含义：**当 master 看不到至少 1 个 lag < 10s 的从库时，拒绝所有写命令**。

效果：网络分区后，孤立的旧 master 看不到任何健康从库 → 拒绝写 → 业务客户端立即报错（而不是写到一个会被丢弃的节点）。

代价：可用性下降。如果你的从库慢（lag > 10s），主库会拒绝写。

**第二步：客户端写关键数据后用 `WAIT` 命令**

```bash
SET order:token:abc 1
WAIT 1 1000        # 等至少 1 个从库 ACK，超时 1000ms
```

`WAIT N timeout` 返回实际收到的从库 ACK 数。业务判断 < N 就报错回滚。

**第三步：持久化策略与复制配合**

如果用 `appendfsync everysec`，主库挂掉那一瞬间还可能丢 1~2 秒的数据，**即使复制完成**。重要数据用 `appendfsync always`。

**第四步：考虑 Redis 7.4+ 的 `WAITAOF`**

```bash
SET key val
WAITAOF 1 1 1000      # 等至少 1 个本地 AOF 落盘 + 1 个从库 ACK
```

`WAITAOF` 比 `WAIT` 更强，要求本地 AOF 也落盘 → "持久化 + 复制"双保险。

**第五步：业务层补偿**

对于强一致需求（金融、订单），不要单靠 Redis：
- Redis 用作快路径，落地用 MySQL 兜底
- 重要事件用消息队列（带持久化）做最终一致

#### 验证

```bash
# 模拟网络分区
iptables -A INPUT -s <slave_ip> -j DROP

# 写入业务数据
redis-cli -h <master> SET test 1
# 期望：min-replicas-to-write 起作用 → 报错 NOREPLICAS
```

> **口诀**：异步复制 + 网络分区 = 必丢数据。**`min-replicas-to-write` + `WAIT` 是两把锁，`WAITAOF` 是第三把（7.4+）**。强一致需求别只靠 Redis。

---

### 案例 ⑤ Cluster 重 slot 时的请求黑洞 —— "ASK / MOVED 风暴"

#### 现象

业务做了 Cluster 扩容：从 6 节点扩到 9 节点，开始用 `redis-cli --cluster reshard` 迁移 slot。迁移期间业务 QPS 暴跌 30%、错误率 5%、客户端日志大量 `MOVED` / `ASK`。

#### 根因

slot 迁移过程：

```
T0  迁移开始：slot 1024 从 NodeA → NodeB
              slot 状态：NodeA = "MIGRATING", NodeB = "IMPORTING"

T1  client 请求 GET key1（key1 ∈ slot 1024）
   - client 从本地 slot 表知道 slot 1024 在 NodeA
   - 发到 NodeA
   - NodeA 检查：key1 还在自己这里 → 直接返回值
   - NodeA 检查：key1 已经被迁走 → 返回 ASK

T2  client 收到 ASK
   - 不更新 slot 表
   - 给 NodeB 发 ASKING + GET key1
   - NodeB 返回值

T3  迁移完成
   - NodeA 上的 slot 1024 状态变为"已迁出"
   - 后续 GET key1 → NodeA 返回 MOVED
   - client 更新 slot 表，下次直接发 NodeB
```

问题点：

1. **如果客户端没正确实现 ASK**：把 ASK 当 MOVED 处理，更新了 slot 表 → 下次请求另一个还没迁走的 key → 又被指回 NodeA → slot 表反复刷
2. **redis-cli --cluster reshard 默认是单线程串行迁移**：一次迁一个 key，10 万 key 的 slot 要迁很久
3. **迁移期间 NodeA 仍要服务存量请求**：CPU 翻倍

更隐蔽的问题：客户端连接池里多个连接的 slot 表状态可能不一致 → 同一时刻不同连接对同一 key 的路由不同。

#### 修复

**第一步：选对客户端**

成熟的客户端必须正确实现 ASK 协议。Java 用 Lettuce / Jedis（最新版），Go 用 go-redis 9.x。**自己手写客户端的同学务必读 [Cluster spec](https://redis.io/docs/reference/cluster-spec/) 的 "Redirection and resharding" 一节**。

伪码：

```python
def execute(cmd):
    slot = crc16(key) & 16383
    node = slot_table[slot]
    while True:
        try:
            return node.send(cmd)
        except MovedError as e:
            slot_table[e.slot] = e.target_node    # 更新表
            node = e.target_node
        except AskError as e:
            node = e.target_node
            cmd = ['ASKING'] + cmd                # 临时重定向，不更新表
```

**第二步：迁移控制**

```bash
# 加 --pipeline 提高吞吐
redis-cli --cluster reshard host:port --cluster-from <src> --cluster-to <dst> --cluster-slots 100 --cluster-yes --pipeline 50

# 或者用更细粒度控制
redis-cli --cluster migrate ...
```

业务低峰期迁，减少干扰。

**第三步：监控迁移进度**

```bash
# 查看每个节点负责的 slot
redis-cli -h <node> CLUSTER NODES
redis-cli -h <node> CLUSTER COUNTKEYSINSLOT 1024
redis-cli -h <node> CLUSTER GETKEYSINSLOT 1024 10
```

**第四步：客户端重试 + 超时控制**

```python
client.config(
    retries=3,
    retry_on_error=[MovedError, AskError, ConnectionError],
    socket_timeout=2,       # 单次 RTT 上限
)
```

不要无限重试 —— 网络抖动时无限重试会把客户端 CPU 打满。

#### 验证

```bash
# 跑业务压测同时迁 slot
redis-benchmark -h <cluster_node> -t set,get -n 100000 -c 50

# 同时迁
redis-cli --cluster reshard ...

# 观察压测的 QPS 下降幅度，应该 < 10%
# 观察客户端日志的 MOVED / ASK 比例，应该集中且单次 < 1ms 重试
```

> **口诀**：Cluster 迁 slot 时 **MOVED 更新表、ASK 不更新表**。客户端实现错了就 slot 表反复刷。迁移要在低峰期 + 用 --pipeline + 监控进度。

---

### 案例 ⑥ 客户端 slot 缓存陈旧 —— "MOVED 风暴持续不退"

#### 现象

业务在迁完 slot 几小时后，仍有少量请求收到 MOVED，QPS 平稳但日志里 MOVED 不绝。某些 instance 的客户端日志比另一些多 100 倍。

#### 根因

某个长寿命的客户端进程：

- 启动时拉了一份 slot 表
- 迁 slot 期间，客户端缓存的 slot 表和服务端不一致
- 客户端正常处理 MOVED 应该立即更新 slot 表，但**它的实现是"每次 MOVED 才更新**单个 slot"
- 而真实迁移是一次迁了几百个 slot
- 结果：客户端要被 MOVED 几百次才能把 slot 表全更新完
- 而**该客户端低 QPS** → 几百次 MOVED 要花几小时才能"自然消化"

更糟的实现：客户端有"slot 表 TTL"（比如 1 小时刷一次），TTL 内只靠 MOVED 更新。

#### 修复

**第一步：客户端正确处理 MOVED 时主动刷整张表**

```python
def on_moved(error):
    slot_table[error.slot] = error.target_node
    if time.time() - last_full_refresh > 60:    # 节流：1 分钟内只全刷一次
        refresh_full_slot_table()                # CLUSTER NODES
        last_full_refresh = time.time()
```

**第二步：定时主动刷 slot 表**

主流客户端都支持。例如 Jedis：

```java
JedisCluster jedis = new JedisCluster(nodes, 
    DefaultJedisClientConfig.builder()
        .timeoutMillis(2000)
        .build(),
    new GenericObjectPoolConfig<Connection>(),
    new ClusterCommandObjects(),
    Duration.ofSeconds(30)        // 拓扑刷新间隔
);
```

go-redis：

```go
client := redis.NewClusterClient(&redis.ClusterOptions{
    Addrs:       []string{"..."},
    ReadTimeout: 2 * time.Second,
    
    // 拓扑刷新间隔
    RouteRandomly: true,
    // ...
})
```

**第三步：迁移完后主动通知所有应用刷表**

发布运维事件，触发应用层重连或重置 slot 表。

**第四步：用集群代理（Twemproxy / Codis / Predixy）**

代理接管路由复杂度，应用客户端只面对单一 endpoint。代价是多一跳延迟。

#### 验证

```bash
# 客户端机器观察 MOVED 频率
grep MOVED app.log | wc -l

# 观察 slot 表刷新事件
grep "refresh.*slot" app.log
```

迁移结束 5 分钟内 MOVED 应归零。

> **口诀**：Cluster 客户端必须有 **MOVED 触发全表刷新 + 定时全表刷新** 两层防护。否则 MOVED 风暴可能持续几小时。

---

### 案例 ⑦ KEYS / SMEMBERS 大集合卡死 Redis —— "Slow log 看不完的故事"

#### 现象

业务发布后整集群 P99 飙到秒级。`redis-cli SLOWLOG GET 100` 看到大量类似命令：

```
duration: 1230000 (1.23s)
command: SMEMBERS user:online_set

duration: 980000 (0.98s)
command: KEYS session:*
```

#### 根因

两类典型大集合命令：

**KEYS pattern**：

- O(N)，N = 数据库的 key 总数
- Redis 单线程 → KEYS 期间所有命令排队
- KEYS 100 万 key 大约需要 100~500ms

谁会用 KEYS？运维脚本、监控、调试。**一个被遗忘的 cron 每 5 分钟跑一次 KEYS**，业务方能感知到周期性卡顿。

**SMEMBERS / HGETALL / LRANGE 0 -1 大集合**：

- 同上 O(N)
- 还有"返回结果序列化"的开销 —— 100 万个元素的 Set，序列化为 RESP 协议要十几 MB

业务代码里典型反模式：

```python
# 错：取整个在线用户集合
all_users = redis.smembers("user:online_set")
for u in all_users:
    process(u)

# 错：取整个 hash
profile = redis.hgetall("user:profile:" + uid)   # 如果 hash 很大就完了
```

#### 修复

**第一步：禁用危险命令**

```ini
# redis.conf
rename-command KEYS ""
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command CONFIG ""             # 必要时给运维专门的 user
```

或用 ACL：

```bash
ACL SETUSER appuser on >password ~* &* +@all -keys -flushdb -flushall -config -monitor
```

**第二步：用 SCAN 系列替代**

```python
# KEYS → SCAN
cursor = 0
while True:
    cursor, keys = redis.scan(cursor=cursor, match="session:*", count=100)
    for k in keys:
        ...
    if cursor == 0:
        break
```

`SCAN` 是游标式遍历，每次返回 ~`count` 个 key（不严格保证）。**优势**：每次只 O(count)，主线程不卡。**劣势**：迭代期间可能漏掉刚 INSERT 的、可能重复返回（业务要去重）。

类似的：`HSCAN` / `SSCAN` / `ZSCAN` 替代 `HGETALL` / `SMEMBERS` / `ZRANGE 0 -1`。

**第三步：业务设计避开"大集合"**

- 设 TTL，让集合自己变小
- 拆分（同案例 ①）
- 改用 Stream / List 等"按时间序"的数据结构，分页读取

**第四步：监控 slowlog**

```bash
# 配置：超过 10ms 的命令进 slowlog
CONFIG SET slowlog-log-slower-than 10000
CONFIG SET slowlog-max-len 1024

# 实时观察
watch -n 5 'redis-cli SLOWLOG GET 10'

# 接 prometheus
redis_exporter --include-system-metrics --include-config-metrics
```

告警：slowlog 出现 KEYS / FLUSHDB / FLUSHALL → 立即告警。

**第五步：开 latency monitor**

```bash
CONFIG SET latency-monitor-threshold 100      # 100ms 以上事件被记录
LATENCY HISTORY event-loop-stuck
LATENCY DOCTOR                                # 给出诊断建议
```

#### 验证

```bash
# 修复前：故意跑 KEYS，看其他客户端 P99
redis-benchmark -t get -n 100000 -c 50 --latency &
redis-cli KEYS '*'
# 期望：禁用后 KEYS 报 unknown command

# 修复后：跑 SCAN 不影响其他命令
redis-cli --scan --pattern 'session:*'
```

> **口诀**：**KEYS / FLUSHDB / FLUSHALL / 大集合的 *GETALL 都是定时炸弹**。生产必须禁用 + 用 SCAN 替代 + slowlog 告警三件套。

---

### 案例 ⑧ Hot key —— "直播间点赞打爆单 shard"

#### 现象

某顶流主播开播，瞬时点赞 QPS 100 万。所有点赞计数走 `INCR live:room:88912:likes`。该 key 落在 cluster 的某一个 shard，CPU 100%、客户端大量超时。加 10 个从库做读分摊**没用**，因为 INCR 是写。

#### 根因

Redis Cluster 的负载均衡是 slot 级别的：

- 16384 个 slot 平均分给 N 个 shard
- 每个 key 的 slot 由 CRC16 决定
- **同一个 key 永远落在同一个 shard**，无法分散

读写分离（一主多从）能分担**读流量**，对**写流量无效**：

- INCR 是写命令，所有写都必须发到 master
- 即使 GET 走了 10 个从库，master 仍要承受 100 万 QPS 写

#### 修复

**方案 A：拆 key（最有效）**

把一个 hot key 拆成 N 个：

```python
# 错：单 key
INCR live:room:88912:likes

# 对：拆 100 份
shard = random.randint(0, 99)
INCR live:room:88912:likes:{shard}

# 读取时聚合
total = sum(GET live:room:88912:likes:{i} for i in range(100))
```

加 `{room_id}` tag 让 100 个分片落在同一个 slot 的话，**读取还是落到一个 shard**（聚合性能差）；不加 tag 让分片分布到 100 个 slot → 读时跨 shard 聚合。两种都比单 key 强。

**方案 B：客户端本地聚合**

```python
# 客户端本地累加，每秒批量同步到 Redis
local_counter = 0

def on_like():
    global local_counter
    local_counter += 1

# 后台线程：每秒一次
def flush():
    redis.incrby("live:room:88912:likes", local_counter)
    local_counter = 0
```

代价：丢失精确的实时性（不影响最终值）。100 个客户端各本地累加 → Redis 只承受 100 QPS。

**方案 C：用 Redis Cluster 的"读副本"+ 接受最终一致**

```bash
# 客户端连从库做读
redis-cli READONLY                     # 在从库连接上开启 READONLY
GET live:room:88912:likes              # 直接从从库读
```

只对**读多写少**有效。点赞场景写仍是瓶颈，读副本帮不上。

**方案 D：用专门的 counter 服务**

阿里云 Tair 的 ExStrAtomic、Redis 的 `CRDT counter`（企业版）等支持"分布式计数器"。

**方案 E：架构层 —— 消息队列 + 异步聚合**

```
客户端 → Kafka → consumer → batch-aggregate → Redis
```

Kafka 承受 100 万 QPS 没问题，consumer 每秒批量 INCR 几百次到 Redis。

#### 验证

```bash
# 找 hot key
redis-cli --hotkeys -i 0.1
# 或者 4.0+ 用 LFU 模式
CONFIG SET maxmemory-policy allkeys-lfu
redis-cli --hotkeys

# 模拟峰值压测
redis-benchmark -t incr -n 1000000 -c 100 -k 1 -h <shard_master>
# -k 1 表示 keep alive，模拟 hot key
```

修复前 QPS 上限被单 shard 锁住，修复后吞吐随分片数线性增长。

> **口诀**：**hot key 是 Redis Cluster 唯一无解的天花板**（在不拆 key 的前提下）。读副本对写无效，加机器对单 key 无效。**只能拆 key + 客户端本地聚合**。

---

### 案例 ⑨ 过期键删除策略 —— "TTL 到期但内存不降"

#### 现象

业务大量设置 TTL=300s 的 key，监控显示设置后 5 分钟过去，`used_memory` 几乎不降，反而偶发突然下降一大块。

#### 根因

Redis 的过期键删除有两种策略，**同时工作**：

1. **惰性删除（Lazy）**：访问 key 时检查 TTL，过期就删除
2. **定期删除（Active）**：后台周期性扫描，删除过期 key

惰性删除的副作用：**没人访问的过期 key 永远不删** → 内存占着。

定期删除的算法（`activeExpireCycle`）：

```c
// src/expire.c (简化)
void activeExpireCycle(int type) {
    // 每个 DB 循环
    for (int db = 0; db < server.dbnum; db++) {
        do {
            int expired = 0;
            int sampled = 0;
            
            // 每轮采样 20 个 key
            for (int i = 0; i < ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP; i++) {
                // 随机选一个有 TTL 的 key
                dictEntry *de = dictGetRandomKey(db->expires);
                if (activeExpireCycleTryExpire(db, de, now)) expired++;
                sampled++;
            }
            
            // 如果"过期比例"高 (> 25%)，继续采样；否则跳出
        } while (expired * 100 / sampled > 25 && time_used < timelimit);
    }
}
```

关键参数：

- 每次采样 20 个 key（`ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP`）
- 如果 20 个里超过 25% 已过期，继续下一轮
- 否则 100ms 后再来
- 总 CPU 时间不能超过 `hz` 配置（默认 10 → 每 100ms 一轮，每轮最多用 25ms）

**算法的盲区**：随机采样 + 25% 阈值 → 如果你设了 100 万个 key 同一时间过期，会陷入"每次都有 100% 命中"，CPU 飙升 → 触发 25ms 限制后退出 → 下一轮再来。**结果**：100 万个过期 key 要好几秒才被全删。

那个"突然下降一大块"就是某轮活跃删除终于扫到大量过期 key。

#### 修复

**第一步：避免大量 key 同时过期**

```python
# 错：所有 key 同一时间过期
EXPIRE key 300

# 对：加随机抖动
EXPIRE key (300 + random.randint(0, 60))
```

**第二步：调高 hz**

```ini
hz 100              # 默认 10，调到 100 让定期删除更频繁
```

代价：CPU 用量上升。生产经验：50~100 是合理上限。

**第三步：开 lazyfree-lazy-expired**

```ini
lazyfree-lazy-expired yes      # 过期 key 异步释放（4.0+）
```

让过期删除走后台线程释放内存，避免主线程被大 value 删除卡住。

**第四步：理解 INFO 里的指标**

```bash
redis-cli INFO stats | grep expired
```

```
expired_keys:1234567                 # 累计过期删除的 key 数
expired_stale_perc:1.5               # 过期但还没被删的 key 比例（采样）
expired_time_cap_reached_count:42    # 因 25ms 限制退出的次数
```

`expired_stale_perc > 5%` 说明定期删除跟不上，需要调 hz 或检查业务。

#### 验证

```bash
# 模拟批量过期
for i in {1..1000000}; do
    redis-cli SET k:$i 1 EX 60
done
# 等 70 秒后看 used_memory 下降速度
```

> **口诀**：Redis 过期删除是 **惰性 + 定期** 双策略。**永远不要让大量 key 同时过期** —— 加随机抖动。监控 `expired_stale_perc`。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 数据结构层

- [ ] 任何 Hash / Set / Zset / List 设计前算清峰值元素数，不超过 listpack 阈值（128/512）
- [ ] 单 key 大小 < 10KB（`MEMORY USAGE` 检查）
- [ ] String value < 1MB（`CONFIG GET proto-max-bulk-len` 默认 512MB，但生产远低于这个）
- [ ] 集合超过 5000 元素必须分桶
- [ ] 不用 KEYS / FLUSHDB / FLUSHALL，全部 rename 或 ACL 禁用

### 命令层

- [ ] 大集合操作用 SCAN / HSCAN / SSCAN / ZSCAN，不用 *GETALL / SMEMBERS / LRANGE 0 -1
- [ ] 删除 bigkey 用 UNLINK 不用 DEL
- [ ] Pipeline 用于批量场景（10 条以上）
- [ ] Lua 脚本入参严格校验、循环必须有绝对上限、`lua-time-limit` 调小（< 500ms）

### 持久化层

- [ ] 选定持久化策略：金融级 `appendfsync always`，普通 `everysec`，缓存型 `no`
- [ ] 开启混合持久化 `aof-use-rdb-preamble yes`
- [ ] 监控 `latest_fork_usec`，> 200ms 告警
- [ ] 监控 `aof_delayed_fsync`，> 0 告警
- [ ] 设 `vm.overcommit_memory = 1`
- [ ] 物理内存 ≥ maxmemory × 2 + 系统开销

### 复制层

- [ ] master 配 `min-replicas-to-write 1` + `min-replicas-max-lag 10`
- [ ] 关键写后用 `WAIT 1 1000` 或 `WAITAOF 1 1 1000`（7.4+）
- [ ] `repl-backlog-size` 至少 256MB（避免主从断连后全量同步）
- [ ] 从库关闭写（`replica-read-only yes`，默认）
- [ ] 监控 `master_link_status` / `master_last_io_seconds_ago`

### Cluster 层

- [ ] 客户端正确实现 ASK / MOVED 协议（不要自己造轮子）
- [ ] 客户端开启定时拓扑刷新（30~60s）
- [ ] 多 key 操作用 `{tag}` 保证同 slot
- [ ] 迁 slot 在低峰期 + 用 --pipeline + 监控进度
- [ ] 监控每个 shard 的 hot key（`redis-cli --hotkeys`）

### 内存与过期

- [ ] 配 maxmemory + maxmemory-policy（推荐 `allkeys-lru` 或 `volatile-lru`）
- [ ] TTL 加随机抖动，避免大批同时过期
- [ ] `lazyfree-lazy-expired yes` / `lazyfree-lazy-eviction yes`
- [ ] 监控 `evicted_keys`、`expired_stale_perc`

### 可观测性

- [ ] slowlog 阈值设 10ms
- [ ] latency monitor 阈值 100ms
- [ ] 接入 redis_exporter / Prometheus
- [ ] 关键指标告警：QPS、connected_clients、used_memory、blocked_clients、slowlog_last_id

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### Redis 7+ 新特性

- **Function**（替代 EVAL/EVALSHA）：服务端注册、ACL 管理、不会被 SCRIPT FLUSH 清掉
- **Sharded Pub/Sub**：Cluster 下的 SPUBLISH，按 slot 分发
- **WAITAOF**（7.4+）：等本地 AOF 落盘 + 从库 ACK，强一致写
- **Active-Active Geo**（社区版用 CRDT，企业版用专用协议）

### 多线程 Redis 替代品

| 实现 | 多线程模型 | 兼容性 |
| --- | --- | --- |
| **KeyDB** | 多线程命令执行 + 共享数据结构（mutex） | Redis 6 协议 |
| **Dragonfly** | 多线程 + shared-nothing + 修改后的数据结构 | Redis 5/6 协议 |
| **Valkey** | Redis 7 fork（许可证之争后），社区维护 | Redis 7 协议 |
| **Redict** | Redis 7 fork，遵循 LGPL | Redis 7 协议 |

KeyDB 和 Dragonfly 在多核机器上吞吐能比 Redis 高 5~10 倍，但**生态成熟度差距大**。生产引入前要做兼容性回归。

### 集群方案演进

- **Twemproxy（nutcracker）**：Twitter 出品，无状态 proxy，分片靠一致性 Hash。**无法在线扩容**
- **Codis**：豌豆荚出品，Group + Proxy 模式，支持在线迁移。**已停止维护**
- **Redis Cluster**：官方方案，gossip + 16384 slot。生产事实标准
- **Redis Enterprise**：商业版，多活 + CRDT + 自动 sharding

### 持久化 & 一致性的边界

- CAP 在 Redis 上的体现：默认 AP（异步复制可用性优先），`min-replicas-to-write` 让你换 CP
- Raft / Paxos 在 KV 上的对应：etcd / Consul（强一致 KV，吞吐远低于 Redis）
- **Redis ≠ 强一致 KV**：用 Redis 实现"分布式锁"必须配合 Redlock 算法（且 Martin Kleppmann 撰文质疑 Redlock 的安全性）

### 内存管理与淘汰算法

- **LRU**：Redis 不是真 LRU，是 Approximated LRU（采样近似）
- **LFU**：Redis 4.0+ 引入，基于访问次数衰减（Logarithmic Counter）
- **W-TinyLFU**（Caffeine 用的）：比 LRU/LFU 命中率更高的现代算法
- **2Q / ARC**：被 W-TinyLFU 超越但仍有教学价值

### 流式数据

- Redis Streams（5.0+）vs Kafka 的边界
  - Streams 优势：低延迟（< 1ms）、ACK 模型类似消费者组、单机简单
  - Kafka 优势：吞吐、持久化、多消费者解耦
  - 不要用 Streams 当 Kafka 用 —— 数据量上去后内存吃不消

### 阿里云 Tair / AWS ElastiCache

- **Tair**：阿里云增强版，多种存储引擎（MDB 内存、RDB 持久内存、SDB 磁盘）
- **ElastiCache**：AWS 托管 Redis，跨 AZ 自动故障转移，有 Cluster 和非 Cluster 两种
- **MemoryDB**：AWS 出的"持久化版 Redis"，多 AZ 强一致 + 副本

---

## 7. 引用

- Redis 7.2 源码：[github.com/redis/redis](https://github.com/redis/redis/tree/7.2)
- Redis Cluster 规范：[Cluster spec](https://redis.io/docs/reference/cluster-spec/)
- Salvatore Sanfilippo 的设计文章合集：[antirez.com](http://antirez.com/news)
- Martin Kleppmann, *How to do distributed locking*（2016）—— Redlock 安全性争论
- 《Redis 设计与实现》—— 黄健宏（基于 Redis 3.0，但底层模型仍适用）
- 《Redis 深度历险》—— 钱文品
- AWS [ElastiCache 最佳实践](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/BestPractices.html)
- 阿里云 [Tair 性能白皮书](https://help.aliyun.com/document_detail/146579.html)
- KeyDB / Dragonfly / Valkey 的官方对比文档

---

> 上一篇综述：[Redis 核心](/article/db/redis) ｜ 配套：[MySQL MVCC 与锁](/article/db/mysql-mvcc) ｜ [线程池生产实战](/article/java/threadpool)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
