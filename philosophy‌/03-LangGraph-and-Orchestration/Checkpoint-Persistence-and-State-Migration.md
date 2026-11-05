# Checkpoint、持久化与 State 迁移

## 本文档在系统中的作用

Checkpoint 让 Graph 在节点边界保存执行状态，使中断、恢复、回放和调试成为可能。本文件定义 Checkpoint 与 Run DB、Event Log、Tool Execution 的边界，以及 State Schema 变化时如何兼容历史任务。

## 1. 四种持久化对象

| 对象 | 保存什么 | 主要用途 |
|---|---|---|
| LangGraph Checkpoint | Graph State / channel versions / pending tasks | Graph 恢复 |
| Run DB | Run 状态、Release、预算、Job、完成结果引用 | 应用状态机 |
| Event Log | 执行事件、工具调用、用户可见进度 | 追踪与 UI |
| Tool Execution Store | 参数摘要、状态、幂等键、结果引用 | 副作用可靠性 |

Checkpoint 不能当审计记录；Event Log 不能当可恢复 Graph State。

## 2. Checkpoint 频率

LangGraph 持久化通常围绕 super-step / 节点推进保存状态。不要对每个 Token Delta 做数据库 Checkpoint；流式 Token 可作为短期事件或合并事件处理。

## 3. State Schema Version

在 State 中保存 `state_schema_version`。升级时通过 `old_state → normalize → migrate → validate → new_state`。

迁移必须可重复、有版本号、保留未知字段或按策略丢弃、对必需字段提供安全默认、不伪造业务事实，并有旧版本样本测试。

## 4. Resume 前置检查

Run 状态允许恢复；Checkpoint 存在；Thread 与 Tenant 匹配；当前 Principal 有权限；Release 仍可执行或通过兼容策略；Tool Binding 未被安全撤销；Approval / Clarification 未过期；预算与 Deadline 允许继续。

## 5. 外部副作用与 Checkpoint 原子性

数据库 Checkpoint 与外部业务 API 通常不在同一事务。因此 Tool Execution 先持久化 intent；外部调用带 idempotency key；收到结果后持久化 outcome；Worker 中间崩溃时状态为 UNKNOWN；恢复时通过外部查询接口对账；无法对账时进入人工处理。

## 6. 时间旅行与生产安全

Checkpoint 可用于调试和状态回溯，但生产环境不能随意从旧 Checkpoint 恢复并重新执行写操作。回放必须使用只读 / Mock Tool 或隔离环境。若要重新执行业务动作，产生新的 Run 和审批。

## 7. 保留与清理

Checkpoint 保留期按任务类型、合规要求和调试价值确定。删除 Thread 时协调 Checkpoint、Event、Artifact、Memory 引用和审计保留规则。
