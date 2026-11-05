# AgentScope 2.0.12dev 源码阅读手册

按本地文档追踪实际代码，从架构进入各功能专题。此前 1,259 份 Markdown 统计错误，混入非 Markdown 文件；重新扫描公开 21 份，含隐藏目录共 24 份。阅读覆盖详见清单，不将扫描数量当作已读数量。

## 阅读顺序

1. [文档逐份解读与源码映射](00-Documentation-and-Source-Map.md)
2. [整体架构、分层与调用链](01-Architecture-and-Call-Chains.md)
3. [Agent Loop 深层源码解析](02-Agent-Loop.md)
4. [Middleware 的执行边界](03-Middleware.md)
5. [上下文工程与压缩](04-Context-Engineering.md)
6. [状态、会话与恢复](05-State-and-Session.md)
7. [长期记忆：AgenticMemory、Mem0、ReMe](06-Long-Term-Memory.md)
8. [RAG 与知识库](07-RAG-and-Knowledge.md)
9. [工具、Skills 与 MCP](08-Tools-Skills-MCP-and-Permissions.md)
10. [Workspace、沙箱执行、环境生命周期与内容外置](09-Workspace-and-Artifacts.md)
11. [模型、Provider 与流协议](10-Models-and-Provider-Abstraction.md)
12. [HITL：授权、询问、暂停恢复与子 Agent 投影](11-Human-in-the-Loop.md)
13. [多 Agent 编排、通信契约、权限与全局预算](12-Multi-Agent-Orchestration-and-Contracts.md)
14. [成本归因、事前预算闸门与有序降级](13-Cost-Attribution-Budgets-and-Degradation.md)
15. [安全、权限与协议治理](14-Security-Permissions-and-Protocol-Governance.md)
16. [计划与任务跟踪：LLM 回路、EXPLORE、依赖和持久化](15-Planning-and-Task-Management.md)
17. [消息渠道：飞书、钉钉、Discord 的连接、路由与双向交互](16-Message-Channels-Feishu-DingTalk-and-Discord.md)
18. [Pipeline 流水线：Goal、Team、固定步骤 SOP 与服务提交协议](17-Pipelines-and-SOP.md)

当前已围绕运行核心、上下文、记忆、知识、能力、环境、模型、消息渠道和 Pipeline/SOP 建立专题；每篇标出深读方法与尚待追踪的分支。应用服务与存储的其它机制、通用消息事件/UI、实时语音、可观测性和分类模型等仍需继续成篇。分析依据静态阅读，没有运行应用或测试。

## 图解阅读导航

`01`～`14` 已完成本轮图解细化，共 86 个 Mermaid 图块。每篇后部新增的“图解深化”或“源码深化”将正文中的关键逻辑展开为流程、时序、状态和故障定位表；`05`、`10` 同时补全了原先较短的源码说明。图中的建议架构与当前实现分别标注，不能把建议流程当成已有默认功能。

| 专题 | 图块数 | 重点看图理解 |
|---|---:|---|
| [01 架构](01-Architecture-and-Call-Chains.md) | 7 | 库/服务入口、资源退出、会话锁、SSE 回放 |
| [02 Agent Loop](02-Agent-Loop.md) | 5 | 动作决策、并发队列、暂停恢复、取消收尾 |
| [03 Middleware](03-Middleware.md) | 5 | hook 位置、洋葱顺序、短路与结束信号 |
| [04 上下文](04-Context-Engineering.md) | 6 | 输入组装、配对切分、压缩提交、外置回读 |
| [05 状态与会话](05-State-and-Session.md) | 7 | 状态分区、尾消息、缓存、持久化与恢复 |
| [06 长期记忆](06-Long-Term-Memory.md) | 6 | 三种方案的读写时序、异步召回、索引可见性 |
| [07 RAG](07-RAG-and-Knowledge.md) | 8 | 摄取、多查询、多库、rerank 与主 LLM 接入 |
| [08 能力系统](08-Tools-Skills-MCP-and-Permissions.md) | 5 | 发现/激活/授权、Skill 与工具、MCP 生命周期 |
| [09 Workspace](09-Workspace-and-Artifacts.md) | 5 | 环境初始化、沙箱网关、清理、共享文件边界 |
| [10 模型适配](10-Models-and-Provider-Abstraction.md) | 8 | provider 分层、流累计、重试、结构化策略 |
| [11 HITL](11-Human-in-the-Loop.md) | 5 | 工具状态、部分恢复、执行权威与 UI 投影 |
| [12 多 Agent](12-Multi-Agent-Orchestration-and-Contracts.md) | 6 | 分派/回收、服务消息、回报保护、独立验收 |
| [13 成本](13-Cost-Attribution-Budgets-and-Degradation.md) | 6 | 事后预算、辅助调用、建议预留账本与降级 |
| [14 安全](14-Security-Permissions-and-Protocol-Governance.md) | 7 | 身份、资源、模式决策、凭证与下载签名 |

已检查本轮文档的本地链接与 Markdown 图块结构；未用 Mermaid 渲染器逐图渲染，也未运行 Agent、模型、外部服务或故障实验。图块数仅表示表达形式，不作为源码覆盖或运行正确性的证明。

新增 [16 消息渠道专题](16-Message-Channels-Feishu-DingTalk-and-Discord.md) 含 25 张 Mermaid 图，串联公共渠道控制面、Gateway、出站投递和三平台实现，并明确审批回调、媒体、群聊作用域和多进程部署的验证边界。

新增 [17 Pipeline 专题](17-Pipelines-and-SOP.md) 含 27 张 Mermaid 图，展开 Goal 的执行/验证与预算、Team 的并发/暂停/清理、SOP 的状态交接及服务端提交，区分参与者终态、业务验收与持久恢复。
