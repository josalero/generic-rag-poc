# Iteration 11 — REST controllers

> Parent: [implementation-plan.md](../implementation-plan.md) · Previous: [iteration-10-query-service.md](./iteration-10-query-service.md)

## Goal

Expose ingest, query, and admin endpoints; delegate to services; return DTOs and appropriate status codes.

## Deliverables

| # | Component | Location |
|---|-----------|----------|
| 1 | `DomainIngestController` | [framework-code.md § 9.1](../framework-code.md#91-domainingestcontroller) |
| 2 | `DomainQueryController` | [framework-code.md § 9.2](../framework-code.md#92-domainquerycontroller) |
| 3 | `DomainAdminController` | [framework-code.md § 9.3](../framework-code.md#93-domainadmincontroller) |

## Acceptance criteria

- Ingest: POST multipart file(s), return 200 with summary or 404/422 for unknown domain / unsupported type.
- Query: POST JSON body (question, optional params), return 200 with answer + results or 404 for unknown domain; guardrail blocked → 200 with blocked payload or 403 as designed.
- Admin: GET domains, GET doc-types for domain, POST reload (optional); appropriate auth/roles as per design.
- **Ledger and dashboard (optional E):** When implemented, expose **GET** `/api/v1/{domainId}/ingestion/ledger` with query params `status`, `since`, `limit`, `offset`; response includes entries (source, status, reason, doc_type, next_steps, optional **llm_reasoning**, created_at) so callers and a dashboard can see what was ingested and what wasn’t, with reason. Optionally **POST** `/api/v1/{domainId}/ingest/preflight` for classify-only. See [technical-design.md § 23](../technical-design.md#23-ingestion-ledger-and-classification-help-flow).

## Tests to add

- `DomainIngestControllerTest`: MockMvc; multipart upload; mock service; expect 200 and response body fields; expect 404 when domain missing.
- `DomainQueryControllerTest`: MockMvc; POST query body; mock service; expect 200 and answer in body; expect 404 for unknown domain; test validation (e.g. blank question → 400 if validated).
- `DomainAdminControllerTest`: GET domains returns list; GET doc-types for domain returns list; reload (if implemented) returns expected status.
- When ledger is implemented (R1–R2): GET ledger returns 200 with entries array; query params status, since, limit, offset applied; response includes llm_reasoning when stored (R3). Preflight (R5): POST preflight returns suggested_doc_type and next_steps.

## Required capabilities delivered here (see [implementation-plan tracking table](../implementation-plan.md#required-capabilities--tracking-table))

| Id | Capability | Taken into account in this iteration |
|----|------------|--------------------------------------|
| **R2** | Dashboard / ledger endpoint | GET `/api/v1/{domainId}/ingestion/ledger`; response shape includes source, status, reason, next_steps, llm_reasoning (when R3 on), created_at. |
| **R3** | Store LLM reasoning | Ledger API response includes `llm_reasoning` when the feature is on and an LLM was used. |
| **R5** | Preflight (classify-only) | POST `/api/v1/{domainId}/ingest/preflight` returns suggested_doc_type and next_steps (and optional reasoning when R3 on). |

## Quality gates

- Controller tests are slice tests or full integration with mocked services; no real DB/LLM in controller tests unless explicitly integration.

See [iteration-00-quality-gates.md](./iteration-00-quality-gates.md) for gates that apply to every iteration.

---

## Code

Full controller implementations: [framework-code.md § 9](../framework-code.md#9-rest-controllers).
