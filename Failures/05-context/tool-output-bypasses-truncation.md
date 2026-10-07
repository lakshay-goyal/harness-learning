---
type: failure
concepts: [tool-output-truncation, mcp-integration, harness-diagnostics-channel]
harnesses: [opencode]
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
