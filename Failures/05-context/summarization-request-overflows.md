---
type: failure
concepts: [transcript-serialization-for-summary, auto-compaction, overflow-recovery]
harnesses: [pi, opencode, codex]
---
**Symptom** — The compaction request itself exceeded the context window, so compaction failed exactly when it was needed.

**Root cause** — Full tool outputs were serialized into the summarizer prompt. `2add465fb` (2025-12-29) originally sliced tool args to 100 chars and results to 1000, but `17ce3814a` (2025-12-30) removed truncation ("already token-budgeted") — wrong, because the summarizer has the same window and the summarized range can be almost the whole window.

**Fix · [[pi]]** — `c950c692a` 2026-03-06 (#1796): tool results truncated to `TOOL_RESULT_MAX_CHARS = 2000` + `[... N more characters truncated]` (`packages/coding-agent/src/core/compaction/utils.ts:94-104`, `149`); mirrored in durable (`packages/durable/src/harness/compaction.ts:55`). 0.74.1 also clamped summary output tokens (`3d9e14d74`, see [[summary-output-budget-misfit]]).

**Fix · [[codex]]**
- Symptom variant: a massive earlier user message copied verbatim into the compacted history made "any future /compact task … fail"; the compaction request (full history + prompt) itself overflowed.
- `c415827ac2` 2025-09-22 (#4068) truncate kept user messages (today a 20,000-token budget, `codex-rs/core/src/compact.rs:63`, boundary message truncated, media dropped).
- `687a13bbe5` 2025-10-08 (#4942) "truncate on compact … iteratively": on ContextWindowExceeded drop the OLDEST history item (and its paired call/output) and retry — "Trim from the beginning to preserve cache (prefix-based) and keep recent messages intact" (`codex-rs/core/src/compact.rs:330-346`; `codex-rs/core/src/context_manager/history.rs:680-692`).
- Remote compaction pre-trims tool outputs newest-first with "Output exceeded the available model context and was truncated" until the estimate fits (`codex-rs/core/src/compact_remote_history.rs:16-17`, `:68-124`).
**Fix · [[opencode]]** `0fd6f365be` 2026-02-10 (#12924) 20k reserve "to ensure that input window has enough room to compact" (`packages/opencode/src/session/overflow.ts:8`); `574b2c2170` 2026-04-22 (#23870) `TOOL_OUTPUT_MAX_CHARS = 2_000` + `[truncated]` in the summarizer input (`packages/opencode/src/session/compaction.ts:30`, `:51-52`); media stripped from overflow compactions (`be20f865ac`). Legacy terminal guard: compaction overflow → "Session too large to compact" and stop (`packages/opencode/src/session/compaction.ts:450-459`). v2 pre-checks `Token.estimate(summaryPrompt) > context − summaryOutput` and skips (`packages/core/src/session/compaction.ts:189-190`).

**Lesson** — The summarizer shares the agent's window: bound its input (cap per-tool-result text, or trim oldest-first to keep the cached prefix), bound the post-compaction keep-set, and never assume the summarized range is "already budgeted".

Related: [[transcript-serialization-for-summary]] · [[auto-compaction]] · [[tool-output-truncation]] · [[overflow-recovery]] · [[codex--auto-compaction|codex]]
