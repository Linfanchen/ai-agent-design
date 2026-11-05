# Dynamic Tool Discovery：搜索、描述与按需装载

## 本文档在系统中的作用

Dynamic Tool Discovery 让模型在庞大能力目录中按需发现工具，而不是每轮携带所有 Tool Schema。它解决 Token 膨胀、工具选择歧义和业务系统 API 数量过多的问题。

## 1. 三阶段协议

### Search
输入自然语言意图、Domain、Entity、Operation 和当前 Principal Scope。输出少量候选 Tool 的 ID、描述、风险、操作类型和相关度。

### Describe
模型选中候选后，系统加载完整 Input / Output Schema、必填字段、枚举、示例、权限、审批和副作用说明。

### Invoke
模型生成参数；系统校验 Schema、权限、Policy、审批、预算、幂等、版本和执行状态后才调用 Handler。

## 2. Search 必须先做硬过滤

Tool Published；Tenant 已启用；Agent Release 已绑定；Principal 有权使用；当前 Runtime Profile 允许；Tool Version 未撤销；Handler 健康；Tool 风险在当前执行模式允许范围内。

不能先把所有工具召回给模型，再依赖模型判断是否越权。

## 3. 混合召回

- Exact：tool_key、alias、operationId；
- Lexical：PostgreSQL FTS / BM25；
- Vector：语义相似度；
- Ontology：Domain / Entity / Relation / Operation；
- Rule：任务类型到 Capability 的规则路由。

## 4. Rerank 特征

`relevance = lexical + semantic + intent_match + entity_match + operation_match + schema_fit + freshness + historical_success`

权重用标注数据集调优。历史成功率只作为排序信号，不能绕过安全策略。

## 5. Search Result 最小化

每个候选仅返回 tool_key、version、short_description、domain、operation、risk_level、side_effect_class、why_matched、describe_ref。完整 Schema 不在第一轮返回。

## 6. Describe 内容

Input Schema、Output Schema、Required / Optional、Enum、Field Description、Examples、Preconditions、Side Effects、Required Permissions、Approval Requirement、Idempotency、Timeout、Data Freshness、Error Codes。

## 7. Search 与 Invoke 的竞态

Search 后 Tool 可能被撤销、权限可能变化、版本可能升级。Invoke 时重新校验所有安全条件。Search 的授权结果不能作为稍后执行的永久凭证。

## 8. 无结果处理

Search 没有合适工具时返回 `NO_CAPABILITY_MATCH`，提供澄清或说明缺少能力；不允许模型临时拼接 SQL / HTTP 请求绕过 Registry；可向管理员提交 Capability Gap Request。

## 9. 评测

Recall@K、Precision@K、MRR、NDCG、Tool Selection Accuracy、Schema Load Rate、Unauthorized Candidate Exposure（必须为 0）、Tool Success Rate。

## 10. 何时不需要 Dynamic Discovery

若 Agent 只有 3–8 个固定工具，直接绑定有限 Toolset 更简单。工具规模大、按业务域变化、权限差异大时才需要 Search / Describe / Invoke。
