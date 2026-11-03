# 09. Context Engineering

## Role in the Agent System

Builds the information set visible to a model for one execution step.

**Primary responsibility:** Own this capability as a reusable subsystem with explicit contracts, state ownership, security boundaries, and observable outcomes.

**Boundary:** This subsystem must not silently take over responsibilities owned by adjacent layers. Its public interface should be testable without running a full Agent conversation.

## 1. Context 的定义

Context 是某一次模型请求实际可见的信息集合。它不是 Memory 的同义词，也不是历史消息的简单拼接。

Context Builder 的任务是：在 Token、时效、权限和相关性约束下，为当前 Step 选择最合适的信息。

## 2. 推荐上下文分层

| 层 | 内容 | 变化频率 | 处理 |
|---|---|---|---|
| System Policy | 安全、合规、不可覆盖规则 | 极低 | 固定 |
| Agent Identity | 角色、目标、风格 | 低 | 固定/缓存 |
| Stable Project Context | 项目约定、组织制度 | 低 | 检索/缓存 |
| User / Agent Memory | 偏好、经验 | 中 | Recall |
| Business Catalog | 实体、字段、API、关系 | 中 | 按任务检索 |
| Skill Instructions | 工作流程和规则 | 中 | 按需加载 |
| Live Business Data | 当前项目/学员状态 | 高 | 实时查询 |
| Tool Schemas | 当前允许工具 | 中/高 | 动态发现 |
| Conversation History | 最近消息和摘要 | 高 | 历史窗口/压缩 |
| Runtime Metadata | 时间、locale、trace、budget | 高 | 请求时注入 |

## 3. 原始 State 与 Prompt 分离

LangGraph State 应保存结构化原始数据，不保存已经格式化的长 Prompt 字符串。

```typescript
interface ContextSource {
  sourceType: "memory" | "catalog" | "business_api" | "skill" | "history";
  sourceId: string;
  version?: string;
  fetchedAt: string;
  freshness: "live" | "cached" | "historical";
  confidence?: number;
  content: unknown;
}
```

Prompt Assembly 根据模型、任务和预算格式化 Context。这样换 Prompt 模板不需要迁移 State。

## 4. Context Assembly 流程

1. 解析任务意图和实体。
2. 获取用户、租户、Agent Scope。
3. 检索相关 Memory。
4. 检索 Business Catalog。
5. 识别并加载 Skill。
6. 动态发现 Tool Schema。
7. 需要时调用业务 API 获取实时数据。
8. 读取最近历史和摘要。
9. 去重、排序、冲突标记。
10. 根据 Token Budget 组装。
11. 记录 Context Snapshot 元数据。

## 5. Token Budget

总预算应满足：

`system + agent + memory + catalog + skills + live_data + tool_schemas + history + output_reserve <= context_window`

分配不应固定为通用百分比。应按任务类型配置：
- 简单查询：少量 Skill、少量 Tool、更多实时数据。
- 复杂分析：更多业务知识和证据、较少无关历史。
- 代码探索：更多代码片段和 API 调用链。
- 多 Agent：子任务上下文独立，避免重复注入父上下文。

## 6. 压缩策略

- 保留最近 N 轮完整消息。
- 更早消息形成摘要。
- Tool Result 大对象保存外部引用，只把摘要/必要字段注入。
- 重要决策、约束和待办不应被普通摘要丢弃。
- 压缩前后保留 Session Lineage。
- 不允许压缩掉尚未完成的 Tool Call / Approval 状态。
- 压缩结果需要可验证、可回溯。

## 7. Prompt Cache

稳定前缀优先：
1. System Policy
2. Agent Identity
3. 固定工具使用规范
4. 稳定项目指令

动态内容放后：
- 时间
- 当前业务数据
- 当前检索结果
- 工具 Schema 变化
- 当前用户消息

Provider 缓存策略不完全一致，由 Provider Adapter 负责处理。

## 8. Context Injection

区分：
- Durable injection：进入 Session Log，作为后续历史的一部分。
- Ephemeral injection：只影响当前请求，不永久写入对话。
- Runtime metadata：系统内部字段，不应作为用户可控消息。

## 9. Context 评估

记录 Context token count、Source count、Retrieval precision、Freshness ratio、Duplicate ratio、Missing required facts、Cache hit rate、Compression loss / task regression。

对业务数据明确 `as_of` 时间和数据来源，防止模型把缓存数据误认为实时数据。
