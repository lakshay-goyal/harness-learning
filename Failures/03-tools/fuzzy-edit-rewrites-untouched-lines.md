---
type: failure
concepts: [fuzzy-edit-matching, search-replace-edit]
harnesses: [pi, opencode]
---
**Symptom** — A single edit that needed the fuzzy fallback silently rewrote the **whole file**: trailing whitespace stripped everywhere, smart quotes and unicode dashes converted to ASCII in lines the model never touched; the diff showed far more changes than requested.

**Root cause** — The replacement was applied to the fully normalized content (the matching view) and that normalized buffer was written back. Fuzzy *matching* leaked into fuzzy *writing*.

**Fix · [[pi]]** — `128330e36` 2026-06-19 (#5899): `applyReplacementsPreservingUnchangedLines` widens each replacement to the lines it touches, rewrites only those from normalized content and copies all other lines byte-for-byte from the original (`packages/coding-agent/src/core/tools/edit-diff.ts:132-173`).

**Lesson** — A normalization used to find text must never become the bytes you write; confine any rewrite to the touched span.

Related: [[fuzzy-edit-matching]] · [[search-replace-edit]] · [[edit-invisible-character-mismatch]] · [[pi--fuzzy-edit-matching|pi]]

**Fix · [[opencode]]** `236cfcbbc3` 2026-06-05 "prevent destructive edit matches (#30932)" — different root cause, same outcome (fuzzy edit destroying code the model never targeted). opencode's `BlockAnchorReplacer` matched on first and last lines only and accepted the middle at similarity ≥ **0.0** (single candidate) / ≥ 0.3 (several) since `541a7a39d3` 2025-07-24 (`541a7a39d3:packages/opencode/src/tool/edit.ts:109-110`), so any span between two matching anchor lines could be replaced. Fix: both thresholds 0.65 and an `isDisproportionateMatch` veto (span ≥ `max(old+3, 2×old)` lines or > `max(old+500, 4×old)` chars → "Refusing replacement because the matched span is much larger than oldString…") (`packages/opencode/src/tool/edit.ts:220-221,709-713,731-737`). opencode never wrote normalized text back (candidates are real file substrings). See [[opencode--fuzzy-edit-matching]]. Extra lesson: a similarity fallback needs a real floor **and** a size-proportionality veto.
