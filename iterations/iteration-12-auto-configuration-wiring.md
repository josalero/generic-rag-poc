# Iteration 12 — Auto-configuration, wiring, and prod/dev profiles

> Parent: [implementation-plan.md](../implementation-plan.md) · Previous: [iteration-11-rest-controllers.md](./iteration-11-rest-controllers.md)

## Goal

Wire all beans so the application starts with domain YAMLs loaded and registry populated. Introduce **production** and **development** profiles so prod uses benchmark-grade models and dev uses free/low-cost models. Delivered config must support both env setups as described in [implementation-plan.md § 16](../implementation-plan.md#16-environment-setup-dev-vs-prod); see also [model-recommendations.md](../model-recommendations.md).

## Deliverables

| # | Component | Location |
|---|-----------|----------|
| 1 | `DomainAutoConfiguration` | § Code 10.1 below |
| 2 | **Base** `application.yml` | Shared: domains path, logging, server |
| 3 | **Production** `application-prod.yml` | OpenRouter models (openai/gpt-4o-mini, etc.), OPENROUTER_API_KEY, embedding via OpenRouter |
| 4 | **Development** `application-dev.yml` | in-process embedding, OpenRouter free tier |
| 5 | Profile-aware embedding bean | When `app.models.embedding` is `"in-process"`, register ONNX `EmbeddingModel`; else API-based |
| 6 | At least one minimal domain YAML | e.g. `domains/minimal.yml` |
| 7 | Optional: `application-test.yml` | Test profile: mocks / in-memory |

## Acceptance criteria

- On startup, loader reads from `app.domains.definitions-path`; registry contains all enabled domains.
- **Profile `prod` (or default):** API embedding and chat via OpenRouter; PGVector 1536; single `OPENROUTER_API_KEY`.
- **Profile `dev`:** In-process ONNX embedding (384); OpenRouter free chat; separate DB or schema from prod.
- Ingest and query endpoints work end-to-end with a test domain in both profiles.
- Health or info endpoint (optional) reports loaded domains and active profile.

## Tests to add

- `DomainAutoConfigurationTest`: `@SpringBootTest` with test profile; assert bean `DomainRegistry` exists and has at least one domain from test YAML.
- **Profile prod:** assert `EmbeddingModel` is not in-process; model definitions for gpt-4o-mini / gpt-4o present.
- **Profile dev:** assert in-process `EmbeddingModel` when `app.models.embedding=in-process`; definitions for extraction-free / query-free loaded.
- Optional: one end-to-end test per profile (stub LLM): start app, POST ingest one file, POST query, assert non-empty answer or result list.

## Quality gates

- `./gradlew bootRun` with `--spring.profiles.active=prod` and with `dev` starts without error.
- `./gradlew test` passes; profile-specific tests do not require real API keys.

See [iteration-00-quality-gates.md](./iteration-00-quality-gates.md) for gates that apply to every iteration.

---

## Code — DomainAutoConfiguration (§ 10.1)

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

Production and development YAML snippets: [framework-code.md § 2.3](../framework-code.md#23-applicationyml--model-configuration), [model-recommendations.md](../model-recommendations.md).
