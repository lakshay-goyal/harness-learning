---
type: failure
concepts: [abort-propagation, shell-execution]
harnesses: [codex]
---
**Symptom** — "Interrupting a running turn (Ctrl+C / Esc) currently also terminates long-running background shells, which is surprising for workflows like local dev servers or file watchers" (`ba463a9dc7` body).

**Root cause** — "Stop the agent" and "kill the agent's long-lived processes" were one operation.

**Fix · [[codex]]**
- `ba463a9dc7` 2026-03-15 "Preserve background terminals on interrupt and rename cleanup command to /stop (#14602)": interrupt leaves unified-exec processes running; explicit `/stop` → `Op::CleanBackgroundTerminals` (`codex-rs/core/src/session/handlers.rs:501-504`, `codex-rs/core/src/tasks/mod.rs:907-912`).
- The interrupt marker tells the model: "Any running unified exec processes may still be running in the background." (`codex-rs/core/src/context/turn_aborted.rs:10`).

**Lesson** — Separate "stop the agent" from "kill its long-lived processes", and tell the model which happened.

Related: [[abort-propagation]] · [[shell-execution]] · [[process-tree-kill]] · [[interrupted-turn-invisible-to-model]] · [[codex--abort-propagation|codex]]
