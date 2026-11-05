# Evaluator、LLM-as-Judge 与指标设计

## 本文档在系统中的作用

Evaluator 判断 Agent 输出和执行过程是否符合任务要求。LLM-as-Judge 可以评估语义质量，但必须与确定性检查结合，不能让另一个 LLM 成为唯一质量裁判。

## 1. Evaluator 类型

- Exact Match：确定字段和 ID；
- Schema Evaluator：Pydantic / JSON Schema；
- Rule Evaluator：业务规则和状态转换；
- Tool Evaluator：工具选择和参数；
- Evidence Evaluator：引用是否支持结论；
- Human Evaluator：人工评分；
- LLM Judge：语义、完整性、表达质量；
- Pairwise Evaluator：比较两个版本。

## 2. 评测维度

Task Success、Factual Correctness、Tool Selection Accuracy、Argument Accuracy、Policy Compliance、Approval Compliance、Evidence Grounding、Completeness、Consistency、Latency、Cost、User Acceptance。

## 3. LLM Judge 设计

Judge Prompt 包含评分 Rubric、正反例、评分范围和证据要求。要求 Judge 引用被评估输出中的证据，不允许只凭“感觉不错”打分。

## 4. Judge 校准

与人工标注样本对比；测量 Judge-Human Agreement；检查不同语言和任务类别偏差；交换候选答案顺序做 Position Bias Test；使用多个 Judge 或抽样人工复核；记录 Judge Model / Prompt Version。

## 5. 硬性门禁

跨租户数据泄漏、未授权 Tool Execution、未审批高风险操作、幂等重复副作用、违反数据保留策略、Schema 不合法不能由 LLM Judge 放行。这些由确定性测试和 Policy Test 负责。

## 6. 指标聚合

按 Agent Release、Model Profile、Task Type、Tenant Segment、Tool Version 分组，避免总体平均掩盖某一类任务严重退化。

## 7. 评分 Rubric 的结构

每个 Judge Rubric 应包含：
- 评测目标；
- 可观察证据；
- 评分维度；
- 每个分数的定义；
- 严重错误的扣分 / Fail 规则；
- 不确定时如何标记；
- 禁止推断的内容；
- 正例和反例。

Judge 输出使用结构化 JSON，包含 `score`、`passed`、`evidence_quotes`、`missing_requirements`、`reasoning_summary`。不要要求 Judge 输出隐藏推理链。

## 8. 指标分层

- Outcome：任务是否成功；
- Process：步骤、Tool 选择和参数；
- Safety：权限、审批、隐私；
- Evidence：事实是否有来源；
- Experience：用户是否理解和接受；
- Efficiency：成本、延迟、Token 和重试。

不能用 Outcome 的高分掩盖 Safety 的失败。

## 9. Judge 不一致处理

若多个 Judge 分歧较大，或 Judge 与确定性 Evaluator 冲突，进入人工复核。Judge 版本变化要重新评估历史基线，否则评分变化可能来自 Judge 本身而不是 Agent。
