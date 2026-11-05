# PostgreSQL：分区、索引、保留与数据生命周期

## 本文档在系统中的作用

V4 数据模型包含高增长的 Event、Audit、Usage、Trajectory 和 Tool Execution。它们的查询模式和保留要求不同，必须提前设计分区、索引和归档策略。

## 1. 表增长分级

高增长：agent_event、audit_record、usage_ledger、trajectory_event、model_call_log。
中增长：agent_run、tool_execution、agent_job、checkpoint metadata。
低增长：agent_definition、agent_release、tool_definition、tool_version、catalog_snapshot、policy version。

## 2. 时间分区

对高增长、主要按时间范围查询的表可使用 RANGE 分区（按月或按日）。分区键选择要结合唯一约束：PostgreSQL 分区表的唯一约束通常需要包含分区键，因此业务级唯一性可能需要独立去重表或额外约束设计。

## 3. 索引原则

- Run 事件：`(run_id, sequence_no)`；
- Tenant 时间查询：`(tenant_id, occurred_at DESC)`；
- Job Claim：`(queue_name, job_status, scheduled_at, priority)`；
- UNKNOWN 对账：`(execution_status, created_at)`；
- 审批待办：`(tenant_id, approval_status, expires_at)`；
- Memory Scope：`(tenant_id, scope_type, scope_id, memory_status)`。

索引不是越多越好。写入密集表应监控索引维护成本和膨胀。

## 4. JSONB 索引

只为稳定、高频过滤的 JSONB 字段建立 GIN 或表达式索引。不能把所有 Payload 都建 GIN。业务关键查询应尽可能投影到显式列。

## 5. 归档策略

Event / Audit / Usage 可按政策转冷存储或只读归档。归档前校验行数、Checksum 和时间范围；归档后仍要保留查询入口或导出流程。

## 6. 删除与备份

在线删除不代表备份立即物理消失。定义主库、只读副本、对象存储、备份和外部日志的保留周期。法律保留要求与用户内容删除要求需要分层管理。

## 7. PostgreSQL RLS

RLS 可作为纵深防御。连接池事务必须正确设置 Tenant Context 并在事务结束后重置。建议用专用 DB Role 测试 RLS，避免超级用户绕过策略导致测试假阳性。

## 8. 容量观测

表大小、分区大小、索引膨胀、Autovacuum、死元组、慢查询、连接池等待、锁等待、WAL 量、归档延迟和备份恢复演练。
