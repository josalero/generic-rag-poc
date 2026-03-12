# Iteration 3 — Extraction strategies (no LLM)

> Parent: [implementation-plan.md](../implementation-plan.md) · Previous: [iteration-02-config-model-registry.md](./iteration-02-config-model-registry.md)

## Goal

Implement regex, keyword, and composite extraction strategies and the factory. Composite may chain regex + keyword only (LLM sub-strategy in Iteration 7). These **custom algorithms** cover most structured metadata (dates, IDs, categories) without LLM; see [technical-design.md § 21](../technical-design.md#21-custom-algorithms-vs-llm). **Prefer:** Strategies and patterns (regex, keyword maps) come from domain YAML/config — no hardcoded patterns or category maps in code.

## Deliverables

| # | Component | Location in this doc |
|---|-----------|----------------------|
| 1 | `ExtractionStrategyFactory` | § Code 4.1 (regex, keyword, composite only) |
| 2 | `RegexExtractionStrategy` | § Code 4.2 |
| 3 | `KeywordExtractionStrategy` | § Code 4.4 |
| 4 | `CompositeExtractionStrategy` | § Code 4.5 |

## Acceptance criteria

- Factory creates strategy from a map/key (e.g. `extraction: regex`, `patterns: [...]`); regex returns first capture; keyword returns first matching category; composite tries strategies in order until one returns non-null.
- No real LLM calls; composite's LLM step can be skipped or mocked.

## Tests to add

- `RegexExtractionStrategyTest`: text containing a date pattern returns captured group; no match returns null; multiple patterns, first match wins.
- `KeywordExtractionStrategyTest`: text with keyword from category A returns A; multiple categories, first match wins; no keyword returns null.
- `CompositeExtractionStrategyTest`: first strategy returns value → that value; first null, second returns value → second's value; all null → null.
- `ExtractionStrategyFactoryTest`: create regex/keyword/composite from config maps; unknown strategy type throws or returns empty optional.

## Quality gates

- `./gradlew test`; no network; tests are deterministic.

**Design note:** When authoring domain YAML, prefer regex/keyword (or composite with these first) for structured fields so ingestion stays high-quality and low-cost; use LLM only for summaries or highly variable fields (§ 21).

See [iteration-00-quality-gates.md](./iteration-00-quality-gates.md) for gates that apply to every iteration.

---

## Code — Extraction strategies (regex, keyword, composite)

For Iteration 7 you will add `LlmExtractionStrategy` and the `llm` case in the factory.

### Code 4.1 ExtractionStrategyFactory (regex, keyword, composite only)

```java
package com.example.rag.feature.domain.engine.strategy;

import com.example.rag.feature.domain.ExtractionStrategy;
import java.util.List;
import java.util.Map;
import org.springframework.stereotype.Component;

@Component
public class ExtractionStrategyFactory {

    @SuppressWarnings("unchecked")
    public ExtractionStrategy create(Map<String, Object> fieldConfig,
                                     Object modelRegistryOrNull,
                                     String resolvedModelId) {
        var extraction = (String) fieldConfig.get("extraction");

        return switch (extraction != null ? extraction : "") {
            case "regex" -> new RegexExtractionStrategy(
                    (List<String>) fieldConfig.get("patterns"));

            case "keyword" -> new KeywordExtractionStrategy(
                    (Map<String, List<String>>) fieldConfig.get("keywords"));

            case "composite" -> {
                var subStrategies = ((List<Map<String, Object>>) fieldConfig.get("strategies"))
                        .stream()
                        .map(sub -> {
                            var subType = (String) sub.get("type");
                            var subConfig = new java.util.HashMap<>(sub);
                            subConfig.put("extraction", subType);
                            return create(subConfig, modelRegistryOrNull, resolvedModelId);
                        })
                        .toList();
                yield new CompositeExtractionStrategy(subStrategies);
            }

            default -> throw new IllegalArgumentException(
                    "Unknown extraction strategy: '%s'. In Iteration 7 add 'llm'.".formatted(extraction));
        };
    }
}
```

### Code 4.2 RegexExtractionStrategy

```java
package com.example.rag.feature.domain.engine.strategy;

import com.example.rag.feature.domain.ExtractionStrategy;
import java.util.List;
import java.util.Map;
import java.util.regex.Pattern;

public class RegexExtractionStrategy implements ExtractionStrategy {

    private final List<Pattern> patterns;

    public RegexExtractionStrategy(List<String> rawPatterns) {
        this.patterns = rawPatterns.stream()
                .map(p -> Pattern.compile(p, Pattern.CASE_INSENSITIVE))
                .toList();
    }

    @Override
    public String extract(String text, Map<String, Object> config) {
        for (var pattern : patterns) {
            var matcher = pattern.matcher(text);
            if (matcher.find() && matcher.groupCount() >= 1) {
                return matcher.group(1).strip();
            }
        }
        return null;
    }
}
```

### Code 4.4 KeywordExtractionStrategy

```java
package com.example.rag.feature.domain.engine.strategy;

import com.example.rag.feature.domain.ExtractionStrategy;
import java.util.List;
import java.util.Locale;
import java.util.Map;

public class KeywordExtractionStrategy implements ExtractionStrategy {

    private final Map<String, List<String>> keywords;

    public KeywordExtractionStrategy(Map<String, List<String>> keywords) {
        this.keywords = keywords;
    }

    @Override
    public String extract(String text, Map<String, Object> config) {
        var lowerText = text.toLowerCase(Locale.ROOT);

        for (var entry : keywords.entrySet()) {
            boolean found = entry.getValue().stream()
                    .anyMatch(kw -> lowerText.contains(kw.toLowerCase(Locale.ROOT)));
            if (found) {
                return entry.getKey();
            }
        }
        return null;
    }
}
```

### Code 4.5 CompositeExtractionStrategy

```java
package com.example.rag.feature.domain.engine.strategy;

import com.example.rag.feature.domain.ExtractionStrategy;
import java.util.List;
import java.util.Map;

public class CompositeExtractionStrategy implements ExtractionStrategy {

    private final List<ExtractionStrategy> strategies;

    public CompositeExtractionStrategy(List<ExtractionStrategy> strategies) {
        this.strategies = strategies;
    }

    @Override
    public String extract(String text, Map<String, Object> config) {
        for (var strategy : strategies) {
            var result = strategy.extract(text, config);
            if (result != null && !result.isBlank()) {
                return result;
            }
        }
        return null;
    }
}
```
