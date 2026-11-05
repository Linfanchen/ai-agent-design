# Evaluation Dataset、Trajectory 与 Golden Cases

## 本文档在系统中的作用

Evaluation Dataset 定义 Agent 应完成的任务和预期行为；Trajectory 记录一次真实执行的步骤；Golden Cases 用于回归验证。三者将 Agent 质量从主观体验转为可重复测量。

## 1. Dataset 结构

每条 Case 包含 case_id、task、tenant_fixture、principal_fixture、agent_release、expected_capabilities、forbidden_capabilities、expected_output_schema、expected_facts、required_evidence、approval_expectation、max_steps、max_cost、evaluator_config、tags。

数据集区分公开、内部、敏感和合成数据。生产数据用于评测前必须脱敏并符合使用政策。

## 2. Trajectory

Trajectory 是执行过程的结构化记录：
- run_id；
- step_index；
- node_name；
- model_profile；
- prompt_version；
- context_refs；
- tool_calls；
- tool_outcomes；
- policy_decisions；
- approval_events；
- state_transition；
- token_usage；
- latency；
- error；
- result_ref。

Trajectory 不等于原始思维链。记录可审计的行动、观察和决策摘要，不默认保存或暴露模型隐藏推理。

## 3. Golden Case 类型

Tool Selection、Entity Resolution、Multi-step Planning、Business Read、Business Write、Approval、Clarification、Memory Recall、Context Conflict、Prompt Injection、Failure Recovery、Multi-Agent Delegation。

## 4. 版本绑定

每次评测固定 Dataset Version、Agent Release、Model Profile、Catalog Snapshot、Evaluator Version，否则不同时间结果不可比较。

## 5. 数据泄漏

评测集不能与 Prompt / Skill 训练示例完全重合。需要 Train / Dev / Test 隔离，并保留隐藏测试集。评测样本更新要有版本和来源记录。

## 6. 评测样本的分层

- Smoke Set：每次提交都运行，少量核心用例；
- Golden Set：发布前运行，覆盖关键业务路径；
- Security Set：Prompt Injection、越权、审批和租户隔离；
- Stress Set：长任务、并行 Tool、多 Agent 和超长 Context；
- Hidden Set：避免 Prompt / Skill 针对公开样本过拟合；
- Production Replay Set：脱敏后的真实失败样本。

## 7. Trajectory 脱敏

Trajectory 可能包含用户输入、Tool 参数和业务实体 ID。入库前按字段分类处理；Evaluation 使用合成 ID 或映射后的 Fixture。禁止将原始敏感 Tool Result 无差别导入评测集。

## 8. 失败样本回流

生产失败 Run → 运营 / 工程分类 → 脱敏 → 根因标签 → 新增 Dataset Case → Candidate Release Regression。不要把每个生产失败都自动变成 Golden Case，必须先确认它代表稳定需求而不是一次性异常。
