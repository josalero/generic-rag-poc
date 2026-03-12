# Domain Configuration Guide

> Parent: [technical-design.md](./technical-design.md) · Schema: [domain-definition-schema.json](./domain-definition-schema.json)
> Examples: [examples/recruiting.yml](./examples/recruiting.yml), [examples/legal.yml](./examples/legal.yml)

---

## 1. Getting Started

To create a new domain for the RAG platform, create a YAML file in the `domains/` directory.
The engine loads all `*.yml` files from this directory at startup.

### Minimum viable domain

```yaml
domain:
  id: my-domain
  display-name: "My Domain"
  enabled: true
  supported-file-types: [".pdf"]
  chunk-size: 500
  chunk-overlap: 100

  classification-rules:
    - doc-type: document
      priority: 1
      match:
        filename-patterns: ["*.pdf"]
        content-keywords: []
        min-keyword-hits: 0

  doc-types:
    document:
      display-name: "Generic Document"
      metadata: []

  guardrails:
    rules: []

  prompts:
    query: |
      Answer the question based only on the following excerpts.
      Context: %s
      Question: %s
    fallback: |
      Found %d matching excerpts.
```

This is the simplest valid domain. It accepts PDFs, classifies everything as `"document"`,
extracts no metadata, has no guardrails, and uses a minimal prompt.

---

## 2. YAML Structure Reference

```yaml
domain:
  # ── Identity ──
  id: <string>                       # required, lowercase alphanumeric + hyphens
  display-name: <string>             # required, human-readable
  enabled: <boolean>                 # required

  # ── File handling ──
  documents-path: <string>           # optional, folder for batch ingestion
  supported-file-types: [<string>]   # required, at least one, e.g. [".pdf", ".docx"]
  chunk-size: <integer>              # required, 50–10000
  chunk-overlap: <integer>           # required, 0–5000

  # ── Entity linkage ──
  entity-id-key: <string>            # optional, e.g. "candidate_id", "matter_id"

  # ── Stop words ──
  stop-words: [<string>]             # optional, domain-specific noise terms

  # ── Model selection ──
  models:                            # optional
    extraction: <string>             # model ID for metadata extraction (e.g. "gpt-4o-mini")
    query: <string>                  # model ID for answer generation (e.g. "gpt-4o")

  # ── Classification rules ──
  classification-rules: [...]        # required, at least one rule

  # ── Document types ──
  doc-types:                         # required, at least one doc_type
    <doc_type_id>:
      display-name: <string>
      metadata: [...]

  # ── Guardrails ──
  guardrails:
    rules: [...]                     # required (can be empty array)

  # ── Prompts ──
  prompts:
    query: <string>                  # required, must contain two %s placeholders
    fallback: <string>               # required, must contain one %d placeholder
```

For the full formal specification, validate against
[domain-definition-schema.json](./domain-definition-schema.json).

---

## 3. Classification Rules

Classification rules determine the `doc_type` of each ingested document.
They are evaluated by priority (highest first); the first matching rule wins.

### Rule anatomy

```yaml
classification-rules:
  - doc-type: certification       # the doc_type assigned when this rule matches
    priority: 30                  # higher = evaluated first
    match:
      filename-patterns:          # glob patterns matched against the filename
        - "*cert*"
        - "*credential*"
      content-keywords:           # keywords searched in the text (case-insensitive)
        - "certified"
        - "credential id"
        - "issued by"
      min-keyword-hits: 2         # how many keywords must be found
```

### Matching logic

A rule matches when:
1. The filename matches **any** of the `filename-patterns` (glob, case-insensitive)
2. **AND** the number of `content-keywords` found in the text >= `min-keyword-hits`

### Tips

- Always include a **fallback rule** with `priority: 1` and `min-keyword-hits: 0`
- Put the most specific rules at highest priority
- Use `min-keyword-hits: 0` only for fallback — otherwise the rule matches everything
- Test classification with representative documents before deploying

---

## 4. Metadata Fields

Each doc_type declares a list of metadata fields to extract.

### Field anatomy

```yaml
metadata:
  - key: cert_name             # metadata key (lowercase snake_case)
    type: string               # data type: string | integer | float | boolean
    extraction: llm            # strategy: regex | llm | keyword | composite
    model: "gpt-4o"           # optional, overrides domain.models.extraction
    prompt: "Extract the certification name. Return only the name."
```

### Extraction strategies

| Strategy | Required fields | Best for |
|---|---|---|
| `regex` | `patterns` (array of regex strings) | Dates, IDs, amounts, codes |
| `llm` | `prompt` (string), optional `model` | Names, summaries, classifications |
| `keyword` | `keywords` (map of category → keyword list) | Enum-like fields (status, level) |
| `composite` | `strategies` (array of sub-strategies, min 2) | Fields with variable format |

See [extraction-strategies.md](./extraction-strategies.md) for full detail.

### Regex field example

```yaml
- key: filing_date
  type: string
  extraction: regex
  patterns:
    - "(?:filed|filing date)[:\\s]*(\\d{4}-\\d{2}-\\d{2})"
    - "(?:filed|filing date)[:\\s]*(\\w+ \\d{1,2},? \\d{4})"
```

### LLM field example

```yaml
- key: holding_summary
  type: string
  extraction: llm
  prompt: "Summarize the court's holding in one sentence."
```

### Keyword field example

```yaml
- key: degree_level
  type: string
  extraction: keyword
  keywords:
    PhD: ["phd", "ph.d", "doctorate"]
    "Master's": ["master", "m.s.", "mba"]
    "Bachelor's": ["bachelor", "b.s.", "b.a."]
```

### Composite field example

```yaml
- key: candidate_location
  type: string
  extraction: composite
  strategies:
    - type: regex
      patterns:
        - "(?:based in|located in)\\s+([A-Z][a-zA-Z\\s,.-]{2,40})"
    - type: llm
      prompt: "Extract the candidate's city and state/country."
```

---

## 5. Guardrail Rules

Guardrails protect the query pipeline from unsafe or policy-violating questions.

### Term-block rule

Blocks queries containing trigger terms combined with intent terms:

```yaml
guardrails:
  rules:
    - type: term-block
      description: "Block bias-based filtering"
      trigger-terms: [age, gender, race, ethnicity, disability]
      intent-terms: [rank, filter, prefer, hire, exclude]
      require-both: true
      blocked-message: "I can't filter by protected attributes."
```

When `require-both: true`, the query must contain at least one trigger-term
AND at least one intent-term to be blocked. This prevents false positives
(e.g. "What is the candidate's experience?" contains "candidate" but no trigger term).

### Pattern-block rule

Blocks queries matching regex patterns:

```yaml
    - type: pattern-block
      description: "Block prompt injection"
      patterns:
        - "\\b(ignore|disregard|bypass)\\b.{0,80}\\b(instruction|prompt)\\b"
      blocked-message: "Request blocked by security guardrail."
```

### LLM-block rule

Delegates the safety decision to an LLM. The LLM receives a system prompt defining
the safety policy and the user's query. It responds with `PASS` or `BLOCK|<reason>`.

```yaml
    - type: llm-block
      description: "Detect subtle bias or context-dependent unsafe queries"
      model: "gpt-4o-mini"
      prompt: |
        You are a safety evaluator for a recruiting search system.
        Determine if the following query attempts to discriminate based on
        protected attributes (age, gender, race, ethnicity, disability,
        religion, marital status) — even indirectly or through coded language.

        Examples of BLOCK:
        - "Find young energetic candidates" (age proxy)
        - "Candidates who would fit our culture" (potential bias proxy)
        - "Candidates without career gaps" (potential discrimination)

        Examples of PASS:
        - "Find candidates with Java and 5+ years experience"
        - "Candidates with AWS certifications in Denver"
      blocked-message: "This query may contain bias. Please rephrase using objective, job-relevant criteria."
```

**Key properties:**

| Property | Required | Description |
|---|---|---|
| `model` | Optional | Model ID from `application.yml`. Defaults to `app.models.default-model`. Use a fast/cheap model — this is a binary classification task. |
| `prompt` | Required | System prompt defining the safety policy. Should include examples of BLOCK and PASS for few-shot guidance. |
| `blocked-message` | Required | User-facing message when the LLM returns BLOCK. The LLM's reason is appended for logging but not shown to the user. |

**Recommended rule ordering:**

```yaml
guardrails:
  rules:
    - type: term-block         # 1st: fast, free, catches obvious terms
      # ...
    - type: pattern-block      # 2nd: fast, free, catches structural attacks
      # ...
    - type: llm-block          # 3rd: slow, costs per-call, catches subtle cases
      # ...                    #      only runs if deterministic rules passed
```

**Failure behavior:** If the LLM call fails (timeout, error), the rule defaults
to PASS — deterministic rules above should already catch the obvious cases.

---

## 6. Prompt Templates

### Query prompt

The main prompt sent to the LLM for answer generation. Must contain exactly two `%s` placeholders:

```yaml
prompts:
  query: |
    Answer based only on the following legal document excerpts.
    Cite specific clauses, sections, or case references when possible.
    Never provide legal advice — only summarize what the documents state.

    Context:
    %s

    Question: %s
```

The first `%s` receives the concatenated text of the top retrieved segments.
The second `%s` receives the user's question.

### Fallback prompt

Used when the LLM is unavailable. Must contain one `%d` placeholder:

```yaml
  fallback: |
    Found %d matching excerpts. Results ranked by hybrid vector and keyword scoring.
    LLM summarization was unavailable.
```

---

## 7. Model Configuration

Each domain can specify which LLM model to use for extraction and query generation.
Model IDs reference entries defined in `application.yml` (see [framework-code.md § Model Registry](./framework-code.md#2-model-registry-config)).

### Domain-level model assignment

```yaml
domain:
  id: legal
  models:
    extraction: "gpt-4o-mini"    # used for all LLM extraction fields by default
    query: "gpt-4o"              # used for answer generation
```

### Per-field model override

Individual metadata fields can override the domain default:

```yaml
  doc-types:
    court_filing:
      metadata:
        - key: case_title
          extraction: llm
          # no 'model' → inherits domain.models.extraction ("gpt-4o-mini")
          prompt: "Extract the case caption."

        - key: holding_summary
          extraction: llm
          model: "deepseek-r1"     # override: complex reasoning needs stronger model
          prompt: "Summarize the court's holding in one sentence."
```

### Resolution order

| Priority | Source | Example |
|---|---|---|
| 1 (highest) | Field-level `model` | `model: "deepseek-r1"` on a specific field |
| 2 | Domain-level `models.extraction` | `models.extraction: "gpt-4o-mini"` |
| 3 | Domain-level `models.query` | Used for query answer generation only |
| 4 (lowest) | App-level `app.models.default-model` | Global fallback in `application.yml` |

### Production vs development

The design supports **market benchmark** models for production and **free / low-cost** for development. See [model-recommendations.md](./model-recommendations.md).

**Production (benchmark-grade):** Define in `application.yml` or `application-prod.yml`: `gpt-4o-mini`, `gpt-4o`, optional `claude-sonnet`. Domain YAML: `models.extraction: "gpt-4o-mini"`, `models.query: "gpt-4o"` (or use neutral aliases like `extraction` / `query` and resolve per profile). Embedding: OpenAI `text-embedding-3-small`; PGVector `vector(1536)`.

**Development (free / low-cost):** Use profile `dev` and `application-dev.yml` with `extraction-free`, `query-free` (OpenRouter free), `embedding: "in-process"`. Domain YAML can use the same aliases if prod/dev define them; or use `extraction-free` / `query-free` in dev. Embedding: LangChain4j in-process ONNX (384 dims); PGVector `vector(384)` in a separate DB/schema.

### Typical model assignments

| Use case | Production (benchmark) | Development (free / low-cost) |
|----------|------------------------|-------------------------------|
| Bulk extraction | `gpt-4o-mini` | `extraction-free` (OpenRouter free) |
| Answer generation | `gpt-4o`, `claude-sonnet` | `query-free` (OpenRouter free) |
| Complex analysis / guardrails | `gpt-4o`, Claude Sonnet | `query-free` or same as extraction-free |

---

## 8. Chunk Size Guidelines

| Document type | Recommended chunk-size | Recommended chunk-overlap | Rationale |
|---|---|---|---|
| Resumes | 300–500 | 50–100 | Short, dense; small chunks give precise skill matching |
| Certifications | 300–500 | 50–100 | Very short documents |
| Legal contracts | 800–1200 | 150–250 | Long clauses with cross-references need larger context |
| Court opinions | 800–1000 | 150–200 | Lengthy reasoning sections |
| Clinical notes | 600–800 | 100–150 | Moderate density, mixed structured/unstructured |
| Academic papers | 600–800 | 100–150 | Sections with internal coherence |
| Statutes | 500–700 | 100–150 | Structured sections, moderate length |

---

## 9. Validation

### IDE validation

Add a YAML schema mapping for inline validation and autocompletion:

```json
// .vscode/settings.json
{
  "yaml.schemas": {
    "./generic-rag-poc/domain-definition-schema.json": "domains/*.yml"
  }
}
```

### CI validation

```bash
# Using ajv-cli
ajv validate -s domain-definition-schema.json -d "domains/*.yml" --spec=draft2020

# Using yajsv
yajsv -s domain-definition-schema.json domains/*.yml
```

### Common validation errors

| Error | Cause | Fix |
|---|---|---|
| `id` does not match pattern | Uppercase or special characters | Use lowercase with hyphens only |
| `extraction: llm` missing `prompt` | LLM fields need a prompt | Add `prompt:` to the field |
| `extraction: regex` missing `patterns` | Regex fields need patterns | Add `patterns:` array |
| `composite` strategies < 2 | Composite needs at least 2 strategies | Add more sub-strategies or switch to single strategy |
| No classification rules | At least one rule required | Add a fallback rule with `min-keyword-hits: 0` |

---

## 10. Checklist — Creating a New Domain

1. Identify the domain and its document types
2. For each doc_type, list the metadata fields you want to extract
3. For each field, choose an extraction strategy (see [strategy selection guide](./extraction-strategies.md#8-strategy-selection-guide))
4. Write classification rules with appropriate priorities
5. Define guardrails relevant to the domain's compliance requirements
6. Write a domain-specific query prompt that guides the LLM's answer format
7. Create the YAML file in `domains/`
8. Validate against the JSON Schema
9. Test with representative documents
10. Set `enabled: true` and restart (or hot-reload)

---

## 11. Human-in-the-Loop and Feedback (Optional)

To let the system **learn per domain** from human input, you can enable feedback collection. See [technical-design.md § 19. Human-in-the-Loop and Feedback](./technical-design.md#19-human-in-the-loop-and-feedback) for the full design.

**Optional block in domain YAML:**

```yaml
feedback:
  query:
    enabled: true
    collect-ratings: true      # thumbs up/down or score
    collect-corrections: true  # corrected answer text
  ingestion:
    enabled: true
    collect-classification-corrections: true   # corrected doc_type
    collect-metadata-corrections: true        # corrected field values
```

When enabled, the API accepts `POST /api/v1/{domainId}/feedback/query` and `.../feedback/ingestion`. Feedback is stored per domain and can drive suggested YAML changes, training data export, or metrics — without hardcoding any learning logic in the engine.
