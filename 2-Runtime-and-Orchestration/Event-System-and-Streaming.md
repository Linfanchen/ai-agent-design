# Event System and Streaming

## Role in the Agent System

The Event System communicates execution progress to the UI, external clients, background workers, and audit services. It separates durable facts from transient progress and provides a stable protocol for streaming text, tool calls, state updates, approvals, and artifacts.

## 1. Event categories

- Durable domain events: user message, tool execution completed, approval resolved, turn completed.
- Runtime events: node started, node completed, model request started, subagent progress.
- Stream deltas: text token, reasoning projection where permitted, audio chunk, image/video frame.
- UI events: progress card, todo update, artifact ready, clarification requested.
- Telemetry events: latency, token usage, cost, retry, error.

Do not persist every token delta as a database event. Persist message boundaries and final content; stream deltas over a transport channel.

## 2. Event envelope

```typescript
interface AgentEvent<T = unknown> {
  eventId: string;
  type: string;
  version: number;
  tenantId: string;
  conversationId: string;
  threadId: string;
  turnId: string;
  runId: string;
  sequence: number;
  occurredAt: string;
  correlationId: string;
  causationId?: string;
  payload: T;
}
```

## 3. Suggested event types

`turn.started`, `turn.progress`, `model.started`, `model.delta`, `model.completed`, `tool.discovered`, `tool.started`, `tool.progress`, `tool.completed`, `tool.failed`, `subagent.started`, `subagent.progress`, `subagent.completed`, `approval.required`, `approval.resolved`, `clarification.required`, `todo.updated`, `artifact.created`, `checkpoint.saved`, `turn.completed`, `turn.failed`, `turn.cancelled`.

## 4. Transport

- SSE: straightforward server-to-client streaming.
- WebSocket: bidirectional control, voice, or live steering.
- Webhook: asynchronous result notification.
- Queue: durable background task delivery.
- REST: create, inspect, resume, cancel, approve, and fetch artifact operations.

## 5. Reliability

Clients should handle reconnect, sequence gaps, duplicate events, out-of-order delivery, stale run IDs, and reconnect from `Last-Event-ID` or an application cursor. Event IDs must be idempotent. The server must expose a way to fetch the authoritative current run state.

## 6. Security

Never stream hidden system prompts, credentials, private chain-of-thought, or unauthorized tool data. Stream only approved user-facing progress projections. Treat event payloads as tenant-scoped data.

## 7. UI contract

Keep separate event channels or typed event discriminators for:
- text delta → chat bubble;
- structured draft → editable form/card;
- tool progress → progress panel;
- approval → approval card;
- artifact → file preview/download;
- final message → completed response.

The UI should not infer business state by parsing free-form text.
