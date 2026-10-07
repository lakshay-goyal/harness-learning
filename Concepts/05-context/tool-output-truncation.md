---
type: concept
stage: context
tier: candidate
aliases: [truncateHead, truncateTail, truncateMiddle, truncateLine, DEFAULT_MAX_LINES, DEFAULT_MAX_BYTES, GREP_MAX_LINE_LENGTH, MCP_OUTPUT_MAX_BYTES, outputLimits, dual-limit-output-truncation, truncation-direction-by-tool, actionable-truncation-notice, TruncationPolicy, truncation_policy, tool_output_token_limit, with_serialization_allowance, history_truncation_token_limit, HeadTailBuffer, max_output_tokens]
harnesses: [pi, codex]
---
Bound every tool result by lines OR bytes (whichever hits first), keep the end that matters for that tool (head for reads/searches, tail for shell, middle for opaque remote output), and tell the model exactly how to get the rest.

## Why
- One `cat` of a log or a minified file can consume the whole window; context is the scarcest resource and every later request re-pays it.
- A line cap alone fails on minified one-line files; a byte cap alone fails on many short lines — hence dual limits.
- A notice without a recovery path makes models stop at the first chunk (pi: read stopped after 2000 lines until the description said "continue with offset until complete", `89636cfe6`).
- A truncation notice that points to a full-output file that doesn't exist is a lie ([[bash-output-integrity]]).

## Design space
- Limits: 2000 lines / 50 KB ✔ pi (started 30 KB, raised same day `306f9cc66`); per-line cap for grep (500 chars); smaller cap for MCP (20 KB); codemode 10k tokens. **Single token budget per model from the catalog** (`truncation_policy {tokens, 10000}`; unknown models bytes 10,000), overridable by `tool_output_token_limit` ✔ codex.
- Collection vs model budget: bound process-output *collection* memory separately (1 MiB head/tail buffer) from the model budget ✔ codex.
- Direction per tool: head (read, grep, find, ls) / tail (bash: errors at end) / middle (MCP, codemode, "like Codex") ✔ pi; **middle elision for everything** (head + tail kept, `…N tokens truncated…`) ✔ codex.
- Unit integrity: whole lines only (head) vs partial last line from the end, UTF-8-safe (tail) ✔ pi.
- Notice content: actionable `[Showing lines a-b of N. Use offset=K to continue.]`, `Full output: <path>`, `Use limit=200 for more` ✔ pi; durable moves notices out of content into a `<harness>` diagnostics block.
- Where enforced: inside each tool ✔ pi coding-agent vs harness-level per-tool `outputLimits{retain}` ✔ pi durable vs **at history-record time for every function/custom tool output (one choke point), ×1.2 serialization allowance, per-result override persisted with the item** ✔ codex.
- Full output kept where? spill file (pi) vs **untruncated in the session log only, not model-readable** ✔ codex ([[no-tool-output-spill-file]]); replay re-truncates with the persisted per-item budget so model switches don't change history ✔ codex.
- Multi-part outputs: shared budget over text/audio items, images exempt ✔ codex.
- Never restate the limit as a literal in the system prompt ([[prompt-states-stale-harness-limits]]).
- Escape hatch: spill full output to file → [[tool-output-spill]].
- Structured callers (scripts) get more than the model (bash 1 MiB structured output) ✔ pi.
- Second truncation at summarization time (2000 chars per result) → [[transcript-serialization-for-summary]].

## Implementations
- [[pi--tool-output-truncation|pi]] — `truncate.ts` head/tail/middle/line helpers, 2000 lines / 50 KB, per-tool direction and notices; durable harness enforces `outputLimits` with tail windows and diagnostics.
- [[codex--tool-output-truncation|codex]] — per-model token policy (10k), middle elision with "Warning: truncated output (original token count: N)", applied when recording history; 1 MiB head/tail exec collection buffer; full payload kept in rollout.

## Failures
- [[bash-output-integrity]] (03-tools) — spill missing on line-limit truncation, wrong line counts, durable tail window dependent on commit timing
- [[partial-file-read-acted-on]] (03-tools) — model stopped at the first truncated chunk
- [[unbounded-tool-output-overflows-context]] (03-tools) — exec paths skipped truncation; double truncation
- [[truncation-budget-drift-on-replay]] — replay under another model re-truncated history differently
- [[prompt-states-stale-harness-limits]] — prompt said "10 kilobytes or 256 lines" after limits changed
- [[unbounded-payload-in-transcript]] (08-state) — multi-hundred-MB MCP results in the rollout
- [[output-window-depends-on-commit-cadence]] (08-state) — In pi-durable, tail-retained tool output (bash) came out differently depending on when progress commits…

## Related
[[tool-output-spill]] · [[shell-execution]] · [[file-read-tool]] · [[search-tools]] · [[mcp-integration]] · [[code-mode]] · [[tool-description-design]] · [[harness-diagnostics-channel]] · [[transcript-serialization-for-summary]]
