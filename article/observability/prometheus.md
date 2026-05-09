# Prometheus 指标设计与查询 —— 从 USE/RED 方法论到 PromQL 实战

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，去 [可观测性主题导览](/article/observability/) 看综述就够了。
> 这里要回答的，是**写指标遇到怪现象、半夜被告警风暴叫起来**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. `rate(http_requests_total[1m])`、`irate(http_requests_total[1m])`、`increase(http_requests_total[1m])` 三者算出来的数差在哪？分别用在什么场景里？为什么 rate 算出来不是整数？
2. Histogram 和 Summary，**哪个能跨实例聚合算总 P99**？为什么另一个不行？给一句源码层面的解释。
3. 业务真实 P99 = 80ms，但 Grafana 显示 P99 一直是 `+Inf` 或者诡异的 10ms 平台。bucket 怎么设才能让 P99 准？不准的时候误差是哪来的？
4. 一个服务 OOM 重启后 counter 从 100000 回到 0，下一次 scrape 算出 `rate(...)` 是负数吗？Prometheus 怎么处理 counter reset？
5. 业务 PM 说"每个用户的请求量我都想看"，于是把 `userId` 加进了 label。第二天 Prometheus 占了 100GB 内存。机理是什么？为什么 label cardinality（基数）是 Prometheus 的命门？
6. Recording Rule 和 Alerting Rule 都跑在同一个 Prometheus 进程里，**它们的执行模型有什么差别**？为什么 Recording Rule 能让查询变快但 Alerting Rule 不能？
7. 你的服务在三个 IDC 部署，Prometheus 拉不到对端机房的指标。Pull 模型的"防火墙穿透"和"短任务汇总"分别用什么方案？
8. Native Histogram 是不是可以直接替代经典 Histogram？升级前需要注意什么？

如果有半数答不上，往下看。

---

## 1. 底层模型

绝大多数人对 Prometheus 的认知止于 "PromQL + 4 种 metric 类型 + 拉模型"。但这层抽象解释不了上面 8 个问题。要看懂它们，必须下沉到 Prometheus 的数据模型与 TSDB 实现。

### 1.1 Pull vs Push 的设计哲学

Prometheus 用 **Pull**（拉）模型，即由 Prometheus 主动去 target 的 `/metrics` 端点抓取。这点和 InfluxDB / OpenTSDB / DataDog 等 push 系都不同。

Brian Brazil（Prometheus 核心作者）在 *Prometheus: Up & Running* 里给出 4 个核心理由：

- **Health check 副作用**：能拉到 = target 活着。Push 模型下"指标停了"和"target 死了"无法区分。
- **集中配置**：scrape 配置在 Prometheus 一处，target 不需要知道"我该把指标推到哪"。
- **避免推送方过载 collector**：push 模型下，几千个 target 同时狂推会把 collector 打爆，且不容易限流。
- **服务发现自然集成**：动态 target（k8s pod、consul service）天然适配 pull —— 服务上线被发现 → 自动开始拉。

代价：
- **短任务**（cron job、批处理）跑完就退出，pull 抓不到 → 必须用 Push Gateway
- **跨网络隔离**（机房、防火墙）pull 不通 → 用 Federation 或 Pushprox 等隧道方案

后面案例 ⑥ 会展开 Push Gateway 的滥用问题。

### 1.2 数据模型：metric_name + label set + value + timestamp

每条采样（sample）的物理形态：

```
http_requests_total{method="POST", handler="/api/v1/login", status="200"}  4892  1730000000000
└────────┬────────┘└──────────────────┬──────────────────────────────────┘└─┬─┘└──────┬──────┘
   metric_name              label set（决定唯一时序）                     value     timestamp(ms)
```

关键认知：**`metric_name` + `label set` 唯一标识一条"时间序列"**（time series）。换一个 label 值就是另一条独立的序列。

```
http_requests_total{method="GET", status="200"}    → 序列 A
http_requests_total{method="POST", status="200"}   → 序列 B
http_requests_total{method="GET", status="500"}    → 序列 C
```

是 3 条独立的时序，各自有独立的 chunk、独立的索引项、独立的内存占用。

这就是为什么 **label 基数（cardinality）是 Prometheus 的命门**。后面 1.5 和案例 ① 会展开。

### 1.3 时序的"时间序列"概念

一条时序在物理存储上是一串 (timestamp, value) 对，按时间排序。Prometheus 默认 `scrape_interval=15s`，意味着每条时序每 15 秒新增一个采样点。

```
http_requests_total{method="GET"}
  t=10s   v=4001
  t=25s   v=4015
  t=40s   v=4032
  t=55s   v=4051
  ...
```

**注意**：value 是浮点数（float64），即使是 counter 的"计数"，存储和传输也都是浮点。这就是为什么 `rate()` 算出来通常不是整数 —— 它是 (Δvalue / Δtime) 的浮点除法。

### 1.4 4 种 metric 类型

| 类型 | 语义 | 单调性 | 典型用法 |
| --- | --- | --- | --- |
| **Counter** | 累计量，只增不减（reset 除外） | 单调递增 | 请求总数、错误总数、字节数 |
| **Gauge** | 瞬时量，可增可减 | 任意 | 内存占用、连接数、温度 |
| **Histogram** | 分桶累计计数 + sum + count | 各 bucket 都是 counter | 请求延迟、响应大小 |
| **Summary** | 客户端预算的分位数 + sum + count | quantile 是 gauge | 同上，但不能跨实例聚合 |

Counter / Gauge 是"原子类型"。Histogram / Summary 实际上是**一组**时序的组合：

```
# Histogram 暴露的是这一组：
http_request_duration_seconds_bucket{le="0.1"}     45000
http_request_duration_seconds_bucket{le="0.5"}     48000
http_request_duration_seconds_bucket{le="1.0"}     49500
http_request_duration_seconds_bucket{le="+Inf"}    50000
http_request_duration_seconds_sum                   2340.5
http_request_duration_seconds_count                 50000
```

`_bucket` 是**累计**计数（≤ 0.1 秒的请求数 = 45000，含所有更小的桶）。这点很多人搞错。后面 2.4 会从源码层面解释为什么必须累计。

### 1.5 TSDB 存储结构：block / chunk / WAL / mmap

Prometheus 自带 TSDB（v2 起），数据组织成"分层"结构：

```
data/
├── 01HXY8K...         ← Block 1（已压缩持久化，2 小时窗口）
│   ├── chunks/
│   │   ├── 000001
│   │   └── 000002
│   ├── index          ← 倒排索引（label → series）
│   ├── meta.json
│   └── tombstones
├── 01HXY9...          ← Block 2
├── chunks_head/       ← Head Block 的 mmap chunk 文件
├── wal/               ← Write-Ahead Log
│   ├── 000001
│   └── 000002
└── lock
```

关键概念：

- **Head Block**（内存）：当前正在写入的数据，最新 2 小时的样本驻留在内存（实际是 mmap）。
- **Persistent Block**（磁盘）：每 2 小时 Head 会被压缩落盘成一个 Block。Block 不可变。
- **Chunk**：一段连续时间的样本压缩在一起的最小单元。一条时序的数据被切成多个 chunk。
- **WAL**：所有写入先进 WAL，用于崩溃恢复。Head 重建依赖 WAL 重放。
- **Compaction**：旧 Block 会被合并（2h → 6h → 24h），减少索引开销。

**为什么用 mmap 而不是普通文件 IO？** chunk 文件是只读的，mmap 能把"按 chunkID 随机访问"做成 O(1) 指针解引用，且 page cache 由 OS 管。Prometheus 2.6 之后 head chunk 也 mmap 出去，进一步减少进程 RSS。

每条时序在内存里维护一个 "active chunk"（当前在写），写满 120 个样本（默认）或时间窗口超 2 小时就 flush 成 mmap chunk。

### 1.6 PromQL 的执行模型

PromQL 的四种返回类型：

| 类型 | 含义 | 例子 |
| --- | --- | --- |
| **Instant Vector** | 一组时序，每条一个**当前**值 | `http_requests_total` |
| **Range Vector** | 一组时序，每条一段**时间窗口内的所有值** | `http_requests_total[5m]` |
| **Scalar** | 单个浮点 | `42` 或 `scalar(...)` 转换 |
| **String** | 字符串（用得极少） | — |

PromQL 的执行规则有两条非常反直觉但必须记住的：

1. **几乎所有"速率类"函数（rate、irate、increase、delta、deriv）的输入必须是 Range Vector**。`rate(http_requests_total)` 是错的，必须 `rate(http_requests_total[1m])`。
2. **`rate()` 的窗口至少要包含 4 个样本**才稳定。窗口太小会 NaN，窗口太大会失去时效性。后面 2.3 会展开。

PromQL 查询是一个 **step-driven** 的迭代：Grafana 给一个 `[start, end, step]`，引擎在每个 step 时间点计算一次表达式，得到一个 Instant Vector。`rate(...[5m])` 在每个 step 都看回过去 5 分钟的窗口。

> **口诀**：Pull = 健康检查副作用 + 集中配置；时序 = metric_name + label set；4 种类型；TSDB 分 Head/Block/WAL/mmap；PromQL 区分 Instant 和 Range。

---

## 2. 源码切片

> 基于 Prometheus 2.x（截至 2.50） 源码。注释为本文添加。

### 2.1 Storage 层：Head Block + Persistent Block 的分层

```go
// tsdb/db.go 简化
type DB struct {
    dir    string
    locker *tsdbutil.DirLocker

    blocks []*Block         // 已落盘的 Persistent Block
    head   *Head            // Head Block（内存 + mmap chunks_head）

    compactor Compactor     // 负责 Block 合并
    // ...
}

func (db *DB) Appender(ctx context.Context) storage.Appender {
    // 写入永远进 Head
    return db.head.Appender(ctx)
}

func (db *DB) Querier(mint, maxt int64) (storage.Querier, error) {
    // 查询根据时间范围合并 Head + 相关 Block
    blocks := []BlockReader{}
    for _, b := range db.blocks {
        if b.OverlapsClosedInterval(mint, maxt) {
            blocks = append(blocks, b)
        }
    }
    if maxt >= db.head.MinTime() {
        blocks = append(blocks, NewRangeHead(db.head, mint, maxt))
    }
    return NewBlockQuerier(blocks, mint, maxt)
}
```

**逐段点透**：

- **写永远进 Head**：所有 Appender 都最终 append 到 Head 的内存结构。这是为什么 Prometheus 写入吞吐能上百万 sample/s —— 没有跨 Block 的写入路径。
- **读会跨 Block 合并**：查询如果跨越 2h 边界，会同时读 Head 和最近的 Persistent Block。这是为什么"刚好压缩前后"的查询会偶发慢一点 —— 可能要打开多个 chunk 文件。
- **Block 不可变**：Persistent Block 落盘后只读。变更（删除时序、修改）通过 tombstone 文件标记，等下一次 compaction 物理清除。

### 2.2 Chunk 编码：XOR / DoubleDelta / Gorilla 压缩

Prometheus 用 Facebook **Gorilla** 论文（2015）的压缩算法。核心思想：时序数据相邻样本差距小，可用差分 + 位级编码极致压缩。

```go
// tsdb/chunkenc/xor.go 简化
func (a *xorAppender) Append(t int64, v float64) {
    var tDelta uint64
    num := binary.BigEndian.Uint16(a.b.bytes())

    if num == 0 {
        // 第一个样本：写完整的 t 和 v
        buf := make([]byte, binary.MaxVarintLen64)
        a.b.writeBits(uint64(t), 64)
        a.b.writeBits(math.Float64bits(v), 64)
    } else if num == 1 {
        // 第二个样本：写 timestamp 的 delta（变长 varint）
        tDelta = uint64(t - a.t)
        a.b.writeVarint(int64(tDelta))
        a.writeVDelta(v)         // value 走 XOR 差分
    } else {
        // 第三个起：写 delta-of-delta
        tDelta = uint64(t - a.t)
        dod := int64(tDelta - a.tDelta)
        switch {
        case dod == 0:
            a.b.writeBit(zero)        // 0 个 bit！
        case bitRange(dod, 14):
            a.b.writeBits(0x02, 2)    // 14 bit
            a.b.writeBits(uint64(dod), 14)
        case bitRange(dod, 17):
            a.b.writeBits(0x06, 3)    // 17 bit
            // ...
        }
        a.writeVDelta(v)
    }
    a.t = t
    a.tDelta = tDelta
}
```

**逐段点透**：

- **Timestamp 用 delta-of-delta**：Prometheus 的 scrape 间隔是固定的（15s），所以相邻 timestamp 差永远是 15000。**delta = 15000，dod = 0**，只用 1 bit 就能编码！这是为什么 Prometheus 能把 16 字节的 (int64, float64) 压到平均 1.37 字节/样本。
- **Value 用 XOR**：浮点数相邻值通常前几个 byte 相同（`7041.5` 和 `7045.2` 的 IEEE 754 表示前缀几乎一致）。XOR 后取前导/尾随 0 数量，只编码"中间变化的位"。
- **代价**：chunk 是只读的，append-only。要修改一个历史样本，必须重写整个 chunk。这是为什么 TSDB 不支持"修改历史数据"。

### 2.3 rate() / irate() / increase() 的内部实现差别

```go
// promql/functions.go 简化
func funcRate(vals []parser.Value, args parser.Expressions, enh *EvalNodeHelper) Vector {
    return extrapolatedRate(vals, args, enh, true /*isCounter*/, true /*isRate*/)
}

func funcIncrease(vals []parser.Value, args parser.Expressions, enh *EvalNodeHelper) Vector {
    return extrapolatedRate(vals, args, enh, true, false /*isRate=false → 不除 duration*/)
}

func extrapolatedRate(vals []parser.Value, args parser.Expressions, enh *EvalNodeHelper,
                     isCounter, isRate bool) Vector {
    ms := args[0].(*parser.MatrixSelector)
    rangeStart := enh.Ts - durationMilliseconds(ms.Range+vs.Offset)
    rangeEnd := enh.Ts - durationMilliseconds(vs.Offset)

    samples := vals[0].(Matrix)[0]
    if len(samples.Floats) < 2 { return enh.Out }

    var resultValue float64
    firstT := samples.Floats[0].T
    lastT := samples.Floats[len(samples.Floats)-1].T
    resultValue = samples.Floats[len(samples.Floats)-1].F - samples.Floats[0].F

    // ★ counter reset 检测
    if isCounter {
        prev := samples.Floats[0].F
        for _, sample := range samples.Floats[1:] {
            if sample.F < prev {
                resultValue += prev   // 把 reset 之前的累计值补回来
            }
            prev = sample.F
        }
    }

    // ★ extrapolation：把窗口边缘的"半采样间隔"外推到完整窗口
    durationToStart := float64(firstT-rangeStart) / 1000
    durationToEnd := float64(rangeEnd-lastT) / 1000
    sampledInterval := float64(lastT-firstT) / 1000
    averageDurationBetweenSamples := sampledInterval / float64(len(samples.Floats)-1)

    if durationToStart < averageDurationBetweenSamples/2 {
        // ... 外推逻辑
    }
    // ...

    if isRate {
        resultValue /= ms.Range.Seconds()    // ★ 除以窗口秒数 → "每秒变化"
    }

    return append(enh.Out, Sample{F: resultValue})
}
```

**逐段点透**：

- **`rate()` 和 `increase()` 共用底层函数**，差别只在最后是否除以窗口秒数。所以 `increase(x[5m]) == rate(x[5m]) * 300`。两者完全等价，只是 unit 不同。
- **Counter reset 检测**：`if sample.F < prev`，发现下降就认为是 reset，把 reset 前的累计值加回来。这是为什么 OOM 重启不会让 rate 算出负数（参见案例 ④ 的真实陷阱）。
- **Extrapolation（外推）**：窗口边缘的样本通常不刚好落在 `[start, end]` 上。算法把第一个和最后一个样本"外推"到完整窗口的边界，避免边缘损失。代价：rate 算出来不会等于"两个端点的差值/秒"，会有小偏差。

`irate()` 完全不一样：

```go
// 简化版
func funcIrate(vals []parser.Value, args parser.Expressions, enh *EvalNodeHelper) Vector {
    samples := vals[0].(Matrix)[0]
    if len(samples.Floats) < 2 { return enh.Out }

    // ★ 只看最后两个样本！
    lastSample := samples.Floats[len(samples.Floats)-1]
    previousSample := samples.Floats[len(samples.Floats)-2]

    var resultValue float64
    if lastSample.F < previousSample.F {
        resultValue = lastSample.F            // counter reset
    } else {
        resultValue = lastSample.F - previousSample.F
    }
    sampledInterval := lastSample.T - previousSample.T
    return append(enh.Out, Sample{F: resultValue * 1000 / float64(sampledInterval)})
}
```

**`irate()` 只看窗口里最后两个样本**。这意味着：

- **更敏感**（瞬时尖刺立即看到），但**更容易抖动**
- 用在告警里几乎肯定误报（一个尖刺就触发）
- 适合**用在仪表盘的实时图表**，不适合用在告警和 SLO 计算

| 函数 | 用途 | 适用场景 |
| --- | --- | --- |
| `rate(x[5m])` | 平滑的"每秒变化" | **告警 / SLO**，绝大多数生产用 rate |
| `irate(x[1m])` | 最近瞬时变化 | **实时仪表盘**，看尖刺 |
| `increase(x[1h])` | 这段时间累计增量 | 容量规划、报表 |

后面案例 ② 会展开"窗口选错"的事故。

### 2.4 Histogram 的 bucket 累积计数 vs Summary 的 quantile 估计

Histogram 在客户端只记**计数**：每来一个样本，落到对应桶的 counter +1。

```go
// client_golang/prometheus/histogram.go 简化
func (h *histogram) Observe(v float64) {
    n := atomic.AddUint64(&h.countAndHotIdx, 1)
    hotCounts := h.counts[n>>63]
    // 找到第一个 le >= v 的 bucket（二分）
    i := sort.SearchFloat64s(h.upperBounds, v)
    if i < len(h.upperBounds) {
        atomic.AddUint64(&hotCounts.buckets[i], 1)
    }
    atomic.AddUint64(&hotCounts.count, 1)
    // sum 用 CAS 累加
    for {
        oldBits := atomic.LoadUint64(&hotCounts.sumBits)
        newBits := math.Float64bits(math.Float64frombits(oldBits) + v)
        if atomic.CompareAndSwapUint64(&hotCounts.sumBits, oldBits, newBits) {
            break
        }
    }
}
```

**关键观察**：

- 客户端**完全不算 quantile**，只暴露计数
- bucket 是**累计**的（`le="0.1"` 包含所有 ≤ 0.1 的，`le="0.5"` 包含 ≤ 0.5 的所有）

为什么累计？因为可加：把多实例的 bucket 直接相加，仍然是合法的 histogram。这是 Histogram **能跨实例聚合 P99** 的根本原因：

```promql
# 跨所有实例的 P99 延迟
histogram_quantile(0.99,
  sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
)
```

`sum by (le)` 把所有实例的同 bucket 加起来，得到全局 histogram，再用 `histogram_quantile` 估计 P99。

Summary 完全相反：

```go
// client_golang/prometheus/summary.go 简化
type summary struct {
    streams []*quantile.Stream     // ★ 客户端预算的滑动窗口分位数
    sum     float64
    count   uint64
}

func (s *summary) Observe(v float64) {
    s.streams[s.hotBufMtx].Insert(v)   // 用 t-digest 类算法实时维护
    // ...
}
```

Summary 在客户端用 [Cormode-Korn streaming quantile] 算法实时维护 quantile（如 0.5/0.9/0.99）。暴露的格式：

```
http_request_duration_seconds{quantile="0.5"}     0.012
http_request_duration_seconds{quantile="0.9"}     0.045
http_request_duration_seconds{quantile="0.99"}    0.180
http_request_duration_seconds_sum                  234.5
http_request_duration_seconds_count                10000
```

**致命问题**：quantile 不可加。

```promql
# ❌ 错误：把多个实例的 P99 加起来
sum(http_request_duration_seconds{quantile="0.99"})
# 这个数学上没有意义。三个实例 P99 分别是 100ms / 200ms / 150ms，
# 全局 P99 不是 450ms，也不是 150ms 平均。无法从局部 quantile 还原全局。
```

后面案例 ⑤ 会展开真实事故。

### 2.5 Recording Rule 的执行：怎么避免重复计算

```yaml
# rules.yml
groups:
  - name: api_perf
    interval: 30s
    rules:
      - record: instance:http_requests:rate1m
        expr: rate(http_requests_total[1m])

      - record: cluster:http_requests:rate1m
        expr: sum(instance:http_requests:rate1m)    # ★ 引用上面的 rule

      - alert: HighErrorRate
        expr: cluster:http_requests:rate1m > 1000
        for: 5m
```

源码里 rule group 的执行：

```go
// rules/group.go 简化
func (g *Group) Eval(ctx context.Context, ts time.Time) {
    for i, rule := range g.rules {
        // ★ 同一组内顺序执行，前一个 rule 的结果立即可见
        vector, err := rule.Eval(ctx, ts, g.opts.QueryFunc, g.opts.ExternalURL, g.GetOpts().Limit)
        if err != nil { /* ... */ continue }

        if rr, ok := rule.(*RecordingRule); ok {
            // 把结果直接写回 TSDB
            for _, s := range vector {
                app.Append(0, s.Metric, s.T, s.F)
            }
        } else if ar, ok := rule.(*AlertingRule); ok {
            // 触发告警评估
            ar.sendAlerts(ctx, ts, ...)
        }
    }
    app.Commit()
}
```

**逐段点透**：

- **同组内顺序执行**：`cluster:http_requests:rate1m` 引用了 `instance:http_requests:rate1m`，因为它们在同一组里且后者排在前面，所以查询时能拿到本次刚算出的值。如果跨组，引用的是上一次评估的结果（可能是 30s 前）。
- **Recording Rule 的结果直接 append 到 TSDB**：等于"预聚合"。下次 Grafana 查询 `cluster:http_requests:rate1m` 就直接读这条预聚合时序，不再实时算。
- **Alerting Rule 不写 TSDB**（除了 `ALERTS` 这条特殊时序），它的执行模型是"算表达式 → 看结果 → 触发/抑制告警"，不预聚合。

为什么 Recording Rule 能让查询变快？因为把"扫描 1000 条原始时序 + 聚合"的开销从"每次查询都做"变成"每 30s 后台做一次"。Grafana 几十个仪表盘看同一聚合时省了大量 CPU。

但 Alerting Rule 不能这样优化 —— 它的本质是"决策"，每次都得算。

### 2.6 Remote Write 协议（snappy + protobuf）

Prometheus 自身存储是单机的（最多 ~30TB / 几亿活跃序列）。要做长期/大规模存储就用 Remote Write 把数据推到 VictoriaMetrics / Cortex / Mimir / Thanos。

协议格式：

```protobuf
// prompb/types.proto
message WriteRequest {
  repeated TimeSeries timeseries = 1;
  repeated MetricMetadata metadata = 3;
}

message TimeSeries {
  repeated Label labels = 1;
  repeated Sample samples = 2;
  repeated Exemplar exemplars = 3;
  repeated Histogram histograms = 4;
}

message Sample {
  double value = 1;
  int64 timestamp = 2;
}
```

发送路径：

```go
// storage/remote/queue_manager.go 简化
func (s *shards) sendBatch(samples []prompb.TimeSeries) {
    req, _, err := buildWriteRequest(samples, nil, pBuf, &buf)
    // ★ snappy 压缩
    compressed := snappy.Encode(nil, req)
    httpReq, err := http.NewRequest("POST", s.url, bytes.NewReader(compressed))
    httpReq.Header.Set("Content-Encoding", "snappy")
    httpReq.Header.Set("Content-Type", "application/x-protobuf")
    httpReq.Header.Set("X-Prometheus-Remote-Write-Version", "0.1.0")
    // ... 发送 + 失败重试 + 退避
}
```

**逐段点透**：

- **Snappy 压缩**比 gzip 快 5~10 倍，CPU 开销小，压缩率略低（30%~50%），是 Prometheus 流式发送的最佳折中。
- **失败有指数退避**，但 WAL 不无限保留：默认 2 小时窗口。如果远端长时间挂掉，超过窗口的样本会丢。生产配 `wal_compression: true` + 设大 `max_samples_per_send`、`min_shards`、`max_shards`。
- **每条样本 16 字节 + label set**：高基数下 Remote Write 流量爆炸。100 万活跃序列 × 4 字节平均压缩后样本 + label set protobuf ≈ 几十 MB/s 持续带宽。

### 2.7 PromQL 执行器的 step 计算

Grafana 给查询的参数：`query=...&start=T1&end=T2&step=15s`。引擎在 `[T1, T2]` 间每隔 15s 算一次表达式。

```go
// promql/engine.go 简化
func (ng *Engine) exec(ctx context.Context, q *query) (parser.Value, ...) {
    if q.opts.Step == 0 {
        // Instant query
        return ev.Eval(ctx, q.expr)
    }
    // Range query: 在每个 step 算一次
    matrix := Matrix{}
    for ts := q.opts.Start; !ts.After(q.opts.End); ts = ts.Add(q.opts.Step) {
        ev.startTimestamp = ts.UnixMilli()
        result, _ := ev.Eval(ctx, q.expr)
        // ... 把结果按时间累积成 Matrix
    }
    return matrix
}
```

**逐段点透**：

- **step 不能太小**：比如 `step=1s` 时，5 分钟范围要算 300 次表达式，每次都做 `rate(...[5m])` 的扫描。
- **step 与 scrape_interval 应匹配**：scrape 是 15s，step 设 5s 没意义（同一个值算 3 次）。
- **Grafana 默认行为**：根据图表宽度自动算 step，约等于"每像素一个数据点"。这是为什么放大缩小图表会得到不同精度的曲线。

> **口诀**：写永远进 Head；rate 用差分 + 外推 + counter reset 检测；Histogram 累计可加，Summary 客户端算分位数不可加；Recording Rule 预聚合写回 TSDB；Remote Write 是 snappy + protobuf。

---

## 3. 实测数据

> 没数据的"性能更好"都是耍流氓。下面给一组在生产环境（或本地复刻）能跑出来的对比。

### 3.1 高基数对内存的实测影响

实验：往同一个 Prometheus 实例（4 核 / 8GB）塞不同基数的指标，观察 RSS。

```
metric: api_request_count{method, path, user_id}
- method   ∈ 5 个值
- path     ∈ 50 个值
- user_id  ∈ N 个值

总活跃序列 = 5 × 50 × N = 250 × N
```

| user_id 基数 N | 活跃序列数 | Prometheus RSS | 单序列内存（约） |
| --- | --- | --- | --- |
| 100 | 25,000 | 280 MB | 11 KB |
| 1,000 | 250,000 | 1.6 GB | 6.4 KB |
| 10,000 | 2,500,000 | 9.8 GB（OOM 边缘） | 4 KB |
| 100,000 | 25,000,000 | OOM | — |

经验公式：每个活跃序列约需 **3~10 KB 内存**（含 head chunk + label 倒排索引 + WAL 缓冲）。

**结论**：
- 一个常规 Prometheus 实例可承载 100 万 ~ 500 万活跃序列
- 超过 1000 万就要分片或换 VictoriaMetrics / Mimir
- **永远不要把请求级标识符（userId / requestId / sessionId）放进 label**

### 3.2 不同 scrape_interval 对成本的影响

同一组 target（500 个实例，每实例约 200 时序），改变 `scrape_interval`：

| scrape_interval | 写入 sample/s | CPU | 30 天磁盘占用 |
| --- | --- | --- | --- |
| 5s | 20,000 | 35% | 76 GB |
| 15s（默认） | 6,667 | 12% | 26 GB |
| 60s | 1,667 | 3% | 7 GB |

**注意**：scrape_interval 是 Prometheus 时序的"采样精度"。如果你的 SLO 要求"5 分钟内发现错误率突变"，scrape_interval=15s 完全够；如果要做容量趋势分析，scrape_interval=60s 也行。**没必要默认 5s**。

但配套：rate 窗口必须 ≥ 4 × scrape_interval。`rate(...[1m])` + scrape=15s = 4 个样本 = 边缘安全。`rate(...[1m])` + scrape=30s = 2 个样本 = NaN 风险。

### 3.3 Recording Rule vs 实时查询

实验：1 个 PromQL 表达式，原始时序 5000 条 + 聚合到 50 条。

```promql
# 原始查询
sum by (service) (rate(http_requests_total[5m]))

# 预聚合后查询
service:http_requests:rate5m
```

| 查询类型 | 单次延迟 | Grafana 仪表盘加载 30 个 panel |
| --- | --- | --- |
| 实时算 | 280 ms | 8.2 s |
| Recording Rule | 18 ms | 540 ms |

**口诀化**：仪表盘里反复出现的聚合表达式，**全部预聚合**。告警表达式也可以预聚合（先算 rate，再用预聚合时序判断阈值），但要注意 Alertmanager 评估周期与 group interval 的匹配。

### 3.4 Histogram bucket 配置与 P99 误差

模拟一个真实分布的延迟（lognormal，均值 50ms，p99 ≈ 200ms），用 5000 个样本测不同 bucket 配置下 `histogram_quantile(0.99, ...)` 的误差。

| bucket 配置 | 计算出的 P99 | 实际 P99 | 误差 |
| --- | --- | --- | --- |
| 默认（0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10） | 250 ms（线性插值在 0.25 桶内） | 200 ms | +25% |
| 业务专配（0.01, 0.05, 0.1, 0.15, 0.2, 0.3, 0.5, 1） | 215 ms | 200 ms | +7.5% |
| 业务专配密集（0.05, 0.1, 0.15, 0.18, 0.2, 0.22, 0.25, 0.3, 0.5） | 203 ms | 200 ms | +1.5% |
| 业务上限太低（0.001, 0.005, 0.01, 0.02, 0.05） | **+Inf** | 200 ms | 无意义 |

**关键认知**：
- `histogram_quantile` 在桶内**线性插值**，桶越宽，插值误差越大
- **必须在你关心的分位数附近放密集桶**。要看 P99 ≈ 200ms 就在 200ms 附近多放几个桶
- **`+Inf` 桶**不是装饰：所有超过最大上限的样本都落进 `+Inf`。如果 P99 落在 `+Inf` 区间，结果就是 `+Inf`，永远报不出真实值

后面案例 ③ 会展开 bucket 配错的真实事故。

### 3.5 Native Histogram vs 经典 Histogram

Native Histogram（Prometheus 2.40+，2.x 实验性，3.0 GA）改用**指数桶**，自动适应数据分布。

```
经典 Histogram：
- 一个 metric 暴露 N 个时序（每 bucket 一条 + sum + count）
- 假设 12 个 bucket → 14 条时序
- bucket 边界硬编码

Native Histogram：
- 整个 histogram 编码成单条时序（一个 sample 里包含所有桶）
- 桶按 2^(2^-resolution) 倍递增（resolution=8 时桶宽 ~1.0027 倍）
- 自动覆盖几个数量级，不需要预设上限
```

实测对比（同一组 5000 样本）：

| 指标 | 经典（12 bucket） | Native（resolution=8） |
| --- | --- | --- |
| 时序数 | 14 | 1 |
| 客户端内存 | ~1 KB | ~3 KB（动态桶元数据） |
| 服务端单 sample 字节 | 14 × ~2B = 28B | 平均 ~80B |
| P99 计算误差 | 1.5% ~ 25%（取决于 bucket） | < 0.1% |
| 跨实例聚合 | ✅ | ✅ |

**结论**：高基数指标（每 metric 几千个 label 组合）用 Native 能大幅减少时序数；低基数指标差别不大。但 Native 还需要：
- 客户端 SDK 支持（client_golang ≥ 1.16，java/python 还在追赶）
- 可视化端支持（Grafana ≥ 10）
- 远端存储支持（Mimir / VictoriaMetrics 已支持）

迁移要点见案例 ⑨ 和"还能往下挖什么"一节。

> **口诀**：Cardinality 是命门；scrape 与 rate 窗口要匹配；预聚合是仪表盘救星；P99 准不准看 bucket 配置；Native Histogram 是未来。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① userId 进 label —— "Prometheus 几小时内吃掉 100GB"

#### 现象

业务方加了一个新指标"用户 API 调用次数"。代码上线一切正常。两小时后告警：Prometheus 实例 RSS 飙到 90GB，scrape 队列堆积，部分 target 抓取超时，Grafana 整体卡顿。

#### 根因

埋点代码：

```go
// 业务代码
apiRequestCount := prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "api_request_total",
        Help: "Total API requests",
    },
    []string{"method", "path", "user_id"},   // ★ user_id 进了 label
)

func handler(w http.ResponseWriter, r *http.Request) {
    userID := getUserID(r)
    apiRequestCount.WithLabelValues(r.Method, r.URL.Path, userID).Inc()
    // ...
}
```

业务有 200 万 DAU，2 小时内大约 50 万独立用户访问过。每个 (method, path, user_id) 三元组就是一条新时序：

- method ≈ 4 个值
- path ≈ 200 个值
- user_id ≈ 50 万个值

活跃时序 ≈ 4 × 200 × 500,000 = **4 亿**。按每序列 4KB 算 = 1.6 TB 内存需求。Prometheus 撑到 90GB 就 OOM 了。

更糟的是 **TSDB 索引爆炸**：每条新时序要在 label → series 倒排索引里加几十个 entry。索引重建成本是 O(label_value 总数)，几亿 label value 让 head compaction 跑不动。

#### 修复

**铁律**：**永远不要把用户级 / 请求级 / 时间级 标识符放进 label**。

错误模式 → 正确替代：

| 错误 | 正确 |
| --- | --- |
| `userId` | 按用户分组分桶（VIP/普通/免费）作为 label |
| `requestId` / `traceId` | 移到日志或 trace 系统，不要进指标 |
| `timestamp` 当 label | 用真实的 sample timestamp |
| `error_message` 字符串 | 错误码（有限枚举） |
| `path` 含动态段（`/user/12345`） | URL 模板（`/user/:id`） |

修复后的代码：

```go
apiRequestCount := prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "api_request_total",
        Help: "Total API requests",
    },
    []string{"method", "route", "user_tier"},   // tier ∈ {vip, normal, free}
)
```

紧急止血：

```yaml
# 临时 metric_relabel_configs 把这条指标 drop 掉
scrape_configs:
  - job_name: 'app'
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: 'api_request_total'
        action: drop
```

drop 后等 head compaction 把旧序列归档到 Block，再 admin API 删除：

```bash
curl -X POST -g 'http://localhost:9090/api/v1/admin/tsdb/delete_series?match[]=api_request_total'
```

#### 监控防御

提前发现高基数：

```promql
# 哪些 metric 时序最多
topk(10, count by (__name__)({__name__=~".+"}))

# 哪些 job 时序总量最多
topk(10, count by (job) ({__name__=~".+"}))

# 单个 metric 的活跃序列数
prometheus_tsdb_head_series

# WAL 写入速度（异常上涨说明有新基数）
rate(prometheus_tsdb_head_samples_appended_total[5m])
```

告警：

```yaml
- alert: HighSeriesCardinality
  expr: prometheus_tsdb_head_series > 5000000
  for: 10m
```

#### 验证

```bash
# 修复后 1 小时观察 RSS
ps -o rss= -p $(pgrep prometheus)
# 应该回到正常水平（GB 量级）

# 时序数
curl -s http://localhost:9090/api/v1/query?query=prometheus_tsdb_head_series | jq .
```

> **口诀**：**任何唯一标识符都不能进 label**。label value 集合必须是有限的、低频变化的、业务有意义的枚举。

---

### 案例 ② rate 窗口选错 —— "告警延迟 5 分钟，复盘发现窗口配反了"

#### 现象

P0 故障复盘：错误率从 14:00 飙升，告警 14:05 才响。值班同学查告警规则：

```yaml
- alert: HighErrorRate
  expr: rate(http_errors_total[1m]) / rate(http_requests_total[1m]) > 0.05
  for: 2m
```

理论上"5% 错误率持续 2 分钟"应该 14:02 就触发，为什么晚了 3 分钟？

更诡异：测试环境用 `rate(...[30s])` 复现时直接告警 NaN，被研发改成 `rate(...[1m])` 上线。

#### 根因

scrape_interval 配的是 `30s`：

```yaml
scrape_configs:
  - job_name: 'app'
    scrape_interval: 30s
```

`rate(x[1m])` 窗口里只有 **2 个样本**（边缘情况下可能只有 1 个）：

```
窗口 [t-60s, t]
  样本 1: t-30s
  样本 2: t-0s
```

`rate()` 需要至少 2 个样本才能算差分。但 extrapolation（外推）算法在样本数太少时**完全不可靠**：

- 边缘对齐不准 → 外推系数偏差大
- 一个 outlier 样本就把 rate 拉飞
- 样本只有 1 个 → 直接 NaN

更要命的：**告警 `for: 2m` 要求"持续 2 分钟"，每 1 分钟评估一次。一个 NaN/抖动样本就让 for 计时器重置**。最终告警延迟数倍于预期。

源码里 `extrapolatedRate` 检查 `if len(samples.Floats) < 2 { return enh.Out }` —— 不够就直接返回空，PromQL 表达式得到 NaN，告警计时不前进。

#### 修复

**铁律**：rate 窗口至少 = `4 × scrape_interval`，告警场景建议 `>= 5 × scrape_interval`。

| scrape_interval | rate 窗口最小 | 告警建议 |
| --- | --- | --- |
| 5s | 20s | 1m |
| 15s | 1m | 2m~5m |
| 30s | 2m | 5m |
| 60s | 4m | 5m~10m |

修正后的告警：

```yaml
- alert: HighErrorRate
  expr: |
    sum(rate(http_errors_total[5m]))
    /
    sum(rate(http_requests_total[5m]))
    > 0.05
  for: 5m            # ★ for 不能小于 rate 窗口（否则可能 stale）
  labels:
    severity: critical
  annotations:
    summary: "Error rate {{ $value | humanizePercentage }} for 5m"
```

为什么还要 `sum(...)` 包一层？错误率应该是**全局错误数 / 全局请求数**，不能逐实例算 rate 再除（每实例分母可能为 0 → 除零 → NaN/Inf 污染聚合）。先 sum 再除是惯用模式。

#### 推论：irate 永远不要用在告警里

```yaml
# ❌ 几乎肯定误报
- alert: HighErrorRate
  expr: irate(http_errors_total[1m]) > 100
```

irate 只看最后两个样本，一次 GC 抖动就能让一个 burst 触发告警。

#### 验证

把 rate 窗口改对后，故意制造一波错误：

```bash
# 持续 2 分钟错误注入
for i in {1..120}; do curl http://app/error; sleep 1; done
```

观察告警从 14:00:00 注入起，14:05:00 左右触发（5m for + 一些抖动），符合预期。

> **口诀**：`rate window >= 4 × scrape_interval`，告警 `for >= rate window`。**irate 只配仪表盘看实时，不进告警规则**。

---

### 案例 ③ Histogram bucket 全用默认值 —— "P99 永远是 +Inf"

#### 现象

业务接入了 prometheus client，监控 HTTP 请求延迟：

```go
httpDuration := prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name: "http_request_duration_seconds",
        Help: "HTTP request latency",
        // 没传 Buckets → 用默认值
    },
    []string{"method", "route"},
)
```

Grafana 上 `histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))` 永远显示 `+Inf` 或固定 10s 平台。但业务知道实际 P99 = 80~120ms。

#### 根因

client_golang 默认 buckets：

```go
var DefBuckets = []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10}
```

只到 10s。业务接的是个图片处理服务，少量请求确实超过 10s（30~60s）。这些样本全部落进 `+Inf` 桶。

`histogram_quantile(0.99, ...)` 计算逻辑：

```
找到累计计数 >= 99% × total 的桶
如果这个桶是 +Inf → 返回 +Inf
否则在桶内线性插值
```

只要有 > 1% 的样本超过最大上限，P99 就落在 `+Inf` 桶 → 直接返回 `+Inf`。

第二个事故：换了一组业务（普通 API），P99 ≈ 80ms，但 Grafana 上显示 P99 一直是 100ms 平台。原因：

- bucket 边界 0.05 → 0.1 → 0.25
- 80ms 落在 (0.05, 0.1] 桶 → 桶上限是 0.1 → 线性插值算出 80~100ms 之间
- 当 99% 累计正好落在 (0.05, 0.1] 桶内 → P99 ≈ 0.1 × (1 - 残差)

bucket 太稀，**插值精度被桶宽吞掉**。

#### 修复

**步骤 1**：估计实际分布。开个 Summary 临时收数据，看真实 P50/P90/P99。

```go
tempSummary := prometheus.NewSummary(prometheus.SummaryOpts{
    Name: "_tmp_http_duration",
    Objectives: map[float64]float64{0.5: 0.05, 0.9: 0.01, 0.99: 0.01},
})
// 跑 24 小时，看 P50/P99 大概在哪
```

**步骤 2**：根据实际分布定制 bucket，**关键分位数附近多放桶**。

```go
// API 服务（P99 ≈ 80ms）
buckets := []float64{
    0.005, 0.01, 0.025, 0.05,        // P50 区间
    0.075, 0.1, 0.15, 0.2, 0.25,     // P90~P99 区间，密
    0.5, 1, 2.5, 5,                  // 慢请求
    10, 30,                          // 异常长请求，必须有
}
```

```go
// 图片处理服务（P99 ≈ 30s）
buckets := prometheus.ExponentialBuckets(0.1, 2, 12)
// → [0.1, 0.2, 0.4, 0.8, 1.6, 3.2, 6.4, 12.8, 25.6, 51.2, 102.4, 204.8]
```

**步骤 3**：改造已上线服务时的兼容性

bucket 一旦定下就难改 —— 改了之后历史数据的 bucket 边界变了，PromQL 算的 P99 会跳变。如果业务确实需要改：

- 定义新指标名（如 `http_request_duration_v2_seconds`），并行跑一段时间
- Grafana 切换到新指标后再 drop 旧指标

#### 防御性编程

写一个通用的"bucket 健康度"检查 PromQL：

```promql
# 落在 +Inf 桶的比例（应该 < 0.1%）
sum(rate(http_request_duration_seconds_bucket{le="+Inf"}[5m]))
- sum(rate(http_request_duration_seconds_bucket{le="10"}[5m]))
/
sum(rate(http_request_duration_seconds_count[5m]))
```

如果这个值 > 0.01（1%），说明 bucket 上限太低，告警提醒研发调 bucket。

#### 验证

修复后用 `histogram_quantile` 算出来的 P99 应稳定在业务真实值附近。可以同时跑一个 Summary 对比：

```promql
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
# 应接近
http_request_duration_summary{quantile="0.99"}
```

> **口诀**：Histogram bucket **必须基于真实分布定制**。关键分位数附近密集放桶，最大上限必须覆盖 99.99% 样本。**默认 bucket 只是占位**。

---

### 案例 ④ Counter reset 检测的边界 —— "rate 算出负数"

#### 现象

服务发生 OOM，被 K8s 重启。重启前 `process_cpu_seconds_total` 累计到 87291，重启后从 0 重新累计。Grafana 上看到一个瞬间的"负 rate"尖刺，告警以为 CPU 暴跌触发了"无健康实例"告警。

#### 根因

Prometheus 的 counter reset 检测在 `extrapolatedRate` 里：

```go
prev := samples.Floats[0].F
for _, sample := range samples.Floats[1:] {
    if sample.F < prev {
        resultValue += prev   // ★ 把 reset 之前的累计值补回来
    }
    prev = sample.F
}
```

正常情况：reset 检测能正确处理。但**有 3 种情况会失败**：

**情况 A**：实例 IP/标签变了

K8s pod 重启后 IP 变了，且 `instance` label 用 IP 暴露：

```
旧 series: process_cpu_seconds_total{instance="10.1.2.3:9090"}
新 series: process_cpu_seconds_total{instance="10.1.2.7:9090"}
```

**这是两条不同时序**！Prometheus 不会把它们关联起来。`rate` 在新 instance 上从 0 算起没问题，但跨实例聚合时如果用了诡异的 label 处理，可能算出怪值。

**情况 B**：reset 与 staleness 混淆

如果服务挂了一段时间没被 scrape，再上线时 Prometheus 可能给该序列打 staleness marker。新值 < 旧值会被 reset 检测处理，但中间的 staleness 间隙影响 extrapolation 的窗口边界。

**情况 C**：counter 真的"逻辑回退"

业务 counter 不是单调（被错误地用 Gauge 当 Counter 用）：

```go
// ❌ 被人当成 Counter 用
gauge.Set(currentRequestCount)  // 这其实是个 Gauge

// 用 rate(...) 查询时会被当成 Counter，遇到下降就"补 reset"
```

得到完全错误的速率值（一个真实的下降被算成"+原值"）。

#### 修复

**情况 A** 修复：稳定的 instance label

```yaml
# ❌ 用变化的 IP 做 instance
- source_labels: [__address__]
  target_label: instance

# ✅ 用 pod 名字（rolling 时可能也会变，但批次稳定）
- source_labels: [__meta_kubernetes_pod_name]
  target_label: instance

# ✅ 更稳：用 deployment + 序号
- source_labels: [__meta_kubernetes_pod_label_app]
  target_label: app
```

**情况 C** 修复：严格区分 Counter 和 Gauge

| 指标语义 | 类型 |
| --- | --- |
| 累计请求数 | Counter（用 `Inc()`） |
| 当前活跃连接数 | Gauge（用 `Set()` 或 `Inc/Dec`） |
| 累计错误数 | Counter |
| 当前错误率（已经是比率） | Gauge |
| 累计字节数 | Counter |
| 当前内存占用 | Gauge |

**Counter 永远只用 `Inc()` 或 `Add(positive)`，永远不 `Set()` 或 `Add(negative)`**。client_golang 的 Counter 其实禁止 `Add(negative)`，会 panic。

#### 验证

OOM 重启后立即查：

```promql
# 看是否有"假负数"rate
rate(process_cpu_seconds_total[1m]) < 0
```

应该没有任何序列。如果有，必有上面三种之一的问题。

> **口诀**：Counter reset 检测靠"下降即 reset"，跨实例标签变化、Gauge 当 Counter 用都会让它失效。**instance label 要稳定，Counter 严格只增**。

---

### 案例 ⑤ Summary 跨实例聚合 —— "多实例 P99 加起来就错"

#### 现象

业务用了 Summary 上报延迟，3 个实例。监控大盘上有这么一个 panel：

```promql
sum(http_request_duration_seconds{quantile="0.99"})
```

显示 P99 = 280ms。研发查了下三个实例：100ms、80ms、100ms，加起来 280ms"对得上"。

但用户那边反馈整体 P99 是 95ms 左右。监控比真实值高 3 倍。

#### 根因

Summary 的 quantile 在客户端计算。**quantile 不可加，不能跨实例聚合**。

数学上：三个实例的 P99 分别是 100ms / 80ms / 100ms，**全局的 P99 不是 280ms 也不是 (100+80+100)/3 = 93ms**。它取决于三个实例各自的请求总数和分布形状。

正确的计算需要回到原始数据：

```
全局所有请求按延迟排序 → 取第 99% 个
```

Summary 暴露的只有客户端预算的局部 quantile，**完全无法还原全局**。

更隐蔽的错误：

```promql
# ❌ "平均一下应该差不多"
avg(http_request_duration_seconds{quantile="0.99"})
```

数学上仍然没意义。三个 P99 求平均 ≠ 全局 P99。

#### 修复

**方案 A**：弃用 Summary 改 Histogram

```go
// ❌
summary := prometheus.NewSummary(prometheus.SummaryOpts{
    Name: "http_duration",
    Objectives: map[float64]float64{0.5: 0.05, 0.9: 0.01, 0.99: 0.001},
})

// ✅
histogram := prometheus.NewHistogram(prometheus.HistogramOpts{
    Name: "http_duration",
    Buckets: []float64{0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5},
})
```

跨实例 P99 用 histogram_quantile：

```promql
histogram_quantile(0.99,
  sum by (le) (rate(http_duration_bucket[5m]))
)
```

`sum by (le)` 把所有实例的同 bucket 合并，得到全局 histogram，再算 P99。**这就是 Histogram 的核心价值**。

**方案 B**：必须留 Summary 时只看单实例

```promql
# 看每个实例自己的 P99，不聚合
http_request_duration_seconds{quantile="0.99"}

# 大盘上画多线，每个实例一条
```

但这只能告诉你"实例 A 的 P99 是 X"，不能给出全局 P99。

#### Summary 还能用吗

Summary 在两种场景仍有意义：
1. **单实例服务**：没有跨实例聚合需求
2. **客户端 SDK 没法配 Histogram bucket**：极少数情况下，业务延迟分布完全无法预估，Summary 自带流式 quantile 算法能给出客户端"准确的"局部 P99

但绝大多数生产场景，**用 Histogram，不用 Summary**。

#### 验证

切换到 Histogram 后对比：

```promql
# 旧：Summary 加和（错误）
sum(http_request_duration_seconds_summary{quantile="0.99"})

# 新：Histogram 全局 P99（正确）
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_bucket[5m])))
```

新值应接近用户真实感知的 P99。

> **口诀**：**Summary 的 quantile 不能跨实例聚合**。要全局分位数必须用 Histogram + `sum by (le) + histogram_quantile`。

---

### 案例 ⑥ Push Gateway 滥用 —— "幽灵指标"

#### 现象

CI/CD 接入了 Pushgateway，跑批任务结束时把 success/fail 计数推上去。一段时间后发现：

- 一个早就被删掉的批处理 job 还在 Grafana 上显示"成功"
- 某次失败的任务把"上次成功" 的计数覆盖掉了
- pushgateway 内存涨到 4GB 不下来

#### 根因

Push Gateway 的设计**只为短任务汇总**而设计：

> Pushgateway is intended to be used as a temporary place to push metrics from short-lived jobs.

它的"反 Prometheus 范式"特征：

- **不区分时序的活/死**：被推过的指标永久保留，直到被显式 DELETE 或 Pushgateway 重启
- **没有 TTL**：旧数据不会自动过期
- **以"分组"为单位覆盖**：相同 grouping key（默认 instance + job）的指标会被新推送覆盖
- **不支持 counter reset**：因为它不知道"上次什么时候 reset"

误用模式：

| 误用 | 问题 |
| --- | --- |
| 长期服务用 Pushgateway 推指标 | 应该用 pull，长服务不需要中转 |
| 把 Pushgateway 当指标缓冲区 | 它不是 buffer，是聚合点 |
| 推完不 DELETE | 旧数据残留，造成"幽灵"指标 |
| 一个 job 多个 instance 推同一 grouping key | 互相覆盖 |

#### 修复

**正确使用方式**：

```bash
# 任务开始前清理（避免上次残留）
curl -X DELETE http://pushgateway:9091/metrics/job/nightly_etl/instance/runner-3

# 任务跑
./run_etl.sh

# 任务结束后推送结果
cat <<EOF | curl --data-binary @- http://pushgateway:9091/metrics/job/nightly_etl/instance/runner-3
# TYPE etl_job_duration_seconds gauge
etl_job_duration_seconds 234.5
# TYPE etl_job_success gauge
etl_job_success 1
# TYPE etl_job_last_success_timestamp gauge
etl_job_last_success_timestamp $(date +%s)
EOF
```

注意：这里 `etl_job_success` 用的是 **Gauge** 而不是 Counter。因为推上来的就是"这次的状态"，不该跨次累加。

**别用 Pushgateway 的场景**：

| 场景 | 应该用 |
| --- | --- |
| 长期运行的服务 | 直接 pull `/metrics` |
| 需要按时间精确分析 | Push Gateway 只保留最新值，无历史 |
| 业务事件流（订单、支付） | 用日志 + 实时聚合（Loki / ELK） |
| 跨机房采集 | Federation / 远端写 / Pushprox |

#### 监控

```promql
# 监控 Pushgateway 自身的指标存留量
pushgateway_build_info  # 存活
push_time_seconds        # 每个 group 的最后推送时间

# 老化告警
time() - push_time_seconds > 86400   # 超过 24 小时没推送
```

发现"幽灵"立即清理：

```bash
# 删除整个 job
curl -X DELETE http://pushgateway:9091/metrics/job/some_job

# 删除特定 grouping
curl -X DELETE http://pushgateway:9091/metrics/job/some_job/instance/foo
```

#### 验证

CI 跑完后立即在 Grafana 看到指标，过一段时间应该有过期检测告警提醒清理。

> **口诀**：**Pushgateway 只用于短任务的最终汇总，不是数据缓冲区**。任务前清理、推完留 Gauge 状态、定期对账避免幽灵指标。

---

### 案例 ⑦ Federation 拉爆 —— "跨集群拉数据导致循环 OOM"

#### 现象

公司有 5 个机房各部署了 Prometheus，想做"全局视图"。架构师设计了一个"中心 Prometheus"，通过 Federation 从 5 个区域 Prometheus 拉数据汇总：

```yaml
# 中心 Prometheus 配置
scrape_configs:
  - job_name: 'federate'
    scrape_interval: 15s
    honor_labels: true
    metrics_path: '/federate'
    params:
      'match[]':
        - '{__name__=~".+"}'      # ★ 全部拉
    static_configs:
      - targets:
          - 'prom-bj.example.com:9090'
          - 'prom-sh.example.com:9090'
          - 'prom-sz.example.com:9090'
          # ...
```

上线一周后，中心 Prometheus 反复 OOM，每个区域 Prometheus 也 CPU 跑高（Federate 接口扫整个 TSDB）。

#### 根因

Federation 的 `/federate` 接口本质是"拉一份**所有匹配时序的当前样本**"。`{__name__=~".+"}` 等于"拉所有时序"。如果每个区域有 200 万活跃序列 × 5 区域 = 1000 万序列被拉到中心。

更糟：
- **Federation 是同步 HTTP 调用**：每 15s 中心要在几秒内从 5 个区域各拉 200 万样本，网络 + 序列化 + 反序列化全是热点
- **Federate 接口在区域 Prometheus 端要扫一次 TSDB**：扫 200 万序列要几秒，期间区域 Prometheus 自己的查询都被拖慢
- **中心 Prometheus 仍然只是单机 TSDB**：1000 万序列写入打不住

Federation 的官方定位是 **"拉聚合数据"**，不是"拉全部"。`/federate` 配合 Recording Rule 才是合理用法。

#### 修复

**方案 A**：Federation 只拉聚合时序

区域 Prometheus 配置 Recording Rule，预聚合到 service 粒度：

```yaml
# 区域 Prometheus 的 rules.yml
groups:
  - name: federate_aggregations
    interval: 30s
    rules:
      - record: federate:http_requests:rate5m
        expr: sum by (region, service, status_class) (rate(http_requests_total[5m]))

      - record: federate:http_request_duration:p99
        expr: histogram_quantile(0.99, sum by (region, service, le)
          (rate(http_request_duration_bucket[5m])))
```

中心 Prometheus 只拉 `federate:*` 前缀：

```yaml
params:
  'match[]':
    - '{__name__=~"federate:.*"}'
```

序列数量从 200 万 → 几千。

**方案 B**：换长期存储方案（推荐）

Federation 不是为大规模设计的。生产规模建议：

- **Thanos**：每个区域 Prometheus 边车 sidecar 上传 Block 到对象存储，全局通过 Querier 联邦查询
- **VictoriaMetrics Cluster**：每个区域 vmagent 远端写到中心 vmstorage 集群
- **Cortex / Mimir**：水平扩展的 Prometheus 兼容存储，多租户 + HA

```yaml
# Prometheus 配置远端写到 VictoriaMetrics
remote_write:
  - url: http://vminsert.example.com/insert/0/prometheus/api/v1/write
    queue_config:
      max_samples_per_send: 10000
      capacity: 50000
      max_shards: 30
    metadata_config:
      send: false                  # 减少 protobuf 体积
```

#### 选型决策树

```
活跃序列规模:
  < 100 万: 单机 Prometheus 够用
  100 万 ~ 500 万: Prometheus + 长期存储（Thanos sidecar 模式最简单）
  500 万 ~ 5000 万: VictoriaMetrics 单实例 / Mimir 集群
  > 5000 万: Mimir / VictoriaMetrics 集群 + 多租户
```

**不要默认上 Thanos / Mimir**。1TB 以下数据用单机 Prometheus + 远端备份就够，引入分布式系统反而带来运维负担（见案例 ⑨）。

#### 验证

迁移后中心 Prometheus 时序数 < 10 万；区域 Prometheus 的 Federate 接口响应 < 1s。

```promql
# Federate 响应时间
prometheus_http_request_duration_seconds{handler="/federate"}
```

> **口诀**：**Federation 只拉聚合，不拉原始**。大规模跨集群上 Thanos / VictoriaMetrics / Mimir，别用 Federation 当长期存储方案。

---

### 案例 ⑧ Alertmanager 告警风暴 —— "抑制规则配错，凌晨上千条短信"

#### 现象

数据库网络抖动 30 秒，Alertmanager 在 5 分钟内发出 1200 条告警短信。值班同学手机震到死机，且事后发现真正重要的"DB 连接池耗尽"告警淹没在风暴里。

#### 根因

告警风暴的常见来源：
1. **没有 grouping**：每条告警单独发，不合并
2. **没有 inhibit（抑制）规则**：高优先级告警没有抑制相关的低优先级
3. **没有 silence**：变更窗口期没静默
4. **告警范围太散**：每个 service / instance 都告警，N 个实例炸时 N 条短信

事故现场的 Alertmanager 配置：

```yaml
route:
  receiver: 'sms'
  # ❌ 没有 group_by！
  group_wait: 0s         # ❌ 立即发
  group_interval: 5m
  repeat_interval: 1h
```

告警规则：

```yaml
- alert: DBConnectionTimeout
  expr: db_connection_timeout > 0
  for: 0s        # ❌ 立即触发，不等
  labels:
    severity: critical
```

DB 网络抖动 → 1000 个业务实例同时报 connection timeout → Alertmanager 收到 1000 条 alert，**没有 group_by 就每条一发**。

#### 修复

**修正 1：合理 group**

```yaml
route:
  receiver: 'sms'
  group_by: ['alertname', 'cluster', 'service']   # ★ 同一类合并
  group_wait: 30s        # 等 30s 看还有没有同类，一起发
  group_interval: 5m     # 同 group 再次发的间隔
  repeat_interval: 4h    # 没恢复的提醒间隔
```

效果：1000 条同 alertname 告警 → 30s 内合并为 1 条短信，附带"affected: 1000 instances"。

**修正 2：抑制规则**

```yaml
inhibit_rules:
  # 集群整体挂时，抑制单实例告警
  - source_matchers:
      - alertname="ClusterDown"
    target_matchers:
      - alertname=~"InstanceDown|ServiceDown"
    equal: ['cluster']

  # DB 主告警时，抑制业务侧的"DB 连接超时"
  - source_matchers:
      - alertname="DatabaseDown"
    target_matchers:
      - alertname="DBConnectionTimeout"
    equal: ['cluster', 'environment']
```

抑制规则的语义：source 告警活跃时，target 告警的通知被抑制（仍然记录为活跃，但不发通知）。

**修正 3：合理的 `for`**

```yaml
- alert: DBConnectionTimeout
  expr: rate(db_connection_timeout[2m]) > 5
  for: 3m              # ★ 持续 3 分钟才告警
  labels:
    severity: critical
```

短暂抖动（< 3m）不告警，避免噪声。

**修正 4：分级路由**

```yaml
route:
  receiver: 'default'
  routes:
    - matchers:
        - severity="critical"
      receiver: 'oncall-sms'
      group_wait: 30s
    - matchers:
        - severity="warning"
      receiver: 'oncall-slack'
      group_wait: 5m
    - matchers:
        - severity="info"
      receiver: 'oncall-email'
      group_wait: 1h
```

把 SMS 配额留给 critical，warning 走 Slack，info 走邮件。

**修正 5：变更窗口静默**

```bash
# 上线前用 amtool 创建 silence
amtool silence add \
  --alertmanager.url=http://alertmanager:9093 \
  --duration=2h \
  --comment="Deployment of v2.5.0" \
  alertname=~".*" cluster=prod
```

#### 验证

模拟 DB 抖动 → 1000 实例报 timeout → 应该收到 1~2 条聚合短信，不是 1000 条。

```promql
# Alertmanager 自身指标
alertmanager_notifications_total{integration="sms"}
alertmanager_alerts{state="active"}
alertmanager_silences{state="active"}
```

#### 抑制规则常见的反向陷阱

```yaml
# ❌ 抑制规则配反，导致重要告警被静默
- source_matchers:
    - alertname="HighCPU"
  target_matchers:
    - alertname="ServiceDown"     # ★ ServiceDown 是更严重的，反而被 HighCPU 抑制
```

**铁律**：source 必须是"更严重 / 更上游"的告警。**抑制方向决定了哪个告警会被压制**。

#### 验证清单

- [ ] 模拟"批量实例故障"，确认告警合并
- [ ] 模拟"上游故障"，确认下游告警被抑制
- [ ] 模拟"短暂抖动"，确认 for 起效不告警
- [ ] 模拟"变更窗口"，确认 silence 生效
- [ ] Alertmanager Web UI 上能看清楚 active / silenced / inhibited 三态

> **口诀**：grouping 合并、inhibit 抑制下游、`for` 滤抖动、severity 分级路由、变更前 silence。**告警风暴永远是配置问题，不是 Prometheus 问题**。

---

### 案例 ⑨ 引入 Thanos 反而带来运维负担

#### 现象

某团队 200GB 监控数据（不到 1TB），看到大公司分享 Thanos 架构后立即上 Thanos：sidecar + store gateway + querier + compactor + ruler + 对象存储。

上线 2 个月后：
- 运维成本上升 → 7 个组件，每个都要监控
- 查询延迟反而上升 → 跨 sidecar/store gateway 查找额外延迟
- 对象存储成本 → S3 请求计费、跨 AZ 流量
- compactor 经常 OOM → 数据量增长后 downsample 跑不动
- Querier 查询超时 → 某些 PromQL 在多组件间协调慢

最终决定：拆掉 Thanos，回到单机 Prometheus + 简单的 snapshot 备份到 S3。

#### 根因

**长期存储选型的核心问题不是"能不能存更多"，而是"运维成本是否能 cover 收益"**。

| 数据规模 | 推荐方案 | 复杂度 |
| --- | --- | --- |
| < 100 GB | 单机 Prometheus + 本地磁盘 | 1 组件 |
| 100GB ~ 1TB | 单机 + 远端备份（snapshot 到 S3） | 2 组件 |
| 1TB ~ 10TB | VictoriaMetrics 单节点（vmsingle） | 1 组件，有 HA 选项 |
| 10TB ~ 100TB | VictoriaMetrics Cluster / Mimir | 4~6 组件 |
| > 100TB / 多租户 | Mimir / Thanos Receive | 7+ 组件 |

Thanos 的真正价值在 PB 级数据 + 多团队 + 长期归档（数年）+ 全球化。1TB 以下用 Thanos 是工程上的"杀鸡用牛刀"。

VictoriaMetrics 在中等规模（10~100TB）有非常显著的优势：
- 单节点 vmsingle 能扛 1000 万活跃序列
- 比 Prometheus 内存占用低 5~7 倍
- 查询性能 PromQL 兼容，部分场景更快
- 集群版只有 3 个组件（vmstorage / vminsert / vmselect）

#### 修复

**步骤 1**：评估真实需求

```
1. 总数据量：现在多少？1 年后预计？
2. 保留时长：30 天？6 个月？1 年？
3. 高可用要求：能容忍 5 分钟故障？还是必须 HA？
4. 多租户：单团队还是几十个团队共享？
5. 团队运维能力：能承担几个组件的维护？
```

**步骤 2**：分阶段演进

```
阶段 1（< 100GB）：
  单 Prometheus + 每天 snapshot 到 S3
  灾难恢复：S3 拉回来重启

阶段 2（100GB ~ 1TB）：
  单 Prometheus + 远程写到 VictoriaMetrics 单节点
  Grafana 数据源指向 VM
  Prometheus 本地保留 7 天

阶段 3（1TB ~ 10TB）：
  vmagent 取代 Prometheus 抓取
  vmstorage HA（2 副本）
  上 vmalert 做规则评估

阶段 4（> 10TB / 多租户）：
  VM Cluster / Mimir
  多租户隔离
  对象存储下沉（vmstorage 仍是热层）
```

#### 验证

迁移到 VictoriaMetrics 后：
- 内存占用从 Prometheus 8GB → vmsingle 1.2GB
- 查询 P99 从 800ms → 200ms
- 组件数从 7（Thanos）→ 1（vmsingle）
- 运维工时从每周 8 小时 → 每周 1 小时

> **口诀**：**长期存储选型按数据规模分阶段**。1TB 以下别上分布式方案；中等规模优先 VictoriaMetrics；只有真正的大规模 + 多租户才需要 Thanos / Mimir。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### Metrics 设计层

- [ ] **指标命名**遵循 `<namespace>_<subsystem>_<name>_<unit>` 规范（如 `http_request_duration_seconds`）
- [ ] **单位**统一用基础单位（秒不毫秒、字节不 KB）—— Prometheus 文档要求
- [ ] **Counter** 名字加 `_total` 后缀（如 `http_requests_total`）
- [ ] **Histogram** bucket 基于实际延迟分布定制，覆盖 99.9% 样本
- [ ] **避免 Summary**，跨实例聚合需求一律用 Histogram
- [ ] **绝不把 userId / requestId / sessionId / 时间戳 等高基数字段放进 label**
- [ ] label 值为有限枚举（< 100 个常见值，< 10000 极限）
- [ ] 动态 URL 必须模板化（`/user/:id` 而非 `/user/12345`）

### Scrape 配置层

- [ ] `scrape_interval` 与业务 SLO 匹配（多数场景 15s/30s 够，别默认 5s）
- [ ] `scrape_timeout < scrape_interval`（避免连续两次抓取重叠）
- [ ] `instance` label 用稳定标识（k8s pod name / deployment + 序号），不用 IP
- [ ] 用 `metric_relabel_configs` 提前 drop 已知不需要的指标
- [ ] 高基数指标用 `relabel_configs` drop 或重命名收敛 label

### PromQL / 告警规则层

- [ ] **`rate` 窗口至少 = 4 × scrape_interval**
- [ ] 告警 `for >= rate window`，避免 stale 计时重置
- [ ] **永远不用 `irate` 写告警**，`irate` 只配仪表盘
- [ ] 错误率告警先 `sum` 分子分母再除（防除零 NaN）
- [ ] 告警 `severity` 分级（critical/warning/info），分级路由
- [ ] 反复出现的聚合表达式做成 Recording Rule
- [ ] Recording Rule 命名 `<level>:<metric>:<operations>`（如 `cluster:http_requests:rate5m`）

### 长期存储层

- [ ] 数据规模 < 1TB 用单机 Prometheus + S3 snapshot，**别上 Thanos**
- [ ] 中等规模优先 VictoriaMetrics（运维成本低）
- [ ] Remote Write 配 `wal_compression: true` 减少磁盘
- [ ] Federation 只拉聚合（Recording Rule 后的指标），不拉原始

### Alertmanager 层

- [ ] `group_by` 合理设置，避免告警风暴
- [ ] `group_wait` 30s 左右（合并相似告警）
- [ ] **inhibit_rules** 配置：上游告警抑制下游
- [ ] 变更窗口必须用 silence 静默
- [ ] SMS 配额留给 critical，warning 走 IM，info 走邮件

### 监控自身

- [ ] 监控 Prometheus 自身：`prometheus_tsdb_head_series`、`prometheus_tsdb_wal_corruptions_total`、`prometheus_remote_storage_samples_failed_total`
- [ ] 高基数告警：`prometheus_tsdb_head_series > 5_000_000`
- [ ] WAL 损坏告警：`prometheus_tsdb_wal_corruptions_total > 0`
- [ ] Scrape 失败告警：`up == 0`
- [ ] Alertmanager 自身告警：`alertmanager_cluster_members != X`

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### 跨产品横向对比

- **VictoriaMetrics**：内存效率、查询性能、运维简单度都比 Prometheus 强；MetricsQL 是 PromQL 超集。中等规模首选。
- **Mimir**（Grafana Labs）：Cortex 的演进版本，多租户原生，水平扩展。云原生大集群推荐。
- **Cortex**：Mimir 的前身，仍在维护但社区力量转向 Mimir。
- **Thanos**：第一代分布式 Prometheus 方案，对象存储下沉做得最早。复杂但生态最成熟。
- **InfluxDB**：自己的 InfluxQL/Flux，写入吞吐高但 PromQL 兼容性差。
- **TimescaleDB**：基于 Postgres 的时序扩展，强 SQL，适合需要复杂关联查询的场景。

### OpenTelemetry Metrics 与 Prometheus 的融合

- OTel Metrics 标准化了"指标的数据模型"，与 Prometheus 兼容性持续增强
- OTel Collector 可以同时充当：scrape 客户端（替代 prometheus 自身的 scrape）、Remote Write 接收端、协议转换
- Prometheus 3.0 原生支持 OTLP（OpenTelemetry Protocol）写入
- 长期看 OTel Metrics 会成为采集层标准，Prometheus 是查询/告警层

### Native Histograms 工程化

- 客户端 SDK 成熟度：client_golang ✅、Java（Micrometer 1.13+） ✅、Python 🚧
- 远端存储兼容：VictoriaMetrics ✅、Mimir ✅、Thanos 部分支持
- Grafana 可视化（10.4+）已支持
- 迁移路径：双写经典 + Native，灰度切流，验证完毕后下线经典

### eBPF 采集 + Prometheus 协同

- **Pixie**：基于 eBPF 自动采集 HTTP/SQL/Redis 协议，零代码改造，吐 Prometheus 格式
- **Parca**：持续 profiling（CPU 火焰图常驻），与 Prometheus 共享 label 体系
- **Beyla**（Grafana Labs）：eBPF + OTel，自动生成 RED 指标
- 趋势：业务自定义指标用 SDK，基础设施指标用 eBPF 自动采集

### Prometheus 3.0 关键变化

- **OTLP 原生写入**：直接接受 OpenTelemetry 协议
- **Native Histograms** 转 GA
- **UTF-8 metric / label name 支持**：不再限制 `[a-zA-Z0-9_]`
- **Range 选择器语法变化**：`rate(metric{}[5m:30s])` 子查询性能优化
- **Service discovery 模块化**：可外部插件

### 高基数问题的工程化方案

- **Sparse Histograms**（OpenHistogram）：极端不均匀分布的紧凑表示
- **Adaptive sampling**：动态减少高 cost 时序的采样频率
- **Topology-aware aggregation**：按业务拓扑预聚合（CDP / 服务网格）
- **Trace-derived metrics**：从 trace 的 span 派生指标，trace 用 sampling 控制 cost

### SLO 工具链

- **Sloth**：声明式 SLO，自动生成 Prometheus rules + Alertmanager 配置
- **Pyrra**：可视化的 SLO 平台，集成 Prometheus + Grafana
- **OpenSLO**：跨厂商 SLO 描述标准
- 模式：定义 SLI（rate/error 等）→ 设定 SLO 目标（99.9% / 30天）→ 自动生成 burn rate 告警

---

## 7. 引用

- Prometheus 官方文档：[prometheus.io/docs](https://prometheus.io/docs)
- Prometheus 源码：[github.com/prometheus/prometheus](https://github.com/prometheus/prometheus)
- Brian Brazil, *Prometheus: Up & Running*（O'Reilly，2018，2023 第二版）—— Prometheus 圣经
- Pethuru Raj, *Site Reliability Engineering*（Google SRE 书）—— USE/RED 方法论原始出处
- Facebook Gorilla 论文：[Gorilla: A Fast, Scalable, In-Memory Time Series Database](http://www.vldb.org/pvldb/vol8/p1816-teller.pdf)（VLDB 2015）
- USE Method（Brendan Gregg）：[brendangregg.com/usemethod.html](https://www.brendangregg.com/usemethod.html)
- RED Method（Tom Wilkie）：[grafana.com/blog/2018/08/02/the-red-method](https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/)
- VictoriaMetrics vs Prometheus 性能对比：[victoriametrics.com/blog](https://victoriametrics.com/blog/)
- Native Histograms 设计文档：[Prometheus Sparse Histograms 提案](https://docs.google.com/document/d/1cLNv3aufPZb3fNfaJgdaRBZsInZKKIHo9E6HinJVbpM/edit)
- Cormode et al., *Effective Computation of Biased Quantiles over Data Streams*（2005）—— Summary 算法基础
- Thanos 文档：[thanos.io](https://thanos.io)
- Grafana Mimir 文档：[grafana.com/docs/mimir](https://grafana.com/docs/mimir/latest/)
- Sloth：[sloth.dev](https://sloth.dev/)

---

> 配套：[可观测性主题导览](/article/observability/) ｜ [系统设计](/article/system-design/) ｜ [JVM 调优与排查](/article/jvm/tuning)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
