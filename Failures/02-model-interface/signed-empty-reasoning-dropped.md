---
type: failure
concepts: [signed-reasoning-replay]
harnesses: [pi]
---
**Symptom** — Three variants, all with the same cause:
- Newer Claude models errored on replay when a thinking block with empty text but a valid signature (`display:"omitted"`) had been dropped.
- Gemini Flash models ended mid-task with an empty, thought-only `STOP` and no tool call.
- OpenAI Responses reasoning replay broke one day after an "omit empty thinking" change (#1878).

**Root cause** — Skip rules looked at the visible text and treated an empty block as disposable. The signature or encrypted payload is the actual replay state, and the provider requires it to be echoed back.

**Fix · [[pi]]**
- `8fc2b7682` (2026-03-05) skipped empty-text Responses thinking blocks. That broke replay, and `b4e7d5c44` (2026-03-06, #1878) restored it. Encrypted reasoning items with empty summaries must still be replayed (`packages/ai/src/api/openai-responses-shared.ts:264-268`).
- `6731a0ba9` (2026-07-09), #6457: Anthropic skips a thinking block only when the text is empty AND there is no signature (`packages/ai/src/api/anthropic-messages.ts:1413-1415`).
- `6138f5a07` (2026-07-31), #7362: Google drops empty text and thinking parts only when they are unsigned (`packages/ai/src/api/google-shared.ts:236-240,250-252`).
- Shared handoff rule: same-model thinking with a signature is kept even when its text is empty (`packages/ai/src/api/transform-messages.ts:107-109`).

**Lesson** — The signature, not the text, is the replay payload. A block with empty visible text can carry required state.

Related: [[signed-reasoning-replay]] · [[empty-signature-semantics-vary]] · [[opaque-reasoning-payload-lost]] · [[pi--signed-reasoning-replay|pi]]
