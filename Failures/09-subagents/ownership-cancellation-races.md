---
type: failure
concepts: [task-owned-subagent, abort-propagation]
harnesses: [pi]
---
**Symptom** — (pi-durable, Package 18) Cancellation through the task/conversation ownership tree had race windows, pinned by the added tests: cascade from an owner the scheduler **orphans**; a child whose dependency completes in the same commit that marks its owner; abort marks found through an ownership edge loaded after reopen when their commit is rejected; a cancelled caller's wait cancelling shared work; subagent handle operations queued on the session line after the invocation ended.

**Root cause** — Abort intent must propagate across commits, reopen and caller/owner boundaries; cascades computed only at the triggering moment miss work that appears or settles in the same or later commits (inferred from test names; commit has no body).

**Fix · [[pi]]** — `1b347794e` 2026-09-29 "cover ownership cancellation races (Package 18)" (+209 lines `packages/durable/test/harness-ownership.test.ts`), on top of `2532a0bef` (conversation abort, ownership cascades, subagent handles). Rule at HEAD: the scheduler derives abort marks in a separate reconcile commit, re-run at open (`packages/durable/docs/spec.md:1990-1998`); abort flows down, held `completing` outcomes, bottom-up abort order (`spec.md:2089-2176`); structured concurrency formalized in `03180653c` 2026-09-30 (Package 19).

**Lesson** — Derive abort cascades from committed state in a dedicated reconcile step that also runs at reopen; never let the triggering transaction apply them ad hoc.

Related: [[task-owned-subagent]] · [[abort-propagation]] · [[durable-execution]] · [[pi--task-owned-subagent|pi]]
