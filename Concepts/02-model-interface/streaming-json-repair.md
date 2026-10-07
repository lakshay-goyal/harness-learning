---
type: concept
stage: tool-design
tier: candidate
aliases: [parseStreamingJson, repairJson, parseJsonWithRepair, partial-json, eager-tool-input-streaming, defensive-streaming-json, eager_input_streaming, fine-grained-tool-streaming]
harnesses: [pi]
---
Parse streamed tool-argument JSON tolerantly. While the call is streaming, the parsed value is only a preview. Model-emitted JSON is treated as untrusted:
- repair raw control characters and invalid escapes,
- fall back to a partial parse,
- never let a strict vendor parser abort the turn.

## Why
- Models emit invalid escapes (`\d`) and raw newlines inside JSON strings. A strict `JSON.parse` in an SDK's stream helper kills the whole turn ([[malformed-tool-json-crashes]]).
- UIs need argument previews while the call is still streaming.
- Scratch buffers that hold partial JSON must not leak into persisted calls ([[stream-scratch-state-persisted]]).
- Lenient coercion has limits ([[tool-arg-coercion-breaks-unions]]).
- Truncated arguments must never execute ([[length-truncated-tool-calls-executed]]).

## Design space
- **Parser**
  - Strict parse, failing the stream.
  - Repair, then a partial parser, then `{}` without ever throwing during streaming. *pi chose this.*
  - Own the SSE decoding so the SDK's strict parsing is bypassed: 4b926a30a, reapplied as e58d631c8.
- **Finalization**
  - Accept salvaged arguments.
  - Strict-parse at execution and refuse calls that were length-truncated or unfinished (loop side).
- **Streaming mode**
  - A legacy beta header.
  - Per-tool eager input streaming, gated by a compat flag.
- **Scratch state**
  - Delete `partialJson` and `partialArgs` when the block finalizes and on error paths.
- **No repair**: strict parse; malformed tool JSON or an undecodable frame fails the whole turn with no per-call error the model could correct (opencode v2 `packages/llm/src/protocols/shared.ts:97-101`).

## Implementations
- [[pi--streaming-json-repair|pi]] — `packages/ai/src/utils/json-parse.ts` (`repairJson`, `parseStreamingJson`, `parseJsonWithRepair`), re-parsed on every tool delta in all adapters. The scratch fields are stripped at block stop.

## Failures
- [[malformed-tool-json-crashes]]
- [[stream-scratch-state-persisted]]
- [[tool-arg-coercion-breaks-unions]]
- [[length-truncated-tool-calls-executed]]

## Related
[[unified-provider-api]] · [[constrained-tool-sampling]] · [[tool-argument-repair]] · [[truncated-tool-call-guard]] · [[errors-as-stream-events]]
