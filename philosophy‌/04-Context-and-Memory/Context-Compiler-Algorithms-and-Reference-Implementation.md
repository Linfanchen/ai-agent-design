# Context Compiler：算法分解与参考实现

## 本文档在系统中的作用

本文件将 Context Compiler 设计为可组合的 Python Pipeline，给出候选来源、硬过滤、排序、预算分配、压缩和渲染的算法顺序，避免 Context 逻辑变成散落在 Prompt 中的临时拼接。

## 1. Candidate Source Adapter

每个来源实现同一类异步接口：
- HistorySource：近期消息和摘要；
- MemorySource：经授权的记忆候选；
- CatalogSource：业务实体、Capability 摘要；
- BusinessDataSource：实时业务结果引用；
- RAGSource：文档片段和证据；
- ToolResultSource：本 Run 已执行的结果；
- SkillSource：已选 Skill 指令和引用资源。

Source 只负责获取候选，不决定最终是否注入。

## 2. 处理顺序

1. Collect：并行读取来源。
2. Normalize：统一 Candidate Schema。
3. Authorize：按 Tenant / ACL / Classification 过滤。
4. Trust Label：标记来源可信级别。
5. Deduplicate：按 source_ref / content_hash 去重。
6. Conflict Resolution：按事实类别和有效时间处理冲突。
7. Rank：按任务相关性排序。
8. Budget：选择内容块。
9. Compress：压缩可压缩内容。
10. Render：转换为 Provider Message。
11. Validate：检查 Token、结构和必需 Block。

## 3. 召回并发与部分失败

多个 Source 可并发，但需要来源级 Timeout。RAG 超时不应阻塞全部任务；Policy / Tenant Filter 不可用则不能继续注入受限内容。每个来源结果带 `source_status` 和 `degradation_reason`。

## 4. 候选评分

建议先用规则评分：
`score = w1 * task_relevance + w2 * freshness + w3 * source_quality + w4 * explicitness + w5 * confidence`

Trust / ACL 不是普通分数：不可信或未授权内容必须硬过滤。评分用于排序，不用于授权。

## 5. Budget Selection 伪代码

```python
def select_blocks(candidates, budget):
    required = [c for c in candidates if c.required]
    required_cost = sum(c.token_estimate for c in required)
    if required_cost > budget:
        raise RequiredContextExceedsBudget()

    remaining = budget - required_cost
    optional = [c for c in candidates if not c.required]
    optional.sort(
        key=lambda c: c.utility / max(c.token_estimate, 1),
        reverse=True,
    )

    selected = list(required)
    for candidate in optional:
        if candidate.token_estimate <= remaining:
            selected.append(candidate)
            remaining -= candidate.token_estimate
    return selected
```

生产算法还应处理来源配额、Bundle 依赖、最小证据数量、去重和公平性。不能只按单一 score 选择。

## 6. Conflict Resolution

冲突需要按 Domain Fact Type 定义：
- 实时业务状态：最新权威 API；
- 用户偏好：用户显式纠正优先；
- 组织政策：当前已发布 Policy / Catalog；
- 历史经历：保留时间和来源，不简单覆盖；
- 推测：不得覆盖事实。

无法判断时保留冲突并要求澄清。

## 7. Compression Adapter

压缩器只接收允许压缩的 Block，并返回：
- compressed_text；
- covered_refs；
- preserved_facts；
- omitted_facts；
- uncertainty_notes；
- compression_model；
- checksum。

压缩后校验关键实体、数字、日期、否定、条件和引用。高风险任务压缩失败时应减少上下文或请求澄清，而不是静默删除关键约束。

## 8. Context Trace

每次 Model Call 记录 Context Manifest：纳入 / 排除来源、Token 估算、压缩摘要、Policy Snapshot、Context Compiler Version。敏感内容仅保存在受控存储，普通 Trace 只存引用和 Hash。

## 9. 单测与性质测试

- 相同输入和版本产生确定排序；
- 未授权 Candidate 永不进入输出；
- Required Block 不会被丢弃；
- 总 Token 不超预算；
- 冲突事实不会静默合并；
- 压缩后保留来源引用；
- 某个可选 Source 失败不会破坏安全边界。
