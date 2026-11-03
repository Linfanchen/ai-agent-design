# Long-Running Tasks and Scheduling

## Role in the Agent System

This subsystem allows an Agent to continue work beyond a single HTTP request or interactive turn. It owns durable job state, background execution, scheduled wakeups, cancellation, timeout, retry, and result delivery.

## 1. Use cases

Deep research, repository analysis, large document processing, batch operations, report generation, scheduled monitoring, delayed follow-up, and external jobs whose completion time is unpredictable.

## 2. Job model

```typescript
interface AgentJob {
  jobId: string;
  tenantId: string;
  agentId: string;
  threadId: string;
  parentRunId?: string;
  jobType: string;
  status: "queued" | "running" | "waiting" | "succeeded" | "failed" | "cancelled" | "expired";
  scheduledAt?: string;
  deadlineAt?: string;
  heartbeatAt?: string;
  attempt: number;
  idempotencyKey: string;
  inputRef: string;
  resultRef?: string;
}
```

## 3. Execution architecture

API accepts request → create durable job → enqueue job ID → worker claims with lease → resume graph → emit progress → checkpoint → persist result/artifact → notify channel → mark terminal status.

Never rely on an in-memory promise or process-local timer for durable scheduling.

## 4. Scheduling

Support one-off delayed jobs, cron-like recurring jobs, event-triggered wakeups, and condition-based triggers. Every schedule needs timezone, owner, tenant, enabled state, misfire policy, max concurrency, and audit.

## 5. Reliability

Use leases, heartbeats, retry budgets, dead-letter queue, idempotent job creation, deduplicated event delivery, and cancellation tokens. On worker restart, expired leases can be reclaimed. A job must not be claimed by two workers simultaneously unless the work is explicitly parallelized.

## 6. Agent wakeup

A completed background task should resume or notify the parent execution using a durable correlation ID. Do not depend on a worker retaining the original HTTP request context.

## 7. Acceptance

- Restart worker during a run and resume from checkpoint.
- Duplicate queue delivery does not duplicate side effects.
- Cancellation stops future tool execution.
- A timed-out external tool is reconciled before retry.
- Scheduled runs respect tenant quotas and permissions at execution time.
