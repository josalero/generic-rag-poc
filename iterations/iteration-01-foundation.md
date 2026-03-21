# Iteration 1 — Foundation (interfaces, records, registry)

> Parent: [implementation-plan.md](../implementation-plan.md) · Previous: [iteration-00-quality-gates.md](./iteration-00-quality-gates.md)

## Goal

Define all domain abstractions and the registry. No YAML, no LLM, no I/O — pure interfaces and in-memory registry so later iterations can depend on them.

## Deliverables

| # | Component | Location in this doc |
|---|-----------|----------------------|
| 1 | `RagDomain` | § Code 1.1 |
| 2 | `DomainModelConfig` | § Code 1.2 |
| 3 | `DocumentClassifier` | § Code 1.3 |
| 4 | `ExtractionStrategy` | § Code 1.4 |
| 5 | `MetadataExtractor` | § Code 1.5 |
| 6 | `MetadataExtractorRegistry` | § Code 1.6 |
| 7 | `GuardrailEvaluator`, `GuardrailDecision` | § Code 1.7 |
| 8 | `GuardrailRule` | § Code 1.8 |
| 9 | `PromptTemplateProvider` | § Code 1.9 |
| 10 | `DomainRegistry` | § Code 1.10 |

## Acceptance criteria

- All interfaces and records compile.
- `DomainRegistry`: `register(domain)`, `get(id)` returns optional, `get(id)` for unknown returns empty.
- No external dependencies (no Spring, no LangChain4j) in this package if desired; otherwise minimal.

## Tests to add

- `DomainRegistryTest`: register a stub `RagDomain`, get by id returns it; get unknown returns empty; get after register two, get first by id.
- `DomainModelConfigTest`: `resolveExtractionModel(fieldOverride)` returns field override when non-blank, else extraction model id.
- Optional: stub implementation of `RagDomain` (e.g. `StubRagDomain`) for use in later iterations.

## Quality gates

- `./gradlew test` passes.
- New code in `feature/domain/` covered by tests (registry + record behavior).

See [iteration-00-quality-gates.md](./iteration-00-quality-gates.md) for gates that apply to every iteration.

---

## Code — Core Abstractions (`feature/domain/`)

All code targets **Java 25** and **Spring Boot 4.0.4** with **LangChain4j 1.11**.

### Code 1.1 RagDomain

```java
package com.example.rag.feature.domain;

import dev.langchain4j.data.document.DocumentSplitter;
import java.util.List;
import java.util.Set;

public interface RagDomain {

    String domainId();

    String displayName();

    List<String> supportedFileTypes();

    DocumentSplitter documentSplitter();

    DocumentClassifier documentClassifier();

    MetadataExtractorRegistry metadataExtractors();

    GuardrailEvaluator guardrailEvaluator();

    PromptTemplateProvider promptProvider();

    Set<String> stopWords();

    DomainModelConfig modelConfig();
}
```

### Code 1.2 DomainModelConfig

Holds model IDs per purpose. Each domain can specify which model to use for extraction vs. query generation, and individual fields can override with a per-field model.

```java
package com.example.rag.feature.domain;

public record DomainModelConfig(
        String extractionModelId,
        String queryModelId
) {
    public String resolveExtractionModel(String fieldLevelOverride) {
        if (fieldLevelOverride != null && !fieldLevelOverride.isBlank()) {
            return fieldLevelOverride;
        }
        return extractionModelId;
    }
}
```

### Code 1.3 DocumentClassifier

```java
package com.example.rag.feature.domain;

public interface DocumentClassifier {

    String classify(String filename, String text);
}
```

### Code 1.4 ExtractionStrategy

```java
package com.example.rag.feature.domain;

import java.util.Map;

public interface ExtractionStrategy {

    String extract(String text, Map<String, Object> config);
}
```

### Code 1.5 MetadataExtractor

```java
package com.example.rag.feature.domain;

import java.util.Map;

public interface MetadataExtractor {

    Map<String, String> extract(String filename, String text);
}
```

### Code 1.6 MetadataExtractorRegistry

```java
package com.example.rag.feature.domain;

public interface MetadataExtractorRegistry {

    MetadataExtractor forDocType(String docType);
}
```

### Code 1.7 GuardrailEvaluator and GuardrailDecision

```java
package com.example.rag.feature.domain;

public interface GuardrailEvaluator {

    GuardrailDecision evaluate(String query);
}
```

```java
package com.example.rag.feature.domain;

public record GuardrailDecision(
        boolean blocked,
        String message
) {
    public static final GuardrailDecision PASS = new GuardrailDecision(false, null);

    public static GuardrailDecision block(String message) {
        return new GuardrailDecision(true, message);
    }
}
```

### Code 1.8 GuardrailRule

```java
package com.example.rag.feature.domain;

public interface GuardrailRule {

    GuardrailDecision evaluate(String query);
}
```

### Code 1.9 PromptTemplateProvider

```java
package com.example.rag.feature.domain;

public interface PromptTemplateProvider {

    String queryTemplate();

    String fallbackTemplate();
}
```

### Code 1.10 DomainRegistry

```java
package com.example.rag.feature.domain;

import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

public class DomainRegistry {

    private final Map<String, RagDomain> domains = new ConcurrentHashMap<>();

    public void register(RagDomain domain) {
        domains.put(domain.domainId(), domain);
    }

    public Optional<RagDomain> get(String domainId) {
        return Optional.ofNullable(domains.get(domainId));
    }

    public List<RagDomain> all() {
        return List.copyOf(domains.values());
    }

    public void clear() {
        domains.clear();
    }
}
```
