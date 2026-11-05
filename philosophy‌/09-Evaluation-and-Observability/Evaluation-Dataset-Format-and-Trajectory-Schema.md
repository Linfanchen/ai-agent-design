# Evaluation Dataset 格式与 Trajectory Schema

## 本文档在系统中的作用

统一评测样本和执行轨迹格式，让不同 Agent Release、模型、Prompt、Tool 和 Context Compiler 能在同一数据集上做可重复比较。

## 1. Dataset Case

每个 Case 建议包含：
- case_id；
- dataset_key / version；
- task_category；
- input；
- principal_fixture；
- tenant_fixture；
- business_fixture_refs；
- allowed_capabilities；
- expected_outcome；
- expected_constraints；
- evaluator_set；
- tags；
- sensitivity；
- source / author。

## 2. JSON 示例

```json
{
  "case_id": "mentorship-status-001",
  "task_category": "business_read_write",
  "input": "查询项目 A 中学员小李的状态；如果满足条件，将状态更新为已完成。",
  "principal_fixture": "mentor_manager",
  "tenant_fixture": "tenant_alpha",
  "business_fixture_refs": ["project_A", "learner_li"],
  "allowed_capabilities": [
    "mentorship.project.search",
    "mentorship.learner.search",
    "mentorship.learner.get",
    "mentorship.learner.status.transition"
  ],
  "expected_constraints": {
    "must_disambiguate": true,
    "must_read_current_status": true,
    "must_check_transition_rule": true,
    "must_verify_after_write": true
  },
  "evaluator_set": [
    "tool_selection",
    "argument_schema",
    "approval_policy",
    "business_postcondition",
    "evidence_grounding"
  ]
}
```

## 3. Trajectory Event

Trajectory 不是完整复制模型内部思维。它记录可观察的执行事实：
- step_index；
- node_key；
- model_request_ref；
- model_response_ref（受控）；
- tool_call_intent；
- tool_execution_id；
- policy_decision；
- observation_ref；
- checkpoint_ref；
- token_usage；
- latency；
- error_code。

不保存隐藏推理链。保存必要的模型输出、Tool Call 和执行结果时，必须遵循隐私与脱敏策略。

## 4. Expected Outcome

Expected Outcome 可以是：
- Exact：稳定 ID、状态和结构；
- Set：允许的候选集合；
- Constraint：必须满足的安全和业务约束；
- Rubric：开放式回答质量评分；
- Evidence：必须引用的来源。

复杂任务通常组合多种判定方式。

## 5. Evaluator Mapping

每个 Case 显式绑定 Evaluator Version，避免同一个 Dataset 在不同运行中悄悄使用不同评分规则。

## 6. Dataset 版本

Dataset Version 不可变。修订输入、期望结果或标签都应创建新版本。每次 Evaluation Run 固定 Dataset、Agent Release、Model Profile、Tool Fixtures 和 Evaluator Set。

## 7. 失败分析标签

Tool Miss、Wrong Arguments、Missing Clarification、Unauthorized Attempt、Approval Bypass、Stale Data、Unsupported Claim、Citation Failure、No Progress、Budget Overrun、Recovery Failure、Poor UX。

## 8. 数据质量

样本必须经过可重复性检查；期望结果要有来源；对非确定任务允许评分区间；隐藏集控制访问；真实生产样本必须脱敏和获得合适使用授权。
