---
type: implementation
harness: pi
concept: max-tokens-context-clamp
commit: b30a6dd77
files: [packages/ai/src/api/simple-options.ts:15, packages/ai/src/api/simple-options.ts:46, packages/ai/src/api/simple-options.ts:92, packages/ai/src/utils/estimate.ts:15, packages/ai/src/api/anthropic-messages.ts:968, packages/ai/src/api/bedrock-converse-stream.ts:263, packages/ai/src/api/openai-responses.ts:32]
---
[[max-tokens-context-clamp]] in [[pi]].

## Mechanism
- `clampMaxTokensToContext(model, context, maxTokens)` = `min(maxTokens, contextWindow − estimatedContextTokens − CONTEXT_SAFETY_TOKENS)`, floor `MIN_MAX_TOKENS = 1` (packages/ai/src/api/simple-options.ts:15-22) — prevents "max_tokens + input > context" 400s on servers that count both against one window / reserve `max_tokens` from context (vLLM).
- Applied in `buildBaseOptions` for every `streamSimple`, on `options.maxTokens ?? model.maxTokens` — i.e. a default cap IS sent again (:46); Anthropic after thinking adjustment (packages/ai/src/api/anthropic-messages.ts:961-975, :968); Bedrock Claude non-adaptive stores clamped budget under `thinkingBudgets[clampReasoning(level)]` (packages/ai/src/api/bedrock-converse-stream.ts:531-580, :562).
- **Estimator** `estimateContextTokens` (packages/ai/src/utils/estimate.ts): last valid assistant usage (`totalTokens || input+output+cacheRead+cacheWrite`, :18-20) + chars/`CHARS_PER_TOKEN=3.5` of trailing messages, image = 4800 chars (~1371 tok) (:15-16, 97-112); skips aborted/error usage and usage older than a later-inserted prefix message (compaction summary) via timestamp (:71-95; 8973ae28a #6464) → [[token-estimation]].
- **Interaction with thinking**: `adjustMaxTokensForThinking` = model cap if no caller cap else `min(base+budget, modelMax)`, shrinking budget if `maxTokens <= budget` (simple-options.ts:92-108); Anthropic budget then `min(adjusted, max(0, maxTokens − 1024))` (anthropic-messages.ts:961-975); `MIN_ANSWER_TOKENS=1024` reserve (simple-options.ts:67-68) → [[pi--thinking-level-abstraction|thinking-level-abstraction]].
- **Provider floors/requirements**:
  - Anthropic: `max_tokens: options.maxTokens ?? model.maxTokens` always sent (required) (anthropic-messages.ts:1162; 2787b601d kept "required Anthropic max_tokens handling").
  - Bedrock: `inferenceConfig.maxTokens = options.maxTokens ?? (isClaude ? model.maxTokens : undefined)` (bedrock-converse-stream.ts:263) — Bedrock's 4096 default truncated Claude (11c3da4f7 #4848) after `a5fac1ef0` omitted it to save TPM quota reservation (Claude 3.7+ 5× output burndown).
  - OpenAI Responses: `max_output_tokens = max(maxTokens, 16)` (OpenAI rejects <16) (openai-responses.ts:32-33, 340-342; 2e4ad6a09 #6265), gated by `supportsMaxOutputTokens` (b8b873b98 #8941); omitted for Sign in with ChatGPT tokens (:328-346); Azure same floor (azure-openai-responses.ts:18-19).
  - Codex: no `max_output_tokens` field at all — Codex ignores maxTokens (openai-codex-responses.ts:89-106).
  - Completions: `maxTokensField` `max_tokens` for chutes/deepseek/moonshot/CF gateway/together/nvidia/ant-ling/zai else `max_completion_tokens` (openai-completions.ts:1629-1637, 1651, 842-849).
- Virtual-model direct calls cap `maxTokens` to the routed model (packages/coding-agent/src/core/model-runtime.ts:716-736) → [[virtual-model-router]].
- Compaction summary `maxTokens = min(0.8·reserveTokens, model.maxTokens)` (durable packages/durable/src/harness/compaction.ts:139-142).
- Cache warmer sends `maxTokens: 1` (packages/coding-agent/src/core/cache-warmer.ts:157-161) → [[cache-warming]].

## Constants
| name | value | path:line |
|---|---|---|
| CONTEXT_SAFETY_TOKENS | 4096 | packages/ai/src/api/simple-options.ts:15 |
| MIN_MAX_TOKENS | 1 | packages/ai/src/api/simple-options.ts:16 |
| MIN_ANSWER_TOKENS | 1024 | packages/ai/src/api/simple-options.ts:68 |
| CHARS_PER_TOKEN | 3.5 (was 4) | packages/ai/src/utils/estimate.ts:15 |
| ESTIMATED_IMAGE_CHARS | 4800 | packages/ai/src/utils/estimate.ts:16 |
| OPENAI_RESPONSES_MIN_OUTPUT_TOKENS | 16 | packages/ai/src/api/openai-responses.ts:33 |
| custom-model default maxTokens | 16384 | packages/coding-agent/src/core/provider-composer.ts:243 |
| generator fallback context/max | 4096 | packages/ai/scripts/generate-models.ts:1494-1495 |

## Evolution (the max_tokens saga)
- 2026-04-19 `a5fac1ef0` Bedrock omits maxTokens (TPM reservation).
- 2026-05-16 `22a9c484e` `streamSimple` had clamped every model to 32000 → respect model max (#4539).
- 2026-05-17 `6d474f8c1` models whose maxTokens ≈ window → cap 32000 when within 1024 of context (#4614).
- 2026-05-19 `2787b601d` stop sending default caps at all (vLLM reserves max_tokens) (#4675) — reversed for `streamSimple` by `09f105957`, which defaults to `model.maxTokens` and clamps to context.
- 2026-05-21 `11c3da4f7` Bedrock Claude default = model cap (#4848).
- 2026-06-25 `09f105957` context-aware clamp `min(max, ctx − est − 4096)` (#5595/#6061).
- 2026-07-06 `2e4ad6a09` Responses floor 16 (#6265).
- 2026-07-09 `8973ae28a` ignore stale pre-compaction usage (#6464).
- 2026-09-01 `b8b873b98` `supportsMaxOutputTokens` compat (#8941).
- 2026-10-06 `27075fe07` 4 → 3.5 chars/token (DeepSeek V4 Flash overflow, #10497).

## Evidence commits
a5fac1ef0 · 22a9c484e · 6d474f8c1 · 2787b601d · 11c3da4f7 · 09f105957 · 2e4ad6a09 · 8973ae28a · b8b873b98 · 27075fe07

## Quirks
- Clamp accuracy depends on the char heuristic; `ESTIMATED_IMAGE_CHARS` duplicated in compaction estimator (chars/4) vs request estimator (3.5) (packages/coding-agent/src/core/compaction/compaction.ts:276).
- Fixed headroom (4096) rather than ratio — consistent with pi's "fixed headroom" style for compaction reserve 16384.

## Failures
- [[output-token-cap-misbudgeted]] · [[thinking-consumes-answer-budget]] · [[endpoint-rejects-request-field]]
