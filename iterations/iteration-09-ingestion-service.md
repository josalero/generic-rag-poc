# Iteration 9 — Ingestion service

> Parent: [implementation-plan.md](../implementation-plan.md) · Previous: [iteration-08-llm-guardrail.md](./iteration-08-llm-guardrail.md)

## Goal

Orchestrate the full ingestion flow: accept, parse, sanitize, classify, extract metadata, split, embed, store. Dependencies: `RagDomain`, parsers, embedding model, embedding store.

## Deliverables

| # | Component | Location |
|---|-----------|----------|
| 1 | `DomainIngestionService` | [framework-code.md § 7.1](../framework-code.md#71-domainingestionservice) |
| 2 | `DomainNotFoundException`, `UnsupportedFileTypeException` | § Code 7.2 below |

## Acceptance criteria

- Service receives `domainId`, `filename`, `bytes`; resolves domain from registry; validates file type; selects parser; parses and sanitizes; classifies; extracts metadata; splits with domain's splitter; embeds; stores in embedding store. Per-file errors do not abort batch when multiple files.
- **Ingestion is executed on virtual threads:** Batch and folder ingest run each file (and optionally each embedding task) on a virtual thread (e.g. `Executors.newVirtualThreadPerTaskExecutor()`), configurable via `app.ingest.virtual-threads-enabled` (default true). See [ingestion-pipeline.md § 1 Concurrency](../ingestion-pipeline.md#concurrency-virtual-threads).
- Sanitization: null bytes, Unicode NFC, whitespace (as in ingestion-pipeline.md).
- Re-ingestion: delete existing segments by source filename before storing new ones.
- **Repeat runs (tracking + hash-based skip):** When the process is run multiple times (e.g. folder ingest on a schedule), track what was ingested (e.g. ledger or store with `source` and `content_hash`). After parse/sanitize, compute content hash; if a record exists for the same `(domain_id, source)` with the **same** hash, skip extract/split/embed/store and record skip (e.g. reason `"Unchanged (same content hash)"`) to save resources. Only new sources or updated content (different hash) go through full pipeline. See [implementation-plan.md § 11.2.1](../implementation-plan.md#1121-running-ingestion-multiple-times-tracking--hash-based-skip) and [ingestion-pipeline.md § 13](../ingestion-pipeline.md#13-deduplication).

## Tests to add

- `DomainIngestionServiceTest`: with mocked `DomainRegistry` (one domain), mocked parser (returns fixed text), mocked embedder and store; call ingest one file; verify store invoked with expected metadata (domain, doc_type, source), and segment count or store call count. Test unknown domain throws; unsupported file type throws.
- **Repeat run / hash-based skip:** when the same file (same source, same content hash) is ingested twice, the second run skips extract/embed/store (e.g. returns or records "skipped — Unchanged (same content hash)") and does not call the embedder or store again. When the same source has different content (different hash), full re-ingestion occurs (delete old segments, then store new).
- Optional: integration test with test profile, in-memory or Testcontainers PGVector, real parser and embedder (or stub embedder), one small PDF/DOCX.

## Quality gates

- Unit tests use mocks only; no real DB or API in unit tests; build passes.

See [iteration-00-quality-gates.md](./iteration-00-quality-gates.md) for gates that apply to every iteration.

---

## Code — Exception classes (§ 7.2)

```java
package com.example.rag.feature.ingest.service;

public class DomainNotFoundException extends RuntimeException {
    public DomainNotFoundException(String domainId) {
        super("Domain not found: '%s'".formatted(domainId));
    }
}
```

```java
package com.example.rag.feature.ingest.service;

import java.util.List;

public class UnsupportedFileTypeException extends RuntimeException {
    public UnsupportedFileTypeException(String filename, List<String> supported) {
        super("File '%s' not supported. Accepted: %s".formatted(filename, supported));
    }
}
```

Full `DomainIngestionService` (parse, sanitize, classify, extract, split, embed, store, batch with virtual threads, dedup gate, `IngestResult`): [framework-code.md § 7.1](../framework-code.md#71-domainingestionservice).
