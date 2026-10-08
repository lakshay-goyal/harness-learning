---
type: implementation
harness: pi
concept: truncated-tool-call-guard
commit: b30a6dd77
files: [packages/agent/src/agent-loop.ts:263, packages/agent/src/agent-loop.ts:478, packages/ai/src/api/openai-responses-shared.ts:766, packages/ai/src/api/google-generative-ai.ts:227, packages/ai/src/utils/json-parse.ts:104, packages/durable/src/harness/generation.ts:452]
---
[[truncated-tool-call-guard]] in [[pi]].

## Mechanism
- **Loop-level guard**: if the assistant message has tool calls and `stopReason === "length"`, every call is routed to `failToolCallsFromTruncatedMessage` instead of `executeToolCalls` (`packages/agent/src/agent-loop.ts:263-270`). Comment: "A 'length' stop means the output was cut off by the token limit, so every tool call in the message may carry truncated arguments. Fail them all instead of executing potentially borked calls."
- Per call: emits `tool_execution_start` → error result `Tool call "<name>" was not executed: the response hit the output token limit, so its arguments may be truncated. Re-issue the tool call with complete arguments.` → `tool_execution_end` → toolResult message; `terminate:false` so the loop continues and the model re-issues (`agent-loop.ts:478-503`) → [[tool-error-as-result]].
- Why at loop level: streamed args are finalized by a **best-effort salvage parser** — `parseStreamingJson`: `JSON.parse`+repair → `partial-json` → partial-parse of repaired → `{}`, never throws (`packages/ai/src/utils/json-parse.ts:104-124`) → [[streaming-json-repair]]; truncated JSON therefore parses *and* validates.
- **Provider-level guards** (feed the same rule):
  - OpenAI Responses: `toolUse` with any call still holding scratch buffers (no `output_item.done`) → stream error, not executed (`packages/ai/src/api/openai-responses-shared.ts:766-776`; `1b2aa0ca0`). Trigger: llama.cpp omitted `output_index` → parallel calls became "echo a, echo a, echo b", two sharing an id, and the agent ran calls never closed.
  - Google: `toolUse` promotion only when mapped reason is `stop` (`google-generative-ai.ts:227`; vertex `:235`) — `MAX_TOKENS` + tool call previously mislabeled `toolUse`, hiding truncation (`5093641a5`, #8059). Anthropic/Bedrock `max_tokens` (and Bedrock `model_context_window_exceeded`) → `length` (`anthropic-messages.ts:1613-1639`; `bedrock-converse-stream.ts:1177-1192`).
  - Responses: only `incomplete` + `max_output_tokens` is `length`; other incomplete → error (`openai-responses-shared.ts:779-809`; `32850ef7c` #7540).
- **Recovery after the guard**: a length stop below the desired max is a "recoverable length" → post-run `_checkCompaction` case 1 omits the failed attempt (context_edit), compacts, retries once (`agent-session.ts:3002-3008`; `packages/ai/src/utils/overflow.ts:179`; `32850ef7c`) → [[overflow-recovery]], [[context-overflow-detection]]. `_overflowRecoveryAttempted` not reset on `length` messages (`agent-session.ts:1169-1171`).
- Visible error for length-stopped messages (`f14b3594c`, #4290); ambiguous repeated truncation no longer mislabeled overflow (`c7c763f5c`, #8130).

## Constants
none (rule is unconditional for `stopReason === "length"`).

## Evolution
- 2026-06-25 `f14b3594c` visible incomplete-response error (#4290).
- 2026-07-07 `351efc828` (PR #6285, fixes #6284). Squash of two approaches: (1) "stop salvaging malformed tool-call argument JSON" — strict-parse finalized args, keep raw on `ToolCall.malformedArguments`, args `{}`, loop refuses; then (2) **reverted** in favor of loop-level handling: "No new fields on ToolCall and no provider changes" (commit body). `malformedArguments` absent at HEAD (grep).
- 2026-08-03 `32850ef7c` Responses length semantics + bounded compact-and-retry (#7540).
- 2026-08-14 `5093641a5` Google length stops with tool calls preserved (#8059).
- 2026-08-17 `c7c763f5c` (#8130).
- 2026-09-28 `1b2aa0ca0` reject unfinished Responses tool calls.

## Evidence commits
`351efc828` `1b2aa0ca0` `5093641a5` `32850ef7c` `f14b3594c` `c7c763f5c`

## Quirks
- A length stop with tool calls gets error results *and* the loop continues (`hasMoreToolCalls = !terminate` = true), so the re-issue request happens inside the same run before any post-run compaction.
- Calls with complete args in the same message are also refused (all-or-nothing), deliberately.

## Durable variant (packages/durable)
- Generation starts a tool round only for `stopReason === "toolUse"` with calls; `stop`/`length` (even with calls) go to `answer` (`packages/durable/src/harness/generation.ts:452-457`) — length-stopped calls are never executed and get no explicit error result; on the next request context derivation synthesizes "Tool result unavailable: history ends before this call completed." for the orphaned calls (`packages/durable/src/harness/context.ts:10`) (inferred path); overflow errors compact once instead (`:460-476`).

## Failures
[[length-truncated-tool-calls-executed]] · [[length-stop-recovery]]
