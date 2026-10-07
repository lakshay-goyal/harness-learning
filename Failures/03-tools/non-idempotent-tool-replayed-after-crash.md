---
type: failure
concepts: [crash-safe-tool-replay, task-owned-subagent]
harnesses: [pi]
---
**Symptom** — (durable) After a crash and resume, the persistent/background subagent example tool was re-run; a repeated "stop" could stop **newer** work, answers could be reported more than once, and scenes could hang polling.

**Root cause** — The tool was treated as replay-safe although its control operations (stop/wait) are not idempotent with respect to work started after the crash.

**Fix · [[pi]]** — `37c9d0d20` 2026-09-30 "harden the persistent subagent example": the tool no longer offers wait, "is not rerun after a crash (a repeated stop could stop newer work)", reports each answer once, checks registry names as own properties. Recovery rule: rerun only when both stored and current policy are `safe`, else "Tool X was interrupted and may have partially run" (`packages/durable/src/harness/tool.ts:93-110`).

**Lesson** — Declare replay-safety per tool honestly; control operations (stop/kill) are not idempotent even if the tool "just" manages state.

Related: [[crash-safe-tool-replay]] · [[task-owned-subagent]] · [[durable-execution]] · [[pi--crash-safe-tool-replay|pi]]
