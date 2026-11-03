# Middleware, Hooks, and Policies

## Role in the Agent System

Middleware wraps model calls, tool execution, or lifecycle boundaries to apply cross-cutting behavior consistently. It keeps the core Agent Loop small while adding sanitization, policy enforcement, memory injection, context compression, metrics, clarification, and safety controls.

## 1. Middleware vs Hook vs Policy

- Middleware: composable wrapper around a defined lifecycle.
- Hook: a named event point where extensions observe or modify behavior.
- Policy: an authoritative rule that returns allow, deny, require approval, or constrained allow.
- Graph Node: explicit business step in the control flow.

A policy must not be merely a prompt instruction. It is executable server-side logic.

## 2. Lifecycle stages

Recommended stages:
1. `before_turn`: identity, tenant, rate limit, input validation.
2. `before_context`: memory retrieval, context policy, data classification.
3. `before_model`: prompt assembly, model routing, tool schema selection.
4. `after_model`: refusal checks, output parsing, loop detection.
5. `before_tool`: schema validation, authorization, approval, idempotency.
6. `after_tool`: result sanitization, audit, persistence, progress events.
7. `before_checkpoint`: state validation and redaction.
8. `after_turn`: final output validation, async memory extraction, metrics.
9. `on_error`: normalized failure handling.
10. `on_cancel` / `on_dispose`: cleanup.

## 3. Middleware ordering

Ordering is semantic. For example, authorization must run before a write tool executes; output sanitization must run before tool results enter the model context; durable task/delegation state must be captured before transcript compaction; clarification should occur after enough context and policy evaluation to ask a meaningful question.

Define:
- required ordering constraints;
- before/after anchors;
- whether a middleware may veto;
- whether it can mutate state;
- failure policy (fail-open vs fail-closed);
- scope (Lead Agent, subagent, tool, tenant, channel).

## 4. Middleware contract

```typescript
interface AgentMiddleware {
  id: string;
  version: string;
  scope: "turn" | "model" | "tool" | "runtime";
  before?(ctx: MiddlewareContext): Promise<MiddlewareDecision | void>;
  after?(ctx: MiddlewareContext, result: unknown): Promise<unknown>;
  onError?(ctx: MiddlewareContext, error: AgentError): Promise<ErrorDecision>;
  dispose?(): Promise<void>;
}
```

## 5. Safety and reliability

- Authorization middleware must fail closed.
- Telemetry middleware should not change business outcomes.
- Optional title-generation or analytics middleware may fail open.
- Middleware must be idempotent when a graph node is retried.
- Avoid hidden model calls inside multiple middleware layers without a cost budget.
- Prevent recursive hook invocation.
- Record middleware chain and versions per run.

## 6. AgentScope / DeerFlow lessons

AgentScope highlights hooks across reply, reasoning, acting, model calling, permission, context compression, and system prompt. DeerFlow shows a long, ordered middleware chain for sanitization, memory, skill activation, durable context, summarization, todo, token usage, deferred tools, subagent limits, loop detection, and clarification. The design lesson is to make order explicit and independently testable.
