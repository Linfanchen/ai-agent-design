# Memory Engine：记忆架构与数据模型

## 本文档在系统中的作用

Memory Engine 负责把跨轮次、跨会话可复用的信息提取为受治理记忆，并在后续任务中按需召回。它解决连续性问题，但不替代业务数据库、事件账本或完整聊天历史。

## 1. Memory 类型

- Profile Memory：稳定用户偏好、语言、输出风格。
- Episodic Memory：某次任务的关键经历、决定和结果。
- Semantic Memory：长期事实、项目知识、实体关系。
- Procedural Memory：经验证的工作方法和操作流程。
- Working Memory：当前 Run 的临时状态，优先放 Graph State，不混入长期 Memory。

## 2. Scope

User、Tenant、Project、Agent、Conversation、Global Approved。每条 Memory 必须有明确 Owner、Scope 和 ACL。跨 Scope 共享通过显式 Grant，不能仅凭向量相似度访问。

## 3. 生命周期

`Candidate → Extracted → Validated → Scored → Approved / Auto-Promoted → Active → Superseded / Expired / Tombstoned`

低风险偏好可按策略自动晋升；敏感信息、组织事实和影响业务决策的记忆应人工审核或要求明确来源。

## 4. Memory Item Schema

memory_id、tenant_id、owner_type / owner_id、scope_type / scope_id、memory_type、subject / predicate / object、natural_language_text、source_refs、confidence、importance、sensitivity、valid_from / valid_until、status、version、supersedes_memory_id、created_by、approved_by。

## 5. 不应存储的内容

密码、API Key、访问 Token；未经允许的敏感个人信息；没有来源的模型猜测；临时一次性参数；权威业务状态的替代副本；过期且无价值信息；需要删除但无法定位来源的原始内容。

## 6. 记忆检索

先做权限过滤，再做候选召回。检索组合 Exact / Keyword、Semantic Vector、Entity / Relation、Recency、Importance、Scope Match。

评分示意：
`score = 0.28 semantic + 0.18 lexical + 0.18 scope + 0.14 freshness + 0.12 importance + 0.10 confidence`。

权重只是起始假设，应使用任务集调优。Tenant / ACL 过滤是硬条件，不进入加权评分。

## 7. Memory 与 Context 的关系

Memory Service 返回 `MemoryCandidate`，Context Compiler 决定是否注入。Memory 不直接写入 System Prompt。被召回 Memory 仍标记来源、置信度和有效期。

## 8. Memory 与业务数据的关系

学员状态、项目状态、订单金额、审批状态属于业务事实，应实时读取权威业务服务。Memory 可以记住“用户经常关注某个项目”，不能用记忆中的旧状态回答“学员当前状态是什么”。

## 9. 存储建议

PostgreSQL 保存元数据、版本、Owner、ACL、来源和状态；向量字段可用 pgvector 或独立向量库；原始证据引用对象存储或受控消息表。向量索引不能成为权限系统。

## 10. 评估

Memory Precision、Recall Utility、Stale Memory Rate、Conflict Rate、User Correction Rate、Sensitive Memory Leakage、Delete Completion Rate。

## 11. Memory Write 与 Recall 分离

Memory Write 是后台治理流程；Memory Recall 是 Run 中的低延迟读取流程。不要在模型每次调用时同步执行完整 Memory Extraction。

### Write Path
`Conversation / Run Event → Candidate Extractor → Evidence Resolver → Sensitivity Gate → Conflict Detector → Scoring → Approval / Auto-Promotion → Store → Vector Index`

### Recall Path
`Principal Scope → ACL Filter → Query Normalize → Hybrid Retrieval → Freshness / Status Filter → Rerank → Context Candidate`

两条路径应有独立的延迟预算、队列和失败处理。

## 12. Memory 事实结构与自然语言摘要

同一条记忆可保存结构化三元组和自然语言摘要。例如：
- subject：user_123；
- predicate：preferred_output_style；
- object：concise；
- condition：general_answers；
- valid_from：时间；
- source_ref：用户消息 ID。

自然语言摘要用于模型理解；结构化字段用于冲突检测、过滤和条件匹配。

## 13. 记忆版本与时间有效性

建议使用双时间概念：
- Valid Time：事实在现实世界中何时有效；
- Transaction Time：系统何时记录这条事实。

这样可以表达“用户在 10 月纠正了 8 月以来的偏好”，也能追踪系统何时知道这件事。旧版本标记 Superseded，不直接覆盖。

## 14. Scope 继承规则

User Memory 不自动变成 Tenant Memory；Project Memory 不自动对所有成员可见；Agent Memory 不自动共享给其他 Agent。跨 Scope 使用必须有显式 Share Grant，并在 Recall 时再次检查。

## 15. Memory 与用户体验

系统应允许用户查看、纠正和要求不再使用某条记忆。对低置信度记忆可在召回时以“我可能记得……”方式确认；对用户明确纠正，应优先更新版本而不是争辩。

## 16. 删除一致性

Memory Tombstone 写入后，Recall 查询立即过滤。向量索引、缓存、摘要和派生结果异步清理。删除任务状态至少包含 pending、processing、partially_completed、completed、failed。失败项必须可重试和审计。

## 17. 记忆质量门槛

上线前用专门 Memory Dataset 验证：提取精确率、错误记忆率、召回相关性、冲突处理、过期过滤、敏感信息拒绝、用户纠正和删除传播。不要只以“记忆数量增加”作为指标。
