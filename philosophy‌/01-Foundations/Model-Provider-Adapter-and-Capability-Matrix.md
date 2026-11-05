# Model Provider Adapter 与能力矩阵

## 本文档在系统中的作用

Provider Adapter 将各模型厂商的请求、响应、流式事件、Tool Call、Usage 和错误映射到内部统一协议。它避免业务 Graph 与某一家厂商 SDK 耦合，同时保留厂商独有能力。

## 1. 内部统一请求

```python
from typing import Any, Literal
from pydantic import BaseModel, ConfigDict, Field

class ToolSpec(BaseModel):
    name: str
    description: str
    input_schema: dict[str, Any]

class ModelRequest(BaseModel):
    model_config = ConfigDict(extra="forbid")
    model_id: str
    messages: list[dict[str, Any]]
    max_output_tokens: int = Field(gt=0)
    temperature: float | None = Field(default=None, ge=0, le=2)
    top_p: float | None = Field(default=None, gt=0, le=1)
    tools: list[ToolSpec] = Field(default_factory=list)
    tool_choice: Literal["auto", "none", "required"] = "auto"
    response_schema: dict[str, Any] | None = None
    reasoning_effort: Literal["low", "medium", "high"] | None = None
    provider_options: dict[str, Any] = Field(default_factory=dict)
```

## 2. 统一响应对象

- `message_id`
- `content_blocks`
- `tool_calls`
- `finish_reason`
- `usage`
- `provider_request_id`
- `raw_metadata_ref`

Content Block 至少区分 text、image、audio、reasoning、tool_use、tool_result、json。不能把所有内容块压成字符串，否则无法正确处理多模态和工具链。

## 3. Provider Port

```python
from typing import Protocol, AsyncIterator

class ModelProvider(Protocol):
    async def complete(self, request: ModelRequest) -> "ModelResponse": ...
    def stream(self, request: ModelRequest) -> AsyncIterator["ModelStreamEvent"]: ...
    async def health_check(self) -> "ProviderHealth": ...
```

Provider Adapter 负责内部请求到厂商请求映射、厂商响应归一化、Stream Chunk 重组、Tool Call 参数片段聚合、Usage 归一化、错误分类、超时和连接池。

## 4. Capability Descriptor

每个 Model Profile 必须声明 `context_window`、`max_output_tokens`、`supports_tools`、`supports_parallel_tools`、`supports_json_schema`、`supports_streaming`、`supports_vision`、`supports_audio`、`supports_reasoning`、`supports_prompt_cache`、`supported_parameters` 和参数冲突规则。

能力矩阵按具体模型和 API Mode 维护。OpenAI Responses 与 Chat Completions 不应假设字段完全相同；OpenAI-Compatible 也不意味着 Qwen、DeepSeek 对所有参数和流式事件行为完全一致。

## 5. 参数兼容规则

- 不支持的参数默认拒绝并返回 `UNSUPPORTED_MODEL_PARAMETER`，不要静默丢弃。
- 温度与推理模式冲突由 Capability Validator 在请求前检查。
- JSON Schema 先检查模型能力。
- Tool Choice 映射各家语义，无法映射时明确报错。
- Prompt Cache 是优化，不是 Memory。
- Provider Conversation ID 是 Adapter 可选引用，不是业务 Thread 的权威 ID。

## 6. 错误分类

`RATE_LIMITED`、`AUTHENTICATION_FAILED`、`INVALID_REQUEST`、`MODEL_NOT_FOUND`、`CONTEXT_TOO_LARGE`、`PROVIDER_TIMEOUT`、`PROVIDER_UNAVAILABLE`、`CONTENT_FILTERED`、`STREAM_INTERRUPTED`、`UNKNOWN_PROVIDER_ERROR`。

每个错误携带 retryable、retry_after、provider_code、safe_message 和 correlation_id。429 遵守 Retry-After；参数错误不重试；超时重试要考虑请求幂等。

## 7. 流式事件规范

`message.started`、`content.delta`、`reasoning.delta`（受权限与产品策略控制，不默认暴露）、`tool_call.started`、`tool_call.arguments.delta`、`tool_call.completed`、`usage.updated`、`message.completed`、`message.failed`。

事件必须有 run_id、sequence、timestamp、provider_request_id。UI 展示事件与模型内部推理内容明确隔离。

## 8. 测试矩阵

每个 Provider Adapter 测试普通文本、多轮消息、Tool Call、并行 Tool Call、流式增量、JSON Schema、Usage、429、超时、取消、非法参数和多模态（若声明支持）。使用录制响应时清理敏感信息。
