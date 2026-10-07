---
type: implementation
harness: opencode
concept: partial-message-persistence
commit: ecc4916b5a
files: [packages/opencode/src/session/session.ts:877-885, packages/opencode/src/session/processor.ts:553-600, packages/opencode/src/session/message-v2.ts:258-266, packages/opencode/src/session/message-v2.ts:362-373, packages/core/src/session/runner/publish-llm-event.ts:91-119, packages/core/src/session/runner/publish-llm-event.ts:195-211, packages/schema/src/session-event.ts:209-210]
---
[[partial-message-persistence]] in [[opencode]].

## Mechanism

### Legacy runtime — start/end writes, deltas never stored
- Text/reasoning deltas go through `updatePartDelta`, which only publishes a `PartDelta` event for live UI — no row write (`packages/opencode/src/session/session.ts:877-885`).
- A part is written at start (empty), at end (full), and by `cleanup()` on abort/error with whatever accumulated: open text and reasoning parts closed with an end time, pending tool calls awaited ≤ 250 ms then finalized, and a final snapshot `patch` part recorded (`packages/opencode/src/session/processor.ts:553-600`). A hard crash (not abort) loses streamed text since the last full write (inference).
- **Replay policy** (`packages/opencode/src/session/message-v2.ts:258-266`): assistant messages with an error are skipped, except `AbortedError` messages that contain a part other than `step-start`/`reasoning` — aborted partial answers are replayed, failed ones dropped.
- Pending/running tool parts in a replayed message become `output-error` "[Tool execution was interrupted]" so every tool_use is answered (`packages/opencode/src/session/message-v2.ts:362-373`).

### v2 runtime — full-value checkpoints
- Deltas are buffered in memory per part id and published once as a durable `Text.Ended` / `Reasoning.Ended` / `Tool.Input.Ended` full value; buffers are flushed on stream end or interruption (`packages/core/src/session/runner/publish-llm-event.ts:91-119`, `packages/core/src/session/runner/publish-llm-event.ts:195-197`). "Stream fragments are live-only; Text.Ended is the replayable full-value boundary" (`packages/schema/src/session-event.ts:209`).
- Protocol violations (delta before start, duplicate start, end before start) are defects, not repairs (`packages/core/src/session/runner/publish-llm-event.ts:96-111`).
- Failed assistant → durable `Step.Failed` after flushing (`packages/core/src/session/runner/publish-llm-event.ts:199-211`); unsettled tools failed with "Tool execution interrupted" / "Tool execution failed: …" ([[opencode--tool-output-spill]]).

## Constants
| name | value | path:line |
|---|---|---|
| tool-call settle wait on cleanup | 250 ms | `packages/opencode/src/session/processor.ts:587` |

## Evolution
- 2025-06-14 `783faf554d` continuing a session after abort.
- 2025-09-17 `ff6a93f355` (#2651) keep aborted messages only with sufficient parts → [[failed-turns-replayed]].
- 2025-12-22 `d4b7f75ce3` snapshot/patch also when `finish-step` never arrives.
- 2026-06-21 `fb43c15f88` v2 event model simplified around `*.Ended` checkpoints.

## Quirks / drift
- Legacy replays aborted reasoning text but `differentModel` strips provider metadata, so cross-model replays lose signatures by design.

Contrast: [[pi--partial-message-persistence|pi]] persists the final partial with `stopReason: aborted|error` and skips it at the provider boundary; opencode legacy writes start/end/cleanup snapshots and replays aborted-but-substantive turns, v2 checkpoints only full values.
