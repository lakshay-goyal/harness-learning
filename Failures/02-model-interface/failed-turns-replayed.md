---
type: failure
concepts: [transcript-replay-repair]
harnesses: [pi]
---
**Symptom** — Three errors after a failed or aborted turn:
- Claude and Gemini returned 400s after a 429 or 500 hit in the middle of tool execution.
- Mistral returned 400 after an aborted, empty assistant message.
- OpenAI Responses returned 400 "reasoning without following item".

**Root cause** — Errored and aborted assistant messages were persisted and then replayed. They held empty content, partial reasoning with no following item, or incomplete tool calls.

**Fix · [[pi]]**
- `76312ea7e` (2025-12-10), #165: skip empty aborted assistant messages for Mistral.
- `fbb74bb29` (2026-01-16): filter empty error assistant messages centrally in `transformMessages` (CHANGELOG `packages/ai/CHANGELOG.md:1885`).
- `2d27a2c72` (2026-01-19), #838: skip errored and aborted assistant messages *entirely*, and delete the `strictResponsesPairing` compat flag (HEAD `packages/ai/src/api/transform-messages.ts:195-203`; `CHANGELOG.md:1845`). This also superseded the text downgrade from [[aborted-reasoning-signature-invalid]].
- Adapter backstops:
  - Completions skips an assistant with no content and no tool_calls (`openai-completions.ts:1393-1404`).
  - Bedrock skips an empty assistant message (`bedrock-converse-stream.ts:1010-1014`).
- Durable variant: `aborted`/`error`/`deferred` stop reasons are excluded from requests (`packages/durable/src/harness/context.ts:9`).

**Lesson** — Incomplete turns are not history. Keep them in the raw log, but resume the provider context from the last valid state.

Related: [[transcript-replay-repair]] · [[partial-message-persistence]] · [[orphaned-tool-calls-and-results]] · [[abandoned-attempts-left-in-context]] · [[pi--transcript-replay-repair|pi]]
