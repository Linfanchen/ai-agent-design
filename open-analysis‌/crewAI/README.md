# CrewAI 源码架构拆解（首轮）

> 仓库含 40,447 个 Markdown/MDX（包括大量历史版本化 docs 与多语言页面），不可能把扫描到的总数误当本轮已读数。已查看根 README 与源码仓库结构；后续需要按当前版本 docs 路径排除历史/翻译副本后细读。

## 定位

CrewAI 把协作抽象成两类：**Crews** 代表自主 Agent 团队协作，**Flows** 代表事件驱动、可控的流程编排。Agent 描述角色、目标、背景与可用工具；Task 描述预期工作、输出和依赖；Crew 选择 sequential/hierarchical 等 process 并执行任务；Flow 通过状态、事件和条件分支把确定性业务流程与自主 Agent 步骤组合。

## 源码结构与核心类型

仓库实现位于 `lib/` workspace，而非常见 `src/`：

- `lib/crewai/src/crewai/agent.py`：Agent 定义、模型和工具绑定。
- `lib/crewai/src/crewai/crew.py`：Crew 配置、kickoff 生命周期和 process 调度。
- `lib/crewai/src/crewai/task.py`：Task 输入、执行、输出与上下文依赖。
- `lib/crewai/src/crewai/flow/`：Flow 基类、装饰器/监听器、状态管理和执行。
- `lib/crewai/src/crewai/agents/`、`utilities/`：执行器、解析器、LLM/tool 调用支撑。
- `lib/crewai-tools/`：独立工具包；`lib/crewai-*/` 扩展能力分包。
- `lib/crewai/src/crewai/events/`、hooks：运行事件与执行边界扩展。

## 运行模型

Crew kickoff 接收 inputs，构建 kickoff context；根据 process 将任务顺序执行或由 manager 分派；每个 Task 将描述、上下文、Agent role/backstory/goal、工具能力编成模型请求；模型可调用工具，执行结果进入 task output；task output 可作为后续任务 context，最终 CrewOutput 收集结果。Flow 则采用事件触发节点：方法产生状态变化/事件，监听器条件性启动后续步骤，适合把审批、API 调用和 Agent 工作编排为可控业务流程。

## 架构取舍

- Crew 抽象易表达“角色团队”，但自主执行链条的可预测性依赖任务边界、工具集、终止/重试和输出验证。
- Flow 将控制流显式化，适合生产业务骨架；Crew 可嵌入 Flow 的某个步骤。
- Agent/Task 配置既支持代码式声明，也强调 YAML/配置化和 CLI/Studio 体验。
- crewAI 还包含 devtools、版本化 docs、工具集合和发布工具，不能把框架核心与仓库全部内容混为一谈。

## 覆盖账本

- 已看：根 README 开头和仓库目录，确认根实现目录是 `lib/`。
- 未完成：40,447 个 Markdown/MDX 的文档分组/历史版本去重；Agent/Crew/Task/Flow 的实现逐行追踪；生产架构、事件、checkpoint、memory、tool hooks 文档尚待按当前文档版本核读。
