# Generic Multi-Domain RAG — Technical Design

## 1. Overview

This document describes the architecture for a **generic, configuration-driven RAG platform**
capable of ingesting and querying documents across independent verticals such as recruiting,
legal, and medical.

The key architectural decision is that **domains, document types, metadata schemas, extraction
rules, classification logic, guardrails, and prompt templates are all defined in external YAML
files** — not in compiled Java code. The Java layer provides a generic engine that interprets
these definitions at runtime.

Adding a new domain = adding a YAML file. Adding a new document type = adding a section in YAML.
No Java changes required.

Stack: Spring Boot 4 + LangChain4j 1.11 + PGVector.

### Child Documents

| Document | Content |
|---|---|
| [framework-code.md](./framework-code.md) | Complete Java code samples — all interfaces, engine classes, model registry, strategies, services, controllers, auto-configuration |
| [ingestion-pipeline.md](./ingestion-pipeline.md) | 10-phase ingestion flow — validation, parsing, classification, extraction, splitting, embedding, storage, dedup, error handling, concurrency model |
| [query-pipeline.md](./query-pipeline.md) | 9-phase query flow — guardrails, retrieval, hybrid re-ranking, deduplication, LLM generation, pagination, caching |
| [extraction-strategies.md](./extraction-strategies.md) | Each strategy in detail (regex, LLM, keyword, composite) with execution flows, code samples, and selection guide |
| [domain-configuration-guide.md](./domain-configuration-guide.md) | How to author domain YAML files — annotated field reference, model configuration, classification tips, guardrail authoring, validation checklist |
| [domain-definition-schema.json](./domain-definition-schema.json) | JSON Schema for validating domain YAML files |
| [examples/recruiting.yml](./examples/recruiting.yml) | Complete recruiting domain definition (6 doc_types, full metadata, guardrails, prompts) |
| [examples/legal.yml](./examples/legal.yml) | Complete legal domain definition (6 doc_types, full metadata, guardrails, prompts) |

---

## 2. Design Principles

### 2.1 Infrastructure Principles

| Principle | Rationale |
|---|---|
| Configuration over code | Domains, doc_types, extraction rules, guardrails, and prompts are YAML — not Java classes |
| Single embedding store, metadata-based isolation | Avoids operational overhead of multiple vector tables; enables future cross-domain queries |
| Shared infrastructure beans | `EmbeddingModel`, `ChatModel`, `EmbeddingStore`, and `DataSource` remain singleton Spring beans |
| Two-level metadata | Shared base keys + doc_type-specific extensions; avoids sparse flat schemas |
| Strategy pattern everywhere | Extraction, classification, guardrails, and prompts are pluggable strategies resolved from config |
| Model-per-purpose | Each domain declares which LLM to use for extraction vs. query answering; individual fields can override |

### 2.2 SOLID Mapping

| Principle | Application in this design |
|---|---|
| **S — Single Responsibility** | Each strategy class does exactly one thing. `RegexExtractionStrategy` only applies regex. `LlmExtractionStrategy` only calls the LLM. `ConfigDrivenDocumentClassifier` only classifies. `DomainIngestionService` only orchestrates — it has zero domain knowledge. `DomainDefinitionLoader` only reads YAML. |
| **O — Open/Closed** | New domain = new YAML file, no Java changes. New extraction strategy (e.g. NER-based) = one new `ExtractionStrategy` implementation, no changes to engine. New file format = one new `DocumentParser`, auto-registered. |
| **I — Interface Segregation** | `ExtractionStrategy`, `ClassificationRule`, `GuardrailRule`, `PromptTemplateProvider` are all narrow, focused interfaces. No god-interface that forces unrelated implementations. |
| **L — Liskov Substitution** | All `ExtractionStrategy` implementations are interchangeable — the engine calls `extract(text)` and gets back a string regardless of whether it used regex, LLM, or keyword matching. A `ConfigDrivenRagDomain` is fully substitutable for any handcoded `RagDomain`. |
| **D — Dependency Inversion** | `DomainIngestionService` depends on `RagDomain` (abstraction). `ConfigDrivenMetadataExtractor` depends on `ExtractionStrategy` (abstraction). No high-level module references a concrete strategy. Spring wires everything via constructor injection against interfaces. |

---

## 3. Architecture — Two Layers

The system has two distinct layers:

```mermaid
flowchart TD
    subgraph CONFIG["CONFIGURATION LAYER"]
        direction LR
        R["domains/recruiting.yml\n← defines domain, doc_types, extraction rules"]
        L["domains/legal.yml\n← defines domain, doc_types, extraction rules"]
        M["domains/medical.yml\n← defines domain, doc_types, extraction rules"]
    end

    CONFIG -- "loaded at startup" --> DDL

    subgraph ENGINE["ENGINE LAYER — Java"]
        direction TB
        DDL["DomainDefinitionLoader"] --> CDRD["ConfigDrivenRagDomain"]
        CDRD --> CDDC["ConfigDrivenDocumentClassifier"]
        CDRD --> CDMER["ConfigDrivenMetadataExtractorRegistry"]
        CDMER --> CDME["ConfigDrivenMetadataExtractor"]
        CDME --> ES["ExtractionStrategy[]"]
        ES --> RegS["RegexStrategy"]
        ES --> LlmS["LlmStrategy"]
        ES --> KeyS["KeywordStrategy"]
        ES --> CompS["CompositeStrategy"]
        CDRD --> CDGE["ConfigDrivenGuardrailEvaluator"]
        CDGE --> GR["GuardrailRule[]"]
        GR --> TBR["TermBlockRule"]
        GR --> PBR["PatternBlockRule"]
        GR --> CGR["CompositeGuardrailRule"]
        CDRD --> CDPTP["ConfigDrivenPromptTemplateProvider"]
    end
```

---

## 4. Domain Definition Schema (YAML)

Each domain is defined by a single YAML file placed in the `domains/` directory.
The engine loads all `*.yml` files from this directory at startup.

**Full YAML authoring guide:** [domain-configuration-guide.md](./domain-configuration-guide.md)

**Runnable examples:** [examples/recruiting.yml](./examples/recruiting.yml), [examples/legal.yml](./examples/legal.yml)

### 4.1 JSON Schema Validation

A formal JSON Schema is provided at [domain-definition-schema.json](./domain-definition-schema.json).
It validates every domain YAML file — structure, required fields, allowed values,
and conditional requirements (e.g. `prompt` is required when `extraction: llm`).

| Area | Validation |
|---|---|
| Domain identity | `id` must be lowercase alphanumeric with hyphens/underscores |
| File types | Each entry must start with `.` followed by lowercase alphanumeric |
| Chunk settings | `chunk-size` between 50–10000, `chunk-overlap` between 0–5000 |
| Classification rules | At least one rule required; each needs `doc-type`, `priority`, and `match` |
| Metadata fields | `key` must be lowercase snake_case; `type` must be `string|integer|float|boolean` |
| Extraction strategy | Must be `regex|llm|keyword|composite`; conditional fields enforced per strategy |
| Strategy conditions | `llm` requires `prompt`; `regex` requires `patterns`; `keyword` requires `keywords`; `composite` requires `strategies` (min 2) |
| Composite nesting | Sub-strategies cannot be `composite` (no recursive nesting) |
| Guardrail rules | `term-block` requires `trigger-terms`; `pattern-block` requires `patterns`; all require `blocked-message` |
| Prompts | Both `query` and `fallback` templates are required |

### 4.2 YAML Top-Level Structure

```yaml
domain:
  id: <string>                       # lowercase alphanumeric + hyphens
  display-name: <string>             # human-readable
  enabled: <boolean>
  documents-path: <string>           # optional, folder for batch ingestion
  supported-file-types: [<string>]   # e.g. [".pdf", ".docx"]
  chunk-size: <integer>              # 50–10000
  chunk-overlap: <integer>           # 0–5000
  entity-id-key: <string>            # optional, e.g. "candidate_id"
  stop-words: [<string>]             # optional, domain-specific noise terms

  models:                            # optional, model selection per purpose
    extraction: <string>             # model ID for metadata extraction (e.g. "gpt-4o-mini")
    query: <string>                  # model ID for answer generation (e.g. "gpt-4o")

  classification-rules: [...]        # priority-ordered rules → doc_type
  doc-types:                         # doc_type definitions + metadata fields
    <doc_type_id>:
      display-name: <string>
      metadata:
        - key: <string>
          type: string|integer|float|boolean
          extraction: regex|llm|keyword|composite
          model: <string>            # optional, per-field model override
          # ... extraction-specific fields
  guardrails:
    rules: [...]                     # term-block / pattern-block rules
  prompts:
    query: <string>                  # two %s placeholders (context, question)
    fallback: <string>               # one %d placeholder (match count)
```

### 4.3 Domain Examples Summary

**Recruiting** — 6 doc_types: resume, certification, course, degree, cover_letter, job_description.
10+ metadata fields per resume including skills, experience, location, education.
Guardrails block bias-based filtering and prompt injection.
See [examples/recruiting.yml](./examples/recruiting.yml).

**Legal** — 6 doc_types: contract, court_filing, opinion, statute, legal_memo, regulatory_notice.
8+ metadata fields per contract including parties, dates, governing law, clauses, value.
Guardrails block legal advice requests, privilege breach, and prompt injection.
See [examples/legal.yml](./examples/legal.yml).

---

## 5. Model Configuration

The platform supports **multiple LLM models** for different purposes. Models are defined
centrally in `application.yml` and referenced by ID in domain YAMLs.

### 5.1 Model Resolution Order

```text
1. Field-level 'model' key         → per-field override for specific extraction fields
2. Domain-level 'models.extraction' → default model for all LLM extraction in this domain
3. Domain-level 'models.query'      → model used for answer generation
4. App-level 'app.models.default-model' → global fallback
```

### 5.2 Typical Model Assignment

| Purpose | Model | Why |
|---|---|---|
| Metadata extraction (names, skills) | `gpt-4o-mini` | High throughput, low cost |
| Complex analysis (legal holdings) | `gpt-4o` or `deepseek-r1` | Needs reasoning |
| Answer generation (query) | `gpt-4o` | User-facing quality |
| Sensitive domains (medical) | `claude-sonnet` | Different provider for compliance |
| Air-gapped / on-prem | `llama-local` (Ollama) | No data leaves the network |

**Full model registry code and application.yml configuration:** [framework-code.md § Model Registry](./framework-code.md#2-model-registry-config)

---

## 6. Extraction Strategies

The core of the config-driven approach is the `ExtractionStrategy` interface.
Each metadata field in YAML declares an `extraction` type that maps to a strategy.

| Strategy | YAML key | How it works | When to use |
|---|---|---|---|
| `RegexExtractionStrategy` | `regex` | Tries each regex pattern in order; returns first capture group match | Dates, IDs, case numbers, monetary values — structured patterns |
| `LlmExtractionStrategy` | `llm` | Sends `prompt` + document text to the `ChatModel`; returns LLM response | Names, summaries, classifications — requires understanding |
| `KeywordExtractionStrategy` | `keyword` | Scans text for keyword lists; returns the matching category key | Enum-like fields (status, level, work mode) — fast, no LLM cost |
| `CompositeExtractionStrategy` | `composite` | Tries strategies in order; returns first non-null result | Location-like fields where regex might work but LLM is fallback |

**Full strategy documentation with execution flows:** [extraction-strategies.md](./extraction-strategies.md)

**Java code for all strategies:** [framework-code.md § Extraction Strategies](./framework-code.md#4-extraction-strategies-featuredomainenginestrategy)

---

## 7. Engine Class Diagram

```mermaid
classDiagram
    class RagDomain {
        <<interface>>
        +domainId()
        +supportedFileTypes()
        +documentSplitter()
        +documentClassifier()
        +metadataExtractors()
        +guardrailEvaluator()
        +promptProvider()
        +stopWords()
    }

    class ConfigDrivenRagDomain {
        built from YAML at startup
    }

    class HandcodedRagDomain {
        optional — for complex domains
        that need custom Java logic
    }

    RagDomain <|.. ConfigDrivenRagDomain : implements
    RagDomain <|.. HandcodedRagDomain : implements

    class ConfigDrivenDocumentClassifier {
        reads classification-rules from YAML
        implements DocumentClassifier
    }

    class ConfigDrivenMetadataExtractorRegistry {
        one ConfigDrivenMetadataExtractor per doc-type
        implements MetadataExtractorRegistry
    }

    class ConfigDrivenMetadataExtractor {
        holds ExtractionStrategy[] per field
    }

    class ConfigDrivenGuardrailEvaluator {
        holds GuardrailRule[] from YAML
        implements GuardrailEvaluator
    }

    class ConfigDrivenPromptTemplateProvider {
        reads prompt templates from YAML
        implements PromptTemplateProvider
    }

    ConfigDrivenRagDomain *-- ConfigDrivenDocumentClassifier : has-a
    ConfigDrivenRagDomain *-- ConfigDrivenMetadataExtractorRegistry : has-a
    ConfigDrivenRagDomain *-- ConfigDrivenGuardrailEvaluator : has-a
    ConfigDrivenRagDomain *-- ConfigDrivenPromptTemplateProvider : has-a
    ConfigDrivenMetadataExtractorRegistry *-- ConfigDrivenMetadataExtractor
    ConfigDrivenMetadataExtractor *-- ExtractionStrategy

    class ExtractionStrategy {
        <<interface>>
    }
    class RegexExtractionStrategy
    class LlmExtractionStrategy
    class KeywordExtractionStrategy
    class CompositeExtractionStrategy

    ExtractionStrategy <|.. RegexExtractionStrategy
    ExtractionStrategy <|.. LlmExtractionStrategy
    ExtractionStrategy <|.. KeywordExtractionStrategy
    ExtractionStrategy <|.. CompositeExtractionStrategy

    class DocumentParser {
        <<interface>>
    }
    class PdfDocumentParser
    class DocxDocumentParser
    class PlainTextParser

    DocumentParser <|.. PdfDocumentParser
    DocumentParser <|.. DocxDocumentParser
    DocumentParser <|.. PlainTextParser

    class GuardrailRule {
        <<interface>>
    }
    class TermBlockGuardrailRule {
        ~0ms, free
        known bad terms
    }
    class PatternBlockGuardrailRule {
        ~0ms, free
        structural regex attacks
    }
    class LlmGuardrailRule {
        200-2000ms, per-call
        nuanced safety via LLM
    }

    GuardrailRule <|.. TermBlockGuardrailRule
    GuardrailRule <|.. PatternBlockGuardrailRule
    GuardrailRule <|.. LlmGuardrailRule
    ConfigDrivenGuardrailEvaluator *-- GuardrailRule
```

The escape hatch: if a domain's logic is too complex for YAML (e.g. the existing recruiting
domain with its `TechnicalRoleCatalog` and candidate-aware ranking), it can implement
`RagDomain` directly in Java. The engine treats `ConfigDrivenRagDomain` and handcoded
implementations identically — Liskov substitution.

---

## 8. Shared Base Metadata (All Domains, All Doc Types)

| Key | Type | Description |
|---|---|---|
| `domain` | `string` | Domain identifier. **Mandatory partition key.** |
| `doc_type` | `string` | Document classification. **Discriminator for extension keys.** |
| `source` | `string` | Original filename |
| `content_hash` | `string` | SHA-256 of normalized text (deduplication) |
| `ingested_at` | `string` | ISO-8601 timestamp of ingestion |
| `language` | `string` | Detected language (ISO 639-1) |
| `entity_id` | `string` | Domain-level entity linkage (`candidate_id`, `matter_id`, etc.) |
| `segment_index` | `integer` | Ordering of segments within a document |

Extension metadata keys per doc_type are defined entirely in the domain YAML.

---

## 9. Ingestion Flow

```mermaid
flowchart TD
    API["POST /api/v1/{domainId}/ingest\n📄 DomainIngestController"] --> SVC

    subgraph SVC["DomainIngestionService — orchestrator, zero domain logic"]
        direction TB

        subgraph ACCEPT["🔒 ACCEPT & VALIDATE"]
            A1["Resolve domain\nDomainRegistry.get(domainId)"]
            A2["Validate file type\nagainst domain.supported-file-types"]
            A3["Dedup gate\nConcurrentHashMap⟨hash, Future⟩"]
            A1 --> A2 --> A3
        end

        subgraph PARSE["📄 PARSE & SANITIZE"]
            B1["Select parser\nDocumentParserRegistry\nPDF → PDFBox | DOCX → POI | TXT → UTF-8"]
            B2["Extract text\nparser.extractText(bytes)"]
            B3["Sanitize\nnull bytes · Unicode NFC · whitespace"]
            B1 --> B2 --> B3
        end

        subgraph ENRICH["🧠 CLASSIFY & EXTRACT — config-driven from YAML"]
            C1["Classify doc_type\nConfigDrivenDocumentClassifier\nrules sorted by priority, first match wins"]
            C2["Extract metadata\nConfigDrivenMetadataExtractor\nper-field: regex → llm → keyword → composite"]
            C3["Resolve entity\ncross-doc linkage by name / hash / ID"]
            C1 --> C2 --> C3
        end

        subgraph STORE["💾 SPLIT · EMBED · STORE"]
            D1["Split text → segments\nDocumentSplitters.recursive\nchunk-size & chunk-overlap from YAML"]
            D2["Embed segments → vectors\nEmbeddingModel\nbatched, virtual threads"]
            D3["Store in PGVector\nEmbeddingStoreIngestor\nbase + extension metadata as JSONB"]
            D1 --> D2 --> D3
        end

        subgraph AUDIT["📊 AUDIT & REPORT"]
            E1["Emit SSE progress\nfile-ingested / file-skipped"]
            E2["Record metrics\nprocessed · skipped · elapsed"]
            E1 --> E2
        end

        ACCEPT --> PARSE --> ENRICH --> STORE --> AUDIT
    end

    YAML[("domain YAML\nclassification-rules\ndoc-types · metadata\nchunk-size · chunk-overlap\nmodels.extraction")] -.->|configures| ENRICH
    YAML -.->|configures| STORE
    MODELS[("application.yml\napp.models.definitions\ngpt-4o-mini · gpt-4o · ...")] -.->|model per field| ENRICH
```

**Full 10-phase detailed design:** [ingestion-pipeline.md](./ingestion-pipeline.md)

Covers: per-file error isolation, LLM field batching, concurrent virtual thread model,
re-ingestion (delete + re-embed), entity resolution strategies, deduplication gates,
SSE progress streaming, audit trail structure.

---

## 10. Query Flow

```mermaid
flowchart TD
    A["query(domainId, question, ?docTypes,\n?maxResults, ?minScore, ?page, ?pageSize)"] --> B["DomainQueryService\n(orchestrator — zero domain logic)"]
    B --> S1["1. Validate\ndomain lookup, param clamping"]
    S1 --> S2["2. Guardrails\nYAML rules, first block wins"]
    S2 --> S3["3. Normalize\nstop words, term extraction"]
    S3 --> S4["4. Retrieve\nvector search with metadata filters"]
    S4 --> S5["5. Hybrid re-rank\nvector × 0.8 + keyword × 0.2"]
    S5 --> S6["6. Deduplicate\none result per entity/source"]
    S6 --> S7["7. Generate answer\nLLM with domain prompt template"]
    S7 --> S8["8. Paginate\npage slicing"]
    S8 --> S9["9. Respond\nstructured JSON + explainability"]
```

**Full 9-phase detailed design:** [query-pipeline.md](./query-pipeline.md)

Covers: guardrail evaluation with examples, hybrid scoring formula, deduplication key
resolution, LLM fallback, in-flight query deduplication, LRU caching, explainability fields.

---

## 11. Embedding Store Schema

```sql
CREATE TABLE IF NOT EXISTS document_embeddings (
    embedding_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    embedding      vector(1536),
    text           text NOT NULL,
    metadata       jsonb NOT NULL
);

CREATE INDEX idx_embeddings_domain ON document_embeddings
    USING btree ((metadata->>'domain'));

CREATE INDEX idx_embeddings_doc_type ON document_embeddings
    USING btree ((metadata->>'doc_type'));

CREATE INDEX idx_embeddings_ivfflat ON document_embeddings
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
```

---

## 12. REST API Surface

```text
POST   /api/v1/{domainId}/ingest                     Upload documents
POST   /api/v1/{domainId}/ingest/folder               Batch ingest from folder
POST   /api/v1/{domainId}/ingest/stream               Upload with SSE progress
POST   /api/v1/{domainId}/query                       Query with optional filters
GET    /api/v1/{domainId}/doc-types                    List supported doc_types (from YAML)
GET    /api/v1/domains                                 List registered domains (from YAML)
GET    /api/v1/{domainId}/stats                        Domain-level metrics
DELETE /api/v1/{domainId}/documents/{source}            Remove document and its segments
POST   /api/v1/admin/domains/reload                    Hot-reload domain YAML definitions
```

---

## 13. Project Structure

```text
generic-rag-poc/                                     ── DOCUMENTATION ──
├── technical-design.md                               Master architecture document (this file)
├── framework-code.md                                 All Java code samples
├── ingestion-pipeline.md                             Detailed ingestion flow
├── query-pipeline.md                                 Detailed query flow
├── extraction-strategies.md                          Strategy details + execution flows
├── domain-configuration-guide.md                     YAML authoring guide + model configuration
├── domain-definition-schema.json                     JSON Schema for YAML validation
└── examples/
    ├── recruiting.yml                                Complete recruiting domain
    └── legal.yml                                     Complete legal domain

be/
├── src/main/java/com/example/rag/
│   │
│   ├── config/
│   │   ├── LangChain4jConfig.java                     Shared embedding beans
│   │   ├── PostgresEmbeddingStoreConfig.java           Shared PGVector store
│   │   ├── ModelDefinitionProperties.java              Model entries from application.yml
│   │   ├── ModelRegistry.java                          Resolves model ID → ChatModel
│   │   └── DomainAutoConfiguration.java                Loads YAML, builds DomainRegistry
│   │
│   ├── feature/
│   │   │
│   │   ├── domain/                                    ── ABSTRACTIONS ──
│   │   │   ├── RagDomain.java                          Top-level domain interface
│   │   │   ├── DomainModelConfig.java                  Record: extraction/query model IDs
│   │   │   ├── DomainRegistry.java                     Map<String, RagDomain>
│   │   │   ├── DocumentClassifier.java                 Interface (S)
│   │   │   ├── MetadataExtractor.java                  Interface (S)
│   │   │   ├── MetadataExtractorRegistry.java          Interface (O)
│   │   │   ├── GuardrailEvaluator.java                 Interface (S)
│   │   │   ├── GuardrailDecision.java                  Record
│   │   │   ├── GuardrailRule.java                      Interface (S)
│   │   │   ├── PromptTemplateProvider.java             Interface (S)
│   │   │   └── ExtractionStrategy.java                 Interface (S)
│   │   │
│   │   ├── domain.engine/                             ── CONFIG-DRIVEN ENGINE ──
│   │   │   ├── DomainDefinitionLoader.java             Reads YAML, builds RagDomain instances
│   │   │   ├── ConfigDrivenRagDomain.java              Implements RagDomain from YAML
│   │   │   ├── ConfigDrivenDocumentClassifier.java     Classification rules from YAML
│   │   │   ├── ConfigDrivenMetadataExtractor.java      Field extraction from YAML strategies
│   │   │   ├── ConfigDrivenMetadataExtractorRegistry.java
│   │   │   ├── ConfigDrivenGuardrailEvaluator.java     Guardrail rules from YAML
│   │   │   └── ConfigDrivenPromptTemplateProvider.java  Prompt templates from YAML
│   │   │
│   │   ├── domain.engine.strategy/                    ── EXTRACTION STRATEGIES ──
│   │   │   ├── ExtractionStrategyFactory.java          Resolves strategy by YAML key
│   │   │   ├── RegexExtractionStrategy.java            (S) regex only
│   │   │   ├── LlmExtractionStrategy.java              (S) LLM only
│   │   │   ├── KeywordExtractionStrategy.java          (S) keyword matching only
│   │   │   └── CompositeExtractionStrategy.java        (S) chain of strategies
│   │   │
│   │   ├── domain.engine.guardrail/                   ── GUARDRAIL RULES ──
│   │   │   ├── GuardrailRuleFactory.java               Resolves rule by YAML type
│   │   │   ├── TermBlockGuardrailRule.java             (S) term + intent matching
│   │   │   ├── PatternBlockGuardrailRule.java          (S) regex pattern matching
│   │   │   └── LlmGuardrailRule.java                  (S) LLM-based safety check
│   │   │
│   │   ├── domain.recruiting/                         ── OPTIONAL HANDCODED OVERRIDE ──
│   │   │   ├── RecruitingDomainOverride.java           Extends config with TechnicalRoleCatalog
│   │   │   └── TechnicalRoleCatalog.java               Skill/role normalization (too complex for YAML)
│   │   │
│   │   ├── ingest/
│   │   │   ├── parser/
│   │   │   │   ├── DocumentParser.java                 Interface (S)
│   │   │   │   ├── DocumentParserRegistry.java         Composite dispatcher (O)
│   │   │   │   ├── PdfDocumentParser.java
│   │   │   │   ├── DocxDocumentParser.java
│   │   │   │   └── PlainTextParser.java
│   │   │   ├── controller/
│   │   │   │   └── DomainIngestController.java
│   │   │   └── service/
│   │   │       └── DomainIngestionService.java         Orchestrator (S, D)
│   │   │
│   │   ├── query/
│   │   │   ├── controller/
│   │   │   │   └── DomainQueryController.java
│   │   │   └── service/
│   │   │       └── DomainQueryService.java             Orchestrator (S, D)
│   │   │
│   │   └── metrics/
│   │
│   └── src/main/resources/
│       ├── application.yml
│       └── domains/                                   ── DOMAIN DEFINITIONS ──
│           ├── domain-definition-schema.json            JSON Schema for validation
│           ├── recruiting.yml
│           ├── legal.yml
│           └── medical.yml                             (enabled: false until ready)
```

---

## 14. Configuration

All domain-specific configuration lives in the YAML files under `domains/`.
The application YAML handles infrastructure and model definitions:

```yaml
app:
  domains:
    definitions-path: ${DOMAIN_DEFINITIONS_PATH:classpath:domains/}
    hot-reload-enabled: ${DOMAIN_HOT_RELOAD:false}

  models:
    default-model: "gpt-4o-mini"              # global fallback
    embedding: "text-embedding-3-small"
    definitions:
      gpt-4o:
        provider: openai
        api-key: ${OPENAI_API_KEY}
        model-name: gpt-4o
        temperature: 0.1
        max-tokens: 4096
        timeout-seconds: 60
      gpt-4o-mini:
        provider: openai
        api-key: ${OPENAI_API_KEY}
        model-name: gpt-4o-mini
        temperature: 0.1
        max-tokens: 2048
        timeout-seconds: 30
      claude-sonnet:
        provider: openrouter
        api-key: ${OPENROUTER_API_KEY}
        model-name: anthropic/claude-3.5-sonnet
        temperature: 0.1
        max-tokens: 4096
        timeout-seconds: 60
```

**Full model registry code:** [framework-code.md § Model Registry](./framework-code.md#2-model-registry-config)

Ingestion and query settings are documented in:
- [ingestion-pipeline.md § Configuration Reference](./ingestion-pipeline.md#16-configuration-reference)
- [query-pipeline.md § Configuration Reference](./query-pipeline.md#13-configuration-reference)

---

## 15. Migration Path

| Step | Action | SOLID | Risk |
|---|---|---|---|
| 1 | Define `RagDomain`, `ExtractionStrategy`, `GuardrailRule`, and all interfaces | I, D | None — additive |
| 2 | Implement engine: `DomainDefinitionLoader`, `ConfigDriven*` classes, strategy impls | S, O, D | Low |
| 3 | Write `recruiting.yml` and `legal.yml` domain definitions | — | None |
| 4 | Implement `ExtractionStrategyFactory` + `RegexStrategy`, `LlmStrategy`, `KeywordStrategy`, `CompositeStrategy` | S, O | Low |
| 5 | Implement `GuardrailRuleFactory` + `TermBlockRule`, `PatternBlockRule` | S, O | Low |
| 6 | Wrap existing recruiting logic as `RecruitingDomainOverride` for complex features (TechnicalRoleCatalog) | L | Low |
| 7 | Add `domain` + `doc_type` metadata to all existing segments (backfill migration) | — | Medium |
| 8 | Create `DomainIngestionService` + `DomainQueryService` orchestrators | D | Low |
| 9 | Add `/api/v1/{domainId}/*` endpoints | O | None — additive |
| 10 | Add `DocxDocumentParser` and optional Tika fallback | O | None |
| 11 | Add hot-reload endpoint for YAML changes without restart | — | Low |
| 12 | Deprecate old `/api/ingest` and `/api/query` endpoints | — | Low |

---

## 16. Dependencies to Add

```groovy
// DOCX support
implementation 'org.apache.poi:poi-ooxml:5.3.0'

// Optional: Tika catch-all parser (1000+ formats)
implementation 'dev.langchain4j:langchain4j-document-parser-apache-tika'
```

---

## 17. Security Considerations

| Concern | Mitigation |
|---|---|
| PII in recruiting metadata | `candidate_email`, `candidate_phone` never logged; mask in error messages |
| Attorney-client privilege | `confidentiality_level` field + guardrail blocks privileged segments |
| Cross-domain data leakage | Mandatory `domain` filter on every query |
| Guardrail bypass via YAML edit | YAML files should be version-controlled; hot-reload requires admin auth |
| LLM extraction hallucination | Composite strategy tries regex first; LLM is fallback with constrained prompts |
| Prompt injection | `pattern-block` guardrail rules applied before every query |
| Secret exfiltration | `pattern-block` guardrail rules match credential-related patterns |

---

## 18. Extensibility Checklist

### Adding a new domain (e.g. medical)

1. Create `domains/medical.yml` following the [domain configuration guide](./domain-configuration-guide.md)
2. Define `classification-rules`, `doc-types` with metadata fields, `guardrails`, and `prompts`
3. Validate against the [JSON Schema](./domain-definition-schema.json)
4. Set `enabled: true`
5. Restart (or call `/api/v1/admin/domains/reload` if hot-reload is enabled)
6. **Zero Java changes**

### Adding a new doc_type to an existing domain

1. Add a new entry under `classification-rules` in the domain YAML
2. Add the doc_type definition under `doc-types` with its metadata fields
3. Each field specifies an `extraction` strategy (`regex`, `llm`, `keyword`, or `composite`)
4. Restart or hot-reload
5. **Zero Java changes**

### Adding a new extraction strategy

1. Implement `ExtractionStrategy` interface
2. Register the YAML key in `ExtractionStrategyFactory`
3. Use it in any domain YAML: `extraction: <new-key>`
4. **No changes to engine, loader, or existing strategies**

See [extraction-strategies.md § Adding a Custom Strategy](./extraction-strategies.md#7-adding-a-custom-strategy)

### Adding a new guardrail rule type

1. Implement `GuardrailRule` interface
2. Register the YAML key in `GuardrailRuleFactory`
3. Use it in any domain YAML: `type: <new-key>`
4. **No changes to engine, loader, or existing rules**

### Adding a new file format parser

1. Implement `DocumentParser` interface, annotate with `@Component`
2. `DocumentParserRegistry` auto-discovers it
3. Add the file extension to `supported-file-types` in any domain YAML
4. **No changes to engine or existing parsers**

### Overriding config with handcoded Java (escape hatch)

1. Implement `RagDomain` directly in Java (e.g. `RecruitingDomainOverride`)
2. Annotate with `@Component`
3. Remove or set `enabled: false` in the corresponding YAML
4. The `DomainRegistry` accepts both config-driven and handcoded domains
5. **Engine treats them identically (Liskov substitution)**
