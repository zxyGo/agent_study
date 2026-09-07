# Agent Harness 入门教程

从零手写一个 Agent Harness，理解 Claude Code 这类编码 Agent 到底是怎么跑起来的。

**技术栈**：TypeScript / Node.js（Node 22+）
**模型**：Claude（`claude-opus-5`），通过 `@anthropic-ai/sdk` 直接调 Messages API

## 为什么手写

模型本身只会输入 messages、输出 text 或 `tool_use`。让它变成一个能自己干活的 Agent，靠的是外面那层循环——执行工具、把结果喂回去、管上下文、卡权限。这层壳就是 **harness**。

现成框架把这层藏起来了。藏起来的东西，恰恰是出问题时你必须理解的东西。所以这门课先手写一遍，最后一步再和官方 SDK 做对照。

## 学习路线

> 📖 **随手查**：[核心术语表](docs/00-glossary.md) —— 全程会用到的词，按主题分组，标注了各自在哪一步出现。不用一次读完。

| 步骤 | 目录 | 学什么 | 产出 |
|---|---|---|---|
| **0** | [docs/01-agent-and-harness.md](docs/01-agent-and-harness.md) | Agent 与 Harness 概念、六大组成部分、主流框架生态与选型 | 心智模型 ✅ |
| **1** | `01-raw-api/` | Messages API 基础：messages 数组、system、stream、stop_reason | 能对话的 CLI，无工具 |
| **2** | `02-agent-loop/` | **harness 的心脏**：`while` 循环 + `tool_use` / `tool_result` 配对 | 带 1 个 `read_file` 工具的最小 Agent |
| **3** | `03-tools/` | 工具集与 schema 设计：bash / write / edit / grep / glob；错误回传而非崩溃 | 能真正改代码的 Agent |
| **4** | `04-permissions/` | 安全边界：路径沙箱、危险命令拦截、人工确认 | 敢放开跑的 Agent |
| **5** | `05-context/` | token 计数、prompt caching、历史压缩、长输出截断 | 能跑长任务不爆上下文 |
| **6** | `06-system-prompt/` | system prompt 分层、环境信息注入、todo / 计划机制 | 行为可控、可调优 |
| **7** | `07-eval/` | **评估与可观测性**：trace 记录、成本统计、跑一套 eval 集、A/B 对比两版 prompt | 改动能被量化，不再靠感觉调 |
| **8** | `08-advanced/` | 子代理、多 Agent、MCP、hooks、Skill；再用官方 Claude Agent SDK 重写对照 | 理解 Claude Code 的完整形态 |

关键分水岭在**第 2 步**——只要写通了 tool_use 循环，后面全是往这个循环上挂能力，难度是平的。

第 7 步容易被跳过，但它决定你后面所有调优是不是在瞎猜：**没有 eval，你改完 prompt 只能凭感觉说"好像好点了"**。

## 环境准备

```bash
node -v          # 需要 >= 20，推荐 22
export ANTHROPIC_API_KEY=sk-ant-...
```

## 进度

- [x] 第 0 课 概念与生态
- [x] 核心术语表
- [ ] 第 1 步 裸 API 调用
- [ ] 第 2 步 Agent Loop
- [ ] 第 3 步 工具集
- [ ] 第 4 步 权限
- [ ] 第 5 步 上下文
- [ ] 第 6 步 System Prompt
- [ ] 第 7 步 评估与可观测性
- [ ] 第 8 步 进阶
