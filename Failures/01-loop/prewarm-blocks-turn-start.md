---
type: failure
concepts: [turn-loop, abort-propagation]
harnesses: [codex]
---
**Symptom** — "initialize -> thread/start -> turn/start could sit behind the prewarm for up to five minutes, so the client would not see turn/started, and even turn/interrupt would block because the turn had not actually started yet" (`6ea041032b` body).

**Root cause** — An optional optimisation (startup WebSocket prewarm) was awaited before the turn became visible and interruptible, bounded only by the 300 s stream idle timeout.

**Fix · [[codex]]**
- `6ea041032b` 2026-03-17 "prevent hanging turn/start due to websocket warming issues (#14838)": 15 s `websocket_startup_timeout_ms` (now `DEFAULT_WEBSOCKET_CONNECT_TIMEOUT_MS`, `codex-rs/model-provider-info/src/lib.rs:71`, `:520-523`), TurnStarted emitted immediately, prewarm consumption cancellable (`codex-rs/core/src/tasks/regular.rs:45-80`); input recorded even if prewarm is cancelled (`:82-91`).
- Follow-ups: `35aaa5d9fc` 2026-05-01 "Bound websocket request sends with idle timeout"; `7769bccbb2` 2026-09-07 avoid WebSocket waits in Guardian classification.

**Lesson** — Make a turn visible and interruptible before any optional warm-up; bound every warm-up and never gate lifecycle events or cancellation on it.

Related: [[turn-loop]] · [[abort-propagation]] · [[http-transport-hardening]] · [[stream-stall-without-header-timeout]] · [[single-active-task-slot]] · [[codex--turn-loop|codex]]
