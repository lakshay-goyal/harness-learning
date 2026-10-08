---
type: implementation
harness: pi
concept: tool-output-truncation
commit: b30a6dd77
files: [packages/coding-agent/src/core/tools/truncate.ts:11, packages/coding-agent/src/core/tools/truncate.ts:78, packages/coding-agent/src/core/tools/truncate.ts:168, packages/coding-agent/src/core/tools/truncate.ts:292, packages/coding-agent/src/core/tools/bash.ts:338, packages/coding-agent/src/core/tools/read.ts:179, packages/coding-agent/src/core/tools/grep.ts:282, packages/coding-agent/src/extensions/mcp/tools.ts:129, packages/coding-agent/src/extensions/codemode/execute.ts:373, packages/durable/src/harness/tool.ts:176, packages/durable/src/harness/output.ts:34]
---
[[tool-output-truncation]] in [[pi]].

## Mechanism
- **Shared helpers** `packages/coding-agent/src/core/tools/truncate.ts`: two independent limits, whichever hits first (`:1-9`).
  - `truncateHead` (`:78-160`): keep first N **complete** lines; byte count includes `\n` per line after the first (`:128`); first line alone > maxBytes → empty content + `firstLineExceedsLimit` (`:103-119`).
  - `truncateTail` (`:168-241`): keep last N lines; if the last line alone > maxBytes keep its **end**, UTF-8-safe via continuation-byte scan (`truncateStringToBytesFromEnd`, `:247-262`), `lastLinePartial`.
  - Trailing `\n` not counted as an extra line (`splitLinesForCounting`, `:47-56`; `f95306781` #4818).
  - `truncateLine` per-line char cap with `... [truncated]` (grep, `:268-276`); `truncateMiddle(content, maxBytes)` head/2 + tail/2 with `…N chars truncated…` "like Codex does" (`:292-314`, used by MCP); `formatSize` (`:61-69`).
- **Direction & notices per tool**:
  - **read** (head): `[Showing lines a-b of T. Use offset=N to continue.]` / `(50.0KB limit)` variant; user `limit` stopped early → `[K more lines in file. Use offset=N to continue.]`; first line > 50 KB → `[Line N is X, exceeds 50.0KB limit. Use bash: sed -n 'Np' path | head -c 51200]` (line content not shown) (`packages/coding-agent/src/core/tools/read.ts:179-199`). Description: "…output is truncated to 2000 lines or 50KB (whichever is hit first). Use offset/limit for large files. When you need the full file, continue with offset until complete." (`packages/coding-agent/src/core/tools/read.ts:96`; last sentence `89636cfe6`).
  - **bash/powershell** (tail, "errors/final results are at the end", `truncate.ts:162-166`): `[Showing last {X} of line {N} (line is {Y}). Full output: {path}]`, `[Showing lines {a}-{b} of {total}. Full output: {path}]`, `… ({50.0KB} limit). Full output: …`; empty → `(no output)` (`packages/coding-agent/src/core/tools/bash.ts:338-356`). Streaming via `OutputAccumulator` (bounded rolling tail, incremental global line counts, `packages/coding-agent/src/core/tools/output-accumulator.ts`, `6b18cdbac`). Structured output for scripts up to `STRUCTURED_OUTPUT_MAX_BYTES = 1 MiB` (first/last 512 KiB, `packages/coding-agent/src/core/tools/bash.ts:24`, `1ff5b6fdd`). Renderer strips the model-facing footer to avoid duplicating the path in the TUI (`packages/coding-agent/src/core/tools/renderers/bash.ts:52-60`, `7dad27e5f` #4819).
  - **grep** (head, bytes only): each line ≤500 chars, `truncateHead(maxLines: MAX_SAFE_INTEGER)` (`packages/coding-agent/src/core/tools/grep.ts:282`); joined notice `[100 matches limit reached. Use limit=200 for more, or refine pattern. 50.0KB limit reached. Some lines truncated to 500 chars. Use read tool to see full lines]` (`packages/coding-agent/src/core/tools/grep.ts:286-303`).
  - **find** `[1000 results limit reached. Use limit=2000 for more, or refine pattern. 50.0KB limit reached]`; **ls** 500 entries.
  - **MCP** (middle): text > `MCP_OUTPUT_MAX_BYTES = 20 KB` → `Warning: truncated output (original token count: N)\nTotal output lines: M` + middle-cut body + `[Full output: path (read it with offset/limit)]` (Codex format) (`packages/coding-agent/src/extensions/mcp/tools.ts:121-140`); scripts get untruncated `structuredContent`.
  - **codemode** (middle, char-based): `DEFAULT_MAX_OUTPUT_TOKENS = 10_000` × 4 chars, head/tail halves + `…N tokens truncated…`, spill to `pi-codemode-*.txt` (`packages/coding-agent/src/extensions/codemode/execute.ts:246-248`, `373-396`); per-script `// @options: {"max_output_tokens": …}`.
- **Escape hatch** → [[pi--tool-output-spill]].
- **Second truncation layer** at summarization time (2000 chars/result) → [[pi--transcript-serialization-for-summary]].
- No ANSI stripping/binary sanitizing on the model path for the bash tool (only in rendering and user `!` commands) (`packages/coding-agent/src/core/tools/render-utils.ts:48`; observed absence).

## Constants
| name | value | path:line |
|---|---|---|
| `DEFAULT_MAX_LINES` | 2000 | `packages/coding-agent/src/core/tools/truncate.ts:11` |
| `DEFAULT_MAX_BYTES` | 50 KB (was 30 KB for hours) | `packages/coding-agent/src/core/tools/truncate.ts:12` |
| `GREP_MAX_LINE_LENGTH` | 500 chars | `packages/coding-agent/src/core/tools/truncate.ts:13` |
| grep / find / ls limits | 100 / 1000 / 500 | `packages/coding-agent/src/core/tools/grep.ts:41`, `packages/coding-agent/src/core/tools/find.ts:41`, `packages/coding-agent/src/core/tools/ls.ts:23` |
| `MCP_OUTPUT_MAX_BYTES` | 20 KB | `packages/coding-agent/src/extensions/mcp/tools.ts:51` |
| codemode output | 10_000 tokens (chars/4) | `.../codemode/execute.ts:246` |
| bash structured output | 1 MiB | `packages/coding-agent/src/core/tools/bash.ts:24` |
| durable mirror | 2000 / 50 KB | `packages/durable/src/truncate.ts:11-12` |

## Evolution
- 2025-11-12 `c7a73d4f8` read capped *line length* at 2000 chars (plain text, no line numbers) — later replaced by byte-based truncation.
- 2025-12-07 `de77cd141` truncation with line/byte limits at **2000 lines / 30 KB**; same day `306f9cc66` (#134) raised to 50 KB; `b813a8b92` (#134) "All notices now in text content (LLM sees them)", grep line cap 500, grep/find/ls limits.
- 2026-01-24 `89636cfe6` read description tells model to continue with offset.
- 2026-04-05 `52d16d5a3` (#2852) spill on line truncation too.
- 2026-05-04 `6b18cdbac` (#4145/#4165) `OutputAccumulator` (no per-chunk re-concat, global line numbers).
- 2026-05-21 `f95306781` (#4818) trailing newline not an extra line; `7dad27e5f` (#4819) renderer footer dedupe.
- 2026-09-24 `601437d5a` durable truncation metadata fixes, same 50 KiB / 2000 defaults.
- 2026-09-29 `8562bcf66` MCP middle truncation + codemode output budget; `1ff5b6fdd` 1 MiB structured bash output.
- 2026-10-04 `cdf79797b` durable windowed shell output with counted skips.

## Evidence commits
`c7a73d4f8` `de77cd141` `306f9cc66` `b813a8b92` `89636cfe6` `52d16d5a3` `6b18cdbac` `f95306781` `7dad27e5f` `601437d5a` `8562bcf66` `1ff5b6fdd` `cdf79797b`

## Quirks
- read reads the whole file into memory before truncating (no size cap; no binary detection for non-images) → [[no-binary-detection-in-read]].
- Raw ANSI sequences reach the model from bash (open question whether intentional).
- Truncation by *bytes* while the summarizer cap is *chars* and the context budget is *tokens* — three units.

## Durable variant (packages/durable)
- Harness-level per-tool `outputLimits{maxBytes, maxLines, retain}` (default `retain:"head"`, bash/powershell `"tail"`) enforced on content by `OutputBuffer`/`boundOutput` (`packages/durable/src/harness/tool.ts:176-181`, `340-373`; `packages/durable/src/harness/output.ts:5-36`); truncation/continuation remarks are **diagnostics**, rendered after content as `<harness>\n[severity] message\n</harness>` (`packages/durable/src/harness/tool.ts:454-485`) → [[harness-diagnostics-channel]]. Read hints ("Use offset=N to continue", `sed -n 'Np' | tail -c +K`) go to diagnostics, never content; oversized first line shows its first 50 KB (`packages/durable/src/tools/read.ts:175-209`).
- Tail tools get an `outputWindow{maxBytes,maxLines,minIntervalMs,bytesPerSecond:100 KiB}` pushed down to the exec env / remote daemon so dropped output is never transferred; skipped bytes/newlines are counted (`packages/durable/src/harness/tool.ts:198-206`; `cdf79797b`) → [[shell-execution]].
- Durable 1.0.3: tail-retained output depended on progress commit timing ([[bash-output-integrity]]).

## Failures
[[bash-output-integrity]] (03-tools: spill on line truncation, line counts, durable deterministic window) · [[partial-file-read-acted-on]] (03-tools)
