# V3.0 到 V4.0 重构映射

## 本文档在系统中的作用

说明 V4.0 如何继承 V3.0 有效架构思想、移除不适合当前要求的实现路线，并把概念型章节拆成 Python 工程实现章节。

## 1. 技术路线变化

| V3.0 | V4.0 |
|---|---|
| TypeScript + LangGraph TS | Python + LangGraph Python |
| TypeScript Core Contracts | Python Protocol / Pydantic / TypedDict |
| TypeScript Graph Builder | Python StateGraph Builder |
| Python 与 TS 混合表达 | Python 作为唯一核心实现语言 |
| 泛化异步说明 | asyncio、TaskGroup、Cancellation、AsyncSession 专项设计 |

## 2. 模块深化变化

| 模块 | V3 主要内容 | V4 新增深化 |
|---|---|---|
| Harness | 组件构成与生命周期概览 | Factory、Container、DI、装配 DAG、Scope、资源逆序释放 |
| Agent Loop | 基本 ReAct 流程 | 并行 Tool、Retry 分类、No-progress、预算和失败矩阵 |
| LangGraph | StateGraph 概念 | Python 代码、Reducer、Graph Pattern、State Migration、Interrupt |
| Context | 分层和 Token 预算 | Compiler Pipeline、候选对象、算法、来源信任、压缩验证 |
| Memory | 生命周期 | 提取评分、冲突、合并、版本、遗忘、删除传播、召回算法 |
| Business Catalog | 业务资产目录 | Python AST、OpenAPI、SQL 扫描、Entity / Capability、Drift |
| Tool Registry | 注册和执行 | Definition / Version / Binding / Handler / Execution 分离 |
| Multi-Agent | 委派协调 | Delegation Contract、预算、权限 Grant、消息协议和失败策略 |
| Skills | 加载和管理 | SKILL.md Loader、Front Matter、路径安全、渐进加载 |
| Plugin | 插件概念 | Host API、Entry Points、Scope、生命周期、卸载和隔离 |
| Security | 安全原则 | PDP / PEP、ABAC、Intent Binding、Injection、Audit |
| Evaluation | 指标介绍 | Dataset、Trajectory、Evaluator、Judge 校准、Runner 和 Release Gate |

## 3. 保留但重新定位的参考项目

AgentScope、DeerFlow、LangGraph、Octop、DeepSeek Harness、Hermes 仍然作为架构参考。V4 将上游项目的语言实现与本方案 Python 设计区分开：即使上游部分实现使用其他语言，也只吸收架构思想，不把其代码当作 Python 实现。

## 4. V4 的边界

V4 是可实施架构蓝图，包含 Python 接口骨架、关键算法、DDL、流程图和验收要求；它不是已运行验证的完整仓库。生产落地仍需按依赖锁定版本完成集成、迁移和测试。
