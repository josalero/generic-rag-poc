# Iteration 7 — Config-driven metadata + LLM strategy

> Parent: [implementation-plan.md](../implementation-plan.md) · Previous: [iteration-06-config-driven-engine.md](./iteration-06-config-driven-engine.md)

## Goal

Add metadata extraction to the config-driven engine: load field definitions from YAML, run regex/keyword/composite/LLM strategies. Wire `LlmExtractionStrategy` with a `ChatModel` (mock in tests). **Composite must try regex/keyword before LLM** so custom algorithms handle structured fields first; LLM is fallback for free-form or variable content (see [technical-design.md § 21](../technical-design.md#21-custom-algorithms-vs-llm)). **Prefer:** Field definitions, strategy types, and model overrides from domain YAML only — no hardcoded field lists or extraction logic in code; LLM only where configured.

## Deliverables

| # | Component | Location in this doc |
|---|-----------|----------------------|
| 1 | `ConfigDrivenMetadataExtractorRegistry` | § Code 3.4 |
| 2 | `ConfigDrivenMetadataExtractor` | § Code 3.5 |
| 3 | `LlmExtractionStrategy` | § Code 4.3 |
| 4 | `ExtractionStrategyFactory` (full: include LLM) | [iteration-03](./iteration-03-extraction-strategies.md) + add `llm` case |
| 5 | `ConfigDrivenRagDomain` (full: with extractors) | [framework-code.md § 3.2](../framework-code.md#32-configdrivenragdomain) |

## Acceptance criteria

- For a doc_type, extractor iterates fields; each field's strategy (regex, llm, keyword, composite) is created by factory; LLM strategy uses `ModelRegistry` (or injected `ChatModel` in tests).
- Model resolution: field override → domain extraction model → default.
- **Composite tries strategies in order** (e.g. regex then keyword then llm); first non-null result wins — so custom algorithms run before LLM (§ 21).
- Composite can include LLM as sub-step; factory passes model resolver where needed.

## Tests to add

- `ConfigDrivenMetadataExtractorTest`: YAML with one regex field and one keyword field; extract returns map with correct values; missing optional field returns null or omitted.
- `LlmExtractionStrategyTest`: with mock `ChatModel` returning fixed string, extract returns that string; null/empty response handled.
- Integration-style test (optional): load full domain YAML with regex + keyword fields only, extract from sample text; no real LLM.

## Quality gates

- LLM tests use only mocks or stubs; no real API keys; coverage for extractor and LLM strategy.

See [iteration-00-quality-gates.md](./iteration-00-quality-gates.md) for gates that apply to every iteration.

---

## Code — Metadata extractors and LlmExtractionStrategy

### Code 3.4 ConfigDrivenMetadataExtractorRegistry

```java
package com.example.rag.feature.domain.engine;

import com.example.rag.config.ModelRegistry;
import com.example.rag.feature.domain.*;
import com.example.rag.feature.domain.engine.strategy.ExtractionStrategyFactory;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class ConfigDrivenMetadataExtractorRegistry implements MetadataExtractorRegistry {

    private final Map<String, MetadataExtractor> extractorsByDocType;

    private ConfigDrivenMetadataExtractorRegistry(Map<String, MetadataExtractor> extractorsByDocType) {
        this.extractorsByDocType = extractorsByDocType;
    }

    @SuppressWarnings("unchecked")
    public static ConfigDrivenMetadataExtractorRegistry from(
            Map<String, Map<String, Object>> docTypes,
            ExtractionStrategyFactory strategyFactory,
            ModelRegistry modelRegistry,
            DomainModelConfig modelConfig) {

        var extractors = new HashMap<String, MetadataExtractor>();

        for (var entry : docTypes.entrySet()) {
            var docType = entry.getKey();
            var definition = entry.getValue();
            var fieldDefs = (List<Map<String, Object>>) definition.get("metadata");

            var fields = fieldDefs.stream()
                    .map(fieldDef -> {
                        var fieldModelOverride = (String) fieldDef.get("model");
                        var resolvedModelId = modelConfig.resolveExtractionModel(fieldModelOverride);

                        return new FieldDefinition(
                                (String) fieldDef.get("key"),
                                (String) fieldDef.get("type"),
                                strategyFactory.create(fieldDef, modelRegistry, resolvedModelId)
                        );
                    })
                    .toList();

            extractors.put(docType, new ConfigDrivenMetadataExtractor(fields));
        }

        return new ConfigDrivenMetadataExtractorRegistry(extractors);
    }

    @Override
    public MetadataExtractor forDocType(String docType) {
        var extractor = extractorsByDocType.get(docType);
        if (extractor == null) {
            throw new IllegalArgumentException("No metadata extractor for doc_type: " + docType);
        }
        return extractor;
    }

    public record FieldDefinition(String key, String type, ExtractionStrategy strategy) {}
}
```

### Code 3.5 ConfigDrivenMetadataExtractor

```java
package com.example.rag.feature.domain.engine;

import com.example.rag.feature.domain.MetadataExtractor;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class ConfigDrivenMetadataExtractor implements MetadataExtractor {

    private static final Logger log = LoggerFactory.getLogger(ConfigDrivenMetadataExtractor.class);
    private static final int MAX_VALUE_LENGTH = 512;

    private final List<ConfigDrivenMetadataExtractorRegistry.FieldDefinition> fields;

    public ConfigDrivenMetadataExtractor(
            List<ConfigDrivenMetadataExtractorRegistry.FieldDefinition> fields) {
        this.fields = fields;
    }

    @Override
    public Map<String, String> extract(String filename, String text) {
        var metadata = new LinkedHashMap<String, String>();

        for (var field : fields) {
            try {
                var value = field.strategy().extract(text, Map.of());
                if (value != null && !value.isBlank()) {
                    var coerced = coerce(value, field.type());
                    metadata.put(field.key(), truncate(coerced));
                }
            } catch (Exception e) {
                log.warn("Extraction failed for field '{}': {}", field.key(), e.getMessage());
            }
        }

        return metadata;
    }

    private String coerce(String value, String targetType) {
        return switch (targetType) {
            case "integer" -> String.valueOf(Integer.parseInt(value.replaceAll("[^\\d-]", "")));
            case "float" -> String.valueOf(Double.parseDouble(value.replaceAll("[^\\d.eE-]", "")));
            case "boolean" -> String.valueOf(Boolean.parseBoolean(value.strip()));
            default -> value.strip();
        };
    }

    private String truncate(String value) {
        return value.length() > MAX_VALUE_LENGTH
                ? value.substring(0, MAX_VALUE_LENGTH)
                : value;
    }
}
```

### Code 4.3 LlmExtractionStrategy

```java
package com.example.rag.feature.domain.engine.strategy;

import com.example.rag.feature.domain.ExtractionStrategy;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.input.Prompt;
import java.util.Map;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class LlmExtractionStrategy implements ExtractionStrategy {

    private static final Logger log = LoggerFactory.getLogger(LlmExtractionStrategy.class);
    private static final int DEFAULT_MAX_CHARS = 8000;

    private final String prompt;
    private final ChatModel chatModel;

    public LlmExtractionStrategy(String prompt, ChatModel chatModel) {
        this.prompt = prompt;
        this.chatModel = chatModel;
    }

    @Override
    public String extract(String text, Map<String, Object> config) {
        try {
            var truncatedText = text.length() > DEFAULT_MAX_CHARS
                    ? text.substring(0, DEFAULT_MAX_CHARS)
                    : text;

            var fullPrompt = """
                    %s
                    
                    Document:
                    %s""".formatted(prompt, truncatedText);

            var response = chatModel.chat(fullPrompt);
            return response != null && !response.isBlank() ? response.strip() : null;
        } catch (Exception e) {
            log.warn("LLM extraction failed: {}", e.getMessage());
            return null;
        }
    }
}
```

Update `ExtractionStrategyFactory` (see [iteration-03](./iteration-03-extraction-strategies.md)) to add the `case "llm"` that builds `LlmExtractionStrategy` using `ModelRegistry.resolve(resolvedModelId)`.
