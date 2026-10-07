---
type: failure
concepts: [truncated-tool-call-guard, streaming-json-repair, terminal-event-required]
harnesses: [pi]
---
**Symptom** — When output hit the token cap mid tool call, pi either ran the call with silently salvaged partial JSON args or waited forever for a result (#6284). Variants: Google `MAX_TOKENS` responses with a tool call were labeled `toolUse` and executed (#8059); llama.cpp via Responses omitted `output_index`, so parallel calls became "echo a, echo a, echo b" (two sharing an id) and the agent ran calls never closed by `output_item.done`.

**Root cause** — Streamed arguments are finalized by a best-effort partial-JSON salvage parser (`packages/ai/src/utils/json-parse.ts:104-124`), so truncated args parse and validate; `stopReason:"length"` was treated like `toolUse`; tool presence was promoted to `toolUse` regardless of the real stop; unfinalized calls were handed to the loop.

**Fix · [[pi]]**
- `351efc828` 2026-07-07 (PR #6285, fixes #6284) — `stopReason === "length"` routes to `failToolCallsFromTruncatedMessage`: every call gets "was not executed… Re-issue the tool call with complete arguments" (`packages/agent/src/agent-loop.ts:263-270,478-503`). Same PR first tried strict-parse + `ToolCall.malformedArguments`, then reverted to loop-level handling ("No new fields on ToolCall and no provider changes").
- `5093641a5` 2026-08-14 — Google promotes to `toolUse` only when mapped reason is `stop` (`packages/ai/src/api/google-generative-ai.ts:227`; vertex `:235`) (#8059).
- `1b2aa0ca0` 2026-09-28 — Responses: fail the stream if `toolUse` has any call still holding scratch buffers (`packages/ai/src/api/openai-responses-shared.ts:766-776`).

**Lesson** — A length stop poisons every tool call in that message; only hand tool calls to the loop once the provider finalized them, and never execute salvaged arguments.

Related: [[truncated-tool-call-guard]] · [[streaming-json-repair]] · [[terminal-event-required]] · [[stop-reason-mapping-gaps]] · [[streamed-tool-call-fragmentation]] · [[length-stop-recovery]] · [[pi--truncated-tool-call-guard|pi]]
