---
type: failure
concepts: [abort-propagation, headless-rpc-mode]
harnesses: [codex]
---
**Symptom** — App-server queued `turn/interrupt` and answered only when a later `TurnAborted` arrived; if the turn had already completed, core treated `Op::Interrupt` as a no-op, so the RPC hung forever (`11806faf71` body). Related: headless `codex exec` never exited after Ctrl-C when streaming over websockets ("immortal `codex exec`", `e3d39013d3`).

**Root cause** — A request/response RPC was answered by an event that is only emitted when there is something to abort.

**Fix · [[codex]]**
- `11806faf71` 2026-04-24 "Fix hang on turn/interrupt (#18392)" (`codex-rs/app-server/src/thread_state.rs`, `12a729f2b2:codex-rs/app-server/src/codex_message_processor.rs`).
- `e3d39013d3` 2026-02-03 "Handle exec shutdown on Interrupt".
- Core at HEAD: `Op::Interrupt` on an idle session cancels MCP startup instead (`codex-rs/core/src/session/mod.rs:5006-5013`); conditional `Op::InterruptIfNoPendingInput{turn_id, reply}` replies via oneshot (`codex-rs/core/src/session/handlers.rs:496-500`).

**Lesson** — An abort request must be acknowledged even when there is nothing to abort; RPCs cannot wait on an event that may never be emitted.

Related: [[abort-propagation]] · [[headless-rpc-mode]] · [[client-server-session-split]] · [[agent-event-stream]] · [[codex--abort-propagation|codex]]
