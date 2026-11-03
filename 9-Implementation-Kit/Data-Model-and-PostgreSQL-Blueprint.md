# 21. Storage 与 Data Model

## Role in the Agent System

Defines storage ownership and a PostgreSQL-oriented data model for runtime, memory, tools, governance, and catalog.

**Primary responsibility:** Own this capability as a reusable subsystem with explicit contracts, state ownership, security boundaries, and observable outcomes.

**Boundary:** This subsystem must not silently take over responsibilities owned by adjacent layers. Its public interface should be testable without running a full Agent conversation.

## 1. 存储职责划分

| 存储 | 适合数据 | 不适合 |
|---|---|---|
| PostgreSQL | Agent 配置、Session 元数据、Event、Tool Execution、Approval、Memory、Catalog | 大型二进制附件 |
| Redis | Cache、短锁、Rate Limit、短期队列协调 | 唯一事实来源 |
| Object Storage | 附件、报告、Workspace 大文件、导出包 | 高频关系查询 |
| FTS / Search | 文本索引、API / Skill / Memory 搜索 | 业务事务 |
| Vector Index | 语义召回 | Canonical Memory / 权限事实 |
| LangGraph Checkpointer | Graph State 快照 | 全部业务审计 |

## 2. 建议核心表域

### Agent / Runtime
- `agent_definition`
- `agent_version`
- `agent_runtime`
- `agent_task`
- `agent_task_step`

### Session / Event
- `agent_session`
- `agent_session_event`
- `agent_checkpoint_ref`
- `agent_turn`
- `agent_step`

### Tool
- `agent_tool_definition`
- `agent_tool_version`
- `agent_tool_execution`
- `agent_tool_discovery_log`

### Memory
- `agent_memory_item`
- `agent_memory_candidate`
- `agent_memory_evidence`
- `agent_memory_entity`
- `agent_memory_relation`
- `agent_memory_episode`

### Skill / Plugin
- `agent_skill`
- `agent_skill_version`
- `agent_skill_evaluation`
- `agent_plugin`
- `agent_plugin_version`
- `agent_plugin_installation`

### Business Catalog
- `business_domain`
- `business_entity`
- `business_field`
- `business_entity_relation`
- `business_api_operation`
- `business_api_schema`
- `business_page_binding`
- `business_capability`
- `business_capability_api`
- `business_state_machine`
- `business_permission_binding`

### Governance / Evaluation
- `agent_approval`
- `agent_audit_log`
- `agent_trace`
- `agent_trajectory`
- `agent_evaluation_run`

## 3. 字段规范

统一使用：
- `id`：UUID / ULID。
- `tenant_id`：多租户隔离字段。
- `created_at` / `updated_at`：UTC 时间。
- `created_by` / `updated_by`。
- `version`：乐观锁或业务版本。
- `status`：明确状态枚举。
- `metadata`：只放非核心、可演进扩展字段，不能把所有结构塞 JSONB。

表和字段应有注释。敏感字段加密或不落库。

## 4. Event Log 分区

高写入量的 Event Log 可以按时间或 Tenant 分区，但需先评估索引、查询和保留策略。必须保证每个 Session 的 sequence 单调递增。

## 5. JSONB 使用边界

适合 Provider 原始元数据、可扩展 Prompt 参数、Tool 原始请求/响应快照（脱敏后）、Skill Frontmatter。

不适合需要频繁 JOIN 的核心关系、权限判断关键字段、高选择性过滤但无索引的字段、需要强约束的状态字段。

## 6. 数据保留

定义 Session、Event、Trace、Memory、Tool Result 大对象、Checkpoint、Approval / Audit 的保留期，以及用户删除请求处理。保留期限按业务和合规要求配置，不在代码中硬编码。
