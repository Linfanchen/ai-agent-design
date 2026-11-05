# PostgreSQL 核心数据模型与 DDL

## 本文档在系统中的作用

给出 Agent 平台控制面、运行面、工具执行、记忆、Catalog、审批、审计和评测的核心数据模型。它不是 LangGraph 内部 Checkpoint 表的替代；Checkpoint 建议由所选 LangGraph Checkpointer 官方实现管理。

## 1. 建模原则

- 所有租户资源显式带 tenant_id；
- 版本化配置不可变；
- Run / Job / Tool Execution 状态使用明确状态机；
- Event / Audit / Usage 采用追加式记录；
- 大文件和长文本优先放对象存储，DB 保存引用；
- JSONB 用于扩展元数据，不替代高频查询列；
- 外键和唯一约束阻止重复和孤儿关系；
- 对高增长表制定分区与保留策略。

## 2. 核心 DDL

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE agent_definition (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id uuid,
    agent_key varchar(120) NOT NULL,
    display_name varchar(180) NOT NULL,
    description text,
    agent_kind varchar(32) NOT NULL,
    definition_status varchar(24) NOT NULL DEFAULT 'draft',
    created_at timestamptz NOT NULL DEFAULT now(),
    updated_at timestamptz NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, agent_key)
);
COMMENT ON TABLE agent_definition IS 'Agent 逻辑定义，不包含运行时对象和密钥';

CREATE TABLE agent_release (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id uuid,
    agent_definition_id uuid NOT NULL REFERENCES agent_definition(id),
    version varchar(40) NOT NULL,
    manifest jsonb NOT NULL,
    manifest_hash varchar(128) NOT NULL,
    graph_key varchar(160) NOT NULL,
    catalog_snapshot_id uuid,
    release_status varchar(24) NOT NULL DEFAULT 'draft',
    published_at timestamptz,
    created_at timestamptz NOT NULL DEFAULT now(),
    UNIQUE (agent_definition_id, version)
);
COMMENT ON TABLE agent_release IS '不可变 Agent 发布快照';

CREATE TABLE model_provider (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    provider_key varchar(100) NOT NULL UNIQUE,
    display_name varchar(180) NOT NULL,
    api_mode varchar(60) NOT NULL,
    base_url text,
    credential_ref varchar(300),
    provider_status varchar(24) NOT NULL DEFAULT 'enabled',
    created_at timestamptz NOT NULL DEFAULT now()
);
COMMENT ON TABLE model_provider IS '模型 Provider 元数据，密钥只保存 Secret Reference';

CREATE TABLE model_profile (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id uuid,
    profile_key varchar(140) NOT NULL,
    version varchar(40) NOT NULL,
    provider_id uuid NOT NULL REFERENCES model_provider(id),
    model_id varchar(180) NOT NULL,
    capability_snapshot jsonb NOT NULL,
    request_defaults jsonb NOT NULL DEFAULT '{}'::jsonb,
    routing_policy jsonb NOT NULL DEFAULT '{}'::jsonb,
    profile_status varchar(24) NOT NULL DEFAULT 'draft',
    created_at timestamptz NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, profile_key, version)
);
COMMENT ON TABLE model_profile IS '版本化模型配置与能力快照';

CREATE TABLE conversation (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id uuid NOT NULL,
    owner_user_id uuid NOT NULL,
    title varchar(300),
    conversation_status varchar(24) NOT NULL DEFAULT 'active',
    created_at timestamptz NOT NULL DEFAULT now(),
    updated_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_conversation_owner ON conversation(tenant_id, owner_user_id, updated_at DESC);
COMMENT ON TABLE conversation IS '用户可见的对话容器';

CREATE TABLE agent_thread (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id uuid NOT NULL,
    conversation_id uuid REFERENCES conversation(id),
    graph_thread_key varchar(180) NOT NULL,
    state_schema_version integer NOT NULL DEFAULT 1,
    thread_status varchar(24) NOT NULL DEFAULT 'active',
    created_at timestamptz NOT NULL DEFAULT now(),
    updated_at timestamptz NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, graph_thread_key)
);
COMMENT ON TABLE agent_thread IS '应用 Thread 与 LangGraph thread_id 的映射';

CREATE TABLE agent_run (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id uuid NOT NULL,
    thread_id uuid NOT NULL REFERENCES agent_thread(id),
    agent_release_id uuid NOT NULL REFERENCES agent_release(id),
    requested_by uuid NOT NULL,
    run_status varchar(32) NOT NULL DEFAULT 'queued',
    termination_reason varchar(80),
    idempotency_key varchar(180) NOT NULL,
    request_hash varchar(128) NOT NULL,
    input_ref text,
    result_ref text,
    budget_snapshot jsonb NOT NULL DEFAULT '{}'::jsonb,
    usage_summary jsonb NOT NULL DEFAULT '{}'::jsonb,
    started_at timestamptz,
    finished_at timestamptz,
    created_at timestamptz NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, idempotency_key)
);
CREATE INDEX idx_agent_run_thread ON agent_run(tenant_id, thread_id, created_at DESC);
CREATE INDEX idx_agent_run_status ON agent_run(run_status, created_at);
COMMENT ON TABLE agent_run IS '一次用户任务执行的应用级权威记录';

CREATE TABLE agent_job (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id uuid NOT NULL,
    run_id uuid NOT NULL REFERENCES agent_run(id),
    job_type varchar(60) NOT NULL,
    queue_name varchar(80) NOT NULL DEFAULT 'default',
    priority integer NOT NULL DEFAULT 100,
    job_status varchar(24) NOT NULL DEFAULT 'queued',
    scheduled_at timestamptz,
    lease_owner varchar(180),
    lease_expires_at timestamptz,
    heartbeat_at timestamptz,
    attempt_count integer NOT NULL DEFAULT 0,
    max_attempts integer NOT NULL DEFAULT 5,
    last_error_code varchar(100),
    created_at timestamptz NOT NULL DEFAULT now(),
    updated_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_agent_job_claim ON agent_job(queue_name, job_status, scheduled_at, priority, created_at);
CREATE INDEX idx_agent_job_lease ON agent_job(job_status, lease_expires_at);
COMMENT ON TABLE agent_job IS 'Worker 可领取的持久化任务';

CREATE TABLE agent_run_step (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id uuid NOT NULL,
    run_id uuid NOT NULL REFERENCES agent_run(id),
    step_index integer NOT NULL,
    node_key varchar(160) NOT NULL,
    step_status varchar(24) NOT NULL,
    input_ref text,
    output_ref text,
    error_code varchar(100),
    started_at timestamptz,
    finished_at timestamptz,
    UNIQUE (run_id, step_index)
);
COMMENT ON TABLE agent_run_step IS 'Run 内部节点步骤索引';

CREATE TABLE agent_event (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id uuid NOT NULL,
    run_id uuid NOT NULL REFERENCES agent_run(id),
    sequence_no bigint NOT NULL,
    event_type varchar(100) NOT NULL,
    schema_version integer NOT NULL DEFAULT 1,
    visibility varchar(24) NOT NULL DEFAULT 'internal',
    payload jsonb NOT NULL,
    occurred_at timestamptz NOT NULL DEFAULT now(),
    UNIQUE (run_id, sequence_no)
);
CREATE INDEX idx_agent_event_replay ON agent_event(run_id, sequence_no);
COMMENT ON TABLE agent_event IS 'Run 事件日志，支持客户端重连和任务回放';

CREATE TABLE tool_definition (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tool_key varchar(180) NOT NULL UNIQUE,
    display_name varchar(180) NOT NULL,
    domain varchar(120) NOT NULL,
    description text NOT NULL,
    definition_status varchar(24) NOT NULL DEFAULT 'draft',
    created_at timestamptz NOT NULL DEFAULT now()
);
COMMENT ON TABLE tool_definition IS '稳定 Tool 逻辑身份';

CREATE TABLE tool_version (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tool_definition_id uuid NOT NULL REFERENCES tool_definition(id),
    version varchar(40) NOT NULL,
    input_schema jsonb NOT NULL,
    output_schema jsonb NOT NULL,
    risk_level varchar(24) NOT NULL,
    side_effect_class varchar(32) NOT NULL,
    timeout_seconds integer NOT NULL DEFAULT 30,
    idempotency_mode varchar(32) NOT NULL DEFAULT 'required',
    required_permissions jsonb NOT NULL DEFAULT '[]'::jsonb,
    handler_ref varchar(240) NOT NULL,
    version_status varchar(24) NOT NULL DEFAULT 'draft',
    created_at timestamptz NOT NULL DEFAULT now(),
    UNIQUE (tool_definition_id, version)
);
COMMENT ON TABLE tool_version IS '不可变 Tool Schema 与执行契约版本';

CREATE TABLE tool_binding (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id uuid,
    agent_release_id uuid NOT NULL REFERENCES agent_release(id),
    tool_version_id uuid NOT NULL REFERENCES tool_version(id),
    binding_status varchar(24) NOT NULL DEFAULT 'enabled',
    created_at timestamptz NOT NULL DEFAULT now(),
    UNIQUE (agent_release_id, tool_version_id)
);
COMMENT ON TABLE tool_binding IS 'Agent Release 对 Tool Version 的显式绑定';

CREATE TABLE tool_execution (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id uuid NOT NULL,
    run_id uuid NOT NULL REFERENCES agent_run(id),
    tool_version_id uuid NOT NULL REFERENCES tool_version(id),
    model_tool_call_id varchar(180),
    idempotency_key varchar(180) NOT NULL,
    arguments_hash varchar(128) NOT NULL,
    execution_status varchar(32) NOT NULL DEFAULT 'created',
    policy_decision_ref varchar(180),
    external_operation_ref varchar(240),
    result_ref text,
    error_code varchar(100),
    started_at timestamptz,
    finished_at timestamptz,
    created_at timestamptz NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, idempotency_key)
);
CREATE INDEX idx_tool_execution_run ON tool_execution(run_id, created_at);
CREATE INDEX idx_tool_execution_unknown ON tool_execution(execution_status, created_at);
COMMENT ON TABLE tool_execution IS 'Tool 执行、幂等和外部副作用对账记录';

CREATE TABLE approval_request (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id uuid NOT NULL,
    run_id uuid NOT NULL REFERENCES agent_run(id),
    tool_execution_id uuid REFERENCES tool_execution(id),
    action varchar(180) NOT NULL,
    resource_refs jsonb NOT NULL,
    proposed_change jsonb NOT NULL,
    intent_hash varchar(128) NOT NULL,
    risk_level varchar(24) NOT NULL,
    approval_status varchar(24) NOT NULL DEFAULT 'pending',
    requested_by uuid NOT NULL,
    decided_by uuid,
    decision_reason text,
    expires_at timestamptz NOT NULL,
    decided_at timestamptz,
    created_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_approval_pending ON approval_request(tenant_id, approval_status, expires_at);
COMMENT ON TABLE approval_request IS '人工审批事实来源，Graph Interrupt 只引用此表';

CREATE TABLE memory_item (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id uuid NOT NULL,
    owner_type varchar(32) NOT NULL,
    owner_id uuid NOT NULL,
    scope_type varchar(32) NOT NULL,
    scope_id varchar(180) NOT NULL,
    memory_type varchar(32) NOT NULL,
    subject varchar(300),
    predicate varchar(180),
    object_value jsonb,
    memory_text text NOT NULL,
    confidence numeric(5,4) NOT NULL,
    importance numeric(5,4) NOT NULL,
    sensitivity varchar(24) NOT NULL DEFAULT 'normal',
    memory_status varchar(24) NOT NULL DEFAULT 'candidate',
    version integer NOT NULL DEFAULT 1,
    supersedes_id uuid REFERENCES memory_item(id),
    valid_from timestamptz,
    valid_until timestamptz,
    created_at timestamptz NOT NULL DEFAULT now(),
    updated_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_memory_scope_status ON memory_item(tenant_id, scope_type, scope_id, memory_status);
CREATE INDEX idx_memory_subject_predicate ON memory_item(tenant_id, subject, predicate);
COMMENT ON TABLE memory_item IS '带 Scope、版本、有效期和治理状态的记忆条目';

CREATE TABLE memory_evidence (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    memory_item_id uuid NOT NULL REFERENCES memory_item(id),
    source_type varchar(40) NOT NULL,
    source_ref varchar(300) NOT NULL,
    evidence_hash varchar(128),
    evidence_excerpt text,
    created_at timestamptz NOT NULL DEFAULT now(),
    UNIQUE (memory_item_id, source_type, source_ref)
);
COMMENT ON TABLE memory_evidence IS '记忆来源证据与可追溯引用';

CREATE TABLE catalog_snapshot (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id uuid,
    version varchar(80) NOT NULL,
    source_commit varchar(128),
    scanner_version varchar(80) NOT NULL,
    checksum varchar(128) NOT NULL,
    snapshot_status varchar(24) NOT NULL DEFAULT 'draft',
    published_at timestamptz,
    created_at timestamptz NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, version)
);
COMMENT ON TABLE catalog_snapshot IS '不可变业务目录快照';
ALTER TABLE agent_release
    ADD CONSTRAINT fk_agent_release_catalog_snapshot
    FOREIGN KEY (catalog_snapshot_id) REFERENCES catalog_snapshot(id);

CREATE TABLE business_entity (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    catalog_snapshot_id uuid NOT NULL REFERENCES catalog_snapshot(id),
    entity_key varchar(180) NOT NULL,
    display_name varchar(180) NOT NULL,
    description text,
    schema_ref jsonb NOT NULL,
    relationships jsonb NOT NULL DEFAULT '[]'::jsonb,
    source_refs jsonb NOT NULL DEFAULT '[]'::jsonb,
    UNIQUE (catalog_snapshot_id, entity_key)
);
COMMENT ON TABLE business_entity IS 'Catalog 中的业务实体和关系';

CREATE TABLE business_capability (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    catalog_snapshot_id uuid NOT NULL REFERENCES catalog_snapshot(id),
    capability_key varchar(200) NOT NULL,
    domain varchar(120) NOT NULL,
    operation varchar(40) NOT NULL,
    target_entity varchar(180) NOT NULL,
    description text NOT NULL,
    input_schema jsonb NOT NULL,
    output_schema jsonb NOT NULL,
    api_binding jsonb NOT NULL,
    required_permissions jsonb NOT NULL DEFAULT '[]'::jsonb,
    risk_level varchar(24) NOT NULL,
    approval_policy_ref varchar(180),
    review_status varchar(24) NOT NULL DEFAULT 'pending',
    UNIQUE (catalog_snapshot_id, capability_key)
);
COMMENT ON TABLE business_capability IS '经审核的业务能力，不等同于原始 API';

CREATE TABLE skill_version (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id uuid,
    skill_key varchar(180) NOT NULL,
    version varchar(40) NOT NULL,
    manifest jsonb NOT NULL,
    package_uri text NOT NULL,
    checksum varchar(128) NOT NULL,
    scan_report jsonb NOT NULL DEFAULT '{}'::jsonb,
    skill_status varchar(24) NOT NULL DEFAULT 'draft',
    created_at timestamptz NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, skill_key, version)
);
COMMENT ON TABLE skill_version IS 'Skill 包版本和安全扫描结果';

CREATE TABLE plugin_installation (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id uuid,
    plugin_key varchar(180) NOT NULL,
    version varchar(40) NOT NULL,
    api_version varchar(40) NOT NULL,
    manifest jsonb NOT NULL,
    permission_grants jsonb NOT NULL DEFAULT '{}'::jsonb,
    credential_refs jsonb NOT NULL DEFAULT '[]'::jsonb,
    install_status varchar(24) NOT NULL DEFAULT 'disabled',
    installed_at timestamptz NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, plugin_key, version)
);
COMMENT ON TABLE plugin_installation IS 'Plugin 安装、授权与生命周期状态';

CREATE TABLE audit_record (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id uuid NOT NULL,
    actor_type varchar(32) NOT NULL,
    actor_id varchar(180) NOT NULL,
    action varchar(180) NOT NULL,
    resource_type varchar(100) NOT NULL,
    resource_ref varchar(240) NOT NULL,
    decision varchar(24),
    reason_code varchar(100),
    correlation_id varchar(180),
    payload_summary jsonb NOT NULL DEFAULT '{}'::jsonb,
    occurred_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_tenant_time ON audit_record(tenant_id, occurred_at DESC);
CREATE INDEX idx_audit_resource ON audit_record(tenant_id, resource_type, resource_ref);
COMMENT ON TABLE audit_record IS '结构化追加式审计记录';

CREATE TABLE usage_ledger (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id uuid NOT NULL,
    run_id uuid REFERENCES agent_run(id),
    tool_execution_id uuid REFERENCES tool_execution(id),
    usage_type varchar(60) NOT NULL,
    provider_key varchar(100),
    model_id varchar(180),
    quantity numeric(24,8) NOT NULL,
    unit varchar(40) NOT NULL,
    amount numeric(24,10),
    currency varchar(12) NOT NULL DEFAULT 'USD',
    pricing_version varchar(60),
    occurred_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_usage_tenant_time ON usage_ledger(tenant_id, occurred_at DESC);
COMMENT ON TABLE usage_ledger IS '模型、工具、沙箱和其他资源用量账本';

CREATE TABLE evaluation_dataset (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_key varchar(180) NOT NULL,
    version varchar(40) NOT NULL,
    description text,
    dataset_status varchar(24) NOT NULL DEFAULT 'draft',
    created_at timestamptz NOT NULL DEFAULT now(),
    UNIQUE (dataset_key, version)
);
COMMENT ON TABLE evaluation_dataset IS '版本化评测数据集';

CREATE TABLE evaluation_case (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id uuid NOT NULL REFERENCES evaluation_dataset(id),
    case_key varchar(180) NOT NULL,
    input_payload jsonb NOT NULL,
    expected_payload jsonb,
    evaluator_config jsonb NOT NULL DEFAULT '{}'::jsonb,
    tags jsonb NOT NULL DEFAULT '[]'::jsonb,
    UNIQUE (dataset_id, case_key)
);
COMMENT ON TABLE evaluation_case IS '评测用例与期望结果';

CREATE TABLE evaluation_run (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id uuid NOT NULL REFERENCES evaluation_dataset(id),
    agent_release_id uuid NOT NULL REFERENCES agent_release(id),
    run_status varchar(24) NOT NULL DEFAULT 'queued',
    metrics jsonb NOT NULL DEFAULT '{}'::jsonb,
    report_ref text,
    started_at timestamptz,
    finished_at timestamptz,
    created_at timestamptz NOT NULL DEFAULT now()
);
COMMENT ON TABLE evaluation_run IS '固定数据集和 Agent Release 的评测执行';

CREATE TABLE outbox_event (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id uuid,
    aggregate_type varchar(100) NOT NULL,
    aggregate_id varchar(180) NOT NULL,
    event_type varchar(120) NOT NULL,
    payload jsonb NOT NULL,
    delivery_status varchar(24) NOT NULL DEFAULT 'pending',
    attempt_count integer NOT NULL DEFAULT 0,
    next_attempt_at timestamptz,
    created_at timestamptz NOT NULL DEFAULT now(),
    delivered_at timestamptz
);
CREATE INDEX idx_outbox_pending ON outbox_event(delivery_status, next_attempt_at, created_at);
COMMENT ON TABLE outbox_event IS '事务 Outbox，保证状态提交与事件投递可恢复';

ALTER TABLE tool_execution
    ADD COLUMN approval_id uuid REFERENCES approval_request(id);
```

## 3. 生产约束与迁移说明

1. `tenant_id` 的外键应关联实际 Tenant 主表；本 DDL 不重复定义用户与租户主数据。
2. `agent_release` / `model_profile` 等全局配置与租户配置混合时，应明确 PostgreSQL NULL 唯一性语义，必要时使用部分唯一索引。
3. 高增长 `agent_event`、`audit_record`、`usage_ledger` 结合保留期和查询模式评估时间分区。
4. `tool_execution` 幂等冲突需比较 `arguments_hash`，不能只依赖唯一键。
5. Run 级 `sequence_no` 通过事务内计数器 / 原子更新分配，避免并发重复。
6. Checkpointer 自带表由官方 migration / setup 管理，不手工复制内部表结构。
7. 向量索引表可独立于 `memory_item` / `business_entity`，但必须保留原始 ID、版本、Tenant 和 ACL 引用。
8. 所有状态字段应有 CHECK Constraint 或由应用状态机约束；生产版本进一步补齐索引、分区、外键和迁移脚本。
