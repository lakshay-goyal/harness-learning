---
type: failure
concepts: [durable-execution, crash-safe-tool-replay]
harnesses: [pi]
---
**Symptom** — In the experimental agent-core harness (removed in `7fd478a2e`), a tool call whose result was already committed as `outcome_ready` but not yet materialized into the transcript (results are placed in source order) dropped out of the lane snapshot's `runningTools`, so observers briefly saw the call disappear. `tool_end` was also emitted *before* the `outcome_ready` staging commit, so it was "observation, not proof of durability" (`git show e26afb63a -- packages/agent/docs/tool-durability.md`).

**Root cause** — The snapshot modelled only running vs. placed states; the interval "settled but not yet placed" had no representation, and the lifecycle event preceded the commit it described.

**Fix · [[pi]]** — `e26afb63a` 2026-09-02 "retain settled tools until placement": `LaneSnapshot.operation.runningTools` became a discriminated union keeping `outcome_ready` calls as settled rows until source-ordered materialization; `tool_end` moved after the staging commit ("durable settlement evidence"); synthetic blocked/invalid/cancelled outcomes emit `tool_start`+`tool_end` from the staging commit (`e26afb63a:packages/agent/docs/tool-durability.md`; touched `packages/agent/src/harness/runtime/drive/tools.ts`, `runtime/lane.ts`; historical — file tree removed in `7fd478a2e`).

**Lesson** — When completion order differs from placement order, give "settled, not yet placed" its own observable state, and emit lifecycle events only after the commit they report.

Related: [[durable-execution]] · [[crash-safe-tool-replay]] · [[pi--durable-execution|pi]]
