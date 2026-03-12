# Framework Code Samples

> Parent: [technical-design.md](./technical-design.md) · Related: [extraction-strategies.md](./extraction-strategies.md), [ingestion-pipeline.md](./ingestion-pipeline.md), [query-pipeline.md](./query-pipeline.md)

All code targets **Java 25** and **Spring Boot 4.0** with **LangChain4j 1.11**.

**Code by iteration:** The same code is organized **per iteration** in [iterations/](./iterations/). Each iteration doc (e.g. [iteration-01-foundation.md](./iterations/iteration-01-foundation.md)) contains the plan details (goal, deliverables, acceptance criteria, tests, quality gates) plus the code for that slice. Use the iteration docs when implementing; use this file as a single reference or when you need to see the full codebase in one place.

---

## 1. Core Abstractions (`feature/domain/`)

### 1.1 RagDomain

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

### 1.2 DomainModelConfig

Holds model IDs per purpose. Each domain can specify which model to use for extraction
vs. query generation, and individual fields can override with a per-field model.

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

### 1.3 DocumentClassifier

```java
package com.example.rag.feature.domain;

public interface DocumentClassifier {

    String classify(String filename, String text);
}
```

### 1.4 ExtractionStrategy

```java
package com.example.rag.feature.domain;

import java.util.Map;

public interface ExtractionStrategy {

    String extract(String text, Map<String, Object> config);
}
```

### 1.5 MetadataExtractor

```java
package com.example.rag.feature.domain;

import java.util.Map;

public interface MetadataExtractor {

    Map<String, String> extract(String filename, String text);
}
```

### 1.6 MetadataExtractorRegistry

```java
package com.example.rag.feature.domain;

public interface MetadataExtractorRegistry {

    MetadataExtractor forDocType(String docType);
}
```

### 1.7 GuardrailEvaluator and GuardrailDecision

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

### 1.8 GuardrailRule

```java
package com.example.rag.feature.domain;

public interface GuardrailRule {

    GuardrailDecision evaluate(String query);
}
```

### 1.9 PromptTemplateProvider

```java
package com.example.rag.feature.domain;

public interface PromptTemplateProvider {

    String queryTemplate();

    String fallbackTemplate();
}
```

### 1.10 DomainRegistry

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

---

## 2. Model Registry (`config/`)

### 2.1 ModelDefinitionProperties

```java
package com.example.rag.config;

import java.util.Map;
import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "app.models")
public record ModelDefinitionProperties(
        String defaultModel,
        String embedding,
        Map<String, ModelEntry> definitions
) {
    public record ModelEntry(
            String provider,
            String apiKey,
            String baseUrl,
            String modelName,
            double temperature,
            int maxTokens,
            int timeoutSeconds
    ) {
        public ModelEntry {
            if (temperature == 0.0) temperature = 0.1;
            if (maxTokens == 0) maxTokens = 4096;
            if (timeoutSeconds == 0) timeoutSeconds = 60;
        }
    }
}
```

### 2.2 ModelRegistry

Resolves a model ID to a `ChatModel` instance. Models are built lazily and cached. **All remote API access is via OpenRouter** — no direct OpenAI or other provider clients. OpenAI and other models are used through OpenRouter (e.g. `openai/gpt-4o-mini`); the registry uses OpenRouter’s base URL and a single `OPENROUTER_API_KEY`.

```java
package com.example.rag.config;

import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.openai.OpenAiChatModel;
import java.time.Duration;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import org.springframework.stereotype.Component;

@Component
public class ModelRegistry {

    private final ModelDefinitionProperties properties;
    private final Map<String, ChatModel> cache = new ConcurrentHashMap<>();

    public ModelRegistry(ModelDefinitionProperties properties) {
        this.properties = properties;
    }

    public ChatModel resolve(String modelId) {
        var effectiveId = (modelId != null && !modelId.isBlank())
                ? modelId
                : properties.defaultModel();

        return cache.computeIfAbsent(effectiveId, this::build);
    }

    private ChatModel build(String modelId) {
        var entry = properties.definitions().get(modelId);
        if (entry == null) {
            throw new IllegalArgumentException("Unknown model: '%s'. Available: %s"
                    .formatted(modelId, properties.definitions().keySet()));
        }

        // All remote models via OpenRouter (OpenAI-compatible API). baseUrl from config or default.
        String baseUrl = entry.baseUrl() != null && !entry.baseUrl().isBlank()
                ? entry.baseUrl()
                : "https://openrouter.ai/api/v1";
        if (!"openrouter".equals(entry.provider())) {
            throw new IllegalArgumentException(
                    "Only provider 'openrouter' is supported (all models via OpenRouter). Got: '%s'"
                            .formatted(entry.provider()));
        }
        return OpenAiChatModel.builder()
                .baseUrl(baseUrl)
                .apiKey(entry.apiKey())
                .modelName(entry.modelName())
                .temperature(entry.temperature())
                .maxTokens(entry.maxTokens())
                .timeout(Duration.ofSeconds(entry.timeoutSeconds()))
                .build();
    }
}
```

### 2.3 application.yml — Model Configuration

Design supports **market benchmark** (production) and **free/low-cost** (development). Use **production** config by default; use **development** when `spring.profiles.active=dev`. See [model-recommendations.md](./model-recommendations.md).

**Production (default) — benchmark-grade. All models via OpenRouter; no direct OpenAI or other provider APIs. Use `OPENROUTER_API_KEY` only.**

```yaml
# application.yml or application-prod.yml
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

**Development (profile dev) — free / low-cost:**

```yaml
# application-dev.yml
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

Domain YAML can use **neutral aliases** (`extraction`, `query`) and define them per profile: in prod map to `gpt-4o-mini` / `gpt-4o`, in dev map to `extraction-free` / `query-free`. Or use concrete IDs in domain YAML and override only in application-{profile}.yml. `ModelRegistry` must support `base-url` for OpenRouter. For `embedding: "in-process"`, wire an ONNX `EmbeddingModel` bean; PGVector uses `vector(384)` in dev, `vector(1536)` in prod.

### 2.4 General stop words (config / resource file)

Avoid hardcoding general stop words. Load them from application config or a resource file. The platform supports **English (en)** and **Spanish (es)**; provide language-specific sets so the correct stop words are used per query language (e.g. `general-stop-words-file` for default locale, `general-stop-words-file-es` for Spanish). See [technical-design.md § 14.1 General stop words](./technical-design.md#141-general-stop-words--non-hardcoded-options) and [§ 22 Supported languages](./technical-design.md#22-supported-languages-english-and-spanish).

**Option A — YAML list** in `application.yml`:

```yaml
app:
  query:
    general-stop-words:
      - the
      - and
      - for
      - with
      - from
      - that
      - this
      - have
      - has
      - are
      - was
      - were
      - into
      - about
      - which
      - when
      - where
      - who
      - what
      - how
      - why
      - can
      - you
      - their
      - they
      - them
```

**Option B — Resource file** (one word per line), e.g. `src/main/resources/stopwords/general-en.txt`. Use `general-stop-words-file: classpath:stopwords/general-en.txt` for the default locale (en). For **Spanish**, add `general-stop-words-file-es: classpath:stopwords/general-es.txt` and implement a provider that returns the set for the request locale (en or es).

**Bean that provides the set** (inject into `DomainQueryService`):

```java
package com.example.rag.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.io.IOException;
import java.io.InputStream;
import java.nio.charset.StandardCharsets;
import java.util.HashSet;
import java.util.List;
import java.util.Set;

@Configuration
public class StopWordsConfig {

    @Bean
    public Set<String> generalStopWords(
            @Value("${app.query.general-stop-words:}") List<String> fromYaml,
            @Value("${app.query.general-stop-words-file:}") String filePath) {
        if (filePath != null && !filePath.isBlank()) {
            return loadFromResource(filePath);
        }
        return fromYaml != null && !fromYaml.isEmpty()
                ? Set.copyOf(fromYaml)
                : defaultEnglishStopWords();
    }

    private static Set<String> loadFromResource(String path) {
        path = path.startsWith("classpath:") ? path.substring("classpath:".length()) : path;
        String resourcePath = path.startsWith("/") ? path.substring(1) : path;
        try (InputStream in = StopWordsConfig.class.getClassLoader().getResourceAsStream(resourcePath)) {
            if (in == null) return defaultEnglishStopWords();
            String content = new String(in.readAllBytes(), StandardCharsets.UTF_8);
            Set<String> words = new HashSet<>();
            for (String line : content.lines().toList()) {
                String w = line.trim().toLowerCase();
                if (!w.isBlank() && !w.startsWith("#")) words.add(w);
            }
            return words.isEmpty() ? defaultEnglishStopWords() : Set.copyOf(words);
        } catch (IOException e) {
            throw new IllegalStateException("Failed to load stop words from " + path, e);
        }
    }

    private static Set<String> defaultEnglishStopWords() {
        return Set.of(
                "the", "and", "for", "with", "from", "that", "this", "have", "has",
                "are", "was", "were", "into", "about", "which", "when", "where",
                "who", "what", "how", "why", "can", "you", "their", "they", "them");
    }
}
```

Use file when set; otherwise use YAML list; if both empty, fall back to default English set. For **multi-language (en, es)** support, either inject a `StopWordsProvider` that accepts a locale and returns the appropriate set (loading from `general-stop-words-file` for `en` and `general-stop-words-file-es` for `es`), or inject a single merged set when only one language is used. `DomainQueryService` receives general stop words (per locale or single set) and merges with `domain.stopWords()`; it uses the request language (from query body or `Accept-Language`) to resolve the locale.

---

## 3. Config-Driven Engine (`feature/domain.engine/`)

### 3.1 DomainDefinitionLoader

Reads YAML files from the configured path and builds `ConfigDrivenRagDomain` instances.

```java
package com.example.rag.feature.domain.engine;

import com.example.rag.config.ModelRegistry;
import com.example.rag.feature.domain.DomainModelConfig;
import com.example.rag.feature.domain.DomainRegistry;
import com.example.rag.feature.domain.RagDomain;
import com.example.rag.feature.domain.engine.guardrail.GuardrailRuleFactory;
import com.example.rag.feature.domain.engine.strategy.ExtractionStrategyFactory;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;
import java.util.Map;
import java.util.stream.Stream;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.core.io.Resource;
import org.springframework.core.io.support.PathMatchingResourcePatternResolver;
import org.yaml.snakeyaml.Yaml;

public class DomainDefinitionLoader {

    private static final Logger log = LoggerFactory.getLogger(DomainDefinitionLoader.class);

    private final String definitionsPath;
    private final ExtractionStrategyFactory strategyFactory;
    private final GuardrailRuleFactory guardrailFactory;
    private final ModelRegistry modelRegistry;

    public DomainDefinitionLoader(String definitionsPath,
                                  ExtractionStrategyFactory strategyFactory,
                                  GuardrailRuleFactory guardrailFactory,
                                  ModelRegistry modelRegistry) {
        this.definitionsPath = definitionsPath;
        this.strategyFactory = strategyFactory;
        this.guardrailFactory = guardrailFactory;
        this.modelRegistry = modelRegistry;
    }

    public List<RagDomain> loadAll() throws IOException {
        var resolver = new PathMatchingResourcePatternResolver();
        var resources = resolver.getResources(definitionsPath + "*.yml");

        return Stream.of(resources)
                .map(this::loadOne)
                .filter(domain -> domain != null)
                .toList();
    }

    @SuppressWarnings("unchecked")
    private RagDomain loadOne(Resource resource) {
        try (var input = resource.getInputStream()) {
            var yaml = new Yaml();
            Map<String, Object> root = yaml.load(input);
            Map<String, Object> domainMap = (Map<String, Object>) root.get("domain");

            var enabled = (Boolean) domainMap.get("enabled");
            if (!enabled) {
                log.info("Domain '{}' is disabled — skipping", domainMap.get("id"));
                return null;
            }

            var modelConfig = parseModelConfig(domainMap);

            return ConfigDrivenRagDomain.from(
                    domainMap, strategyFactory, guardrailFactory, modelRegistry, modelConfig);
        } catch (IOException e) {
            log.error("Failed to load domain from {}: {}", resource.getFilename(), e.getMessage());
            return null;
        }
    }

    @SuppressWarnings("unchecked")
    private DomainModelConfig parseModelConfig(Map<String, Object> domainMap) {
        var models = (Map<String, String>) domainMap.getOrDefault("models", Map.of());
        return new DomainModelConfig(
                models.getOrDefault("extraction", null),
                models.getOrDefault("query", null)
        );
    }
}
```

### 3.2 ConfigDrivenRagDomain

```java
package com.example.rag.feature.domain.engine;

import com.example.rag.config.ModelRegistry;
import com.example.rag.feature.domain.*;
import com.example.rag.feature.domain.engine.guardrail.GuardrailRuleFactory;
import com.example.rag.feature.domain.engine.strategy.ExtractionStrategyFactory;
import dev.langchain4j.data.document.DocumentSplitter;
import dev.langchain4j.data.document.splitter.DocumentSplitters;
import java.util.*;

public class ConfigDrivenRagDomain implements RagDomain {

    private final String domainId;
    private final String displayName;
    private final List<String> supportedFileTypes;
    private final int chunkSize;
    private final int chunkOverlap;
    private final Set<String> stopWords;
    private final DomainModelConfig modelConfig;
    private final DocumentClassifier classifier;
    private final MetadataExtractorRegistry extractors;
    private final GuardrailEvaluator guardrails;
    private final PromptTemplateProvider prompts;

    private ConfigDrivenRagDomain(String domainId, String displayName,
                                  List<String> supportedFileTypes, int chunkSize, int chunkOverlap,
                                  Set<String> stopWords, DomainModelConfig modelConfig,
                                  DocumentClassifier classifier, MetadataExtractorRegistry extractors,
                                  GuardrailEvaluator guardrails, PromptTemplateProvider prompts) {
        this.domainId = domainId;
        this.displayName = displayName;
        this.supportedFileTypes = supportedFileTypes;
        this.chunkSize = chunkSize;
        this.chunkOverlap = chunkOverlap;
        this.stopWords = stopWords;
        this.modelConfig = modelConfig;
        this.classifier = classifier;
        this.extractors = extractors;
        this.guardrails = guardrails;
        this.prompts = prompts;
    }

    @SuppressWarnings("unchecked")
    public static ConfigDrivenRagDomain from(Map<String, Object> domainMap,
                                             ExtractionStrategyFactory strategyFactory,
                                             GuardrailRuleFactory guardrailFactory,
                                             ModelRegistry modelRegistry,
                                             DomainModelConfig modelConfig) {
        var id = (String) domainMap.get("id");
        var displayName = (String) domainMap.get("display-name");
        var fileTypes = (List<String>) domainMap.get("supported-file-types");
        var chunkSize = (int) domainMap.get("chunk-size");
        var chunkOverlap = (int) domainMap.get("chunk-overlap");
        var stopWords = Set.copyOf(
                (List<String>) domainMap.getOrDefault("stop-words", List.of()));

        var rules = (List<Map<String, Object>>) domainMap.get("classification-rules");
        var classifier = ConfigDrivenDocumentClassifier.from(rules);

        var docTypes = (Map<String, Map<String, Object>>) domainMap.get("doc-types");
        var extractors = ConfigDrivenMetadataExtractorRegistry.from(
                docTypes, strategyFactory, modelRegistry, modelConfig);

        var guardrailMap = (Map<String, Object>) domainMap.get("guardrails");
        var guardrails = ConfigDrivenGuardrailEvaluator.from(guardrailMap, guardrailFactory);

        var promptMap = (Map<String, String>) domainMap.get("prompts");
        var prompts = new ConfigDrivenPromptTemplateProvider(
                promptMap.get("query"), promptMap.get("fallback"));

        return new ConfigDrivenRagDomain(id, displayName, fileTypes, chunkSize, chunkOverlap,
                stopWords, modelConfig, classifier, extractors, guardrails, prompts);
    }

    @Override public String domainId() { return domainId; }
    @Override public String displayName() { return displayName; }
    @Override public List<String> supportedFileTypes() { return supportedFileTypes; }
    @Override public DocumentSplitter documentSplitter() {
        return DocumentSplitters.recursive(chunkSize, chunkOverlap);
    }
    @Override public DocumentClassifier documentClassifier() { return classifier; }
    @Override public MetadataExtractorRegistry metadataExtractors() { return extractors; }
    @Override public GuardrailEvaluator guardrailEvaluator() { return guardrails; }
    @Override public PromptTemplateProvider promptProvider() { return prompts; }
    @Override public Set<String> stopWords() { return stopWords; }
    @Override public DomainModelConfig modelConfig() { return modelConfig; }
}
```

### 3.3 ConfigDrivenDocumentClassifier

```java
package com.example.rag.feature.domain.engine;

import com.example.rag.feature.domain.DocumentClassifier;
import java.util.*;
import java.util.regex.Pattern;

public class ConfigDrivenDocumentClassifier implements DocumentClassifier {

    private final List<ClassificationRule> rules;

    private ConfigDrivenDocumentClassifier(List<ClassificationRule> rules) {
        this.rules = rules;
    }

    @SuppressWarnings("unchecked")
    public static ConfigDrivenDocumentClassifier from(List<Map<String, Object>> rulesConfig) {
        var rules = rulesConfig.stream()
                .map(r -> {
                    var match = (Map<String, Object>) r.get("match");
                    return new ClassificationRule(
                            (String) r.get("doc-type"),
                            (int) r.get("priority"),
                            ((List<String>) match.get("filename-patterns")).stream()
                                    .map(ConfigDrivenDocumentClassifier::globToRegex)
                                    .toList(),
                            (List<String>) match.get("content-keywords"),
                            (int) match.get("min-keyword-hits")
                    );
                })
                .sorted(Comparator.comparingInt(ClassificationRule::priority).reversed())
                .toList();
        return new ConfigDrivenDocumentClassifier(rules);
    }

    @Override
    public String classify(String filename, String text) {
        var lowerFilename = filename.toLowerCase(Locale.ROOT);
        var lowerText = text.toLowerCase(Locale.ROOT);

        for (var rule : rules) {
            boolean filenameMatch = rule.filenamePatterns().stream()
                    .anyMatch(pattern -> pattern.matcher(lowerFilename).matches());

            if (!filenameMatch) continue;

            long keywordHits = rule.contentKeywords().stream()
                    .filter(kw -> lowerText.contains(kw.toLowerCase(Locale.ROOT)))
                    .count();

            if (keywordHits >= rule.minKeywordHits()) {
                return rule.docType();
            }
        }

        return rules.getLast().docType();
    }

    private static Pattern globToRegex(String glob) {
        var regex = glob.replace(".", "\\.").replace("*", ".*").replace("?", ".");
        return Pattern.compile(regex, Pattern.CASE_INSENSITIVE);
    }

    private record ClassificationRule(
            String docType,
            int priority,
            List<Pattern> filenamePatterns,
            List<String> contentKeywords,
            int minKeywordHits
    ) {}
}
```

### 3.4 ConfigDrivenMetadataExtractorRegistry

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

### 3.5 ConfigDrivenMetadataExtractor

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

### 3.6 ConfigDrivenGuardrailEvaluator

```java
package com.example.rag.feature.domain.engine;

import com.example.rag.feature.domain.GuardrailDecision;
import com.example.rag.feature.domain.GuardrailEvaluator;
import com.example.rag.feature.domain.GuardrailRule;
import com.example.rag.feature.domain.engine.guardrail.GuardrailRuleFactory;
import java.util.List;
import java.util.Map;

public class ConfigDrivenGuardrailEvaluator implements GuardrailEvaluator {

    private final List<GuardrailRule> rules;

    private ConfigDrivenGuardrailEvaluator(List<GuardrailRule> rules) {
        this.rules = rules;
    }

    @SuppressWarnings("unchecked")
    public static ConfigDrivenGuardrailEvaluator from(Map<String, Object> guardrailsMap,
                                                       GuardrailRuleFactory factory) {
        var rulesConfig = (List<Map<String, Object>>) guardrailsMap.get("rules");
        var rules = rulesConfig.stream()
                .map(factory::create)
                .toList();
        return new ConfigDrivenGuardrailEvaluator(rules);
    }

    @Override
    public GuardrailDecision evaluate(String query) {
        for (var rule : rules) {
            var decision = rule.evaluate(query);
            if (decision.blocked()) {
                return decision;
            }
        }
        return GuardrailDecision.PASS;
    }
}
```

### 3.7 ConfigDrivenPromptTemplateProvider

```java
package com.example.rag.feature.domain.engine;

import com.example.rag.feature.domain.PromptTemplateProvider;

public class ConfigDrivenPromptTemplateProvider implements PromptTemplateProvider {

    private final String queryTemplate;
    private final String fallbackTemplate;

    public ConfigDrivenPromptTemplateProvider(String queryTemplate, String fallbackTemplate) {
        this.queryTemplate = queryTemplate;
        this.fallbackTemplate = fallbackTemplate;
    }

    @Override public String queryTemplate() { return queryTemplate; }
    @Override public String fallbackTemplate() { return fallbackTemplate; }
}
```

---

## 4. Extraction Strategies (`feature/domain.engine.strategy/`)

### 4.1 ExtractionStrategyFactory

The factory resolves YAML extraction keys to strategy instances. For `llm` strategies,
it uses the `ModelRegistry` to resolve the appropriate `ChatModel`.

```java
package com.example.rag.feature.domain.engine.strategy;

import com.example.rag.config.ModelRegistry;
import com.example.rag.feature.domain.ExtractionStrategy;
import java.util.List;
import java.util.Map;
import org.springframework.stereotype.Component;

@Component
public class ExtractionStrategyFactory {

    @SuppressWarnings("unchecked")
    public ExtractionStrategy create(Map<String, Object> fieldConfig,
                                     ModelRegistry modelRegistry,
                                     String resolvedModelId) {
        var extraction = (String) fieldConfig.get("extraction");

        return switch (extraction) {
            case "regex" -> new RegexExtractionStrategy(
                    (List<String>) fieldConfig.get("patterns"));

            case "llm" -> new LlmExtractionStrategy(
                    (String) fieldConfig.get("prompt"),
                    modelRegistry.resolve(resolvedModelId));

            case "keyword" -> new KeywordExtractionStrategy(
                    (Map<String, List<String>>) fieldConfig.get("keywords"));

            case "composite" -> {
                var subStrategies = ((List<Map<String, Object>>) fieldConfig.get("strategies"))
                        .stream()
                        .map(sub -> {
                            var subType = (String) sub.get("type");
                            var subConfig = new java.util.HashMap<>(sub);
                            subConfig.put("extraction", subType);
                            return create(subConfig, modelRegistry, resolvedModelId);
                        })
                        .toList();
                yield new CompositeExtractionStrategy(subStrategies);
            }

            default -> throw new IllegalArgumentException(
                    "Unknown extraction strategy: '%s'".formatted(extraction));
        };
    }
}
```

### 4.2 RegexExtractionStrategy

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

### 4.3 LlmExtractionStrategy

Uses the model resolved by the factory — different fields in the same doc_type
can use different models.

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

### 4.4 KeywordExtractionStrategy

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

### 4.5 CompositeExtractionStrategy

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

---

## 5. Guardrail Rules (`feature/domain.engine.guardrail/`)

### 5.1 GuardrailRuleFactory

```java
package com.example.rag.feature.domain.engine.guardrail;

import com.example.rag.config.ModelRegistry;
import com.example.rag.feature.domain.GuardrailRule;
import java.util.List;
import java.util.Map;
import org.springframework.stereotype.Component;

@Component
public class GuardrailRuleFactory {

    private final ModelRegistry modelRegistry;

    public GuardrailRuleFactory(ModelRegistry modelRegistry) {
        this.modelRegistry = modelRegistry;
    }

    @SuppressWarnings("unchecked")
    public GuardrailRule create(Map<String, Object> ruleConfig) {
        var type = (String) ruleConfig.get("type");
        var blockedMessage = (String) ruleConfig.get("blocked-message");

        return switch (type) {
            case "term-block" -> new TermBlockGuardrailRule(
                    (List<String>) ruleConfig.get("trigger-terms"),
                    (List<String>) ruleConfig.getOrDefault("intent-terms", List.of()),
                    (Boolean) ruleConfig.getOrDefault("require-both", false),
                    blockedMessage);

            case "pattern-block" -> new PatternBlockGuardrailRule(
                    (List<String>) ruleConfig.get("patterns"),
                    blockedMessage);

            case "llm-block" -> new LlmGuardrailRule(
                    (String) ruleConfig.get("prompt"),
                    modelRegistry.resolve((String) ruleConfig.get("model")),
                    blockedMessage);

            default -> throw new IllegalArgumentException(
                    "Unknown guardrail rule type: '%s'".formatted(type));
        };
    }
}
```

### 5.2 TermBlockGuardrailRule

```java
package com.example.rag.feature.domain.engine.guardrail;

import com.example.rag.feature.domain.GuardrailDecision;
import com.example.rag.feature.domain.GuardrailRule;
import java.util.List;
import java.util.Locale;

public class TermBlockGuardrailRule implements GuardrailRule {

    private final List<String> triggerTerms;
    private final List<String> intentTerms;
    private final boolean requireBoth;
    private final String blockedMessage;

    public TermBlockGuardrailRule(List<String> triggerTerms, List<String> intentTerms,
                                  boolean requireBoth, String blockedMessage) {
        this.triggerTerms = triggerTerms;
        this.intentTerms = intentTerms;
        this.requireBoth = requireBoth;
        this.blockedMessage = blockedMessage;
    }

    @Override
    public GuardrailDecision evaluate(String query) {
        var lower = query.toLowerCase(Locale.ROOT);

        boolean hasTrigger = triggerTerms.stream().anyMatch(lower::contains);
        if (!hasTrigger) return GuardrailDecision.PASS;

        if (!requireBoth) {
            return GuardrailDecision.block(blockedMessage);
        }

        boolean hasIntent = intentTerms.stream().anyMatch(lower::contains);
        return hasIntent
                ? GuardrailDecision.block(blockedMessage)
                : GuardrailDecision.PASS;
    }
}
```

### 5.3 PatternBlockGuardrailRule

```java
package com.example.rag.feature.domain.engine.guardrail;

import com.example.rag.feature.domain.GuardrailDecision;
import com.example.rag.feature.domain.GuardrailRule;
import java.util.List;
import java.util.regex.Pattern;

public class PatternBlockGuardrailRule implements GuardrailRule {

    private final List<Pattern> patterns;
    private final String blockedMessage;

    public PatternBlockGuardrailRule(List<String> rawPatterns, String blockedMessage) {
        this.patterns = rawPatterns.stream()
                .map(p -> Pattern.compile(p, Pattern.CASE_INSENSITIVE))
                .toList();
        this.blockedMessage = blockedMessage;
    }

    @Override
    public GuardrailDecision evaluate(String query) {
        for (var pattern : patterns) {
            if (pattern.matcher(query).find()) {
                return GuardrailDecision.block(blockedMessage);
            }
        }
        return GuardrailDecision.PASS;
    }
}
```

### 5.4 LlmGuardrailRule

Delegates the safety decision to an LLM. The LLM receives a system prompt that
defines the safety policy and the user's query. It returns `BLOCK` or `PASS`.

This is useful for nuanced checks that deterministic rules can't handle:
subtle bias, context-dependent inappropriateness, obfuscated prompt injection,
or domain-specific compliance (e.g. "is this query requesting legal advice?").

```java
package com.example.rag.feature.domain.engine.guardrail;

import com.example.rag.feature.domain.GuardrailDecision;
import com.example.rag.feature.domain.GuardrailRule;
import dev.langchain4j.model.chat.ChatModel;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class LlmGuardrailRule implements GuardrailRule {

    private static final Logger log = LoggerFactory.getLogger(LlmGuardrailRule.class);

    private final String prompt;
    private final ChatModel chatModel;
    private final String blockedMessage;

    public LlmGuardrailRule(String prompt, ChatModel chatModel, String blockedMessage) {
        this.prompt = prompt;
        this.chatModel = chatModel;
        this.blockedMessage = blockedMessage;
    }

    @Override
    public GuardrailDecision evaluate(String query) {
        try {
            var fullPrompt = """
                    %s
                    
                    Query to evaluate:
                    %s
                    
                    Respond with ONLY one of:
                    - PASS
                    - BLOCK|<reason>""".formatted(prompt, query);

            var response = chatModel.chat(fullPrompt).strip();

            if (response.startsWith("BLOCK")) {
                var reason = response.contains("|")
                        ? response.substring(response.indexOf('|') + 1).strip()
                        : null;
                var message = reason != null && !reason.isBlank()
                        ? blockedMessage + " Reason: " + reason
                        : blockedMessage;
                return GuardrailDecision.block(message);
            }

            return GuardrailDecision.PASS;
        } catch (Exception e) {
            log.warn("LLM guardrail evaluation failed, defaulting to PASS: {}", e.getMessage());
            return GuardrailDecision.PASS;
        }
    }
}
```

**Design decisions:**

| Decision | Rationale |
|---|---|
| Default to PASS on LLM failure | Availability over safety — deterministic rules (term/pattern) still catch the obvious cases. A failed LLM check should not block all queries. |
| Structured response format (`BLOCK\|reason`) | Easy to parse, LLM-friendly, avoids JSON overhead for a binary decision |
| Reason appended to `blocked-message` | The YAML author controls the user-facing message; the LLM reason adds context for logging/debugging |
| Separate `model` per rule | A cheap/fast model (`gpt-4o-mini`) is often sufficient for safety classification; no need to use the query model |

**When to use which guardrail type:**

| Type | Latency | Cost | Best for |
|---|---|---|---|
| `term-block` | ~0ms | Free | Known bad terms (protected attributes, forbidden topics) |
| `pattern-block` | ~0ms | Free | Structural attacks (prompt injection, exfiltration patterns) |
| `llm-block` | 200–2000ms | Per-call | Nuanced safety (subtle bias, context-dependent compliance, obfuscated attacks) |

**Recommended ordering in YAML:** Put cheap deterministic rules first. The `llm-block`
rule should be last — it only runs if all deterministic rules passed, avoiding
unnecessary LLM calls for queries that would have been caught by term/pattern matching.

---

## 6. Document Parsers (`feature/ingest/parser/`)

### 6.1 DocumentParser

```java
package com.example.rag.feature.ingest.parser;

public interface DocumentParser {

    boolean supports(String filename);

    String extractText(byte[] content);
}
```

### 6.2 DocumentParserRegistry

```java
package com.example.rag.feature.ingest.parser;

import java.util.List;
import org.springframework.stereotype.Component;

@Component
public class DocumentParserRegistry {

    private final List<DocumentParser> parsers;

    public DocumentParserRegistry(List<DocumentParser> parsers) {
        this.parsers = parsers;
    }

    public String parse(String filename, byte[] content) {
        for (var parser : parsers) {
            if (parser.supports(filename)) {
                return parser.extractText(content);
            }
        }
        return null;
    }
}
```

### 6.3 PdfDocumentParser

```java
package com.example.rag.feature.ingest.parser;

import java.io.IOException;
import org.apache.pdfbox.Loader;
import org.apache.pdfbox.text.PDFTextStripper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

@Component
@Order(1)
public class PdfDocumentParser implements DocumentParser {

    private static final Logger log = LoggerFactory.getLogger(PdfDocumentParser.class);

    @Override
    public boolean supports(String filename) {
        return filename.toLowerCase().endsWith(".pdf");
    }

    @Override
    public String extractText(byte[] content) {
        try (var document = Loader.loadPDF(content)) {
            var stripper = new PDFTextStripper();
            return stripper.getText(document);
        } catch (IOException e) {
            log.warn("PDF parse failed: {}", e.getMessage());
            return null;
        }
    }
}
```

### 6.4 DocxDocumentParser

```java
package com.example.rag.feature.ingest.parser;

import java.io.ByteArrayInputStream;
import java.io.IOException;
import org.apache.poi.xwpf.extractor.XWPFWordExtractor;
import org.apache.poi.xwpf.usermodel.XWPFDocument;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

@Component
@Order(2)
public class DocxDocumentParser implements DocumentParser {

    private static final Logger log = LoggerFactory.getLogger(DocxDocumentParser.class);

    @Override
    public boolean supports(String filename) {
        return filename.toLowerCase().endsWith(".docx");
    }

    @Override
    public String extractText(byte[] content) {
        try (var doc = new XWPFDocument(new ByteArrayInputStream(content));
             var extractor = new XWPFWordExtractor(doc)) {
            return extractor.getText();
        } catch (IOException e) {
            log.warn("DOCX parse failed: {}", e.getMessage());
            return null;
        }
    }
}
```

### 6.5 PlainTextParser

```java
package com.example.rag.feature.ingest.parser;

import java.nio.charset.StandardCharsets;
import java.util.Set;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

@Component
@Order(10)
public class PlainTextParser implements DocumentParser {

    private static final Set<String> EXTENSIONS = Set.of(".txt", ".md", ".csv");

    @Override
    public boolean supports(String filename) {
        var lower = filename.toLowerCase();
        return EXTENSIONS.stream().anyMatch(lower::endsWith);
    }

    @Override
    public String extractText(byte[] content) {
        return new String(content, StandardCharsets.UTF_8);
    }
}
```

---

## 7. Ingestion Service (`feature/ingest/service/`)

### 7.1 DomainIngestionService

```java
package com.example.rag.feature.ingest.service;

import com.example.rag.feature.domain.DomainRegistry;
import com.example.rag.feature.domain.RagDomain;
import com.example.rag.feature.ingest.parser.DocumentParserRegistry;
import dev.langchain4j.data.document.Document;
import dev.langchain4j.data.document.Metadata;
import dev.langchain4j.store.embedding.EmbeddingStoreIngestor;
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

@Service
public class DomainIngestionService {

    private static final Logger log = LoggerFactory.getLogger(DomainIngestionService.class);

    private final DomainRegistry domainRegistry;
    private final DocumentParserRegistry parserRegistry;
    private final EmbeddingStoreIngestor ingestor;
    private final ConcurrentHashMap<String, CompletableFuture<Void>> dedupGates =
            new ConcurrentHashMap<>();

    public DomainIngestionService(DomainRegistry domainRegistry,
                                  DocumentParserRegistry parserRegistry,
                                  EmbeddingStoreIngestor ingestor) {
        this.domainRegistry = domainRegistry;
        this.parserRegistry = parserRegistry;
        this.ingestor = ingestor;
    }

    public IngestResult ingest(String domainId, String filename, byte[] content) {
        var domain = domainRegistry.get(domainId)
                .orElseThrow(() -> new DomainNotFoundException(domainId));

        validateFileType(domain, filename);

        var rawText = parserRegistry.parse(filename, content);
        if (rawText == null || rawText.isBlank()) {
            return IngestResult.skipped(filename, "No extractable text");
        }

        var cleanText = sanitize(rawText);
        if (cleanText.isBlank()) {
            return IngestResult.skipped(filename, "Empty after sanitization");
        }

        var contentHash = sha256(cleanText);
        if (isDuplicate(contentHash)) {
            return IngestResult.skipped(filename, "Duplicate content");
        }

        var docType = domain.documentClassifier().classify(filename, cleanText);
        var extractor = domain.metadataExtractors().forDocType(docType);
        var extensionMetadata = extractor.extract(filename, cleanText);

        var metadata = new Metadata();
        metadata.put("domain", domainId);
        metadata.put("doc_type", docType);
        metadata.put("source", filename);
        metadata.put("content_hash", contentHash);
        metadata.put("ingested_at", Instant.now().toString());
        extensionMetadata.forEach(metadata::put);

        var document = Document.from(cleanText, metadata);
        ingestor.ingest(document);

        return IngestResult.ingested(filename, docType, extensionMetadata.size());
    }

    public List<IngestResult> ingestBatch(String domainId, Map<String, byte[]> files) {
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            var futures = files.entrySet().stream()
                    .map(entry -> CompletableFuture.supplyAsync(
                            () -> ingestSafe(domainId, entry.getKey(), entry.getValue()),
                            executor))
                    .toList();

            return futures.stream()
                    .map(CompletableFuture::join)
                    .toList();
        }
    }

    private IngestResult ingestSafe(String domainId, String filename, byte[] content) {
        try {
            return ingest(domainId, filename, content);
        } catch (Exception e) {
            log.error("Ingestion failed for '{}': {}", filename, e.getMessage());
            return IngestResult.failed(filename, e.getMessage());
        }
    }

    private void validateFileType(RagDomain domain, String filename) {
        var lower = filename.toLowerCase();
        boolean supported = domain.supportedFileTypes().stream().anyMatch(lower::endsWith);
        if (!supported) {
            throw new UnsupportedFileTypeException(filename, domain.supportedFileTypes());
        }
    }

    private String sanitize(String text) {
        return text
                .replace("\u0000", "")
                .replaceAll("[\\x01-\\x08\\x0B\\x0C\\x0E-\\x1F]", "")
                .replaceAll("\\s+", " ")
                .strip();
    }

    private String sha256(String text) {
        try {
            var digest = MessageDigest.getInstance("SHA-256");
            var hash = digest.digest(text.getBytes(StandardCharsets.UTF_8));
            return HexFormat.of().formatHex(hash);
        } catch (Exception e) {
            throw new RuntimeException("SHA-256 unavailable", e);
        }
    }

    private boolean isDuplicate(String contentHash) {
        var gate = new CompletableFuture<Void>();
        var existing = dedupGates.putIfAbsent(contentHash, gate);
        return existing != null;
    }

    public record IngestResult(String filename, Status status, String docType,
                                int fieldsExtracted, String reason) {

        public enum Status { INGESTED, SKIPPED, FAILED }

        static IngestResult ingested(String filename, String docType, int fieldsExtracted) {
            return new IngestResult(filename, Status.INGESTED, docType, fieldsExtracted, null);
        }
        static IngestResult skipped(String filename, String reason) {
            return new IngestResult(filename, Status.SKIPPED, null, 0, reason);
        }
        static IngestResult failed(String filename, String reason) {
            return new IngestResult(filename, Status.FAILED, null, 0, reason);
        }
    }
}
```

### 7.2 Exception classes

```java
package com.example.rag.feature.ingest.service;

import java.util.List;

public class DomainNotFoundException extends RuntimeException {
    public DomainNotFoundException(String domainId) {
        super("Domain not found: '%s'".formatted(domainId));
    }
}
```

```java
package com.example.rag.feature.ingest.service;

import java.util.List;

public class UnsupportedFileTypeException extends RuntimeException {
    public UnsupportedFileTypeException(String filename, List<String> supported) {
        super("File '%s' not supported. Accepted: %s".formatted(filename, supported));
    }
}
```

---

## 8. Query Service (`feature/query/service/`)

### 8.1 DomainQueryService

The query service resolves the domain's query model to generate the answer.
The model used for answers can differ from the model used for extraction.

```java
package com.example.rag.feature.query.service;

import com.example.rag.config.ModelRegistry;
import com.example.rag.feature.domain.DomainRegistry;
import com.example.rag.feature.domain.GuardrailDecision;
import com.example.rag.feature.domain.RagDomain;
import com.example.rag.feature.ingest.service.DomainNotFoundException;
import dev.langchain4j.data.segment.TextSegment;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.rag.content.Content;
import dev.langchain4j.rag.content.retriever.EmbeddingStoreContentRetriever;
import dev.langchain4j.store.embedding.EmbeddingStore;
import dev.langchain4j.store.embedding.filter.MetadataFilterBuilder;
import java.util.*;
import java.util.stream.Collectors;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

@Service
public class DomainQueryService {

    private static final Logger log = LoggerFactory.getLogger(DomainQueryService.class);

    private final DomainRegistry domainRegistry;
    private final ModelRegistry modelRegistry;
    private final EmbeddingStoreContentRetriever.Builder retrieverBuilder;
    private final Set<String> generalStopWords;

    public DomainQueryService(DomainRegistry domainRegistry,
                              ModelRegistry modelRegistry,
                              EmbeddingStoreContentRetriever.Builder retrieverBuilder,
                              Set<String> generalStopWords) {
        this.domainRegistry = domainRegistry;
        this.modelRegistry = modelRegistry;
        this.retrieverBuilder = retrieverBuilder;
        this.generalStopWords = generalStopWords != null ? generalStopWords : Set.of();
    }

    public QueryResult query(String domainId, QueryRequest request) {
        var domain = domainRegistry.get(domainId)
                .orElseThrow(() -> new DomainNotFoundException(domainId));

        var guardrailResult = domain.guardrailEvaluator().evaluate(request.question());
        if (guardrailResult.blocked()) {
            return QueryResult.blocked(guardrailResult.message());
        }

        var queryTerms = extractTerms(request.question(), domain);

        var filter = MetadataFilterBuilder.metadataKey("domain").isEqualTo(domainId);
        var retriever = retrieverBuilder
                .filter(filter)
                .maxResults(request.maxResultsOrDefault(50))
                .build();

        var contents = retriever.retrieve(dev.langchain4j.rag.query.Query.from(request.question()));

        var scored = contents.stream()
                .map(content -> score(content, queryTerms))
                .filter(s -> s.hybridScore() >= request.minScoreOrDefault(0.75))
                .sorted(Comparator.comparingDouble(ScoredContent::hybridScore).reversed())
                .toList();

        var deduped = deduplicate(scored);

        var queryModelId = domain.modelConfig().queryModelId();
        var chatModel = modelRegistry.resolve(queryModelId);

        var answer = generateAnswer(domain, chatModel, deduped, request.question());

        var page = request.pageOrDefault(1);
        var pageSize = request.pageSizeOrDefault(10);
        var paged = paginate(deduped, page, pageSize);

        return new QueryResult(answer, paged, page, pageSize, deduped.size(),
                buildExplainability(queryTerms, scored));
    }

    private List<String> extractTerms(String question, RagDomain domain) {
        var allStopWords = new HashSet<>(generalStopWords);
        allStopWords.addAll(domain.stopWords());

        return Arrays.stream(question.toLowerCase(Locale.ROOT).split("[^a-z0-9]+"))
                .filter(t -> t.length() >= 3)
                .filter(t -> !allStopWords.contains(t))
                .distinct()
                .limit(8)
                .toList();
    }

    private ScoredContent score(Content content, List<String> queryTerms) {
        var text = content.textSegment().text().toLowerCase(Locale.ROOT);
        var matched = queryTerms.stream().filter(text::contains).toList();
        var missing = queryTerms.stream().filter(t -> !text.contains(t)).toList();

        double vectorScore = content.score();
        double keywordScore = queryTerms.isEmpty() ? 0.0
                : (double) matched.size() / queryTerms.size();
        double hybridScore = (vectorScore * 0.8) + (keywordScore * 0.2);

        return new ScoredContent(content, vectorScore, keywordScore, hybridScore,
                matched, missing);
    }

    private List<ScoredContent> deduplicate(List<ScoredContent> scored) {
        var seen = new LinkedHashMap<String, ScoredContent>();
        for (var item : scored) {
            var key = deduplicationKey(item);
            seen.merge(key, item, (existing, incoming) ->
                    incoming.hybridScore() > existing.hybridScore() ? incoming : existing);
        }
        return List.copyOf(seen.values());
    }

    private String deduplicationKey(ScoredContent item) {
        var metadata = item.content().textSegment().metadata();
        var entityId = metadata.getString("entity_id");
        if (entityId != null && !entityId.isBlank()) return "entity:" + entityId;
        return "source:" + metadata.getString("source");
    }

    private String generateAnswer(RagDomain domain, ChatModel chatModel,
                                  List<ScoredContent> results, String question) {
        if (results.isEmpty()) return "No matching documents found.";

        var context = results.stream()
                .limit(20)
                .map(s -> s.content().textSegment().text())
                .collect(Collectors.joining("\n\n"));

        var prompt = domain.promptProvider().queryTemplate().formatted(context, question);

        try {
            return chatModel.chat(prompt);
        } catch (Exception e) {
            log.warn("LLM answer generation failed: {}", e.getMessage());
            return domain.promptProvider().fallbackTemplate().formatted(results.size());
        }
    }

    private List<ScoredContent> paginate(List<ScoredContent> all, int page, int pageSize) {
        int start = (page - 1) * pageSize;
        int end = Math.min(start + pageSize, all.size());
        return start >= all.size() ? List.of() : all.subList(start, end);
    }

    private Explainability buildExplainability(List<String> queryTerms,
                                                List<ScoredContent> scored) {
        var allMatched = scored.stream()
                .flatMap(s -> s.matchedTerms().stream())
                .distinct()
                .toList();
        var allMissing = queryTerms.stream()
                .filter(t -> !allMatched.contains(t))
                .toList();
        var confidence = scored.stream()
                .limit(3)
                .mapToDouble(ScoredContent::hybridScore)
                .average()
                .orElse(0.0);

        return new Explainability(allMatched, allMissing, confidence);
    }

    public record QueryRequest(
            String question,
            List<String> docTypes,
            Integer maxResults,
            Double minScore,
            Integer page,
            Integer pageSize
    ) {
        int maxResultsOrDefault(int def) { return maxResults != null ? maxResults : def; }
        double minScoreOrDefault(double def) { return minScore != null ? minScore : def; }
        int pageOrDefault(int def) { return page != null ? page : def; }
        int pageSizeOrDefault(int def) { return pageSize != null ? pageSize : def; }
    }

    public record QueryResult(
            String answer,
            List<ScoredContent> sources,
            int page,
            int pageSize,
            int totalSources,
            Explainability explainability
    ) {
        static QueryResult blocked(String message) {
            return new QueryResult(message, List.of(), 1, 10, 0,
                    new Explainability(List.of(), List.of(), 0.0));
        }
    }

    public record ScoredContent(
            Content content,
            double vectorScore,
            double keywordScore,
            double hybridScore,
            List<String> matchedTerms,
            List<String> missingTerms
    ) {}

    public record Explainability(
            List<String> matchedTerms,
            List<String> missingTerms,
            double confidence
    ) {}
}
```

---

## 9. REST Controllers

### 9.1 DomainIngestController

```java
package com.example.rag.feature.ingest.controller;

import com.example.rag.feature.ingest.service.DomainIngestionService;
import com.example.rag.feature.ingest.service.DomainIngestionService.IngestResult;
import java.io.IOException;
import java.util.LinkedHashMap;
import java.util.List;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

@RestController
@RequestMapping("/api/v1/{domainId}/ingest")
public class DomainIngestController {

    private final DomainIngestionService ingestionService;

    public DomainIngestController(DomainIngestionService ingestionService) {
        this.ingestionService = ingestionService;
    }

    @PostMapping
    public ResponseEntity<List<IngestResult>> ingest(
            @PathVariable String domainId,
            @RequestParam("files") List<MultipartFile> files) throws IOException {

        var fileMap = new LinkedHashMap<String, byte[]>();
        for (var file : files) {
            fileMap.put(file.getOriginalFilename(), file.getBytes());
        }

        var results = ingestionService.ingestBatch(domainId, fileMap);
        return ResponseEntity.ok(results);
    }
}
```

### 9.2 DomainQueryController

```java
package com.example.rag.feature.query.controller;

import com.example.rag.feature.query.service.DomainQueryService;
import com.example.rag.feature.query.service.DomainQueryService.QueryRequest;
import com.example.rag.feature.query.service.DomainQueryService.QueryResult;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/{domainId}/query")
public class DomainQueryController {

    private final DomainQueryService queryService;

    public DomainQueryController(DomainQueryService queryService) {
        this.queryService = queryService;
    }

    @PostMapping
    public ResponseEntity<QueryResult> query(
            @PathVariable String domainId,
            @RequestBody QueryRequest request) {

        var result = queryService.query(domainId, request);
        return ResponseEntity.ok(result);
    }
}
```

### 9.3 DomainAdminController

```java
package com.example.rag.feature.admin.controller;

import com.example.rag.feature.domain.DomainRegistry;
import com.example.rag.feature.domain.RagDomain;
import com.example.rag.feature.domain.engine.DomainDefinitionLoader;
import java.io.IOException;
import java.util.List;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1")
public class DomainAdminController {

    private final DomainRegistry domainRegistry;
    private final DomainDefinitionLoader loader;

    public DomainAdminController(DomainRegistry domainRegistry,
                                  DomainDefinitionLoader loader) {
        this.domainRegistry = domainRegistry;
        this.loader = loader;
    }

    @GetMapping("/domains")
    public ResponseEntity<List<DomainSummary>> listDomains() {
        var summaries = domainRegistry.all().stream()
                .map(d -> new DomainSummary(d.domainId(), d.displayName(),
                        d.supportedFileTypes()))
                .toList();
        return ResponseEntity.ok(summaries);
    }

    @GetMapping("/{domainId}/doc-types")
    public ResponseEntity<List<String>> listDocTypes(@PathVariable String domainId) {
        var domain = domainRegistry.get(domainId)
                .orElseThrow(() -> new com.example.rag.feature.ingest.service
                        .DomainNotFoundException(domainId));
        // doc types are the keys of the metadata extractor registry
        return ResponseEntity.ok(List.of()); // populated from registry
    }

    @PostMapping("/admin/domains/reload")
    public ResponseEntity<String> reload() throws IOException {
        domainRegistry.clear();
        var domains = loader.loadAll();
        domains.forEach(domainRegistry::register);
        return ResponseEntity.ok("Reloaded %d domains".formatted(domains.size()));
    }

    public record DomainSummary(String id, String displayName, List<String> supportedFileTypes) {}
}
```

---

## 10. Auto-Configuration (`config/`)

### 10.1 DomainAutoConfiguration

```java
package com.example.rag.config;

import com.example.rag.feature.domain.DomainRegistry;
import com.example.rag.feature.domain.RagDomain;
import com.example.rag.feature.domain.engine.DomainDefinitionLoader;
import com.example.rag.feature.domain.engine.guardrail.GuardrailRuleFactory;
import com.example.rag.feature.domain.engine.strategy.ExtractionStrategyFactory;
import java.io.IOException;
import java.util.List;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableConfigurationProperties(ModelDefinitionProperties.class)
public class DomainAutoConfiguration {

    private static final Logger log = LoggerFactory.getLogger(DomainAutoConfiguration.class);

    @Value("${app.domains.definitions-path:classpath:domains/}")
    private String definitionsPath;

    @Bean
    public ModelRegistry modelRegistry(ModelDefinitionProperties properties) {
        return new ModelRegistry(properties);
    }

    @Bean
    public DomainDefinitionLoader domainDefinitionLoader(
            ExtractionStrategyFactory strategyFactory,
            GuardrailRuleFactory guardrailFactory,
            ModelRegistry modelRegistry) {
        return new DomainDefinitionLoader(
                definitionsPath, strategyFactory, guardrailFactory, modelRegistry);
    }

    @Bean
    public DomainRegistry domainRegistry(DomainDefinitionLoader loader,
                                          List<RagDomain> handcodedDomains) throws IOException {
        var registry = new DomainRegistry();

        var yamlDomains = loader.loadAll();
        yamlDomains.forEach(registry::register);

        handcodedDomains.stream()
                .filter(d -> registry.get(d.domainId()).isEmpty())
                .forEach(registry::register);

        log.info("Loaded {} domains: {}", registry.all().size(),
                registry.all().stream().map(RagDomain::domainId).toList());

        return registry;
    }
}
```

---

## 11. Handcoded Domain Override (escape hatch)

### 11.1 RecruitingDomainOverride

When YAML is insufficient (e.g. complex skill normalization), implement `RagDomain` directly:

```java
package com.example.rag.feature.domain.recruiting;

import com.example.rag.feature.domain.*;
import dev.langchain4j.data.document.DocumentSplitter;
import dev.langchain4j.data.document.splitter.DocumentSplitters;
import java.util.List;
import java.util.Set;
import org.springframework.stereotype.Component;

@Component
public class RecruitingDomainOverride implements RagDomain {

    private final TechnicalRoleCatalog roleCatalog;
    private final DocumentClassifier classifier;
    private final MetadataExtractorRegistry extractors;
    private final GuardrailEvaluator guardrails;
    private final PromptTemplateProvider prompts;
    private final DomainModelConfig modelConfig;

    public RecruitingDomainOverride(TechnicalRoleCatalog roleCatalog,
                                    DocumentClassifier classifier,
                                    MetadataExtractorRegistry extractors,
                                    GuardrailEvaluator guardrails,
                                    PromptTemplateProvider prompts,
                                    DomainModelConfig modelConfig) {
        this.roleCatalog = roleCatalog;
        this.classifier = classifier;
        this.extractors = extractors;
        this.guardrails = guardrails;
        this.prompts = prompts;
        this.modelConfig = modelConfig;
    }

    @Override public String domainId() { return "recruiting"; }
    @Override public String displayName() { return "Recruiting (custom)"; }
    @Override public List<String> supportedFileTypes() { return List.of(".pdf", ".docx"); }
    @Override public DocumentSplitter documentSplitter() {
        return DocumentSplitters.recursive(500, 100);
    }
    @Override public DocumentClassifier documentClassifier() { return classifier; }
    @Override public MetadataExtractorRegistry metadataExtractors() { return extractors; }
    @Override public GuardrailEvaluator guardrailEvaluator() { return guardrails; }
    @Override public PromptTemplateProvider promptProvider() { return prompts; }
    @Override public Set<String> stopWords() {
        return Set.of("candidate", "candidates", "resume", "resumes");
    }
    @Override public DomainModelConfig modelConfig() { return modelConfig; }
}
```

---

## 12. Domain YAML — Model Configuration

### 12.1 Domain-level model selection

```yaml
domain:
  id: recruiting
  display-name: "Recruiting"
  enabled: true

  # Model configuration — references model IDs defined in application.yml
  models:
    extraction: "gpt-4o-mini"       # cheap/fast for metadata extraction
    query: "gpt-4o"                 # capable model for answer generation

  # ... rest of domain config
```

### 12.2 Per-field model override

Individual metadata fields can override the domain default when a specific
field needs a more capable (or cheaper) model:

```yaml
  doc-types:
    court_filing:
      display-name: "Court Filing"
      metadata:
        - key: case_number
          type: string
          extraction: regex
          patterns:
            - "(?:case|docket)\\s*(?:no\\.?|#)[:\\s]*([A-Z0-9-:]+)"

        - key: holding_summary
          type: string
          extraction: llm
          model: "deepseek-r1"          # complex reasoning needs stronger model
          prompt: "Summarize the court's holding in one sentence."

        - key: case_title
          type: string
          extraction: llm
          # no 'model' → inherits domain.models.extraction = "gpt-4o-mini"
          prompt: "Extract the case caption."
```

### 12.3 Resolution order

```text
1. Field-level 'model' key        → if present, use this model
2. Domain-level 'models.extraction'→ if present, use this for extraction
3. Domain-level 'models.query'    → used for answer generation
4. Application-level 'app.models.default-model' → global fallback
```

### 12.4 Use case matrix

| Purpose | Typical model | Why |
|---|---|---|
| Metadata extraction (names, skills, dates) | `gpt-4o-mini` | High throughput, low cost per document |
| Complex analysis (legal holdings, summaries) | `gpt-4o` or `deepseek-r1` | Needs reasoning capability |
| Answer generation (query) | `gpt-4o` | User-facing — quality matters |
| Classification (future) | `gpt-4o-mini` | Simple categorization task |
| Batch extraction (resume fields) | `gpt-4o-mini` | 5 fields in one call, cost-sensitive |
| Sensitive domains (medical) | `claude-sonnet` | Different provider for compliance |
| Air-gapped / on-prem | `llama-local` | No data leaves the network |

---

## 13. Global Exception Handling (`config/`)

Uses Spring Boot's RFC 9457 `ProblemDetail` for structured error responses.

```java
package com.example.rag.config;

import com.example.rag.feature.ingest.service.DomainNotFoundException;
import com.example.rag.feature.ingest.service.UnsupportedFileTypeException;
import java.time.Instant;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.multipart.MaxUploadSizeExceededException;

@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(DomainNotFoundException.class)
    public ProblemDetail handleDomainNotFound(DomainNotFoundException ex) {
        var problem = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Domain not found");
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }

    @ExceptionHandler(UnsupportedFileTypeException.class)
    public ProblemDetail handleUnsupportedFileType(UnsupportedFileTypeException ex) {
        var problem = ProblemDetail.forStatusAndDetail(
                HttpStatus.UNPROCESSABLE_ENTITY, ex.getMessage());
        problem.setTitle("Unsupported file type");
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }

    @ExceptionHandler(MaxUploadSizeExceededException.class)
    public ProblemDetail handleMaxUploadSize(MaxUploadSizeExceededException ex) {
        var problem = ProblemDetail.forStatusAndDetail(
                HttpStatus.PAYLOAD_TOO_LARGE, "File exceeds maximum upload size");
        problem.setTitle("Payload too large");
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        var errors = ex.getFieldErrors().stream()
                .map(e -> e.getField() + ": " + e.getDefaultMessage())
                .toList();
        var problem = ProblemDetail.forStatusAndDetail(
                HttpStatus.BAD_REQUEST, "Validation failed");
        problem.setTitle("Invalid request");
        problem.setProperty("errors", errors);
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }

    @ExceptionHandler(Exception.class)
    public ProblemDetail handleUnexpected(Exception ex) {
        log.error("Unexpected error", ex);
        var problem = ProblemDetail.forStatusAndDetail(
                HttpStatus.INTERNAL_SERVER_ERROR, "An unexpected error occurred");
        problem.setTitle("Internal error");
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }
}
```

---

## 14. Request / Response DTOs with Jakarta Validation

### 14.1 QueryRequest

```java
package com.example.rag.feature.query.controller;

import jakarta.validation.constraints.Max;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;
import java.util.List;

public record QueryRequest(

        @NotBlank(message = "Question is required")
        @Size(max = 2000, message = "Question must not exceed 2000 characters")
        String question,

        List<String> docTypes,

        @Min(value = 1, message = "maxResults must be at least 1")
        @Max(value = 200, message = "maxResults must not exceed 200")
        Integer maxResults,

        @Min(value = 0, message = "minScore must be at least 0")
        @Max(value = 1, message = "minScore must not exceed 1")
        Double minScore,

        @Min(value = 1, message = "page must be at least 1")
        Integer page,

        @Min(value = 1, message = "pageSize must be at least 1")
        @Max(value = 100, message = "pageSize must not exceed 100")
        Integer pageSize
) {
    public int effectiveMaxResults() { return maxResults != null ? maxResults : 50; }
    public double effectiveMinScore() { return minScore != null ? minScore : 0.75; }
    public int effectivePage() { return page != null ? page : 1; }
    public int effectivePageSize() { return pageSize != null ? pageSize : 10; }
}
```

### 14.2 QueryResponse

```java
package com.example.rag.feature.query.controller;

import java.util.List;

public record QueryResponse(
        String answer,
        List<SourceResult> sources,
        int page,
        int pageSize,
        int totalSources,
        Explainability explainability
) {
    public record SourceResult(
            String text,
            String source,
            double score,
            int rank,
            double vectorScore,
            double keywordScore,
            List<String> matchedTerms,
            List<String> missingTerms
    ) {}

    public record Explainability(
            List<String> matchedTerms,
            List<String> missingTerms,
            double confidence
    ) {}
}
```

### 14.3 IngestResponse

```java
package com.example.rag.feature.ingest.controller;

import java.util.List;

public record IngestResponse(
        int processed,
        int skipped,
        int failed,
        long elapsedMs,
        List<FileResult> files
) {
    public record FileResult(
            String filename,
            String status,
            String docType,
            int fieldsExtracted,
            String reason
    ) {}
}
```

### 14.4 Updated DomainQueryController with validation

```java
package com.example.rag.feature.query.controller;

import com.example.rag.feature.query.service.DomainQueryService;
import jakarta.validation.Valid;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/{domainId}/query")
public class DomainQueryController {

    private final DomainQueryService queryService;

    public DomainQueryController(DomainQueryService queryService) {
        this.queryService = queryService;
    }

    @PostMapping
    public ResponseEntity<QueryResponse> query(
            @PathVariable String domainId,
            @Valid @RequestBody QueryRequest request) {

        var result = queryService.query(domainId, request);
        return ResponseEntity.ok(result);
    }
}
```

---

## 15. Virtual Thread Configuration

```java
package com.example.rag.config;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.web.embedded.tomcat.TomcatProtocolHandlerCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AsyncConfig {

    @Bean
    @ConditionalOnProperty(name = "app.ingest.virtual-threads-enabled", havingValue = "true",
            matchIfMissing = true)
    public ExecutorService ingestionExecutor() {
        return Executors.newVirtualThreadPerTaskExecutor();
    }

    @Bean
    public TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadExecutorCustomizer() {
        return protocolHandler ->
                protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
    }
}
```

---

## 16. Actuator Health Indicator

```java
package com.example.rag.config;

import com.example.rag.feature.domain.DomainRegistry;
import com.example.rag.feature.domain.RagDomain;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

@Component("domainRegistry")
public class DomainHealthIndicator implements HealthIndicator {

    private final DomainRegistry domainRegistry;

    public DomainHealthIndicator(DomainRegistry domainRegistry) {
        this.domainRegistry = domainRegistry;
    }

    @Override
    public Health health() {
        var domains = domainRegistry.all();
        if (domains.isEmpty()) {
            return Health.down()
                    .withDetail("reason", "No domains loaded")
                    .build();
        }
        return Health.up()
                .withDetail("domainCount", domains.size())
                .withDetail("domains", domains.stream().map(RagDomain::domainId).toList())
                .build();
    }
}
```

---

## 17. build.gradle

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '4.0.3'
    id 'io.spring.dependency-management' version '1.1.6'
}

java {
    toolchain { languageVersion = JavaLanguageVersion.of(25) }
}

repositories {
    mavenCentral()
}

ext {
    langchain4jVersion = '1.11.0'
}

dependencyManagement {
    imports {
        mavenBom "org.springframework.boot:spring-boot-dependencies:4.0.3"
    }
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-jdbc'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'org.springframework.boot:spring-boot-starter-flyway'
    implementation 'org.flywaydb:flyway-database-postgresql'

    implementation platform('dev.langchain4j:langchain4j-bom:' + langchain4jVersion)
    implementation 'dev.langchain4j:langchain4j'
    implementation 'dev.langchain4j:langchain4j-open-ai'
    implementation 'dev.langchain4j:langchain4j-pgvector'
    implementation 'dev.langchain4j:langchain4j-document-parser-apache-pdfbox'
    // In-process embedding for development (free, 384 dims); use vector(384) in dev PGVector
    implementation 'dev.langchain4j:langchain4j-embeddings-bge-small-en-v1.5-q:' + langchain4jVersion
    implementation 'org.apache.pdfbox:pdfbox:3.0.4'
    implementation 'org.apache.poi:poi-ooxml:5.3.0'

    implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:3.0.1'

    runtimeOnly 'com.h2database:h2'
    runtimeOnly 'org.postgresql:postgresql'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

tasks.named('test') {
    useJUnitPlatform()
}
```
