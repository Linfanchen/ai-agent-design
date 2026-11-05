# DeepSeek Harness 源码架构拆解（首轮）

> 当前仓库是 TypeScript/Node 为主并带 Python runtime SDK；README 标明 developer preview、兼容性可能破坏。Markdown/MDX 扫描约 20,531，含多语言/i18n 元数据和文档生成材料。本页是源码地图和初步设计理解，不表示全部文档已读。

## 架构理念

README 将核心描述为 “everything-is-a-plugin”，底层由 Cordis 提供插件化组合机制。理解 DSH 的关键，不是把它看作一个单体 Agent 类，而是看作由 Host/应用、Cordis context/service/event/fiber、插件集合、workspace/provider/model/tool 等共同构成的 Agent harness。插件贡献服务和生命周期行为，Host 将其装配为 CLI/Web 产品；模型适配与工具/界面是可换装部件。

## 仓库地形与执行方向

- `packages/`：主要 TypeScript packages，包括 core/harness、plugin、workspace、providers、persistence 等领域组件；包间以 workspace package 依赖组合。
- `apps/`：可运行应用/host 层，通常负责 CLI/Web 入口和插件装配。
- `plugins/`：内置功能插件，是理解“能力如何进入运行时”的主要位置。
- `packages/workspace/`：Agent 文件和命令执行空间适配层。
- `native/`：原生辅助组件；`python/`：Python runtime SDK 辅助远程或嵌入式调用。
- `docs/user/guide/`、`docs/cookbook/`、`docs/cordis-api/`：用户操作、扩展开发和 Cordis API 文档。

典型路径可概括为：CLI/Web Host 选配置并初始化 Cordis App → 注册核心/内置/第三方插件 → 插件声明服务、事件订阅和生命周期钩子 → Agent/模型服务接收请求并驱动工具调用 → 工具经 workspace/provider 等能力执行 → 状态/事件经应用层输出。具体的模型循环落在哪个插件/包需按 package entrypoint 逐层验证，避免仅凭“插件化”口号把执行细节想当然。

## 设计特征

1. **插件是组合单位**：能力可以独立开发、依赖注入、注册服务和响应事件；架构扩展点较清晰，但插件依赖/启动顺序/生命周期和冲突治理成为核心复杂度。
2. **Cordis 提供运行时语义**：Context、Service、Event、Fiber 等抽象使插件可按上下文组合，并可管理异步任务/生命周期。
3. **Host 与 runtime 分离**：Web/CLI 可复用底层插件能力；但持久化与用户态配置需要清晰归属。
4. **developer preview**：仓库自身提示可能有 breaking changes，分析必须记录 checkout revision 并谨慎对待长期稳定性。

## 关键追读路径

- `package.json`、`pnpm-workspace.yaml`：workspace 包集合和脚本。
- `packages/` 下 Cordis 集成、runtime/harness 包：插件装配与核心执行协议。
- `apps/` 中 CLI/Web bootstrap：生产运行时到底装入哪些插件。
- `plugins/` 中 Agent、model/provider、tool、memory、MCP、workspace 相关插件：功能实现的主要落点。
- `packages/workspace/`：执行环境接口、安全边界和后端实现。
- `docs/cordis-api/` 与 extension cookbook：插件作者可依赖的公开契约。

## 覆盖账本

- 已读 README 顶部定位/预览提示；扫描顶层目录及文档主题路径。
- 未完成 20,531 Markdown/MDX 逐篇覆盖；未逐个 package 跟进 plugin activation、Agent loop、数据持久化和权限决策实现。后续应先由 workspace 清单生成去重文档索引，再按核心 guide/cookbook 对代码。
