# Model Recommendations — Development vs Production

> Parent: [technical-design.md](./technical-design.md)

**All LLM and embedding API access is via OpenRouter.** There are no direct OpenAI (or other provider) API connections. OpenAI models (e.g. GPT-4o, text-embedding-3-small) are used **through OpenRouter** (e.g. `openai/gpt-4o-mini`); use `OPENROUTER_API_KEY` only for production and development API calls.

The design **supports market-benchmark models** so production can use embedding and chat models that meet quality and benchmark standards — all through OpenRouter. For **development**, use **free or low-cost** options (in-process embeddings, OpenRouter free tier). Same configuration shape; switch via **profiles** or **environment** (e.g. `spring.profiles.active=dev` vs `prod`).

---

## 1. Design principle: benchmark-ready, cost-aware

| Environment | Embedding | Chat (extraction / query) | Goal |
|-------------|-----------|----------------------------|------|
| **Production** | Benchmark-grade embeddings via OpenRouter (e.g. openai/text-embedding-3-small) | Benchmark-grade LLMs via OpenRouter (e.g. openai/gpt-4o, anthropic/claude-3.5-sonnet) | Quality, reliability, market benchmarks; single OPENROUTER_API_KEY |
| **Development** | Free in-process (LangChain4j ONNX) or OpenRouter | Free / low-cost (OpenRouter free tier) | No/minimal cost, fast iteration; OPENROUTER_API_KEY for free tier |

Use **one** embedding model and **one** set of chat model definitions per environment; domain YAML references **model aliases** (e.g. `extraction`, `query`) that resolve to the right concrete model per profile.

---

## 2. Production — market benchmark

### 2.1 Embedding (production)

Use API-based embeddings **via OpenRouter** (no direct provider APIs). OpenRouter exposes OpenAI and other embedding models:

| Model (via OpenRouter) | Dimensions | Notes |
|------------------------|-------------|--------|
| `openai/text-embedding-3-small` | 1536 | Strong retrieval, widely benchmarked |
| `openai/text-embedding-3-large` | 3072 | Higher quality, higher cost |

PGVector: `vector(1536)` for OpenAI small, or match the chosen model. **Do not** mix dimensions across environments if you share a DB (use separate DB or schema for dev). Single `OPENROUTER_API_KEY` for all API access.

### 2.2 Chat (production)

Use models that perform well on standard benchmarks — **all via OpenRouter** (no direct OpenAI or other provider clients):

| Purpose | Recommended (via OpenRouter) | Alternative |
|---------|-----------------------------|-------------|
| **Extraction** (bulk metadata) | `openai/gpt-4o-mini` | `anthropic/claude-3-haiku`, etc. |
| **Query** (answer generation) | `openai/gpt-4o` or `anthropic/claude-3.5-sonnet` | Claude 3.5 Sonnet, Gemini Pro — quality and safety |
| **Guardrails** (llm-block) | Same as query or a dedicated safety model | Per-domain |

Define these in `app.models.definitions` with `provider: openrouter`, `base-url: https://openrouter.ai/api/v1`, and `api-key: ${OPENROUTER_API_KEY}`. Model names use OpenRouter’s format (e.g. `openai/gpt-4o-mini`).

### 2.3 Example production config (application.yml or application-prod.yml)

**All models via OpenRouter** — no direct OpenAI or other provider APIs. Use `OPENROUTER_API_KEY` only.

```yaml
app:
  models:
    default-model: "gpt-4o-mini"
    embedding: "openai/text-embedding-3-small"
    definitions:
      gpt-4o-mini:
        provider: openrouter
        api-key: ${OPENROUTER_API_KEY}
        base-url: https://openrouter.ai/api/v1
        model-name: openai/gpt-4o-mini
        temperature: 0.1
        max-tokens: 2048
        timeout-seconds: 30
      gpt-4o:
        provider: openrouter
        api-key: ${OPENROUTER_API_KEY}
        base-url: https://openrouter.ai/api/v1
        model-name: openai/gpt-4o
        temperature: 0.1
        max-tokens: 4096
        timeout-seconds: 60
      claude-sonnet:
        provider: openrouter
        api-key: ${OPENROUTER_API_KEY}
        base-url: https://openrouter.ai/api/v1
        model-name: anthropic/claude-3.5-sonnet
        temperature: 0.1
        max-tokens: 4096
        timeout-seconds: 60
```

Domain YAML then uses aliases like `models.extraction: "gpt-4o-mini"`, `models.query: "gpt-4o"` (or `claude-sonnet`). These resolve to OpenRouter with the corresponding `openai/` or `anthropic/` model names. **No direct OpenAI (or other) API keys.**

---

## 3. Development — free and low-cost

### 3.1 Embedding (development)

Use **LangChain4j in-process ONNX** embeddings: no API key, no per-token cost.

| Artifact | Dimensions | Notes |
|----------|------------|--------|
| `langchain4j-embeddings-bge-small-en-v1.5-q` | 384 | Recommended for dev |
| `langchain4j-embeddings-all-minilm-l6-v2-q` | 384 | Alternative |
| `langchain4j-embeddings-e5-small-v2` | 384 | Multilingual-friendly |

PGVector for **dev**: `vector(384)`. Use a separate DB or schema from production so dimension mismatch is avoided.

### 3.2 Chat (development)

Use **OpenRouter free** or low-cost models (no Ollama):

| Alias | OpenRouter model | Use for |
|-------|-------------------|--------|
| `openrouter/free` | Free router | Extraction, query, guardrails |
| `meta-llama/llama-3.2-3b-instruct:free` | Free tier | Extraction, simple query |
| `google/gemma-2-2b-it:free` | Free tier | Extraction, query |

### 3.3 Example development config (application-dev.yml)

```yaml
app:
  models:
    default-model: "extraction-free"
    embedding: "in-process"
    definitions:
      extraction-free:
        provider: openrouter
        api-key: ${OPENROUTER_API_KEY}
        base-url: https://openrouter.ai/api/v1
        model-name: "openrouter/free"
        temperature: 0.1
        max-tokens: 2048
        timeout-seconds: 30
      query-free:
        provider: openrouter
        api-key: ${OPENROUTER_API_KEY}
        base-url: https://openrouter.ai/api/v1
        model-name: "openrouter/free"
        temperature: 0.1
        max-tokens: 4096
        timeout-seconds: 60
```

When `embedding: "in-process"`, the app must wire an ONNX `EmbeddingModel` bean (e.g. `BgeSmallEnV15QuantizedEmbeddingModel`) and use **vector(384)** in PGVector.

---

## 4. Limitations of free and low-cost options

Be aware of these limits when using **development** (free/low-cost) setups. They are acceptable for local/dev but not for production or benchmark reporting.

### 4.1 In-process embedding (dev)

| Limitation | Detail | Mitigation |
|------------|--------|------------|
| **Dimension** | 384 only (vs 1536 for production via OpenRouter). Retrieval quality and benchmark scores are not directly comparable to production. | Use only for dev; use separate dev DB/schema so you never mix 384 with 1536. |
| **Language** | Most artifacts are English-optimized (e.g. BGE small en, All-MiniLM). Multilingual is weaker than API embeddings. | Use e5-small-v2 for better multilingual; or accept English-only for dev. |
| **Quality** | Small models; MTEB/retrieval benchmarks will rank below production embeddings. | Expected for dev; do not report dev embedding metrics as production benchmarks. |
| **Resource** | Runs on CPU; first load and large batch embed can be slower than API. Memory footprint for model in JVM. | Allocate enough heap; use quantized (`-q`) artifacts to reduce size. |
| **No GPU** | ONNX embedding in LangChain4j does not use GPU. | Accept for dev; production API can use provider-side GPU. |

### 4.2 OpenRouter free tier (dev)

| Limitation | Detail | Mitigation |
|------------|--------|------------|
| **Rate limits** | Requests per minute and per day are limited; exact limits depend on OpenRouter policy and whether you have credits. | Throttle batch ingestion in dev; or use low-cost paid (e.g. gpt-4o-mini) for heavier dev runs. |
| **Model availability** | `openrouter/free` routes to whichever free model is available; model can change. Specific `:free` models may be deprecated or rotated. | Prefer specific free model IDs if you need consistency; accept occasional failures and retries. |
| **Quality** | Free models are smaller/weaker than gpt-4o or Claude Sonnet. Extraction and answer quality will not match production benchmarks. | Use only for dev; do not evaluate RAG quality or run benchmarks on free models. |
| **Context length** | Free models often have smaller context windows than paid frontier models. | Keep `max-tokens` and prompt sizes conservative in dev config. |
| **Latency / SLA** | No SLA; free tier can be slower or occasionally unavailable. | Add timeouts and retries; do not rely for CI or load tests. |
| **API key** | OpenRouter still requires an API key (free tier may have $0 usage but key is required). | Sign up at openrouter.ai; set `OPENROUTER_API_KEY` in dev env. |

### 4.3 Low-cost paid (e.g. gpt-4o-mini) when used in dev

| Limitation | Detail |
|------------|--------|
| **Cost** | Still incurs cost; suitable for dev if usage is low. |
| **Rate limits** | Provider limits apply; lower than free tier issues but can hit during bulk ingestion. |
| **Benchmarks** | gpt-4o-mini is not the “benchmark” model for query quality; use for extraction and light query in dev only. |

---

## 5. Aligning domain YAML with dev vs prod

Domain YAML can use **the same alias names** in both environments if you define those aliases in each profile:

| Alias (in domain YAML) | Production (application-prod.yml) | Development (application-dev.yml) |
|------------------------|-----------------------------------|------------------------------------|
| `extraction` | `gpt-4o-mini` | `extraction-free` (openrouter/free) |
| `query` | `gpt-4o` or `claude-sonnet` | `query-free` (openrouter/free) |

**Option A — Same aliases per profile:** In `application-prod.yml` define `extraction` → gpt-4o-mini, `query` → gpt-4o. In `application-dev.yml` define `extraction` → extraction-free, `query` → query-free. Domain YAML always sets `models.extraction: "extraction"`, `models.query: "query"`.

**Option B — Explicit per env:** Domain YAML stays the same; only `application-{profile}.yml` changes which concrete model each alias points to. E.g. prod defines `extraction: gpt-4o-mini`, dev defines `extraction: extraction-free` under the same key names used in domain YAML.

**Option C — Different alias names in domain YAML:** Domain YAML uses `extraction-free` / `query-free` in dev and `gpt-4o-mini` / `gpt-4o` in prod; then use Spring `@Profile` or separate domain YAML files per env (e.g. `recruiting-dev.yml` vs `recruiting.yml`). Less DRY; Option A or B is simpler.

Recommendation: **Option A** — domain YAML references neutral aliases (`extraction`, `query`); each profile defines those aliases to the right concrete model.

---

## 6. Summary table

| Purpose | Production (benchmark) | Development (free / low-cost) |
|---------|------------------------|--------------------------------|
| **Embedding** | Via OpenRouter: `openai/text-embedding-3-small` (1536) | LangChain4j in-process BGE small / All-MiniLM (384) |
| **Extraction** | `gpt-4o-mini`, Claude Haiku | OpenRouter `openrouter/free` or `llama-3.2-3b:free` |
| **Query** | `gpt-4o`, Claude 3.5 Sonnet | OpenRouter `openrouter/free` |
| **Guardrails (llm)** | Same as query or dedicated | Same as query-free |
| **PGVector dimension** | 1536 (OpenRouter embedding, e.g. openai/text-embedding-3-small) | 384 (in-process) |

---

## 7. PGVector schema by environment

**Production (via OpenRouter, e.g. openai/text-embedding-3-small):**

```sql
embedding vector(1536)
```

**Development (in-process 384-dim):**

```sql
embedding vector(384)
```

Use separate databases or schemas for dev and prod so dimension and model choices do not conflict.

---

## 8. Where each document points

| Document | Production | Development |
|----------|------------|-------------|
| **technical-design.md** | § 14 shows production defaults; benchmark-ready. | Dev config via profile or link here. |
| **framework-code.md** | application-prod.yml: all models via OpenRouter (openai/gpt-4o, etc.), OPENROUTER_API_KEY. | application-dev.yml with in-process + OpenRouter free. |
| **domain-configuration-guide.md** | Recommend `extraction` / `query` aliases → benchmark models. | Same aliases → free models in dev profile. |
| **examples/recruiting.yml**, **legal.yml** | `models.extraction: "extraction"`, `models.query: "query"` (aliases resolved per profile). | Same; profile selects concrete model. |

The **design always supports market benchmark**; production uses the proper models, and development uses low-cost or free equivalents.
