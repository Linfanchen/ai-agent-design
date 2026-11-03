# Channels and Gateway

## Role in the Agent System

Channels and Gateway connect users and external systems to the Agent Runtime. They normalize different entry points—web chat, mobile, IM, API, webhook, cron, voice—into a consistent invocation contract.

## 1. Gateway responsibilities

Authentication, tenant resolution, request validation, rate limiting, routing, streaming transport, idempotent request creation, cancellation, webhook validation, and API versioning.

## 2. Channel adapter responsibilities

Translate platform messages and identities into internal events; handle thread mapping, attachments, message limits, delivery receipts, retries, and platform-specific formatting.

## 3. Normalized invocation

```typescript
interface AgentInvocation {
  tenantId: string;
  userId: string;
  agentId: string;
  conversationId: string;
  threadId: string;
  input: NormalizedContentBlock[];
  channel: string;
  requestId: string;
  metadata: Record<string, unknown>;
}
```

## 4. Message routing

Map external conversation/user IDs to internal tenant/user/conversation/thread identifiers. Never use a platform username as the sole identity key.

## 5. Delivery semantics

Support at-least-once inbound delivery with deduplication, retryable outbound delivery, and explicit final status. If an IM platform has message length limits, split output while preserving turn correlation.

## 6. Security

Verify webhook signatures, validate platform tokens, map platform users to tenant identities, restrict channel-specific Agent access, and redact sensitive content in logs.

## 7. Recommended architecture

Keep Gateway stateless where possible. Use durable job/queue for long tasks. The Gateway should not own graph state or business workflow logic.
