# Evaluation Runner：数据集执行、评分与 Release Gate

## 本文档在系统中的作用

本文件将评测系统设计成独立运行时：从 Dataset 取样、创建隔离 Run、采集 Trajectory、执行 Evaluator、汇总指标、生成报告，并决定 Agent Release 是否允许发布。

## 1. Evaluation Runner 不复用生产身份

评测环境使用专用 Principal 和合成 Tenant Fixture。不能借用真实用户凭证，也不能默认对生产业务执行写操作。所有 Tool 默认 Mock / Sandbox；需要验证真实只读能力时使用只读凭证和脱敏数据。

## 2. Execution Pipeline

`Load Dataset Version → Resolve Agent Release → Create Evaluation Run → Prepare Fixtures → Execute Cases → Collect Trajectory → Run Evaluators → Aggregate Metrics → Compare Baseline → Gate Decision → Report`

## 3. Evaluator Contract

```python
from typing import Protocol

class Evaluator(Protocol):
    evaluator_key: str
    version: str

    async def evaluate(self, case, result, trajectory) -> "EvaluationResult":
        ...
```

EvaluationResult 至少包含 score、passed、reason_code、evidence_refs、details 和 evaluator_version。

## 4. Deterministic First

先运行确定性 Evaluator：Schema、Exact ID、Tool Call、Policy、Approval、Resource Version、引用存在性。只有确定性检查通过后，才运行 LLM Judge 评估语义质量和表达完整性。

## 5. LLM Judge 风险

Judge 与被评估模型可能共享偏差；Judge 对表达流畅的错误答案可能打高分；Judge 会受位置偏差和 Prompt 影响。因此需要人工标注校准、Pairwise Order Swap、重复评分和抽样复核。

## 6. Baseline 比较

候选 Release 与基线 Release 使用相同 Dataset、Fixtures 和预算。模型输出存在随机性时多次运行，报告均值、分位数和置信区间。不要只比较一次结果。

## 7. Gate Rule

Gate 是规则组合：
- Critical Security Failure = 0；
- Unauthorized Tool = 0；
- Approval Bypass = 0；
- Core Golden Cases 全部通过；
- Task Success 不低于基线阈值；
- Cost / Latency 在预算内；
- Checkpoint / Resume Tests 通过；
- Tool Schema / Catalog Binding 兼容。

任何硬性安全失败直接 Block，不允许由 LLM Judge 的高分抵消。

## 8. 报告结构

- Release / Dataset / Evaluator Version；
- Overall Pass Rate；
- Per Task Category；
- Tool Selection / Argument Metrics；
- Policy / Approval Metrics；
- Factual / Evidence Metrics；
- Cost / Latency；
- Failed Case List；
- Baseline Delta；
- Gate Decision；
- Human Review Notes。

## 9. 回归定位

每条失败关联 Prompt Version、Model Profile、Context Manifest、Catalog Snapshot、Tool Version、Graph Version 和 Policy Version。否则只能知道“分数下降”，无法定位原因。

## 10. 发布流程

Draft → Evaluation Queued → Running → Reported → Review Required / Passed / Blocked → Published。发布动作应由 Control Plane 完成，Evaluation Runner 不直接修改 Agent Definition。

## 11. 评测数据治理

Dataset Version 不可变；敏感数据脱敏；隐藏测试集权限受限；样本来源和授权明确；生产 Trajectory 进入 Dataset 前需脱敏和人工筛选。
