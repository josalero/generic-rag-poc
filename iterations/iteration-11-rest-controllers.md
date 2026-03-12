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

## Tests to add

- `DomainIngestControllerTest`: MockMvc; multipart upload; mock service; expect 200 and response body fields; expect 404 when domain missing.
- `DomainQueryControllerTest`: MockMvc; POST query body; mock service; expect 200 and answer in body; expect 404 for unknown domain; test validation (e.g. blank question → 400 if validated).
- `DomainAdminControllerTest`: GET domains returns list; GET doc-types for domain returns list; reload (if implemented) returns expected status.

## Quality gates

- Controller tests are slice tests or full integration with mocked services; no real DB/LLM in controller tests unless explicitly integration.

See [iteration-00-quality-gates.md](./iteration-00-quality-gates.md) for gates that apply to every iteration.

---

## Code

Full controller implementations: [framework-code.md § 9](../framework-code.md#9-rest-controllers).
