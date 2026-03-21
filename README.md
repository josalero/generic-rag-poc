# Generic Multi-Domain RAG — Design & Implementation

A **configuration-driven RAG platform** for ingesting and querying documents across independent domains (e.g. recruiting, legal). All design and implementation steps are documented here; code is built iteratively with quality gates.

---

## Documentation

| Document | Description |
|----------|--------------|
| [technical-design.md](./technical-design.md) | Master architecture: domain YAML, ingestion/query flow, API, embedding store, languages (en/es), ingestion ledger & classification-help |
| [implementation-plan.md](./implementation-plan.md) | Testable iterations, quality gates, required capabilities (R1–R7) tracking table, optional iterations (A–D) |
| [framework-code.md](./framework-code.md) | Java code samples and interfaces for each iteration |
| [ingestion-pipeline.md](./ingestion-pipeline.md) | Ingestion phases (accept → parse → classify → extract → split → embed → store), ledger, preflight |
| [query-pipeline.md](./query-pipeline.md) | Query flow: term extraction, retrieval, rerank, answer generation |
| [extraction-strategies.md](./extraction-strategies.md) | Regex, keyword, composite, LLM strategies and execution |
| [domain-configuration-guide.md](./domain-configuration-guide.md) | How to author domain YAML (classification, metadata, guardrails, prompts) |
| [model-recommendations.md](./model-recommendations.md) | OpenRouter models for prod/dev; embedding and chat |
| [domain-definition-schema.json](./domain-definition-schema.json) | JSON Schema for domain YAML validation |
| [examples/](./examples/) | Example domain YAMLs: `recruiting.yml`, `legal.yml` |

### Separate product: leather inventory agent

The **leather store** agentic chat POC (PostgreSQL + pgvector, OpenRouter, embeddable widget) is **not** the generic RAG platform. Its design and iterations live in **[`leather-inventory-agent/`](./leather-inventory-agent/README.md)**. Engineering handoff checklists still appear in [implementation-plan § 18](./implementation-plan.md#18-leather-inventory-agent-poc).

---

## Implementation Steps

Each step has a **dedicated iteration doc** under [iterations/](./iterations/) with goal, deliverables, acceptance criteria, tests, and code. The [implementation-plan](./implementation-plan.md) is the single index and defines quality gates for all steps.

### Quality gates (apply to every iteration)

| Doc | Content |
|-----|--------|
| [iteration-00-quality-gates.md](./iterations/iteration-00-quality-gates.md) | Build, test, coverage, lint, no PII, meaningful assertions; Definition of Done |

### Core iterations (1–13)

| # | Iteration | Document |
|---|-----------|----------|
| 1 | Foundation (interfaces, records, registry) | [iteration-01-foundation.md](./iterations/iteration-01-foundation.md) |
| 2 | Config & model registry | [iteration-02-config-model-registry.md](./iterations/iteration-02-config-model-registry.md) |
| 3 | Extraction strategies (regex, keyword, composite) | [iteration-03-extraction-strategies.md](./iterations/iteration-03-extraction-strategies.md) |
| 4 | Guardrail rules (term, pattern) | [iteration-04-guardrail-rules.md](./iterations/iteration-04-guardrail-rules.md) |
| 5 | Document parsers (PDF, DOCX, TXT, fallback) | [iteration-05-document-parsers.md](./iterations/iteration-05-document-parsers.md) |
| 6 | Config-driven engine (YAML loader, classifier, prompts, guardrails) | [iteration-06-config-driven-engine.md](./iterations/iteration-06-config-driven-engine.md) |
| 7 | Config-driven metadata + LLM strategy | [iteration-07-config-driven-metadata-llm.md](./iterations/iteration-07-config-driven-metadata-llm.md) |
| 8 | LLM guardrail rule | [iteration-08-llm-guardrail.md](./iterations/iteration-08-llm-guardrail.md) |
| 9 | Ingestion service (orchestrator, 10 phases) | [iteration-09-ingestion-service.md](./iterations/iteration-09-ingestion-service.md) |
| 10 | Query service | [iteration-10-query-service.md](./iterations/iteration-10-query-service.md) |
| 11 | REST controllers (ingest, query, doc-types, domains, stats) | [iteration-11-rest-controllers.md](./iterations/iteration-11-rest-controllers.md) |
| 12 | Auto-configuration & wiring (prod/dev profiles) | [iteration-12-auto-configuration-wiring.md](./iterations/iteration-12-auto-configuration-wiring.md) |
| 13 | Production hardening (exception handling, validation, health) | [iteration-13-production-hardening.md](./iterations/iteration-13-production-hardening.md) |

**Required capabilities (R1–R7):** Ingestion ledger, dashboard/ledger endpoint, store LLM reasoning, LLM classification fallback, preflight, virtual threads, and hash-based skip are **required** (delivered in iterations 6, 9, 11). See the [implementation-plan Required capabilities — tracking table](./implementation-plan.md#required-capabilities--tracking-table).

### Optional iterations (after core 13)

| Id | Goal | Reference |
|----|------|-----------|
| **A** | Handcoded domain override | implementation-plan § 17 |
| **B** | Feedback API (HITL: query + ingestion) | [technical-design § 19](./technical-design.md#19-human-in-the-loop-and-feedback) |
| **C** | Hot-reload domains | Admin reload endpoint |
| **D** | SSE ingest progress | Ingest stream endpoint |

---

## Design highlights

- **OpenRouter only** — All LLM and embedding API access via OpenRouter (no direct provider clients).
- **Languages** — English and Spanish for stop words, query language, and answer generation; extensible.
- **Config over hardcoding** — Stop words, classification rules, extraction patterns, models, prompts, and parsers come from YAML/config; guardrails remain YAML-defined.
- **Classification** — Rule-based (filename + content keywords) by default; **LLM classification fallback** (required, flag on/off) to suggest doc_type when the fallback rule would apply.
- **Ingestion ledger & classification-help** — **Required:** persistent ledger, GET ledger endpoint (what was ingested and what wasn't, with reason), preflight (classify-only), virtual threads, hash-based skip. See [implementation-plan Required capabilities](./implementation-plan.md#required-capabilities--tracking-table).
- **Store LLM reasoning** — **Required** (flag on/off, default off) to store the model’s reasoning in the ledger; exposed in GET ledger when enabled.

---

## How to implement

1. Read [technical-design.md](./technical-design.md) for architecture and [implementation-plan.md](./implementation-plan.md) for iteration order, quality gates, and the **required capabilities (R1–R7)** tracking table.
2. Start with [iterations/iteration-00-quality-gates.md](./iterations/iteration-00-quality-gates.md), then implement [iteration-01-foundation.md](./iterations/iteration-01-foundation.md) through iteration 13 in order. Ledger, ledger endpoint, store LLM reasoning, LLM classification fallback, preflight, virtual threads, and hash-based skip are required (see tracking table).
3. Use the corresponding iteration doc as the single source for that slice; [framework-code.md](./framework-code.md) supplies the code to implement.
4. Run quality gates after each iteration: build, tests, coverage, and **Checkstyle, SpotBugs, PMD** (no violations). Before submitting or pushing, run `mvn verify` or `./gradlew check` / `./gradlew build`. Fix any violations before considering the change done. Sonar runs in CI with `-Dsonar.skip=false`; locally skipped by default. See [iteration-00-quality-gates.md](./iterations/iteration-00-quality-gates.md) and Cursor rule **quality-gates**.
5. Optionally add iterations A–D (override, feedback, reload, SSE) after the core 13; required capabilities (R1–R7) are included in core iterations 6, 9, 11.
