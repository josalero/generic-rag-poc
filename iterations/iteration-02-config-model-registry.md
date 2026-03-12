# Iteration 2 — Config & model registry

> Parent: [implementation-plan.md](../implementation-plan.md) · Previous: [iteration-01-foundation.md](./iteration-01-foundation.md)

## Goal

Load model definitions from `application.yml` (and profile-specific `application-{profile}.yml`) and provide a `ModelRegistry` that resolves a model id to a `ChatModel`. Add general stop words (config or resource file). Config should support both prod and dev profiles (Iteration 12 adds the profile files). **Prefer:** No hardcoded model URLs or stop-word lists in code — use config/resource files or YAML only.

## Deliverables

| # | Component | Location in this doc |
|---|-----------|----------------------|
| 1 | `ModelDefinitionProperties` | § Code 2.1 |
| 2 | `ModelRegistry` | § Code 2.2 |
| 3 | `application.yml` — model config snippet | § Code 2.3 |
| 4 | `StopWordsConfig` (general stop words bean) | § Code 2.4 |

## Acceptance criteria

- `ModelRegistry.resolve(modelId)` returns a `ChatModel` for a configured id; unknown id throws or returns optional (align with framework-code).
- General stop words load from `app.query.general-stop-words` (list) or `app.query.general-stop-words-file` (classpath file); fallback to default English set when both empty. **Language support (en, es):** support per-locale files (e.g. `general-stop-words-file-es`) and a provider or bean that returns the correct set for the request language.
- Bean `Set<String> generalStopWords` (or locale-aware provider) available for injection; supported languages: **en**, **es**.

## Tests to add

- `ModelRegistryTest`: with `@TestConfiguration` and a test `application.yml` (or `@ConfigurationProperties` test), resolve known id returns non-null; resolve unknown behaves as designed.
- `StopWordsConfigTest`: with test profile YAML, verify loaded set contains expected words; with classpath file, verify file wins over list; with empty config, verify default set.
- Use mock or stub `ChatModel` where needed (e.g. LangChain4j test utilities or simple impl).

## Quality gates

- All tests pass; no real API keys in tests.

See [iteration-00-quality-gates.md](./iteration-00-quality-gates.md) for gates that apply to every iteration.

---

## Code — Model Registry and stop words (`config/`)

### Code 2.1 ModelDefinitionProperties

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

### Code 2.2 ModelRegistry

Resolves a model ID to a `ChatModel` instance. Models are built lazily and cached. **All remote API access is via OpenRouter** — no direct OpenAI or other provider clients.

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

### Code 2.3 application.yml — Model configuration

**Production (default) — benchmark-grade. All models via OpenRouter; use `OPENROUTER_API_KEY` only.**

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
```

**Development (profile dev):**

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

### Code 2.4 StopWordsConfig

General stop words from config or resource file; support en/es via per-locale files.

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

Optionally add `general-stop-words-file-es` and a locale-aware provider for Spanish (see [technical-design.md § 22](../technical-design.md#22-supported-languages-english-and-spanish)).
