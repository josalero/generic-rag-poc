# Iteration 10 — Query service

> Parent: [implementation-plan.md](../implementation-plan.md) · Previous: [iteration-09-ingestion-service.md](./iteration-09-ingestion-service.md)

## Goal

Orchestrate query flow: validate domain, run guardrails, extract terms, retrieve, hybrid score, deduplicate, generate answer with LLM, paginate, build explainability.

## Deliverables

| # | Component | Location |
|---|-----------|----------|
| 1 | `DomainQueryService` | [framework-code.md § 8.1](../framework-code.md#81-domainqueryservice) |

## Acceptance criteria

- Input: domainId, question, optional filters (docTypes, maxResults, minScore, page, pageSize), optional **language** (en, es). Resolve domain; resolve language from body or Accept-Language or default locale; evaluate guardrails (block → return blocked result); extract terms using **language-appropriate** general stop words (en/es) + domain stop words; retrieve from store with domain filter; hybrid scoring; deduplicate by entity/source; generate answer (using localized prompt if defined for request language) via domain prompt template and query model; paginate; return answer + results + explainability.

## Tests to add

- `DomainQueryServiceTest`: mocked registry (one domain with stub classifier, guardrails, prompt provider, model config), mocked retriever (returns fixed list of contents), mocked `ChatModel` (returns fixed answer); call query; assert answer text, that guardrail was called, that retriever was called with domain filter. Test guardrail blocked → result is blocked. Test term extraction uses domain + general stop words when both provided. Test with `language: "es"` uses Spanish stop words when provider is locale-aware.

## Quality gates

- All external collaborators mocked; no real embedding store or LLM in unit tests.

See [iteration-00-quality-gates.md](./iteration-00-quality-gates.md) for gates that apply to every iteration.

---

## Code

Full `DomainQueryService` (guardrails, term extraction, retriever, hybrid scoring, deduplication, answer generation, pagination, `QueryRequest`/`QueryResult`/`ScoredContent`/`Explainability`): [framework-code.md § 8.1](../framework-code.md#81-domainqueryservice).
