---
type: implementation
harness: codex
concept: durable-execution
commit: 622e9e3696
files: [codex-rs/core/src/session/turn_suspension.rs:1, codex-rs/core/src/session/turn_input.rs:293, codex-rs/core/src/session/daemon_recovery.rs:9, codex-rs/core/src/session/turn.rs:450, codex-rs/thread-store/src/store.rs:68, codex-rs/app-server-daemon/src/lib.rs:49]
---
[[durable-execution]] in [[codex]].

Turn-granular durability: persisted history + explicit suspend/recover ops let a managed daemon restart and continue an unfinished root turn by **re-sampling** from history (no per-tool checkpoint/replay policy).

## Mechanism
- **Suspend** (`Op::SuspendTurnAndShutdown`): only a Regular root turn with no live descendants; flush rollout BEFORE cancelling ("so a persistence failure leaves the original turn running"); cancel without emitting a terminal event (so another worker may recover it under the same turn id); 100 ms grace then abort; drop in-process pending input ("Handoff intentionally drops that state"); flush + close writer; `ShutdownComplete` (`codex-rs/core/src/session/turn_suspension.rs:1-119`).
- **Recover** (`Op::RecoverTurn`, `recover_turn_if_idle`, `TurnStartKind::Recovery`): starts a turn only if idle, with the original turn id and no empty user message (`codex-rs/core/src/session/turn_input.rs:293-321`).
- **Capture precondition**: daemon recovery only once the `RecordedTurnInput` marker is set (input persisted) and only with a single environment (`codex-rs/core/src/session/daemon_recovery.rs:9-40`; marker set at `codex-rs/core/src/session/turn.rs:450`).
- **Durability fences**: `PersistContext` makes user input durable before the next sampling request (`codex-rs/thread-store/src/store.rs:68-94`) → [[codex--session-tree|session-tree]].
- **Host**: app-server daemon with thread recovery after restart (`codex-rs/app-server-daemon/src/thread_recovery.rs`, `codex-rs/app-server-daemon/src/update_loop.rs`) → [[codex--client-server-session-split|client-server-session-split]].
- **Cloud variant**: gRPC cloud ThreadService client; "Requests are never retried: a timeout or disconnect can leave Resume admitted. Attach delivers live events only; dropping its stream detaches without interrupting the thread or answering approvals." (`codex-rs/cloud-client/src/lib.rs:1-5`; `b707714ae4` 2026-10-01) → [[cloud-task-delegation]].

## Constants
| name | value | path:line |
|---|---|---|
| suspend grace before abort | 100 ms (`GRACEFULL_INTERRUPTION_TIMEOUT_MS`, sic) | codex-rs/core/src/tasks/mod.rs:71; used codex-rs/core/src/session/turn_suspension.rs:78 |
| cloud gRPC request timeout | 150 s, no retry | codex-rs/cloud-client/src/lib.rs:32 |

## Evolution
- 2026-08-13 `363427b5e3` (#38303) interrupted turn recovery.
- 2026-08-22 `4f39251a01` (#40038) unfinished root turn suspension.
- 2026-09-16 `f2b5b81f39` (#45820) continue interrupted work after managed daemon restarts.

## Versus pi
- [[pi--durable-execution]]: pi-durable checkpoints every generation/tool step as a task state machine with per-tool replay policy (`safe|unsafe`); codex resumes at turn granularity, re-sampling from persisted completed items — tools interrupted mid-run are not replayed, they surface as aborted outputs ([[codex--partial-message-persistence|partial-message-persistence]], [[crash-safe-tool-replay]]).
