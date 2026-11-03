# Tool Audit and Execution Reliability

## Role in the Agent System

This subsystem ensures that every external action is authorized, explainable, deduplicated, and recoverable. It bridges model-proposed Tool Calls and real-world side effects.

## 1. Tool execution state machine

`CREATED → VALIDATED → AUTHORIZED → APPROVAL_PENDING → APPROVED → EXECUTING → SUCCEEDED / FAILED / UNKNOWN`.

UNKNOWN means the caller cannot tell whether the downstream system committed the operation. It is not equivalent to FAILED.

## 2. Required execution metadata

Tool ID/version, execution ID, idempotency key, tenant/user, session/turn, resource reference, normalized arguments, policy decision, approval ID, start/end timestamps, downstream request ID, result reference, error code, and retry count.

## 3. Idempotency

Use a deterministic or persisted idempotency key derived from business command identity and request intent. The downstream domain service must enforce deduplication; a runtime-only cache is insufficient.

## 4. Timeout reconciliation

If a write request times out:
1. Mark execution UNKNOWN.
2. Query downstream command status by command/idempotency ID.
3. If committed, record success and verify resulting state.
4. If not committed, retry only if safe.
5. If status cannot be determined, pause for reconciliation or human action.

## 5. Read-after-write

After important mutations, query the resource again and compare expected state. Do not tell the user that a change succeeded merely because the tool request was sent.

## 6. Audit

Keep an immutable audit record of who requested, which Agent proposed, what action/target/parameters were approved, which policy version decided, who approved, what downstream service executed, and what changed.

## 7. Compensation

For multi-step operations, define a Saga plan with compensating actions and manual recovery paths. Compensation is a new audited action, not deletion of the original event.
