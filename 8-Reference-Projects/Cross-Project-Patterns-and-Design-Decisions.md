# 05. 三项目横向对比与设计取舍

## Role in the Agent System

Synthesizes reference projects into explicit architecture choices rather than copying their APIs.

**Primary responsibility:** Own this capability as a reusable subsystem with explicit contracts, state ownership, security boundaries, and observable outcomes.

**Boundary:** This subsystem must not silently take over responsibilities owned by adjacent layers. Its public interface should be testable without running a full Agent conversation.

## 1. 对比矩阵

| 能力 | Octop / Harness / Memory | DeepSeek Harness | Hermes |
|---|---|---|---|
| 产品定位 | 多用户 Agent 平台 + Runtime + Memory | 可组合插件化 Harness | 通用自主助手 |
| 核心运行时 | deepagents / LangGraph | 自有 Agent Loop / Cordis | AIAgent Conversation Loop |
| Agent Registry | HarnessAgentManager | Agent Registry / Scope | Bot / Agent 配置体系 |
| Session | LangGraph Checkpoint + Session 管理 | Append-only Session Event | SQLite Session Store |
| Memory | Octop Memory 独立系统 | 以 Session / Context 能力为主，需按版本核对扩展 | Memory Provider 插件 |
| Context | middleware / workspace / memory | System Prompt Sections / Inject | Prompt Builder / Context Engine |
| Tool | Built-in / MCP / Subagent | Scoped Registry + Guarded Pipeline | Tool Registry / Toolsets |
| Dynamic Tool Search | 需看具体 Harness 版本与接入实现 | 可组合 Scope / Registry；不能等同 Hermes 完整搜索机制 | tool_search / describe / call |
| Skill | Agent Workspace / Skill Filter | Skill Section + Tool / Inject | Progressive Disclosure |
| Plugin | 主应用插件 / Middleware | Everything is Plugin / Cordis | Plugin API / Hooks / Providers |
| Multi-Agent | AgentManager / Teams | Subagent Provider / Child Session | Delegation / Subagents |
| Workspace | BackendWorkspace | FS / Sandbox / Workspace capability | Terminal / Files / Runtime backends |
| Provider | ChatModelFactory | LLM Adapter Seam | Provider Resolution |

## 2. 最佳借鉴组合

- 平台与多用户管理：Octop
- Memory 数据模型与 Pipeline：Octop Memory
- Harness 组合和生命周期：DeepSeek Harness
- Event / Scope / Plugin：DeepSeek Harness
- Tool Search：Hermes
- Skills：Hermes
- Prompt Assembly / Context Compression：Hermes
- 多 Agent 管理：Octop + Hermes + DeepSeek Subagent Provider
- 持久执行：LangGraph Checkpoint + Event Log

## 3. 不能机械拼装

不同项目的抽象层级不同：
- Octop Harness 是对 deepagents 的生产化封装。
- DeepSeek Harness 是插件组合式运行时。
- Hermes 是端到端助手应用，包含大量产品层和平台层能力。

不能把三者的类名、目录、接口直接复制后放进同一项目。应统一领域术语、定义内部 Contract，再选择实现。

## 4. 本方案的差异化增强

1. 企业多租户和细粒度授权。
2. Business Catalog / Ontology / Capability Registry。
3. 查询通用化、写操作领域化。
4. Context Snapshot 可审计。
5. Memory 的证据、时效、可信等级。
6. Tool Execution 幂等和 UNKNOWN 状态恢复。
7. Skill / Plugin 供应链安全。
8. 代码变更触发 Catalog Diff 和 Tool/Skill 影响分析。
9. 轨迹回放和模型对比评估。
10. 多 Agent 最小权限委派。

## 5. 最终取舍

采用 LangGraph TS 作为业务流程编排；吸收 DeepSeek Harness 的插件化设计，但不复刻 Cordis；吸收 Hermes 的 Tool Search、Skill 和 Context 机制；吸收 Octop 的 Agent Manager、Workspace 和 Memory 设计；将企业权限、业务状态机和数据正确性留在 Java 业务系统。
