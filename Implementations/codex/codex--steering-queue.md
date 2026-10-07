---
type: implementation
harness: codex
concept: steering-queue
commit: 622e9e3696
files: [codex-rs/core/src/session/turn_input.rs:323, codex-rs/core/src/session/turn_input.rs:814, codex-rs/core/src/session/input_queue.rs:368, codex-rs/core/src/session/turn.rs:428, codex-rs/core/src/session/turn.rs:563, codex-rs/core/src/session/input_queue.rs:293, codex-rs/core/src/session/turn.rs:2634, codex-rs/core/src/session/inject.rs:17]
---
[[steering-queue]] in [[codex]].

## Mechanism
1. **One op for start and steer**: `start_or_steer` tries `steer_input` into the active turn and only starts a new task on `NoActiveTurn` (`codex-rs/core/src/session/turn_input.rs:323-446`). Explicit `TurnInputMode::Steer{expected_turn_id}` rejects with `ExpectedTurnMismatch` if the turn changed (`:814-821`). Rejections: Review and Compact tasks are `ActiveTurnNotSteerable` (`:824-835`); empty input; mismatched final-output JSON schema (`:838-848`). App-server: `turn/steer` (`0d8b2b74c4`).
2. **Enqueue**: steered input appended to `turn_state.pending_input`; `InputQueueActivity::Steer` broadcast on a watch channel (`codex-rs/core/src/session/input_queue.rs:368-379`). Wakes blocking waits: `wait_agent` returns "Wait interrupted by new input." ([[subagent-result-mailbox]]); `clock.sleep` "ends early when new input arrives for the active turn" ([[wall-clock-tools]]).
3. **Injection boundary (default)**: drained only at the top of the next loop iteration, i.e. after the current sampling request AND all its tool calls completed (`codex-rs/core/src/session/turn.rs:428-436`; tools always drained `:3156-3165`) — pending tool calls are never skipped.
4. **Continue forcing**: pending input forces `needs_follow_up` so the turn continues even if the model produced a final answer (`codex-rs/core/src/session/turn.rs:563`).
5. **Hooks**: steered input runs through UserPromptSubmit hooks with `PersistContext::SteeredUserInput`; a blocking hook breaks the loop (`codex-rs/core/src/session/turn.rs:438-447`) → [[turn-lifecycle-hooks]].
6. **Deferral rules**: not drained at the first iteration (fresh input samples first) nor right after a mid-turn auto-compact while a model/tool continuation is pending (`codex-rs/core/src/session/turn.rs:413-417`, `:604-611`; `8f705b0702`) → [[side-phase-input-lost]].
7. **Instant preemption** (opt-in `Feature::InstantInterrupt` key `instant_interrupt`, `Stage::UnderDevelopment`, default off, `codex-rs/features/src/lib.rs:1166-1171`): each `StepContext` gets a `preempt` token (`codex-rs/core/src/session/mod.rs:3867-3871`); `watch_user_input` spawns a watcher that cancels it when user input lands (`codex-rs/core/src/session/input_queue.rs:293-313`). On preempt the stream either sends `response.interrupt` (Responses Lite) and drains the response to keep the WebSocket continuation, or drops the stream and returns `needs_follow_up: true` (`codex-rs/core/src/session/turn.rs:2634-2652`); retry backoff also preempted (`:1705-1729`).
8. **Programmatic injection**: `inject_if_running` pushes response items into the active turn's pending input (same drain point) or returns them if idle; `inject_no_new_turn` records into history without starting a turn when idle (`codex-rs/core/src/session/inject.rs:17-43`, `:170-188`). Used by goal steering (`8f6a945ec9`) and async hook context → [[persistent-goal-continuation]].
9. **Mailbox shares the queue**: inter-agent mail is drained with steers; queue-only mail defers to the next turn after a final answer (`MailboxDeliveryPhase`) → [[follow-up-queue]], [[subagent-result-mailbox]].
10. **TUI**: Enter = steer into the running turn, Tab = queue until idle (`cbca43d57a`); during manual `/compact` follow-ups are queued (`e838645fa2`).
11. **Interrupt cleanup**: on interrupt, un-drained steers are dropped — `clear_pending` clears waiters and `pending_input.items` after tasks observe cancellation (`codex-rs/core/src/tasks/mod.rs:620-622`; `codex-rs/core/src/session/input_queue.rs:316-320`); leftover input at task end recorded into history (through hooks) rather than dropped (`codex-rs/core/src/tasks/mod.rs:672-697`); suspension drops in-process pending input ("Handoff intentionally drops that state", `codex-rs/core/src/session/turn_suspension.rs:1-119`).

## Constants
| name | value | path:line |
|---|---|---|
| `instant_interrupt` feature | UnderDevelopment, default off | `codex-rs/features/src/lib.rs:1166-1171` |
| steer delivery batching | all pending input drained at once at the loop top (unverified: no one-at-a-time mode seen) | `codex-rs/core/src/session/turn.rs:428-436` |

## Evolution
- 2026-01-12 `cbca43d57a` "Send message by default mid turn. queue messages by tab (#9077)".
- 2026-02-05 `0d8b2b74c4` "feat(app-server): turn/steer API (#10821)".
- 2026-03-23 `e838645fa2` "tui: queue follow-ups during manual /compact (#15259)".
- 2026-04-09 `8f705b0702` "Defer steering until after sampling the model post-compaction (#17163)".
- 2026-05-18 `afa0101ae2` "Move pending input into input queue (#22728)".
- 2026-05-29 `8f6a945ec9` "Use inject_if_running for active goal steering (#24924)".
- 2026-06-15 `ee40dddbf6` steer interrupts `wait_agent` → [[blocking-wait-ignores-user-steer]].
- 2026-09-25 `f92655d07f` "Preempt model responses when new user input arrives (#48141)"; 2026-09-26 `12de0e395d` "Preserve WebSocket continuations when steering a turn (#48508)".

## Quirks
- Steering is impossible into Review/Compact tasks ([[no-steer-into-review-or-compact]]); the client has to queue.
- Steers bypass nothing: they still pass UserPromptSubmit hooks, which may stop the turn.

## Versus pi
- [[pi--steering-queue]]: both wait for the whole tool batch (codex never skipped pending calls; cf. pi's [[steering-skips-pending-tool-calls]]). pi one-at-a-time default; codex drains all pending input per boundary.
- codex adds opt-in mid-stream preemption (`InstantInterrupt`) and steer-wakes for blocking tools; pi has a separate abort that returns queued text to the editor.
- codex fuses start and steer in one op with optional turn-id guard; pi splits `steer()`/`followUp()`.
