---
type: implementation
harness: pi
concept: partial-message-persistence
commit: b30a6dd77
files: [packages/agent/src/agent-loop.ts:416, packages/agent/src/agent.ts:523, packages/coding-agent/src/core/agent-session.ts:1143, packages/ai/src/api/transform-messages.ts:195, packages/ai/src/utils/assistant-message-frame.ts:8, packages/coding-agent/docs/message-types.md:145, packages/durable/src/harness/generation.ts:361, packages/durable/src/harness/agent.ts:34]
---
[[partial-message-persistence]] in [[pi]].

## Mechanism
- **During streaming** the loop keeps the live partial in `context.messages` (`stopReason: "pending"`) (`packages/agent/src/agent-loop.ts:416-441`); "Pi does not persist `"pending"` assistant messages in session JSONL" (`packages/coding-agent/docs/message-types.md:145`).
- **On abort/error**: providers set `stopReason = signal.aborted ? "aborted" : "error"` + `errorMessage` and push an `error` event carrying partial content + usage ([[errors-as-stream-events]]; e.g. `packages/ai/src/api/anthropic-messages.ts:887-896`, which also strips `index`/`partialJson` scratch). Loop replaces the partial with `response.result()` and emits `message_end` (`agent-loop.ts:443-456`) → AgentSession `appendMessage` persists it (`packages/coding-agent/src/core/agent-session.ts:1143-1163`). Loop hard-exits (`agent-loop.ts:245-256`).
- **Thrown failures** in hooks/convertToLlm/stream: `handleRunFailure` synthesizes an empty assistant with `stopReason` aborted/error + `errorMessage`, emits `message_start/end`, `turn_end`, `agent_end` so it is persisted too (`packages/agent/src/agent.ts:523-548`).
- **Replay**: aborted/errored assistants stay in JSONL but `transformMessages` drops them ("incomplete turns that shouldn't be replayed") and synthesizes `"No result provided"` `isError` results for orphaned tool calls (`packages/ai/src/api/transform-messages.ts:158-185,195-203,221-232`) → [[transcript-replay-repair]]. Retry/overflow recovery additionally omit the attempt via `context_edit` ([[context-edit-overlay]]).
- **Pre-prompt compaction** still sees aborted messages ("catches aborted responses"), post-run check skips them (`agent-session.ts:2051-2056,2954-2955`).
- **Session switches** abort and persist the outgoing turn first (`agent-session-runtime.ts:167-178`; `cefa40ed8` #7022 dangling tool calls).
- **Persistable frames** (pi-ai, SDK-facing): `AssistantMessageFrameEncoder` turns events into compact frames — start frame with empty content + `stopReason:"pending"`, skips already-covered deltas via per-block `coveredChars/deltaChars`, `toolcall_checkpoint` JSON for tool calls already advanced; terminal done/error produce no frame ("final message settlement is separate") (`packages/ai/src/utils/assistant-message-frame.ts:8-33,77-92,139-326`); pure reducer `reduceAssistantMessageFrames` validates order and reconstructs, parsing unfinished tool args with `parseStreamingJson` (`:372-490`). No non-test consumer in the repo at HEAD.
- Print mode exits 1 on error/aborted stop reason (`packages/coding-agent/src/modes/print-mode.ts:139-148`).

## Constants
| name | value | path:line |
|---|---|---|
| durable `partialIntervalMs` | 100 ms | packages/durable/src/harness/agent.ts:34 |
| durable `outputIntervalMs` | 100 ms | packages/durable/src/harness/agent.ts:35 |
| durable `PROGRESS_BYTES_PER_SECOND` | 100 KiB/s | packages/durable/src/harness/output.ts:261 |

## Evolution
- 2025-11-18 `387cc97ba` resubmitting aborted thinking blocks rejected by Anthropic → unsigned/partial thinking converted to text ([[signed-reasoning-replay]]).
- 2026-01-16 `fbb74bb29` empty 429/500 assistant messages broke tool_use→tool_result chains → filtered in `transformMessages`; later generalized to all error/aborted.
- 2026-04-14 `e2b40dfc8` `partialJson` scratch leaked into persisted tool calls (#3078) → stripped at completion.
- 2026-07-28 `cefa40ed8` abort + persist before tree navigation/session switch (#7022).
- 2026-09-25 `ff72faba2` user prompt persisted before first response ([[session-lost-before-first-response]]).
- Durable 1.0.3 (`packages/durable/CHANGELOG.md:74`): tail-retained tool output no longer depends on progress-commit timing ([[output-window-depends-on-commit-cadence]]).

## Evidence commits
`387cc97ba`, `fbb74bb29`, `e2b40dfc8`, `cefa40ed8`, `ff72faba2`

## Quirks
- An aborted run with tool calls may make one extra immediately-aborted provider request (no signal check between tool batch and next request, `agent-loop.ts:183-242`) → one extra aborted entry persisted (inferred, untested).
- Persisted aborted partials remain visible in UI/export but never in context — a user can see text the model never "remembers".

## Durable variant (packages/durable)
- Partials **committed while streaming** every `partialIntervalMs` (default 100) with one commit in flight (`packages/durable/src/harness/generation.ts:361-372`); "a crash loses at most that window"; remote storage hosts may raise it (`packages/durable/README.md:304`).
- `request` phase first converts any leftover committed partial into an aborted `pi.assistant` entry, then "Recovery resends the same committed messages with the same pinned model, thinking level, and stream options" (`packages/durable/docs/spec.md:3522-3531`).
- Context derivation excludes `aborted`/`error`/`deferred` assistants (`src/harness/context.ts:9`).

## Failures
- [[output-window-depends-on-commit-cadence]] · see also [[session-lost-before-first-response]]
