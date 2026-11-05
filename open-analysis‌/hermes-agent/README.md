# Hermes Agent 源码架构拆解（首轮）

> 当前 Hermes 是大型 Python Agent 产品仓库，另含 JS 网关/UI 和多种平台接入。Markdown/MDX 扫描约 22,205，通常包括 skills、翻译、网站和测试资产。本页是初轮地图，尚未全文读完文档或完整审计代码。

## 定位与架构形状

Hermes 更接近完整个人 Agent 产品，而不是仅供嵌入的编排库。根部 Python CLI/`run_agent.py` 驱动 Agent loop；`tools/` 和 `toolsets.py` 定义能力；`hermes_state*.py` 是拆分后的本地状态/会话存储与迁移层；`gateway/`、`tui_gateway/`、`acp_adapter/` 将 Agent 接入 Telegram/其他消息平台、终端和 ACP；`apps/`、`web/` 提供用户界面；plugins/skills/providers 形成扩展生态。

## 运行链路

平台消息或 CLI 输入先经过 adapter/gateway 归一化为对话 turn；运行器加载 profile、provider、会话历史和可用 toolsets，构建模型请求；模型返回文本或工具调用后，工具执行结果被追加到 transcript 并再次进入 loop；最终响应沿原平台发送。会话记录、分支、压缩、resume、后台/detached 工具结果由持久化状态层支撑。Gateway 还承担多客户端/长期连接下的命令、事件和会话控制，因此是产品服务层，不应与一次 Agent 推理混为一谈。

## 关键文件地图

- `run_agent.py`、`agent/`：Agent 生命周期、循环和模型交互路径。
- `model_tools.py`、`toolsets.py`、`tools/`：模型可见工具 schema、toolset 选择和具体工具。
- `hermes_state.py` 及 `hermes_state_*.py`：状态门面与按职责分离的 DB、schema、消息、session、search、WAL、mutation、migration 等模块；文件拆分体现状态一致性/迁移复杂度已成为核心领域。
- `gateway/`、`tui_gateway/`：网关会话与交互协议。
- `providers/`：模型 Provider 适配。
- `plugins/`、`skills/`：扩展和可复用操作指令/脚本。
- `acp_adapter/`：Agent Client Protocol 接入。
- `cli.py`：交互入口与配置命令。

## 设计特点与值得关注处

1. **产品化优先**：跨平台接入、profile/tool selection、持久化 resume、skills 和 gateway 是一等能力。
2. **本地状态的耐久性**：SQLite/WAL、migration、锁、repair、reconcile、branch/rewind 等模块表明运行轨迹需要可维护、可恢复；分析时应追踪事务边界及并发 owner。
3. **工具安全面宽**：shell/browser/filesystem/MCP 与插件共同扩大攻击面，需检查 allowlist、路径约束、权限和审计，不可从 README 功能表推断安全性。
4. **多入口共用核心**：CLI、消息平台和 ACP 需要将平台特定协议转换成统一 turn/session 语义。

## 覆盖账本

- 已看 README 的总体定位，扫描主要顶层目录、Python state 模块和 gateway/adapter 路径。
- 未完成 22,205 Markdown/MDX 全读；尚需精读 Agent loop、state schema/transaction、tool dispatch/security、gateway 生命周期与平台 adapters。
