# Iteration 13 — Production hardening

> Parent: [implementation-plan.md](../implementation-plan.md) · Previous: [iteration-12-auto-configuration-wiring.md](./iteration-12-auto-configuration-wiring.md)

## Goal

Global exception handling, request validation, virtual threads, and actuator health so the app is production-ready.

## Deliverables

| # | Component | Location |
|---|-----------|----------|
| 1 | Global exception handler (`@RestControllerAdvice`) | [framework-code.md § 13](../framework-code.md#13-global-exception-handling-config) |
| 2 | Request/response DTOs with Jakarta Validation | [framework-code.md § 14](../framework-code.md#14-request--response-dtos-with-jakarta-validation) |
| 3 | Virtual thread configuration | [framework-code.md § 15](../framework-code.md#15-virtual-thread-configuration) |
| 4 | Actuator health indicator (domains) | [framework-code.md § 16](../framework-code.md#16-actuator-health-indicator) |
| 5 | `build.gradle` (validation, actuator, dependencies) | [framework-code.md § 17](../framework-code.md#17-buildgradle) |

## Acceptance criteria

- Unknown domain, validation errors, and server errors return RFC 9457 ProblemDetail or consistent JSON; no stack traces in body.
- Query request: `@Valid`; blank question or invalid range → 400 with clear message.
- Virtual threads: executor and Tomcat configured for virtual threads as in design.
- Actuator health (or custom health) exposes status and list of loaded domains (or count).

## Tests to add

- `GlobalExceptionHandlerTest`: trigger DomainNotFoundException, validation exception, generic exception; assert status code and body structure (type, title, detail).
- `QueryRequestValidationTest`: invalid request (blank question, negative page) returns 400 when validated.
- Health indicator test: with mock registry, indicator returns expected status and details.

## Quality gates

- All tests pass; security rules respected (no PII in error bodies); build includes checkstyle if required.

See [iteration-00-quality-gates.md](./iteration-00-quality-gates.md) for gates that apply to every iteration.

---

## Code

All production-hardening code is in [framework-code.md](../framework-code.md):

- **§ 13** — `GlobalExceptionHandler`: DomainNotFoundException, UnsupportedFileTypeException, MaxUploadSizeExceededException, MethodArgumentNotValidException, generic Exception → ProblemDetail
- **§ 14.1–14.4** — `QueryRequest` (Jakarta Validation), `QueryResponse`, `IngestResponse`, `DomainQueryController` with `@Valid @RequestBody QueryRequest`
- **§ 15** — Virtual thread executor and Tomcat customizer
- **§ 16** — `DomainHealthIndicator`: up with domain count and list, down when no domains
- **§ 17** — build.gradle: Spring Boot 4, Java 25, LangChain4j, validation, actuator, PDFBox, POI, etc.
