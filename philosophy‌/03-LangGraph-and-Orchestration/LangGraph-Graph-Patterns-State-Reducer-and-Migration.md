# LangGraph：Graph 模式、Reducer、迁移与工程陷阱

## 本文档在系统中的作用

进一步拆解 LangGraph Graph 结构设计，帮助工程师把 Agent 工作流设计成可理解、可恢复和可测试的状态图，而不是一个巨大 Node 或一个不可控的 Agent Loop。

## 1. Graph 的粒度

一个 Node 应表达一个有明确输入输出、可观察、可重试的阶段。Node 太大，会让 Checkpoint 粒度粗、故障定位困难；Node 太碎，会增加调度和状态管理复杂度。

建议按“业务步骤”而非“每个 Python 函数”拆 Node：
- classify_task；
- resolve_entities；
- retrieve_context；
- plan_actions；
- execute_read_tools；
- request_approval；
- execute_write；
- verify_effect；
- compose_response。

## 2. Graph Pattern

### Router Graph
先识别任务类别，再进入不同子图。适合多业务域入口。

### Planner / Executor
Planner 产生结构化步骤；Executor 逐步执行并验证。计划必须有 Schema、步骤数限制和能力约束。

### Supervisor / Subagents
Supervisor 委派受限子任务；子 Agent 返回结构化结果；Supervisor 负责汇总和冲突处理。

### Human-in-the-loop
关键节点调用 interrupt，保存 Checkpoint，等待外部响应后恢复。

### Map / Reduce
对独立文档或实体集合并行处理，最后聚合。注意并行节点的 State Reducer 必须确定性。

## 3. Reducer 语义

Reducer 需要满足明确的合并规则。对并行更新字段，优先设计成每个分支写入独立 key 或追加不可变结果。不要让多个分支同时“最后写入者胜出”更新同一业务字段。

## 4. Command 与动态路由

若使用 Command / goto 等机制，模型输出不能直接决定任意节点名称。先将模型选择映射到受控枚举，再进行路由。节点名属于代码级控制面，不是用户可任意输入的参数。

## 5. Graph State 与业务状态

Graph State 可保存当前业务实体引用、期望动作、Tool Outcome 和待审批 ID。业务状态必须通过 Tool / Domain Service 读取和修改。不要把数据库实体快照作为长期 Graph State 的唯一事实。

## 6. State Migration

State Schema 变化必须有版本。迁移函数按版本链执行：
`v1 → v2 → v3`，每个版本迁移有独立测试。不要在恢复时用随意的 `dict.setdefault` 掩盖字段语义变化。

## 7. 节点重放与副作用

Graph 节点可能在恢复时重新执行。纯计算节点可以安全重跑；外部副作用节点必须调用幂等 Executor；发送邮件、扣费、状态变更等操作需要独立执行记录和对账。

## 8. Checkpointer 选择

开发测试可用内存 Checkpointer；生产使用持久化后端。若使用 PostgreSQL Checkpointer，按官方包版本执行 setup / migration，独立配置连接池和保留策略。不要把 Checkpointer 内部表当成平台业务表。

## 9. Subgraph 状态隔离

子图输入只传任务需要的字段，输出结构化 Result。子图不应默认访问父图全部消息、Memory 或工具权限。需要持久子图时明确选择每次调用隔离还是跨 Thread 保留。

## 10. 测试策略

Graph Topology Test、Node Unit Test、State Reducer Property Test、Checkpoint Resume Test、Interrupt Replay Test、State Migration Test、Subgraph Isolation Test、Tool Side Effect Replay Test。
