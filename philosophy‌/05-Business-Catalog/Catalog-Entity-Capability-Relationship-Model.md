# Catalog：Entity、Relationship、Capability 与业务规则模型

## 本文档在系统中的作用

本文件定义业务语义目录的核心 Schema，解决如何把技术 API 资产转成模型可理解、执行器可验证的业务动作。Catalog 是 Agent 理解业务系统的桥梁，不是 API 网关或业务数据库的替代。

## 1. Domain / Entity / Attribute

Domain 是业务边界，例如 mentorship、crm、billing。Entity 是领域对象，例如 MentorshipProject、Learner、Mentor。Attribute 是实体属性，需区分公开、内部、敏感和受限字段。

每个 Entity 至少包含：
- stable entity_key；
- display_name / aliases；
- primary identifiers；
- attributes；
- relationships；
- lifecycle / status model；
- source refs；
- owner domain；
- sensitivity profile。

## 2. Relationship

Relationship 描述实体之间的基数和方向：
- project has_many learners；
- learner belongs_to mentor；
- mentor belongs_to department。

关系需要说明查询方式、权限继承规则、删除行为和有效时间。不要仅靠模型从字段名猜关系。

## 3. Capability

Capability 是模型使用的业务动作：
- `mentorship.project.search`
- `mentorship.project.get`
- `mentorship.learner.search`
- `mentorship.learner.get`
- `mentorship.learner.status.transition`

每个 Capability 定义目标实体、操作类型、输入输出 Schema、前置条件、权限、风险、审批、幂等和数据新鲜度要求。

## 4. API Binding

API Binding 说明 Capability 如何落到具体实现：
- adapter_type：http / python_service / sql_read_model / workflow / mcp；
- service_key；
- operation_id；
- method / path；
- request_mapping；
- response_mapping；
- timeout；
- retry_policy；
- idempotency_mode；
- source_commit。

模型不直接读取 API Binding 中的内部路径和服务地址；它看到的是 Capability Contract。

## 5. Business Rule

状态机规则示例：
- active → completed：要求必填阶段已完成；
- active → paused：需要 pause_reason；
- completed → active：需要管理员角色和特殊审批。

规则以结构化配置 + 可执行 Validator 表达。自然语言说明可帮助模型解释，但不能替代 Validator。

## 6. Capability Composition

一个复合业务任务可由多个 Capability 组成：
- search project；
- get project；
- search learner；
- resolve learner；
- get status history；
- transition status；
- verify status。

固定、稳定的组合可定义 Workflow Capability；动态任务可由 Agent 规划，但每一步仍通过 Capability Registry。

## 7. Catalog Snapshot

Snapshot 保存所有实体、关系、Capability、Schema、权限引用、来源和审核状态。每次发布产生不可变版本。Run 固定 Snapshot ID，Invoke 检查 Binding 是否仍有效。

## 8. 语义搜索

支持中文 / 英文别名、同义词、缩写、历史名称、实体类别和动作词。搜索结果返回 why_matched，方便模型和调试人员理解命中原因。

## 9. 审核界面要求

管理员查看原始 API、源码位置、自动生成语义、关联 Entity、推荐 Capability、权限推断、风险级别和变更 Diff。写操作和权限定义必须人工审核。

## 10. 质量指标

Entity Link Precision、Capability Mapping Precision、Source Coverage、Manual Review Coverage、Business Rule Test Coverage、Stale Binding Count。
