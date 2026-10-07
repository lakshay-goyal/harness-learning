---
type: failure
concepts: [crash-safe-tool-replay, task-owned-subagent]
harnesses: [pi, opencode]
---
**Symptom** — (durable) After a crash and resume, the persistent/background subagent example tool was re-run; a repeated "stop" could stop **newer** work, answers could be reported more than once, and scenes could hang polling.

**Root cause** — The tool was treated as replay-safe although its control operations (stop/wait) are not idempotent with respect to work started after the crash.

**Fix · [[pi]]** — `37c9d0d20` 2026-09-30 "harden the persistent subagent example": the tool no longer offers wait, "is not rerun after a crash (a repeated stop could stop newer work)", reports each answer once, checks registry names as own properties. Recovery rule: rerun only when both stored and current policy are `safe`, else "Tool X was interrupted and may have partially run" (`packages/durable/src/harness/tool.ts:93-110`).

**Fix · [[opencode]]** (v2, by design; no per-tool policy) — before assembling a provider request the runner durably fails every tool still `pending`/`running` from a previous process with `Tool execution interrupted` (`packages/core/src/session/runner/llm.ts:119-139`); "abandoned side effects are never silently replayed" (`specs/v2/session.md:50`). Idempotency-gated automatic retry is only a TODO (`specs/v2/todo.md:56-74`). Simpler than pi's `safe` policy: nothing is ever rerun, so safe tools also lose their result. See [[opencode--crash-safe-tool-replay]].

**Lesson** — Declare replay-safety per tool honestly; control operations (stop/kill) are not idempotent even if the tool "just" manages state.

Related: [[crash-safe-tool-replay]] · [[task-owned-subagent]] · [[durable-execution]] · [[pi--crash-safe-tool-replay|pi]]
