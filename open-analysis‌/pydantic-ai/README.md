# Pydantic AI 源码架构拆解（首轮）

> 范围：当前 `open/pydantic-ai` checkout；仓库有 3,641 个 Markdown/MDX。已看根 README 的产品分层，并检索核心包 `pydantic_ai_slim/pydantic_ai` 的 Agent、run、graph、model、tool 入口。没有逐篇读完全部文档；仓库也包含独立的 evals、graph、harness 等包，不能把单一 Agent 包等同于整个生态。

## 定位与设计主张

Pydantic AI 将自己定位为 Python AI SDK：Agent loop、模型适配、依赖注入、工具、结构化输出和运行事件尽量保持类型化；同一 Agent 可以通过 Python `run()`、Web/CLI/语音、后台队列等不同接口运行。README 进一步把 Harness 能力描述为可插拔能力集合（memory、guardrails、subagents、planning、context compaction、persistence、coder），而 Pydantic Graph 与 Evals 是可组合但独立的包。

## 结构理解

- `Agent` 是用户配置入口：绑定模型/模型选择策略、deps 类型、输出类型、系统提示、工具和能力配置。
- 一次执行由 `AgentRun` 表达，内部以节点和事件驱动，不是简单的一次模型 HTTP 调用。
- `_agent_graph.py` 体现核心执行状态机：模型请求节点、工具执行节点、结果验证/重试、结束节点等围绕消息历史推进。
- `RunContext` 将 deps、当前 agent、消息历史、使用量、工具/模型调用上下文注入运行步骤；依赖数据不需要塞进全局单例。
- `messages.py` 定义规范化请求/响应消息与 part，使不同 Provider 的差异映射到统一消息结构。
- 工具注册使用可检查的签名/schema；工具执行和模型请求可异步，Agent 通过事件流暴露中间进展。
- Graph/Harness/Evals 等可在 SDK Agent 之上继续组合，提供显式控制流、持久化与评估能力。

## 一次运行的执行图

`Agent.run(input, deps, message_history) → AgentRun → user prompt/message history → ModelRequestNode → ModelResponse`。若响应包含工具调用，状态转入工具执行节点，执行对应 Python 工具并追加 `ToolReturnPart`，然后再次请求模型；若响应满足输出规则，则校验结构化结果并结束；校验或模型错误可能触发重试策略，超出预算后返回错误。流式执行使用相同运行状态机，只是逐步产出 AgentRunEvent/部分结果。

这里的关键设计是：模型请求、工具执行、输出验证成为显式节点；消息历史是过程载体，运行上下文携带 deps 和策略，结果对象保留最终输出、使用量和新消息。这样便于类型校验、流式 UI、评估轨迹和持久化衔接。

## 关键源码

| 路径 | 用途 |
|---|---|
| `pydantic_ai_slim/pydantic_ai/agent/__init__.py` | `Agent` 用户 API、配置与 run/stream 入口。 |
| `pydantic_ai_slim/pydantic_ai/agent/abstract.py` | Agent 抽象与 run event 类型。 |
| `pydantic_ai_slim/pydantic_ai/_agent_graph.py` | Agent 状态图/节点执行逻辑，是理解 loop、工具回路和结束条件的核心。 |
| `pydantic_ai_slim/pydantic_ai/run.py` | `AgentRun`、结果对象及流式事件处理。 |
| `pydantic_ai_slim/pydantic_ai/_run_context.py` | deps 与运行期上下文。 |
| `pydantic_ai_slim/pydantic_ai/messages.py` | Provider-neutral 的消息/part/request/response 表达。 |
| `pydantic_ai_slim/pydantic_ai/tools.py`、`_tool_execution.py` | 工具声明、schema 和调用路径。 |
| `pydantic_ai_slim/pydantic_ai/models/` | 模型协议与提供商实现。 |
| `pydantic_graph/` | 独立的类型化图运行时，可用于更复杂的确定性控制流。 |
| `pydantic_evals/` | 数据集/评估器/评估运行能力，独立于 Agent 内核。 |

## 与 LangGraph 的差异（初步）

Pydantic AI 的默认入口更偏“类型化单 Agent loop + deps + tools + output”；LangGraph 的默认入口更偏“用户声明状态和图，由 Pregel runtime 调度”。Pydantic AI 把常见 Agent 使用路径做得直接，复杂图控制流再交给 Graph；LangGraph 从一开始就把流程状态与调度显式化。两者都使用图/节点思想，但抽象层级和默认心智模型不同。

## 后续需要读透

需继续追读 `_agent_graph.py` 中各节点转移、重试和工具结果处理；确认结构化输出如何进入 tool/native output 模式；检查 durable execution 与 conversation store 是否保证恢复语义；梳理 Harness 包与基础 SDK 的边界；读取 Threat Model、README、架构/概念文档主题并核对实现。

## 覆盖账本

- 已看：根 README、`THREAT_MODEL.md` 仅需后续深入，目前只扫到存在；核心包路径和核心类定义索引。
- 未完成：3,641 个 Markdown/MDX 的逐篇覆盖，尤其 `docs/`、`agent_docs/`、翻译与版本化材料；本文为源码地图和首轮架构解释，不是完整审计。
