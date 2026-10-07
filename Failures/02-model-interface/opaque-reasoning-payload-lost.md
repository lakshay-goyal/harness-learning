---
type: failure
concepts: [signed-reasoning-replay]
harnesses: [pi]
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

**Lesson** — Opaque reasoning must round-trip verbatim. Take it from the most authoritative event, accumulate it natively, serialize it once, and flush scratch buffers on every terminal path.

Related: [[signed-reasoning-replay]] · [[signed-empty-reasoning-dropped]] · [[stream-scratch-state-persisted]] · [[pi--signed-reasoning-replay|pi]]
