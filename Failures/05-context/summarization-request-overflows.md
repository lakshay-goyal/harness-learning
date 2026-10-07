---
type: failure
concepts: [transcript-serialization-for-summary, auto-compaction]
harnesses: [pi, opencode]
---
**Symptom** — The compaction request itself exceeded the context window, so compaction failed exactly when it was needed.

**Root cause** — Full tool outputs were serialized into the summarizer prompt. `2add465fb` (2025-12-29) originally sliced tool args to 100 chars and results to 1000, but `17ce3814a` (2025-12-30) removed truncation ("already token-budgeted") — wrong, because the summarizer has the same window and the summarized range can be almost the whole window.

**Fix · [[pi]]** — `c950c692a` 2026-03-06 (#1796): tool results truncated to `TOOL_RESULT_MAX_CHARS = 2000` + `[... N more characters truncated]` (`packages/coding-agent/src/core/compaction/utils.ts:94-104`, `149`); mirrored in durable (`packages/durable/src/harness/compaction.ts:55`). 0.74.1 also clamped summary output tokens (`3d9e14d74`, see [[summary-output-budget-misfit]]).

**Fix · [[opencode]]** `0fd6f365be` 2026-02-10 (#12924) 20k reserve "to ensure that input window has enough room to compact" (`packages/opencode/src/session/overflow.ts:8`); `574b2c2170` 2026-04-22 (#23870) `TOOL_OUTPUT_MAX_CHARS = 2_000` + `[truncated]` in the summarizer input (`packages/opencode/src/session/compaction.ts:30`, `:51-52`); media stripped from overflow compactions (`be20f865ac`). Legacy terminal guard: compaction overflow → "Session too large to compact" and stop (`packages/opencode/src/session/compaction.ts:450-459`). v2 pre-checks `Token.estimate(summaryPrompt) > context − summaryOutput` and skips (`packages/core/src/session/compaction.ts:189-190`).

**Lesson** — The summarizer shares the agent's window: compress its input before sending (cap per-tool-result text), and never assume the summarized range is "already budgeted".

Related: [[transcript-serialization-for-summary]] · [[auto-compaction]] · [[tool-output-truncation]]
