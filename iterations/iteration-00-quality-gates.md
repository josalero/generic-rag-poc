# Quality Gates (apply to every iteration)

> Parent: [implementation-plan.md](../implementation-plan.md) · See also: Cursor rule **quality-gates** (Checkstyle, SpotBugs, PMD, Sonar on every code change).

These gates apply to **every iteration** before merge. They keep the codebase buildable, tested, and maintainable.

**Required capabilities (R1–R7):** Ledger, dashboard/ledger endpoint, store LLM reasoning, LLM classification fallback, preflight, virtual threads, and hash-based skip are **required** and tracked in the [implementation-plan Required capabilities — tracking table](../implementation-plan.md#required-capabilities--tracking-table). They are delivered in iterations 6, 9, and 11. The same quality gates apply; the relevant iteration docs state acceptance criteria and tests.

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
| **Lint / Checkstyle** | No new violations in changed files | See § 1.5 — Checkstyle, SpotBugs, PMD required |
| **Meaningful assertions** | Tests assert behavior, not only "no exception" | Review: every test has at least one assertion on outcome |
| **No PII in logs** | No names, emails, phones in log messages | Review + grep for sensitive patterns |
| **Specific exceptions** | Catch specific exception types; no empty catch blocks | Code review |

## 1.5 Static analysis (Checkstyle, SpotBugs, PMD, Sonar)

**Checkstyle, SpotBugs, and PMD must be run on every code change.** Fix any violations before considering the change done (Checkstyle: line length 100, Javadoc where required; SpotBugs and PMD: resolve all reported issues).

| Tool | Requirement | How to verify |
|------|--------------|----------------|
| **Checkstyle** | No violations (line length 100, Javadoc where required) | **Maven:** `mvn checkstyle:check` · **Gradle:** `./gradlew checkstyleMain checkstyleTest` |
| **SpotBugs** | No bugs reported | **Maven:** `mvn spotbugs:check` · **Gradle:** `./gradlew spotbugsMain spotbugsTest` (if SpotBugs plugin applied) |
| **PMD** | No violations | **Maven:** `mvn pmd:check` · **Gradle:** `./gradlew pmdMain pmdTest` (if PMD plugin applied) |
| **Sonar** | In CI only; locally skipped by default | **CI:** run with `-Dsonar.skip=false` and `sonar.host.url` set; **local:** Sonar skipped by default |

**Before submitting or pushing:** Run the full verify phase so all gates run: **Maven** `mvn verify` (Checkstyle, SpotBugs, PMD are bound to verify) or **Gradle** `./gradlew check` / `./gradlew build` (if Checkstyle, SpotBugs, PMD are bound to `check`). Fix any Checkstyle, SpotBugs, or PMD violations before considering the change done.

## 1.3 Definition of Done (per iteration)

- [ ] All deliverables for the iteration implemented and referenced in this plan
- [ ] Unit tests (and integration tests where specified) added and passing
- [ ] No hardcoded secrets or PII in new code
- [ ] **Checkstyle, SpotBugs, PMD** run and **no violations** (fix before merge); see § 1.5
- [ ] Build green: `./gradlew clean build` or `mvn clean verify` (including tests and static analysis)
- [ ] PR description links to this plan and lists the iteration number

## 1.4 Optional (recommended)

- **Integration tests** for critical paths (e.g. load YAML → classify, or ingest one file) using test profiles and in-memory or Testcontainers DB
- **JaCoCo** coverage report; fail build below 80% on new code
- **Conventional commits** for each commit (`feat:`, `fix:`, `test:`, `chore:`)
- **Domain YAML authoring:** Prefer regex/keyword/composite for structured metadata; use LLM only where § 21 indicates (summaries, variable phrasing, nuanced guardrails)
- **Config over hardcoding:** Where a configurable alternative exists (stop words, extraction patterns, classification rules, model defs, prompts, parser selection), implement it via config/YAML/resource files rather than hardcoding in code; guardrails are excluded from this preference and remain YAML-defined as designed
