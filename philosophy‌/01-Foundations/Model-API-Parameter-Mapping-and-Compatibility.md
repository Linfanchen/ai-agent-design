# 主流模型 API 参数映射与兼容性

## 本文档在系统中的作用

将 OpenAI、Anthropic、Qwen、DeepSeek 等模型 API 的差异映射到内部统一请求模型，避免业务 Agent 直接拼接厂商 JSON。重点不是声称各家完全兼容，而是定义哪些语义可统一、哪些参数必须保留 Provider Extension。

## 1. 四类参数

1. 通用语义：messages / input、max output、temperature、top_p、stop。
2. 工具参数：tools、tool_choice、parallel tool calls。
3. 结构化输出：JSON Mode、JSON Schema。
4. 供应商扩展：reasoning effort、thinking budget、prompt cache、metadata、service tier 等。

统一抽象不等于每家字段完全一致。内部请求模型应保留受白名单 Schema 验证的 `provider_options`。

## 2. 参数映射矩阵

| 内部语义 | OpenAI API | Anthropic Messages | Qwen OpenAI-Compatible | DeepSeek Chat |
|---|---|---|---|---|
| 消息 | messages / Responses input | messages + 顶层 system | messages | messages |
| 最大输出 | 按 API Mode 使用对应字段 | max_tokens | 常见 max_tokens | max_tokens |
| 温度 | temperature | temperature | temperature | temperature |
| Top-P | top_p | top_p | top_p | top_p |
| 工具 | tools | tools | 兼容能力依模型 | tools |
| 工具选择 | tool_choice | tool_choice | 依兼容实现 | tool_choice |
| 结构化输出 | JSON Schema / Format | 按版本和能力支持 | 依模型能力 | 依接口版本 |
| 停止序列 | stop | stop_sequences | stop | stop |
| 推理参数 | reasoning 相关字段 | thinking 配置 | 模型特有字段 | 模型特有字段 |
| 缓存 | Prompt Cache 能力 | Cache Control | 依服务能力 | 依服务能力 |

此表是适配设计，不代表所有模型支持全部字段。每个 Model Profile 必须有 Capability Descriptor。

## 3. Anthropic 特殊点

`system` 与 `messages` 结构不同于 OpenAI Chat Completions；Tool Use / Tool Result 使用不同内容块；Thinking Block 有独立语义；Prompt Cache 有自己的控制方式；`stop_reason` 不能简单映射为 OpenAI finish_reason；多模态内容块需要独立转换。

Adapter 必须保留 Thinking / Tool Use / Tool Result 的正确关联，不能把所有内容块压成文本。

## 4. OpenAI API Mode

OpenAI 平台存在不同 API 形态，例如 Chat Completions 与 Responses。不能只根据 Provider 名称假设请求字段、流事件、Tool Call 和 Usage 完全相同。Adapter 必须显式声明 `api_mode`。

## 5. Qwen / DeepSeek 兼容层

OpenAI-Compatible 代表请求形态在一定程度上兼容，不代表高级参数、结构化输出、推理模式、缓存和流式行为完全相同。逐模型维护支持矩阵；不支持参数明确拒绝或降级；不静默丢字段；流式 Tool Call 增量片段要完整组装。

## 6. 参数校验

- `temperature` 与某些推理模式可能冲突；
- `max_output_tokens` 不得超过模型上限；
- `tools` 与 `tool_choice` 组合必须符合厂商约束；
- `response_schema` 必须先检查能力；
- `stop_sequences` 在不同 API 中可能限制不同；
- `provider_options` 不接受任意用户输入。

## 7. Streaming 归一化

统一内部事件：`message.started`、`content.delta`、`tool_call.started`、`tool_call.arguments.delta`、`tool_call.completed`、`usage.updated`、`message.completed`、`message.failed`。Adapter 负责 Chunk 拼接、工具参数 JSON 增量和 Finish Reason。

## 8. 官方入口

- OpenAI：<https://developers.openai.com/api/docs>
- Anthropic：<https://platform.claude.com/docs/en/api/overview>
- Qwen：<https://docs.bailian.console.aliyun.com/zh/model-studio/qwen-api-via-openai-chat-completions>
- DeepSeek：<https://api-docs.deepseek.com/zh-cn/api/create-chat-completion>
