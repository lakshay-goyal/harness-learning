---
type: failure
concepts: [cross-provider-handoff, signed-reasoning-replay]
harnesses: [pi]
---
**Symptom** — Claude started emitting literal `</thinking>` tags in its answers. After a model switch, the new model imitated `<thinking>` markup, so thinking showed up as text and text showed up as thinking.

**Root cause** — Unsigned reasoning (from an aborted stream or another model) was replayed as text wrapped in `<thinking>\n…\n</thinking>`. `transformMessages` was born this way in `46b5800d3` (2025-09-01), and the wrapper is still visible in the `51f5448a5` (2025-10-01) diff. The model read that markup as a few-shot example of its own output format.

**Fix · [[pi]]**
- `29379ea0a` (2025-12-24), #302: the Anthropic adapter converts unsigned thinking to plain text with no tags (`packages/ai/src/providers/anthropic.ts:466-471` at `29379ea0a`; file moved to `src/api/` in `ba93da9a9`, HEAD equivalent `packages/ai/src/api/anthropic-messages.ts:1416-1430`, which now lets `allowEmptySignature` models keep the block).
- `16e142ef7` (2026-01-08), #561: the same rule in the Google, completions and transform paths. Thinking is kept only for the same provider and model, and empty thinking is skipped. At HEAD: `packages/ai/src/api/transform-messages.ts:112-116`, `packages/ai/src/api/google-shared.ts:245-264`, `packages/ai/src/api/openai-completions.ts:1323-1328` ("no tags to avoid model mimicking them"), and CHANGELOG `packages/ai/CHANGELOG.md:2001`.
- `5d3e7d5aa` (2026-01-16): the Gemini text fallback added the note "Historical context… do not mimic this format", which shows the same worry ([[gemini-unsigned-tool-call-replay]]).
- Doc drift that remains: `packages/ai/README.md:1534-1537,1557` and the `requiresThinkingAsText` doc at `packages/ai/src/types.ts:827` still say `<thinking>` tags.

**Lesson** — Whatever format you use to serialize foreign or hidden reasoning becomes a few-shot example, and the model will imitate it. Replay it as plain text.

Related: [[cross-provider-handoff]] · [[signed-reasoning-replay]] · [[pi--cross-provider-handoff|pi]] · [[aborted-reasoning-signature-invalid]]
