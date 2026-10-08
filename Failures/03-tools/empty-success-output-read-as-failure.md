---
type: failure
concepts: [structured-tool-output, lsp-diagnostics-feedback]
harnesses: [opencode]
---
**Symptom** — After a successful edit with no diagnostics the tool output was empty, and with diagnostics it read "This file has errors, please fix". The model assumed the edit had not applied and re-applied it.

**Root cause** — Success was implied by silence; diagnostic framing sounded like a tool failure. An LSP spawn failure was also injected as edit diagnostics.

**Fix · [[opencode]]**
- `aa4dba1541` 2025-08-21 LSP spawn failures no longer injected into edit diagnostics.
- `66f9bdab32` 2026-01-12 "tweak edit and write tool outputs to prevent agent from thinking edit didn't apply": output starts "Edit applied successfully." / "Wrote file successfully."; diagnostics follow as "LSP errors detected in this file, please fix:" (`packages/opencode/src/tool/edit.ts:193-198`; `packages/opencode/src/tool/write.ts:74-89`).

**Lesson** — State success explicitly; keep follow-up diagnostics visibly separate from the success line.

Related: [[structured-tool-output]] · [[lsp-diagnostics-feedback]] · [[harness-diagnostics-channel]] · [[tool-result-misreports-facts]] · [[opencode--lsp-diagnostics-feedback|opencode]]
