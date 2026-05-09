# 可观测性

> 不是"日志 / 监控 / 链路追踪"三件套介绍。这一节要讲的是：**怎么用 Metrics / Logs / Traces 真的把生产问题定位到根因**。

---

## 为什么单独开这块

绝大多数公司的"可观测性建设"停在"接了 Prometheus + ELK + SkyWalking"。但真出了问题：
- 一个 P99 飙升告警，你能 5 分钟定位到具体的 SQL 还是远程调用还是 GC 还是邻居 Pod 抢 CPU 吗？
- 业务说"用户反馈下单失败"，你能从日志反查到具体哪一笔 trace 吗？
- SLO 是怎么定的？错误预算用完了怎么办？

可观测性的真正难点不在工具，在**信号设计 + 因果建模 + 工程文化**。这一节按 [桃花岛深度模板](/STYLE) 切，每篇都从一次真实的"5 分钟定位"反向讲。

---

## 已完成的深文

- ✅ **[Prometheus 指标设计与查询 —— 从 USE/RED 到 PromQL 实战](/article/observability/prometheus)** · 1793 行
- ✅ **[分布式链路追踪 —— OpenTelemetry / SkyWalking / Jaeger 全景](/article/observability/tracing)** · 1484 行
- ✅ **[日志架构深入 —— 结构化、采集、Loki / ClickHouse](/article/observability/logging)** · 1677 行
- ✅ **[SLO 错误预算 —— Google SRE 工作簿落地](/article/observability/slo)** · 1262 行
- ✅ **[故障定位方法论 —— 5 分钟定位 P99 飙升的工程剧本](/article/observability/troubleshooting)** · 1603 行

---

## 规划中的深文

### 1. Prometheus 与监控指标设计 ✅ 已完成 → [阅读深文](/article/observability/prometheus)

**核心问题**：监控加了一堆 metrics，告警越来越多但越来越没人看。哪些指标是"信号"、哪些是"噪音"？

深文涵盖：Pull vs Push 设计哲学 / 4 种 metric 类型 / TSDB block + chunk + WAL + Gorilla 压缩 / PromQL 执行模型 / rate vs irate vs increase 精确差别 / Histogram vs Summary 跨实例聚合 / 9 个生产事故（高基数爆 100GB、bucket 失真、Counter reset、Federation OOM、Push Gateway 滥用等）/ 32 条上线清单 / 7 个延伸方向（含 OTel 融合、Native Histograms、SLO 工具链）。

### 2. 分布式链路追踪 ✅ 已完成 → [阅读深文](/article/observability/tracing)

**核心问题**：链路追踪号称"自动接入"，但跨线程池断链、跨 MQ 断链、跨 RPC 协议断链怎么办？采样策略怎么设？

**写作骨架**：
- **原理推导**：Trace / Span / Context 的数据模型；W3C Trace Context 标准；采样策略（头采样 / 尾采样）
- **源码切片**：OpenTelemetry SDK 的 ContextStorage；SkyWalking Agent 的字节码增强
- **实测数据**：不同采样率下的开销与诊断有效性
- **生产案例**：跨线程池追踪丢失（与 [线程池深文](/article/java/threadpool#案例--mdc--traceid-失踪--链路追踪在线程池断了) 案例 ⑤ 呼应）、采样不合理导致核心错误漏采、Span 数据爆炸
- **延伸**：eBPF 实现无侵入追踪、OpenTelemetry 的"统一三件套"愿景

### 3. 日志架构 ✅ 已完成 → [阅读深文](/article/observability/logging)

**核心问题**：业务日志打了 TB 级，ES 撑不住、查询慢、成本高。日志要不要全采、采完放哪、怎么查得动？

**写作骨架**：
- **原理推导**：日志的 5 个生命周期（生成 / 采集 / 传输 / 存储 / 查询），每一步的权衡
- **源码切片**：Loki 的 chunk 存储模型；ClickHouse 做日志的优势；Filebeat / Fluent-bit / Vector 对比
- **实测数据**：ELK vs Loki vs ClickHouse 在万亿规模下的查询延迟与成本
- **生产案例**：ES 集群被告警日志打挂、日志采集 Agent OOM、采样错过关键诊断信息
- **延伸**：日志即指标（log-based metrics）、Trace ID 与日志的关联、日志安全审计

### 4. SLO / SLI / SLA 与错误预算 ✅ 已完成 → [阅读深文](/article/observability/slo)

**核心问题**：SLO 听起来高大上，但落地全是坑。错误预算用完了真的停发布吗？SLI 怎么选才反映"用户视角"？

**写作骨架**：
- **原理推导**：SLI（指标）/ SLO（目标）/ SLA（合同）三者的精确语义；多窗口多次燃烧率告警
- **源码切片**：Sloth / OpenSLO 配置工具；Prometheus 的 SLO Rule
- **实测数据**：不同窗口（30 天 / 90 天）的错误预算变化；多窗口告警的"假阳性"率
- **生产案例**：SLO 太松（业务感受差但 SLO 仍达标）、错误预算"挪用"、SLA 与 SLO 脱钩
- **延伸**：Google SRE 工作簿、错误预算驱动的发布管理、Toil 治理

### 5. 故障定位方法论 ✅ 已完成 → [阅读深文](/article/observability/troubleshooting)

**核心问题**：告警响了，你的诊断顺序是什么？怎么避免"先看一遍所有图表"的陷阱？

**写作骨架**：
- **原理推导**：USE 模型的诊断流；从"症状"到"根因"的反向溯源；常见因果链
- **工具切片**：jcmd / async-profiler / arthas / pyspy / py-spy / perf 的快速使用
- **实测案例**：从一个真实告警走到根因的完整复盘（含每一步执行的命令和判断逻辑）
- **生产案例**：误判（GC vs IO）、定位太慢错过黄金时间、根因找错（修了表象问题）
- **延伸**：火焰图阅读、断点调试 vs 观测、Chaos 演练在故障定位中的作用

### 6. eBPF 与下一代可观测性

**核心问题**：eBPF 凭什么是"未来"？它解决了 Agent 模式的什么问题？Cilium / Pixie / Parca 怎么选？

**写作骨架**：
- **原理推导**：eBPF 的虚拟机模型；为什么能"零侵入"；安全边界
- **源码切片**：Pixie 的 BPF 程序、Parca 的连续性能剖析
- **实测数据**：eBPF Agent 的开销；与传统 Agent 的对比
- **生产案例**：eBPF 程序内存泄漏、内核版本不兼容、生产部署的权限管理
- **延伸**：与 OpenTelemetry 的融合、Cilium 在 Service Mesh 中的角色

---

## 关联现有文章
- [线程池生产实战](/article/java/threadpool) —— MDC / TraceId 跨线程池
- [JVM 调优与排查](/article/jvm/tuning) —— 进程级诊断工具
- [系统设计](/article/system-design/) —— 容量规划与压测

## 想看哪一篇？

[告诉我](/article/about) 你最想先看哪个主题的深文，我会优先排期。
