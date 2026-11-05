# Memory 提取、评分、冲突、合并与遗忘

## 本文档在系统中的作用

深入记忆治理流程：模型如何提出候选记忆、系统如何验证来源、如何处理矛盾、如何更新旧记忆，以及用户纠正或删除时如何清理。

## 1. 提取时机

在 Conversation 结束、Run 完成、用户明确说“记住”、用户纠正已有记忆、项目关键决策完成时异步提取。不要在每个 Token Delta 后提取。提取任务通过 Outbox / Job 执行。

## 2. Candidate Extraction

LLM 只负责提出候选，输出结构：
- fact；
- memory_type；
- suggested_scope；
- source_message_refs；
- confidence；
- sensitivity；
- reason_to_remember；
- expires_at；
- conflict_candidates。

候选不能直接写入 Active Memory。

## 3. 评分

评分维度：Explicitness（用户是否明确表达）、Reusability（未来是否会用到）、Stability（是否长期稳定）、Evidence（是否有明确来源）、Confidence（是否存在歧义）、Sensitivity（是否敏感）、Freshness（是否仍有效）。

敏感性不是普通扣分项。达到敏感阈值时进入拒绝存储或人工审核分支。

## 4. 冲突检测

比较同一 Subject + Predicate、同一 Scope、有效时间重叠、来源可信度、用户是否明确纠正，以及是否可并存于不同场景。例如“默认用中文”和“给海外客户写英文”不是冲突，而是不同 Context 条件。

## 5. 合并策略

- Exact Duplicate：合并来源引用和证据；
- Compatible Facts：合并为更完整记忆；
- Superseding Fact：新事实创建新版本并标记旧版本 superseded；
- Uncertain Conflict：保留并标记 conflict，必要时追问；
- Explicit Correction：优先采用用户纠正，保留纠正来源。

不应静默覆盖旧记忆。所有版本变化要可追溯。

## 6. 遗忘与删除

1. 定位 Memory Item；
2. 写入 Tombstone；
3. 阻止新 Recall；
4. 删除或脱敏向量索引；
5. 删除派生摘要和缓存；
6. 清理对象存储引用；
7. 记录删除任务状态；
8. 在保留期允许时清理原始来源；
9. 生成可验证删除结果。

如果原始对话或外部文件仍保留，不能声称所有来源都已删除。必须明确删除范围。

## 7. 自动过期

Memory 可配置 TTL；临时偏好和项目状态更容易过期。长期稳定偏好可以更长，但仍应允许用户纠正。过期记忆默认不注入上下文，除非作为历史信息展示。

## 8. 反馈闭环

用户纠正 → Memory Candidate → Source Link → Conflict Resolution → New Version → Recall Test。用户说“不是这样”应形成负反馈样本，但不能自动将所有否定内容都记成反向事实。

## 9. 测试数据集

覆盖同义表达、否定、条件偏好、时间变化、同名实体、跨租户相似内容、敏感信息、删除请求、用户纠正和来源缺失。
