---
type: implementation
harness: opencode
concept: follow-up-queue
commit: ecc4916b5a
files: [packages/core/src/session/input.ts:268-288, packages/core/src/session/runner/llm.ts:392-416, packages/core/src/session/run-coordinator.ts:51-92, packages/opencode/src/session/prompt.ts:1106-1115]
---
[[follow-up-queue]] in [[opencode]].

## Mechanism

### Legacy runtime
- No separate follow-up queue. A message persisted while the final text step streams still fails the exit test (`lastAssistant.parentID !== lastUser.id`, `packages/opencode/src/session/prompt.ts:1115`), so it gets another step — follow-up semantics for free.
- User `!cmd` shell runs through `SessionPrompt.shell()`; since `def907ae4b` it triggers the loop if messages are queued.

### v2 runtime
- `delivery: "queue"` rows are promoted **one FIFO row at a time** only when the session would otherwise go idle (`packages/core/src/session/input.ts:268-288`).
- Outer loop of `SessionRunner.run`: after the inner provider-turn loop settles, `hasPending(queue)` → promote one and restart with `step = 1`; queue promotion also promotes eligible steers (`packages/core/src/session/runner/llm.ts:191-194,412-413`).
- Coordinator `wake` coalesces: an active drain gets `pendingWake = true`, and a successful drain with `pendingWake` starts a successor drain (`packages/core/src/session/run-coordinator.ts:51-56,81-92`).
- Durable inbox rows survive interrupt (`specs/v2/session.md:22-27`).

## Constants
| name | value | path:line |
|---|---|---|
| v2 queue promotion batch | 1 row | `packages/core/src/session/input.ts:268-288` |

## Evolution
- 2026-02-05 `a45841396f` aborting with queued messages rejected their callbacks → unhandled errors.
- 2026-02-07 `def907ae4b` `SessionPrompt.shell()` triggers the loop if messages are queued.
- 2026-06-08 `0a7cb20e66` `opencode run` exited before the event loop drained.
- 2026-06-04 `76ecf2e58c` v2 durable inbox.

## Quirks / drift
- Legacy cannot distinguish steer from follow-up: both are "the newest user message".

Failures: [[queued-messages-stranded-at-run-end]].

Contrast: [[pi--follow-up-queue|pi]] has a second in-memory queue drained at natural stop; opencode v2 makes it a durable FIFO inbox with one-at-a-time promotion.
