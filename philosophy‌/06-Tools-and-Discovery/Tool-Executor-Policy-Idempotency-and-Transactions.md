# Tool Executor：策略、幂等、事务与对账

## 本文档在系统中的作用

Tool Executor 是模型意图与真实副作用之间的安全闸门。模型只提出 Tool Call；Executor 负责判断是否允许、是否需要审批、是否重复、如何执行、如何验证和如何审计。

## 1. 执行状态机

`created → validated → authorized → awaiting_approval → ready → executing → succeeded`

异常分支：rejected、failed_retryable、failed_terminal、unknown、cancelled_before_execution、compensating、compensated。

`unknown` 很重要：请求超时、连接断开或 Worker 崩溃时，系统可能不知道外部副作用是否已发生。

## 2. 执行管线

1. 解析 Tool Definition / Version；
2. 校验 Tenant / Agent Binding；
3. Pydantic Input Validation；
4. 业务参数校验；
5. 重新鉴权；
6. Policy Decision；
7. 风险评估；
8. 审批检查；
9. 幂等记录；
10. 预算预留；
11. 写入 Execution Intent；
12. 调用 Handler；
13. 验证结果；
14. 写入 Execution Outcome；
15. Audit / Event / Usage；
16. 返回结构化 Tool Result。

## 3. 幂等键

推荐 `tenant_id + tool_key + idempotency_key`，并保存 `arguments_hash`。相同 Key、相同 Hash 返回已有结果；相同 Key、不同 Hash 返回冲突。不能仅依赖模型生成的 tool_call_id，因为重试时可能产生新 ID。

## 4. 外部副作用对账

对于外部系统支持幂等键的写操作，传递相同 Idempotency Key。若不支持，尽量使用业务唯一键或操作查询接口。超时后进入 UNKNOWN，由 Reconciliation Worker 查询外部状态。无法判断时转人工，不能自动再次写入。

## 5. 数据库事务边界

本地数据库中的 Execution Intent、状态变更和 Outbox 可在同一事务提交；外部 HTTP API 不在同一个 PostgreSQL 事务中。不要长时间打开数据库事务等待外部 API。

## 6. Outbox Pattern

在事务中同时写业务记录和 Outbox Event；后台 Worker 投递消息；成功后标记 delivered。消费者通过 Inbox / Dedup Key 防止重复处理。

## 7. 补偿操作

补偿不是数据库 Rollback。若外部操作已成功，需要显式定义补偿 Tool，例如取消预订、撤销分配。补偿本身也是有副作用操作，需要权限、幂等和审计。

## 8. ToolResult Contract

execution_id、status、safe_summary、output_ref、evidence_refs、retryable、error_code、side_effect_confirmed、next_action_hint。不要把完整内部异常堆栈返回模型。

## 9. 必测故障

调用前崩溃、调用中断网、外部成功但响应丢失、结果写库失败、重复消息、幂等冲突、审批撤销、权限中途撤销、补偿失败。
