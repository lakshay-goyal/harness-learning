---
type: failure
concepts: [streaming-json-repair]
harnesses: [pi]
---
**Symptom**
- The Anthropic SDK stream hard-failed on tool-call JSON containing invalid escapes (`\d`) or raw control characters or newlines inside strings (#3175).
- OpenAI streams failed on malformed trailing JSON (#1424).
- Renderers crashed when models sent objects where strings were expected (#1259).
- Before partial parsing existed, streaming args were `undefined` until the stream completed.

**Root cause** The SDK SSE layer calls strict `JSON.parse` on every delta, so a recoverable glitch in model-emitted JSON aborted the whole turn.

**Fix · [[pi]]**
- `39c626b6c` 2025-09-16: partial JSON parsing of streaming tool args with `partial-json`. Args are always an object, never undefined.
- `4b926a30a` 2026-04-21: pi owns Anthropic SSE decoding (`client.beta.messages.create(params).asResponse()` + `iterateSseMessages`) and uses `parseJsonWithRepair` (`packages/ai/src/api/anthropic-messages.ts:400-551`) (#3175). It was reverted by `fc9220d2d` and reapplied by `e58d631c8` the same day, 2026-04-21. The revert added `agent-session-model-switch-thinking.test.ts`; the regression behind the revert is unverified.
- `ed0cfcbda` 2026-02-12: tolerate malformed trailing JSON on the OpenAI side (#1424).
- `1614e95ec` 2026-02-05: guard renderers (#1259).
- Current behavior: `repairJson` escapes raw control characters inside strings and doubles backslashes before invalid escapes (`packages/ai/src/utils/json-parse.ts:27-83`). `parseStreamingJson` tries JSON.parse+repair, then partial-json, then partial on the repaired text, else `{}`, and never throws (`json-parse.ts:104-124`).

**Lesson** Model-emitted JSON is untrusted input. Repair it for previews, never let a vendor SDK's strict parser abort the turn, and strict-parse before execution.

Related: [[streaming-json-repair]] · [[unified-provider-api]] · [[stream-scratch-state-persisted]] · [[length-truncated-tool-calls-executed]] · [[pi--streaming-json-repair|pi]]
