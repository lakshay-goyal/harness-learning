---
type: failure
concepts: [errors-as-stream-events, streaming-json-repair]
harnesses: [pi]
---
**Symptom** Streaming-only buffers (`partialJson`, `partialArgs`, `index`, `streamIndex`, `customInput`) leaked into persisted tool calls and corrupted resumed payloads (#3078). Bedrock redacted-reasoning byte chunks were lost when a stream settled without stopping every block.

**Root cause** Scratch fields lived on the same content-block objects as the durable message. They were cleared only on the happy `*_stop` path, not on errors, aborts or settlements that skipped block stops.

**Fix · [[pi]]**
- `e2b40dfc8` 2026-04-14: strip `partialJson` in place at block completion "so replay only carries parsed arguments" (`packages/ai/src/api/anthropic-messages.ts:783-814`), plus the bedrock, mistral, codex and completions adapters (#3078).
- Error paths: every adapter's `catch` strips scratch from the partial before the `error` event (`anthropic-messages.ts:887-897`; `openai-completions.ts:707-729`; `openai-responses.ts:214-231`; Mistral `mistral-conversations.ts:169-172,753`).
- `d57e531f5` 2026-08-19: Bedrock flushes buffered `redactedChunks` to a base64 `thinkingSignature` at block stop *and* on every terminal path, because "a stream can settle without stopping every block" (`packages/ai/src/api/bedrock-converse-stream.ts:683-703,344-351`).

**Lesson** Keep stream-time scratch apart from the durable message shape. Strip or flush it on every terminal path, not only at clean block ends.

Related: [[errors-as-stream-events]] · [[streaming-json-repair]] · [[partial-message-persistence]] · [[opaque-reasoning-payload-lost]] · [[pi--errors-as-stream-events|pi]]
