# Context Compiler：上下文编译器架构

## 本文档在系统中的作用

Context Compiler 把分散的上下文来源转换成模型当前这一轮真正需要的输入。它不是简单拼接历史消息，而是一个带权限、信任、优先级、Token 预算和来源追踪的编译过程。

## 1. 为什么需要 Compiler

Agent 可能拥有系统指令、长期记忆、Thread 历史、当前请求、业务 Catalog、实时 API 结果、RAG 文档、Tool Schema 和子 Agent 结果。直接拼接会导致 Prompt 过长、过期信息与实时信息冲突、低信任数据伪装成指令、租户内容混入、重要约束被截断，以及无法解释模型为什么看到某段信息。

## 2. 编译阶段

`Collect → Normalize → Authorize → Classify Trust → Deduplicate → Resolve Conflict → Rank → Budget → Compress → Render → Validate`

每个阶段产出可测试中间对象，而不是一个大字符串。

## 3. Context Candidate

```python
from pydantic import BaseModel
from typing import Literal

class ContextCandidate(BaseModel):
    candidate_id: str
    source_type: Literal[
        "system", "agent_config", "user_input", "thread",
        "memory", "business_api", "catalog", "rag", "tool_result"
    ]
    content: str
    source_ref: str | None = None
    trust_level: Literal[
        "platform_trusted", "agent_trusted", "user_supplied",
        "retrieved_untrusted", "external_untrusted"
    ]
    priority: int
    token_estimate: int
    freshness_at: str | None = None
    expires_at: str | None = None
    required: bool = False
    tenant_id: str | None = None
    classification: str = "internal"
```

生产对象还应携带 ACL / Resource Scope 引用；不要把复杂权限表达式塞进 DTO。

## 4. 来源优先级

一般建议：
1. 平台安全规则和系统级约束；
2. Agent Release 中的稳定角色和任务契约；
3. 当前用户明确指令；
4. 当前 Run 的实时业务事实；
5. 当前 Tool Result；
6. 当前 Thread 的近期对话；
7. 经审核的 Memory；
8. RAG / 历史资料。

这不是简单覆盖顺序。冲突时按事实类型判断：业务状态以实时业务 API 为准，用户偏好可以以用户纠正后的 Memory 为准。

## 5. Trust Boundary

检索到的文档、网页、邮件、文件、Tool Result 都是数据。即使出现“忽略之前指令”“调用某个工具”等文字，也不能提升其信任级别。Renderer 使用清晰边界标签和引用元数据。

## 6. ContextBlock

每个 Block 保存 block_id、source_type、source_ref、content、trust_level、priority、token_count、included / omitted、omission_reason、transform_history、checksum。这样 Trace 可以回答模型为什么看到了这段信息。

## 7. Compiler Port

```python
class ContextCompiler:
    async def compile(self, request, scope):
        candidates = await self.collector.collect(request, scope)
        authorized = await self.authorizer.filter(candidates, scope.principal)
        normalized = self.normalizer.normalize(authorized)
        ranked = self.ranker.rank(normalized, request)
        budgeted = self.budgeter.select(ranked, request.token_budget)
        compressed = await self.compressor.compress_if_needed(budgeted, scope)
        return self.renderer.render(compressed, request)
```

## 8. 输出结构

CompiledContext 包含 messages、included_refs、omitted_refs、estimated_tokens、actual_tokens（若可得）、policy_snapshot_id、context_version、compression_summary_ref。模型调用前进行最终长度检查。

## 9. 增量更新

新 Tool Result 不应触发整个历史重建。可将其作为新的 Context Candidate，重新执行局部排序和预算分配。Thread Summary、Memory 和 RAG 索引有独立更新机制。

## 10. 可观测性

记录每类来源候选数、授权过滤数、纳入 Token、压缩 Token、冲突数、过期数和检索延迟。不要把敏感全文写进普通日志；使用受控引用和脱敏摘要。

## 11. 来源适配器的职责

每个 Context Source 只负责把自己的数据转换成 Candidate：
- History Adapter：近期对话、Thread Summary；
- Memory Adapter：符合 Scope 的记忆；
- Catalog Adapter：Domain / Entity / Capability 摘要；
- Business Data Adapter：实时业务 API 返回的结构化结果；
- RAG Adapter：文档 Chunk 和引用；
- Skill Adapter：选中 Skill 的指令和资源；
- Tool Observation Adapter：当前 Run 已执行工具的结果。

Source Adapter 不决定最终是否注入；该决策由 Compiler 统一完成。

## 12. 事实冲突的处理规则

冲突不能只按“新内容覆盖旧内容”处理。按事实类别建立 Resolution Policy：
- 实时业务状态：权威业务系统的最新查询；
- 用户偏好：用户明确纠正优先；
- 组织政策：当前已发布 Policy / Catalog；
- 历史经历：保留时间和来源，不简单覆盖；
- 推测性结论：不得覆盖已验证事实；
- 外部文档：保留版本和发布日期，过期内容降权或排除。

无法判定时，Compiler 输出 Conflict Marker，Agent 应追问或明确不确定性。

## 13. Context Compiler 的幂等与缓存

Context 编译结果可缓存，但 Cache Key 必须包含 Tenant、Thread、Run、Release、Catalog Snapshot、Policy Snapshot、Memory Version、Prompt Version 和输入摘要。权限发生变化时必须失效。缓存命中后仍要执行当前权限检查。

## 14. 编译报告

每次 Model Call 保存轻量 Context Manifest：
- compiler_version；
- included_block_ids；
- omitted_block_ids；
- source_refs；
- token_estimate；
- compression_refs；
- policy_snapshot_id；
- conflict_count；
- freshness_warnings。

该 Manifest 用于排查模型“为什么不知道某件事”，而不是保存所有敏感原文到普通日志。

## 15. 故障降级策略

- Memory 召回失败：可按任务策略继续，但不能声称使用了记忆；
- RAG 失败：要求引用的任务应降级为无法验证；
- Catalog 不可用：禁止猜测工具和 API；
- 实时业务 API 失败：不得用旧 Memory 冒充当前状态；
- Tokenizer 估算失败：使用保守预算并记录；
- Compressor 失败：优先省略可选历史，不删除必需约束。
