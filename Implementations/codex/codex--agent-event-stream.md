---
type: implementation
harness: codex
concept: agent-event-stream
commit: 622e9e3696
files: [codex-rs/core/src/session/mod.rs:515, codex-rs/core/src/session/mod.rs:603, codex-rs/core/src/session/submission.rs:8, codex-rs/core/src/session/handlers.rs:439, codex-rs/core/src/session/mod.rs:1053, codex-rs/core/src/codex_thread.rs:258, codex-rs/core/src/thread_manager.rs:126, codex-rs/core/src/tasks/mod.rs:982, codex-rs/exec/src/exec_events.rs:13]
---
[[agent-event-stream]] in [[codex]].

Three layers: core **SQ/EQ** (submission queue / event queue) per thread → app-server notifications (slash-named, typed items + deltas) → `codex exec --json` JSONL (dot-named `ThreadEvent`).

## Mechanism
- **Core SQ/EQ**: each thread owns a bounded submission channel `SUBMISSION_CHANNEL_CAPACITY = 512` and an UNBOUNDED event channel (`codex-rs/core/src/session/mod.rs:515`, `:603-604`). One `submission_loop` task per session processes `Submission{id, op, trace, parent_turn_id, root_turn_id, residency_guard}` (`codex-rs/core/src/session/submission.rs:8-20`) strictly in order; turns are spawned as tasks so the loop stays responsive to `Op::Interrupt`, approvals etc. (`codex-rs/core/src/session/handlers.rs:439-720`). The loop `select!`s (biased) on shutdown token, submissions and the agent-mailbox watch (`:450-479`).
- **Ops** (`codex-rs/core/src/session/handlers.rs:492-720`): Interrupt, InterruptIfNoPendingInput, CleanBackgroundTerminals, Realtime*, TurnInput, RecoverTurn, SuspendTurnAndShutdown, ThreadSettings, TurnSettings, InterAgentCommunication, ExecApproval, PatchApproval, UserInputAnswer, RequestPermissionsResponse, DynamicToolResponse, RefreshMcpServers, ReloadUserConfig, Compact, SetThreadMemoryMode, RunUserShellCommand, ResolveElicitation, Shutdown, Review, ApproveGuardianDeniedAction.
- **Ops with replies** (`TurnInput`, `TurnSettings`, `InterruptIfNoPendingInput`, `SuspendTurnAndShutdown`) return a routing decision via oneshot, not the turn result: "Once queued, dropping the waiter does not retract the call. If the session loop exits before replying, the caller gets InternalAgentDied" (`codex-rs/core/src/session/mod.rs:1053-1060`).
- **Conduit**: `CodexThread` (`submit`, `start_or_steer_turn`, `start_turn_if_idle`, `steer_turn`, `recover_turn_if_idle`, `next_event`, `queued_event_count`) (`codex-rs/core/src/codex_thread.rs:258-773`); `ThreadManager` owns all threads, forks, sub-agent spawns (`THREAD_CREATED_CHANNEL_CAPACITY = 1024`, `codex-rs/core/src/thread_manager.rs:126`).
- **Terminal events are barriers**: abort callbacks awaited and rollout flushed before `TurnAborted`, flushed again after (`codex-rs/core/src/tasks/mod.rs:982-1028`); task completion flushes before `on_task_finished` (`:357-374`) → [[listeners-see-stale-agent-state]], [[terminal-event-required]].
- **App-server layer**: thread/turn/item lifecycle notifications with typed per-item deltas (`codex-rs/app-server-protocol/src/protocol/common.rs:1935-2030`); clients may opt out of notification methods at `initialize` → [[codex--client-server-session-split|client-server-session-split]].
- **Exec JSONL layer** (`codex exec --json`): `thread.started`, `turn.started`, `turn.completed`, `turn.failed`, `item.started`, `item.updated`, `item.completed`, `error`; item kinds agent_message, reasoning, command_execution, file_change, mcp_tool_call, collab_tool_call, web_search, todo_list, error (`codex-rs/exec/src/exec_events.rs:13-36`); usage includes reasoning tokens since `ddfa691752` 2026-04-24 → [[codex--headless-rpc-mode|headless-rpc-mode]].
- **Persistence vs transient**: exec deltas, approvals, stream errors, warnings, TurnDiff never persisted (`codex-rs/rollout/src/policy.rs:156-215`).

## Constants
| name | value | path:line |
|---|---|---|
| `SUBMISSION_CHANNEL_CAPACITY` | 512 | codex-rs/core/src/session/mod.rs:515 |
| event channel | unbounded | codex-rs/core/src/session/mod.rs:603-604 |
| `THREAD_CREATED_CHANNEL_CAPACITY` | 1024 | codex-rs/core/src/thread_manager.rs:126 |
| event queue capacity | unbounded (never blocks the loop on slow clients; memory risk). `78932f4493` exposes queued count | `codex-rs/core/src/session/mod.rs:604` |

## Evolution
- 2025-10-28 `5ba2a17576` (#5854) decompose submission loop.
- 2026-04-16 `a1736fcd20` (#18206) turn logic split out of `codex.rs`.
- 2026-04-24 `ddfa691752` exec usage includes reasoning tokens.
- 2026-09-30 `9ef9cb1d9f` (#49475) terminal event after abort callbacks + flush.

## Versus pi
- [[pi--agent-event-stream]]: pi layers `AgentEvent` → `AgentSessionEvent` in-process with awaited plugin listeners; codex separates an input queue of typed Ops (with routing replies) from an output event queue, then re-encodes events twice (app-server slash-names, exec dot-names). Both strip cumulative snapshots on the wire (codex sends item deltas).
