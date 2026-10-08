---
type: failure
concepts: [patch-envelope-edit, path-normalization]
harnesses: [codex]
---
**Symptom** — A single patch could contain two operations on the same file through different spellings (`duplicate.txt` and `./duplicate.txt`), and was accepted — the second hunk was verified against stale in-memory content. Earlier, a rename's move destination was resolved against the main repo instead of the `cd <worktree>` effective cwd (issue #5485).

**Root cause** — Hunks were keyed by the path string as written, not by the resolved path; the effective cwd from a `cd dir &&` prefix wasn't applied to every path.

**Fix · [[codex]]**
- `316352be94` 2025-11-06 "Fix apply_patch rename move path resolution (#5486)".
- `a1c88e865d` 2026-08-10 "Reject duplicate resolved paths in apply_patch (#37867)" — hunks resolved via `PathUri` against `effective_cwd`; a repeat → `"multiple operations target {path}"` (`codex-rs/apply-patch/src/invocation.rs:190-206`).

**Lesson** — Canonicalize every path (including workdir and move targets) before validating a batch edit, and reject aliasing.

Related: [[patch-envelope-edit]] · [[path-normalization]] · [[apply-patch-path-and-permission-hazards]] · [[codex--patch-envelope-edit|codex]] · [[codex--path-normalization|codex paths]]
