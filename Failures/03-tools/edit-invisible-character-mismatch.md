---
type: failure
concepts: [search-replace-edit, fuzzy-edit-matching, patch-envelope-edit]
harnesses: [pi, codex]
---
**Symptom** — Edit failed with "Could not find the exact text" although the model's `oldText` was visually identical to the file: on Windows CRLF files, on files with a UTF-8 BOM, when the model emitted straight vs smart quotes, en/em dashes, NBSP/unicode spaces, trailing whitespace differences, or full-width/compatibility characters (CJK).

**Root cause** — Exact byte matching against content the model reproduces through tokenization: models emit LF and never the invisible BOM, and normalize typography inconsistently.

**Fix · [[pi]]**
- `c214a3340` 2025-12-29 (#360) / `8c43a9fbc` 2025-12-30 (#355) — detect line ending, normalize file and edits to LF for matching, restore original ending on write (`packages/coding-agent/src/core/tools/edit-diff.ts:11-21`).
- `d9adf659c` 2026-01-02 — strip BOM before matching, re-add on write ("LLM won't include invisible BOM in oldText"; `edit.ts:191,197-198`); `1355cd36e` 2026-08-19 (#8337) normalize BOMs in text inputs.
- `0c135d014` 2026-01-14 (#713, external contributor) — fuzzy fallback: trailing whitespace, smart quotes, unicode dashes and spaces (`edit-diff.ts:34-55`).
- `700bcf345` 2026-03-15 (#2044) — NFKC normalization added to fuzzy matching.

**Fix · [[codex]]**
- Symptom in codex: patches authored with ASCII punctuation failed against files containing typographic dashes / quotes / NBSP ("weird unicode characters").
- `15bf5ca971` 2025-04-25 — 4th `seek_sequence` pass normalizing dashes U+2010–2015/U+2212, smart quotes, NBSP and U+2002–200A/202F/205F/3000 to ASCII (`codex-rs/apply-patch/src/seek_sequence.rs:66-110`), after exact / trim_end / trim passes; "mirrors the fuzzy behaviour of `git apply`".
- No NFKC and no BOM normalization; line endings handled by preservation on write (`685270a56a`, [[edit-rewrites-line-endings]]).

**Lesson** — Match on a normalized view (line endings, BOM, typography), write back on the original bytes.

Related: [[search-replace-edit]] · [[fuzzy-edit-matching]] · [[fuzzy-edit-rewrites-untouched-lines]] · [[pi--search-replace-edit|pi edit]] · [[pi--fuzzy-edit-matching|pi fuzzy]] · [[codex--fuzzy-edit-matching|codex]] · [[patch-envelope-edit]]
