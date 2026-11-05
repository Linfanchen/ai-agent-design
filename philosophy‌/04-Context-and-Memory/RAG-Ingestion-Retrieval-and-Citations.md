# RAG：数据接入、切分、检索、重排与引用

## 本文档在系统中的作用

RAG 为 Agent 提供外部文档知识，适合政策、手册、产品说明和非实时知识，不适合替代交易状态、学员状态等实时业务事实。

## 1. Ingestion Pipeline

`Source Discovery → Authorization Snapshot → Fetch → Parse → Normalize → Chunk → Embed → Index → Validate → Publish`

每阶段有独立状态和重试机制。文档更新后按 content hash 增量处理。

## 2. Chunk 元数据

tenant_id、source_id、document_id、document_version、page / section、chunk_index、content_hash、classification、ACL reference、parser_version、embedding_model_version。

按标题、段落、表格和语义结构切分，避免只用固定字符长度。父子 Chunk 可同时保存：小 Chunk 用于召回，Parent Context 用于解释。

## 3. 混合检索

BM25 / Full Text、Vector Similarity、Metadata Filter、Entity Search、Reranker。先应用 ACL、Tenant 和 Classification 过滤，再将内容交给模型。向量数据库 Metadata Filter 是防御层之一，不替代最终授权校验。

## 4. Reranking

Reranker 输入候选文档和 Query，输出相关性排序。可增加来源可信度、版本新鲜度和文档类型特征。不能把“模型觉得相关”当作内容真实。

## 5. 引用

每个 Chunk 必须有可定位 Source Ref。最终回答中的引用映射到文档、版本、页码 / 段落。没有证据的结论标记为推断或未知。

## 6. 更新与删除

文档更新创建新版本，旧版本按策略保留。删除需清理向量索引、全文索引、缓存和派生摘要。索引任务失败时不能将文档标记为 Active。

## 7. 评测

Recall@K、Precision@K、MRR、NDCG、Citation Correctness、Answer Faithfulness、ACL Leakage、Freshness Lag、Ingestion Failure Rate。

## 8. 检索参数

为每个数据集定义 Chunk Size、Overlap、Top-K、Hybrid Weight、Reranker、Similarity Threshold 和 Max Context Tokens。参数按文档类型和任务类型调优，不应所有知识库使用同一组固定值。

## 9. 文档版本与新鲜度

RAG Chunk 必须关联 Document Version。文档更新后先索引新版本并验证，再将其设为 Active。旧版本可以按审计政策保留，但默认 Recall 应使用当前有效版本。

## 10. 表格与结构化内容

表格不能简单按字符切碎。需要保留列名、行上下文、单位、合并单元格关系和页码。对结构化数据，优先使用 SQL / API 查询而不是把大量表格转成文本向量。

## 11. Citation Validation

回答生成后，Citation Validator 检查引用是否存在、版本是否有效、引用内容是否支持对应断言。若引用仅相关但不支持结论，应标记为弱证据或移除该断言。

## 12. 安全测试

覆盖跨租户相似文档、权限撤销后旧向量残留、已删除文档缓存、恶意文档注入、文档版本冲突和过期政策引用。
