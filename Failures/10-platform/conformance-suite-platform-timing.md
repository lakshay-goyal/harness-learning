---
type: failure
concepts: [harness-evals, remote-execution-env]
harnesses: [pi]
---
**Symptom** — Shared env conformance cases timed out under the runner's default timeout (watch cases wait for polling latency) and were slow / left processes behind on Windows.

**Root cause** — Portable suites run against every backend and platform (local Node, remote daemon, SSH, Windows) whose latencies differ; cases relied on the host runner's timing.

**Fix · [[pi]]** — `be882f3fa` 2026-10-04: per-case `timeoutMs` (30 000 ms for watch cases; steps wait up to 3 s) (`packages/durable/src/testing/env-conformance.ts:107-108`; `packages/durable/src/testing/runner.ts:36`); `acfc60198` 2026-10-04: `sleep 5` → `sleep 2`, cleanup waits for killed Windows commands.

**Lesson** — A portable conformance case must carry its own timing budget and clean up on every platform.

Related: [[harness-evals]] · [[remote-execution-env]] · [[remote-errno-parity-drift]] · [[pi--harness-evals|pi]]
