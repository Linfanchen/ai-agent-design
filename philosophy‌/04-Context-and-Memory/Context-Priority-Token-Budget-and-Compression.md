# Context 优先级、Token Budget 与压缩算法

## 本文档在系统中的作用

细化有限上下文窗口下哪些内容必须保留、哪些可以压缩、哪些可以省略，以及如何避免压缩破坏业务事实和安全约束。

## 1. 预算拆分

总 Context Budget 不是模型最大窗口。先预留 System / Policy、Agent Role / Task Contract、Tool Schema、当前用户请求、Output Token、Provider 特殊开销和 Safety Margin。剩余预算才分配给 History、Memory、RAG、Business Context 和 Tool Result。

`usable_input = context_window - max_output_tokens - provider_overhead - safety_margin`

每个来源设置 minimum / maximum。Required Block 先占用预算；可选 Block 按效用排序。

## 2. 选择算法

对候选块计算：
`utility = relevance × freshness × trust × task_fit × source_quality`

然后按 `utility / token_cost` 排序。互相依赖的 Block（例如 Tool Result 的解释依赖 Tool Call）作为 Bundle 选择，不能单独丢弃。

简单贪心算法适合实时编译；高精度预算可用 0/1 Knapsack，但要控制计算开销。实际应加入来源配额，防止大量 RAG Chunk 挤占全部上下文。

## 3. 不能压缩的内容

当前用户明确要求、平台安全约束、关键审批状态、有效业务实体 ID / Version、未解决关键问题、Tool Execution UNKNOWN 状态、结构化输出契约。

## 4. 可压缩内容

已完成对话、重复背景、旧 Tool Result、重复 RAG 文档、长文档证据摘要、已完成子任务输出。压缩必须保留来源引用、时间和置信度，不能把“可能”改成确定事实。

## 5. 分层压缩

1. Exact Dedup：去除重复内容。
2. Message Folding：合并重复问答。
3. Structured Summary：按目标、已知事实、决定、未解决项、引用生成摘要。
4. Retrieval Instead of Inclusion：保留引用，后续按需重取原文。
5. Window Truncation：最后才使用，并遵循明确策略。

## 6. 压缩质量校验

对压缩前后检查关键实体 ID、数值、日期、状态、否定和条件、来源引用、未解决问题、用户纠正。高风险任务可要求压缩结果通过结构化 Schema 和关键事实比对。

## 7. 版本化摘要

Thread Summary 保存 summary_version、covered_message_range、generated_by_model、source_message_refs、summary_schema_version、checksum、created_at。新消息加入后不要直接覆盖旧摘要而丢失覆盖范围；保存 lineage。

## 8. Token 估算

优先使用对应 Provider / Model 的 Tokenizer 或 SDK 估算。字符数只能作为粗略预估。模型请求前保留安全余量，并处理供应商实际 Token Usage 与估算不一致。

## 9. 指标

Context Overflow Rate、Required Block Retention、Context Token per Run、Compression Ratio、Summary Fact Preservation、Retrieval Miss Rate、Context Source Precision。
