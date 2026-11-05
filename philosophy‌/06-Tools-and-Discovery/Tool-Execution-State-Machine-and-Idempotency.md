# Tool Execution：状态机、幂等与外部副作用对账

## 本文档在系统中的作用

Tool Execution 是平台执行外部动作的权威记录。它必须解决模型重复提出调用、Worker 重启、网络超时、业务系统已成功但响应丢失等问题。

## 1. 执行状态机

`created → validated → authorized → approval_pending → ready → running → succeeded`

其他终态 / 中间态：
- `denied`
- `invalid`
- `failed`
- `cancelled`
- `unknown`
- `reconciling`
- `compensated`

不是每个 Tool 都会经过 Approval；状态迁移由 Side Effect Class 和 Policy 决定。

## 2. Idempotency Key

推荐组成：
`tenant_id + run_id + tool_key + tool_version + model_tool_call_id`

对于同一业务意图跨 Run 重试，还可使用业务级 Request Key。Key 必须由服务端规范化生成，不能完全信任模型传入的 ID。

## 3. 参数 Hash

对 Pydantic 校验并规范化后的参数进行 Canonical JSON 序列化，再计算 Hash。相同 Idempotency Key + 相同 Hash 返回既有执行结果；相同 Key + 不同 Hash 返回 Conflict。

## 4. UNKNOWN 的必要性

请求超时可能发生在：
- 请求尚未发送；
- 请求发送但服务端未处理；
- 服务端处理成功但响应丢失；
- 服务端处理失败但响应丢失；
- 业务成功但本地结果持久化失败。

客户端超时不等于业务失败。非幂等写操作无法确定结果时必须进入 UNKNOWN。

## 5. Reconciliation

Reconciliation Worker 使用外部 Operation ID、业务资源状态、业务审计记录或查询接口确认结果：
- 已成功：标记 succeeded；
- 明确未执行：可重新提交；
- 明确失败：标记 failed；
- 仍无法判断：保持 unknown 并告警 / 人工介入。

## 6. Compensation

补偿不是数据库事务回滚。若外部副作用已经发生，补偿是另一个业务动作，必须有独立权限、审批、幂等键和审计记录。不能假设所有动作都可补偿。

## 7. Transactional Outbox

本地 DB 中 Tool Execution、Run Event 和 Outbox 尽可能同事务提交。外部 HTTP 调用不放在 DB Transaction 中。流程通常是先持久化执行意图，再调用外部系统，再持久化结果。

## 8. 并发冲突

对同一资源写操作可使用业务锁、乐观版本号或串行队列。建议输入携带 `expected_resource_version`，业务系统执行 Compare-and-Set。版本冲突返回 PRECONDITION_FAILED，Agent 需重新读取而非盲目覆盖。

## 9. 必测场景

重复调用、参数 Hash 冲突、Worker 在请求前退出、请求发出后退出、外部成功后本地写入失败、UNKNOWN 对账、审批过期、业务版本冲突、补偿失败。
