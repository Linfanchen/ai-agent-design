# RAG and Knowledge Retrieval

## Role in the Agent System

RAG supplies grounded information from documents and knowledge sources that are too large, dynamic, or numerous to place directly in a prompt. It complements Memory and Business Catalog but does not replace either.

## 1. Distinguish the three knowledge systems

- Memory: user-, project-, or agent-specific information retained across tasks.
- RAG: searchable document and knowledge corpus.
- Business Catalog: structured metadata about software entities, APIs, permissions, and workflows.

## 2. Ingestion pipeline

Source connectors → file validation → malware/content scan → text and layout extraction → normalization → metadata enrichment → chunking → embedding → indexing → quality checks → publish.

Keep source file, source version, owner, tenant, ACL, classification, ingestion timestamp, and content hash.

## 3. Chunking

Choose chunking by content type:
- prose: semantic/heading-aware chunks;
- manuals: section and procedure boundaries;
- code: symbol/function/class boundaries;
- tables: preserve row/column context;
- PDFs: preserve page and bounding-box metadata where useful;
- transcripts: speaker and time-window boundaries.

Do not apply one fixed chunk size to every corpus.

## 4. Retrieval pipeline

1. Parse query and intent.
2. Apply tenant and document ACL filters before retrieval.
3. Generate lexical and semantic candidates.
4. Merge and deduplicate.
5. Rerank.
6. Expand neighboring chunks when context is incomplete.
7. Apply freshness and trust filters.
8. Return citations and source metadata.
9. Enforce token budget.

## 5. Grounding

Every retrieved item should carry source ID, title, URL or file reference, page/section, version, timestamp, and confidence. Final responses should cite claims when the product requires traceability.

## 6. Evaluation

Build a dataset with expected documents, expected passages, answerable/unanswerable questions, ACL adversarial cases, stale-document cases, and conflicting-source cases. Measure Recall@K, Precision@K, MRR/nDCG, groundedness, citation correctness, and latency.

## 7. Security and freshness

Retrieval filters must enforce ACL, not just post-filter results. Revoke access promptly when source permissions change. Use source versioning and re-indexing; stale indexes must not be mistaken for current business facts.
