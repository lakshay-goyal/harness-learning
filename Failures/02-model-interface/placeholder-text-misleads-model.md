---
type: failure
concepts: [cross-provider-handoff]
harnesses: [pi]
---
**Symptom** — Two opposite cases:
- Empty tool results told the model "(see attached image)", and the model hallucinated image contents.
- Images sent to non-vision models silently vanished. The model did not know an image had existed, and adapters dropped the blocks inconsistently.

**Root cause** — Placeholder text is part of the prompt. One placeholder covered both "empty" and "image-only" results. Silent omission gave the model no signal that content had been removed.

**Fix · [[pi]]**
- `279f53b09` (2026-07-07), #6290: empty results now say `"(no tool output)"`. `"(see attached image)"` is used only for image-only results (`packages/ai/src/api/openai-completions.ts:1407-1432`; CHANGELOG `packages/ai/CHANGELOG.md:670`).
- `2f4f283cc` (2026-04-20), #3429: explicit placeholders. User images become `"(image omitted: model does not support images)"` and tool-result images become `"(tool image omitted: model does not support images)"`. Consecutive images collapse into one placeholder (`packages/ai/src/api/transform-messages.ts:12-57`; `CHANGELOG.md:1251`).
- Per-adapter variants:
  - Mistral `[tool image omitted: …]`, `(image omitted: …)` (`packages/ai/src/api/mistral-conversations.ts:809-823,861-907`).
  - Bedrock `"<empty>"` (`bedrock-converse-stream.ts:120`).

**Lesson** — Placeholder text must be literally true. Tell the model what was removed, because silent omission and misleading filler both cause hallucination.

Related: [[cross-provider-handoff]] · [[image-normalization]] · [[empty-payload-rejections]] · [[pi--cross-provider-handoff|pi]]
