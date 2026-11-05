# Memory：评分、检索、冲突与治理算法

## 本文档在系统中的作用

把 Memory Engine 的关键算法落实为可测量流程：什么值得记、如何评分、如何检索、如何解决冲突、何时过期以及删除如何完成。

## 1. Candidate Scoring

候选评分可由多个信号组成：
- explicitness：是否明确要求记住；
- durability：是否长期稳定；
- reuse_probability：未来复用概率；
- evidence_strength：来源证据强度；
- confidence：提取可信度；
- sensitivity：敏感程度；
- volatility：事实变化频率。

一个可用的起始分数：
`remember_score = 0.22 explicitness + 0.18 durability + 0.18 reuse + 0.18 evidence + 0.14 confidence - 0.10 volatility`

这只是实验初值。敏感内容应由硬性规则决定拒绝或人工审核，不应仅通过扣分处理。

## 2. Retrieval Pipeline

`Scope Filter → ACL Filter → Candidate Recall → Freshness Filter → Conflict Check → Rerank → Context Candidate`

先做 Scope 和 ACL，不能把无权访问的向量结果先返回给模型再要求模型忽略。

## 3. 召回组合

- Exact Subject / Predicate；
- PostgreSQL Full Text；
- Vector Similarity；
- Entity Link；
- Recency；
- Importance；
- User Correction Weight。

多个召回通道去重后统一排序，保留每条 Memory 的 Evidence Ref。

## 4. 冲突检测

同一 Subject + Predicate + Scope 且有效时间重叠时进入冲突检测。按来源优先级、显式纠正、时间、置信度和事实类型决定是否 supersede。对于不同场景条件的偏好，使用 Context Condition 表达，而不是简单判冲突。

## 5. Merge 不等于覆盖

合并应生成新版本并记录 `supersedes_memory_id`，保留来源证据。不可变版本可用于解释过去为什么召回了旧信息。Active Pointer 指向当前有效版本。

## 6. Forgetting / Deletion

删除首先写 Tombstone 并从 Recall 结果排除；然后异步清理向量、缓存、摘要和对象引用；最后按保留政策处理原始来源。删除任务需要状态机和可验证完成条件。

## 7. Memory Quality Feedback

用户接受、纠正、忽略或反复重述可作为反馈信号，但不能直接把“没有再次提及”当成错误。Recall 结果应记录被选中、被注入、被引用和被纠正的次数。

## 8. 指标

Candidate Precision、Promotion Rate、Recall Precision@K、Memory Usefulness、Conflict Resolution Accuracy、Stale Memory Rate、Sensitive Candidate Rejection Rate、Delete Propagation Completion、User Correction Rate。

## 9. 测试样本

“我偏好简短回答”与“这份报告要详细”；“之前项目延期”与“现在项目已完成”；“请记住 API Key”；“忘记之前的偏好”；两个租户同名用户；已删除 Memory 仍残留在向量索引。
