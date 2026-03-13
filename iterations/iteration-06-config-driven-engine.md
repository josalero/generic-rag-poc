# Iteration 6 — Config-driven engine (YAML loader + classifier, prompts, guardrails)

> Parent: [implementation-plan.md](../implementation-plan.md) · Previous: [iteration-05-document-parsers.md](./iteration-05-document-parsers.md)

## Goal

Load domain YAML from a path, build a `ConfigDrivenRagDomain` with classifier, prompt provider, and guardrail evaluator. No metadata extraction yet (no LLM, no extractors). Classifier and deterministic guardrails are **custom-algorithm** steps per § 21 (no LLM required for quality). **Prefer:** All classifier rules, prompts, and guardrail rule definitions loaded from domain YAML — no hardcoded rules or prompt templates in code.

## Deliverables

| # | Component | Location in this doc |
|---|-----------|----------------------|
| 1 | `DomainDefinitionLoader` | § Code 3.1 |
| 2 | `ConfigDrivenRagDomain` | § Code 3.2 (extractors can be empty/stub in I6) |
| 3 | `ConfigDrivenDocumentClassifier` | § Code 3.3 |
| 4 | `ConfigDrivenGuardrailEvaluator` | § Code 3.6 |
| 5 | `ConfigDrivenPromptTemplateProvider` | § Code 3.7 |

## Acceptance criteria

- Loader reads YAML from classpath or file path; builds at least one `RagDomain` with correct `domainId`, `supportedFileTypes`, `chunkSize`, `chunkOverlap`.
- Classifier: priority-ordered rules; filename + content keywords; first match wins; fallback rule.
- **Optional LLM classification fallback (optional F):** When `app.ingest.classification.llm-fallback-enabled` or domain `classification.llm-fallback-enabled` is true and the matching rule is the **fallback rule**, the classifier may call an LLM to suggest doc_type (and optionally reasoning). When **store-llm-reasoning** is on (optional G), return or expose reasoning so the ingestion service can persist it in the ledger. See [technical-design.md § 9.1](../technical-design.md#91-optional-llm-based-classification-fallback) and [§ 23.5](../technical-design.md#235-storing-llm-reasoning-track-how-decisions-are-taken).
- Guardrail evaluator: evaluates rules in order; first block wins.
- Prompt provider: returns query and fallback template from YAML.

## Tests to add

- `DomainDefinitionLoaderTest`: load a minimal domain YAML (e.g. one doc_type, one classification rule, one guardrail, one prompt); assert domain id, classifier classifies a sample text to expected doc_type, guardrail blocks/permits sample queries, prompt provider returns expected template.
- `ConfigDrivenDocumentClassifierTest`: given a list of rules (e.g. from map), assert first matching rule's doc_type returned; fallback when no rule matches.
- `ConfigDrivenGuardrailEvaluatorTest`: given list of term/pattern rules, assert blocked when query matches, passed when not.
- When R4 (LLM classification fallback) is implemented: test that with flag on and fallback rule matching, classifier calls LLM and returns doc_type (and optionally reasoning when R3 is on).

## Required capabilities delivered here (see [implementation-plan tracking table](../implementation-plan.md#required-capabilities--tracking-table))

| Id | Capability | Taken into account in this iteration |
|----|------------|--------------------------------------|
| **R4** | LLM classification fallback | Classifier is the extension point; when fallback rule matches and flag is on, call LLM and return doc_type (and optionally reasoning for R3). |
| **R3** | Store LLM reasoning | Classifier/loader should support returning or accepting reasoning when store-llm-reasoning is on so it can be persisted in the ledger (iteration 9/11). |

## Quality gates

- Tests use test YAMLs in `src/test/resources/domains/`; no production YAML paths; no real LLM.

See [iteration-00-quality-gates.md](./iteration-00-quality-gates.md) for gates that apply to every iteration.

---

## Code — Config-driven engine

In I6 you can build **extractors as empty or stub** (e.g. `ConfigDrivenMetadataExtractorRegistry` that returns a no-op extractor per doc_type, or from empty `docTypes`). In Iteration 7 you add the full metadata extractor registry and LLM strategy.

### Code 3.1 DomainDefinitionLoader

```java
package com.example.rag.feature.domain.engine;

import com.example.rag.config.ModelRegistry;
import com.example.rag.feature.domain.DomainModelConfig;
import com.example.rag.feature.domain.DomainRegistry;
import com.example.rag.feature.domain.RagDomain;
import com.example.rag.feature.domain.engine.guardrail.GuardrailRuleFactory;
import com.example.rag.feature.domain.engine.strategy.ExtractionStrategyFactory;
import java.io.IOException;
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

### Code 3.2 ConfigDrivenRagDomain

(Full implementation builds classifier, **extractors** (stub in I6 or full in I7), guardrails, prompts from YAML. See [framework-code.md § 3.2](../framework-code.md#32-configdrivenragdomain) for full listing. In I6 use an empty or stub `ConfigDrivenMetadataExtractorRegistry` so no metadata extraction runs.)

### Code 3.3 ConfigDrivenDocumentClassifier

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

### Code 3.6 ConfigDrivenGuardrailEvaluator

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

### Code 3.7 ConfigDrivenPromptTemplateProvider

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

For full `ConfigDrivenRagDomain` and `ConfigDrivenMetadataExtractorRegistry` / `ConfigDrivenMetadataExtractor` (used in I7), see [framework-code.md § 3.2, 3.4, 3.5](../framework-code.md#32-configdrivenragdomain).
