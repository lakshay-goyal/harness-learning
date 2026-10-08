---
type: implementation
harness: pi
concept: signed-reasoning-replay
commit: b30a6dd77
files: [packages/ai/src/types.ts:407, packages/ai/src/api/transform-messages.ts:104, packages/ai/src/api/anthropic-messages.ts:1233, packages/ai/src/api/anthropic-messages.ts:1405, packages/ai/src/api/anthropic-messages.ts:1518, packages/ai/src/api/bedrock-converse-stream.ts:655, packages/ai/src/api/bedrock-converse-stream.ts:792, packages/ai/src/api/openai-responses-shared.ts:264, packages/ai/src/api/openai-responses-shared.ts:534, packages/ai/src/api/openai-completions.ts:1312, packages/ai/src/api/google-shared.ts:141, packages/ai/src/api/google-shared.ts:146]
---
[[signed-reasoning-replay]] in [[pi]].

## Mechanism
- **Storage shape**: `ThinkingContent{thinking, thinkingSignature?, redacted?}` (packages/ai/src/types.ts:407-415); `TextContent.textSignature` (legacy id or `TextSignatureV1{v:1,id,phase}`, :395-405); `ToolCall.thoughtSignature` (Google) (:423-431). The signature field is overloaded per adapter as the opaque replay payload.
- **Shared rule** (transform-messages.ts): replay payload kept only when `isSameModel` (provider+api+model id, :95-98); redacted thinking dropped cross-model (:104-106); **signed thinking kept even with empty text** (:107-109); unsigned empty dropped (:111); cross-model → plain text (:112-116); cross-model `thoughtSignature` deleted (:131-134).

**Per-provider payload table**

| provider/API | what is signed | capture | replay |
|---|---|---|---|
| Anthropic Messages | `thinking.signature`; `redacted_thinking.data` | `signature_delta` appended silently (anthropic-messages.ts:737-782); redacted → block text `"[Reasoning redacted]"`, `redacted:true`, data in `thinkingSignature` (:713-722; 9825c13f5 #1665) | `{type:"thinking", thinking, signature}`; `redacted_thinking{data}`; skip only if text empty AND no signature (:1405-1438; 6731a0ba9 #6457); blank signature → plain text unless compat `allowEmptySignature` → `signature:""` (458a7bc27 #4464; Fireworks, all Vercel AI Gateway models generate-models.ts:1686; c1449660c OpenCode qwen) |
| Anthropic managed effort (Opus 5/5.5, Sonnet 5.5, Fable/Mythos 5.1) | signed thinking **bound to prefix + effort** | output records `providerThinkingLevel = options.effort ?? "high"` (:581, 588) | always `thinking:{type:"adaptive", display, block_binding:{prefix_mismatch_behavior:"drop_block"}}` + `output_config.effort:"high"` "so prefix mismatches can be dropped instead of surfacing as persistent 400 responses" (:1233-1241); `insertThinkingLevelMessages` inserts `{role:"system", content:[], output_config:{effort}}` before each historical assistant that recorded a level + trailing marker (:1518-1532; 4e69b0c28); trailing marker inserted after cache-marker placement so not marked (:1159-1161); server drops reported as `anthropic_input_transformations` diagnostic (:871-883) |
| Bedrock Converse (Claude) | `reasoningText.signature`; `redactedContent` bytes | signature appended unless block redacted ("never both") (bedrock-converse-stream.ts:655-660); redacted bytes buffered in scratch `redactedChunks`, flushed to base64 on block stop and every terminal path, base64 in 0x8000 windows (:661-703, 344-351, 1397-1409; d57e531f5) | Claude `reasoningText{text, signature}`, missing sig → text (c961cda2c); non-Claude no signature field (9a438465e); `block_binding` only Opus 4.7+/Sonnet 5/Fable 5 — 4.6 rejects "Extra inputs are not permitted" (:792-806; 69f0be6f0 #10324); GovCloud omits `display`+block_binding (:1244-1252, 1263-1270; 454b9619c) |
| OpenAI Responses / Azure / Codex | whole `ResponseReasoningItem` incl. `encrypted_content` | `output_item.done` reasoning → signature = JSON of item (openai-responses-shared.ts:688-700); **backfill `encrypted_content` from terminal `response.output`** (Azure omits it per-item; 1f0dbc008 #6409) (:534-551); `include:["reasoning.encrypted_content"]` (openai-responses.ts:364-373; xAI always :379; Codex openai-codex-responses.ts:560) | JSON-parse and push verbatim (:264-268); empty-summary items still replayed (8fc2b7682 → b4e7d5c44 #1878); text `phase` (commentary/final_answer) via `TextSignatureV1` (87d71380e #1819); item ids paired with `rs_` dropped for other models (d327b9c76) |
| OpenAI Completions family | `reasoning_details` (OpenRouter), or raw field name | `reasoning_details` deltas kept in memory, consecutive `reasoning.text`/`reasoning.summary` merged, `reasoning.encrypted` discrete, serialized ONCE into `thinkingSignature` at block end or error (openai-completions.ts:331-338, 670-681, 257-271; c5ad7c1b0 #8605, 7aab6c26e #8671); reasoning field = first non-empty of `reasoning_content`/`reasoning`/`reasoning_text` (:607-638; 36e774282) | structured `reasoning_details` (or legacy encrypted detail on `toolCall.thoughtSignature`) (:1312-1319, 1383-1385); else raw field named by `thinkingSignature` (:1340-1349); DeepSeek `reasoning_content:""` on assistants lacking it when compat `requiresReasoningContentOnAssistantMessages` (:1386-1392; 9b103e5e4); Qwen `preserve_thinking:true` (e3f6912d4 #3325); Z.AI `clear_thinking:false` (b91bdd5a3 #6083); Azure Foundry keeps `reasoning_content` so cached prefix stays byte-identical (a37306d43) |
| Google Gemini / Vertex | `thoughtSignature` on **any** part (TYPE_BYTES) | thinking iff `thought===true`; signature on text → `textSignature` (google-shared.ts:113-130; 4f757fbe2 reverted e42e9e630); `retainThoughtSignature` keeps last non-empty within block (:141-144) | kept only same provider+model+valid base64 (:146-160; 934e7e470 #654); signed empty blocks kept (:236-240, 250-252; 6138f5a07 #7362); unsigned foreign Gemini 3 calls sent unsigned (f7df47408) → [[gemini-unsigned-tool-call-replay]] |
| Mistral | native thinking chunks | — | same-provider `{type:"thinking", thinking:[{type:"text"}]}` (4c175790b); empty deltas ignored or thinking splits into two blocks → "Expected at most one leading ThinkChunk" (8930b9ec0 #9674) |

- **Stateless replay** (`store:false`): OpenAI Responses always `store:false` (openai-responses.ts:337; d1fce2ba1 #1308); Azure (84cdd0240); Codex "rejects store:true; continuation works via connection-scoped previous_response_id" (openai-codex-responses.ts:1519-1520). Encrypted reasoning is the continuity mechanism.
- `display` default `"summarized"` because API default for Opus 4.7 / Mythos Preview is `"omitted"` (anthropic-messages.ts:259-271, 1244-1246; acbf8eca0) — omitted display yields signed empty-text blocks that must be replayed.
- Replay identity uses requested `model`, not `responseModel` (1283afd0d #9188) → [[model-relabel-breaks-same-model-check]].
- Virtual-model router docs: return `previous` on continuation / `failed` on retry "to keep prompt caches and thinking signatures valid" (packages/coding-agent/docs/virtual-models.md:84) → [[virtual-model-router]].
- Cache warmer skips Anthropic budget-thinking replays (`budget_tokens` derives from `max_tokens`, cache keyed on it) (packages/coding-agent/src/core/cache-warmer.ts:48-58) → [[cache-warming]].

## Constants
| name | value | path:line |
|---|---|---|
| redacted placeholder text | `[Reasoning redacted]` | packages/ai/src/api/anthropic-messages.ts:713-722 |
| managed-effort default | `high` | packages/ai/src/api/anthropic-messages.ts:581 |
| thinking-binding beta | `thinking-binding-controls-2026-08-01` | packages/ai/src/api/anthropic-messages.ts:190-195 |
| managed-effort beta | `mid-conversation-output-config-2026-07-01` | packages/ai/src/api/anthropic-messages.ts:1117-1119 |
| Google signature base64 check | `/^[A-Za-z0-9+/]+={0,2}$/`, len%4==0 | packages/ai/src/api/google-shared.ts:146-160 |
| base64 window (Bedrock) | 0x8000 | packages/ai/src/api/bedrock-converse-stream.ts:1397-1409 |

## Evolution
- 2025-11-18 `387cc97ba` aborted unsigned thinking → text; 2025-12-24 `29379ea0a` untagged.
- 2026-01-12 `934e7e470` signatures scoped to same provider+model, base64-validated; `4f757fbe2` thinking = `thought===true` only.
- 2026-01-14 `9a438465e` Bedrock non-Claude no signature.
- 2026-01-15..2026-04-30 Gemini unsigned tool-call saga `b18f401d9` → `5d3e7d5aa` → `a0d839ce8` → `f7df47408`.
- 2026-02-06 `d1fce2ba1` Responses `store:false`.
- 2026-02-27 `9825c13f5` redacted_thinking round-trip (#1665).
- 2026-03-05 `8fc2b7682` omit empty Responses thinking → 2026-03-06 `b4e7d5c44` restored (#1878); `4c175790b` Mistral native thinking; `87d71380e` phase.
- 2026-03-14 `c961cda2c` Bedrock unsigned → text.
- 2026-04-17 `e3f6912d4` Qwen preserve_thinking; 2026-04-24 `9b103e5e4` DeepSeek reasoning_content.
- 2026-05-28 `458a7bc27` `allowEmptySignature`.
- 2026-06-29 `b91bdd5a3` Z.AI clear_thinking false.
- 2026-07-09 `6731a0ba9` signed empty Anthropic blocks; 2026-07-13 `1f0dbc008` Azure backfill; 2026-07-31 `6138f5a07` Gemini signed empty.
- 2026-08-19 `d57e531f5` Bedrock redacted round-trip; 2026-08-25/26 `c5ad7c1b0`, `7aab6c26e` reasoning_details.
- 2026-09-02 `4e69b0c28` per-turn effort markers + block_binding; 2026-09-17 `1283afd0d`; 2026-09-28 `c1449660c`; 2026-10-02 `69f0be6f0` Bedrock block_binding.

## Evidence commits
387cc97ba · 29379ea0a · 934e7e470 · 4f757fbe2 · e42e9e630 · 9a438465e · b18f401d9 · 5d3e7d5aa · a0d839ce8 · f7df47408 · d1fce2ba1 · 84cdd0240 · 9825c13f5 · 8fc2b7682 · b4e7d5c44 · 4c175790b · 87d71380e · c961cda2c · e3f6912d4 · 9b103e5e4 · 21d80deda · 458a7bc27 · b91bdd5a3 · 6731a0ba9 · 1f0dbc008 · 6138f5a07 · d57e531f5 · c5ad7c1b0 · 7aab6c26e · 4e69b0c28 · 1283afd0d · c1449660c · 69f0be6f0 · a37306d43 · acbf8eca0

## Quirks
- Managed-effort models cannot turn thinking off (`thinkingLevelMap.off = null` forced, generate-models.ts:831).
- Effort markers invented only for same api+provider assistants with valid effort (anthropic-messages.ts:1449-1461; test anthropic-mid-conversation-effort.test.ts:139); legacy/other-provider history gets none.
- OpenRouter Opus 5 excluded from managed effort — rejects `configuration_update` (generate-models.ts:604-607).

## Failures
- [[thinking-tag-mimicry]] · [[gemini-unsigned-tool-call-replay]] · [[signed-empty-reasoning-dropped]] · [[empty-signature-semantics-vary]] · [[aborted-reasoning-signature-invalid]] · [[foreign-reasoning-signature-replayed]] · [[model-relabel-breaks-same-model-check]] · [[stale-thinking-signature-after-prefix-change]] · [[opaque-reasoning-payload-lost]] · [[reasoning-not-replayed-degrades-tool-args]]
