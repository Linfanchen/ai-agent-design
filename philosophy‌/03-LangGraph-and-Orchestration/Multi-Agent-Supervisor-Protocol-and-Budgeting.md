# Multi-Agent Supervisor：协议、调度、预算与结果汇总

## 本文档在系统中的作用

本文件深入定义 Supervisor 与 Worker Agent 的协作协议，解决任务如何拆分、子 Agent 如何获得有限上下文和权限、如何调度并发、如何合并结果，以及失败时如何继续或终止。

## 1. Supervisor 不等于“聊天转发器”

Supervisor 的职责：
- 理解总目标；
- 选择可用 Worker Agent；
- 将任务拆分成独立子目标；
- 为每个子目标定义输入、输出 Schema 和预算；
- 授予最小权限；
- 监控状态和 deadline；
- 合并证据；
- 处理冲突和部分失败；
- 验证最终结果。

Worker Agent 只负责被委派的任务，不拥有全局任务决策权。

## 2. Delegation Contract

```python
from pydantic import BaseModel, Field
from typing import Literal

class DelegationContract(BaseModel):
    delegation_id: str
    parent_run_id: str
    child_agent_key: str
    objective: str
    input_refs: list[str] = Field(default_factory=list)
    output_schema_ref: str
    capability_grants: list[str] = Field(default_factory=list)
    max_steps: int = Field(default=8, ge=1, le=30)
    max_model_calls: int = Field(default=10, ge=1, le=50)
    max_cost_usd: float = Field(default=0.25, gt=0)
    deadline_seconds: int = Field(default=120, ge=5, le=1800)
    failure_policy: Literal["required", "optional", "best_effort"]
```

## 3. Fan-out 规划

拆分前检查任务是否真正独立。若子任务 B 需要子任务 A 的结果，不能同时 Fan-out。对独立任务并行执行；依赖任务按 DAG 层级调度。

## 4. 预算分配

Parent Budget = Supervisor Budget + Child Reservations + Aggregation Reserve。子任务开始前原子预留 Token / Cost / Runtime。子任务未执行部分在完成或取消时释放。禁止 Child 自动要求无限追加预算。

## 5. 权限 Grant

Grant 包含 tenant、resource scope、actions、expires_at、parent delegation、policy version。子 Agent 不继承父 Agent 全部 Tool Binding。每次 Tool Invoke 仍由 Policy Engine 检查。

## 6. 消息协议

Parent → Child：DelegationRequest。
Child → Parent：TaskAccepted / Progress / ClarificationRequired / Completed / Failed / Cancelled。
所有消息有 message_id、delegation_id、run_id、sequence、schema_version。不要只靠自由文本识别子任务状态。

## 7. 结果聚合

汇总前验证每个结果的 Schema、证据来源、时间、Agent Release 和权限范围。对于互相冲突的结果，保留冲突并请求更多证据；不能简单按最后返回的 Agent 覆盖。

## 8. 取消传播

Parent Cancel 时向所有 Child 发出取消请求。已完成子任务保留结果；执行中子任务停止新 Tool；外部副作用仍需对账。取消失败要记录并持续监控。

## 9. 失败策略

Required Child 失败则任务失败或人工介入；Optional Child 失败可继续并标记缺失；Best Effort 允许部分成功但必须披露缺失。不要把所有子任务异常吞掉。

## 10. 评测

比较 Supervisor 拆分正确率、重复子任务率、权限越界率、预算超支率、结果冲突率、部分成功准确性和相对单 Agent 的质量 / 成本收益。
