---
type: concept
stage: tool-design
tier: candidate
aliases: [fuzzyFindText, normalizeForFuzzyMatch, applyReplacementsPreservingUnchangedLines, exact-then-fuzzy-edit-match, preserve-untouched-bytes, seek_sequence]
harnesses: [pi, codex]
---
When an exact edit anchor is not found, retry the match on a normalized view of file and anchor (trailing whitespace, smart quotes, unicode dashes/spaces, NFKC) — and write back only the touched lines so untouched bytes stay identical.

## Why
- Models reproduce text imperfectly: trailing whitespace, typographic quotes, en/em dashes, NBSP, full-width compat characters → exact match fails on a semantically correct edit ([[edit-invisible-character-mismatch]]).
- A naive fuzzy implementation that writes the normalized buffer back silently rewrites the whole file (pi: stripped trailing whitespace and converted quotes everywhere, [[fuzzy-edit-rewrites-untouched-lines]]).

## Design space
- Exact only (pi until 2026-01).
- **Exact first, then explicit normalization set** (pi) vs similarity/edit-distance matching (absent; riskier).
- Indentation-insensitive matching (absent in pi).
- Normalize-everything-on-any-fuzzy (pi: if any edit needs fuzzy, all edits re-matched in normalized space) vs per-edit.
- Uniqueness check in normalized space (pi; can reject an exactly-unique anchor) vs exact space.
- **Fuzzy match ≠ fuzzy write**: rewrite only touched line spans, copy others byte-for-byte (pi since 2026-06).
- Tell the model a fuzzy match happened (absent in pi; error text still claims exact-match requirement).
- **Line-wise passes, first match wins, no uniqueness check**: exact → trim_end → trim → Unicode punctuation/space → ASCII (✔ codex `seek_sequence`, "mirrors the fuzzy behaviour of `git apply`"); no NFKC.
- Replacement text taken verbatim from the edit (✔ codex patch `+` lines); untouched lines keep bytes and line endings (✔ codex since `685270a56a`).

## Implementations
- [[pi--fuzzy-edit-matching|pi]] — `fuzzyFindText` exact→`normalizeForFuzzyMatch` (NFKC, trailing ws, quotes, dashes, unicode spaces); line-preserving overlay write.
- [[codex--fuzzy-edit-matching|codex]] — 4-pass `seek_sequence` for `apply_patch` context lines, `@@` anchor seek, EOF-first matching.

## Failures
- [[fuzzy-edit-rewrites-untouched-lines]]
- [[edit-invisible-character-mismatch]]
- [[edit-rewrites-line-endings]]

## Related
[[search-replace-edit]] · [[tool-description-design]] · [[path-normalization]] · [[patch-envelope-edit]] · [[edit-format]]
