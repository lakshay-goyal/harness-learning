---
type: failure
concepts: [tool-description-design, shell-execution]
harnesses: [codex]
---
**Symptom** — On Windows the model enumerated paths in PowerShell and then deleted/moved them via `cmd /c` or computed paths, risking recursive deletes outside the workspace; it also launched services with `Start-Process`, leaving a "user-visible powershell window that probably doesn't get cleaned up" (`2e228969be` body).

**Root cause** — Quoting/escaping semantics differ across shells; paths built in one shell and consumed by another can resolve to unintended targets. Guidance was keyed to the *client* platform, not where commands run.

**Fix · [[codex]]**
- `69750a0b5a` 2026-03-19 "add specific tool guidance for Windows destructive commands (#15207)": "Do not compose destructive filesystem commands across shells. Do not enumerate paths in PowerShell and then pass them to `cmd /c`, batch builtins, or another shell for deletion or moving. Use one shell end-to-end, prefer native PowerShell cmdlets such as `Remove-Item` / `Move-Item` with `-LiteralPath`…"; "Before any recursive delete or move on Windows, verify the resolved absolute target paths stay within the intended workspace…".
- `2e228969be` 2026-04-23 "When using `Start-Process` to launch a background helper or service, pass `-WindowStyle Hidden`…".
- `5ed294d49d` 2026-08-28 "Match Windows shell guidance to the executor platform" — appended only when the **executor** is Windows (`codex-rs/core/src/tools/handlers/shell_spec.rs:339-345`).

**Lesson** — Platform-specific hazards belong in the tool description, keyed to the platform where the tool executes, not where the client runs.

Related: [[tool-description-design]] · [[shell-execution]] · [[windows-process-tree-and-shells]] · [[dangerous-command-heuristics]] · [[codex--shell-execution|codex]]
