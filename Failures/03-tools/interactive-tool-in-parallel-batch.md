---
type: failure
concepts: [parallel-tool-execution]
harnesses: [pi]
---
**Symptom** — When the model called the example `question` tool (asks the human) several times in one turn, the calls ran in parallel and the questions became unanswerable (overlapping dialogs).

**Root cause** — Parallel is the default execution mode; tools that need exclusive access to the human (or shared UI state) had not opted out.

**Fix · [[pi]]** — `ec857fece` 2026-07-02 (#6189): `executionMode: "sequential"` on the question example tool (`packages/coding-agent/examples/extensions/question.ts:50`); a sequential tool serializes its whole batch (`packages/agent/src/agent-loop.ts:515-523`).

**Lesson** — Tools that need the human or shared mutable state must opt out of parallel execution.

Related: [[parallel-tool-execution]] · [[plugin-tools]] · [[pi--parallel-tool-execution|pi]]
