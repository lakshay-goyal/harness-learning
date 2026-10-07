---
type: failure
concepts: [unified-provider-api]
harnesses: [pi]
---
**Symptom**
- Reasoning replay was lost or wrong when Responses items finished out of order (#6009).
- An empty content delta in the middle of a GLM-on-Mistral thinking run split one thinking block into two. Replay then failed with "Expected at most one leading ThinkChunk" (#9674).
- chutes.ai thinking appeared twice.
- Anthropic initial text and thinking were dropped.
- Gemini thought text leaked into the answer, or answer text was classified as thinking.
- OpenRouter `reasoning_details` arriving before tool deltas broke ordering (#5114).
- Responses reasoning summary parts ran together.

**Root cause** Each parser assumed a single "current item" and a clean delta stream. Out-of-order `output_index` events, zero-length deltas that opened new blocks, servers sending both `reasoning_content` and `reasoning`, and content placed in `content_block_start` (assumed empty) all broke that assumption. On Gemini, any `thoughtSignature` was treated as marking thinking (`e42e9e630`), but the signature is replay data that can sit on ANY part.

**Fix · [[pi]]**
- `8c9dbffa3` 2026-06-25: slot table keyed by `output_index` (`packages/ai/src/api/openai-responses-shared.ts:441-533`) (#6009).
- `8930b9ec0` 2026-09-25: ignore empty content deltas before block bookkeeping (`packages/ai/src/api/mistral-conversations.ts:633-636,659,678`) (#9674).
- `36e774282` 2026-01-04: use the first non-empty of `reasoning_content`/`reasoning`/`reasoning_text` (`packages/ai/src/api/openai-completions.ts:607-638`).
- `59ad3dead` 2026-07-31: text and thinking blocks are initialized with their start-event content (`packages/ai/src/api/anthropic-messages.ts:663-722`).
- `e42e9e630` 2026-01-06 treated the signature as thinking. Reverted by `4f757fbe2` 2026-01-12: only `part.thought === true` counts as thinking, and a signature on a text part is stored as `textSignature` (`packages/ai/src/api/google-shared.ts:113-130,141-144`).
- `7d0497fdb` 2026-06-22: `reasoning_details` ordered before tool deltas (#5114).
- `31f5c2327` 2026-05-06: `reasoning_summary_part.done` emits a `"\n\n"` delta (`openai-responses-shared.ts:605-634`).

**Lesson** Treat a provider stream as a multiplexed set of items keyed by index. Drop zero-length deltas before they reach block bookkeeping. Keep "is reasoning" separate from "carries opaque reasoning state".

Related: [[unified-provider-api]] · [[signed-reasoning-replay]] · [[streamed-tool-call-fragmentation]] · [[opaque-reasoning-payload-lost]] · [[pi--unified-provider-api|pi]]
