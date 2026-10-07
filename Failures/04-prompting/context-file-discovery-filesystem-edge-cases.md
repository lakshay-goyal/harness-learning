---
type: failure
concepts: [context-file-hierarchy]
harnesses: [pi, codex]
---
**Symptom** — Context-file discovery hung or emitted spurious errors on filesystem edge cases: on Windows the ancestor walk never terminated (#6369); a *directory* named like a context file caused a yellow "Warning: Could not read /Users/…/AGENTS.md: Error: EISDIR" on every start (#7106; commit body of `58c0bc2fb`).

**Root cause** — Walk termination compared against `resolve("/")` and stepped with `resolve(currentDir, "..")` (removed lines in `2170363af`), which never matches a Windows drive root; candidate names were checked with `existsSync` only.

**Fix · [[pi]]**
- `2170363af` 2026-07-09 (#6369) "avoid Windows context file walk hang": stop when `dirname(dir) === dir` (HEAD `packages/coding-agent/src/core/resource-loader.ts:262-264`).
- `58c0bc2fb` 2026-07-25 (#7106) "exclude directories from resource loader": `statSync().isFile()` check, skip non-files (`resource-loader.ts:190-192`).

**Fix · [[codex]]**
- Symptom variant: `project_doc_fallback_filenames` entries with path syntax caused metadata probes on Windows network paths, which "can send ambient credentials".
- `50d77959bf` 2026-09-16: reject `.`, `..`, `/`, NUL (and `\`, `:` on Windows executors) before probing — "Probing Windows network paths can send ambient credentials, even during metadata checks" (`50d77959bf` body; check at `codex-rs/core/src/agents_md.rs:281-292`).
- Related bound: walk stops at the nearest `project_root_markers` ancestor and never past it; probes concurrent, capped at 256 (`codex-rs/core/src/agents_md.rs:1-18`, `:54`; `6ad0e943cc`). `85e0661c3b` 2026-08-07: per-environment budgets let the total AGENTS.md payload grow with environment count → one shared `project_doc_max_bytes` budget.

**Lesson** — Terminate path walks on a fixed point (`dirname(d) === d` or a root marker), verify file type, and validate configured names as plain file names before any filesystem probe — even a metadata check can leak credentials on network paths.

Related: [[context-file-hierarchy]] · [[context-file-loaded-twice-in-worktrees]] · [[pi--context-file-hierarchy|pi]] · [[codex--context-file-hierarchy|codex]]
