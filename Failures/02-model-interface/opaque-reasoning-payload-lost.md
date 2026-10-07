---
type: failure
concepts: [signed-reasoning-replay]
harnesses: [pi, opencode, codex]
---
**Symptom** — Multi-turn reasoning continuity broke because the opaque reasoning payload never reached replay. Five variants:
- Anthropic `redacted_thinking` blocks were silently dropped.
- Encrypted reasoning from GPT on Bedrock was lost.
- Azure multi-turn with `store:false` lost `encrypted_content`.
- OpenRouter `reasoning_details` were replayed in the wrong order or merged wrongly, and were re-serialized on every delta.
- Mistral thinking was flattened to text.

**Root cause** — Adapters did not capture or round-trip the provider's opaque payload faithfully:
- Unknown block types were ignored.
- Bedrock `redactedContent` bytes were ignored.
- Azure omits `encrypted_content` on `output_item.done` and sends it only in `response.completed.output`.
- Deltas were appended as separate entries.
- Same-provider thinking was downgraded to text.

**Fix · [[pi]]**
- `9825c13f5` (2026-02-27), #1665: `redacted_thinking` becomes `ThinkingContent{redacted:true}` with text "[Reasoning redacted]" and the payload in `thinkingSignature`. It is replayed as `{type:"redacted_thinking", data}` (`packages/ai/src/api/anthropic-messages.ts:713-722,1405-1412`) and dropped for other models (`transform-messages.ts:104-106`).
- `d57e531f5` (2026-08-19), #8314: Bedrock `redactedContent` bytes are buffered in scratch `redactedChunks` and flushed to a base64 `thinkingSignature` at block stop and on every terminal path. A `Uint8Array` would serialize at about 10× the size (`packages/ai/src/api/bedrock-converse-stream.ts:661-703,344-351`). Base64 is built in 0x8000 windows (`:1397-1409`).
- `1f0dbc008` (2026-07-13), #6409/#6608: `encrypted_content` is backfilled from the terminal `response.output` (`packages/ai/src/api/openai-responses-shared.ts:534-556`).
- `c5ad7c1b0` (2026-08-25), #7994/#8605: consecutive `reasoning.text`/`reasoning.summary` entries are merged and `reasoning.encrypted` is kept discrete. `7aab6c26e` (2026-08-26), #8671: serialized into `thinkingSignature` once, at block end or on error (`packages/ai/src/api/openai-completions.ts:331-338,670-681,257-271`).
- `4c175790b` (2026-03-05): Mistral thinking is replayed as native `{type:"thinking", thinking:[{type:"text"}]}` chunks (`packages/ai/src/api/mistral-conversations.ts:854-857`).

**Fix · [[codex]]**
- `414b8be8b6` 2025-09-12 (#3539) "Always request encrypted cot": without it, follow-up stateless requests "will fail with 500". The request was then built in `414b8be8b6:codex-rs/core/src/client_common.rs`; now `include` is at `codex-rs/core/src/client.rs:961`.
- `5bcc9d8b77` 2025-09-09 (#3292) "Do not send reasoning item IDs": replayed ids caused errors once the API stopped requiring them.
- `dc2f26f7b5` 2025-11-04: `is_api_message` misclassified `ResponseItem::Reasoning`.
- `d2d00b6632` 2026-07-10 (#32206): always send reasoning parameters; the `supports_reasoning_summaries` flag was removed.
**Fix · [[opencode]]** `e3e459fc50` 2025-09-14 reasoning metadata not persisted; `260ab60c0b` 2026-01-19 Copilot Responses reasoning tracked by `output_index`; `d1d7447493` 2026-02-01 Copilot `reasoning_opaque` pairing; OpenRouter `reasoning_details` SDK patch (`839c5cda12` 2026-02-14, dropped in `c33d9996f0` 2026-03-27 once upstream); `20589d66d5` 2026-07-23 Mistral thinking chunks kept via a 714-line `@ai-sdk/mistral` patch; v2 `61390dbb49` 2026-05-21 native continuation metadata lost on round-trip.

**Lesson** — Opaque reasoning must round-trip verbatim. Take it from the most authoritative event, accumulate it natively, serialize it once, and flush scratch buffers on every terminal path. With `store:false` the harness owns replay: always request the encrypted payload, and resend it without server item ids.
Related: [[signed-reasoning-replay]] · [[signed-empty-reasoning-dropped]] · [[stream-scratch-state-persisted]] · [[pi--signed-reasoning-replay|pi]] · [[opencode--signed-reasoning-replay|opencode]]

Related: [[signed-reasoning-replay]] · [[signed-empty-reasoning-dropped]] · [[stream-scratch-state-persisted]] · [[pi--signed-reasoning-replay|pi]] · [[codex--signed-reasoning-replay|codex]]
