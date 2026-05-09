# Agent 设计模式（速记）

> "Agent" 已经从 demo 走进生产。本文用工程师视角讲清楚：**什么是 Agent、它有哪些设计模式、什么时候不应该用 Agent**。
>
> ReAct / Plan-and-Execute / Reflection / Multi-Agent / 上下文管理 / 死循环 / JSON Schema / 9 个生产事故 → **[Agent 设计模式深入（深文）](/article/ai/agent-deep)**

---

## 1. 什么是 Agent？

> **Agent = LLM + 工具 + 循环**。LLM 决定下一步做什么，调用工具取得新观察，再决定下一步……直到任务完成。

```
       ┌─────────────┐
       │   用户目标   │
       └──────┬──────┘
              │
   ┌──────────▼──────────┐
   │       LLM            │← 系统 prompt + 工具集 + 历史
   │  （Reason + Plan）   │
   └──────┬───────────────┘
          │ 工具调用
   ┌──────▼──────┐
   │    Tools     │  搜索、读写文件、API、SQL、子 Agent
   └──────┬──────┘
          │ 观察
          └────────►（回到 LLM）
```

**Anthropic 官方区分**：
- **Workflow**：路径预先编排，LLM 在固定节点出现；
- **Agent**：LLM 自己决定下一步做什么，**循环**直到完成。

> 90% 实际应用其实是 Workflow，不是 Agent。**别为了"显得聪明"硬上 Agent**。

---

## 2. 五大经典工作流模式（Workflow Patterns）

### 2.1 Prompt Chaining（提示链）
```
LLM-1 → 中间结果 → LLM-2 → ... → 最终输出
```
适用：任务可拆成顺序步骤，每步要求清晰。
例：写文章 → 提纲 → 段落扩写 → 校对。

### 2.2 Routing（路由）
```
                 ┌→ LLM-客服分诊
LLM-分类器  ──→  ├→ LLM-技术支持
                 └→ LLM-售前
```
按输入类型路由到专门的 prompt 或模型。
例：客服系统按工单类型分流。

### 2.3 Parallelization（并行）
- **分片**：长文档切分并行处理；
- **投票**：同一问题多次采样，做多数表决。

### 2.4 Orchestrator-Workers（调度器+工人）
```
        Orchestrator
       /     |      \
  Worker-1 Worker-2 Worker-3
```
调度器拆任务，工人并行执行，调度器汇总。
例：代码生成时拆模块并行写。

### 2.5 Evaluator-Optimizer（评测-优化器）
```
LLM-生成 → LLM-打分 → 不达标？回到生成（带反馈）
```
适用：可迭代改进的任务（写作、翻译、代码）。

---

## 3. 真正的 Agent 模式

### 3.1 ReAct
```
Thought: 我应该先搜索这个问题
Action: search("...")
Observation: <返回结果>
Thought: 信息不够，再查
Action: search("...")
...
Thought: 我有答案了
Final Answer: ...
```

### 3.2 Plan-and-Execute
- Planner 一次性产出执行计划；
- Executor 顺序/并行执行；
- 每步后可让 Planner 决定是否调整。

### 3.3 Reflection（反思）
LLM 完成后，让它（或另一个 LLM）做**自我批评**，再迭代一次。

### 3.4 多 Agent 协作
- 角色分工：Researcher / Coder / Reviewer；
- 通讯协议：消息总线、共享 scratchpad；
- **风险**：通信成本爆炸、责任不清——能用单 Agent 别上多 Agent。

---

## 4. 工具调用（Tool Use）的工程要点

### 4.1 工具描述就是 prompt 的一部分

**工具的 description 比 LLM 的"prompt"更影响调用质量**。模板：

```yaml
name: search_kb
description: |
  在公司内部知识库中搜索文档。
  使用场景：用户问及产品功能、配置、报错时。
  返回：top 5 相关片段（包含 url、score）。
  注意：query 用中文短句，避免长段落。
parameters:
  query:
    type: string
    description: 搜索关键词，建议 3~10 个字
```

### 4.2 工具粒度
- 太细：LLM 要调几十次，token 爆炸；
- 太粗：调用方无法控制行为；
- **甜区**：每个工具完成"一个清晰意图"。

### 4.3 错误反馈
工具失败时，**返回结构化错误**，让 LLM 学会重试：
```json
{
  "error": "INVALID_DATE",
  "message": "日期格式应为 YYYY-MM-DD",
  "hint": "请改写参数后重试"
}
```

### 4.4 长输出截断
工具返回超过 N token 时：
- 截断 + 提示"已截断，可调用 read_more 获取更多"；
- 或预 summarize。

---

## 5. 上下文管理（Agent 最大挑战）

Agent 跑十几步后，对话历史动辄数万 token。两条原则：

### 5.1 主动压缩
- **每 N 步生成摘要**，替换早期消息；
- **结构化记忆**：维护 `state` 字段（当前任务、已知事实、待办），LLM 每步基于 state 而非全历史；
- **检索式记忆**：把对话存向量库，按需召回。

### 5.2 工具结果隔离
- 工具输出用 XML 标签包起来，**用完即丢**；
- 关键事实主动 extract 到 state。

> Anthropic 提供的 Context Editing 思路：调用结束后让 LLM 自己修剪历史。

---

## 6. 何时不要用 Agent

| 场景 | 替代方案 |
| --- | --- |
| 任务路径固定 | Workflow |
| 错误代价高（金融、医疗下单） | 人在回路 + 规则 |
| 实时性要求 < 1s | 单轮模型调用 |
| 输入域已知且枚举有限 | 路由 + 模板 |
| 调试困难、合规要求溯源 | 显式状态机 |

---

## 7. 安全与人在回路

### 7.1 工具能力分级

| 级别 | 例子 | 控制 |
| --- | --- | --- |
| 只读 | search、read_file | 无需确认 |
| 局部写 | save_draft | 自动 |
| 外部影响 | send_email、create_pr | **二次确认** |
| 不可逆 | delete、payment | 二次确认 + 审计日志 |

### 7.2 终止条件
Agent 必须有硬性"停止"：
- 最大步数（如 30 步）；
- 最大 token；
- 检测到死循环（同一 action 重复 N 次）；
- 时长超过 T。

---

## 8. Anthropic Claude Agent SDK 核心理念

Claude Code 与 Anthropic Agent SDK 的设计提炼：
- **工具优先**：把能力变成工具，让模型选；
- **少即是多**：少而精的工具集 > 一堆细粒度工具；
- **结构化输出**：要求 LLM 返回 JSON / XML / 工具调用，而非自由文本；
- **可观测**：每一次工具调用、每一轮 token 都可 trace；
- **prompt 即代码**：版本化、CI、灰度。

```python
# 伪代码：一个最小 Agent 循环
def agent_loop(goal, tools, max_steps=30):
    history = [{"role": "system", "content": SYS_PROMPT}]
    history.append({"role": "user", "content": goal})

    for _ in range(max_steps):
        resp = llm.create(messages=history, tools=tools)
        if resp.stop_reason == "end_turn":
            return resp.content
        for call in resp.tool_calls:
            result = run_tool(call.name, call.args)
            history.append({"role": "assistant", "tool_use": call})
            history.append({"role": "tool", "content": result})
    raise TimeoutError("max_steps reached")
```

---

## 9. Agent 评测方法

| 维度 | 怎么测 |
| --- | --- |
| **任务完成率** | 端到端跑数据集，看是否达成目标 |
| **步数与成本** | 平均步数、总 token、成本 |
| **工具准确率** | 调用对的工具、给对的参数 |
| **稳定性** | 多次跑同一任务，方差 |
| **安全性** | 注入攻击集、危险指令拒答率 |

> 别只看"看上去能跑"——给 100 个任务样本，统计成功率与成本，才是科学评测。

---

## 10. 一份"上线即用"的 Agent 检查表

- [ ] Workflow vs Agent，已论证为什么必须 Agent
- [ ] 工具集 ≤ 10 个，每个 description 详尽且有示例
- [ ] 危险工具二次确认 / 审计
- [ ] 最大步数、最大 token、死循环检测
- [ ] 上下文压缩策略（状态、摘要、检索）
- [ ] 工具失败结构化错误，LLM 可重试
- [ ] 离线评测集，跑出 baseline
- [ ] 在线 trace（每步的 prompt、工具、结果、token）
- [ ] 成本告警（单次任务上限）
- [ ] 用户可中断、可看到 Agent 在做什么

---

## 11. 推荐阅读

- Anthropic：[Building Effective Agents](https://www.anthropic.com/research/building-effective-agents)
- Anthropic：[How we built our multi-agent research system](https://www.anthropic.com/research)
- LangGraph 文档（State Machine 视角）
- Lilian Weng：LLM Powered Autonomous Agents

---

> 上一篇：[RAG 知识库](/article/ai/rag) ｜ 下一篇：[Skill / MCP / 工具调用](/article/ai/skill-mcp)
