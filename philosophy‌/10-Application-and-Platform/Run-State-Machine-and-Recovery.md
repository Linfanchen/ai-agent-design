# Run 状态机、Job 协调与恢复流程

## 本文档在系统中的作用

Run 是用户可见的一次任务事实；Job 是 Worker 的调度事实；Graph Checkpoint 是编排恢复事实。三者必须有明确映射，避免任务 UI、Worker 和 LangGraph 各自维护一套互相冲突的状态。

## 1. Run 状态机

`created → queued → running`

从 running 可进入：
- `waiting_input`
- `waiting_approval`
- `succeeded`
- `failed`
- `cancel_requested`
- `cancelled`
- `timed_out`
- `budget_exceeded`

恢复时可能从 waiting_input / waiting_approval 回到 queued，再由 Worker 继续 running。最终状态不可随意回退。

## 2. Job 状态机

`queued → leased → running → succeeded / failed / retry_scheduled / dead_letter`

Lease 过期后进入 reclaim 流程。Job Retry 不意味着重新创建新的用户 Run。

## 3. 关键状态协调规则

- Run 创建与 Job 入队同事务；
- Worker Claim Job 后更新 Lease；
- Worker 执行前验证 Run 仍可执行；
- Graph Interrupt 时持久化 Checkpoint，再把 Run 标记 waiting；
- 用户 Resume 时创建新 Job，不直接在 API 进程内恢复 Graph；
- Run 最终状态与 Job 最终状态分开记录；
- Run 终态后拒绝重复 Resume。

## 4. Worker Crash 恢复

1. Worker 进程消失；
2. Heartbeat 停止；
3. Lease 到期；
4. Reaper 标记旧 Attempt Lost；
5. 检查 Run Cancel Request / Deadline；
6. 检查未完成 Tool Execution；
7. 对 UNKNOWN 副作用进行对账；
8. 读取最近有效 Checkpoint；
9. 创建新 Job Attempt；
10. 恢复 Graph。

## 5. Cancel 语义

Cancel API 首先写入 `cancel_requested_at` 并发布事件。Worker 在 Node 边界和 Tool 边界检查取消。对已提交的外部写操作，不保证回滚；必须完成对账并向用户说明实际结果。

## 6. Deadline 与 Timeout

区分：
- Request Timeout：客户端等待时间；
- Run Deadline：整个任务最大时间；
- Model Call Timeout；
- Tool Timeout；
- Approval Expiry；
- Lease Expiry。

这些时间不能共用一个模糊的 `timeout` 字段。

## 7. 恢复判定

恢复前检查：
- Release 是否仍可执行；
- Policy 是否被撤销；
- Tool Version 是否仍启用；
- Checkpoint Schema 是否可迁移；
- Approval 是否有效；
- Budget 是否仍有余额；
- 外部副作用是否已发生。

## 8. 验收

双 Worker 竞争、Worker Kill -9、Lease 过期、重复 Resume、取消与完成竞态、Checkpoint 失败、Approval 过期、UNKNOWN 写操作、Deadline 到期。
