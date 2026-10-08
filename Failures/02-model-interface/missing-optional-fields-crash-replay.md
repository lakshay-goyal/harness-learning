---
type: failure
concepts: [transcript-replay-repair, unified-provider-api]
harnesses: [pi]
---
**Symptom** — Later turns failed or crashed for three related reasons:
- `tool_use.input` was missing after Gemini no-arg tool calls.
- `content is not iterable` crashed replay and agent code (#6259, #6276).

**Root cause** — Optional or nullable fields reached code that assumed they were present:
- Gemini omits `args` for no-arg tools, which left `arguments: undefined`.
- Untyped JS extensions and old or hand-edited session files carry `content: null`.

**Fix · [[pi]]**
- `a6d878e80` (2026-01-25): Google sends `functionCall{args ?? {}}` (`packages/ai/src/api/google-shared.ts:265-276`).
- `af813f904` (2026-01-30): default tool-call arguments across adapters. Anthropic sends `input: arguments ?? {}` (`packages/ai/src/api/anthropic-messages.ts:1439-1446`).
- `8c0ccd14b` (2026-07-06), #6343: normalize `content == null → []` at ingestion boundaries: tool results, session load, extension messages and `transformMessages` (`packages/ai/src/api/transform-messages.ts:71-73`; `packages/ai/CHANGELOG.md:659`).
- Earlier: `39c626b6c` (2025-09-16) made streamed args always an object, never undefined.

**Lesson** — Normalize optional fields once at the choke point where data is ingested, instead of guarding every consumer.

Related: [[transcript-replay-repair]] · [[unified-provider-api]] · [[streaming-json-repair]] · [[pi--transcript-replay-repair|pi]]
