# AutoGen 源码架构拆解（首轮）

> AutoGen checkout 含 Python 与 .NET 两套实现；总 Markdown/MDX 约 1,837。根 README 明确标记仓库 archived、不再维护，并建议新项目看 Microsoft Agent Framework。本文按当前代码作历史架构分析，不构成新项目选型建议；尚未逐篇读完文档。

## 架构主线

AutoGen 的重要演进是从早期“可对话 Agent 对象”走向分层运行时。Python `autogen-agentchat` 提供 AssistantAgent/UserProxyAgent 和团队抽象；`autogen-core` 提供消息、runtime、agent identity、组件序列化和跨 Agent 消息投递；`autogen-ext` 放模型/工具等扩展。AgentChat team（如 RoundRobinGroupChat、SelectorGroupChat）把成员 Agent 的消息交给团队管理器，由 manager 控制下一位发言者、终止条件和状态。

## 执行模型

在 AgentChat 层，调用方给 Team/Agent 输入任务，team manager 按策略向参与者广播/路由消息；AssistantAgent 将对话历史转换成模型请求，解析文本/工具调用；工具调用执行后以结果消息回到对话循环；终止条件结束轮次并返回 TaskResult。更底层的 Core 是 message-driven actor runtime：Agent 注册到 runtime，通过 topic/direct message 收发消息，runtime 管理调度和生命周期。分层后，高层对话 API 不必暴露底层 transport 细节。

## 关键路径

- `python/packages/autogen-core/`：核心 runtime、Agent、message、Component 配置/序列化协议。
- `python/packages/autogen-agentchat/src/autogen_agentchat/agents/_assistant_agent.py`：AssistantAgent 模型交互和工具执行。
- `.../agents/_base_chat_agent.py`：ChatAgent 抽象契约。
- `.../teams/_group_chat/`：团队 manager、参与者状态、消息与事件协议。
- `python/packages/autogen-ext/`：OpenAI 等模型 client 和外围集成。
- `dotnet/src/AutoGen/Agent/ConversableAgent.cs`、`AssistantAgent.cs`、`UserProxyAgent.cs`：.NET API 的对话 Agent 实现；它与 Python 包结构不是逐文件同构。
- `python/packages/autogen-studio/`：可视化构建/运行界面，属于产品层，不是核心 runtime。

## 设计特征与边界

1. **消息驱动协作**：Agent 间通过消息形成协作，不要求所有参与者共享一个大状态对象。
2. **团队策略外置**：Manager 决定轮转/选择/停止，成员实现各自能力。
3. **组件可序列化**：Core 的 Component config 让 Agent/team 可配置、保存和重建，支撑 Studio。
4. **Python 与 .NET 并存**：概念相似但实现/API 不完全相同，学习源码时必须限定语言和包版本。
5. **维护状态是关键事实**：README 明确 archived，安全/依赖修复不再承诺；架构适合研究，不应忽略其生命周期风险。

## 覆盖账本

- 已读入口：`README.md`；已定位 Python AgentChat/Core/Ext、Studio 和 .NET Agent 实现路径。
- 未完成：1,837 个 Markdown/MDX 逐篇阅读；尚未精读 group-chat manager、runtime scheduler、checkpoint/component loader 的完整实现。源码级分析应继续限定 Python 主路径并以 .NET 为单独对照。
