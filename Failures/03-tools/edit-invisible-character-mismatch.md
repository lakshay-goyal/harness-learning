---
type: failure
concepts: [search-replace-edit, fuzzy-edit-matching]
harnesses: [pi]
---
**Symptom** — Edit failed with "Could not find the exact text" although the model's `oldText` was visually identical to the file: on Windows CRLF files, on files with a UTF-8 BOM, when the model emitted straight vs smart quotes, en/em dashes, NBSP/unicode spaces, trailing whitespace differences, or full-width/compatibility characters (CJK).

**Root cause** — Exact byte matching against content the model reproduces through tokenization: models emit LF and never the invisible BOM, and normalize typography inconsistently.

**Fix · [[pi]]**
- `c214a3340` 2025-12-29 (#360) / `8c43a9fbc` 2025-12-30 (#355) — detect line ending, normalize file and edits to LF for matching, restore original ending on write (`packages/coding-agent/src/core/tools/edit-diff.ts:11-21`).
- `d9adf659c` 2026-01-02 — strip BOM before matching, re-add on write ("LLM won't include invisible BOM in oldText"; `edit.ts:191,197-198`); `1355cd36e` 2026-08-19 (#8337) normalize BOMs in text inputs.
- `0c135d014` 2026-01-14 (#713, external contributor) — fuzzy fallback: trailing whitespace, smart quotes, unicode dashes and spaces (`edit-diff.ts:34-55`).
- `700bcf345` 2026-03-15 (#2044) — NFKC normalization added to fuzzy matching.

**Lesson** — Match on a normalized view (line endings, BOM, typography), write back on the original bytes.

Related: [[search-replace-edit]] · [[fuzzy-edit-matching]] · [[fuzzy-edit-rewrites-untouched-lines]] · [[pi--search-replace-edit|pi edit]] · [[pi--fuzzy-edit-matching|pi fuzzy]]
