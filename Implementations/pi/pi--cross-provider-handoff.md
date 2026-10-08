---
type: implementation
harness: pi
concept: cross-provider-handoff
commit: b30a6dd77
files: [packages/ai/src/api/transform-messages.ts:12, packages/ai/src/api/transform-messages.ts:64, packages/ai/src/api/transform-messages.ts:95, packages/ai/src/api/transform-messages.ts:131, packages/ai/src/api/openai-completions.ts:1193, packages/ai/src/api/openai-responses-shared.ts:145, packages/ai/src/api/google-shared.ts:146, packages/ai/src/api/anthropic-messages.ts:1308, packages/ai/src/api/bedrock-converse-stream.ts:968, packages/ai/src/api/mistral-conversations.ts:130, packages/ai/src/types.ts:558]
---
[[cross-provider-handoff]] in [[pi]].

## Mechanism
**Shared pass `transformMessages(messages, model, normalizeToolCallId)`** (packages/ai/src/api/transform-messages.ts), called by every adapter with its own id callback (openai-completions.ts:1228, anthropic-messages.ts:1134, google-shared.ts:200, bedrock-converse-stream.ts:975, openai-responses-shared.ts:179, mistral-conversations.ts:142).
- Pass 0: `msg.content == null → []` (:71-73; 8c0ccd14b #6343).
- Pass 0b **non-vision downgrade**: model lacks `"image"` input → user images → `"(image omitted: model does not support images)"`, tool-result images → `"(tool image omitted: model does not support images)"`; consecutive images collapse to one placeholder (:12-57; 2f4f283cc #3429) → [[placeholder-text-misleads-model]].
- Pass 1 per assistant message, `isSameModel = provider && api && model.id equal` (:95-98) — compares the **requested** `model`, not server-echoed `responseModel` (packages/ai/src/types.ts:558; 1283afd0d #9188) → [[model-relabel-breaks-same-model-check]].
  - redacted thinking: same-model only, else dropped (:104-106).
  - thinking with signature on same model: kept even if text empty (OpenAI encrypted reasoning) (:107-109).
  - empty/whitespace thinking dropped (:111).
  - cross-model thinking → **plain text block, no `<thinking>` tags** (:112-116; 16e142ef7 #561) → [[thinking-tag-mimicry]].
  - cross-model text rebuilt `{type,text}` → strips `textSignature` (:119-125).
  - cross-model tool call: delete Google `thoughtSignature` (:131-134; 934e7e470); id normalized via callback, recorded in `toolCallIdMap`; later toolResults remapped (:136-142, 84-90) → [[tool-call-id-normalization]].
- Pass 2 sequencing repair (failed turns, orphan results, deferred system messages) → [[pi--transcript-replay-repair|transcript-replay-repair]].

**Per-adapter handoff rules** (beyond the shared pass)

| adapter | foreign thinking | signatures / ids | content-shape rules |
|---|---|---|---|
| anthropic-messages | no/blank signature → plain `text` block (no tags) unless `allowEmptySignature` → `{type:"thinking", signature:""}` (anthropic-messages.ts:1416-1431); signed → `{thinking, signature}` (:1432-1438) | redacted → `redacted_thinking{data}` (:1405-1412); skip only if text empty AND no signature (:1413-1415; 6731a0ba9); OAuth tool names mapped to CC casing (:1443) → [[provider-identity-shim]] | whitespace user strings dropped, empty blocks dropped, message skipped if empty (:1355-1392); tool-result images: text-only → joined string, image-only → `"(see attached image)"` (:137-184); consecutive toolResults merged into one `user` msg ("needed for z.ai Anthropic endpoint") (:1462-1478); `tool_use.input = arguments ?? {}` (:1439-1446; af813f904) |
| bedrock-converse-stream | Claude → `reasoningText{text, signature}`, missing sig → plain text (c961cda2c); non-Claude → `reasoningText{text}` without signature (models reject field) (:1040-1068, 894-904; 9a438465e) | redacted → `redactedContent` bytes (invalid base64 → dropped) (:1033-1039, 1388-1395) | blank user → `"<empty>"` (:120, 931-938, 985-1007; 34719d3f4); unknown content types skipped (9e2bfc7c4); empty assistant skipped (:1010-1014); replayed toolUse input `sanitizeBedrockDocument` drops `""` keys (:940-952, 1027; 98145a6c0 #7782); unknown image mime → throw (:1350-1371) |
| openai-completions | `requiresThinkingAsText` → leading plain text part "no tags to avoid model mimicking them" (openai-completions.ts:1323-1328; 1d488626d #3387); else structured `reasoning_details` from signature (:1312-1319, 1383-1385); else raw field named by `thinkingSignature` (`reasoning_content`/`reasoning`/`reasoning_text`) (:1340-1349, list :273) | DeepSeek `reasoning_content:""` on every assistant (:1386-1392; `requiresReasoningContentOnAssistantMessages` 9b103e5e4); opencode-go `reasoning`→`reasoning_content` (:625-628, 1343-1345; 21d80deda) | assistant text always **plain joined string** (:1330-1337; 23109b113); content `null` or `""` (:1293-1296); empty assistant skipped (:1393-1404); `requiresAssistantAfterToolResult` → synthetic `"I have processed the tool results."` (:1239-1246); tool-result images → ONE synthetic user msg "Attached image(s) from tool result:" (:1434-1468); `"(no tool output)"` for empty (279f53b09); developer role iff `model.reasoning && supportsDeveloperRole` (:1233); grammar calls replayed `{type:"custom"}` (:1361-1372) |
| openai-responses (+azure/codex) | signature = JSON of full `ResponseReasoningItem` incl. `encrypted_content`, replayed verbatim (openai-responses-shared.ts:264-268); empty-summary signed items replayed (b4e7d5c44) | text → `message` item id from `TextSignatureV1` or `msg_pi_<n>[_<k>]`, >64 → `msg_<shortHash>`, `phase` preserved (:53-77, 269-289; 87d71380e, 3f1ce9b6e, d1fb34bc8); tool call id `call_id\|item_id`; item id dropped for same provider+api different model (pairs with `rs_`) or prefix mismatch (`fc_`/`ctc_`) (:290-327; d327b9c76, bc2d8dc1c); `namespace` only same model (:316, 325; 02bd2d1c6) | tool-result images inside output as `input_image` if vision (:81-108, 332-349; 933348593 #2104); `includeSystemPrompt:false` for Codex (:213, 223); developer role iff `model.reasoning && compat.supportsDeveloperRole !== false` (:214-215; b8f6f660e) |
| google (+vertex) | same provider+model → `{thought:true, text, thoughtSignature}`; else plain text (google-shared.ts:245-264) | signature kept only if same provider AND model AND valid base64 (`/^[A-Za-z0-9+/]+={0,2}$/`, len%4==0; TYPE_BYTES) (`resolveThoughtSignature` :146-160; 934e7e470); signed empty blocks kept (:236-240, 250-252; 6138f5a07); unsigned Gemini 3 tool calls sent as plain functionCall (tests google-shared-gemini3-unsigned-tool-call.test.ts:100-128; f7df47408) → [[gemini-unsigned-tool-call-replay]] | `functionCall.args ?? {}` (:265-276; a6d878e80); results `{output}`/`{error}` (:300-314; 84018b070); Gemini ≥3 nests images in `functionResponse.parts`, <3 separate user turn "Tool result image:" (:180-186, 315, 332-338; 453b22397) → [[tool-result-image-routing]] |
| mistral-conversations | same-provider thinking replayed native `{type:"thinking"}` chunks (4c175790b) | 9-char alnum ids per request map (:237-257) | tool results role `tool` with `[tool error] ` prefix, `(no tool output)`, `[tool image omitted: …]` (:861-907); non-vision user images → placeholder (:809-823) |

- Mid-conversation system messages: Anthropic holds them until before next assistant (tool_result must follow tool_use) — an update before a user message lands after it on the wire (anthropic-messages.ts:1319-1328, 1333-1354; 9e05370b2); Google/Bedrock collapse (`collapseSystemMessages`) → [[transcript-carried-system-prompt]].
- Managed-effort replay markers inserted per historical assistant (anthropic-messages.ts:1518-1532) only for same api+provider → [[pi--signed-reasoning-replay|signed-reasoning-replay]].
- Server-side handoff: pi-messages sends normalized transcript; translation happens in Radius gateway (pi-messages.ts:1-10) — not in repo.
- Custom-provider test matrix includes cross-provider handoff (packages/coding-agent/docs/custom-provider.md:156-169); historical 503-line `handoff.test.ts` (46b5800d3).

## Constants
| name | value | path:line |
|---|---|---|
| user image placeholder | `(image omitted: model does not support images)` | packages/ai/src/api/transform-messages.ts:12-57 |
| tool image placeholder | `(tool image omitted: model does not support images)` | packages/ai/src/api/transform-messages.ts:12-57 |
| image-only tool result text | `(see attached image)` | packages/ai/src/api/anthropic-messages.ts:137-184; google-shared.ts:301 |
| empty tool result text | `(no tool output)` | packages/ai/src/api/openai-completions.ts:1407-1432 |
| Bedrock blank placeholder | `<empty>` | packages/ai/src/api/bedrock-converse-stream.ts:120 |
| synthetic bridge assistant | `I have processed the tool results.` | packages/ai/src/api/openai-completions.ts:1239-1246 |

## Evolution
- 2025-09-01 `46b5800d3` `transformMessages` born; thinking → `<thinking>`-tagged text; handoff test matrix.
- 2025-12-24 `29379ea0a` Anthropic unsigned thinking → plain text (#302); 2026-01-08 `16e142ef7` tags removed everywhere (#561).
- 2026-01-12 `934e7e470` foreign thought signatures filtered (#654).
- 2026-01-19 `2c7c23b86` id normalization becomes adapter callback applied only `!isSameModel` with id map (#821).
- 2026-01-22 `d327b9c76` Responses drops item id for different model.
- 2026-03-18 `453b22397` Gemini 3+ inline tool-result images (#2052).
- 2026-04-20 `2f4f283cc` non-vision placeholders (#3429).
- 2026-04-30 `f7df47408` Gemini unsigned calls sent as-is (end of 4-step saga, #4032).
- 2026-07-06 `8c0ccd14b` null content normalized (#6343).
- 2026-07-31 `6138f5a07` signed empty blocks kept (#7362).
- 2026-09-16 `9e05370b2` mid-convo system messages; held back for adjacency (#9548).
- 2026-09-17 `1283afd0d` requested vs response model identity (#9188).

## Evidence commits
46b5800d3 · 51f5448a5 · 29379ea0a · 16e142ef7 · 2c7c23b86 · 934e7e470 · d327b9c76 · bc2d8dc1c · 02bd2d1c6 · 87d71380e · 1283afd0d · 2f4f283cc · 279f53b09 · 8c0ccd14b · 23109b113 · 1d488626d · 9a438465e · c961cda2c · 98145a6c0 · 34719d3f4 · 9e2bfc7c4 · 4c175790b · b18f401d9 · 5d3e7d5aa · a0d839ce8 · f7df47408 · 6138f5a07 · 9e05370b2

## Quirks
- **Doc drift**: packages/ai/README.md:1534-1537 (and example :1557) still says thinking is converted with `<thinking>` tags; `requiresThinkingAsText` doc mentions `<thinking>` delimiters (packages/ai/src/types.ts:827); code emits plain text since 16e142ef7.
- Anthropic non-strict tool schemas send only `{type, properties, required}` — top-level `$defs` dropped, refs would dangle (anthropic-messages.ts:1588-1600) (unverified impact).
- Anthropic tool-result image media types cast to jpeg/png/gif/webp without validation (anthropic-messages.ts:137-184); Bedrock throws on unknown mime.
- Bedrock `toolChoice:"none"` removes `toolConfig` entirely (bedrock-converse-stream.ts:1140-1175) — acceptance with tool history unverified.

## Failures
- [[thinking-tag-mimicry]] · [[gemini-unsigned-tool-call-replay]] · [[tool-call-id-requirement-drift]] · [[foreign-reasoning-signature-replayed]] · [[model-relabel-breaks-same-model-check]] · [[responses-reasoning-item-pairing]] · [[reasoning-not-replayed-degrades-tool-args]] · [[placeholder-text-misleads-model]] · [[assistant-content-shape-misread]] · [[tool-result-image-routing]]
