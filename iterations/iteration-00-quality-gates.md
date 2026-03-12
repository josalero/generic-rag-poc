# Quality Gates (apply to every iteration)

> Parent: [implementation-plan.md](../implementation-plan.md)

These gates apply to **every iteration** before merge. They keep the codebase buildable, tested, and maintainable.

## 1.1 Build & test

| Gate | Requirement | How to verify |
|------|--------------|----------------|
| **Compile** | No compilation errors | `./gradlew compileJava compileTestJava` |
| **Unit tests** | All tests pass | `./gradlew test` |
| **No flaky tests** | Tests are deterministic; no external calls in unit tests | Review: no network, no real DB in unit tests |
| **Coverage (new code)** | ≥ 80% line coverage for code added in the iteration | `./gradlew test jacocoTestReport` (if JaCoCo applied); or team rule: new classes have tests |

## 1.2 Code quality

| Gate | Requirement | How to verify |
|------|--------------|----------------|
| **Lint / Checkstyle** | No new violations in changed files | `./gradlew checkstyleMain checkstyleTest` (if configured) or IDE lint |
| **Meaningful assertions** | Tests assert behavior, not only "no exception" | Review: every test has at least one assertion on outcome |
| **No PII in logs** | No names, emails, phones in log messages | Review + grep for sensitive patterns |
| **Specific exceptions** | Catch specific exception types; no empty catch blocks | Code review |

## 1.3 Definition of Done (per iteration)

- [ ] All deliverables for the iteration implemented and referenced in this plan
- [ ] Unit tests (and integration tests where specified) added and passing
- [ ] No hardcoded secrets or PII in new code
- [ ] Build green: `./gradlew clean build` (including tests; optional checkstyle)
- [ ] PR description links to this plan and lists the iteration number

## 1.4 Optional (recommended)

- **Integration tests** for critical paths (e.g. load YAML → classify, or ingest one file) using test profiles and in-memory or Testcontainers DB
- **JaCoCo** coverage report; fail build below 80% on new code
- **Conventional commits** for each commit (`feat:`, `fix:`, `test:`, `chore:`)
- **Domain YAML authoring:** Prefer regex/keyword/composite for structured metadata; use LLM only where § 21 indicates (summaries, variable phrasing, nuanced guardrails)
- **Config over hardcoding:** Where a configurable alternative exists (stop words, extraction patterns, classification rules, model defs, prompts, parser selection), implement it via config/YAML/resource files rather than hardcoding in code; guardrails are excluded from this preference and remain YAML-defined as designed
