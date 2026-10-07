---
type: failure
concepts: [transcript-serialization-for-summary, auto-compaction]
harnesses: [pi]
---
**Symptom** — The compaction request itself exceeded the context window, so compaction failed exactly when it was needed.

**Root cause** — Full tool outputs were serialized into the summarizer prompt. `2add465fb` (2025-12-29) originally sliced tool args to 100 chars and results to 1000, but `17ce3814a` (2025-12-30) removed truncation ("already token-budgeted") — wrong, because the summarizer has the same window and the summarized range can be almost the whole window.

**Fix · [[pi]]** — `c950c692a` 2026-03-06 (#1796): tool results truncated to `TOOL_RESULT_MAX_CHARS = 2000` + `[... N more characters truncated]` (`packages/coding-agent/src/core/compaction/utils.ts:94-104`, `149`); mirrored in durable (`packages/durable/src/harness/compaction.ts:55`). 0.74.1 also clamped summary output tokens (`3d9e14d74`, see [[summary-output-budget-misfit]]).

**Lesson** — The summarizer shares the agent's window: compress its input before sending (cap per-tool-result text), and never assume the summarized range is "already budgeted".

Related: [[transcript-serialization-for-summary]] · [[auto-compaction]] · [[tool-output-truncation]]
