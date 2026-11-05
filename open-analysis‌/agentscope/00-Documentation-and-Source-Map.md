# 文档逐份解读与源码映射

当前版本为 2.0.12dev。此前 1,259 个 Markdown 的统计错误，过滤失效混入了其它类型文件。按扩展名重新扫描：公开 21 份，含隐藏目录共 24 份。批量输出发生截断，不计为全文阅读。

`CONTRIBUTING.md` 说明正式用户文档在独立 agentscope-ai/docs 仓库；本地文档不是完整官方文档站。`docs/changelog.md` 是 v1 历史，不能直接拿其中 ReActAgent/Memory 模块描述解释当前实现。

| 文件 | 文档要点及对应源码 | 阅读状态 |
|---|---|---|
| README.md | 2.x 产品入口，对应 agent/app/model/workspace | 入口已读，全文待补 |
| README_zh.md | 模型推理和工具调用、SOP/Team/实时等发布入口 | 入口已读，全文待补 |
| CONTRIBUTING.md | 统一核心 Agent，model/credential/card/formatter 配套，lazy import | 精读 |
| CONTRIBUTING_zh.md | 中文贡献约定 | 定位，全文待复核 |
| docs/changelog.md | v1 历史变化 | 精读 |
| docs/roadmap.md | AOP、语音阶段与生态方向，含历史完成项 | 精读 |
| docs/NEWS.md | 发布历史逐条核对源码 | 定位 |
| docs/NEWS_zh.md | 中文发布历史 | 定位 |
| scripts/model_examples/README.md | 模型样例脚本，进入 scripts/model_examples、model | 定位 |
| examples/a2a/README.md | A2AAgent 不继承 Agent，本地掌握 context/task，远端掌握推理 | 精读 |
| examples/realtime/README.md | 持续 transport、音频/模型 turn 限制，进入 agent/_realtime 与 realtime | 部分已读 |
| examples/tui/README.md | UI 展示 Msg，launcher 串行驱动 reply_stream | 精读 |
| examples/rag/README.md | parse/chunk/embed/insert/search，static/agentic middleware，多 backend | 精读 |
| examples/workspace/apple-container-workspace.md | Workspace backend 与 app manager | 定位 |
| examples/pipeline/goal/README.md | Executor/Verifier 共享 workspace，结构化验证，恢复保留预算 | 精读 |
| examples/long_term_memory/reme/README.md | middleware/_longterm_memory/_reme | 定位 |
| examples/long_term_memory/mem0/README.md | middleware/_longterm_memory/_mem0 与模型适配 | 定位 |
| examples/long_term_memory/agentic_memory/README.md | Markdown memory 索引和 topic 文件，索引进 system prompt，相关文件按需注入 | 精读 |
| examples/console/README.md | console 事件输出 | 定位 |
| examples/agent_service/README.md | app Manager/FastAPI 的服务模式 | 定位 |
| examples/web_ui/frontend/README.md | React/Vite 模板，不是 Agent 专属协议 | 精读 |
| .gemini/styleguide.md | 开发风格 | 定位 |
| .github/copilot-instructions.md | 仓库开发指引 | 定位 |
| .github/PULL_REQUEST_TEMPLATE.md | 变更说明模板 | 定位 |

## 从文档推到架构

TUI 串行调用 reply_stream 是因为 Agent 有共享 context/reply_context/tool 状态；并发工具不代表并发 reply。RAG 文档将摄取与推理时检索分开：rag 包负责数据构建，RAGMiddleware 负责 static 注入或 agentic 搜索工具。

GoalPipeline 的 Verifier 是普通 Agent，读取实际 workspace 产物后给结构化 verdict；它的 pipeline budget 与每个 Agent 的 ReAct budget 是两个维度。Agentic Memory 持久性来自磁盘 Markdown，新 Agent 复用 workdir 可回忆，不依赖旧 AgentState。

A2A INPUT_REQUIRED/AUTH_REQUIRED 在本地会结束当前 reply 并保留 task_id，下一条消息继续远端 task；它与本地 Agent park 不发 ReplyEnd 的协议有区别，不能只凭相同 reply_stream 名称推断生命周期。实现入口为 agent/_a2a_agent.py、state/_a2a_state.py。
