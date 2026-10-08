---
type: implementation
harness: pi
concept: terminal-event-required
commit: b30a6dd77
files: [packages/ai/src/api/anthropic-messages.ts:567, packages/ai/src/api/anthropic-messages.ts:866, packages/ai/src/api/openai-completions.ts:702, packages/ai/src/api/openai-responses-shared.ts:761, packages/ai/src/api/google-generative-ai.ts:277, packages/ai/src/api/pi-messages.ts:423, packages/agent/src/proxy.ts:224, packages/ai/src/utils/retry.ts:80, packages/ai/src/types.ts:456]
---
[[terminal-event-required]] in [[pi]].

## Mechanism
- **Protocol**: every stream ends with exactly one `done` (reason stop/length/toolUse/deferred) or `error` (aborted/error) event; `StopReason` includes `"pending"`, valid **only** in partials (`packages/ai/src/types.ts:456`; `packages/ai/README.md:1093`); rules: `start` before updates, `done`/updates never before `start`, setup failure may `error` without `start` (`packages/ai/src/types.ts:773-787`) → [[unified-provider-api]]. `AssistantMessageEventStream` completes on `done|error`; `push()` ignored after done (`packages/ai/src/utils/event-stream.ts:43-44,102-111`). Persistable-frame reducer throws "event follows a terminal event" (`packages/ai/src/utils/assistant-message-frame.ts:145`).
- **Per-adapter "no terminal = error" checks** (all thrown inside the adapter → caught → `error` event with partial content, [[errors-as-stream-events]]):

| Adapter | Check | path:line | commit |
|---|---|---|---|
| Anthropic Messages | saw `message_start` but no `message_stop` → "Anthropic stream ended before message_stop" | `packages/ai/src/api/anthropic-messages.ts:566-568` | `83592bb2d` (#3936) |
| Anthropic Messages | still `pending` after loop → "Anthropic stream ended without a stop reason" | `anthropic-messages.ts:861-870` | — |
| OpenAI Chat Completions | no `finish_reason` → "Stream ended without finish_reason" unless compat `supportsFinishReason:false` (then infer `toolUse`/`stop`) | `packages/ai/src/api/openai-completions.ts:688-705` | `98ffad043` (#4345), `2c3041242` |
| OpenAI Chat Completions | unknown/`network_error`/`content_filter` finish_reason → error "Provider finish_reason: X", raw kept | `openai-completions.ts:1562-1586`, `:582` | `a79ca4119`, `d914d1c19` (#2313), `fe1c9b6d5` |
| OpenAI Responses | no `response.completed/incomplete` → "OpenAI Responses stream ended before a terminal response event"; `toolUse` with unfinished call → throw; post-check `pending` → throw | `openai-responses-shared.ts:760-776`; `openai-responses.ts:201-210` | `cd95c2749`, `1b2aa0ca0` |
| Codex (WS/SSE) | completion only on `response.completed|done|incomplete`; WS close before completion → error; SSE decoder flushed at EOF, residual treated as frame | `openai-codex-responses.ts:1307-1423`, `:799-859` | `64eeb82a4` (#9047) |
| Google / Vertex | no finishReason → "Google [Vertex] stream ended without a finish reason" | `google-generative-ai.ts:276-278`; `google-vertex.ts:285` | `f9a49869c` (#7151, `pending` stop reason) |
| pi-messages (Radius) | "<provider> stream ended without a terminal event" | `packages/ai/src/api/pi-messages.ts:423` | — |
| agent `streamProxy` | clean EOF without terminal event → "Connection closed by proxy server before the response completed" | `packages/agent/src/proxy.ts:219-230` | — |

- **Made retryable**: agent-level classifier contains `"ended without"`, `"stream ended before message_stop"`, `"stream ended before a terminal response event"` (`packages/ai/src/utils/retry.ts:80-84`) → [[pi--auto-retry-backoff|auto-retry-backoff]]. So a truncated stream = transient transport failure → `_prepareRetry` → omitted from projection → retried.
- Stop-reason mapping must be total: Anthropic `mapStopReason` default arm throws `Unhandled stop reason: X` → error; `refusal` → error with `stop_details.explanation`; `sensitive` → error (`anthropic-messages.ts:1613-1639`; `eb1f87fa9`, `ee7c0a7d1` #978); Google exhaustive `never` check (`google-shared.ts:441-469`); Bedrock unknown → `Provider stopped with: <reason>` (`bedrock-converse-stream.ts:1177-1192`; `f8f75544b`); Responses status map exhaustive, `queued/in_progress → stop` ("wonky") (`openai-responses-shared.ts:779-809`).
- Downstream consumers also demand a terminal: eval harness errors if last assistant `stopReason` ∉ {stop, toolUse} or `stop` with empty text (`packages/evals/src/harness.ts:247-253`); remote client awaits terminal prompt event (`b7f788194`).

## Constants
| name | value | path:line |
|---|---|---|
| compat flag | `supportsFinishReason` (default true) | `packages/ai/src/api/openai-completions.ts:695-703` |

## Evolution
- 2026-03-17 `d914d1c19` z.ai `network_error` finish → error (#2313); `8e3bb4ff5` "provider returned error" retryable (#2264).
- 2026-04-08 `f10cce943` retry "ended without" (sending any chunks) errors (#2892).
- 2026-04-29 `83592bb2d` Anthropic incomplete streams (previously persisted as successful partial) (#3936).
- 2026-05-12 `5ac874c84` retry Anthropic missing message_stop (#4433).
- 2026-05-15 `98ffad043` Completions missing finish_reason → error (#4345).
- 2026-06-23 `cd95c2749` Responses terminal events required; 0.80.0 `response.incomplete` = length (#5526).
- 2026-07-17 `b0c2a90e5` retry Responses early EOF (#6727).
- 2026-07-27 `f9a49869c` expose `pending` stop reason while streaming (#7151).
- 2026-07-30 `2c3041242` `supportsFinishReason:false` for servers that never send one.
- 2026-09-03 `64eeb82a4` Codex SSE terminal event without trailing blank line (#9047).
- 2026-09-19 `b7f788194` remote client awaits terminal prompt event.
- 2026-09-28 `1b2aa0ca0` unfinished Responses tool calls rejected.

## Evidence commits
`d914d1c19` `8e3bb4ff5` `f10cce943` `83592bb2d` `5ac874c84` `98ffad043` `cd95c2749` `b0c2a90e5` `f9a49869c` `2c3041242` `64eeb82a4` `b7f788194` `1b2aa0ca0` `a79ca4119` `fe1c9b6d5` `eb1f87fa9` `ee7c0a7d1` `f8f75544b`

## Quirks
- Google `mapStopReasonString` legacy mapper appears unused (unverified) (`google-shared.ts:474-483`).
- Responses `phase:"final_answer"` sets a provisional `stop` later overwritten by terminal status (`openai-responses-shared.ts:443-447`).
- Mid-stream failures are not retried by adapter layer (only request→headers), so all terminal-missing cases rely on agent-level retry.

## Durable variant (packages/durable)
- Partials committed every `partialIntervalMs` (100 ms); `request` phase first converts any leftover committed partial into an **aborted** `pi.assistant` entry (`spec.md:3522-3531`; `generation.ts:361-372`) — a crash = missing terminal event, resolved by re-request.
- Context derivation excludes `aborted`/`error`/`deferred` assistant messages (`harness/context.ts:9`).

## Failures
[[truncated-stream-accepted-as-success]] · [[length-truncated-tool-calls-executed]]
