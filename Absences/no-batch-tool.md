---
type: absence
harnesses: [opencode]
---
# no-batch-tool

Removed: a meta-tool that ran 1–25 other tool calls in parallel.

**What's missing**
- No `batch` tool at HEAD. Parallelism comes from native parallel tool calls in one assistant message, and (experimental) from code mode's `execute` with `TOOL_CALL_CONCURRENCY = 8` (`packages/codemode/src/stdlib/promise.ts:6`) → [[code-mode]].

**Evidence of decision**
- Added behind `experimental.batch_tool` in `1056b36eae` (2025-11-15, #2983). Description (`463318486f^:packages/opencode/src/tool/batch.txt:1-14`):
  - "Executes multiple independent tool calls concurrently to reduce latency."
  - "USING THE BATCH TOOL WILL MAKE THE USER HAPPY."
  - "1–25 tool calls per batch", "Partial failures do not stop other tool calls", "Do NOT use the batch tool within another batch tool."
- Deleted in the tool-system refactor `463318486f` (2026-04-07, #21052). No stated reason (unverified). Likely: native parallel tool calls made it redundant, and it never left the experimental flag.
- Sibling removal: `multiedit` ("multiple find-and-replace operations" on one file) commented out `35b03e4cb3` (2025-06-05), deleted as unused `2486621ca1` (2026-04-21) → [[removed-builtin-tools]].

**Contrast**
- pi never had a batch tool; its batching is codemode scripts plus a bounded nested-call record (`packages/coding-agent/src/core/nested-tool-calls.ts:25-30`) → [[nested-tool-calls]], [[parallel-tool-execution]].

**Implication**
- A wrapper tool that re-implements the provider's parallel tool calls competes with the native path and needs emotional prompt pressure ("WILL MAKE THE USER HAPPY") to get used. Both harnesses converged on scripts (code mode) instead of a list-of-calls meta-tool.

Related: [[parallel-tool-execution]] · [[code-mode]] · [[nested-tool-calls]] · [[tool-description-design]] · [[removed-builtin-tools]] · [[opencode]] · [[Absences]]
