# Worker、Queue、Scheduler 与 Lease

## 本文档在系统中的作用

Worker 承担长任务执行；Queue 管理待执行任务；Scheduler 产生定时和延迟任务；Lease 保证多实例下同一 Job 不被多个 Worker 同时认领。

## 1. Job 与 Run

Run 表示用户任务；Job 表示可调度执行单元。一个 Run 可能经历多次 Job Attempt，例如首次执行、审批后恢复、失败重试。不要把 Job Attempt 与 Run ID 混为一体。

## 2. Job Claim

Worker 领取 Job 时原子地锁定候选行、检查 status、检查 scheduled_at、写入 lease_owner、写入 lease_expires_at、增加 attempt_count、更新 heartbeat。

```sql
WITH candidate AS (
  SELECT id
  FROM agent_job
  WHERE job_status = 'queued'
    AND (scheduled_at IS NULL OR scheduled_at <= now())
  ORDER BY priority ASC, created_at ASC
  FOR UPDATE SKIP LOCKED
  LIMIT 1
)
UPDATE agent_job j
SET job_status = 'running',
    lease_owner = :worker_id,
    lease_expires_at = now() + interval '60 seconds',
    heartbeat_at = now(),
    attempt_count = attempt_count + 1,
    updated_at = now()
FROM candidate
WHERE j.id = candidate.id
RETURNING j.*;
```

领取事务必须尽快提交；实际模型和工具执行不能持有数据库行锁。

## 3. Lease

Worker 周期性 Heartbeat。Lease 过期后 Reaper 才允许重新分配。重新执行前检查 Run、Checkpoint、Tool UNKNOWN 和 Cancel Request。

## 4. 队列隔离

建议分队列 interactive、long_running、approval_resume、indexing、evaluation、sandbox、scheduled。不同队列有独立并发和优先级，避免长任务阻塞交互任务。

## 5. Scheduler

Cron 只负责生成 Job，不直接在调度进程中运行 Agent。每次触发使用稳定 Schedule Key + Occurrence Time 生成幂等键，防止调度器重启后重复创建任务。

## 6. 背压

队列长度、等待时间、Tenant 并发、Provider 限流和 Sandbox 容量共同决定是否接受新任务。过载时返回排队或拒绝，不应无限积压。

## 7. 关闭与重启

Worker Shutdown：停止领取新 Job；标记正在执行任务为 draining；允许短任务完成；长任务保存 Checkpoint 或标记可恢复；释放 Lease；关闭资源。

## 8. 测试

双 Worker 竞争、Worker Kill -9、Lease 过期、重复 Job、调度器重启、队列过载、长任务取消和 Job 迁移。
