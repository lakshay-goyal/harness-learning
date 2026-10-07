---
type: implementation
harness: pi
concept: usage-cost-accounting
commit: b30a6dd77
files: [packages/ai/src/types.ts:433, packages/ai/src/types.ts:1077, packages/ai/src/models.ts:1200, packages/ai/src/api/anthropic-messages.ts:669, packages/ai/src/api/anthropic-messages.ts:826, packages/ai/src/api/bedrock-converse-stream.ts:705, packages/ai/src/api/openai-completions.ts:1519, packages/ai/src/api/openai-responses.ts:391, packages/ai/src/api/openai-responses-shared.ts:561, packages/ai/src/api/google-generative-ai.ts:232, packages/ai/src/api/mistral-conversations.ts:541, packages/coding-agent/src/core/usage-totals.ts:30]
---
[[usage-cost-accounting]] in [[pi]].

## Mechanism
- **Normalized shape** `Usage{input, output, cacheRead, cacheWrite, cacheWrite1h?, reasoning?, totalTokens, cost{input,output,cacheRead,cacheWrite,total}}` (packages/ai/src/types.ts:433-454); contract: disjoint partition `input / cacheRead / cacheWrite`; `reasoning` ⊂ `output` (:440-445; d7868b099); `cacheWrite1h` ⊂ `cacheWrite`, Anthropic-family only (:438).
- **`calculateCost(model, usage)`** (packages/ai/src/models.ts:1200-1220), mutates `usage.cost` in place: tier input = `input + cacheRead + cacheWrite`; highest matching `tiers[].inputTokensAbove` applies to the **whole request** (:1201-1209; types.ts:1084-1092; a9ecf301f); **1h cache writes at 2× base input**, short writes at `cacheWrite` rate (:1211-1217; 0be5bb6c9 #5738). Rates $/M tokens (types.ts:1077-1082).
- Generator pricing: OpenAI long-context tier above `272000` input → input×2, output×1.5, cacheRead×2, cacheWrite×2 (packages/ai/scripts/generate-models.ts:395, 431-445); models.dev `cost.tiers[].tier.type==="context"` mapped (:1265-1287; Bedrock tiers kept 4665fafb4 #10326); authoritative `OPENAI_STANDARD_COSTS` (:449).
- **Service tier** (`applyServiceTierPricing`, packages/ai/src/api/openai-responses.ts:391-419): multiplier on all components — flex 0.5; priority/fast 2 (gpt-5.5 → 2.5); `fast` for GPT-6 Fast mode (a6ca86102 #10034). Codex copy (openai-codex-responses.ts:602-614) has **no `fast` case** (drift); Codex trusts requested tier when response says `default` (:631-639; 2cdac7382 #3307).
- **Fallback pricing**: Anthropic `message_start.model !== model.id` → `responseModel` set, cost from matching `allowedFallbackModels[].cost` (anthropic-messages.ts:669-677; a6c6f8018 → revert 59a71b235 → 4809c2abc → ed867e909) → [[server-side-refusal-fallback]].

**Per-adapter usage normalization**

| adapter | input | cacheRead | cacheWrite | reasoning | total / timing |
|---|---|---|---|---|---|
| Anthropic | `input_tokens` at `message_start` (abort-safe) | `cache_read_input_tokens` | `cache_creation_input_tokens`; `cacheWrite1h = cache_creation.ephemeral_1h_input_tokens`, also read from deltas (Vercel AI Gateway; 667fc3dd3 #9210) | `output_tokens_details.thinking_tokens` | computed sum (no API total); `message_delta` fields overwritten only if non-null (cumulative `=` not `+=`), block skipped if `usage` absent (anthropic-messages.ts:680-688, 826-858; bc8d994a7, 8b5c81f21 Portkey, 0e6909f05 #6611) |
| Bedrock | `metadata.usage.inputTokens` | cacheRead | cacheWrite; `cacheWrite1h = Σ cacheDetails[ttl==ONE_HOUR]` (8a7b0c03d #9457) | — | `usage.totalTokens \|\| input+output` (bedrock-converse-stream.ts:705-722) |
| OpenAI Completions | `max(0, prompt − read − write)` | `prompt_tokens_details.cached_tokens ?? prompt_cache_hit_tokens (DeepSeek) ?? cached_tokens (Kimi)` (fc3cbedc6, d3ab2af96) | `prompt_tokens_details.cache_write_tokens` (OpenRouter) | `completion_tokens_details.reasoning_tokens` (output already includes it; c681d35d7 #3581) | recomputed; `chunk.usage` fallback `choice.usage` (Moonshot, 453541530) (openai-completions.ts:568-579, 1519-1560) |
| OpenAI Responses | `input_tokens − cached − cache_write` | cached | cache_write | reasoning details | (openai-responses-shared.ts:561-576) |
| Google | `promptTokenCount − cachedContentTokenCount` (6d744f02e #2588) | `cachedContentTokenCount` | 0 always (implicit caching only) | `thoughtsTokenCount` (output = candidates + thoughts) | last chunk wins (google-generative-ai.ts:232-251) |
| Mistral | `prompt − cached` | any of `promptTokensDetails.cachedTokens \| prompt_tokens_details.cached_tokens \| … \| numCachedTokens \| num_cached_tokens`, clamped [0,prompt] | 0 | — | (mistral-conversations.ts:541-560, 602-614; 651d10d90) |
| OpenRouter images | `prompt − read − write` | `cached − cacheWrite` when `cache_write_tokens` reported | write | — | (openrouter-images.ts:167-198) |
| pi-messages | server-supplied, no local `calculateCost` | | | | (pi-messages.ts:180-274) |
| classifiers | `parseClassifierUsage({input_tokens, output_tokens})` priced via calculateCost (classifier-shared.ts:114-128; 89a5c7bda); usage set **before** parsing answers (system-one-shared.ts:125-127); llama.cpp reports no usage (packages/ai/README.md:938) | | | | |

- OpenRouter semantics flip-flop: `6044cabb1` (#2802) subtracted cache_write from cached_tokens → `87881ca68` follow documented semantics (cached = reads only) → [[usage-double-counting]].
- **Coding-agent aggregation**: by `provider/responseModel ?? model`; tool/summarization usage bucketed `Tools/summaries` (packages/coding-agent/src/core/usage-totals.ts:61-97); `combineUsage` keeps `cacheWrite1h`/`reasoning` splits (:30-52). Cache-warm usage persisted as `usage` entry kind `cache_warm` (cache-warmer.ts:342-349) → [[cache-warming]]; cache-miss cost = missed × (paid rate − cache-read rate) (cache-stats.ts:70-86) → [[cache-miss-accounting]].
- Bedrock `requestMetadata` passthrough for AWS cost-allocation tags (bedrock-converse-stream.ts:100-104, 280; 3bcbae490 #2511).
- Codemode `models.*` call usage added to session cost (9a100c7cc) → [[code-mode]].
- `ToolResultMessage.usage` = tool's own usage, not context accounting (packages/ai/src/types.ts:607-624; `NestedToolCallRecord` :585-605).

## Constants
| name | value | path:line |
|---|---|---|
| 1h cache write multiplier | 2× base input | packages/ai/src/models.ts:1211-1217 |
| OPENAI_LONG_CONTEXT_INPUT_THRESHOLD | 272000 (in×2, out×1.5) | packages/ai/scripts/generate-models.ts:395 |
| service tier flex / priority / fast | 0.5 / 2 / 2 (gpt-5.5 priority 2.5) | packages/ai/src/api/openai-responses.ts:391-404 |
| rate unit | $ per million tokens | packages/ai/src/types.ts:1077-1082 |

## Evolution
- 2025-10-26 `bc8d994a7` Anthropic usage from `message_start`, assign not add.
- 2026-01-29 `8b5c81f21` Portkey nullable delta usage.
- 2026-03-10 `453541530` Moonshot `choice.usage`.
- 2026-03-26 `6d744f02e` Google cached subtraction (#2588).
- 2026-04-04 `6044cabb1` → 2026-05-16 `87881ca68` OpenRouter cached semantics.
- 2026-04-17 `2cdac7382` Codex requested tier.
- 2026-04-23 `c681d35d7` reasoning not double-counted (#3581); 2026-04-29 `fc3cbedc6` DeepSeek cache hits.
- 2026-06-15 `0be5bb6c9` 1h writes 2× (#5738); 2026-06-18 `651d10d90` Mistral caching prices; 2026-06-25 `d7868b099` `Usage.reasoning`.
- 2026-07-09 `a9ecf301f` input-based tiers; 2026-07-13 `0e6909f05` empty usage skip.
- 2026-08-16 `d3ab2af96` Kimi cached (#8119); 2026-08-18/19 fallback cost arc `a6c6f8018`/`59a71b235`/`4809c2abc`/`ed867e909`.
- 2026-09-15 `8a7b0c03d` Bedrock 1h; 2026-09-23 `667fc3dd3` Vercel 1h; 2026-09-25 `a6ca86102` fast tier; 2026-09-29 `89a5c7bda` classifier cost; 2026-10-02 `4665fafb4` Bedrock tiers.

## Evidence commits
bc8d994a7 · 8b5c81f21 · 0e6909f05 · 453541530 · 6d744f02e · 6044cabb1 · 87881ca68 · 2cdac7382 · c681d35d7 · fc3cbedc6 · 0be5bb6c9 · 651d10d90 · d7868b099 · a9ecf301f · d3ab2af96 · a6c6f8018 · 59a71b235 · 4809c2abc · ed867e909 · 8a7b0c03d · 667fc3dd3 · a6ca86102 · 89a5c7bda · 4665fafb4

## Quirks
- Bedrock `totalTokens` semantics (provider total) differ from Anthropic's computed sum; whether Bedrock `inputTokens` includes cache reads is unverified — matters for tier selection.
- Codex service-tier table lacks `fast` (openai-codex-responses.ts:602-614) — duplicated tables drift.
- Google adapters never report cacheWrite and ignore `cacheRetention`/`sessionId` (no explicit context caching) — deliberate absence unverified.

## Failures
- [[usage-double-counting]] · [[streamed-usage-misread]] · [[usage-priced-at-wrong-rate]] · [[billed-call-lost-on-parse-error]] · [[fallback-model-output-misattributed]] · [[model-relabel-breaks-same-model-check]]
