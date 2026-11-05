# LlamaIndex 源码架构拆解（首轮）

> 当前仓库含约 9,798 个 Markdown/MDX，且 README 声明 OSS Framework 仍可用，但公司重心已转向 LlamaParse/LiteParse 和文档理解/抽取。本文先从核心架构与源码包布局建图；未逐篇阅读全部版本化 docs、集成包文档。

## 定位

LlamaIndex 的核心价值是把非结构化/半结构化数据接入 LLM 应用：读取与解析 Document → 切分为 Node → embedding/index/storage → retriever 检索 → response synthesizer/query engine 生成答案。其 Agent 能力建立在数据工具、检索器和模型组件之上；不应只把它理解成通用多 Agent orchestrator。

## 架构分层

- **Core abstractions**：`llama-index-core/llama_index/core/` 提供 Document/Node、Index、Retriever、QueryEngine、Storage、Prompt、LLM、Workflow 等基础协议与默认实现。
- **Ingestion**：Reader/transformations 将原始数据转换成 Documents/Nodes；常用 transformations 做 metadata、chunk、embedding。
- **Index/storage**：Index 协调构建和查询；StorageContext 组合 docstore/index store/vector store；集成包把向量 DB、文件源和模型接到统一接口。
- **Retrieval/query**：Retriever 返回 NodeWithScore；QueryEngine 组合 Retriever 与 ResponseSynthesizer；Agent 可以把查询/索引能力封装成工具。
- **Integrations**：大量独立 `llama-index-*` packages，按 provider/vector store/readers 等拆包；core 与 integrations 的发布/依赖边界是关键。
- **Workflow**：现代异步事件驱动工作流，提供 step/event/context/resource/retry 等原语，适合有多个阶段的数据/Agent 流程。

## 典型 RAG 数据流

`Reader → Document → transformations/chunking → Node + embeddings → Index/VectorStore`；查询时 `query → Retriever → scored nodes → postprocessors → response synthesizer + LLM → Response`。抽象层允许更换 vector store/retriever/model，而不重写业务查询逻辑。需要注意 ingestion、indexing、retrieval 不是同一个对象生命周期，持久化具体由 StorageContext 和所选 integration 决定。

## 关键源码路径

- `llama-index-core/llama_index/core/indices/`：BaseIndex 与 vector/list 等索引实现。
- `.../retrievers/`：检索抽象、合并/组合检索器。
- `.../query_engine/retriever_query_engine.py`：检索到回答合成的默认桥接。
- `.../response_synthesizers/`：上下文组织、响应生成模式。
- `.../ingestion/`、`.../readers/`、`.../node_parser/`：文档摄取和切分链路。
- `.../storage/`、`.../vector_stores/`：抽象存储接口与内建实现。
- `.../workflow/`：事件驱动 Workflow API。
- `llama-index-integrations/`：provider 和连接器生态；这些不是 core 内部代码。
- `llama-dev/`：monorepo 包管理、发布和测试工具。

## 设计取舍

1. **高层组件组合 + 插件式集成**降低连接器替换成本，但包数量多、版本兼容和 API 演进复杂。
2. **Node/Document 抽象**统一不同来源的内容及 metadata，利于检索；解析质量决定下游效果，当前产品策略明显加强文档解析/抽取。
3. **Index/Retriever/QueryEngine 分离**让 indexing 与 query 策略可替换，也容易造成初学者对象层级过多，需要沿数据流理解而非死记类名。
4. **Workflow 与 Index/RAG 并行发展**：工作流适合事件驱动任务，传统 query engine 更偏组件管线；二者职责不应混淆。

## 覆盖账本

- 已读根 README 的定位变化；查看 core 顶层包、workflow、vector store、indices、query engine 源码位置并确认主要类。
- 未完成：9,798 Markdown/MDX 逐篇覆盖；尚未精读 ingestion pipeline 实现、StorageContext 持久化语义、VectorStoreIndex 插入/查询过程、Workflow 调度代码及集成版本兼容策略。
