---
type: failure
concepts: [process-tree-kill, remote-execution-env]
harnesses: [pi]
---
**Symptom** — (Inferred from the fix comment; not recorded as an observed bug — unverified.) A large frame (up to 16 MiB) arriving slowly over a weak link could look like 30 s of silence to the remote daemon's dead-man switch, which would then kill every process group it started and exit mid-transfer.

**Root cause** — Liveness measured per complete frame/ping rather than per received byte; on slow links a single large frame outlasts the silence window.

**Fix · [[pi]]** — `46d0ff936` 2026-10-05 "daemon robustness": `Seen<R>` stdin wrapper stamps `last_seen` on every non-empty read — "so a large frame arriving slowly counts as a live client" (`packages/env/daemon/src/main.rs:313-327`); dead-man switch kills all groups only after `SILENCE_LIMIT_MS` 30 000 with no bytes (`main.rs:34,464-474`). Same commit put pings/cancel replies on a control queue ahead of bulk output (`packages/env/daemon/src/output.rs:1-4,71-99`) so the client's liveness view isn't starved either.

**Lesson** — Liveness for a kill-on-silence switch must count bytes received, not frames completed, and control traffic must not queue behind bulk data.

Related: [[process-tree-kill]] · [[remote-execution-env]] · [[pi--process-tree-kill|pi]]
