# Context Token Budget：预算分配与压缩实战

## 本文档在系统中的作用

将 Token Budget 从“尽量控制上下文长度”变成可计算的资源分配策略。预算应在每次 Model Call 前执行，并根据模型上下文窗口和输出预留动态调整。

## 1. 可用输入预算

`input_budget = model_context_limit - reserved_output - safety_margin`

例如模型上下文上限为 64,000 tokens，预留输出 4,000，安全余量 2,000，则可用输入预算为 58,000。该数值只是演示，实际以所选模型能力和 API 约束为准。

## 2. 预算分区

示例比例：
- System / Safety：10%；
- Current User Request：8%；
- Current Run State / Plan：10%；
- Tool Schemas：12%；
- Business Data / Observations：20%；
- Relevant Memory：10%；
- RAG Evidence：15%；
- Recent History：10%；
- Slack / Buffer：5%。

比例不是固定配置。工具型任务可能提高 Tool Schema 和 Business Data；文档问答提高 RAG；多轮对话提高 History。

## 3. 硬约束与软预算

硬约束：
- System Policy 不可裁剪；
- 当前用户请求不可裁剪；
- 当前执行所需关键 ID 不可裁剪；
- 写操作的审批和资源版本信息不可裁剪；
- 必需 Tool Schema 不可裁剪。

软预算：
- 远期 History；
- 非相关 Memory；
- 次要 RAG；
- 大段 Tool Output；
- 可由引用重新读取的历史细节。

软预算超出时按优先级和效用密度裁剪。

## 4. Candidate 的效用密度

`utility_density = relevance × trust × freshness × confidence / estimated_tokens`

Trust / ACL 不合格时直接排除，不参与排序。Freshness 需要按数据类型定义：业务状态分钟级，政策文件按发布版本，用户偏好可能长期有效。

## 5. 压缩策略

1. 去重；
2. 移除已完成且无后续依赖的旧 Tool Schema；
3. 摘要远期对话；
4. 将大型 Tool Result 替换为 Result Ref + 核心字段；
5. RAG Chunk 只保留支持当前断言的段落；
6. 保留来源和不确定性标记。

禁止把关键审批、否定条件、金额、日期、状态版本和用户纠正压缩掉。

## 6. 压缩校验

对压缩前后内容抽取关键实体、数字、日期、条件、否定词和 Source Ref。若关键字段缺失，拒绝压缩结果或回退到更少的可选内容。

## 7. Token 估算误差

不同 Provider 的 Tokenizer 不同。Adapter 提供模型级估算；无法精确估算时采用保守比例，并在实际 Usage 返回后校正预算模型。不能把字符数当成准确 Token 数。

## 8. 预算观测

记录 Estimated Input Tokens、Actual Input Tokens、Compression Ratio、Truncation Count、Required Context Overflow、Context Compile Latency 和最终任务成功率。若频繁发生 Required Overflow，应优化 Tool Schema 和业务结果投影，而不是不断提高模型上下文窗口。
