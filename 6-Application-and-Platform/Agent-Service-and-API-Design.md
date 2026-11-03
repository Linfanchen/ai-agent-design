# Agent Service and API Design

## Role in the Agent System

Agent Service exposes the Harness as a stable network service. It provides invocation, streaming, thread/run inspection, resume, cancellation, approval, artifact retrieval, and administrative endpoints.

## 1. API groups

- Agent Catalog: list/get/publish Agent definitions.
- Invocation: create run, stream run, get result.
- Thread: create, inspect, list messages, compact, archive.
- Run: inspect status, resume, cancel, replay in safe mode.
- Approval: list pending, approve, reject, expire.
- Tool: inspect execution and audit.
- Artifact: list, preview, download.
- Admin: configuration, plugin/skill management, health, usage.
- Webhook: asynchronous completion and channel delivery.

## 2. API principles

Use explicit resource IDs, idempotency keys for create operations, pagination, API versioning, typed errors, and tenant scoping. Avoid one giant endpoint that accepts arbitrary graph state from the client.

## 3. Streaming protocol

Return typed event envelopes with event ID, run ID, sequence, event type, and payload. Provide reconnect cursor support. Keep the authoritative run status retrievable through REST.

## 4. Long-running invocation

The initial API may return a run ID immediately. The client then subscribes to events. A disconnected browser must not cancel a durable job unless the user explicitly requests cancellation.

## 5. API error model

Normalize authentication, authorization, validation, rate limit, provider failure, tool failure, approval required, conflict, timeout unknown, and internal errors.

## 6. API security

Resolve identity from authenticated credentials; never trust tenant/user fields supplied in a JSON body. Check resource ownership on every operation. Use webhook signatures and short-lived artifact URLs.
