# DeerFlow 2.0 源码架构拆解（首轮）

> DeerFlow README 声明 2.0 ground-up rewrite，与 1.x 不共享代码。当前 checkout Markdown/MDX 约 4,500，含 backend 大量架构/运维设计文档与 skills；本页只完成入口和源码地图，不代表所有文档已读。

## 定位与系统边界

DeerFlow 2.0 是 super-agent harness 产品：以 LangGraph 负责 Agent 执行，以 backend API/worker 提供任务生命周期，以前端提供会话产品；通过 skills、subagents、memory、sandbox、MCP/extensions 连接能力。重要的架构区分是：LangGraph graph 是 Agent 逻辑，DeerFlow gateway/worker 是运行服务和多租户产品控制面，Skill 是被 harness 装配的说明/脚本资产。

## 主要组件

- `backend/app/gateway/`：HTTP/API、认证、run 创建/等待/取消、流协议与外部兼容接口。
- `backend/app/` 下 worker/runtime 模块：消费 run、构造 graph/runtime、转发 LangGraph 流、更新任务状态。
- `backend/packages/harness/`：DeerFlow Agent harness 能力封装及执行图相关逻辑。
- `backend/packages/extension-api/` 与 extensions：扩展契约和可插拔集成。
- `backend/app/scheduler/`、`mcp_tasks/`、`subagent_batches/`：调度、MCP 后台任务和子任务管理。
- `frontend/`：会话 UI 与 SSE/事件呈现；`skills/`：public/private 技能与脚本。
- `backend/docs/ARCHITECTURE.md`、`API.md`、`AUTH_DESIGN.md` 等：当前源码旁的系统设计材料，值得作为后续逐篇阅读起点。

## 请求到回答的生命周期

用户请求先进入 gateway，认证/授权与请求 schema 检查后创建 run；任务被投递到执行 worker，worker 根据用户/Agent 配置建 graph config 与执行上下文；graph 以流模式运行，产生消息、工具、子图等事件；worker 将受支持事件编码并发布到 run stream，同时持久化进度/终态；客户端订阅流并展示；断连后的 wait/reconnect 通过 run/stream 状态继续观察。可见它把用户连接和长执行解耦，SSE 连接断开不等于 Agent 自动取消。

## 架构优势与风险点

- Worker/gateway 分离支持长任务、重连和水平扩展，但要求 run 状态与流事件之间有严格终态/恢复语义。
- Graph 的工具权限、审批和 workspace 风险控制必须与产品租户身份绑定，不能只依赖 Agent prompt。
- 技能系统增强能力但把脚本/依赖带进执行边界；sandbox、文件路径和导出流程是安全核心。
- 2.0 重写意味着不能用 1.x 的论文/文章来推断当前实现。

## 覆盖账本

- 已看 README 定位；已定位 backend、frontend、skills、extension packages 和 ARCHITECTURE/API/AUTH 文档入口。
- 未完成：4,500 个 Markdown/MDX 逐篇阅读；尚需完整追读 run 创建到 worker/checkpointer/stream终态、数据库持久化、subagent 和 sandbox 代码。
