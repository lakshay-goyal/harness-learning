---
type: concept
stage: failure-handling
tier: candidate
aliases: ["stream ended before message_stop", "Stream ended without finish_reason", "stream ended before a terminal response event", "ended without a terminal event", sawStop, supportsFinishReason, pending stop reason, Provider stream ended without a terminal finish event, rawFinishReason network_error]
harnesses: [pi, opencode]
---
A stream that ends without an explicit terminal event / stop reason is an error (and retryable), never a successful partial answer.

## Why
- Proxies, gateways and flaky connections close streams early; treated as "done", half answers and half tool arguments are persisted as successes and the agent silently stops ([[truncated-stream-accepted-as-success]]).
- Unfinalized tool calls from such streams would be executed ([[length-truncated-tool-calls-executed]]).
- A total stop-reason mapping (unknown → error with raw reason) is the same principle for known-but-unmapped terminals.

## Design space
- **Detection**: protocol-specific terminal marker per adapter (`message_stop`, `finish_reason`, `response.completed`, WS completion) (pi) · generic "stream closed" check.
- **Exceptions**: explicit compat flag for servers that never send one, inferring stop/toolUse (pi `supportsFinishReason:false`).
- **Classification**: retryable transport failure (pi) · terminal error.
- **Partial state**: explicit `pending` stop reason valid only in partials (pi) · nullable stop reason.
- **Parser hygiene**: flush SSE decoder at EOF so a terminal frame without trailing blank line isn't lost (pi `64eeb82a4`).
- **Check location**: only in the non-streaming `generate` fold, not in the streaming runner path (opencode v2 — gap) · in every adapter (pi).
- **Unmapped finish**: map to `unknown` and keep looping (opencode legacy).

## Implementations
- [[pi--terminal-event-required|pi]] — every adapter throws on missing terminal (Anthropic, Completions, Responses, Codex, Google, pi-messages, proxy); messages match agent retry patterns.
- [[opencode--terminal-event-required|opencode]] — legacy relies on AI SDK + `network_error` finish → retryable error; v2 runner accepts a stream with no `step-finish` as a normal end.

## Failures
- [[retry-classifier-regex-sprawl]]
- [[truncated-stream-accepted-as-success]]
- [[stop-reason-mapping-gaps]] (02-model-interface)
- [[sse-framing-errors]] (02-model-interface)
- [[length-truncated-tool-calls-executed]]
- [[streamed-tool-call-fragmentation]] (02-model-interface)

## Related
[[auto-retry-backoff]] · [[errors-as-stream-events]] · [[unified-provider-api]] · [[truncated-tool-call-guard]] · [[partial-message-persistence]]
