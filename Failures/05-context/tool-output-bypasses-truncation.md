---
type: failure
concepts: [tool-output-truncation, mcp-integration, harness-diagnostics-channel, context-overflow-detection, shell-execution]
harnesses: [opencode, codex]
---
**Symptom** — A single MCP tool call (or an edit that surfaced every LSP error in a file) dumped unbounded text into context and blew the window, while built-in tools were capped.

**Root cause** — Truncation lived in the built-in tool wrapper; MCP results took a separate path in the prompt loop, and appended LSP diagnostics were not counted against any budget. In v2 the first unified runtime shipped with no output bound at all.

**Fix · [[opencode]]**
- `aedb5550a8` 2025-12-14 (#5480) cap LSP diagnostics at 20 per file, plus "... and N more", and 5 other project files.
- `0d49df46ef` 2026-01-19 "ensure truncation handling applies to mcp servers too": MCP text parts go through `Truncate.output` with the same spill/hint.
- `a9094fd059` 2026-06-05 (#30999) v2: one bound at the tool-registry settlement boundary for every tool (`packages/core/src/tool/registry.ts:75`; `packages/core/src/tool-output-store.ts:138-174`); "Tools … do not truncate model-facing output" (`specs/v2/tools.md:155`).
- `69f1ec22e3` 2026-06-21 bound web tool bodies before reading (`MAX_RESPONSE_BYTES = 5 MiB`, `packages/core/src/tool/webfetch.ts:17`).

**Lesson** — Put the output bound at the single generic tool-result boundary that every source (built-in, MCP, plugin, harness-appended diagnostics) must pass, not inside each tool.

Related: [[tool-output-truncation]] · [[tool-output-spill]] · [[mcp-integration]] · [[opencode--tool-output-truncation|opencode impl]] · [[bash-output-integrity]]

## Also: [[codex]] (folded from `unbounded-tool-output-overflows-context`)
**Symptom**
- Some exec code paths skipped output truncation, so users "with 76% context window left" got `input exceeded context window` (`d7acd146fb` body).
- Tool-level truncation followed by history-level truncation re-truncated already-cut output (`9890ceb939`).
- "just 3 MCP tool calls that each were multi-hundred MBs" made the rollout JSONL enormous — both the event and the `function_call_output` records (`3516cb9751` body).
- Replaying tool outputs under another model changed how much history the model saw (truncation used the new model's budget) → [[truncation-budget-drift-on-replay]].

**Root cause** — Truncation enforced per call site instead of at one choke point; persisted copies (events, rollout, paginated history) were not bounded like the model-visible copy.

**Fix · [[codex]]**
- `d7acd146fb` 2025-10-04 "fix: exec commands that blows up context window (#4706)" (`codex-rs/core/src/tools/mod.rs`).
- `9890ceb939` 2025-11-13 "Avoid double truncation" — history gets headroom over the tool constant; today a 20 % serialization allowance (`codex-rs/utils/output-truncation/src/lib.rs:17-21`; `codex-rs/core/src/tools/context.rs:553-560`); `d62cab9a06` 2025-11-19 don't truncate at newlines.
- MCP: oversized serialized results → bounded text preview preserving `isError` (`codex-rs/utils/output-truncation/src/lib.rs:43-89`); `3516cb9751` 2026-04-30 truncate large MCP outputs in rollouts; event copies capped `MCP_TOOL_CALL_EVENT_RESULT_MAX_BYTES = 1 MiB` (`codex-rs/core/src/mcp_tool_call.rs:125,987-1001`); `820f85cf59` 2026-10-02 64 KiB preview in paginated thread history; `bee28e8a06` 2026-10-02 persisted command output capped at 64 KiB.
- `aa88a0333c` 2026-09-09 truncation budget persisted with each tool output (resume / fork / model switch deterministic).
- Pre-send guard: when the estimate still exceeds the window, newest→oldest tool outputs replaced with an "Output exceeded the available model context…" placeholder (M6, [[context-overflow-detection]]).

**Lesson** — Enforce truncation at one choke point for every tool path, give layered truncators headroom so they don't compound, and bound every persisted copy of a tool result — not only the model-visible one.

Related: [[tool-output-truncation]] · [[mcp-integration]] · [[context-overflow-detection]] · [[shell-execution]] · [[bash-output-integrity]] · [[truncation-budget-drift-on-replay]] · [[codex--mcp-integration|codex mcp]] · [[codex--shell-execution|codex shell]]
