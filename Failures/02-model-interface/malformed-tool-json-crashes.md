---
type: failure
concepts: [streaming-json-repair, tool-description-design, patch-envelope-edit, constrained-tool-sampling]
harnesses: [pi, codex]
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

**Fix · [[codex]]** — the patch tool's JSON wrapper.
- Symptom: GPT-4.1 produced predictably invalid `apply_patch` invocations (`6fcc528a43` body). JSON-wrapped patch strings also needed escaping.
- `6fcc528a43` 2025-06-03: lenient parse.
- `4764fc1ee7` 2025-10-04: freeform custom tool with a Lark grammar, so the sampler itself is constrained (`codex-rs/core/src/tools/handlers/apply_patch_spec.rs:5-27`).
- `e783341b70` 2026-05-08: the JSON/function variant was deleted.
- Codex never parses streamed argument JSON. Tool calls come only from the completed `output_item.done` item (`codex-rs/codex-api/src/sse/responses.rs:353-358`).

**Lesson** — Model-emitted JSON is untrusted input. Repair it for previews, never let a strict parser abort the turn, and strict-parse before execution. For structured free text such as patches, a grammar-constrained freeform tool removes the JSON layer entirely.

Related: [[streaming-json-repair]] · [[unified-provider-api]] · [[stream-scratch-state-persisted]] · [[length-truncated-tool-calls-executed]] · [[pi--streaming-json-repair|pi]] · [[constrained-tool-sampling]] · [[patch-envelope-edit]] · [[codex--constrained-tool-sampling|codex]]
