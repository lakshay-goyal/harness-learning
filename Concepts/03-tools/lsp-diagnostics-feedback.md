---
type: concept
stage: tools
tier: variant
aliases: [LSP.Service, touchFile, "<diagnostics file=…>", "LSP errors detected in this file", lsp tool, OPENCODE_DISABLE_LSP_DOWNLOAD]
harnesses: [opencode]
---
After a file mutation, the harness notifies a language server and appends the resulting error diagnostics to the tool result.

## Why
- Without it the model learns about type/syntax errors only when it runs a build, often several steps later.
- Unfiltered diagnostics waste context and send the model chasing warnings ([[stale-diagnostics-after-edit]]).
- Language servers must be installed, started and waited for; this costs latency and surprises users.

## Design space
- **Push into the edit/write result** (opencode legacy) vs a separate diagnostics tool the model calls (opencode's `lsp-diagnostics` tool, removed as unused) vs none.
- Severity filter: errors only (opencode, since 2025-12) vs all.
- Caps: per file (opencode 20) and other files after a write (opencode 5).
- Wait strategy: debounce pushed diagnostics, bounded document/full waits, pull diagnostics where supported (opencode).
- Server provisioning: built-in server table with auto-download (opencode, ~38 servers) vs user-installed only.
- Default: on (opencode until 2026-04-16) vs **opt-in by config** (opencode since).
- Navigation tool (definition, references, hover, call hierarchy) as a separate experimental tool (opencode `lsp`).
- None; model runs `tsc`/linters through the shell (pi, [[no-lsp]]).

## Implementations
- [[opencode--lsp-diagnostics-feedback|opencode]] — edit/write/apply_patch call `touchFile` + `diagnostics()`; ERROR-only `<diagnostics file>` blocks, 20/file, +5 other files on write; opt-in since `220e3e9a2b`.

## Failures
- [[stale-diagnostics-after-edit]]
- [[empty-success-output-read-as-failure]]

## Tradeoffs
- [[lsp-feedback-vs-none]]

## Related
[[no-lsp]] · [[post-edit-formatting]] · [[harness-diagnostics-channel]] · [[search-replace-edit]] · [[patch-envelope-edit]] · [[file-read-tool]]
