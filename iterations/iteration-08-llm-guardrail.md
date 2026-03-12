# Iteration 8 — LLM guardrail rule

> Parent: [implementation-plan.md](../implementation-plan.md) · Previous: [iteration-07-config-driven-metadata-llm.md](./iteration-07-config-driven-metadata-llm.md)

## Goal

Add `LlmGuardrailRule` that uses a `ChatModel` to decide block/pass. Wire into factory and evaluator.

## Deliverables

| # | Component | Location |
|---|-----------|----------|
| 1 | `LlmGuardrailRule` | § Code 5.4 below |
| 2 | `GuardrailRuleFactory` (include `llm-block`) | [framework-code.md § 5.1](../framework-code.md#51-guardrailrulefactory) — add `case "llm-block"` and inject `ModelRegistry` |

## Acceptance criteria

- Rule takes prompt template and optional model; calls `ChatModel`; parses response to blocked/passed and message.
- Factory creates `llm-block` rule from YAML (prompt, optional model id).

## Tests to add

- `LlmGuardrailRuleTest`: mock `ChatModel` returns "BLOCK: reason" → decision blocked with message; returns "ALLOW" or similar → passed. Test timeout/error handling (e.g. treat as block or pass by policy).

## Quality gates

- No real LLM in tests; mock only.

See [iteration-00-quality-gates.md](./iteration-00-quality-gates.md) for gates that apply to every iteration.

---

## Code — LlmGuardrailRule

Delegates the safety decision to an LLM. The LLM receives a system prompt that defines the safety policy and the user's query. It returns `BLOCK` or `PASS`. Useful for nuanced checks: subtle bias, context-dependent inappropriateness, obfuscated prompt injection.

### Code 5.4 LlmGuardrailRule

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

Update `GuardrailRuleFactory` (see [iteration-04](./iteration-04-guardrail-rules.md)): inject `ModelRegistry`, and add:

```java
case "llm-block" -> new LlmGuardrailRule(
        (String) ruleConfig.get("prompt"),
        modelRegistry.resolve((String) ruleConfig.get("model")),
        blockedMessage);
```
