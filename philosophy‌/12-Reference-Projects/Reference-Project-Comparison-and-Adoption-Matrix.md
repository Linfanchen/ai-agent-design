# 参考项目比较与本方案采用矩阵

## 本文档在系统中的作用

本文件把 AgentScope、DeerFlow、LangGraph、Octop、DeepSeek Harness、Hermes 的架构观察转化为本方案的选择，不追求照搬全部功能。重点区分上游项目事实、架构推断和本方案建议。

## 1. 项目适用侧重点

| 项目 | 主要研究价值 | 本方案重点借鉴 |
|---|---|---|
| AgentScope | Python Agent 组件、Message、Toolkit、Memory、Middleware | Python 组件化、消息与工具抽象 |
| DeerFlow | 复杂任务 Harness、Subagent、Skill、Sandbox、Artifact | 长任务产品化、任务委派和沙箱边界 |
| LangGraph | Graph Runtime、State、Checkpoint、Interrupt、Streaming | 核心编排引擎 |
| Octop | 多用户平台、Gateway、Workspace、Agent Manager | Application 与 Runtime 分离、多入口 |
| DeepSeek Harness | Core / Plugin / Scope / Event / Tool Guard | 可插拔能力、生命周期 Scope |
| Hermes Agent | Loop、Context、Memory、Tool Search、Gateway、Cron | 完整 Agent 工程子系统拆分 |

## 2. 本方案组件选择

| 子系统 | 选择 | 原因 |
|---|---|---|
| Orchestration | LangGraph Python | 有状态 Graph、持久化和中断恢复 |
| Agent Components | 自建 Protocol + 参考 AgentScope | 避免被单一 Agent 基类锁定 |
| Harness | 自建 Python Composition Root | 需要租户、预算、Policy、Tool 的企业边界 |
| Context | 自建 Compiler | 需要信任、权限、Token Budget 和来源追踪 |
| Memory | 自建治理层 + 可替换 Store | 需要 Scope、Evidence、Conflict、Delete |
| Tool | 自建 Registry / Executor | 需要业务语义、幂等、审批和审计 |
| Plugin | Python Entry Points + Host SDK | 只对可信包启用进程内插件 |
| MCP | 官方 Python SDK Adapter | 协议互操作，内部权限仍自建 |
| Sandbox | 独立 Docker 执行服务 | 隔离不可信代码 |
| Evaluation | 自建 Dataset / Evaluator Pipeline | 质量门禁需要和业务目标一致 |

## 3. 不照搬的内容

- 不把个人助手的本地存储模式直接用于多租户企业服务；
- 不把 Agent Framework 的 Toolkit 注册当成权限控制；
- 不把 LangGraph Checkpoint 当审计日志；
- 不把 Skill 当作授权；
- 不把 Python Plugin Entry Point 当沙箱；
- 不把 Multi-Agent 自由通信当可靠工作流；
- 不将 Memory 当作业务数据库。

## 4. 源码级阅读方法

对每个仓库锁定 Tag / Commit，按一条端到端任务追踪入口、对象构建、状态传递、工具调用、异常恢复、存储和测试。至少记录：
- 文件路径和类 / 函数；
- 谁创建该对象；
- 谁持有其生命周期；
- 输入输出 Schema；
- 错误如何传播；
- 是否持久化；
- 测试覆盖；
- 许可证和依赖版本。

## 5. 采用决策

V4 的核心架构以 Python + LangGraph 为主。其他项目提供设计思想和对照，不意味着引入其全部依赖。每次引入上游代码都必须检查许可证、依赖安全和维护活跃度。
