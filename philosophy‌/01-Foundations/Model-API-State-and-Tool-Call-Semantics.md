# 模型 API 的状态语义与 Tool Calling 协议

## 本文档在系统中的作用

明确模型 Provider 与 Agent Application 的状态责任边界，回答 API 是否会记住上一次对话、Tool Call 和用户记忆，以及应用如何正确组织多轮 Tool Calling。

## 1. 默认工程假设：每次调用独立构造

普通 Chat / Completion API 调用应被视为无状态请求。应用需要在每次调用时提供模型当前需要的 messages / input、工具 Schema、输出约束和必要上下文。Provider 不会自动知道业务数据库刚刚发生了什么，也不会自动持有 Tool Execution Result。

部分 API 提供 Conversation / Response Reference 或托管会话能力，但它们是供应商特定机制，不应成为应用唯一状态来源。

## 2. 一次 Tool Calling 循环

1. 应用发送当前消息和 Tool Schema；
2. 模型返回 Tool Call；
3. 应用校验 Tool Call 参数；
4. 应用执行 Tool；
5. 应用把 Tool Result 追加到下一轮模型请求；
6. 模型根据当前请求中提供的结果继续；
7. 直到 Final、Interrupt 或终止。

Tool Call 是模型提出的动作意图，不是执行成功证明。执行结果只有在 Executor 持久化后才成为应用事实。

## 3. 五类状态

| 状态 | 维护方 | 是否自动进入下一请求 |
|---|---|---|
| 单次推理内部状态 | Provider / Model | 不应依赖跨请求保留 |
| Provider Conversation / Response Ref | 某些 Provider | 需按 API 明确引用 |
| Agent Thread / Graph State | Application + Checkpointer | Context Builder 选择性注入 |
| Tool Execution / Business Result | Application + 业务系统 | 下一轮按需提供 |
| Long-term Memory | Memory Service | Recall 后选择性注入 |

## 4. 为什么 Provider Conversation 不能代替业务 Thread

Provider 会话结构可能受供应商限制，不一定支持租户隔离、业务审批、审计、跨 Provider 迁移、字段级 ACL 和删除策略。Provider Conversation ID 只能作为 Adapter 可选引用。

## 5. Tool Call 持久化字段

provider_request_id、model_tool_call_id、internal_tool_execution_id、tool_key、tool_version、arguments_hash、execution_status、result_ref、external_operation_ref、retry_attempt、correlation_id。

## 6. Prompt Cache 不等于 Memory

Prompt Cache 是供应商对重复输入前缀的性能 / 成本优化，不代表模型理解或记住用户。缓存命中也不能成为跳过权限检查的理由。

## 7. 设计结论

模型 API 是推理能力，不是企业业务状态管理系统。Agent Platform 必须拥有 Session、Thread、Tool Result、Memory、Permission、Approval 和 Audit。每次模型请求由应用重新构造当前所需上下文。
