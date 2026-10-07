---
type: failure
concepts: [parallel-tool-execution]
harnesses: [pi]
---
**Symptom** — With parallel tools, UI rows of fast tools stayed "pending" until the slowest tool in the batch finished; attempts to emit eagerly risked persisting tool-result messages out of the assistant's call order.

**Root cause** — Completion events and transcript artifacts were coupled: one ordering served both UI progress and persisted history.

**Fix · [[pi]]** — `759d55152` 2026-04-22 (#3503): emit `tool_execution_end` eagerly in completion order, but append persisted tool-result messages after all finish in assistant **source order** (`packages/agent/src/agent-loop.ts:586-660`; contract `packages/agent/src/types.ts:36-47`).

**Lesson** — Decouple completion events (any order) from transcript order (call order).

Related: [[parallel-tool-execution]] · [[turn-loop]] · [[pi--parallel-tool-execution|pi]]
