---
type: implementation
harness: opencode
concept: run-settlement
commit: ecc4916b5a
files: [packages/opencode/src/session/run-state.ts:37-110, packages/opencode/src/effect/runner.ts:37-213, packages/opencode/src/session/prompt.ts:1329-1332, packages/core/src/session/run-coordinator.ts:51-104, packages/core/src/session/runner/llm.ts:286-355]
---
[[run-settlement]] in [[opencode]].

## Mechanism

### Legacy runtime
- One `Runner` per session in an `InstanceState` map, states `Idle | Running | Shell | ShellThenRun` (`packages/opencode/src/effect/runner.ts:37-213`; `packages/opencode/src/session/run-state.ts:37-64`).
- `ensureRunning` while `Running` → join the in-flight run's `Deferred` (no second loop); while `Shell` (user `!cmd`) → `ShellThenRun`, agent starts after the shell ends (`run-state.ts:88-110`).
- `startShell` while not idle → `RunnerBusy` → `Session.BusyError`; revert/unrevert call `assertNotBusy` (`packages/opencode/src/session/revert.ts:39,93`).
- `onIdle` deletes the runner and sets `SessionStatus` idle; status values `busy | idle | retry` (retry carries `attempt`, `next`).
- Interrupted run resolves joiners with the last assistant message (`onInterrupt`), not an error.
- Interrupt path finalizes the record: `finalizeInterruptedAssistant` sets `AbortedError` + `time.completed` when the processor never got that far (`packages/opencode/src/session/prompt.ts:1329-1332`); processor `cleanup()` closes open parts and sets `time.completed` (`packages/opencode/src/session/processor.ts:553-611`).
- Side work forked into the layer scope, not the run fiber: title (step 1), diff summary, post-run prune (`prompt.ts:1133-1139,1252-1253,1338`) — settlement does not wait for them.

### v2 runtime
- `SessionRunCoordinator`: process-global map `Key → {done Deferred, owner fiber, pendingWake, stopping}`; `run` joins or starts a forced drain, `wake` sets `pendingWake` or starts an unforced drain; a successful drain with `pendingWake` spawns a successor (`packages/core/src/session/run-coordinator.ts:51-92`). Different sessions run concurrently.
- Turn settlement runs inside `Effect.uninterruptibleMask`: flush buffered text/reasoning, fail unsettled tools, fail the active assistant step, write `Step.Ended` only if a `step-finish` arrived (`packages/core/src/session/runner/llm.ts:286-355`).
- Drain ends on: no local tool call and no steer/queue; provider error; step limit; user decline (`Effect.interrupt`, `llm.ts:306-310`); location moved (`llm.ts:180-181`); second overflow after overflow compaction (defect, `llm.ts:370`); external interrupt.
- No durable "busy/retrying/idle/interrupted" status yet (`llm.ts:52` unchecked); `sessions.active()` is the in-memory coordinator registry, empty after restart (`specs/v2/session.md:169`).

## Constants
| name | value | path:line |
|---|---|---|
| `SessionStatus` types | `busy`, `idle`, `retry` | `packages/opencode/src/session/processor.ts:678-685` |

## Evolution
- 2025-08-13 `796bc390db` session stuck in "Working..."; 2025-09-29 `c398485213` TUI stuck "generating".
- 2026-02-01 `6c9b2c37a5` new sessions allowed after errors (stuck busy status).
- 2026-05-12 `822eec0d62` cancel failed the run's `done` instead of awaiting it.
- 2026-05-14 `e76cf967e6` finalize interrupted assistant messages; 2026-06-21 `49593c1ec4` v2 settles interrupted assistant steps.
- 2026-06-22 `fe840d42b8` v2 run coordination simplified (−1378 lines; durable interrupt event removed).

## Quirks / drift
- v2 settlement does not fail a stream that ended without `step-finish` → [[opencode--terminal-event-required|terminal-event-required]].
- Settlement and "idle" are separate from title/summary side calls, which can still be running after idle ([[opencode--auxiliary-model-calls|auxiliary-model-calls]]).

Failures: [[interrupted-message-never-finalized]] · [[queued-messages-stranded-at-run-end]].

Contrast: [[pi--run-settlement|pi]] emits an explicit `agent_settled` after a post-run driver; opencode's settle signal is the runner going `Idle` (legacy) or the coordinator entry being deleted (v2).
