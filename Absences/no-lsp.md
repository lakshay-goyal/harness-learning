---
type: absence
harnesses: [pi, codex]
---
# no-lsp

**What's missing**
- No Language Server Protocol client. There are no post-edit diagnostics, go-to-definition, symbol rename, or type errors fed back to the model.
- The edit/write tools return only a diff or success text. The model must run `tsc`, linters or tests through bash to see errors.

**Evidence of decision**
- Absence by omission. No stated rationale was found.
- `git log -i -E --grep='\blsp\b|language.server'` returns no relevant commit; the one hit, `8386a807f` "externalize koffi from bun binary builds", is unrelated.
- The only mention at HEAD lists language servers among child processes that run with user permissions: "Extensions, package installers, language servers, and other child processes run with those same permissions…" (`packages/coding-agent/docs/security.md:3`). This presumes LSPs arrive via extensions.
- Consistent with the minimal 4-tool default (`DEFAULT_TOOL_NAMES = ["read","bash","edit","write"]`, `packages/coding-agent/src/core/settings-manager.ts:215`) and "pi's core is minimal" (`CONTRIBUTING.md:7-9`).

**Opt-in replacement**
- None in `examples/extensions/`. Third-party packages are unverified.
- Building one: an extension could hook `tool_result` for edit/write and append diagnostics ([[tool-result-rewriting]], [[extension-event-hooks]]), or register an `lsp` tool ([[plugin-tools]]). MCP LSP bridges also work since `8562bcf66` ([[mcp-integration]]).

**History**
- Never present, never discussed in commits.

**Implication**
- Feedback about correctness comes only from what the model chooses to run: bash, tests, builds. Edits can leave a broken build until the model or user checks.
- Avoids per-language server lifecycle, indexing latency and process management. That fits [[no-background-bash]] (no long-lived helper processes).

**codex** — *absent too*: no `lsp` / `language server` identifiers in any `codex-rs/**/*.rs` (`git grep -l -iE '\blsp\b|language.?server'` → no hits at `622e9e3696`). Diagnostics come from the model running compilers/tests through the shell.
**opencode contrast**: implements it, opt-in since `220e3e9a2b` (2026-04-16): ERROR diagnostics (≤20/file) appended to edit/write/apply_patch results, ~30 servers auto-downloaded, experimental `lsp` tool (`packages/opencode/src/lsp/lsp.ts:151`; `packages/opencode/src/lsp/diagnostic.ts:3`) — see [[lsp-diagnostics-feedback]] / [[lsp-feedback-vs-none]].

Related: [[minimal-default-toolset]] · [[tool-result-rewriting]] · [[plugin-tools]] · [[mcp-integration]] · [[no-codebase-index]] · [[Absences]]
