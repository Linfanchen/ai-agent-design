# Deployment and Scaling

## Role in the Agent System

Deployment and Scaling turn the runtime into a reliable service. They define process topology, worker roles, stateful storage, autoscaling, health checks, rollout, and operational isolation.

## 1. Deployment units

- API/Gateway: authentication, invocation, streaming.
- Runtime workers: graph execution and model/tool orchestration.
- Background workers: memory consolidation, indexing, long jobs, scheduled wakeups.
- Sandbox workers: isolated code/browser execution.
- Storage services: PostgreSQL, Redis, object storage, search/vector index.
- Admin/Studio UI: configuration, evaluation, trace inspection.

These can begin in one or two deployables and split as scaling or security requires.

## 2. Horizontal scaling

Keep API instances stateless. Persist thread state and job leases. Route resume requests to any worker capable of loading the same checkpoint. Use queue-based work distribution for long tasks and separate pools for expensive sandboxes.

## 3. Health checks

Liveness checks process health. Readiness checks DB, queue, provider configuration, and required dependencies. Dependency degradation should be reported distinctly from process failure.

## 4. Release strategy

Use immutable images, environment-specific config, database migration gates, canary rollout, backward-compatible state schemas, Agent Definition version pinning, and rollback plans.

## 5. Scaling dimensions

Concurrent turns, model concurrency, tool throughput, sandbox CPU/memory, event fanout, memory recall QPS, vector retrieval, and artifact storage. Scale each bottleneck independently.

## 6. Operational limits

Per-tenant concurrency, max run duration, max tool calls, token/cost budgets, sandbox quota, queue depth, and retention limits.
