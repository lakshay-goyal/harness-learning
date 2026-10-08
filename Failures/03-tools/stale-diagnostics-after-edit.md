---
type: failure
concepts: [lsp-diagnostics-feedback, harness-diagnostics-channel]
harnesses: [opencode]
---
**Symptom** — After an edit the model chased errors the edit had already fixed (diagnostics captured before the server finished), or "fixed" warnings because a diagnostics block appeared when only warnings existed; output also flooded context.

**Root cause** — Diagnostics collected on the first publish (partial/stale); all severities shown; no caps.

**Fix · [[opencode]]**
- `aedb5550a8` 2025-12-14 limit diagnostics "to prevent context window waste" (20 per file, 5 other files).
- `ef78fd8bae` 2025-12-16 "debounce LSP diagnostics to get complete results (#5600)" — 150 ms (`packages/opencode/src/lsp/client.ts:13`).
- `46c7a41d5f` 2025-12-26 "only show diagnostics block when errors exist (#6175)" — severity 1 only (`packages/opencode/src/lsp/diagnostic.ts:20-21`).
- `e383df4b17` 2026-04-23 pull diagnostics where supported.

**Lesson** — Wait for the language server to settle and show only actionable severities, capped.

Related: [[lsp-diagnostics-feedback]] · [[harness-diagnostics-channel]] · [[empty-success-output-read-as-failure]] · [[opencode--lsp-diagnostics-feedback|opencode]]
