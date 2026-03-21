# Leather inventory agent (separate POC)

This folder holds the **leather store inventory + agentic chat** design — a **different product** from the **generic multi-domain RAG** platform documented at the repository root.

| Document | Description |
|----------|-------------|
| [**diseno-agente-consultas-leather-openrouter.md**](./diseno-agente-consultas-leather-openrouter.md) | Full technical design: PostgreSQL + pgvector, `searchCatalogRag`, OpenRouter, LangChain4j Agentic, BFF widget, demo kit (§19) |

**Related (repo root — generic RAG):**

- [implementation-plan.md § 18](../implementation-plan.md#18-leather-inventory-agent-poc) — leather POC checklist, tests, demo kit mirrors, mapping to generic iterations 9–10
- [technical-design.md](../technical-design.md), [ingestion-pipeline.md](../ingestion-pipeline.md), [query-pipeline.md](../query-pipeline.md), [framework-code.md](../framework-code.md) — patterns this POC reuses for embeddings and retrieval
