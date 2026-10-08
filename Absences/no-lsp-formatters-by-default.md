---
type: absence
harnesses: [opencode]
---
# no-lsp-formatters-by-default

Reversed default: LSP diagnostics and post-edit formatters went from on to opt-in.

**What's missing (by default)**
- With no `lsp` key in config, all language servers are disabled: `if (!cfg.lsp) … "all LSPs are disabled"` (`packages/opencode/src/lsp/lsp.ts:150-151`).
- With no `formatter` key, all formatters are disabled: `if (!cfg.formatter) … "all formatters are disabled"` (`packages/opencode/src/format/index.ts:120-121`).
- So by default, edit/write/apply_patch results carry no `<diagnostics>` block and files are not reformatted.

**Evidence of decision**
- `81ebf56cf1` (2025-11-18) added top-level `lsp: false` / `formatter: false` kill switches.
- `220e3e9a2b` (2026-04-16, #22997, "refactor: make formatter config opt-in") flipped the default for both; it also touched `packages/opencode/src/lsp/lsp.ts` and the LSP config.
- `6b68b1020e` (2026-05-03, #25502) "docs: clarify LSP and formatter opt-in config".
- No rationale in the commit body (unverified). Plausible: ~30 auto-downloaded servers (`OPENCODE_DISABLE_LSP_DOWNLOAD`, `packages/opencode/src/lsp/server.ts:152`), startup latency (`INITIALIZE_TIMEOUT_MS = 45_000`, `packages/opencode/src/lsp/client.ts:18`), and formatters rewriting user files outside the requested change.

**Implication**
- Even the harness that invests in LSP feedback ([[lsp-diagnostics-feedback]]) found the default cost too high: process management, downloads and surprise rewrites. Default-off narrows the practical gap with pi ([[no-lsp]]) to "available when configured".

Related: [[lsp-diagnostics-feedback]] · [[post-edit-formatting]] · [[no-lsp]] · [[lsp-feedback-vs-none]] · [[opencode]] · [[Absences]]
