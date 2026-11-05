# Audit、隐私与数据保留

## 本文档在系统中的作用

Audit 保存 Agent 关键执行事实，Privacy 定义数据如何收集、使用、保留和删除。两者必须共同设计：不能为审计无限保留敏感全文，也不能为隐私删除所有责任证据。

## 1. Audit 与普通日志

Audit 是结构化、可查询、受访问控制的事实记录；普通日志用于排障。Audit 不应被普通应用代码随意更新或删除。

## 2. 审计事件

run.created / completed / failed；model.requested / responded；tool.proposed / authorized / denied；tool.execution_started / succeeded / failed / unknown；approval.requested / approved / rejected；memory.created / updated / tombstoned；policy.changed；plugin.installed / disabled；artifact.created / downloaded / deleted。

## 3. 最小记录原则

保存 ID、版本、摘要、参数 Hash、结果引用、决策、时间和 Actor。敏感全文默认不写普通 Audit；如必须保存，使用独立加密存储和严格权限。

## 4. 删除与保留

按数据类别定义 Conversation Content、Memory、Artifact、Audit、Usage Ledger、Checkpoint、RAG Index、Temporary Files。删除流程传播到缓存、向量索引、摘要、对象存储和派生数据。法定审计保留与用户内容删除明确区分。

## 5. 脱敏

日志和 Trace 使用字段级 Redaction。禁止记录 Authorization Header、API Key、Cookie、密码、原始凭证。Trace Attributes 优先保存资源 ID 和分类，而不是完整内容。

## 6. 审计完整性

可采用追加写、哈希链、分区归档和受控访问。哈希链不能防止拥有数据库最高权限的攻击者重写所有数据，仍需结合外部归档和权限分离。

## 7. 审计事件的字段规范

每条 Audit 至少包括 event_id、tenant_id、actor_type、actor_id、action、resource_type、resource_ref、decision、reason_code、policy_version、run_id、tool_execution_id、correlation_id、occurred_at。对于写操作，还应保存 intent_hash、before_version、after_version 或外部 operation_ref。

## 8. 审计写入策略

高风险操作的审计意图应在执行前持久化，结果在执行后追加。若审计系统不可用，是否允许执行取决于风险等级；Critical 写操作必须 Fail-closed。审计不能只依赖应用日志异步收集。

## 9. 数据保留矩阵

按 Conversation、Checkpoint、Tool Result、Artifact、Memory、Audit、Usage、RAG Source 和 Vector Index 分别定义保留期。不要用一个全局 TTL 删除所有 Agent 数据。

## 10. 删除任务

删除请求创建 Data Erasure Job，列出直接数据、派生数据、索引、缓存和备份处理状态。备份可能有独立保留周期，必须明确其到期清理方式。删除失败可重试，且要能向用户准确报告已完成范围。

## 11. 访问控制

审计查询本身也需要 RBAC / ABAC。普通用户只能查看自己的任务摘要；管理员按租户范围查看；安全审计员可以访问受控详情。审计导出必须记录操作者和导出范围。
