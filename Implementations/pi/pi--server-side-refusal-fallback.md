---
type: implementation
harness: pi
concept: server-side-refusal-fallback
commit: b30a6dd77
files: [packages/ai/src/api/anthropic-messages.ts:211, packages/ai/src/api/anthropic-messages.ts:669, packages/ai/src/api/anthropic-messages.ts:690, packages/ai/src/api/anthropic-messages.ts:1281, packages/ai/src/api/anthropic-messages.ts:1613, packages/ai/scripts/generate-models.ts:277, packages/coding-agent/src/core/model-config.ts:181]
---
[[server-side-refusal-fallback]] in [[pi]].

## Mechanism
- Metadata: `AnthropicMessagesCompat.allowedFallbackModels` — server-side refusal fallback targets **with local pricing** (`packages/ai/src/types.ts:975`). Generated: `claude-fable-5 → [opus-4-8, opus-5]`, `claude-opus-5 → [opus-4-8]` (`packages/ai/scripts/generate-models.ts:277-280, 843-862`); filtered to mid-convo-effort-capable targets when the source has `compat.supportsMidConvoEffort` (`:849-851`). User-configurable in models.json compat, maxItems 3 (`packages/coding-agent/src/core/model-config.ts:181-190`; `b03a367a4`).
- Request: beta `server-side-fallback-2026-07-01` iff list non-empty (`anthropic-messages.ts:211-213, 1116`); body `fallbacks: allowedFallbackModels.map(f => ({model: f.model}))` (`:1281-1284`).
- Stream: `message_start.model !== model.id` → set `responseModel`, take cost from matching `allowedFallbackModels[].cost` (`:669-677`); `message.model` stays the **requested** id so replay `isSameModel` keeps working (`1283afd0d`; see [[cross-provider-handoff]]).
- `content_block_start` of type `fallback`: no output yet → `continue` (clean switch); after output → throw `Anthropic performed an unsupported mid-output model fallback` (`:690-695`; `4e69b0c28`) — refuse to splice two models' output.
- Refusal stop: `refusal` → `stopReason:"error"`, message = `stop_details.explanation` or "The model refused to complete the request" (`:1613-1639`; `eb1f87fa9` #8258). `sensitive` → error "Provider stopped with: sensitive" (`ee7c0a7d1` #978).
- Usage aggregation buckets by `provider/responseModel ?? model` (`packages/coding-agent/src/core/usage-totals.ts:61-97`).
- Only client-side failover alternative in pi is a [[virtual-model-router]] (no built-in cross-provider failover; 08-absences "Model routing / fallback").

## Constants
| name | value | path:line |
|---|---|---|
| fallback beta | server-side-fallback-2026-07-01 | packages/ai/src/api/anthropic-messages.ts:190-195 |
| max configured fallbacks | 3 | packages/coding-agent/src/core/model-config.ts:181-190 |
| generated fallbacks | fable-5 → [opus-4-8, opus-5]; opus-5 → [opus-4-8] | packages/ai/scripts/generate-models.ts:277-280 |

## Evolution
- 2026-08-17 `eb1f87fa9` refusal error + fallbacks (#8258).
- 2026-08-18 `a6c6f8018` fallback usage (#8308) → reverted `59a71b235` (#8313) → 2026-08-19 `4809c2abc` (#8319) → `ed867e909` "fallback cost not via stream options; always pass beta header" (#8352).
- 2026-09-02 `4e69b0c28` mid-output fallback throws.
- 2026-09-16 `b03a367a4` user-configurable Anthropic fallback models.
- 2026-09-17 `1283afd0d` keep requested model id, `responseModel` separate (#9188).

## Evidence commits
eb1f87fa9, a6c6f8018, 59a71b235, 4809c2abc, ed867e909, 4e69b0c28, b03a367a4, 1283afd0d, ee7c0a7d1

## Quirks
- Fallback cost lives in compat metadata of the *requesting* model, so a custom fallback target lacking pricing in that list is under-costed (inferred, unverified).

## Failures
- [[fallback-model-output-misattributed]]
- [[model-relabel-breaks-same-model-check]]
- [[stop-reason-mapping-gaps]]
