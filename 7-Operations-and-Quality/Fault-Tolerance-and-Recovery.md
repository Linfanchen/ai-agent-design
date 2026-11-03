# Fault Tolerance and Recovery

## Role in the Agent System

This subsystem keeps long-running execution correct when models, APIs, queues, workers, databases, or clients fail. It defines retry, timeout, backoff, checkpoint, graceful shutdown, idempotency, and reconciliation behavior.

## 1. Failure classes

Transient network errors, rate limits, provider outage, malformed model response, invalid tool arguments, business rule rejection, authorization denial, tool timeout with unknown outcome, worker crash, checkpoint failure, queue redelivery, and client disconnect.

## 2. Retry policy

Retry only transient failures. Use bounded attempts, exponential backoff with jitter, retry-after support, and per-tenant/provider concurrency controls. Never retry a business rule rejection as if it were a network error.

## 3. Timeout layers

Set separate timeouts for model call, tool call, graph node, total run, approval wait, queue lease, and downstream business API. A node timeout does not prove a downstream write did not happen.

## 4. Checkpoint and resume

Persist graph state at durable boundaries. Use stable thread ID. Ensure tool execution result is persisted before the graph advances. Reconcile external side effects after uncertain failure.

## 5. Graceful shutdown

Stop accepting new work, stop claiming jobs, allow safe nodes to finish, checkpoint running graphs, release leases, close provider and DB clients, and expose readiness/health state.

## 6. Fault injection tests

Kill worker after external write but before result persistence; disconnect SSE; return provider 429; timeout tool after commit; duplicate queue message; expire approval during pause; deploy a graph schema change with old checkpoints.

## 7. Recovery policy

For each failure, decide automatic retry, fallback, compensation, resume, human intervention, or terminal failure. Record the decision and reason.
