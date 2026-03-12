# Iteration 4 — Guardrail rules (deterministic)

> Parent: [implementation-plan.md](../implementation-plan.md) · Previous: [iteration-03-extraction-strategies.md](./iteration-03-extraction-strategies.md)

## Goal

Implement term-block and pattern-block guardrail rules and the guardrail rule factory. No LLM. These **custom algorithms** handle most guardrail needs (blocklist terms, prompt-injection patterns); llm-block is added in Iteration 8 for nuanced intent. See [technical-design.md § 21](../technical-design.md#21-custom-algorithms-vs-llm).

## Deliverables

| # | Component | Location in this doc |
|---|-----------|----------------------|
| 1 | `GuardrailRuleFactory` | § Code 5.1 (term-block, pattern-block only) |
| 2 | `TermBlockGuardrailRule` | § Code 5.2 |
| 3 | `PatternBlockGuardrailRule` | § Code 5.3 |

## Acceptance criteria

- Term-block: if query contains configured term (and optional intent), rule blocks with message.
- Pattern-block: if query matches configured regex, rule blocks.
- Factory builds rule from YAML-like map (`type`, `terms`, `pattern`, etc.).
- Evaluator (or single-rule evaluation) returns `GuardrailDecision` (blocked / passed).

## Tests to add

- `TermBlockGuardrailRuleTest`: query containing term → blocked; query without term → passed; optional intent matching.
- `PatternBlockGuardrailRuleTest`: query matching pattern → blocked; no match → passed.
- `GuardrailRuleFactoryTest`: create term-block and pattern-block from config; unknown type throws or returns empty.

## Quality gates

- All tests pass; no external calls.

See [iteration-00-quality-gates.md](./iteration-00-quality-gates.md) for gates that apply to every iteration.

---

## Code — Guardrail rules (term-block, pattern-block)

In Iteration 8 you will add `LlmGuardrailRule` and the `llm-block` case in the factory.

### Code 5.1 GuardrailRuleFactory (term-block, pattern-block only)

```java
package com.example.rag.feature.domain.engine.guardrail;

import com.example.rag.feature.domain.GuardrailRule;
import java.util.List;
import java.util.Map;
import org.springframework.stereotype.Component;

@Component
public class GuardrailRuleFactory {

    @SuppressWarnings("unchecked")
    public GuardrailRule create(Map<String, Object> ruleConfig) {
        var type = (String) ruleConfig.get("type");
        var blockedMessage = (String) ruleConfig.get("blocked-message");

        return switch (type != null ? type : "") {
            case "term-block" -> new TermBlockGuardrailRule(
                    (List<String>) ruleConfig.get("trigger-terms"),
                    (List<String>) ruleConfig.getOrDefault("intent-terms", List.of()),
                    (Boolean) ruleConfig.getOrDefault("require-both", false),
                    blockedMessage);

            case "pattern-block" -> new PatternBlockGuardrailRule(
                    (List<String>) ruleConfig.get("patterns"),
                    blockedMessage);

            default -> throw new IllegalArgumentException(
                    "Unknown guardrail rule type: '%s'. In Iteration 8 add 'llm-block'.".formatted(type));
        };
    }
}
```

(When adding Iteration 8, inject `ModelRegistry` and add the `llm-block` case that builds `LlmGuardrailRule`.)

### Code 5.2 TermBlockGuardrailRule

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

### Code 5.3 PatternBlockGuardrailRule

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
