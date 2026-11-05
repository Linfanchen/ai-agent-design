# LangGraph Python 源码架构拆解（首轮）

> 范围：基于 `open/langgraph` 当前工作树。仓库扫描到 701 个 Markdown/MDX；本轮读根 README，查看核心库结构，并核对 `StateGraph`、`Pregel`、编译图和执行 runner 的实现入口。仓库主要用户文档由外部 docs.langchain.com 托管，当前 checkout 的 `docs/` 不是完整的产品文档源码。因此本文不是 701 篇逐篇阅读的完成声明。

## 核心定位

LangGraph 是低层有状态编排运行时。它不规定 Agent 产品形态，而把状态、节点、边、并发调度、检查点和暂停恢复变成可组合的执行模型。README 明确把长时间运行、有状态 Agent、durable execution、human-in-the-loop 和 memory 列为目标；并说明灵感来自 Pregel/Apache Beam，公开图 API 借鉴 NetworkX。

## 心智模型

- **StateGraph 是构图 DSL**：开发者声明状态 schema、节点和边/条件边；节点读取状态并返回局部更新。
- **Channel 是状态字段的更新语义**：状态 schema 会被映射到 channel；普通 last-value、聚合 reducer、topic/barrier 等 channel 决定更新如何保存与合并。
- **编译产物是 Pregel**：`StateGraph.compile()` 将声明式图转成 `CompiledStateGraph`，继承 Pregel 执行协议。图定义与执行引擎因此分离。
- **执行按 super-step 推进**：每一轮从已触发节点中生成任务、并发执行、收集写入、统一应用 channel 更新，再根据边决定下一批任务。这个批次语义让并行节点有明确汇合点。
- **Checkpoint 把过程变成可恢复执行**：执行配置携带 thread/checkpoint 标识，持久层记录状态快照与任务进展；interrupt 让图在指定点暂停，后续可恢复并注入人工结果。
- **Send/Command 提供动态控制流**：静态边适合固定拓扑；`Send` 可在运行时向节点发出带独立输入的任务，`Command` 可把状态更新与 goto/控制动作组合起来。

## 运行路径

`StateGraph(schema) → add_node/add_edge/add_conditional_edges → compile(checkpointer/store/cache/interrupts) → invoke/stream(input, config) → PregelLoop/PregelRunner → 节点任务 → channel writes → checkpoint/stream events → output`。

关键不是把函数按拓扑顺序简单调用，而是让 Pregel runtime 管理任务队列、并发、写集、错误/重试、step 边界、检查点和事件流。图节点只实现业务步骤；执行语义由通用 runtime 承担。

## 源码文件地图

| 路径 | 作用 |
|---|---|
| `libs/langgraph/langgraph/graph/state.py` | `StateGraph` DSL、schema/channel 推导、图 compile、`CompiledStateGraph` 节点/边绑定。 |
| `libs/langgraph/langgraph/pregel/main.py` | Pregel 通用运行时和 invoke/stream 入口。 |
| `libs/langgraph/langgraph/pregel/_loop.py` | 单次运行 loop 与 step 边界、执行生命周期。 |
| `libs/langgraph/langgraph/pregel/_runner.py` | 已调度 task 的执行、错误处理与重试控制。 |
| `libs/langgraph/langgraph/pregel/_algo.py` | Pregel 任务/写入处理与执行算法核心。 |
| `libs/langgraph/langgraph/pregel/_checkpoint.py` | checkpoint 装载、保存和执行上下文连接。 |
| `libs/langgraph/langgraph/channels/` | 状态通道实现；更新策略是理解 reducer/并行汇合的关键。 |
| `libs/langgraph/langgraph/types.py` | `Send`、`Command`、task 等控制流与执行类型。 |
| `libs/checkpoint/`、`libs/checkpoint-sqlite/`、`libs/checkpoint-postgres/` | checkpoint 协议及不同持久化适配；需进一步追读具体事务/兼容语义。 |
| `libs/prebuilt/` | 常用 Agent/tooling 预制组件，属于建立在低层图运行时上的便利层。 |
| `libs/sdk-py/` | LangGraph 托管服务的 Python SDK；它是远程 API client，不等于本地 graph runtime。 |

## 设计取舍

1. **显式控制流换取可检查性**：图结构和 state 更新可检查、可可视化；代价是开发者要理解节点调度、reducers 和 checkpoint 的语义。
2. **低层内核与高层 Agent 分开**：可用于非 LLM 工作流，也允许上层封装 Agent；不把 prompt/tool-loop 固化为唯一模式。
3. **step 边界支持耐久性**：checkpoint 不是普通日志，而是恢复执行所需状态。节点副作用仍需要应用层保证幂等，避免恢复重放造成重复外部写入。
4. **局部状态更新**：节点通常返回部分 state，channel 合并结果；并行写同一字段时必须有明确 reducer，否则冲突应暴露而不能隐式猜测。

## 已核验与待补齐

- 已读：根 `README.md` 的定位/能力/生态说明；源码入口 `graph/state.py`、`pregel/main.py`、`pregel/_runner.py`、`types.py` 的类定义索引和组织方式。
- 当前还需逐段追读 compile channel 构造、PregelLoop/runner 调用次序、interrupt resume/checkpoint saver 和 store 的一致性语义。
- 文档覆盖：`open/langgraph` 总计 701 个 Markdown/MDX；本轮未逐篇读完，也未覆盖外部官方文档站各主题页面。下一轮应按 checkpoint、interrupt、streaming、subgraph、persistence 文档组核对源码。
