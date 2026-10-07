---
type: concept
stage: tool-design
tier: candidate
aliases: [fuzzyFindText, normalizeForFuzzyMatch, applyReplacementsPreservingUnchangedLines, exact-then-fuzzy-edit-match, preserve-untouched-bytes, Replacer, BlockAnchorReplacer, SINGLE_CANDIDATE_SIMILARITY_THRESHOLD, isDisproportionateMatch]
harnesses: [pi, opencode]
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
- Similarity scoring with a floor and a size-proportionality veto (opencode block anchors: Levenshtein ≥ 0.65, span ≤ max(n+3, 2n) lines) — the risk pi avoided materialized for ~11 months at threshold 0.0.
- Cascade of candidate generators, each yielding real file substrings, first unique wins (opencode, 9 replacers) vs one normalized view (pi).
- Defer fuzzy entirely in a rewrite until exact behaviour is established (opencode v2).

## Implementations
- [[pi--fuzzy-edit-matching|pi]] — `fuzzyFindText` exact→`normalizeForFuzzyMatch` (NFKC, trailing ws, quotes, dashes, unicode spaces); line-preserving overlay write.
- [[opencode--fuzzy-edit-matching|opencode]] — cascade of 9 candidate generators (line-trimmed, block-anchor Levenshtein, whitespace, indentation, escapes…); first unique real-file substring wins; 0.65 similarity floor + disproportionate-span veto since `236cfcbbc3` (was 0.0).

## Failures
- [[fuzzy-edit-rewrites-untouched-lines]]
- [[edit-invisible-character-mismatch]]
- [[edit-oldstring-drops-lines]]

## Tradeoffs
- [[edit-tool-variants]]

## Related
[[search-replace-edit]] · [[tool-description-design]] · [[path-normalization]]
