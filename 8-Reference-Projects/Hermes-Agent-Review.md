# 04. Hermes Agent 源码拆解

## Role in the Agent System

Studies a complete assistant product's loop, Context Engine, Memory Provider, deferred Tool Search, Skills, and Plugins.

**Primary responsibility:** Own this capability as a reusable subsystem with explicit contracts, state ownership, security boundaries, and observable outcomes.

**Boundary:** This subsystem must not silently take over responsibilities owned by adjacent layers. Its public interface should be testable without running a full Agent conversation.

## 1. 项目定位

Hermes 是一个具备工具调用、持久记忆、Skills、插件、Gateway、Subagent、自动化和持续学习能力的通用 Agent。它更接近可运行的个人/工作助手，而非单纯 Agent SDK。

截至 2026-11-03，GitHub Releases 页面显示 v0.22.6 于 2026-11-02 发布。分析时应以具体 Release Tag / Commit 复核，不能把 main 分支接口当成稳定 API。

## 2. 总体结构

官方 Developer Architecture 将核心职责分成：
- Entry Points：CLI、Gateway、ACP、Batch Runner、API Server、Python Library。
- AIAgent / Conversation Loop。
- Prompt Builder / Provider Resolution / Tool Dispatch。
- Session Storage。
- Context Compression / Prompt Caching。
- Gateway / Plugins / Skills / Memory Providers。

建议关注源码入口：
- `run_agent.py`
- `agent/conversation_loop.py`
- `agent/turn_*.py`
- `agent/prompt_builder.py`
- `agent/context_engine.py`
- `agent/memory_manager.py`
- `agent/memory_provider.py`
- `tools/registry.py`
- `tools/model_tools.py`
- `tools/tool_search.py`
- `hermes_state.py`
- `plugins/`
- `skills/`

## 3. Prompt Assembly

Hermes 将 System Prompt 分层：
- Stable：身份、工具使用约束、稳定指令。
- Context：调用方上下文、项目文件、工作区快照、平台信息。
- Volatile：Skills Index、Memory / User Profile Snapshot、外部 Memory Provider、时间、Session / Model / Runtime 信息。

它还区分缓存稳定前缀与每次 API 请求的临时内容，并使用 Provider 相关 Prompt Cache 能力。

借鉴：
- 稳定内容尽量前置，提升 Prompt Cache 命中。
- 动态业务数据放在易变化的后段。
- Skill 目录只注入索引，正文按需加载。
- Prompt 模板与原始 State 分离。
- 不同 Provider 的缓存语义由 Adapter 处理。

## 4. Tool Registry / Tool Search

Hermes 的工具系统有集中 Registry，工具按 Toolset 组织，支持平台可用性、参数 Schema、Dispatch、错误包装。

Tool Search 是 Progressive Disclosure：
- 启用延迟加载时，MCP / Plugin 等工具不全部放入模型可见工具数组。
- 通过 `tool_search`、`tool_describe`、`tool_call` 三个桥接工具发现、读取 Schema、调用目标工具。
- 默认核心工作集可常驻，冷门/事件触发工具可延迟。
- Catalog 从当前实时 Tool Definitions 重新构建，避免按 Session 缓存导致工具集合漂移。

企业系统借鉴：
- 全量 API 不等于全量模型 Tool。
- Tool Catalog 与 Business Capability Catalog 分离。
- 搜索结果必须受 Tenant / User / Agent Scope 过滤。
- Tool Schema 必须带版本。
- `tool_call` 桥接层不能绕过目标 Tool 的权限、审批、审计和参数校验。

## 5. Skills

Hermes Skill 是按需加载的知识文档，遵循渐进式披露并兼容 Agent Skills 开放格式：
- Skill Index：名称、描述、触发条件。
- `SKILL.md`：完整工作流指令。
- references / scripts / assets：按需读取。
- Skills 可由仓库内置、外部目录、Hub 或 Agent 创建。
- Skill Curator 可以评估和维护 Skill。

借鉴：
- Skill 与 Tool 分离。
- Skill 负责“怎么做”，Tool 负责“能做什么”。
- 通过候选、测试、审核、发布的 Skill 生命周期。
- 外部 Skill 默认不可信，必须校验来源、权限和脚本。

## 6. Memory / Context Engine

Hermes 将 Memory Provider 与 Context Engine 抽象为不同扩展点：
- Memory Provider：跨 Session 保存和检索长期记忆。
- Memory Manager：协调 Provider 生命周期和调用。
- Context Engine：影响上下文构建/压缩策略。
- Session Search：查询历史会话，不等同于 Memory。
- End-of-session / Background Review：在会话后评估是否需要保存记忆或更新 Skill。

借鉴：
- Memory Provider 可替换。
- Context Engine 可替换。
- Memory 不等同于 Session History。
- 会话后异步提取，避免每轮主对话增加明显延迟。
- 后台 Review 应使用隔离快照，不修改主会话的 Prompt Cache 或正在执行状态。

## 7. Session Storage

Hermes 使用 SQLite Session Storage，支持 FTS5 搜索、压缩后的 Session Lineage、平台来源标签和原子写入等机制。

借鉴：
- Session 有来源和父子关系。
- 压缩不能简单删除历史，应保留摘要和 lineage。
- 搜索历史应有时间、平台、用户等过滤。
- 高并发部署要评估 SQLite 单写者约束，企业 SaaS 可选择 PostgreSQL。

## 8. Plugins

Hermes Plugin 可注册：
- Tools
- Hooks
- CLI Commands
- Integrations
- Memory Provider
- Context Engine
- Image / Model 等 Provider

插件发现来源包括用户目录、项目目录和 Python Entry Points。部分扩展类型是单选 Provider，需明确 active selection。插件应显式启用并可版本管理。

借鉴：
- Plugin Manifest + Handler 分离。
- Tool Plugin 和 Provider Plugin 区分。
- Hooks 用于横切行为。
- Memory Provider / Context Engine 用接口抽象。
- 插件启用、升级、回滚和信任策略必须可控。

## 9. Multi-Agent / Delegation / Gateway

Hermes 有 Subagent Delegation、不同平台 Gateway、Session 路由、授权、Cron 等能力。对企业 Agent 的启示：
- 主 Agent 做任务拆分与汇总。
- 子 Agent 拥有独立 Session / Context。
- Delegation Provider 可替换。
- 跨 Agent 消息应带任务 ID、权限和来源。
- Gateway 只负责消息接入、路由和交付，不应承载业务规则。

## 10. 不照搬的部分

- Hermes 是 Python 生态，不建议将其内部对象直接引入 TS Runtime。
- Terminal / Browser 能力对企业业务 Agent 不应默认开启。
- 个人助手的 USER.md / MEMORY.md 方式可作为轻量实现，但多租户企业系统应使用数据库 Scope 和 ACL。
- 自我改进必须经过评估和审核，不能允许 Agent 无约束修改生产规则。

## 11. 上游资料
- <https://github.com/NousResearch/hermes-agent>
- <https://github.com/NousResearch/hermes-agent/blob/main/website/docs/developer-guide/architecture.md>
- <https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/tool-search.md>
- <https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/skills.md>
- <https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/plugins.md>
- <https://github.com/NousResearch/hermes-agent/blob/main/agent/prompt_builder.py>
- <https://github.com/NousResearch/hermes-agent/blob/main/agent/memory_manager.py>
- <https://github.com/NousResearch/hermes-agent/blob/main/tools/tool_search.py>
