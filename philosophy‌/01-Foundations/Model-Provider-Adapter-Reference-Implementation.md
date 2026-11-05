# Model Provider Adapter：统一接口与 Python 参考实现

## 本文档在系统中的作用

定义模型供应商与 Agent Runtime 之间的稳定边界。业务 Graph 不直接依赖 OpenAI、Anthropic、Qwen 或 DeepSeek SDK；所有差异由 Adapter 处理。这样可以替换模型、运行 A/B、记录成本，并在模型能力不足时做显式降级。

## 1. 内部统一对象

内部请求建议包含：
- `model_profile_ref`：固定模型配置版本；
- `messages`：标准化消息列表；
- `tools`：本轮允许使用的工具 Schema；
- `tool_choice`：auto / none / required / named；
- `output_contract`：JSON Schema 或文本；
- `generation`：max output、temperature、top_p；
- `deadline`：绝对截止时间；
- `trace_context`：Trace / Run / Step 关联；
- `provider_options`：受白名单约束的厂商扩展。

内部响应建议包含：
- `finish_kind`；
- `content_blocks`；
- `tool_calls`；
- `usage`；
- `provider_request_id`；
- `model_id`；
- `latency_ms`；
- `raw_finish_reason`；
- `warnings`。

## 2. Protocol

```python
from typing import Protocol, AsyncIterator
from dataclasses import dataclass, field

@dataclass(frozen=True)
class ModelRequest:
    model_id: str
    messages: list[dict]
    tools: list[dict] = field(default_factory=list)
    tool_choice: str = "auto"
    max_output_tokens: int = 2048
    temperature: float | None = None
    top_p: float | None = None
    output_schema: dict | None = None
    provider_options: dict = field(default_factory=dict)
    request_id: str | None = None

@dataclass(frozen=True)
class ModelToolCall:
    call_id: str
    name: str
    arguments_json: str

@dataclass(frozen=True)
class ModelResponse:
    finish_kind: str
    content: str | None
    tool_calls: tuple[ModelToolCall, ...]
    input_tokens: int | None
    output_tokens: int | None
    provider_request_id: str | None
    model_id: str

class ModelAdapter(Protocol):
    async def complete(self, request: ModelRequest) -> ModelResponse: ...
    def stream(self, request: ModelRequest) -> AsyncIterator[dict]: ...
```

生产实现应使用强类型 ContentBlock，而不是把所有多模态内容压成 `str`。

## 3. Adapter 责任

Adapter 负责：
1. 内部消息到 Provider 消息结构的转换；
2. Tool Schema / Tool Choice 转换；
3. Provider 能力校验；
4. 请求超时和连接复用；
5. 流式 Chunk 归一化；
6. Usage 和 Finish Reason 归一化；
7. Provider 错误分类；
8. Request ID 和模型版本记录；
9. 重试策略中可重试错误的标记。

Adapter 不负责：
- Tool 是否允许执行；
- Tenant 授权；
- Agent Loop 决策；
- Memory 提取；
- 业务状态写入；
- 审批。

## 4. 能力描述符

```python
from pydantic import BaseModel

class ModelCapabilities(BaseModel):
    supports_tools: bool
    supports_parallel_tool_calls: bool
    supports_json_schema: bool
    supports_streaming: bool
    supports_vision: bool
    supports_audio: bool
    supports_reasoning: bool
    max_context_tokens: int
    max_output_tokens: int
    supported_modalities: list[str]
```

Capability 必须按 Model ID / API Mode 维护，而不是只按厂商维护。

## 5. Provider 错误分类

| 错误 | 默认处理 |
|---|---|
| Authentication | 不重试，告警 |
| Invalid Request | 不重试，修复参数 |
| Rate Limit | 遵循 Retry-After + 并发限流 |
| Temporary 5xx | 有限指数退避 |
| Timeout before send | 可安全重试 |
| Timeout after send | 需结合 Provider 语义判断 |
| Stream Interrupted | 记录已收到 Delta，按策略恢复 |
| Context Too Large | 触发 Context Recompile，不盲目重试 |

## 6. 多模型路由

Router 可依据任务类别、结构化输出要求、上下文长度、模态、延迟、成本和可用性选择 Model Profile。路由策略必须可解释，并记录实际选择原因。高风险动作不能因为切换到更强模型就获得额外权限。

## 7. 兼容性测试

每个 Adapter 维护 Provider Contract Suite：普通文本、Tool Call、并行 Tool Call、结构化输出、流式 Delta、Usage、超时、429、5xx、无效 Schema 和多模态。升级 SDK 后必须重新运行。
