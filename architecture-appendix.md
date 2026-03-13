# Architecture Appendix — Operational Gaps to Address

This appendix captures confirmed risks that remain open. Each item lists the issue and the recommended action path.

1. Single-table scalability (domain isolation)
   - Issue: One `document_embeddings` table with a JSONB `domain` filter and a B-tree index will degrade with millions of rows.
   - Action: Partition the table by domain (`PARTITION BY LIST (metadata->>'domain')`) and ensure the primary/unique keys include the partition key. Add migration guidance for existing data.

2. OpenRouter rate limiting vs virtual-thread ingest
   - Issue: Ingest and query share one `OPENROUTER_API_KEY`; virtual-thread ingest can burst requests and exhaust shared limits, impacting query latency.
   - Action: Use separate HTTP clients/connection pools and API keys or rate-limiters for ingest vs query. Add bounded ingestion concurrency (queue or token bucket) independent of virtual-thread count.

3. Embedding dimension changes (model drift)
   - Issue: Schema is fixed to `vector(1536)` while dev uses 384-dim; switching embedding models in prod would break inserts.
   - Action: Define a migration pattern: versioned embedding tables (e.g., `document_embeddings_v2`) or dimension-agnostic staging, plus re-embed/re-index playbook when models change.

4. Persistence framework coupling
   - Issue: Persistence is bound to LangChain4j `EmbeddingStoreIngestor` + PGVector; moving to another vector store requires a rewrite.
   - Action: Introduce a thin persistence port (interface) with PGVector as one adapter, and document the contract (store, retrieve, delete, filter) so future backends (e.g., Milvus/Pinecone) can plug in without touching pipeline logic.
