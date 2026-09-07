# 核心术语表

> 这门课会反复用到的词。**不用一次读完**——第一遍扫一下建立印象，后面每步遇到不认识的词回来查。
>
> 「步骤」列指这个词在哪一步会真正用到。带 `code` 格式的是 Anthropic Messages API 里的**真实字段名**，写代码时会原样出现。

---

## 一、模型与 API 层

| 术语 | 英文 / 字段 | 说明 | 步骤 |
|---|---|---|---|
| 大模型 | LLM (Large Language Model) | 一个**无状态纯函数**：给它对话历史，返回一段内容。它不能读文件、跑命令、上网 | 1 |
| 消息数组 | `messages[]` | 完整对话历史。每轮请求都要把**全部历史重新发一遍**——模型不记得上一次说过什么 | 1 |
| 角色 | `role` | `user` / `assistant`。工具结果也是用 `user` 角色发回去的 | 2 |
| 系统提示 | `system` | 给模型的全局指令，不属于对话历史。放身份、规则、环境信息 | 1 / 6 |
| 内容块 | content block | 一条消息的 `content` 是一个**数组**，元素类型有 `text` / `tool_use` / `tool_result` / `thinking` 等 | 1 |
| 停止原因 | `stop_reason` | 模型为什么停下来：`end_turn`（说完了）、`tool_use`（要调工具）、`max_tokens`（被截断）、`refusal`（拒绝） | 1 |
| 输出上限 | `max_tokens` | **硬性截断**，模型不知道它的存在，撞上就是话说到一半没了。别设太小 | 1 |
| 流式输出 | streaming | 边生成边返回。长输出必须用，否则容易 HTTP 超时 | 1 |
| Token | token | 计费和上下文的基本单位。中文大约 1 字 ≈ 1~2 token | 1 / 5 |
| 上下文窗口 | context window | 一次请求能塞进去的 token 上限（当代模型多为 200K~1M） | 5 |
| 思考 | `thinking` | 模型回答前的内部推理。当代模型用 `{ type: "adaptive" }`，由模型自己决定想多久 | 1 |
| 努力度 | `output_config.effort` | `low` / `medium` / `high` / `xhigh` / `max`，控制思考深度与花费。**降本增效的第一个旋钮** | 5 |

---

## 二、Agent 与 Harness

| 术语 | 说明 | 步骤 |
|---|---|---|
| **Agent** | 模型在一个循环里，**自主决定**调哪些工具、调几次，直到目标达成 | 2 |
| **Harness** | 让这个循环跑起来的工程代码：执行工具、管上下文、卡权限、处理错误、存状态。**本课的主题** | 全程 |
| **Agent Loop** | harness 的心脏。`请求模型 → 拿到 tool_use → 执行 → 结果喂回去 → 再请求`，直到 `stop_reason !== "tool_use"` | 2 |
| **Workflow** | 控制流由**你的代码**写死（先 A 再 B）。可预测、可测试、便宜。**默认应该先选它** | — |
| **Agentic Workflow** | 中间态：大框架你定，某几步交给模型自己决定 | — |
| **ReAct** | Reason + Act，早期的经典模式：思考 → 行动 → 观察 → 再思考。现在的 tool_use 循环就是它的工程化版本 | 2 |
| **回合 / 轮次** | turn / iteration | 循环跑一圈。一个复杂任务可能要 20~50 轮 | 2 |
| **Deployment** | 谁提供跑代码的机器。**和 harness 是两个独立问题**——详见 [概念课 §3](01-agent-and-harness.md) | 8 |

---

## 三、工具调用

这一组是**第 2 步的全部内容**，也是最容易写错的地方。

| 术语 | 英文 / 字段 | 说明 | 步骤 |
|---|---|---|---|
| 工具调用 | Function Calling / Tool Use | 模型输出一个结构化请求，说"我想调 X，参数是 Y"。**它只是提出请求，不会真的执行** | 2 |
| 工具定义 | `tools[]` | 每个工具有 `name`、`description`、`input_schema`（JSON Schema） | 2 |
| 工具描述 | `description` | **它就是 prompt 的一部分**。模型完全靠这段文字判断该不该调。写不好，工具就形同虚设 | 3 |
| 调用块 | `tool_use` | 模型返回的调用请求，含 `id`、`name`、`input` | 2 |
| 结果块 | `tool_result` | 你执行完塞回去的结果，必须带 `tool_use_id` 与请求配对 | 2 |
| 配对 ID | `tool_use_id` | **漏一个整个请求就 400**。每个 `tool_use` 都必须有对应的 `tool_result` | 2 |
| 错误回传 | `is_error: true` | 工具失败时用它标记，**不要抛异常中断循环**。让模型看到错误自己重试——这是 Agent "自我修复"的来源 | 2 / 3 |
| 并行工具调用 | parallel tool use | 一条 assistant 消息里可能有**多个** `tool_use`。所有 `tool_result` 必须放在**同一条** user 消息里返回，拆开会让模型以后不再并行调用 | 2 |
| 严格模式 | `strict: true` | 保证模型给的参数一定符合你的 schema。需要 schema 有 `additionalProperties: false` 和 `required` | 3 |
| 结构化输出 | `output_config.format` | 约束模型**回复本身**的格式（不是工具参数） | 3 |
| MCP | Model Context Protocol | 工具接入的**事实标准协议**。让工具能跨 harness 复用，Claude Code / Cline / OpenCode 都原生支持 | 8 |

---

## 四、上下文与记忆

| 术语 | 说明 | 代价 | 步骤 |
|---|---|---|---|
| **截断** truncation | 单个工具输出超过 N 行就砍掉 | 最便宜，可能丢关键信息 | 5 |
| **Prompt Caching** | 把稳定前缀缓存住，只为增量付费。字段 `cache_control` | 几乎无损，**纯赚** | 5 |
| **缓存前缀** | 缓存按**前缀精确匹配**，渲染顺序是 `tools` → `system` → `messages`。前缀改一个字节，后面全部失效 | — | 5 |
| **缓存命中检查** | `usage.cache_read_input_tokens`。如果反复请求它一直是 0，说明有东西在悄悄让缓存失效（比如 system 里放了 `new Date()`） | — | 5 |
| **Context Editing** | 直接**删掉**旧的 tool_result | 信息真的没了 | 5 |
| **Compaction** | 把旧历史**总结**成摘要 | 保留语义，但要多一次模型调用，摘要有损 | 5 |
| **短期记忆** | 就是 `messages[]` 本身，随会话消失 | — | 5 |
| **长期记忆** | 跨会话持久化。做法：文件（如 `CLAUDE.md`）或向量库检索 | — | 8 |
| **RAG** | Retrieval-Augmented Generation，检索增强。先搜出相关片段再塞进上下文 | — | 8 |
| **上下文腐化** | context rot。历史太长时模型会忽略中间部分——**上下文不是越长越好** | — | 5 |

> 处理顺序永远是：**先做免费的（缓存、截断），再做有损的（删除、压缩）**。

---

## 五、权限与安全

| 术语 | 说明 | 步骤 |
|---|---|---|
| **沙箱** sandbox | 限制 Agent 能碰的范围。最低配是路径沙箱，进阶是容器隔离 | 4 |
| **路径逃逸** | Agent 用 `../../` 或绝对路径跑出工作目录。**必须显式拦截** | 4 |
| **权限模式** permission mode | 只读 / 需确认 / 自动放行。让用户按场景切换 | 4 |
| **人在回路** HITL (Human-in-the-Loop) | 危险操作前停下来等用户确认 | 4 |
| **Guardrail** | 护栏。输入输出的规则校验层，挡住越界请求 | 4 |
| **审计日志** audit log | 每次工具调用都记录，出事能回溯 | 4 / 7 |
| **提示注入** prompt injection | 攻击者把指令藏在 Agent 会读到的内容里（网页、文件、issue），劫持它的行为。**所有外部内容都应视为数据而非指令** | 4 |

---

## 六、多 Agent 与扩展

| 术语 | 说明 | 步骤 |
|---|---|---|
| **子代理** subagent | 主 Agent 派一个独立上下文的小弟去干活，只把结论带回来。**主要目的是省上下文**，不是"更聪明" | 8 |
| **交接** handoff | 把对话控制权整个转交给另一个 Agent | 8 |
| **编排** orchestration | 决定多个 Agent / 步骤怎么串起来跑 | 8 |
| **Hooks** | 在 harness 生命周期的固定点插入自己的代码（工具调用前后、会话结束等） | 8 |
| **Skill** | 打包好的一组指令 / 流程，按需加载进上下文 | 8 |
| **渐进式加载** progressive disclosure | 平时每个能力只在上下文里留一行描述，真要用了才加载完整 schema。**上下文优化的前沿做法** | 8 |

---

## 七、评估与可观测性

| 术语 | 说明 | 步骤 |
|---|---|---|
| **Trace / Span** | 一次完整运行的调用链记录 / 其中的单个环节。调试 Agent 的基本单位 | 7 |
| **Eval 集** | 一组固定的测试任务 + 期望结果。**20~50 个真实场景就够起步** | 7 |
| **自动判分** | 跑通测试 / 文件 diff 是否正确 / 用另一个模型当裁判 | 7 |
| **LLM-as-Judge** | 用模型给模型的输出打分。便宜灵活，但裁判本身也会错，要抽样人工校验 | 7 |
| **通过率** pass rate | eval 集里做对了几个。**和成本一起看，只看一个会误判** | 7 |
| **Token 成本核算** | 从 `usage` 字段累计 input / output / cache token，换算成钱 | 7 |
| **不确定性** | 同样的输入，Agent 两次跑出来可能不一样。所以**单次试跑说明不了任何问题** | 7 |

---

## 八、最容易混淆的几组词

这一节是整个术语表里最值得细看的部分。

### `tool_use` vs `tool_result` vs `tool_use_id`

一次工具调用涉及三个东西，方向不同：

```
模型 ──► tool_use  { id: "abc", name: "read_file", input: {...} }
                                  ↓ 你的代码执行
你  ──► tool_result { tool_use_id: "abc", content: "文件内容..." }
```

`tool_use` 是模型发的**请求**，`tool_result` 是你回的**答复**，`tool_use_id` 是把两者绑在一起的**绳子**。

### Tool Runner vs Claude Agent SDK

名字像，量级差一个数量级：

| | Tool Runner | Claude Agent SDK |
|---|---|---|
| 包 | `@anthropic-ai/sdk`（普通 API SDK 的一部分） | `@anthropic-ai/claude-agent-sdk`（独立产品） |
| 提供 | 只有一个循环辅助函数 | Claude Code 全套 harness |
| 内置工具 | **没有**，全得你自己写 | Read / Write / Edit / Bash / Glob / Grep / WebSearch |
| 还带 | — | 子代理、hooks、权限系统、上下文管理 |

### Context Editing vs Compaction

都是给上下文瘦身，但性质完全不同：**Editing 是删除**（信息没了），**Compaction 是总结**（信息还在，但有损且要花一次模型调用）。

### `max_tokens` vs `task_budget`

| | `max_tokens` | `task_budget` |
|---|---|---|
| 性质 | 硬性截断 | 建议性预算 |
| 模型知道吗 | **不知道**，撞上就断在半句话 | **知道**，会自己调节节奏收尾 |
| 范围 | 单次响应 | 整个 agent 任务 |

### Agent vs Harness

最根本的一组。**Agent 是行为**（模型在循环里自主用工具），**Harness 是代码**（让这个循环能跑的那层工程实现）。

同一个模型套不同的 harness，能力差距可以非常大——所以"哪个模型最会写代码"这个问题，一半答案在 harness 上。

---

## 延伸阅读

- [第 0 课：AI Agent 与 Agent Harness](01-agent-and-harness.md) —— 这些术语的完整来龙去脉
- [Anthropic Messages API 文档](https://docs.claude.com/en/api/messages)
- [Model Context Protocol 规范](https://modelcontextprotocol.io)
