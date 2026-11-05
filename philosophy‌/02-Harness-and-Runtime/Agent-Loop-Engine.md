# Agent Loop：完整推理与行动循环

## 本文档在系统中的作用

Agent Loop 控制模型何时调用工具、观察结果、继续或终止。它是 Agent 行为执行循环，但不承担业务权限、数据持久化和资源装配等所有职责。

## 1. 标准循环

```text
Receive Task → Build Context → Call Model → Parse Response
  → Is Final?
      ├─ Yes → Validate / Persist / Return
      └─ No
          → Tool Calls?
              ├─ No → Continue / Ask Clarification / Fail
              └─ Yes → Validate → Authorize → Execute → Observe
                         → Update State → Budget Check → Next Model Call
```

每一轮有 `step_index`、`model_call_id`、`tool_call_count`、`token_usage`、`elapsed_ms` 和 `state_version`。

## 2. Loop Termination 条件

模型返回 Final；Graph 到达 END；用户取消；Run Deadline；Max Steps；Max Model Calls；Max Tool Calls；Token / Cost Budget；安全策略终止；连续无进展检测触发；关键依赖不可恢复失败。终止原因保存为结构化 `termination_reason`。

## 3. 并行 Tool Call

模型一次返回多个 Tool Calls 时，先对每个调用独立做 Schema / Policy / Approval 判断。只有被批准且声明可并行的工具才能并发执行。

并行约束：
- 同一资源写操作默认串行；
- 无依赖的只读 Tool 可并行；
- 同一 SQLAlchemy AsyncSession 不得被多个 Task 共享；
- Tool Group 设置最大并发；
- 父 Run 取消时取消子 Task；
- 部分成功必须显式表达。

## 4. Retry 分类

| 错误 | 策略 |
|---|---|
| Provider 429 | 遵守 Retry-After，指数退避 + Jitter |
| Provider 5xx | 有限重试 |
| Invalid Request | 不重试 |
| Tool Schema Error | 让模型修正一次或返回参数错误 |
| Tool 业务校验失败 | 不盲目重试，提供可解释 Observation |
| Tool Timeout / Side Effect Unknown | 对账，不直接重放 |
| Policy Deny | 不重试，不向模型泄露敏感策略细节 |
| Checkpoint Failure | 暂停 / 失败，不能继续无持久化执行 |

## 5. 死循环与无进展检测

检测相同 Tool + 相同参数哈希重复调用、同一错误 Observation 连续重复、模型连续输出空内容、State Hash 多轮不变化、反复执行同一无效计划步骤和超过最大循环次数。不能仅依赖模型自己说“我完成了”。

## 6. Loop 伪代码

```python
async def run_agent(task, runtime):
    state = await runtime.state_store.load(task.run_id)
    while not state.is_terminal:
        runtime.budget.assert_available(state)
        context = await runtime.context.compile(state)
        response = await runtime.model.complete(context.request)
        parsed = runtime.response_parser.parse(response)

        if parsed.is_final:
            result = await runtime.output_validator.validate(parsed.output)
            await runtime.state_store.complete(task.run_id, result)
            return result

        if not parsed.tool_calls:
            state = await runtime.clarification.handle_or_fail(state, parsed)
            continue

        outcomes = await runtime.tool_executor.execute_batch(
            parsed.tool_calls, runtime.principal, runtime.scope
        )
        state = await runtime.state_store.append_observations(state, outcomes)
        await runtime.budget.record(response.usage, outcomes)
    return state.result
```

这是 Loop 概念伪实现。若使用 LangGraph，Loop 的状态迁移应建模为 Graph Nodes / Conditional Edges，不要在外层重复维护一套不一致状态机。

## 7. 必须持久化的事实

Model Request / Response Reference、Tool Call、Tool Execution、Tool Result Reference、Retry Attempt、Termination Reason、Usage、Approval Reference、Graph Checkpoint Reference。大体积 Prompt 和 Tool Result 可放对象存储或受控内容表，通过引用关联。

## 9. Model Response 分类

Loop 不应只检查 `tool_calls` 是否为空。统一 Response Parser 至少区分：
- `FINAL_TEXT`：最终文本；
- `FINAL_STRUCTURED`：结构化最终结果；
- `TOOL_REQUEST`：一个或多个工具调用；
- `NEED_CLARIFICATION`：缺少关键信息；
- `REFUSAL`：模型拒绝；
- `INCOMPLETE`：输出被截断或流中断；
- `PROVIDER_ERROR`：调用失败；
- `UNSUPPORTED_RESPONSE`：Adapter 无法映射的返回。

不同类别走不同路由。`INCOMPLETE` 不能被当作最终回答；`REFUSAL` 不应进入 Tool Dispatch。

## 10. Tool Call 生命周期

模型返回 Tool Call 后，先创建内部 `ToolCallIntent`，并将外部模型的 Tool Call ID 映射到内部 ID。随后校验工具是否属于当前 Agent Release，加载 Schema，校验参数，执行 Policy，再创建 Tool Execution。工具结果返回模型时必须保留对应 Tool Call ID 关联。

## 11. 流式响应与 Tool Call

流式模式下，参数可能以多个 Delta 片段返回。Adapter 必须按 Tool Call Index / ID 聚合完整 JSON，再做 Schema Validation。不能在 JSON 片段未完整时执行工具。若 Stream 在参数中途断开，应将 Tool Call 标记为 incomplete，不执行。

## 12. 最大轮数与停止条件

建议同时限制：
- max_graph_steps；
- max_model_calls；
- max_tool_calls；
- max_parallel_tools；
- max_delegation_depth；
- max_runtime_seconds；
- max_cost_usd；
- max_repeated_tool_fingerprint。

只限制 `max_iterations` 不够，因为一次迭代可能执行多个 Tool 或多个子 Agent。

## 13. 观察结果的可信度

Tool Result 不是模型指令。Result 需要结构化 status、data_ref、evidence_ref、freshness、source_system 和 error_code。Context Builder 在下一轮将其作为 Observation 注入，并标记来源可信度。

## 14. 输出验证

Final Output 至少通过：
1. Schema Validation；
2. Business Invariant Validation；
3. Evidence Validation；
4. Sensitive Field Filter；
5. Output Length / Format；
6. User-visible Error Sanitization。

模型说“操作成功”不等于业务操作成功。写操作必须有业务系统返回的 operation reference 或二次读取证据。

## 15. Agent Loop 与 Workflow 的边界

对于固定流程，使用 Graph / Workflow 的确定节点；仅在需要选择工具、澄清或规划时让 LLM 参与。避免把所有分支都交给模型，也避免把所有开放任务都硬编码为静态流程。
