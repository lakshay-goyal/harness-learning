---
type: absence
harnesses: [codex]
---
# no-csv-fanout-jobs

Batch fan-out (`spawn_agents_on_csv` / `report_agent_job_result`) was removed.

**What's missing**
- `enable_fanout` kept as a `Stage::Removed` no-op (`codex-rs/features/src/lib.rs:1466`).

**Evidence of decision**
- Added `dcab40123f` 2026-02-24 (agent jobs); removed 2026-07-20 `687f05cb94`.

**Implication**
- Fan-out is expressed through ordinary `spawn_agent` calls under concurrency caps ([[subagent-concurrency-limits]]); cloud best-of-N covers parallel attempts ([[cloud-task-delegation]]).

Related: [[in-process-subagent-threads]] · [[subagent-concurrency-limits]] · [[feature-flag-stages]] · [[Absences]]
