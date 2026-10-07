---
type: failure
concepts: [tool-call-id-normalization]
harnesses: [pi, opencode]
---
**Symptom** — Switching provider mid-session failed because of tool-call id shapes:
- Anthropic and Copilot rejected OpenAI Responses ids, which are 450+ chars and contain `|`.
- Bedrock rejected non-alphanumeric ids.
- Codex rejected truncated ids ending in `_`, and foreign Copilot item ids over 400 chars containing `+/=`.
- Handoff from Copilot or Codex pipe ids (`call_id|item_id`) to other providers failed.

**Root cause** — Tool-call id grammar is a per-target constraint. Anthropic requires `^[a-zA-Z0-9_-]+$` with max 64 chars (`packages/ai/src/api/transform-messages.ts:59-63`). OpenAI Chat Completions allows 40, and Responses id parts allow 64. Ids were replayed raw.

**Fix · [[pi]]** — chronological:
1. `6679a83b3` (2025-09-04): sanitize ids for Anthropic.
2. `c5543f758` (2025-12-15), #198: Copilot-only normalization.
3. `934e7e470` (2026-01-12) / `3c60ffa67` (2026-01-13): normalize when the target is Anthropic or Copilot (strip `[^a-zA-Z0-9_-]`, 40 chars).
4. `2c7c23b86` (2026-01-19), #821: normalization becomes an adapter-supplied callback `normalizeToolCallId(id, model, source)`, applied only when not the same model. An id map rewrites later toolResults (`transform-messages.ts:64-68,83-90,136-142`).
5. `b74535dc8` (2026-01-14), #707 + `ba8059a50` (2026-01-16), #781: Bedrock applies the transforms with the same regex, ≤64 chars (`packages/ai/src/api/bedrock-converse-stream.ts:926-929,975-979`).
6. `605f6f494` (2026-01-29), #1022: pipe-separated ids.
7. `b2548ce48` (2026-03-18), #2328: Responses `normalizeIdPart` sanitizes, caps at 64 and strips trailing `_` (`packages/ai/src/api/openai-responses-shared.ts:154-177`).
8. `b21b42d03` (2026-03-22): foreign item ids become `fc_<shortHash(itemId)>`.

Anthropic at HEAD: `id.replace(/[^a-zA-Z0-9_-]/g,"_").slice(0,64)` (`anthropic-messages.ts:1289-1292`).

**Fix · [[opencode]]** `3033afba51` 2026-07-20: the Mistral 9-char id rule matched only "mistral"/"devstral" and missed codestral, pixtral and mixtral (`packages/opencode/src/provider/transform.ts:255-264`).

**Lesson** — Tool-call ids are provider-shaped. The target adapter should own the grammar, while the shared pass owns the id map so results follow their calls.

Related: [[tool-call-id-normalization]] · [[cross-provider-handoff]] · [[tool-call-id-collision]] · [[responses-reasoning-item-pairing]] · [[pi--tool-call-id-normalization|pi]] · [[opencode--tool-call-id-normalization|opencode]]
