# LangGraph Python：StateGraph 工程实现

## 本文档在系统中的作用

把 LangGraph Python 从概念介绍落实为可维护的 Graph Builder 设计，覆盖 State Schema、Node、Reducer、Conditional Edge、Runtime Context、Checkpointer、Interrupt、Subgraph、Streaming 和测试。

## 1. LangGraph 的定位

LangGraph 是有状态编排运行时，提供 Graph / Functional API、状态更新、节点调度、持久化、流式事件和中断恢复。它不替代应用层 Run / Job 状态、企业身份和权限、Tool Executor、业务事务、Memory 治理、审计和成本账本。

## 2. Graph API 还是 Functional API

- Graph API：流程节点和分支明确，适合企业审批、复杂路由、多 Agent 和可视化流程。
- Functional API：希望在现有 Python 函数流程中引入持久化、中断和流式能力时使用。
- 两者可以组合，但不能重复管理同一份状态。

V4 默认 StateGraph 作为主要业务 Agent 编排方式；纯线性任务可考虑 Functional API。

## 3. State Schema 原则

State 保存执行所需的可序列化数据，不保存 DB Session、HTTP Client、密钥、Provider 对象、Tool Handler 和完整 Principal。

```python
from typing import Annotated, Literal, TypedDict
from operator import add

class ToolOutcome(TypedDict):
    tool_call_id: str
    status: Literal["succeeded", "failed", "denied", "unknown"]
    result_ref: str | None
    error_code: str | None

class AgentState(TypedDict, total=False):
    state_schema_version: int
    request_text: str
    intent: str
    entity_refs: list[str]
    messages: list[dict]
    tool_outcomes: Annotated[list[ToolOutcome], add]
    step_count: int
    termination_reason: str | None
    final_result_ref: str | None
```

Reducer 不是“列表字段都加起来”。只有追加式事件列表适合 `operator.add`。普通字段通常使用覆盖语义；并行更新同一字段时需要确定性 reducer。

## 4. Runtime Context 与 State

可信身份和依赖通过运行时配置 / Context 注入 Node：tenant_id、principal_ref、permission snapshot、model provider、Tool Executor、Repository Factory、budget controller、trace context。State 可保存不可变引用，但不能保存凭证和可执行对象。

## 5. Graph Builder 示例

```python
from typing import TypedDict, Literal
from langgraph.graph import StateGraph, START, END

class State(TypedDict, total=False):
    user_input: str
    intent: str
    answer: str
    step_count: int

async def classify(state: State, config):
    # 生产实现调用注入的 Model Port；这里仅演示节点边界
    return {
        "intent": "lookup",
        "step_count": state.get("step_count", 0) + 1,
    }

async def answer(state: State, config):
    return {"answer": f"已识别任务类型：{state['intent']}"}

def route(state: State) -> Literal["answer", "end"]:
    if state.get("step_count", 0) > 10:
        return "end"
    return "answer"

builder = StateGraph(State)
builder.add_node("classify", classify)
builder.add_node("answer", answer)
builder.add_edge(START, "classify")
builder.add_conditional_edges(
    "classify",
    route,
    {"answer": "answer", "end": END},
)
builder.add_edge("answer", END)
graph = builder.compile()
```

Node 的 config / Runtime Context 注入方式应按锁定版本验证。不要把硬编码 intent 用作生产分类。

## 6. Node 设计

每个 Node 只做一个可观察步骤：`resolve_task`、`retrieve_context`、`plan_steps`、`select_tools`、`execute_tools`、`verify_result`、`compose_answer`。外部副作用通过幂等 Tool Executor 或 Application Use Case 执行。

## 7. Reducer 设计

- Append Reducer：事件、消息、Tool Outcome。
- Replace Reducer：当前 intent、当前 plan、当前结果。
- Set Union Reducer：去重 ID 集合；若顺序重要需稳定排序。
- Custom Merge Reducer：明确冲突规则。
- Parallel Branch：两个分支同时更新同一字段时，验证 reducer 是否可交换、可结合，避免结果依赖执行顺序。

## 8. Checkpointer 与 Thread

Checkpointer 保存 Graph 执行快照，通常以 thread_id 组织。业务 Thread ID 由应用生成并映射到 LangGraph configurable，不直接信任客户端输入。生产使用持久化 Checkpointer；测试可用内存实现。Checkpointer 表结构和迁移由对应官方实现管理。

## 9. Interrupt / Resume

在澄清或审批 Node 中发出 Interrupt。中断值应是 JSON 可序列化的最小展示数据，如 approval_id、summary、required_fields、expires_at。Resume 时重新鉴权并验证响应，不允许客户端提交任意 Graph State。

## 10. Subgraph

适合封装可复用子流程：实体消歧、文档分析、审批、检索、验证。Subgraph 应有明确输入输出 Schema。独立任务通常使用 per-invocation 隔离；只有确实需要跨多轮保留子 Agent 会话时才使用 per-thread 持久状态。

## 11. Streaming

LangGraph 支持 updates、values、messages、custom、checkpoints、tasks、debug 等 Stream Mode。应用层应将内部事件归一化为稳定 Event Schema，不要直接把所有 Graph Debug Event 暴露给终端用户。

## 12. 测试策略

Node 单测、Graph 分支测试、Checkpoint 恢复测试、State Migration 测试、并行 Reducer 测试、节点异常 / 超时 / 取消测试、configurable 伪造测试、Fake Provider 回放测试。

## 13. 关键注意事项

Checkpoint 是执行恢复机制，不是完整企业事件账本。外部副作用可能发生在 Checkpoint 保存之前或之后，因此 Tool Executor 必须具备幂等和对账。Graph 恢复不代表外部系统状态自动回滚。

## 14. Interrupt 的最小示例

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import interrupt, Command

class ApprovalState(TypedDict, total=False):
    action_summary: str
    approval_id: str
    approved: bool
    result: str

async def request_approval(state: ApprovalState):
    decision = interrupt({
        "type": "approval",
        "approval_id": state["approval_id"],
        "summary": state["action_summary"],
    })
    return {"approved": bool(decision.get("approved"))}

def approval_route(state: ApprovalState):
    return "execute" if state.get("approved") else "end"
```

此处展示 Interrupt 的节点形状，不代表完整审批实现。生产系统应在调用 Resume 前由 Approval Service 验证审批人、审批状态、intent_hash、资源版本和过期时间。不能仅信任 Resume Payload 中的 `approved=True`。

## 15. Checkpointer 配置边界

- 开发测试使用内存 Checkpointer；
- 生产使用持久化 Checkpointer；
- Checkpointer 连接池独立配置；
- Checkpoint 数据按 Tenant / Thread 映射；
- 保留和删除策略与 Thread 生命周期一致；
- 不将密钥和数据库 Session 放入 State；
- Checkpoint 写失败时不继续执行依赖持久化的高风险步骤。

LangGraph Checkpointer 的具体类、导入路径和初始化方式随版本变化，生产实现必须按锁定版本官方文档和集成测试确认。

## 16. Graph Builder Registry

Graph Builder 由代码显式注册，例如 `mentorship_assistant_v1` → `build_mentorship_graph`。Agent Definition 只能引用已注册 Graph Key，不允许从数据库读任意 Python Import Path。

Graph Builder 应返回 Compiled Graph，并声明：
- graph_key；
- graph_version；
- state_schema_version；
- required_runtime_ports；
- required_checkpointer；
- interrupt_types；
- supported_stream_modes；
- compatibility_range。

## 17. Graph 与 Run 状态协调

LangGraph 运行状态与 Application Run 状态需要协调但不能混为一体。Run 状态可由 Worker 根据 Graph 输出、Interrupt 和异常映射更新。出现状态不一致时，以明确的恢复协调器进行对账，不要让 UI 直接根据某个 Graph Event 推断 Run 最终状态。

## 18. Graph 测试示例

- 用 `InMemorySaver` 验证一次简单中断和恢复；
- 用 Fake Model 验证 Tool Call 路由；
- 用固定 State 验证每条 Conditional Edge；
- 用 Property Test 验证 Reducer 对输入顺序是否稳定；
- 模拟 Node 抛异常，验证 Checkpoint 和 Run 状态；
- 模拟相同 Tool Node 被重放，验证 Executor 幂等；
- 验证非法 `thread_id` 无法跨 Tenant 恢复。
