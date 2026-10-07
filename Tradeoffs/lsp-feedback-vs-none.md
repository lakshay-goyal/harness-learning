---
type: tradeoff
concepts: [lsp-diagnostics-feedback, post-edit-formatting, tool-result-rewriting]
harnesses: [pi, opencode]
---
# lsp-feedback-vs-none

**Axis**: after an edit, does the harness push compiler/linter diagnostics into the tool result, or leave checking to the model's own commands?

| option | pi | opencode | evidence |
|---|---|---|---|
| No LSP; the model runs `tsc`/tests via bash | ✅ (omission) | default since 2026-04-16 | pi: no LSP commit ever (`git log --grep=lsp`) → [[no-lsp]]; opencode: `if (!cfg.lsp)` disables all (`packages/opencode/src/lsp/lsp.ts:150-151`) |
| Diagnostics appended to edit/write/apply_patch results | hook-buildable (`tool_result`) | ✅ when configured: ERROR only, ≤20 per file, ≤5 other files after `write` | `packages/opencode/src/lsp/diagnostic.ts:3`; `packages/opencode/src/tool/write.ts:18,75-89` → [[lsp-diagnostics-feedback]] |
| Navigation tool (definition, references, symbols) | — | experimental `lsp` tool; older `lsp-hover`/`lsp-diagnostics` tools deleted as unused (`1e2ef07c97`) | `packages/opencode/src/tool/lsp.ts:11-21` |
| Formatter after write | — | ✅ when configured; re-reads so the diff is post-format | `packages/opencode/src/format/index.ts:118-140` → [[post-edit-formatting]] |
| Server lifecycle cost | none | ~30 servers, auto-download, 45 s init timeout, 150 ms debounce | `packages/opencode/src/lsp/client.ts:13-18` |

**Trajectory**: opencode went default-on → opt-in (`220e3e9a2b`) → [[no-lsp-formatters-by-default]]. The two harnesses now differ in *availability*, not in default behavior.

**When each wins**
- **No LSP (pi)**: polyglot or exotic repos, short sessions, containers without language toolchains; zero background processes ([[no-background-bash]]).
- **LSP feedback (opencode)**: typed languages with a fast server, where catching a broken import in the same step saves a full test cycle. Cap the volume (errors only, per-file limit) or the diagnostics flood context.

Related: [[no-lsp]] · [[no-lsp-formatters-by-default]] · [[minimal-vs-rich-toolset]] · [[pi]] · [[opencode]] · [[Tradeoffs]]
