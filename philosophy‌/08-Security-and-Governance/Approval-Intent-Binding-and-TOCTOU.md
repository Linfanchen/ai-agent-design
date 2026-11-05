# Approval：Intent Binding、TOCTOU 与安全恢复

## 本文档在系统中的作用

人工审批必须批准一个明确动作，而不是批准一个模糊的“继续执行”。本文件定义审批如何绑定 Tool 参数、资源版本、权限快照和有效期，防止审批后参数被替换。

## 1. Approval Intent

审批对象至少包含：
- tenant_id；
- run_id；
- tool_execution_id；
- tool_key / tool_version；
- principal_id；
- action；
- resource_refs；
- canonical_arguments_hash；
- expected_resource_version；
- policy_version；
- risk_level；
- expires_at。

## 2. Intent Hash

对规范化后的关键动作字段生成 Hash：
`hash(tenant + principal + action + resource refs + canonical args + resource version + tool version)`

审批后执行前重新计算 Hash。任何关键字段变化都使审批失效。

## 3. TOCTOU

Time-of-check to time-of-use：审批时检查资源版本为 12，审批等待期间资源被其他人更新为 13。执行时必须再次检查资源版本；若不匹配，停止执行并要求重新读取 / 审批。

## 4. 审批状态机

`pending → approved / rejected / expired / cancelled`

审批后可进入执行状态，但 Approval 本身不等于 Tool 已成功。Tool Execution 独立记录结果。

## 5. 审批人权限

审批人不能仅因为是“当前登录用户”就自动拥有审批权。需要检查：
- 对目标资源有审批权限；
- 不是被禁止的自批角色（若策略禁止）；
- 审批金额 / 风险在授权额度内；
- 审批时身份有效；
- 不能通过 Agent 模拟审批人。

## 6. Resume 的校验

Resume API 只接受 `approval_id` 和结构化决策引用，不接受任意 Graph State。服务端先读取 Approval 事实，验证状态、审批人、Intent Hash、Resource Version 和有效期，再创建 Resume Job。

## 7. 执行前二次授权

审批通过后，执行器仍需重新运行 Policy。权限可能已被撤销，租户状态可能变化，Tool Version 也可能被紧急撤销。审批不能覆盖平台 Deny。

## 8. 证据与审计

记录审批人、时间、理由、Intent Hash、资源版本、Policy Version、Tool Version 和执行结果。审批页面展示清晰的人类可读动作摘要，并提供资源链接和风险说明。

## 9. 测试

审批后修改参数、资源版本变化、审批人权限撤销、审批过期、重复 Resume、拒绝后重放、管理员紧急撤销、审批成功但执行失败。
