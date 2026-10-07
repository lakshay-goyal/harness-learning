---
type: implementation
harness: codex
concept: partial-message-persistence
commit: 622e9e3696
files: [codex-rs/core/src/context/turn_aborted.rs:10, codex-rs/core/src/context_manager/normalize.rs:52, codex-rs/core/src/stream_events_utils.rs:347, codex-rs/core/src/stream_events_utils.rs:388, codex-rs/core/src/session/turn.rs:2473, e95abcdf49:codex-rs/core/src/tools/parallel.rs:363, codex-rs/core/src/session/turn.rs:2679, codex-rs/core/src/tasks/mod.rs:972]
---
[[partial-message-persistence]] in [[codex]].

## Mechanism
- **Unit of persistence = completed output item**, not turn: each `OutputItemDone` is recorded into history as it arrives (`codex-rs/core/src/stream_events_utils.rs:347`, `:388-394`); tool outputs recorded as they drain (`codex-rs/core/src/session/turn.rs:2473-2478`). Streaming deltas (text, `ToolCallInputDelta`) are UI-only and never persisted (`codex-rs/core/src/session/turn.rs:3029-3045`; transient events excluded by `codex-rs/rollout/src/policy.rs:156-215`).
- **No half item ever enters history**: tools dispatch only from complete `OutputItemDone` items (`codex-rs/core/src/session/turn.rs:2679-2784`); a call cut off mid-stream never becomes an item → [[truncated-tool-call-guard]].
- **Failed attempt keeps its completed items**: in-flight tools still drained after a stream error (`codex-rs/core/src/session/turn.rs:3156-3165`); retry rebuilds its prompt from history (`:1641-1649`), so retries continue from partial progress.
- **Abort**: aborted tools get synthetic "aborted by user" outputs (`e95abcdf49:codex-rs/core/src/tools/parallel.rs:363-380`); unanswered calls get synthetic "aborted" outputs at prompt time (`codex-rs/core/src/context_manager/normalize.rs:52-66`).
- **Interrupt marker**: hidden user-role `<turn_aborted>…</turn_aborted>` fragment (content kind `generic.turn_aborted`) — "The user interrupted the previous turn on purpose. Any running unified exec processes may still be running in the background. If any tools/commands were aborted, they may have partially executed." (`codex-rs/core/src/context/turn_aborted.rs:10-38`); recorded and flushed by `handle_task_abort` before `TurnAborted` is emitted (`codex-rs/core/src/tasks/mod.rs:972-990`, `:982-1028`).
- `response.incomplete` with reason `interrupted` is treated as completed with `end_turn = Some(false)`; other reasons are retryable stream errors (`codex-rs/codex-api/src/sse/responses.rs:418-454`).

## Evolution
- 2025-10-23 `f59978ed3d` (#5543) items bubbled up and recorded even on abort → [[turn-items-lost-on-abort]].
- 2026-01-20 `b236f1c95d` (#9043) model-visible `<turn_aborted>` marker → [[interrupted-turn-invisible-to-model]]; wording softened `09251387e0` 2026-01-26, `2b8d29ac0d` 2026-03-31; `120aa07d81` 2026-04-24 MultiAgentV2 markers non-user-authored.
- 2026-02-12 `fd7f2aedc7` (#11558) handle `response.incomplete`.
- 2026-07-15 `70a0b1eef8` keep the output-free interrupted prompt in the transcript.
- 2026-09-30 `9ef9cb1d9f` (#49475) await abort callbacks + flush before `TurnAborted` → [[listeners-see-stale-agent-state]].

## Versus pi
- [[pi--partial-message-persistence]]: pi persists the final partial assistant message with `stopReason: aborted|error` and skips it on replay; codex never persists a partial message — only completed items plus an explicit abort marker, so replay needs no skip rule for half messages.
- pi-durable commits streaming partials every 100 ms; codex has no partial-delta commits.
