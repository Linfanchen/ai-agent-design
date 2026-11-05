# 术语、版本策略与官方资料

## 本文档在系统中的作用

统一术语、版本锁定原则和参考资料入口，避免把 Thread、Run、Session、Memory、Checkpoint 等概念混用。

## 1. 核心术语

- Agent Definition：Agent 逻辑定义。
- Agent Release：不可变发布快照。
- Harness：运行时组合外壳。
- Runtime Scope：一次 Run 的依赖和资源边界。
- Thread：跨多个 Run 的执行连续性。
- Run：一次用户任务执行。
- Job：可调度的后台执行单元。
- Step：Run 内一个节点或动作。
- Checkpoint：Graph 执行状态快照。
- Event Log：执行事实和客户端事件。
- Memory：跨任务可复用的受治理知识。
- Context：当前模型调用所需输入集合。
- Tool：可调用能力。
- Skill：任务方法和指令包。
- Plugin：扩展平台能力的代码或远程组件。
- Capability：业务语义动作。
- Catalog Snapshot：不可变业务语义目录版本。
- UNKNOWN：外部副作用结果暂时无法确认的状态。

## 2. 版本策略

- Python：部署镜像和 CI 固定版本。
- LangGraph：固定具体版本，升级时测试 Checkpoint / Interrupt / Stream。
- Pydantic：固定 V2 兼容范围，Schema 变化跑 Contract Test。
- SQLAlchemy：固定 2.x 版本，测试 AsyncSession 生命周期。
- MCP SDK：官方文档当前区分 v2 稳定线与 v1 维护线，锁定目标版本并按 Migration Guide 处理。
- Provider SDK：每家独立 Adapter 和 Contract Test。
- 数据库：Alembic Migration 必须可升级，必要时支持兼容窗口。

## 3. 官方文档入口

- Python asyncio：<https://docs.python.org/3/library/asyncio-task.html>
- FastAPI Lifespan：<https://fastapi.tiangolo.com/advanced/events/>
- Pydantic：<https://docs.pydantic.dev/latest/>
- SQLAlchemy Async：<https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html>
- LangGraph Python：<https://docs.langchain.com/oss/python/langgraph/overview>
- LangGraph Persistence：<https://docs.langchain.com/oss/python/langgraph/persistence>
- LangGraph Interrupts：<https://docs.langchain.com/oss/python/langgraph/interrupts>
- LangGraph Subgraphs：<https://docs.langchain.com/oss/python/langgraph/use-subgraphs>
- MCP Python SDK：<https://py.sdk.modelcontextprotocol.io/>
- OpenTelemetry Python：<https://opentelemetry.io/docs/languages/python/>

## 4. 参考项目

- AgentScope：<https://github.com/agentscope-ai/agentscope>
- DeerFlow：<https://github.com/bytedance/deer-flow>
- LangGraph：<https://github.com/langchain-ai/langgraph>
- Octop：<https://github.com/TencentCloud/Octop>
- DeepSeek Harness：<https://github.com/deepseek-ai/deepseek-harness>
- Hermes Agent：<https://github.com/NousResearch/hermes-agent>

## 5. 资料使用纪律

上游仓库会持续变化。每次源码级分析记录 Tag / Commit SHA、阅读日期、模块路径和测试证据。文档中区分“上游已实现”“从源码观察”“本方案推导”和“本方案建议”，不能把设计推断写成上游项目事实。
