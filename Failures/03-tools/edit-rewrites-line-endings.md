---
type: failure
concepts: [patch-envelope-edit, fuzzy-edit-matching]
harnesses: [codex]
---
**Symptom** — "Updating a file with apply_patch historically normalized its contents to LF, which can rewrite line endings outside the requested change." (`21aa552e87` body) — CRLF files came back with every line changed.

**Root cause** — The applier rebuilt the whole file from LF-split lines; untouched lines were re-emitted with the writer's newline, not their original bytes.

**Fix · [[codex]]**
- `21aa552e87` 2026-08-10 "Add a line-ending preservation mode to `apply_patch`" (opt-in) — untouched/context lines keep their original endings; inserted lines use the file's first line ending.
- `685270a56a` 2026-10-05 "Make apply_patch preserve line endings unconditionally"; env var `CODEX_APPLY_PATCH_PRESERVE_LINE_ENDINGS` forced to `1` for older standalone executables (`codex-rs/apply-patch/src/lib.rs:54-57`).

**Lesson** — A minimal-diff editor must preserve bytes outside the hunk, including line endings; ship such fixes as opt-in first, then unconditional.

Related: [[patch-envelope-edit]] · [[fuzzy-edit-matching]] · [[fuzzy-edit-rewrites-untouched-lines]] · [[edit-invisible-character-mismatch]] · [[codex--patch-envelope-edit|codex]]
