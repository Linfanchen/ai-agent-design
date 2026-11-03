# 22. 实施路线与验收标准

## Role in the Agent System

Turns the architecture into phases, deliverables, acceptance criteria, and dependencies.

**Primary responsibility:** Own this capability as a reusable subsystem with explicit contracts, state ownership, security boundaries, and observable outcomes.

**Boundary:** This subsystem must not silently take over responsibilities owned by adjacent layers. Its public interface should be testable without running a full Agent conversation.

## 1. 建设原则

先做一个可验证的企业业务闭环，再扩展为通用平台。不要一开始实现所有插件市场、多 Agent 自治和复杂记忆学习。

## 2. Phase 0：技术基线

交付：Node.js / TypeScript 工程、LangGraph JS/TS、Provider Adapter、PostgreSQL、Redis、Migration、Trace / Structured Logging、CI / Unit Test。

验收：
- 至少两家 Provider 可切换。
- 同一业务任务不依赖 Provider 专属结构。
- 错误和 Token Usage 可标准化。
- 运行状态可持久化。

## 3. Phase 1：单 Agent Harness

交付：Agent Registry、Runtime Factory、Session / Turn / Step、Checkpoint、Tool Registry、Tool Executor、Context Builder、Cancellation / Retry。

验收：
- 多轮 Tool Calling 正确。
- 中断后恢复。
- 同一 Session 并发控制。
- Tool Result 先持久化后继续模型调用。
- 失败可定位到 Step / Tool Execution。

## 4. Phase 2：Business Catalog + Tool Discovery

交付：Frontend / Backend / DDL Scanner、API Catalog、Ontology、Capability Registry、Dynamic Tool Search / Describe / Call、Generic Query Engine、Domain Command Registry。

验收：
- 可检索系统全部已扫描 API。
- 结果可追溯源文件和版本。
- 只暴露当前 Scope 允许工具。
- Tool Search Precision@K 达到项目设定目标。
- 查询不允许任意 SQL。
- 写操作经过 Java Domain Service。

## 5. Phase 3：企业 Memory + Skills

交付：Raw Event / Candidate / Canonical Memory、Recall、Entity / Episode、Skill Registry / Loader、Skill Evaluation、Async Consolidation。

验收：
- 明确用户、项目、组织、Agent Scope。
- 记忆有来源、可信度和生命周期。
- 用户纠正可以更新旧记忆。
- Skill 只按需加载。
- 未审核 Skill 不进入生产默认集合。

## 6. Phase 4：Governance + Approval

交付：Identity / Tenant Context、RBAC / ABAC、Tool Policy、Approval、Audit、Idempotency、Read-after-write Verification。

验收：
- 跨租户访问测试全部拒绝。
- 未授权写操作全部拒绝。
- 重复请求不会重复产生业务副作用。
- Approval 与具体参数/资源版本绑定。

## 7. Phase 5：Multi-Agent + Plugin

交付：Supervisor、Subagent Provider、Delegation Contract、Plugin Manifest、Plugin Lifecycle、Scope / Event / Disposer。

验收：
- 子 Agent 上下文隔离。
- 子 Agent 权限不超过父任务授权。
- Plugin 可卸载且无残留注册。
- 子任务可取消、超时和恢复。

## 8. Phase 6：自我改进与运营

交付：Trajectory Store、Evaluation、Regression Dataset、Skill Candidate Generator、Catalog Diff / Impact Analysis、Model Routing Optimization。

验收：
- 历史任务可回放（默认不重放外部写操作）。
- 失败能分类。
- 任何生产改进先经过测试和审核。
- 模型升级有对比报告。

## 9. 首个落地场景

企业导师带教：
1. 查询带教项目。
2. 查询项目学员。
3. 查询某学员的导师、状态、计划和任务进度。
4. 判断是否满足状态变更条件。
5. 生成待确认操作。
6. 用户确认。
7. 调用 Java Domain Command。
8. 再次查询验证。
9. 记录审计和 Trajectory。

这是一个足以验证 Runtime、Context、Tools、Catalog、Approval、Checkpoint 的端到端用例。
