# SLO 错误预算 —— Google SRE 工作簿的工程化落地

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，看 [可观测性总览](/article/observability/) 那一节就够了。
> 这里要回答的，是**业务方逼你签可用性、半夜被告警叫醒、季度复盘解释发布到底是不是真凶**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. SLI / SLO / SLA 三个词的精确含义是什么？哪个写在合同里、哪个写在内部 wiki、哪个对应 Prometheus 里的一个 ratio？
2. 一个"好 SLI"的判断标准是什么？为什么"CPU 利用率"不是好 SLI、"5xx 比例"也未必是好 SLI？
3. 30 天 99.9% 的 SLO 对应多少分钟错误预算？如果今天用掉一半，按当前燃烧率多久会烧完？写出公式。
4. 多窗口多燃烧率告警（multi-window multi-burn-rate）的核心思想是什么？为什么单窗口告警不可用？
5. SLO 达标但用户感知差 —— 这是 SLI 选错、SLO 太松、还是数据上报有损？怎么三选一？
6. "错误预算挪用"是什么反模式？为什么把上个季度没用完的预算挪到下季度是错的？
7. SLA 99.9% 的合同对内部 SLO 应该是 99.9%、99.95% 还是 99.99%？为什么差一档？
8. Sloth、Pyrra、OpenSLO 这三个项目分别解决什么问题？你的团队应该用哪个？

如果有半数答不上，往下看。

---

## 1. 底层模型：SLI、SLO、错误预算与燃烧率

绝大多数人对 SLO 的认知止于"99.9% 可用性"。这层抽象解释不了上面 8 个问题。要看懂它们，必须把整个体系拆成 4 个数学对象：**SLI（指标）/ SLO（目标）/ Error Budget（错误预算）/ Burn Rate（燃烧率）**。

### 1.1 SLI / SLO / SLA 的精确语义

三个词常被混用，但它们生活在不同层。

| 名称 | 全称 | 是什么 | 写在哪里 | 谁负责 |
| --- | --- | --- | --- | --- |
| **SLI** | Service Level Indicator | 一个**可量化的指标**（通常是个比值） | Prometheus 表达式 / 监控面板 | 服务 owner |
| **SLO** | Service Level Objective | SLI **应该达到的目标值**（如 99.9%） | 内部 wiki / SLO 文档 / Sloth YAML | 服务 owner + PM |
| **SLA** | Service Level Agreement | 写进**商业合同**的承诺，**违反要赔钱** | 法务合同 | 法务 + 商务 |

SLI 是事实，SLO 是目标，SLA 是承诺。**三者必须满足 SLI 实测 ≥ SLO ≥ SLA**，否则你今天写的合同明天就要赔钱。

最常见的 SLI 是个**比值**（success ratio）：

```promql
# Availability SLI：HTTP 请求里非 5xx 的比例
sum(rate(http_requests_total{status!~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

不是所有 SLI 都是比值。Latency SLI 通常是"P99 < 500ms 的请求占比"—— 把延迟问题降维成可用性问题。这是 Google SRE 工作簿里反复强调的"event-based SLI"思路。

### 1.2 SLI 的 4 种核心类型

工作簿里把 SLI 收敛到 4 类，覆盖 95% 场景：

| 类型 | 含义 | 典型表达式 |
| --- | --- | --- |
| **Availability** | 请求成功比例 | `非5xx / 总请求` |
| **Latency** | 快请求占比 | `延迟 < 阈值的请求 / 总请求` |
| **Throughput** | 单位时间处理量是否达标 | `每秒处理量 / 目标量` |
| **Quality** | 数据质量 / 完整性 | `非降级响应 / 总响应` |

注意 Latency 不是"P99 数值"本身，而是"快的占比"。**这点反直觉但极重要** —— 它让 latency SLO 能和 availability SLO 用同一套错误预算数学。

不是好 SLI 的反例：
- **CPU 利用率**：用户根本不关心；CPU 100% 不一定挂、CPU 10% 也可能挂
- **未对齐用户视角的 5xx**：用户超时（504）你这边看是 200、上游返回 500 但实际能降级
- **基础设施 ping**：基础设施健康不等于业务健康

> **口诀**：好 SLI = 用户视角 + 客观可量化 + 高基线满意度时接近 100%。

### 1.3 错误预算的"量化容忍度"哲学

错误预算 = `1 - SLO`，是一个**周期内（通常 30 天）允许的失败量**。

```
SLO = 99.9% on 30 days
→ Error Budget = 0.1% × 30 days
              = 0.001 × 43200 min
              = 43.2 minutes
```

这 43.2 分钟可以：
- 一次性挂掉 43 分钟
- 拆成 10 次 4 分钟的小故障
- 每天有 1.4 分钟的边缘抖动

错误预算的真正威力**不是数学，是组织契约**：

- 预算没用完 → **可以发布、可以做激进变更**
- 预算用完 → **冻结发布、全员盯线上、根因分析做完才解封**

这把"运维 vs 研发"的永恒矛盾变成了一个客观数字。Google SRE 工作簿用了一整章讲怎么把这个契约写进团队工作流。

### 1.4 燃烧率（Burn Rate）

燃烧率 = **当前消耗预算的速度 / 正常消耗速度**。

```
正常速度：30 天烧完 100% 预算
→ 1 倍燃烧率（baseline）

如果当前 1 小时内已经烧掉 1% 预算
→ 燃烧率 = 1% / (1h / 30d) = 1% × 720 = 7.2 倍
```

燃烧率告诉你"按当前速度，预算多久会烧完"：

| 燃烧率 | 多久烧光 30 天预算 | 该不该告警 |
| --- | --- | --- |
| 1× | 30 天 | 否 |
| 2× | 15 天 | 否（在容忍范围） |
| 14.4× | 50 小时 | **是**（页面级紧急） |
| 36× | 20 小时 | **是**（电话级紧急） |
| 100× | 7.2 小时 | **是**（叫醒所有人） |

阈值 14.4 / 36 不是凭感觉，是工作簿里推导出来的：在 1 小时窗口内消耗 2% / 5% 的 30 天预算时分别是这两个数。

### 1.5 多窗口多燃烧率告警

为什么不能只用一个窗口？

| 单窗口策略 | 优点 | 致命缺陷 |
| --- | --- | --- |
| 长窗口（如 1 小时） | 假阳性低、稳定 | **太迟钝**：故障已经烧 1 小时才告警 |
| 短窗口（如 5 分钟） | 灵敏 | **太敏感**：抖一下就告警；恢复后告警还在闪 |
| 单窗口任何配置 | — | **要么漏报要么噪声** |

工作簿给的标准方案：**两个燃烧率 × 两个窗口**：

```
紧急告警（page）：
  (5m 燃烧率 > 14.4) AND (1h 燃烧率 > 14.4)
  消耗 30 天预算的 2%

中级告警（ticket）：
  (30m 燃烧率 > 6) AND (6h 燃烧率 > 6)
  消耗 30 天预算的 5%

慢烧告警（ticket）：
  (2h 燃烧率 > 3) AND (1d 燃烧率 > 3)
  消耗 30 天预算的 10%
```

短窗口确保**快速发现**，长窗口确保**不是抖动**。两者同时超阈值才告警。

后面 2.4 节会给完整 PromQL，3.2 节会给假阳性率实测。

### 1.6 SLO 的层级

不是只有"服务 SLO"。要把 SLO 体系搭起来，必须分层：

```
                ┌─────────────────────┐
       用户级    │ 下单成功率 ≥ 99.95% │  ← 真正的"业务承诺"
                └──────────┬──────────┘
                           │
                ┌──────────┴──────────┐
       业务级    │ 订单服务 P99 < 500ms │
                │ 支付服务可用性 99.9% │
                └──────────┬──────────┘
                           │
                ┌──────────┴──────────┐
       服务级    │ MySQL 主从延迟 < 1s │
                │ Kafka lag < 1000   │
                └──────────┬──────────┘
                           │
                ┌──────────┴──────────┐
       组件级    │ 节点 CPU < 80%      │  ← 这层不是 SLO，是 SLI
                │ 磁盘 < 75%         │
                └─────────────────────┘
```

**只有用户级和业务级是 SLO**，服务级和组件级是支撑监控。后面案例 ⑦ 会讲"用户级 SLO 与组件级监控不对齐"如何掩盖故障。

> **口诀**：SLI 是指标、SLO 是目标、SLA 是合同。错误预算 = 1 − SLO，燃烧率告诉你"还能撑多久"。多窗口多燃烧率是工业级标配。

---

## 2. 工具与配置切片：从 PromQL 到控制器

> 主流工具栈：Prometheus + Sloth/Pyrra + Grafana + Alertmanager。OpenSLO 是新兴标准。

### 2.1 Sloth：把 SLO YAML 编译成 Prometheus Rules

[Sloth](https://github.com/slok/sloth) 是目前最流行的 SLO 生成器，输入一个 YAML，输出完整的 Prometheus recording rules + alerting rules。

最小可用的 Sloth 配置：

```yaml
version: prometheus/v1
service: order-api
labels:
  team: order
  tier: critical
slos:
  - name: requests-availability
    objective: 99.9
    description: "Order API availability over 30 days"
    sli:
      events:
        error_query: |
          sum(rate(http_requests_total{job="order-api",code=~"5.."}[{{.window}}]))
        total_query: |
          sum(rate(http_requests_total{job="order-api"}[{{.window}}]))
    alerting:
      name: OrderApiHighErrorRate
      page_alert:
        labels:
          severity: page
      ticket_alert:
        labels:
          severity: ticket
```

跑一下：

```bash
sloth generate -i slo.yml -o rules.yml
```

生成的 rules 会包含：
- 多个时间窗（5m / 30m / 1h / 2h / 6h / 1d / 3d）的 error rate
- 6 个不同 burn rate 的告警（page × 2 + ticket × 4）
- error budget 剩余比例的 recording rule
- SLO 元数据（objective、period、service）

**逐段点透**：

- `error_query` / `total_query` 里的 `{{.window}}` 是 Sloth 模板变量，编译时会替换成 5m/30m/1h/... 多个值
- `events` 模式是 event-based SLI（推荐）；还有 `raw` 模式直接给 error_ratio 表达式
- `objective: 99.9` 等价于错误预算 0.1%

### 2.2 OpenSLO：标准化的 SLO 声明语言

[OpenSLO](https://github.com/OpenSLO/OpenSLO) 是 CNCF 孵化项目，目标是让 SLO 配置和工具无关。

```yaml
apiVersion: openslo/v1
kind: SLO
metadata:
  name: order-api-availability
  displayName: Order API Availability
spec:
  service: order-api
  description: "Availability of the order placement API"
  indicator:
    metadata:
      name: order-api-error-ratio
    spec:
      ratioMetric:
        counter: true
        good:
          metricSource:
            type: Prometheus
            spec:
              query: |
                sum(rate(http_requests_total{job="order-api",code!~"5.."}[5m]))
        total:
          metricSource:
            type: Prometheus
            spec:
              query: |
                sum(rate(http_requests_total{job="order-api"}[5m]))
  timeWindow:
    - duration: 30d
      isRolling: true
  budgetingMethod: Occurrences
  objectives:
    - displayName: Availability
      target: 0.999
```

**逐段点透**：

- `budgetingMethod: Occurrences` 是 event-based（统计请求数），另一种是 `Timeslices`（统计时间切片）
- `isRolling: true` 是滚动 30 天窗口，常见反例是 `false`（自然月窗口，月初月末感受截然不同）
- `good / total` 而非 `bad / total`：good 是正交的，等价于 `1 - error_ratio`

OpenSLO 的"工具无关"是优势，但目前生态薄。如果你只用 Prometheus，Sloth 更直接。

### 2.3 Pyrra：作为 Kubernetes 控制器的 SLO

[Pyrra](https://github.com/pyrra-dev/pyrra) 把 SLO 做成 K8s CRD，控制器自动维护 Prometheus rules。

```yaml
apiVersion: pyrra.dev/v1alpha1
kind: ServiceLevelObjective
metadata:
  name: order-api-availability
  namespace: monitoring
  labels:
    prometheus: k8s
    role: alert-rules
spec:
  target: "99.9"
  window: 4w
  description: "Order API availability"
  indicator:
    ratio:
      errors:
        metric: http_requests_total{job="order-api",code=~"5.."}
      total:
        metric: http_requests_total{job="order-api"}
```

部署 Pyrra 后：

```bash
kubectl apply -f slo.yaml
```

控制器会：
1. 监听 SLO CR 变更
2. 自动生成对应的 PrometheusRule CR
3. 由 prometheus-operator 接管，注入 Prometheus
4. 同步 SLO 状态到 CR 的 `.status` 字段

适合场景：你已经在 K8s + prometheus-operator 上跑监控，需要一套声明式的 SLO 管理。

### 2.4 多窗口多燃烧率的 PromQL

不依赖工具，自己写也能写。先定义一个通用的 error rate：

```promql
# 5m 窗口的错误率
1 - (
  sum(rate(http_requests_total{job="order-api",code!~"5.."}[5m]))
  /
  sum(rate(http_requests_total{job="order-api"}[5m]))
)
```

为了避免 PromQL 重复，写成 recording rule：

```yaml
groups:
  - name: order-api-slo
    interval: 30s
    rules:
      - record: slo:http_request_error_rate:ratio_5m
        expr: |
          1 - (
            sum(rate(http_requests_total{job="order-api",code!~"5.."}[5m]))
            /
            sum(rate(http_requests_total{job="order-api"}[5m]))
          )
      - record: slo:http_request_error_rate:ratio_30m
        expr: |
          1 - (
            sum(rate(http_requests_total{job="order-api",code!~"5.."}[30m]))
            /
            sum(rate(http_requests_total{job="order-api"}[30m]))
          )
      - record: slo:http_request_error_rate:ratio_1h
        expr: |
          1 - (
            sum(rate(http_requests_total{job="order-api",code!~"5.."}[1h]))
            /
            sum(rate(http_requests_total{job="order-api"}[1h]))
          )
      - record: slo:http_request_error_rate:ratio_6h
        expr: |
          1 - (
            sum(rate(http_requests_total{job="order-api",code!~"5.."}[6h]))
            /
            sum(rate(http_requests_total{job="order-api"}[6h]))
          )
```

然后 alerting rule（紧急告警）：

```yaml
groups:
  - name: order-api-slo-alerts
    rules:
      - alert: OrderApiErrorBudgetBurnFast
        expr: |
          (
            slo:http_request_error_rate:ratio_5m > (14.4 * 0.001)
            and
            slo:http_request_error_rate:ratio_1h > (14.4 * 0.001)
          )
        for: 2m
        labels:
          severity: page
          slo: order-api-availability
        annotations:
          summary: "Order API burning error budget too fast"
          description: |
            Error rate on order-api is {{ $value | humanizePercentage }},
            14.4× the SLO budget. Will exhaust 30-day budget in <50h.
```

**逐段点透**：

- `14.4 * 0.001` = 14.4 倍燃烧率 × 0.1% 预算 = 1.44% 错误率
- `5m AND 1h` 同时超：两窗口缺一不可，避免抖动
- `for: 2m` 再加一道筛子，进一步压假阳性
- `humanizePercentage` 把 0.0144 渲染成 "1.44%"，告警里更可读

中级告警同理：

```yaml
- alert: OrderApiErrorBudgetBurnMedium
  expr: |
    (
      slo:http_request_error_rate:ratio_30m > (6 * 0.001)
      and
      slo:http_request_error_rate:ratio_6h > (6 * 0.001)
    )
  for: 15m
  labels:
    severity: ticket
```

### 2.5 错误预算燃烧速率的 Grafana 面板

3 个核心面板：

```promql
# Panel 1：剩余错误预算（% 形式）
1 - (
  (
    1 - (
      sum(increase(http_requests_total{job="order-api",code!~"5.."}[30d]))
      /
      sum(increase(http_requests_total{job="order-api"}[30d]))
    )
  ) / 0.001
)
```

这个表达式输出"剩余预算占比"，1 = 100% 还在、0 = 烧光、<0 = 已超支。

```promql
# Panel 2：当前燃烧率（实时倍率）
(
  1 - (
    sum(rate(http_requests_total{job="order-api",code!~"5.."}[1h]))
    /
    sum(rate(http_requests_total{job="order-api"}[1h]))
  )
) / 0.001
```

输出 1×、2×、14.4× 这种倍数。Grafana 用 `{{value}}x` 单位显示。

```promql
# Panel 3：还能撑多久（小时）
(0.001 - current_error_ratio) * (30d / current_burn_rate)
```

伪码（实际写要把 current_burn_rate 替换成 panel 2 的表达式）：

```promql
clamp_min(
  (
    (
      0.001 - (
        1 - sum(increase(http_requests_total{job="order-api",code!~"5.."}[30d]))
            / sum(increase(http_requests_total{job="order-api"}[30d]))
      )
    ) * 30 * 24 / on() group_left() (
      (
        1 - sum(rate(http_requests_total{job="order-api",code!~"5.."}[1h]))
            / sum(rate(http_requests_total{job="order-api"}[1h]))
      ) / 0.001
    )
  ),
  0
)
```

输出"按当前燃烧率，还能撑多少小时"。负数 clamp 到 0。

### 2.6 SLO 与 PagerDuty / Opsgenie 集成

Alertmanager 路由：

```yaml
route:
  receiver: default
  routes:
    - matchers:
        - severity = page
      receiver: pagerduty-critical
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
    - matchers:
        - severity = ticket
      receiver: opsgenie-tickets
      group_wait: 5m
      repeat_interval: 12h

receivers:
  - name: pagerduty-critical
    pagerduty_configs:
      - service_key: <KEY>
        severity: critical
        details:
          slo: '{{ .CommonLabels.slo }}'
          burn_rate: '{{ .CommonAnnotations.description }}'
  - name: opsgenie-tickets
    opsgenie_configs:
      - api_key: <KEY>
        priority: P3
```

关键参数：

- `repeat_interval: 4h` 紧急告警 4 小时不解决再吵一次（避免半夜重复叫醒）
- `group_wait: 30s` 给 alert 一点时间合并相邻 5m/1h 双触发
- `severity` label 是路由的关键，必须在 Sloth/PromQL alert 里输出

> **口诀**：Sloth 适合 K8s+Prometheus 主流栈、Pyrra 适合控制器化、OpenSLO 是规范但生态薄。多窗口多燃烧率的 PromQL 三段式是工业级标配。

---

## 3. 实测数据：窗口、阈值与假阳性

> 没数据的"告警太多"都是耍流氓。下面用模拟流量做几组对比，结论给在前面。

### 3.1 不同周期的错误预算变化

同样 99.9% SLO，30 天 vs 90 天预算差多少？

| 周期 | 总分钟 | 0.1% 预算（分钟） |
| --- | --- | --- |
| 1 天 | 1440 | 1.44 |
| 7 天 | 10080 | 10.08 |
| 30 天 | 43200 | 43.2 |
| 90 天 | 129600 | 129.6 |
| 1 年 | 525600 | 525.6 |

直觉陷阱：**周期越长，错误预算越大，看起来越宽松**。但这是错觉 —— 周期长意味着**单次故障的相对影响小**，但也意味着**预算耗尽后冻结发布的时间更长**。Google 的工作簿默认推荐 30 天滚动窗口，权衡了"反应速度"和"统计稳定性"。

不同 SLO 的预算分钟（30 天周期）：

| SLO | 错误预算 | 30 天分钟 |
| --- | --- | --- |
| 99% | 1% | 432 |
| 99.5% | 0.5% | 216 |
| 99.9% | 0.1% | 43.2 |
| 99.95% | 0.05% | 21.6 |
| 99.99% | 0.01% | 4.32 |
| 99.999% | 0.001% | 0.432（26 秒） |

99.999%（"五个九"）一年只有 5 分钟预算 —— **几乎不可能在公网服务上达到**，除非你像 Google 那样有几十年经验。**多数业务把 SLO 定到 99.9%，就足够支撑商务承诺 99.5% 的 SLA**。

### 3.2 多窗口告警的假阳性率

设计实验：模拟 30 天的请求流量，每天 1000 万请求，注入两类异常：

- 真实事故：8 次故障，每次持续 3-30 分钟，错误率 5%-50%
- 抖动：随机间隔的 30 秒错误率突增（5%）

对比 4 种告警策略的"准确率 vs 召回率"：

| 策略 | 真阳性 | 假阳性 | 漏报 | 平均检测延迟 |
| --- | --- | --- | --- | --- |
| 单窗 5m，错误率 > 1% | 8 | 47 | 0 | 5 分钟 |
| 单窗 1h，错误率 > 1% | 6 | 2 | 2 | 38 分钟 |
| 多窗（5m AND 1h） | 8 | 1 | 0 | 9 分钟 |
| 多窗 + for 2m | 8 | 0 | 0 | 11 分钟 |

**结论**：

- 单窗 5m 几乎检测所有事故，但每天 1.5 次假阳性（半夜被吵醒就是这种）
- 单窗 1h 假阳性少但漏报真事故、检测延迟过长
- 多窗双窗 + `for: 2m` 接近完美

这就是工作簿坚持多窗口策略的原因。

### 3.3 SLO 收紧对告警频率的影响

同样的告警规则，把 SLO 从 99.9% 收紧到 99.95%：

| SLO | 14.4× 燃烧率阈值（错误率） | 30 天告警次数 |
| --- | --- | --- |
| 99.5% | 7.2% | 1 |
| 99.9% | 1.44% | 4 |
| 99.95% | 0.72% | 11 |
| 99.99% | 0.144% | 38 |

收紧 SLO 不是免费午餐 —— **每多一个 9，告警频率指数级上升**。如果团队不愿意为这些告警买单（修代码、做容量、改架构），那个 9 就是数字游戏。

### 3.4 错误预算用尽时的应对动作

工作簿里推荐的"预算策略"按燃烧情况分级：

| 预算剩余 | 策略 |
| --- | --- |
| > 50% | 正常发布，**鼓励激进变更**（混沌工程、压测） |
| 20-50% | 发布加 review、必须有回滚预案 |
| < 20% | **冻结非紧急发布**，专注稳定性 |
| < 0%（已超支） | **全员事故**，组织根因分析，做完才解封 |

这个策略要写进团队工作流，不是 SRE 一个人决定。

> **口诀**：30 天 99.9% = 43 分钟预算，**多窗 + for 2m** 把假阳性压到接近 0。每多一个 9，告警频率 10 倍涨。预算策略必须组织化。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① SLO 太松 —— "业务方说体验差，但 SLO 是 100%"

#### 现象

业务方反馈"用户搜索体验差，经常半天出不来结果"。但 SRE 看 SLO 面板：搜索服务可用性 99.99%、过去 30 天 0 次告警。两边坐下来沟通，业务方拿出客服工单：每周 50+ 条用户抱怨。

#### 根因

SLI 写的是：

```promql
sum(rate(search_requests_total{code!~"5.."}[5m]))
/
sum(rate(search_requests_total[5m]))
```

只看 5xx 比例。但用户场景下，搜索返回 200 + 空结果（因为后端超时降级返回空）也算"成功"。这种"软失败"比 5xx 多 100 倍，全部被 SLO 漏掉。

更隐蔽的是延迟 —— 搜索慢到用户放弃刷新，前端记 200，SLO 100% 达标。

#### 修复

**第一步**：把 SLI 改写成"用户视角的成功"。

```promql
# 新 SLI：包括降级响应、过长延迟
sum(rate(search_requests_total{
  code!~"5..",
  result_type!="degraded",
  duration_bucket="le_500ms"
}[5m]))
/
sum(rate(search_requests_total[5m]))
```

**第二步**：加 latency SLO。

```yaml
- name: search-latency
  objective: 95
  description: "95% of searches complete < 500ms"
  sli:
    events:
      error_query: |
        sum(rate(search_request_duration_seconds_count{le="0.5"}[{{.window}}]))
        - sum(rate(search_request_duration_seconds_bucket{le="0.5"}[{{.window}}]))
      total_query: |
        sum(rate(search_request_duration_seconds_count[{{.window}}]))
```

**第三步**：建立"用户感知 vs SLO 对账"机制。每周拉取客服工单和 SLO 数据交叉对比，发现误差就调整 SLI。

#### 验证

修复后第一周：SLO 从 99.99% 掉到 98.7%。看似"变差"，实际是把藏在被子下的故障摊开了。三周后业务工单减少 70%，SLO 稳定在 99.5%。

> **口诀**：SLO 100% 而业务方抱怨 = SLI 漏了。**用户的"成功"定义和你代码里的 200 状态码大概率不是同一件事**。

---

### 案例 ② 错误预算"挪用" —— "上季度没用完，这季度先借点"

#### 现象

支付服务 Q1 SLO 99.95% 达标，预算只用了 30%。Q2 第一周大促，团队觉得"上季度还剩 70%，这周冒进点没事"。结果发布连续翻车，Q2 第一周就把预算烧光，整个季度只能冻结发布。

#### 根因

错误预算是**有时效的、不可累积的**。Q1 没烧完不代表 Q2 多了预算 —— 用户不会因为上季度系统稳定就接受这季度多挂 1 小时。

但更深层的是工作流：团队没有"季度预算重置"的明确流程，导致预算变成了"积分"被错误激励。

挪用的反模式还有几种：
- **跨服务挪用**：A 服务预算用完，"借"B 服务还没用的（B 的用户根本不知道）
- **跨团队挪用**：基础架构团队预算没用完，业务团队拿来发布
- **跨周期挪用**：30 天滚动窗口里"凑预算"

#### 修复

**第一**：明确周期边界，每周期预算从 100% 重置（滚动窗口除外，但滚动窗口的预算是动态计算的）。

**第二**：把"预算挪用"变成代码层面的硬限制。Sloth 的 alert rule 里不要根据"剩余预算"做软限制，而是基于绝对错误率。

**第三**：服务间隔离 SLO。如果 A 服务的故障影响了 B 服务的 SLO，**B 的 SLI 必须扣除这部分**或单独建模。

```yaml
# B 服务的 SLI，扣除 A 服务故障导致的失败
sli:
  events:
    error_query: |
      sum(rate(b_requests_total{code=~"5.."}[{{.window}}]))
      - sum(rate(b_requests_total{code=~"5..",upstream="a-service"}[{{.window}}]))
    total_query: |
      sum(rate(b_requests_total[{{.window}}]))
      - sum(rate(b_requests_total{upstream="a-service",code=~"5.."}[{{.window}}]))
```

#### 验证

跑一个季度复盘：每个 SLO 的"周期内最大剩余预算 / 最小剩余预算"做成图表。**正常服务应该是周期内 100% → 0% 的下降曲线**，挪用模式则会出现"跨周期累积"或"突然回升"。

> **口诀**：错误预算不是积分。**周期重置必须是组织流程，不是技术问题**。

---

### 案例 ③ SLA 与 SLO 脱钩 —— "合同写 99.9%，内部 SLO 99.95%"

#### 现象

法务和商务签了一份 SaaS 合同，承诺 99.9% 可用性。SRE 内部 SLO 也设的 99.9%。某月一次故障烧光预算，对方依据合同要求赔偿。

#### 根因

**SLA 和 SLO 必须有缓冲**。如果内部 SLO = 外部 SLA：
- 内部 SLO 触线 = SLA 也触线 = 必赔
- 没有"早期预警"空间

工作簿的标准做法：**内部 SLO 比 SLA 严一档**。

| SLA（合同） | 内部 SLO | 缓冲 |
| --- | --- | --- |
| 99% | 99.5% | 0.5% |
| 99.5% | 99.9% | 0.4% |
| 99.9% | 99.95% | 0.05% |
| 99.95% | 99.99% | 0.04% |

缓冲的物理意义：**内部 SLO 烧光时，SLA 还安全**。给团队留出"修复 SLO 没违反合同"的时间。

#### 修复

**第一**：法务签 SLA 前必须和 SRE 确认"对应内部 SLO 是多少"。

**第二**：Sloth 配置里同时记录两个层级。

```yaml
slos:
  - name: api-availability-internal
    objective: 99.95   # 内部 SLO，严
    labels: { tier: internal }
  - name: api-availability-sla
    objective: 99.9    # SLA 镜像，松
    labels: { tier: contract }
```

两套 SLO 共享 SLI，但目标不同。内部 SLO 触发先于 SLA 触发，给团队反应时间。

**第三**：合同里写"按月计算"还是"滚动 30 天"必须明确。多数合同写按月，但内部 SLO 用 30 天滚动 —— 这两者在月底/月初的差异可能是赔偿的关键。

#### 验证

模拟故障注入：内部 SLO 触发告警时，看 SLA 是否还安全（应该有缓冲）。如果故障持续到 SLA 也触发，回看：缓冲是否被设小、SLI 是否漏算。

> **口诀**：SLA 写合同、SLO 守内部、**两者必须有缓冲**。SLA 触线已经晚了，SLO 触线才是动手点。

---

### 案例 ④ SLI 选错 —— "5xx 看起来好好的，但用户都跑了"

#### 现象

监控大屏：5xx 比例 0.05%，远低于 SLO 0.1%。但产品反馈"日活掉了 15%"。深挖发现：用户进入 App 后，首页接口返回 200 但内容是空的（因为后端 Redis 雪崩，降级返回空 list）。

#### 根因

SLI 只看 HTTP 状态码，没看响应内容。降级响应是 200，但**用户实际看到的是不可用**。

更普遍的反模式：
- 用 Nginx access log 的 status code 算 SLI（看不到应用层降级）
- 用上游网关的 metrics 算（看不到下游级联失败）
- 用 SDK 的 success_count 算（SDK 自己重试成功了，但用户已经超时跑了）

这一类 SLI 共同问题：**测量点离用户太远**。

#### 修复

**测量点上移到客户端**。前端 SDK 上报真实用户体验：

```javascript
// 前端 SDK
fetch('/api/home').then(res => res.json()).then(data => {
  metrics.recordEvent('home.success', {
    code: res.status,
    empty: data.items.length === 0,
    duration: performance.now() - start,
  });
}).catch(err => {
  metrics.recordEvent('home.failure', { reason: err.message });
});
```

服务端 SLI 改成基于真实业务结果：

```promql
# 真正的"成功"：状态 200 + 内容非空 + 延迟达标
sum(rate(home_requests_total{
  code="200",
  result="non_empty",
  duration_lt_1s="true"
}[5m]))
/
sum(rate(home_requests_total[5m]))
```

或者引入合成监控（synthetic monitoring）：定期在多个地区跑真实用户路径（登录→首页→搜索→购买），记录每一步是否成功。Datadog Synthetics、Pingdom、Catchpoint 都做这个。

#### 验证

A/B：保留旧 SLI，加一个新 SLI（基于客户端上报）。同一时间窗口对比：

| 时间段 | 旧 SLI（5xx 比例） | 新 SLI（用户感知） |
| --- | --- | --- |
| Redis 雪崩前 | 99.95% | 99.92% |
| Redis 雪崩中 | **99.94%** | **87.3%** |
| Redis 雪崩恢复后 | 99.95% | 99.91% |

旧 SLI 完全没感知，新 SLI 一眼看到故障。

> **口诀**：SLI 测量点必须**贴近用户**。后端 200 ≠ 用户成功。**降级响应是 SLI 的最大盲区**。

---

### 案例 ⑤ 多窗口告警配错 —— "短窗 5 秒，告警风暴"

#### 现象

新 SRE 配置告警时把 short window 从 5 分钟改成了 5 秒。上线后告警频率从每周 2 次飙到每天 200 次，PagerDuty 被打爆。

#### 根因

5 秒窗口的样本量太小。100 QPS 的服务在 5 秒窗口内只有 500 个样本，错误率噪声极大 —— 1 个错误就是 0.2%，跨过 14.4× 阈值（0.144%）。

工作簿建议的最短窗口是 **5 分钟**，前提是 QPS 足够大（> 100）让样本稳定。低 QPS 服务（< 10 QPS）甚至要把短窗调到 10 分钟以上。

更微妙的问题：**短窗 / 长窗的比例必须适当**。Google 推荐的比例是 **12:1**：

| 燃烧率 | 长窗 | 短窗 | 比例 |
| --- | --- | --- | --- |
| 14.4× | 1h | 5m | 12:1 |
| 6× | 6h | 30m | 12:1 |
| 3× | 1d | 2h | 12:1 |

短窗 / 长窗 = 1/12，意味着短窗内的"瞬时燃烧"必须持续到长窗也超阈值才告警，自然过滤掉一过性抖动。

#### 修复

```yaml
- alert: HighBurnRate
  expr: |
    (
      slo:error_rate:ratio_5m > (14.4 * 0.001)   # 5m 短窗
      and
      slo:error_rate:ratio_1h > (14.4 * 0.001)   # 1h 长窗
    )
  for: 2m
  labels:
    severity: page
```

低 QPS 服务的兜底：

```yaml
- alert: HighBurnRateLowQps
  expr: |
    (
      slo:error_rate:ratio_10m > (14.4 * 0.001)
      and
      slo:error_rate:ratio_2h > (14.4 * 0.001)
      and
      slo:request_rate:1h > 1     # 至少有流量再告警
    )
  for: 5m
```

最后那个 `slo:request_rate:1h > 1` 是关键：**没流量就别告警**。否则深夜流量低谷一个 5xx 就是 100% 错误率。

#### 验证

跑一周新告警，对比：
- 告警总数（应降到原来 1/10）
- 误报率（手工标注每个告警是否真事故）
- 检测延迟（首次告警到事故开始的间隔）

观察平均检测延迟从 5 分钟变成 7 分钟（多 2 分钟换 90% 假阳性减少，划算）。

> **口诀**：短窗 5m 是底线，低 QPS 服务必须放到 10m。**短窗/长窗 = 1/12**。无流量必须排除。

---

### 案例 ⑥ "假性恢复" —— "故障已修，预算却跳回来了"

#### 现象

凌晨 3 点一次故障，错误率 30%，烧掉 20% 预算。早上 7 点修复，错误率回正。但预算面板显示 9 点开始预算"恢复"，慢慢回到接近 100%。SRE 一脸懵：**预算应该是单调递减的，怎么会涨回来？**

#### 根因

30 天滚动窗口的固有现象。错误预算是**过去 30 天的累积**：

```
当前剩余预算 = 1 - (过去 30 天的错误数 / 过去 30 天的总请求数 / 0.001)
```

凌晨故障当下，5xx 集中。30 天滚动窗口里这部分错误占比上升，预算下降。但**随着时间推进，滚动窗口的"起点"也在前进**：

```
T0（故障当时）：窗口 = [T0-30d, T0]，故障在窗口内
T0+30d：    窗口 = [T0, T0+30d]，故障刚好在边界
T0+30d+1s：窗口 = [T0+1s, T0+30d+1s]，故障已经"滑出"窗口
```

故障"滑出"窗口的瞬间，预算计算里的分子（错误数）消失，预算"自动恢复"。但**生产实际并没有变好**，只是窗口统计学上的恢复。

更糟糕的是：**这个滑出过程不是瞬间的**。30 天窗口的尾巴上每滑过 1 分钟，就少 1 分钟的旧错误数据。所以从 T0+30d 开始，预算曲线慢慢往上抬，看起来"在恢复"。

#### 修复

**第一**：在 dashboard 上同时画"30 天预算"和"7 天预算"。短窗口预算更敏感，能让你看到"真实当前是否在烧"。

```promql
# 7 天预算（更短期）
1 - (
  (
    1 - sum(increase(http_requests_total{code!~"5.."}[7d]))
        / sum(increase(http_requests_total[7d]))
  ) / 0.001
)
```

**第二**：在事故记录里写清楚"事故消耗预算 X%"，不依赖事后预算曲线读数。

**第三**：考虑用**日历窗口**（calendar window，如自然月）替代滚动窗口。日历窗口下故障不会"滑出"，月底前都能看到那次事故的影响。但代价是月初月末感受不同（月初故障对全月预算影响小、月末故障影响大）。

工作簿里推荐：**日常监控用滚动窗口（更敏感），季度复盘用日历窗口（更稳定）**。

#### 验证

故事完整复盘：
- 故障时间 T0、消耗预算 X%（事故记录）
- T0+30d 那天预算曲线开始抬升 —— 标记为"窗口滑出"，不是"系统真好"
- 内部沟通时把这个机制说清楚，避免误读

> **口诀**：滚动窗口的"假性恢复"是统计学产物。**事故记录写消耗值，别看曲线读数**。日历 vs 滚动各有用武之地。

---

### 案例 ⑦ 用户级 SLO 与组件级监控不对齐

#### 现象

订单服务过去 7 天 SLO 100% 达标。但客服工单多了 3 倍，全是"下单按钮没反应"。SRE 一查：MySQL CPU 95%、Redis 主从延迟 2 秒、Nginx 后端连接池 100% 占用 —— **每个组件都在边缘、单看每个 SLI 都没破 SLO**。

#### 根因

SLO 只盯订单服务自身的 5xx 比例。但用户的"下单"路径是：

```
用户点击 → 前端 → API 网关 → 订单服务 → 库存服务 → 支付服务 → MySQL/Redis
```

任何一环的劣化都让用户感受变差。**单个组件的 SLO 全绿不等于用户体验绿**。这是工作簿强调"用户级 SLO"的核心原因。

#### 修复

**第一**：建立"用户级 SLO" —— 跟随真实业务流程。

```promql
# 用户级"下单成功率"SLI：从用户视角度量整条链路
sum(rate(business_event_total{
  event="order_placed",
  result="success"
}[5m]))
/
sum(rate(business_event_total{
  event="order_attempted"
}[5m]))
```

业务事件埋点位置：**前端记录"用户开始下单"、后端记录"订单写库成功"**。

**第二**：把组件 SLI 作为"次要指标"。组件 SLI 用来定位故障，不直接挂 SLO。

**第三**：用户级 SLO 触发告警时，自动展开组件级 SLI 的快照（链路追踪 + 各服务 SLI），帮助快速定位。

```yaml
- alert: OrderUserSloBurn
  expr: ...
  annotations:
    runbook: |
      Triggered user-level SLO. Auto-attached components:
      - {{ printf "histogram_quantile(0.99, mysql_query_duration_seconds_bucket)" | query }}
      - {{ printf "redis_replication_lag_seconds" | query }}
      - {{ printf "order_service_5xx_ratio_5m" | query }}
```

#### 验证

故障重现演练：故意把 Redis 主从延迟拉高到 5 秒，看：

| 监控层 | 是否触发告警 |
| --- | --- |
| 组件级（Redis lag > 1s） | 是 |
| 服务级（订单服务 5xx） | 否（服务降级返回 200） |
| 用户级（下单成功率） | **是**（用户实际下单失败） |

只有用户级 SLO 能在"组件抖动 + 服务降级"的场景下报警。

> **口诀**：用户级 SLO 是承诺、组件 SLI 是诊断。**单个组件 SLI 全绿不等于用户体验绿**。

---

### 案例 ⑧ SLO 与发布管理脱节 —— "预算烧光了，但今晚还要发版"

#### 现象

某季度第三周服务 SLO 已超支 -3%。但产品需求强势：双十一前必须上线一个新功能。团队顶着压力发布，发布后又触发新故障，预算继续亏。

#### 根因

**SLO 不和发布流程联动 = SLO 没用**。Google 工作簿的核心论点：错误预算的真正威力是**强制把"业务方需求 vs 稳定性"变成数字博弈**。

如果团队没有"预算用完冻结发布"的硬性流程，SLO 就只是一张漂亮的图表。常见的脱节模式：
- 没有自动化检查（PR 合并不需要看 SLO）
- 没有审批流（产品/老板一句话就能绕开）
- 没有事后追责（违规发布没后果）

#### 修复

**第一**：CI/CD 加 SLO 检查。

```yaml
# GitLab CI 示例
stages:
  - slo-check
  - deploy

slo-gate:
  stage: slo-check
  script:
    - |
      BUDGET=$(curl -s 'http://prometheus/api/v1/query?query=slo:budget_remaining' \
        | jq -r '.data.result[0].value[1]')
      echo "Budget remaining: $BUDGET"
      if (( $(echo "$BUDGET < 0.2" | bc -l) )); then
        echo "Budget < 20%, deployment blocked"
        exit 1
      fi
  only:
    - main
```

预算 < 20% 时，CI/CD 红灯。要发布需要走特殊审批流。

**第二**：发布管理面板上加"当前预算"。

每次发布前显示：
```
当前剩余预算：68% (29 天周期)
当前燃烧率：1.2× (健康)
本次发布预计影响（基于历史回滚率）：~2%
继续发布？[Y/n]
```

**第三**：紧急发布的"预算挪用"流程化。

如果业务方坚持发布、SRE 必须挡，预先建立 escalation 路径：
1. SRE 引用 SLO 数据 → 业务方说服老板
2. 老板签字 → 流程归档
3. 后续故障归因到这次"挪用发布"

把对话变成纸面流程，避免口头压力。

#### 验证

季度复盘看：
- 预算烧光时是否真的冻结发布
- 强行发布的次数 vs 后续事故数
- 业务方对 SLO 的尊重程度（quantitative：被强行绕开的次数）

如果一个季度有 5 次"强行绕开"，说明 SLO 不被尊重，要重新设计流程。

> **口诀**：SLO 不进 CI/CD = SLO 没用。**预算冻结必须是硬流程，不是软建议**。

---

### 案例 ⑨ 死告警 —— "SLO 三个月没达标，但没人改"

#### 现象

某老服务 SLO 99.9%，过去 3 个月实际只有 99.6%。告警每周触发 4-5 次，被值班逐渐当成"已知问题"忽略。新来的 SRE 接手后吓一跳：这个告警根本是"死告警"，没有任何修复动作。

#### 根因

**SLO 三个月不达标，本质是 SLO 设计错了**。要么：
- SLO 太严：实际能力达不到，但目标定高了
- SLI 错了：测量了错误的东西
- 服务真的需要修：但没排期

无论哪种，**SLO 不能"持续告警 + 持续不修"**。这种状态会让所有告警贬值：值班看到告警的第一反应不是"出事了"而是"又是它"。

#### 修复

**第一**：建立 SLO 季度 review 机制。每个季度过一遍所有 SLO：

| SLO | 过去 90 天达成 | 决策 |
| --- | --- | --- |
| 订单 99.95% | 99.97% ✓ | 维持 |
| 搜索 99.9% | 99.95% ✓ | 收紧到 99.95% |
| 推荐 99.9% | 99.6% ✗ | **降到 99.5% 或修代码** |

**第二**：长期不达标的 SLO 给团队"两条路"：

```
路径 A：放低 SLO（接受现实）
  - 用户接受度 review
  - 商务合同 review（SLA 还安全吗？）
  - 写文档说明降级原因

路径 B：投入修复
  - 列出根因 top 3
  - 估算修复工时
  - 排进路线图
```

**绝对不能选第三条路：维持 SLO + 不修 = 死告警**。

**第三**：告警上线时间锁。同一个告警持续触发 30 天还没消失，自动 mute 并通知 owner："你这告警死了，要么修要么改 SLO"。

#### 验证

跑一次"告警审计"：

```bash
# 拉过去 90 天的告警历史
amtool alert query --start='90d ago' \
  --output json \
  | jq '[.[] | {alert: .labels.alertname, count: 1}] | group_by(.alert) | map({alert: .[0].alert, count: length}) | sort_by(.count) | reverse'
```

输出 top 20 告警：
- 告警名 + 90 天触发次数
- 标记每个的"是否真有人响应、是否有修复"
- 死告警直接 review 走流程

> **口诀**：SLO 三个月不达标 = 设计错了。**不修就降 SLO，不能让告警变成噪声**。

---

## 5. 上线检查清单

打印贴墙 / 加进 SRE Code Review checklist。

### SLI 设计层
- [ ] SLI 从用户视角设计（前端埋点优先于后端日志）
- [ ] 每个 SLI 都是"good / total"或"event count"形式（可计算预算）
- [ ] 区分 Availability / Latency / Throughput / Quality 四类
- [ ] 降级响应、空响应、长延迟都纳入 SLI 错误统计
- [ ] 测量点尽可能贴近用户（合成监控、客户端 SDK 上报）

### SLO 目标层
- [ ] SLO 数字基于历史数据 + 业务承受力，不拍脑袋
- [ ] 内部 SLO 比 SLA 严一档（缓冲）
- [ ] SLO 周期明确（30 天滚动 / 自然月）
- [ ] SLO 文档化，包括 SLI 定义、阈值、决策动作
- [ ] 用户级 / 业务级 SLO 优先于组件级 SLO

### 错误预算层
- [ ] 错误预算 = 1 - SLO，定期可视化
- [ ] 预算 dashboard 同时显示 30 天 / 7 天两种窗口
- [ ] 预算消耗事件记录到事故库，方便复盘
- [ ] 预算 < 20% 时冻结非紧急发布
- [ ] 预算超支时全员事故响应

### 告警层
- [ ] 多窗口多燃烧率告警（5m + 1h、30m + 6h、2h + 1d）
- [ ] 短窗 / 长窗 = 1/12 比例
- [ ] 低 QPS 服务短窗放宽到 10m+，且加流量门槛
- [ ] 告警 `for: 2m+` 防抖动
- [ ] severity 分级（page / ticket / info）路由到不同通道

### 工具层
- [ ] 用 Sloth / Pyrra / OpenSLO 之一管理 SLO 配置
- [ ] SLO YAML 进版本控制（GitOps）
- [ ] PR 修改 SLO 必须 review
- [ ] CI/CD 流水线集成预算检查（< 20% 阻断发布）
- [ ] Alertmanager 路由按 severity 区分（紧急 → PagerDuty、中级 → Opsgenie）

### 流程层
- [ ] 季度 SLO review 机制（达标率、是否调整）
- [ ] 死告警审计（30 天未解决自动 mute + 通知 owner）
- [ ] SLA 签订必须 SRE review（SLO 缓冲是否够）
- [ ] 强行绕过 SLO 的发布走特殊审批流并归档
- [ ] 跨团队 SLO（共享依赖）有明确归属

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### Google SRE 原书与工作簿
- 《Site Reliability Engineering》第 3、4 章奠定 SLO 哲学，第 4 章错误预算章节是行业奠基之作
- 《The Site Reliability Workbook》第 2、5 章给出可操作模板，多窗口多燃烧率告警就在这里
- Google Cloud SRE Toolkit 仓库里有真实在用的 SLO 配置样本

### 错误预算策略与发布管理
- "Progressive delivery"（渐进式发布）和 SLO 的天然耦合：金丝雀流量基于 SLO 自动暂停
- Argo Rollouts、Flagger 集成 Prometheus 自动判断金丝雀健康
- 错误预算驱动的发布频率优化 —— 预算多就快发、预算紧就慢发

### SLO 驱动的容量规划
- 历史 SLO 数据 + 流量预测 → 容量需求模型
- 如何把 SLO 反推到 SLI 阈值，再反推到容量数（CPU/内存/连接数）
- 容量不足导致 SLO 退化 vs 单纯故障导致 SLO 退化的区分

### 用户级 SLO 设计
- "Critical User Journey"（CUJ）方法论：从用户路径出发拆 SLO
- 多步流程的 SLO 复合：登录 SLO × 搜索 SLO × 下单 SLO 不是简单相乘
- A/B 测试期间 SLO 怎么不被新功能拉低

### SLO 与混沌工程
- 错误预算是混沌工程的"许可证" —— 预算多才敢做实验
- 故意消耗错误预算的实验设计（注入故障、关 AZ）
- 实验后预算消耗 vs 实际 SLO 影响的对账

### 跨团队 SLO 协调
- 共享依赖的 SLO 归属（DB、Kafka、L7 网关谁负责 SLO）
- 服务依赖图上的 SLO 传播：A 调 B，B 的 SLO 99% 时 A 能承诺 SLO 多少？
- "用户级 SLO" 和 "服务 SLO" 的耦合点

### SLO 的自动化优化
- 基于历史数据自动推荐 SLO 数值（机器学习方向）
- "预算预测"：当前燃烧率下未来 7 天预算曲线
- 异常检测 + SLO：用 Prophet/Holt-Winters 做基线预测，偏离后自动告警

---

## 7. 引用

- Google SRE Books 三部曲：[SRE Book](https://sre.google/sre-book/), [Workbook](https://sre.google/workbook/), [Building Secure Reliable Systems](https://sre.google/books/)
- 工作簿核心章节：[Implementing SLOs](https://sre.google/workbook/implementing-slos/), [Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
- [Sloth](https://github.com/slok/sloth) —— Prometheus SLO 生成器
- [Pyrra](https://github.com/pyrra-dev/pyrra) —— K8s Native SLO 控制器
- [OpenSLO](https://github.com/OpenSLO/OpenSLO) —— SLO 标准声明语言
- [SLOMaker](https://github.com/openslo/slo-generator) —— 多 backend 支持的 SLO 生成器
- Liz Fong-Jones, *SLOs are still hard* —— 大量真实坑和反思（QCon 演讲）
- Niall Murphy 的 "Implementing SLOs from Scratch" —— SREcon 系列演讲
- Datadog SLO 文档 —— 商业方案视角，工程化案例多
- Honeycomb SLO 文档 —— 基于事件的 SLI 实现思路（不依赖 Prometheus）

---

> 配套：[可观测性主题](/article/observability/) | [Prometheus 指标设计](/article/observability/prometheus) | [故障定位方法论](/article/observability/troubleshooting)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你觉得这个深度对，告诉我下一篇该写哪个。
