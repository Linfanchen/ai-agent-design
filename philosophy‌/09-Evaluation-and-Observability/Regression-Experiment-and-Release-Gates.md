# 回归评测、实验与发布门禁

## 本文档在系统中的作用

定义 Prompt、Model、Tool、Skill、Graph、Context 和 Policy 发生变化时，如何判断新版本是否优于旧版本，以及如何阻止质量或安全回归进入生产。

## 1. 变更类型

Prompt / System Instruction；Model Provider / Model Profile；Tool Schema / Handler；Skill；Plugin；Graph Node / Edge；Context Ranking / Compression；Memory Retrieval；Catalog Snapshot；Policy；Dataset / Evaluator。

## 2. 回归流程

`Baseline → Candidate → Same Dataset → Deterministic Checks → LLM Judge → Human Sample → Statistical Comparison → Release Decision`

## 3. 比较方式

同一任务分别运行旧 Release 和新 Release，控制 Dataset、输入、工具 Fixture 和随机性。模型非确定性需要多次运行或统计区间，不能只比较单次结果。

## 4. 门禁示例

安全关键指标不得退化；Tool Selection Accuracy 不低于阈值；Task Success 不出现显著下降；平均成本不超过预算；P95 延迟不超过目标；关键 Golden Cases 全部通过；新增 Tool 无未审核权限；State Migration 能恢复旧 Checkpoint。阈值由实际基线和业务风险确定。

## 5. Canary

先对内部用户或少量流量启用新 Release，监控失败率、UNKNOWN Tool、审批异常、成本、延迟和用户纠正。发生安全问题立即撤销 Tool / Release。

## 6. 回滚

回滚不是删除新版本，而是重新指向上一稳定 Release。若数据库 Schema 已迁移，确保旧 Runtime 仍能读取或有兼容层。

## 7. 实验设计注意事项

- 固定 Dataset Version；
- 固定 Tool Fixture 和业务数据快照；
- 固定 Agent Release 之外的其他条件；
- 对模型非确定性进行多次采样；
- 区分模型波动和版本变化；
- 报告样本数和置信区间；
- 不把不同难度任务简单混合成一个平均分。

## 8. Canary 监控维度

除了总体成功率，还要监控 Tool 选择分布变化、平均步骤数、UNKNOWN 执行、审批率、拒绝率、Token / Cost、P95 延迟、用户纠正率和人工接管率。

## 9. 立即回滚条件

出现跨租户泄漏、审批绕过、未授权副作用、密钥泄漏、不可逆重复操作、Sandbox 逃逸时立即停止候选 Release，不等待统计显著性。
