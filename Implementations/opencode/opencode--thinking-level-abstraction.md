---
type: implementation
harness: opencode
concept: thinking-level-abstraction
commit: ecc4916b5a
files: [packages/opencode/src/provider/transform.ts:576-660, packages/opencode/src/provider/transform.ts:741-860, packages/opencode/src/provider/transform.ts:1220-1414, packages/opencode/src/provider/transform.ts:1717-1890, packages/opencode/src/session/llm/request.ts:80-91]
---
[[thinking-level-abstraction]] in [[opencode]].

## Mechanism (legacy runtime)
- **Variants**, not levels: `ProviderTransform.variants(model)` generates a per-model map of named option bundles (`low|medium|high|xhigh|none|minimal|max`), user-selected via a toggle; no default effort is applied by the harness (`packages/opencode/src/provider/transform.ts:790-860`).
- Effort lists per family with **release-date gates**: OpenAI `none` only for models released ≥ `2025-11-13`, `xhigh` ≥ `2025-12-04` (`transform.ts:576-649`); Anthropic adaptive efforts vs budgets (`transform.ts:657-692`); Gemini `thinkingLevel` vs 2.5 budgets (`transform.ts:741-789`).
- Budget variants: `high = floor((max+1)/2)`, `max = min(limit.output−1, 31999)` (`budgetVariants`, `transform.ts:1749-1762`).
- Family defaults in `options()`: gpt-5 → `reasoningEffort: "medium"` + summaries + `textVerbosity: "low"` for gpt-5.x non-codex; Gemini → `includeThoughts`, `thinkingLevel: "high"`; Kimi/MiniMax via Anthropic-compatible → adaptive; DashScope `enable_thinking`; z.ai `clear_thinking: false` (`transform.ts:1220-1388`).
- Merge order: base options → model options → agent options → selected variant (`packages/opencode/src/session/llm/request.ts:80-91`). Side calls (`small: true`) skip the user's variant and use `smallOptions` = lowest effort or thinking off (`transform.ts:1390-1414`).
- `reasoningVariants` derives variants from catalog reasoning metadata (`transform.ts:1717-1747`).

## Constants
| name | value | path:line |
|---|---|---|
| Anthropic/Bedrock budgets | `high` 16 000, `max` 31 999 | `packages/opencode/src/provider/transform.ts:916,922` |
| Gemini 2.5 max budget | 32 768 (pro) / 24 576 (flash) | `packages/opencode/src/provider/transform.ts:760-764` |
| `OPENAI_NONE_EFFORT_RELEASE_DATE` | `2025-11-13` | `packages/opencode/src/provider/transform.ts:589` |
| `OPENAI_XHIGH_EFFORT_RELEASE_DATE` | `2025-12-04` | `packages/opencode/src/provider/transform.ts:592` |

## Evolution
- 2025-09-07 `4654fb88de`, 2025-10-04 `71a7e8ef36` budget ate `max_tokens`; 2025-11-13 `0c51feb9c2` check keyed on npm `@ai-sdk/anthropic` not providerID.
- 2025-12-29 `ed0c0d90be` variants toggle.
- 2026-01-04 `554572bc39` main-model variant leaked into the small (title) model; 2026-01-07 `2e4fe973c9` options conflict during title gen.
- 2026-01-30 `0c32afbc35` snake_case `thinking.budget_tokens` for OpenAI-compatible; 2026-03-22 `cc818f8032` `thinkingConfig` only for reasoning models; 2026-04-13 `c2403d0f15` / 2026-08-21 `3a4c253969` guard `reasoningSummary`/`textVerbosity` for openai-compatible.
- 2026-05..07 long tail: `1cf8123bc6`, `e0396b809a`, `c36ab3f935`, `6f8e1dda15`, `a8062ea314`, `49d2dd8a38` "align efforts/variants".
- 2026-08-12 `6fea419feb` Groq SDK effort enum → string (patch).

## Quirks / drift
- Only two budget levels (`high`, `max`); 31 999 = `OUTPUT_TOKEN_MAX − 1` so the budget always fits under the 32 k cap.
- Effort tables are hand-maintained per model family — the drift source of [[thinking-config-per-model-drift]].

Failures: [[thinking-consumes-answer-budget]] · [[thinking-config-per-model-drift]] · [[endpoint-rejects-request-field]] · [[sdk-enum-lags-provider-options]].

Contrast: [[pi--thinking-level-abstraction|pi]] exposes a neutral `ThinkingLevel` scale clamped per model with an answer reserve; opencode exposes named per-model variants with no default.
