# 15. Model Provider Adapter

## Role in the Agent System

Abstracts model vendors and routes work by capability, quality, latency, cost, and policy.

**Primary responsibility:** Own this capability as a reusable subsystem with explicit contracts, state ownership, security boundaries, and observable outcomes.

**Boundary:** This subsystem must not silently take over responsibilities owned by adjacent layers. Its public interface should be testable without running a full Agent conversation.

## 1. 目标

让 Agent Runtime 不依赖单一 LLM 厂商。Provider Adapter 统一模型调用契约，同时保留各厂商的特有能力。

## 2. 统一内部请求

```typescript
interface ModelRequest {
  model: string;
  messages: NormalizedMessage[];
  tools?: NormalizedTool[];
  toolChoice?: "auto" | "none" | "required" | { name: string };
  responseFormat?: ResponseFormat;
  temperature?: number;
  maxOutputTokens?: number;
  metadata: {
    tenantId: string;
    sessionId: string;
    turnId: string;
    stepId: string;
    traceId: string;
  };
}
```

## 3. 统一响应

```typescript
interface ModelResponse {
  responseId?: string;
  finishReason: "stop" | "tool_calls" | "length" | "refusal" | "unknown";
  message?: NormalizedMessage;
  toolCalls: NormalizedToolCall[];
  usage?: TokenUsage;
  providerMetadata?: Record<string, unknown>;
}
```

## 4. Adapter 能力矩阵

不同厂商在 Tool Schema 格式、Tool Choice 语义、Strict JSON Schema 支持、Tool Call ID / 并行调用、Streaming Chunk 格式、Reasoning / Thinking 字段、Structured Output、Prompt Cache、Provider-side Tools、Response / Conversation Continuation、Usage 统计字段、超时和错误码方面可能不同。

Adapter 必须提供 Capability Descriptor，不能假设所有模型支持相同功能。

## 5. Tool Calling 往返

1. Runtime 发送工具 Schema。
2. Provider 返回 Tool Call。
3. Runtime 校验并执行。
4. Runtime 将 Tool Result 作为后续消息回传。
5. 再次调用 Provider。
6. 直到最终回答或停止。

Provider 托管工具（如厂商 Web Search）与 Client-side Tool 要分开建模。企业业务工具通常是 Client-side Tool。

## 6. Stateless 与托管状态

默认按 Stateless 设计：Runtime 每次提供完整所需消息和工具结果。若 Provider 支持 `previous_response_id` 或 Conversation ID，可作为优化，但必须考虑数据留存与隐私、状态过期、Provider 切换、工具执行结果一致性、供应商上下文与本地 Event Log 对账。

本地 Session/Event Log 仍然是企业应用的权威记录。

## 7. Model Routing

路由条件：任务类型、复杂度、是否需要 Tool Calling、Structured Output 能力、上下文窗口、延迟、成本、数据区域/合规、模型可用性。

不要只根据模型名称判断能力。使用模型能力注册表和健康状态。

## 8. Retry / Fallback

- 仅对可重试错误重试。
- Tool Call 已执行后，不能因为模型请求超时而无条件重放写操作。
- Provider fallback 需要保持消息和 Tool Call 兼容。
- 对 reasoning / multimodal 内容不支持的模型要做明确降级。
- 记录原 Provider、Fallback Provider 和原因。

## 9. 建议实现接口

```typescript
interface ModelProviderAdapter {
  capabilities(): ProviderCapabilities;
  invoke(request: ModelRequest): Promise<ModelResponse>;
  stream(request: ModelRequest): AsyncIterable<ModelStreamEvent>;
  normalizeError(error: unknown): ModelError;
}
```

## 10. 适配策略

OpenAI、Claude、Qwen、DeepSeek 的 API 格式差异全部在 Adapter 内处理。业务 Agent 不直接判断 `tool_calls`、`tool_use`、`function_call_output` 等厂商专属结构。
