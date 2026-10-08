---
type: concept
stage: failure-handling
tier: must-have
aliases: [failToolCallsFromTruncatedMessage, "stopReason length", truncated-turn-tool-refusal, unfinished-tool-call-guard, malformedArguments, partialJson check, OutputItemDone-only dispatch, response.incomplete, "Incomplete response returned, reason"]
harnesses: [pi, codex]
---
Never execute tool calls from an output-length-truncated or provider-unfinalized assistant message; answer them with an error telling the model to re-issue.

## Why
- Streamed tool arguments are usually finalized by tolerant partial-JSON parsers, so truncated arguments parse and validate — the harness runs a different command than the model intended ([[length-truncated-tool-calls-executed]]).
- Providers that promote "has tool calls" to a tool-use stop reason hide the truncation.

## Design space
- **Where**: loop level on `stopReason === length` (pi `351efc828`) · strict-parse finalized args + `malformedArguments` field (pi tried, reverted same PR) · adapter rejects unfinalized calls (pi Responses `1b2aa0ca0`) · structural: dispatch only from complete `output_item.done` items, argument deltas feed UI only — a cut-off call never becomes an item (✔ codex).
- **Scope**: all calls in the message (pi) · only calls whose JSON fails strict parse.
- **Response**: error tool results + continue so model re-issues (pi stable) · treat as answer, no tools (pi-durable) · compact-and-retry when truncation is a context ceiling (pi `32850ef7c`) · retry the whole request (✔ codex: `response.incomplete` with reason ≠ interrupted/content_filter → retryable `Stream` error; already-completed items kept).
- **Server-interrupted response**: `response.incomplete` reason `interrupted` → treated as completed with `end_turn=false`, forcing a follow-up (✔ codex).
- **Stop-reason mapping**: tool presence must not override length/error (pi `5093641a5`).

## Implementations
- [[pi--truncated-tool-call-guard|pi]] — every call of a length-stopped message gets "was not executed… Re-issue the tool call with complete arguments"; Responses unfinished calls → stream error.
- [[codex--truncated-tool-call-guard|codex]] — tools dispatched only from `ResponseEvent::OutputItemDone`; `response.incomplete` (e.g. max_output_tokens) → retryable "Incomplete response returned, reason: …".

## Failures
- [[length-truncated-tool-calls-executed]]
- [[length-stop-recovery]]

## Related
[[turn-loop]] · [[terminal-event-required]] · [[streaming-json-repair]] · [[tool-error-as-result]] · [[context-overflow-detection]] · [[overflow-recovery]] · [[max-tokens-context-clamp]]
