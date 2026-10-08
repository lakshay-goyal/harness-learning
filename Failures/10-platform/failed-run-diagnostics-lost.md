---
type: failure
concepts: [harness-evals]
harnesses: [pi]
---
**Symptom** — When `dispose` or temp-dir removal threw after an eval run, the run's usage and transcript were lost; post-run validation failures also discarded partial usage.

**Root cause** — Cleanup errors replaced the run result; diagnostics lived only in the temp dir.

**Fix · [[pi]]** — `b0864a66c` 2026-07-27 preserve diagnostics on cleanup failure (AggregateError); `5a3a03a7f` 2026-09-17 (#9706) `attachHarnessRunToError` with partial usage (`packages/evals/src/harness.ts:448-466`); session JSONL snapshotted as artifact `piSessionJsonl` before temp dir deletion (`harness.ts:420-431`; `report.ts:9`).

**Lesson** — Always surface partial telemetry from failed runs; snapshot artifacts before cleanup.

Related: [[harness-evals]] · [[pi--harness-evals|pi]]
