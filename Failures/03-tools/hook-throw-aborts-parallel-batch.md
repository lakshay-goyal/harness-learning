---
type: failure
concepts: [tool-error-as-result, parallel-tool-execution, tool-result-rewriting]
harnesses: [pi]
---
**Symptom** — An extension's `afterToolCall`/`tool_result` hook that threw aborted the **whole parallel tool batch**, losing sibling results.

**Root cause** — The hook ran inside the per-tool finalization awaited by `Promise.all`; its exception propagated out and rejected the batch.

**Fix · [[pi]]** — `b9cd557d1` 2026-04-16 (#3084): hook errors become an error tool result for that call in `finalizeExecutedToolCall` (`packages/agent/src/agent-loop.ts:898-901`); `runToolCall` "never rejects for tool failures" (`:802-810`).

**Lesson** — Every failure inside one tool's lifecycle — including pre/post hooks — must collapse into that tool's result, never the batch.

Related: [[tool-error-as-result]] · [[parallel-tool-execution]] · [[tool-result-rewriting]] · [[pi--tool-error-as-result|pi]]
