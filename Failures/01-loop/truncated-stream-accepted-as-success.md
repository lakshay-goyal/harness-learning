---
type: failure
concepts: [terminal-event-required, errors-as-stream-events]
harnesses: [pi, opencode]
---
**Symptom** — Streams that ended early were persisted as successful partial answers (half answers, half tool args), and the agent stopped silently:
- an Anthropic stream with no `message_stop`;
- a Chat Completions stream with no `finish_reason`;
- a Responses stream with no terminal event;
- z.ai `network_error`.

**Root cause** — Adapters treated EOF as "done". There was no check that the protocol's terminal marker had arrived.

**Fix · [[pi]]**
- `d914d1c19` 2026-03-17 — z.ai `finish_reason:"network_error"` mapped to an error (#2313) (`packages/ai/src/api/openai-completions.ts:1562-1586`).
- `8e3bb4ff5` 2026-03-17 — "provider returned error" made retryable (#2264) (`packages/ai/src/utils/retry.ts:30-107`).
- `83592bb2d` 2026-04-29 — throws `Anthropic stream ended before message_stop` when `message_start` was seen without `message_stop` (#3936) (`packages/ai/src/api/anthropic-messages.ts:566-568`). Post-loop: still `pending` → "Anthropic stream ended without a stop reason" (`:861-870`).
- `98ffad043` 2026-05-15 — openai-completions throws "Stream ended without finish_reason" (#4345) (`openai-completions.ts:688-705`).
- `cd95c2749` 2026-06-23 — Responses requires `response.completed|incomplete`, else throws "OpenAI Responses stream ended before a terminal response event"; release 0.80.0, #5526 (`packages/ai/src/api/openai-responses-shared.ts:760-762`).
- `2c3041242` 2026-07-30 — `supportsFinishReason:false` compat infers `toolUse`/`stop` for servers that never send a finish reason (`openai-completions.ts:695-703`).
- Missing-terminal texts are in the retryable list ("ended without", "stream ended before message_stop", "stream ended before a terminal response event", `retry.ts:79-84`). Also Google "stream ended without a finish reason" (`google-generative-ai.ts:276-278`) and pi-messages "<provider> stream ended without a terminal event" (`packages/ai/src/api/pi-messages.ts:423`).
- `f10cce943` 2026-04-08 — retry "ended without" (sending any chunks) stream errors (#2892); `5ac874c84` 2026-05-12 — retry Anthropic `message_stop` stream endings (#4433); `b0c2a90e5` 2026-07-17 — retry OpenAI Responses early EOF (#6727) (`packages/ai/src/utils/retry.ts:80-84`).
- `f9a49869c` 2026-07-27 — explicit `pending` stop reason while streaming (#7151); valid only in partials (`packages/ai/src/types.ts:456`).
- `64eeb82a4` 2026-09-03 — Codex SSE terminal event without trailing blank line was lost; decoder flushed at EOF (#9047) (`packages/ai/src/api/openai-codex-responses.ts:799-859`).
- `b7f788194` 2026-09-19 — experimental remote client awaits terminal remote prompt event. Agent `streamProxy`: clean EOF without terminal → "Connection closed by proxy server before the response completed" (`packages/agent/src/proxy.ts:219-230`).

**Fix · [[opencode]]** legacy `e0b9e68a68` 2026-08-21: a raw `finish_reason: network_error` was treated as a normal finish; now a retryable `ResponseStreamError` (`packages/opencode/src/session/llm/ai-sdk.ts:88-90`). **Open in v2**: the runner consumes `llm.stream`, whose only completeness check lives in `generate` (`packages/llm/src/route/client.ts:384-392`); a stream with no `step-finish` writes no `Step.Ended`, fails nothing and ends the drain (`packages/core/src/session/runner/llm.ts:325-354`). Confirmed by code reading; no runner test covers it.

**Lesson** — The absence of a terminal event is an error, not "done". Make it retryable, and make the exception (a server that never sends one) an explicit opt-in compat flag.

Related: [[terminal-event-required]] · [[errors-as-stream-events]] · [[unified-provider-api]] · [[auto-retry-backoff]] · [[retry-classifier-regex-sprawl]] · [[stop-reason-mapping-gaps]] · [[length-truncated-tool-calls-executed]] · [[pi--terminal-event-required|pi]] · [[opencode--terminal-event-required|opencode]]
