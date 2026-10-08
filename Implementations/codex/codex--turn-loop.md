---
type: implementation
harness: codex
concept: turn-loop
commit: 622e9e3696
files: [codex-rs/core/src/session/turn.rs:150, codex-rs/core/src/session/turn.rs:424, codex-rs/core/src/session/turn.rs:563, codex-rs/core/src/session/turn.rs:1638, codex-rs/core/src/session/turn.rs:2513, codex-rs/core/src/stream_events_utils.rs:356, codex-rs/core/src/session/turn_input.rs:323, codex-rs/core/src/tasks/regular.rs:37]
---
[[turn-loop]] in [[codex]].

## Mechanism
1. **Entry**: client sends `Op::TurnInput{request, mode, reply}` → `submission_loop` → `turn_input::handle` (`codex-rs/core/src/session/handlers.rs:560`) → `start_or_steer` tries `steer_input` into the active turn; on `NoActiveTurn` builds a TurnContext (`apply_started`) and `session.spawn_task(turn_context, task_input, RegularTask::new())` (`codex-rs/core/src/session/turn_input.rs:323-446`). The reply returns after the start/steer decision, NOT after sampling (`codex-rs/core/src/session/turn_input.rs:1-13`) → [[steering-queue]], [[single-active-task-slot]].
2. **Task wrapper** `RegularTask`: emits TurnStarted inline (first turn doesn't wait on prewarm), runs turn-start extension contributors cancellably, consumes the startup WebSocket prewarm, then loops `run_turn` while pending input remains, stopping if a terminal error was recorded (`codex-rs/core/src/tasks/regular.rs:37-124`) → one task can contain several `run_turn`s.
3. **Loop contract** (doc comment): each sampling request yields function calls (execute and resample) or an assistant message (turn complete) (`codex-rs/core/src/session/turn.rs:150-170`).
4. **Pre-loop phases** in `run_turn`: drain async hook results from previous turn (`codex-rs/core/src/session/turn.rs:177`); pre-sampling compaction BEFORE new input is recorded — on failure the input is still recorded (`codex-rs/core/src/session/turn.rs:184-224`); MCP-server requirement resolution from @mentions, skills/plugin injection; SessionStart hooks (`codex-rs/core/src/session/turn.rs:321`); UserPromptSubmit hooks + record input (`run_hooks_and_record_inputs`, `codex-rs/core/src/session/turn.rs:363-373`); shell-snapshot prewarm only after hooks accept the turn (`codex-rs/core/src/session/turn.rs:375-382`).
5. **Outer loop** (`codex-rs/core/src/session/turn.rs:424-837`), per iteration: (1) drain pending input (steers + mailbox) unless deferred (`:428-436`); (2) UserPromptSubmit hooks on steered input — a blocking hook breaks the loop (`:438-447`); (3) rollout-budget reminder ([[session-token-budget]]), `RecordedTurnInput` marker set (`:450-457`); (4) capture a fresh `StepContext` (tools, MCP binding, settings, AGENTS.md) per sampling request, re-captured when new input arrived (`:460-495`) → [[mid-turn-settings-switch]]; (5) time reminder + world-state diff + reasoning-effort override recorded into history (`:497-518`) → [[current-time-reminder]], [[world-state-diff-injection]]; (6) `run_sampling_request` over `clone_history().for_prompt(...)`.
6. **Continue rule**: `needs_follow_up = model_needs_follow_up || has_pending_input` (`codex-rs/core/src/session/turn.rs:563`). `model_needs_follow_up` set by ANY dispatched tool call (`codex-rs/core/src/stream_events_utils.rs:356`), a `RespondToModel` tool-routing error (`codex-rs/core/src/stream_events_utils.rs:424`), or server `end_turn == Some(false)` on `response.completed` (`codex-rs/core/src/session/turn.rs:2989-2991`). No tool calls + no pending input + `end_turn` not false ⇒ turn ends.
7. **Turn end** (`!needs_follow_up`): Stop hooks (may block → inject continuation prompt as hook-prompt message, set `stop_hook_active`, `continue`) (`codex-rs/core/src/session/turn.rs:622-668`); legacy after-agent hook; optional post-turn compaction; `break` (`:669-725`) → [[turn-lifecycle-hooks]].
8. **Errors**: `TurnAborted` propagates; `InvalidImageRequest` → error event "Invalid image in your last message..." + break; any other error → `emit_turn_error_lifecycle` + `EventMsg::Error`, break "let the user continue the conversation" (`codex-rs/core/src/session/turn.rs:763-834`).
9. **Retry loop** `run_sampling_request` rebuilds the prompt from current history each attempt (`codex-rs/core/src/session/turn.rs:1638-1729`); `ResponsesStreamRetryState` created per call (`:1634`) → [[auto-retry-backoff]].
10. **Stream loop** `try_run_sampling_request` (`codex-rs/core/src/session/turn.rs:2513-3191`): opens `client_session.stream(...)`, polls with `.or_cancel(&preempt).or_cancel(&cancellation_token)` (`:2625-2630`); stream `None` before completion ⇒ `CodexErr::Stream("stream closed before response.completed")` (`:2658-2663`) → [[terminal-event-required]]; `ResponseEvent::Completed` records usage, applies token budget (may error), honours `end_turn`, returns (`:2947-2995`).
11. **Tools mid-stream**: tool futures pushed into a `FuturesOrdered` as `OutputItemDone` tool calls arrive DURING streaming (`codex-rs/core/src/session/turn.rs:2580`, `:2774-2784`); after the stream loop `drain_in_flight` awaits them in model order and records outputs (`:2460-2488`, `:3156-3165`, runs regardless of outcome). A tool future failing during drain → `error_or_panic` ("in-flight tool future failed during drain"), not a turn failure (`:2480-2483`). Every prompt sets `parallel_tool_calls: true` (`:1576`) → [[parallel-tool-execution]].
12. **Progress signals**: token-count event emitted only after tools resolve, so clients don't see progress while e.g. `request_user_input` pauses the turn (`codex-rs/core/src/session/turn.rs:3167-3173`); `TurnDiff` event after each sampling request (`:3179-3188`).

## Constants
| name | value | path:line |
|---|---|---|
| turn/step cap | none (no counter in `loop`) | `codex-rs/core/src/session/turn.rs:424-837` |
| analytics tool-call ids per response | `MAX_ANALYTICS_TOOL_CALL_IDS_PER_RESPONSE = 256` | `codex-rs/core/src/session/turn.rs:2590` |
| async-runtime thread stack | 16 MiB | `codex-rs/async-utils/src/lib.rs:9` |
| websocket connect/startup timeout (bounds prewarm before first turn) | 15 000 ms | `codex-rs/model-provider-info/src/lib.rs:71` |
| `parallel_tool_calls` request flag | always true (Responses Lite excepted) | `codex-rs/core/src/session/turn.rs:1576` |

## Evolution
- 2025-04-16 `59a180ddec` initial TypeScript CLI, loop in `75febbdefa:codex-cli/src/utils/agent/agent-loop.ts`; 2025-04-24 `31d0d7a305` Rust `codex-rs/core` imported; 2025-08-08 `408c7ca142` TypeScript code removed — Rust loop only.
- 2025-10-28 `5ba2a17576` "chore: decompose submission loop (#5854)".
- 2026-03-17 `6ea041032b` TurnStarted emitted immediately, prewarm bounded (#14838) → [[prewarm-blocks-turn-start]].
- 2026-04-16 `a1736fcd20` "[codex] Split codex turn logic (#18206)" — `turn.rs` split out of `codex.rs`.
- 2026-06-22 `3b32d861c5` per-step world-state diffs recorded before every sampling request.
- 2026-08-14 `86b1123ff6` "Enable parallel tool calls for all model prompts (#38499)" — per-model `supports_parallel_tool_calls` removed.
- 2026-08-25 `68301fa45f` per-step settings snapshots; `d7510aa4b4` pending input no longer restarts a terminally failed turn → [[pending-input-restarts-failed-turn]].

## Quirks
- No explicit loop guard; comment: "as long as compaction works well in getting us way below the token limit, we shouldn't worry about being in an infinite loop" (`codex-rs/core/src/session/turn.rs:588`) → [[no-turn-cap]].
- `TurnDiffTracker` spans all sampling steps of one user turn ("from the perspective of the user, it is a single turn") (`codex-rs/core/src/session/turn.rs:404-409`) → [[file-op-tracking]].
- `ModelClientSession` is turn-scoped and reused across retries (caches WebSocket + sticky routing) (`codex-rs/core/src/session/turn.rs:411-412`) → [[session-affinity-cache-routing]].
- Items are recorded into history as each `OutputItemDone` arrives, so a retry after a mid-stream failure continues from partial progress (`codex-rs/core/src/stream_events_utils.rs:347`, `codex-rs/core/src/session/turn.rs:1641-1649`) → [[turn-items-lost-on-abort]].

## Versus pi
- [[pi--turn-loop]]: pi continues on tool presence after the full message; codex starts tools mid-stream and continues on tool dispatch, pending input, or server `end_turn=false`.
- pi 3 loop levels (turn / follow-up / session driver) vs codex 4 (task / turn / retry / stream); pi retries outside the loop, codex retries per sampling request inside it.
- Both: no turn cap; codex adds token-budget and goal breakers as economic stops.
