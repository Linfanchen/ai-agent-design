# 设计原则与架构取舍

## 本文档在系统中的作用

约束架构决策，防止功能增长后出现过度耦合、过度抽象和安全边界失控。它给出每类技术选择的判断依据。

## 1. 先定义业务能力，再决定 Agent 形态

固定输入、固定输出、无工具决策、无中断恢复的任务，普通 LLM Chain 或服务函数更简单。只有需要动态决策、工具选择、多步状态、环境反馈或人工干预时才使用 Agent / Graph。

## 2. Graph 与 Harness 分离

Graph 负责“下一步做什么”；Harness 负责“这次运行允许做什么、依赖是什么、资源如何释放、结果如何被记录”。Graph 不直接构造数据库连接、加载密钥或创建工具实例。

## 3. Control Plane 与 Data Plane 分离

Control Plane 管理 Agent Definition、Release、Model Profile、Tool Binding、Skill Version、Plugin Installation、Policy Profile。Data Plane 执行 Run、Tool、Graph、Memory Recall 和 Event。配置变更只影响新 Run；运行中的 Run 默认使用创建时固定的 Release Snapshot。

## 4. 同步与异步边界

短任务可同步等待；任何可能超过 HTTP 超时、需要审批、需要多 Agent 或有较大外部 I/O 的任务，都应转为持久化 Job。HTTP 请求断开不等于 Run 自动取消。

## 5. 状态分层

- Application State：Run、Job、Approval、Quota。
- Graph State：当前任务执行需要的变量。
- Checkpoint：Graph 在特定 super-step 的可恢复快照。
- Event Log：执行事实和 UI 投影。
- Memory：跨会话可复用的用户 / 项目 / 组织知识。
- Business State：权威业务系统中的真实业务状态。

## 6. 失败语义优先于重试策略

只读模型请求可以有限重试；有副作用的 Tool 必须先确定是否已执行。超时不代表未执行。系统应使用幂等键、外部操作 ID、查询对账和 UNKNOWN 状态。

## 7. 最小可用抽象

先定义稳定 Port，再按实际需要提供 Adapter。不要为了“未来可能替换”提前设计几十个泛型接口。抽象是否值得存在，应看它是否有多个真实实现、是否形成独立生命周期、是否能隔离变化。

## 8. 选择矩阵

| 场景 | 推荐方式 | 避免方式 |
|---|---|---|
| 固定摘要 | 单次 LLM Chain | Multi-Agent |
| 固定审批流 | Workflow / LangGraph | 模型自由规划所有步骤 |
| 开放式研究 | Agent Loop + 工具 | 全部硬编码分支 |
| 业务查询 | Catalog + Read Tool | 全部数据库表暴露给模型 |
| 业务写入 | Command Tool + Policy + Approval | 模型生成 SQL 直接执行 |
| 长任务 | Durable Job + Checkpoint | API 请求中 await 数分钟 |
| 多角色任务 | 有界 Delegation | 无限制 Agent 自由互聊 |
| 稳定重复能力 | Tool / Skill | 每次从零 Prompt |

## 9. 架构决策门槛

每项新增子系统至少回答：解决什么真实问题？能否用现有模块实现？状态由谁持有？是否增加权限边界？失败是否可恢复？是否需要独立部署？是否有测试和指标？维护成本是否小于收益？
