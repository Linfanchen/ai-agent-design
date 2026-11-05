# Tool Search：召回、排序与评测算法

## 本文档在系统中的作用

Tool Search 决定在大规模能力目录中找出与当前任务最相关、当前用户有权使用、当前环境可执行的能力。它是 Dynamic Tool Discovery 的核心算法模块。

## 1. 先过滤，再排序

硬过滤条件：Tool Published；Agent Release Binding 存在；Tenant 已启用；Principal 当前有权；Tool Version 未撤销；Handler 健康；当前执行模式允许其风险等级。

禁止先召回所有工具，再只靠 LLM 判断哪些可以调用。

## 2. 四路召回

1. 精确匹配：tool_key、alias、operationId。
2. 关键词检索：PostgreSQL FTS / BM25。
3. 向量检索：语义相似。
4. Ontology / Rule：Domain、Entity、Relationship、Operation、任务路由规则。

多路结果按 Tool Version ID 去重，保留命中原因。

## 3. Rerank 特征

- lexical relevance；
- semantic relevance；
- intent match；
- entity match；
- operation match；
- schema fit；
- freshness；
- historical success。

可从线性加权开始，再用标注数据优化。历史成功率不能改变权限和风险硬过滤。

## 4. Search Result 最小化

返回 Tool Key、Version、Short Description、Domain、Operation、Risk Level、Side Effect Class、Why Matched 和 Describe Ref。不要在 Search 阶段返回所有复杂 Schema。

## 5. Describe

加载完整 Input / Output Schema、必填字段、枚举、约束、前置条件、权限、风险、审批、幂等、超时、错误码和示例。

## 6. 评测集

每条用例标注用户任务、正确 Capability、可接受候选、错误候选、Principal、Tenant、Domain、Entity、读写属性和预期澄清问题。

## 7. 指标

Recall@K、Precision@K、MRR、NDCG、Tool Selection Accuracy、Schema Load Rate、Unauthorized Candidate Exposure、Tool Call Success。

## 8. 动态发现策略

工具少且稳定：直接提供有限工具集。
工具中等：Tool Groups + 任务路由。
工具多且权限差异大：Search → Describe → Invoke。
外部 MCP：先缓存元数据，再按需加载。
原始业务 API：先映射 Capability，不直接暴露。

## 9. 降级

向量检索不可用时退化到 FTS / Exact；Catalog Snapshot 过期时阻止高风险写操作；Policy 不可用时拒绝写操作；Tool Binding 不存在时返回 `TOOL_BINDING_UNAVAILABLE`。
