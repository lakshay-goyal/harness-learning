---
type: failure
concepts: [task-owned-subagent, abort-propagation]
harnesses: [pi, opencode]
---
**Symptom** — (pi-durable, Package 18) Cancellation through the task/conversation ownership tree had race windows, pinned by the added tests: cascade from an owner the scheduler **orphans**; a child whose dependency completes in the same commit that marks its owner; abort marks found through an ownership edge loaded after reopen when their commit is rejected; a cancelled caller's wait cancelling shared work; subagent handle operations queued on the session line after the invocation ended.

**Root cause** — Abort intent must propagate across commits, reopen and caller/owner boundaries; cascades computed only at the triggering moment miss work that appears or settles in the same or later commits (inferred from test names; commit has no body).

**Fix · [[pi]]** — `1b347794e` 2026-09-29 "cover ownership cancellation races (Package 18)" (+209 lines `packages/durable/test/harness-ownership.test.ts`), on top of `2532a0bef` (conversation abort, ownership cascades, subagent handles). Rule at HEAD: the scheduler derives abort marks in a separate reconcile commit, re-run at open (`packages/durable/docs/spec.md:1990-1998`); abort flows down, held `completing` outcomes, bottom-up abort order (`spec.md:2089-2176`); structured concurrency formalized in `03180653c` 2026-09-30 (Package 19).

**Fix · [[opencode]]** (legacy runtime; a run of point fixes rather than one reconcile step)
- `139c4fd555` 2026-04-27 (#24553) "harden shell cancellation": a cancel arriving while a shell was still starting could miss it → `ready` latch + `cancelled` deferred; `stopShell` waits for ready, signals cancelled, then interrupts the fiber (`packages/opencode/src/effect/runner.ts:108-113`).
- `75d141b574` 2026-05-04 (#25798) "cancel subtask child sessions": `SessionPrompt.cancel` had been forked fire-and-forget and the `task` tool removed its abort listener without cancelling the child on interruption → parent abort left the child session running. Now the tool's release cancels the child (and, at HEAD, its background jobs) when the exit has interrupts (`packages/opencode/src/tool/task.ts:347-357`).
- `822eec0d62` 2026-05-12 (#27115) "Fix runner cancel completion": cancel *awaited* the run's `done` deferred instead of failing it, so cancel hung until the run finished on its own → `Deferred.fail(st.run.done, new Cancelled())` (`packages/opencode/src/effect/runner.ts:179,196`).
- `2a33addd29` 2026-06-03 (#30641) "avoid shell cancel race": `SessionRunState.cancel` returned early when the runner existed but was not yet `busy`, dropping a cancel issued during the start transition → only a missing runner short-circuits (`packages/opencode/src/session/run-state.ts:77-86`).

**Lesson** — Derive abort cascades from committed state in a dedicated reconcile step that also runs at reopen; never let the triggering transaction apply them ad hoc.

Related: [[task-owned-subagent]] · [[abort-propagation]] · [[durable-execution]] · [[pi--task-owned-subagent|pi]] · [[opencode--task-owned-subagent|opencode]] · [[opencode--abort-propagation|opencode abort]]
