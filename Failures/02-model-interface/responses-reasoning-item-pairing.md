---
type: failure
concepts: [cross-provider-handoff, tool-call-id-normalization]
harnesses: [pi, opencode]
---
**Symptom** — OpenAI Responses returned 400s in several cases:
- "function_call without required reasoning item" after switching models within OpenAI.
- `Expected an ID that begins with 'ctc'` when switching OpenAI models via Radius after a codemode (grammar) call.
- Duplicate synthetic message ids after switching from Anthropic or Codex (#5148).
- The commentary vs final_answer phase was lost across turns (#1819).

**Root cause** — OpenAI tracks server-side which `fc_*` and `msg_*` item ids were paired with `rs_*` reasoning items. Item ids are typed by prefix: `fc_` for function_call, `ctc_` for custom_tool_call, `msg_`, `rs_`. Replaying foreign or coerced ids violates that pairing.

**Fix · [[pi]]**
- `d327b9c76` (2026-01-22): drop the item id when the provider and api are the same but the model differs (`packages/ai/src/api/openai-responses-shared.ts:290-327`).
- `bc2d8dc1c` (2026-10-01): drop item ids whose prefix does not match the replay type (`:302-305`).
- `87d71380e` (2026-03-05), #1819: `TextSignatureV1 {v:1,id,phase}` preserves `phase` (`packages/ai/src/types.ts:395-405`; `openai-responses-shared.ts:53-77,269-289`).
- `3f1ce9b6e` + `d1fb34bc8` (2026-05-28), #5148: unique and valid fallback ids `msg_pi_<n>[_<k>]`, with ids over 64 chars becoming `msg_<shortHash>`.
- `02bd2d1c6` (2026-08-07): tool `namespace` is replayed only for the same model (`:316,325`).
- Related: `2d27a2c72` handles errored turns ("reasoning without following item") → [[failed-turns-replayed]].

**Fix · [[opencode]]** `8c2aec43b8` 2025-09-17 → reverted the same day `3c3d6b65c2` → re-landed `5d95846df1` 2025-09-26 after `e3e459fc50` persisted reasoning metadata ("Item of type 'reasoning' was provided without its required following item"). With `store:false` Responses item ids are stripped: `a740d2c667` 2026-04-29 Azure; `a86ecf3bba` 2026-06-08 moved before request signing (a fetch-hook rewrite broke Bedrock-mantle signatures); `f1407e41c4` 2026-06-30 stale Copilot ids (`packages/opencode/src/provider/transform.ts:501-516`). v2 re-hit it: `f254476043` 2026-06-26 stateless requests replayed item ids → 404 "item not found".

**Lesson** — Provider item ids carry type and pairing provenance. Strip ids you did not mint for this exact model, and omit an optional id rather than coerce it.

Related: [[cross-provider-handoff]] · [[tool-call-id-normalization]] · [[signed-reasoning-replay]] · [[cross-provider-tool-call-id-normalization]] · [[pi--cross-provider-handoff|pi]] · [[opencode--signed-reasoning-replay|opencode]]
