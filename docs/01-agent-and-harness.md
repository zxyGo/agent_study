# 第 0 课：AI Agent 与 Agent Harness

> 本课不写代码，只建立心智模型。读完你应该能回答三个问题：
>
> 1. Agent 和一次普通的 API 调用差在哪？
> 2. Harness 到底是什么，由哪几块组成？
> 3. 现在市面上的框架各自在解决哪一块，我该学哪个 / 用哪个？
>
> 生态部分是 **2026-09 的快照**，这个领域半年就会换一批名字，结论会过期，但第一部分的概念不会。

---

## 一、从一次 API 调用说起

大模型 API 本质上是一个**无状态的纯函数**：

```
f(messages[], system, tools[]) -> { content[], stop_reason }
```

你给它对话历史，它返回一段内容。它**不能**做任何事情——不能读文件、不能跑命令、不能上网。它唯一的"动作能力"，是在返回内容里放一个 `tool_use` 块，意思是：

> "我想调用 `read_file`，参数是 `{ path: 'src/index.ts' }`，麻烦你帮我执行一下，把结果告诉我。"

注意**"麻烦你"**三个字。模型只是**提出请求**，真正去执行的，是模型外面的那段代码。

这段代码就是 **Harness（挽具 / 载具）**。

### 一张图说清楚

```
┌──────────────────────────────────────────────────────┐
│                Harness  (你写的代码)                   │
│                                                      │
│   ┌──────────┐                                       │
│   │  循环控制  │ ◄────────── stop_reason ──────────┐   │
│   └────┬─────┘                                   │   │
│        │  messages[] + tools[]                   │   │
│        ▼                                         │   │
│   ┌──────────────────────────────────────┐       │   │
│   │      大模型 (无状态 · 只会说话)          │ ──────┘   │
│   └──────────────────────────────────────┘           │
│        │  tool_use: read_file(...)                   │
│        ▼                                             │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐          │
│   │ 权限检查 │ ─► │ 工具执行 │ ─► │ 结果回填 │ ────┐     │
│   └─────────┘    └────┬────┘    └─────────┘    │     │
│                       │                        │     │
│   ┌───────────────────┼────────────────────┐   │     │
│   │ 上下文管理：截断 / 压缩 / 缓存           │ ◄─┘     │
│   └───────────────────┼────────────────────┘         │
└───────────────────────┼──────────────────────────────┘
         ▲              ▼
    用户输入      真实副作用：文件系统 / Shell / 网络
```

**一句话定义：**

- **Agent** = 模型在一个循环里，自主决定调用哪些工具、调用几次，直到把目标达成。
- **Harness** = 让这个循环真正跑起来的那层工程代码：执行工具、管上下文、卡权限、处理错误、保存状态。

模型提供**智能**，harness 提供**手脚、记忆和缰绳**。同一个模型套不同的 harness，能力差距可以非常大——这也是为什么"哪个模型最会写代码"这个问题，一半答案其实在 harness 上。

---

## 二、Harness 的六个组成部分

后面 8 步教程，本质就是按顺序把这六块实现一遍。

### 1. Agent Loop（循环）—— 心脏

```ts
// 这十几行就是所有 Agent 的核心。剩下的全是往它上面挂东西。
let messages = [{ role: "user", content: userInput }];

while (true) {
  const res = await client.messages.create({ model, messages, tools });
  messages.push({ role: "assistant", content: res.content });

  if (res.stop_reason !== "tool_use") break; // 模型说完了，退出

  const results = await Promise.all(
    res.content.filter((b) => b.type === "tool_use").map(runTool)
  );
  messages.push({ role: "user", content: results }); // 结果喂回去，继续下一轮
}
```

几个**新手必踩的坑**（第 2 步会详细讲）：

- 一条 assistant 消息里可能有**多个** `tool_use`（并行工具调用），所有 `tool_result` 必须放在**同一条** user 消息里返回。拆成多条会让模型以后不再并行调用。
- 工具执行失败**不要抛异常中断循环**，而要返回 `tool_result` 且 `is_error: true`，让模型自己看到错误并重试——这是 Agent 具备"自我修复"能力的来源。
- `tool_use_id` 必须严格配对，漏一个整个请求就 400。

### 2. Tools（工具集）—— 手脚

工具的 `description` 和 JSON Schema **就是 prompt 的一部分**，模型完全靠它判断该不该调、怎么调。工具设计是 harness 里最影响效果、也最被低估的一环。

一个典型编码 Agent 的工具面：`read` / `write` / `edit` / `bash` / `glob` / `grep` / `web_fetch`。设计上的经典权衡：

| 取向 | 做法 | 优点 | 缺点 |
|---|---|---|---|
| **少而通用** | 只给一个 `bash` | 模型自由度高，你代码少 | 难限权、难观测、输出不可控 |
| **多而专用** | read/edit/grep 各一个 | 好限权、好观测、结果结构化 | schema 多，占上下文，模型要学 |

现实中的成熟 harness 都是**混合**：高频操作给专用工具（省 token、可校验），长尾操作留一个 `bash` 兜底。

### 3. Context Management（上下文）—— 记忆

Agent 跑长任务必然撞上下文墙。手段有四层，代价递增：

| 手段 | 做什么 | 代价 |
|---|---|---|
| **截断** | 单个工具输出超过 N 行就截断 | 最便宜，但可能丢关键信息 |
| **Prompt Caching** | 把稳定前缀缓存住，只为增量付费 | 几乎无损，纯赚；但前缀改一个字节就全部失效 |
| **Context Editing** | 直接**删掉**旧的 tool_result | 信息真的没了 |
| **Compaction** | 把旧历史**总结**成摘要 | 保留语义，但要额外一次模型调用，且摘要有损 |

顺序很重要：**先做免费的（缓存、截断），再做有损的（删除、压缩）**。

### 4. Permissions & Safety（权限）—— 缰绳

Agent 会真的删你的文件。必须有：

- **路径沙箱**：所有文件操作限制在工作目录内，挡住 `../../` 和绝对路径逃逸
- **命令黑名单 / 白名单**：`rm -rf`、`curl | sh`、写 `~/.ssh` 之类先拦下
- **权限模式**：只读 / 需确认 / 自动放行，让用户按场景切换
- **审计日志**：每次工具调用都记下来，出事能回溯

### 5. System Prompt 与环境注入（大脑的初始化）

模型不知道今天几号、你在什么目录、这个项目用什么框架。harness 要主动把这些塞进 system prompt：工作目录、操作系统、git 分支与状态、项目约定（如 `CLAUDE.md`）、工具使用规范。

还有一类是**行为脚手架**：todo 列表、计划模式、"先读再改"的硬性要求。这些不改模型，但显著改变它的工作方式。

### 6. 可观测性、评估与恢复

**可观测性**：Token 计数与成本统计、流式输出、每次工具调用的 trace、中断（Ctrl+C）后能否续跑、会话持久化。这块决定 harness 是"demo"还是"能用"。

**评估（eval）**：这是最容易被跳过、但决定你能不能持续改进的一环。Agent 的输出是不确定的——同样的输入两次跑出来可能不一样，所以**单次试跑说明不了任何问题**。你需要：

- 一组固定的测试任务（20~50 个真实场景就够起步）
- 一个自动判分方式（跑通测试 / 文件 diff 是否正确 / 用另一个模型当裁判）
- 每次改动后跑一遍，看**通过率和成本**两条曲线

没有这套东西，你改 prompt、换工具描述、调上下文策略，全部只能凭感觉说"好像好点了"。有了它，harness 优化才从玄学变成工程。

> 这是本教程第 7 步的内容。很多入门材料把 eval 完全略过，直接从"跑通"跳到"进阶功能"——那样搭出来的 harness 是调不动的。

---

## 三、两个关键区分（初学者最容易混）

### 区分一：Workflow vs Agent —— 谁决定下一步？

| | Workflow（工作流） | Agent（智能体） |
|---|---|---|
| 控制流 | **你的代码**写死：先 A 再 B，if 就 C | **模型**每轮自己决定下一步 |
| 可预测性 | 高，能测试、能画图 | 低，同样输入可能走不同路径 |
| 成本 | 可控 | 不可控（可能循环 30 轮） |
| 适用 | 步骤已知：抽取 → 校验 → 入库 | 步骤未知："把这个 issue 修了" |

**默认应该选 workflow。** 只有当任务真的无法提前穷举步骤时，才升级到 agent。很多"我要做个 Agent"的需求，其实一个 `if/else` 加两次模型调用就解决了，还更稳更便宜。

### 区分二：Harness vs Deployment —— 谁提供什么？

这是理解整个框架生态的钥匙。有两个**互相独立**的问题：

- **谁写循环？**（harness）
- **谁提供运行的机器？**（deployment）

以 Anthropic 的四种做法为例：

| 做法 | 你写什么 | Harness 谁给 | Deployment 谁给 |
|---|---|---|---|
| **Messages API 手写循环** | 整个 loop | 你自己 | 你自己 |
| **Tool Runner**（`client.beta.messages.tool_runner`） | 只写工具函数 | SDK | 你自己 |
| **Claude Agent SDK**（`@anthropic-ai/claude-agent-sdk`） | 一个 prompt + options | SDK（= Claude Code 全套 + 内置工具） | 你自己 |
| **Managed Agents** | Agent 配置 | Anthropic | **Anthropic**（托管沙箱） |

> ⚠️ **最常见的混淆：`Tool Runner` ≠ `Claude Agent SDK`。**
>
> 前者是普通 API SDK 里的一个循环辅助函数，**没有任何内置工具**，工具全得你自己写；
> 后者是把 Claude Code 整个打包成库，自带 Read/Write/Edit/Bash/Glob/Grep/WebSearch、子代理、hooks、权限系统。
>
> 名字像，量级差着一个数量级。

**我们这门课要手写的，是上表第一行。** 只有手写过一遍，你才看得懂后面三行分别帮你省了什么、又拿走了哪些控制权。

---

## 四、生态地图（2026-09 快照）

市面上笼统叫"Agent 框架"的东西，其实是三个不同的物种，混在一起比较没有意义：**A 类**是你拿来搭 Agent 的库，**B 类**是别人已经搭好的 Agent，**C 类**是插在旁边看数据的平台。

很多教程只讲 A 类（而且往往只讲 LangChain 那一支），这会让人误以为"学 Agent = 学 LangChain"。

### A 类：通用 Agent 框架（你用它构建自己的 Agent）

面向开发者，作为库被引入你的应用。

| 框架 | 语言 | 主攻方向 | 优势 | 劣势 |
|---|---|---|---|---|
| **Claude Agent SDK** | TS / Py | 把 Claude Code 的完整 harness 当库用 | 编码 / 文件系统类任务开箱即用；内置工具 + 子代理 + 权限，工程成熟度最高 | 强绑定 Anthropic；抽象层厚，想改内部循环不容易 |
| **OpenAI Agents SDK** | Py / TS | 轻量的模型驱动循环 + 多 Agent 交接（handoff） | 抽象少、上手 20 分钟；guardrails 与 tracing 干净 | 编排能力弱，复杂分支 / 持久化要自己补；偏 OpenAI 生态 |
| **LangGraph** | Py / TS | **图式**编排：显式状态机 + 持久化 + 断点续跑 | 控制力最强，可中断 / 可恢复 / 可人工介入；生态与可观测性最全 | 概念负担重（node / edge / state / checkpointer），简单场景严重过度设计 |
| **Mastra** | TS | TS 全栈 Agent 框架：workflow + RAG + eval + 部署 | TS 生态里最完整的一体化方案，DX 好 | 相对年轻，社区规模小于 Python 阵营 |
| **Vercel AI SDK** | TS | 统一模型接口 + 流式 UI | 换模型只改一行；前端流式渲染无出其右 | 定位是"模型接入层"而非"Agent 编排层"，复杂 agent 仍要自己搭 |
| **Pydantic AI** | Py | 类型安全 + 结构化输出校验 | Python 里类型体验最好，输出校验强，心智负担低 | 编排偏薄，多 Agent 协作要自己写 |
| **CrewAI** | Py | 角色化多 Agent 协作（研究员 / 写手 / 审校） | 概念直观，多角色任务原型极快 | 抽象偏"拟人"，不易调试；生产可控性一般 |
| **Microsoft Agent Framework** | Py / .NET | 企业级编排（AutoGen + Semantic Kernel 合流） | 微软 / Azure 生态整合，企业合规友好 | 体系庞大，非微软栈收益不明显 |
| **Google ADK** | Py / Java | Gemini 生态的 Agent 开发套件 | 与 Vertex AI / GCP 部署链路顺畅 | 绑定 Google 云 |
| **Strands Agents** | Py | AWS 出品的模型驱动循环 | 简洁，Bedrock 集成好 | 生态与社区仍在早期 |
| **LlamaIndex Workflows** | Py | 事件驱动编排，RAG 出身 | 检索 / 文档类任务积累最深 | 非 RAG 场景优势不突出 |

**License**：LangGraph、Microsoft Agent Framework、CrewAI、OpenAI Agents SDK、LlamaIndex、Pydantic AI 为 MIT；Google ADK、Mastra core 为 Apache 2.0。

### B 类：编码 Agent Harness（成品产品，也是最好的教材）

这类不是给你当库用的，而是**已经做好的 Agent**。但它们大多开源——**读源码是学 harness 最快的路**。

| 产品 | 形态 | 主攻 | 值得学的点 |
|---|---|---|---|
| **Claude Code** | CLI / IDE / Web | Anthropic 官方编码 Agent | harness 工程的参考实现：权限模式、子代理、hooks、skills、上下文压缩 |
| **OpenAI Codex CLI** | CLI | OpenAI 官方编码 Agent | 沙箱执行模型、审批流设计 |
| **OpenCode** | CLI（MIT） | 开源版 Claude Code，provider 无关 | 如何做**模型无关**的 harness 抽象层 |
| **Cline** | VS Code 插件 | IDE 内的自主编码 | plan / act 双模式、逐步确认的 UX |
| **OpenHands** | 平台 | 通用软件开发 Agent（含浏览器操作） | 容器化沙箱、多 Agent 协作 |
| **Aider** | CLI | git 原生的结对编程 | 极简 diff / patch 编辑策略，token 效率高 |
| **Gemini CLI** | CLI | Google 官方 | 超大上下文窗口的利用方式 |
| **DeepSeek Harness** | CLI | 2026-08 新出，**万物皆插件**（连 agent loop 本身都是插件） | 极致可插拔的架构设计 |
| **Pi** | CLI | 轻量：每个能力在上下文里只留一行描述，调用时才加载完整 schema | **渐进式工具加载**——上下文优化的前沿做法 |

> 星标数据（公开榜单口径，仅供参考量级）：DeepSeek Harness ~203k、OpenCode ~202k、Codex CLI ~119k、Pi ~98k、OpenHands ~85k、Cline ~67k。这类数字变化极快，别太当真。

**一个跨产品的共识**：MCP（Model Context Protocol）已经成为工具接入的事实标准——Claude Code、Cline、CrewAI、OpenCode 都原生支持。第 8 步会讲。

### C 类：可观测性与评估平台（配套设施）

严格说它们不是 harness，而是**插在 harness 旁边**的基础设施——但一旦你的 Agent 要上生产，这层几乎躲不掉。

| 工具 | 定位 | 备注 |
|---|---|---|
| **LangSmith** | LangChain 官方的 trace + eval 平台 | 与 LangGraph 集成最深，闭源 SaaS |
| **Langfuse** | 开源的 LLM 可观测性平台 | 框架无关，可自托管，适合不想绑 LangChain 的团队 |
| **AgentOps** | 面向 Agent 的会话回放与成本分析 | 侧重多 Agent 场景 |
| **Phoenix**（Arize） | 开源 trace + eval，OpenTelemetry 生态 | 标准化程度高 |

自己写 harness 的话，第 7 步会先手搓一个最小版本（JSONL trace + 成本统计 + 一个跑分脚本）——理解了原理再决定要不要上平台。

---

## 五、怎么选？

```
你要做什么？
│
├─ 步骤能提前写死 ───────────────► 别用 Agent 框架，写 workflow + 直接调 API
│
├─ 要一个能改代码 / 操作文件的 Agent
│   ├─ 想快速产出，接受厂商绑定 ──► Claude Agent SDK
│   └─ 想理解原理 / 要深度定制 ───► 手写 harness（本教程）+ 读 OpenCode 源码
│
├─ 要在业务系统里跑自定义工具 Agent
│   ├─ 流程复杂，要持久化 / 断点 ─► LangGraph
│   ├─ TypeScript 栈 ───────────► Mastra（重）/ Vercel AI SDK（轻）
│   ├─ Python 栈，看重类型安全 ──► Pydantic AI
│   └─ 只要一个简单循环 ────────► Tool Runner / OpenAI Agents SDK
│
└─ 不想管服务器和沙箱 ───────────► Managed Agents 这类托管方案
```

**给学习者的建议**：先手写，再用框架。框架帮你藏起来的东西，恰恰是出问题时你必须理解的东西。这门课的 8 步走完，你再回头看任何一个框架的文档，都会觉得"哦，它就是把我第 N 步那段代码封装了一下"。

---

## 六、这和 8 步路线图怎么对应

| 步骤 | 对应上面哪一块 |
|---|---|
| 1. 裸 API | §1 理解"模型只是个函数" |
| 2. Agent Loop | §2.1 循环 —— **最重要的一步** |
| 3. 工具集 | §2.2 工具 |
| 4. 权限 | §2.4 权限 |
| 5. 上下文 | §2.3 上下文 |
| 6. System Prompt | §2.5 提示与环境注入 |
| 7. 评估与可观测性 | §2.6 可观测性与评估 + §4 的 C 类 |
| 8. 进阶 + SDK 对照 | §3 区分二表格的第 2~4 行 |

---

## 参考来源

- [The best AI agent frameworks in 2026 — LangChain](https://www.langchain.com/resources/ai-agent-frameworks)
- [Comparing Open-Source AI Agent Frameworks — Langfuse](https://langfuse.com/blog/2025-03-19-ai-agent-comparison)
- [AI Agent Frameworks Compared: Which Ones Ship? — Chanl](https://www.channel.tel/blog/ai-agent-frameworks-compared-2026-what-ships)
- [awesome-cli-coding-agents — GitHub](https://github.com/bradAGI/awesome-cli-coding-agents)
- [Top Agent Harnesses: Claude Code vs Codex — AIMultiple](https://aimultiple.com/agent-harness)
- [Best Agent Harnesses 2026: A Builder's Field Guide — Future AGI](https://futureagi.com/blog/best-agent-harness/)
- [Best Open Source CLI Coding Agents in 2026 — Pinggy](https://pinggy.io/blog/best_open_source_cli_coding_agents/)
