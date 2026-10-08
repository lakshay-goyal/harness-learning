---
type: implementation
harness: pi
concept: tool-call-id-normalization
commit: b30a6dd77
files: [packages/ai/src/api/transform-messages.ts:59, packages/ai/src/api/transform-messages.ts:136, packages/ai/src/api/anthropic-messages.ts:1289, packages/ai/src/api/bedrock-converse-stream.ts:926, packages/ai/src/api/openai-completions.ts:1202, packages/ai/src/api/openai-responses-shared.ts:154, packages/ai/src/api/google-shared.ts:165, packages/ai/src/api/google-generative-ai.ts:195, packages/ai/src/api/mistral-conversations.ts:237]
---
[[tool-call-id-normalization]] in [[pi]].

## Mechanism
- **Split ownership**: the target adapter supplies `normalizeToolCallId(id, model, source)`; the shared pass applies it only to assistant messages where `!isSameModel` and keeps a `toolCallIdMap` so later `toolResult.toolCallId`s follow (packages/ai/src/api/transform-messages.ts:64-68, 83-90, 136-142; 2c7c23b86 2026-01-19 #821). Rationale comment: OpenAI Responses ids "450+ chars with special characters like `|`", Anthropic requires `^[a-zA-Z0-9_-]+$` max 64 (:59-63).

| target API | rule | evidence |
|---|---|---|
| anthropic-messages | `id.replace(/[^a-zA-Z0-9_-]/g,"_").slice(0,64)` | anthropic-messages.ts:1289-1292; origin 6679a83b3 (2025-09-04, Responses ids contain `\|`) |
| bedrock-converse-stream | same regex, ≤64 | bedrock-converse-stream.ts:926-929, 975-979; b74535dc8 (#707), ba8059a50 (#781) |
| google / vertex | same regex ≤64, **only when `requiresToolCallId(model.id)`**: `claude-*`, `gpt-oss-*`, Gemini major ≥3 (`/^gemini(?:-live)?-(\d+)/`) | google-shared.ts:165-178, 195-198; cbaca6038 (#7494) |
| openai-completions | pipe ids `call_id\|item_id` → sanitize `[^a-zA-Z0-9_-]`→`_`, join `callId_itemId`; >40 → `callIdPrefix_<8-char shortHash>`; plain ids truncated to 40 only for provider `openai` (40 = OpenAI limit) | openai-completions.ts:1202-1226; 605f6f494 (#1022), d9f7f8147 (#6854) |
| openai-responses (+azure/codex) | `normalizeIdPart` = sanitize, cap 64, strip trailing `_` (Codex rejects); allowed providers (openai/openai-codex/opencode, +azure `azure-openai-responses.ts:17`) keep pipe form; foreign item ids → `fc_<shortHash(itemId)>`; non-`fc_` → prefix `fc_`; item id dropped for different model / prefix mismatch (`fc_` vs `ctc_`) | openai-responses-shared.ts:154-177, 290-327; b2548ce48 (#2328), b21b42d03, d327b9c76, bc2d8dc1c |
| mistral-conversations | **exactly 9 alnum chars** (`MISTRAL_TOOL_CALL_ID_LENGTH`); strip non-alnum; keep if already 9; else `shortHash(seed)` alnum slice 9; collision-safe bidirectional maps + `:attempt` reseed; per-request map through transformMessages | mistral-conversations.ts:28, 141-144, 237-257; eb9f1183a |

- **Id synthesis on ingest**: Google — if provider id missing OR duplicate within message → `${name}_${Date.now()}_${++toolCallCounter}` (module-level counter) (google-generative-ai.ts:56-57, 195-201; vertex :65-66, 204-209; 6c3580828 2025-09-04). Mistral — missing/`"null"` streamed id → `deriveMistralToolCallId("toolcall:<index>")` (mistral-conversations.ts:702-705). Responses — tool id stored as `${call_id}|${item.id}` (openai-responses-shared.ts:489, 510). Responses message ids `msg_pi_<n>[_<k>]`, >64 → `msg_<shortHash>` (:53-77; 3f1ce9b6e, d1fb34bc8 #5148).
- Responses-family deterministic synthetic tool-search ids `pi_tool_load_<shortHash(seed:names)>` (openai-responses-shared.ts:196-211).
- Non-tool identifiers clamped too: `prompt_cache_key` 64 code points (packages/ai/src/api/openai-prompt-cache.ts:1-8; 7be75bade), Codex session header 64 (dcfe36c79 #6630) → [[server-limited-identifier-rejected]].

## Constants
| name | value | path:line |
|---|---|---|
| Anthropic/Bedrock/Google id max | 64, `[a-zA-Z0-9_-]` | packages/ai/src/api/anthropic-messages.ts:1289-1292 |
| Chat Completions id max | 40 | packages/ai/src/api/openai-completions.ts:1211-1224 |
| Responses id part max | 64, trailing `_` stripped | packages/ai/src/api/openai-responses-shared.ts:154-158 |
| MISTRAL_TOOL_CALL_ID_LENGTH | 9 | packages/ai/src/api/mistral-conversations.ts:28 |
| shortHash suffix (completions) | 8 chars | packages/ai/src/api/openai-completions.ts:1202-1226 |

## Evolution
- 2025-09-04 `6679a83b3` sanitize ids for Anthropic; `6c3580828` unique Google ids.
- 2025-12-15 `c5543f758` Copilot gpt-5↔claude normalization (#198).
- 2026-01-12 `4f757fbe2` Google id fields removed (Vertex unsupported) — later partly reversed.
- 2026-01-13 `3c60ffa67` normalize for switches to Anthropic/Copilot (40 chars); 2026-01-14 `b74535dc8`, 2026-01-16 `ba8059a50` Bedrock (#781).
- 2026-01-19 `2c7c23b86` adapter callback + id map, only `!isSameModel` (#821).
- 2026-01-29 `605f6f494` pipe ids (#1022).
- 2026-03-03 `eb9f1183a` Mistral 9-char normalizer.
- 2026-03-18 `b2548ce48` replayed Responses ids capped/stripped (#2328); 2026-03-22 `b21b42d03` hash foreign item ids.
- 2026-05-28 `3f1ce9b6e`/`d1fb34bc8` unique valid synthetic message ids (#5148).
- 2026-07-20 `d9f7f8147` injective `callId_itemId` + hash (#6854).
- 2026-08-03 `cbaca6038` keep Gemini 3 ids (#7494).
- 2026-10-01 `bc2d8dc1c` drop `fc_` ids when replaying grammar (`ctc_`) calls.

## Evidence commits
6679a83b3 · 6c3580828 · c5543f758 · 4f757fbe2 · 3c60ffa67 · 934e7e470 · b74535dc8 · ba8059a50 · 2c7c23b86 · 605f6f494 · eb9f1183a · b2548ce48 · b21b42d03 · 3f1ce9b6e · d1fb34bc8 · d9f7f8147 · cbaca6038 · d327b9c76 · bc2d8dc1c

## Quirks
- Same-model replay never normalizes ids — relies on the provider accepting its own ids verbatim.
- Google synthetic ids use `Date.now()` + module counter: non-deterministic across replays (replay uses stored id, so harmless) (unverified impact).
- Mistral replays all tool calls with `index: 0` (mistral-conversations.ts:850).

## Failures
- [[cross-provider-tool-call-id-normalization]] · [[tool-call-id-collision]] · [[tool-call-id-requirement-drift]] · [[responses-reasoning-item-pairing]] · [[streamed-tool-call-fragmentation]] · [[server-limited-identifier-rejected]]
