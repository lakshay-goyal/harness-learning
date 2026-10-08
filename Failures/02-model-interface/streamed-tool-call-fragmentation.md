---
type: failure
concepts: [unified-provider-api, tool-call-id-normalization]
harnesses: [pi]
---
**Symptom** One streamed tool call arrived as several bogus calls, or two calls fused into one. Gateways changed ids mid-stream and Mistral continuation chunks came without an id. llama.cpp (Responses) turned two parallel calls into three (`echo a, echo a, echo b`), two of them sharing an id, and the agent ran the mixed-up commands. Args that arrived only in `.done` were never emitted. Function args were discarded when the same delta also carried an empty `custom` object.

**Root cause** The parsers tracked a single "current block" cursor keyed by provider id, or keyed by a composite `${callId}:${index}`. Servers sent the index and id inconsistently, skipped deltas, or omitted `output_index`. The loop also executed every tool call in the final message, including calls never closed by `output_item.done`.

**Fix · [[pi]]**
- `01509156b` 2026-04-23: coalesce malformed streamed tool calls by stable stream index (`packages/ai/src/providers/openai-completions.ts:153-178` at `01509156b`; moved to `src/api/` in `ba93da9a9`) (#3576). `6b271842e` 2026-05-07: blocks tracked by both the `index` and `id` maps, with text/thinking as singletons (HEAD `packages/ai/src/api/openai-completions.ts:406-407,499-556,640-668`) (#4228).
- `34239180a` 2026-07-29: prefer `function` over an empty `custom` payload (`packages/ai/src/api/openai-completions.ts:510,546`) (#7160, PR #7288; released 0.83.0).
- `6c87d9a02` 2026-08-28: Mistral chunks merged by `toolCall.index ?? callId`. Missing or "null" id becomes `deriveMistralToolCallId("toolcall:<index>")` (`packages/ai/src/api/mistral-conversations.ts:702-706`) (#8387).
- `2f8019b61` 2026-04-02: Responses `function_call_arguments.done` computes the missing suffix delta when the stream skipped deltas (`packages/ai/src/api/openai-responses-shared.ts:661-671`; at fix time `:411-414`) (#2745). Related: `151099e17` 2026-01-24.
- `1b2aa0ca0` 2026-09-28: a `toolUse` stop with any call still holding scratch buffers (no `output_item.done`) throws instead of executing (`packages/ai/src/api/openai-responses-shared.ts:766-776`) (#9974).

**Lesson** Assemble streamed tool calls by stream index and id together, never by a single cursor. Hand a call to the executor only after the provider has finalized it.

Related: [[unified-provider-api]] · [[tool-call-id-normalization]] · [[streaming-json-repair]] · [[truncated-tool-call-guard]] · [[stream-delta-assembly-errors]] · [[length-truncated-tool-calls-executed]] · [[pi--unified-provider-api|pi]]
