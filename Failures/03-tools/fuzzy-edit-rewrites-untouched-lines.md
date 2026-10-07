---
type: failure
concepts: [fuzzy-edit-matching, search-replace-edit]
harnesses: [pi]
---
**Symptom** — A single edit that needed the fuzzy fallback silently rewrote the **whole file**: trailing whitespace stripped everywhere, smart quotes and unicode dashes converted to ASCII in lines the model never touched; the diff showed far more changes than requested.

**Root cause** — The replacement was applied to the fully normalized content (the matching view) and that normalized buffer was written back. Fuzzy *matching* leaked into fuzzy *writing*.

**Fix · [[pi]]** — `128330e36` 2026-06-19 (#5899): `applyReplacementsPreservingUnchangedLines` widens each replacement to the lines it touches, rewrites only those from normalized content and copies all other lines byte-for-byte from the original (`packages/coding-agent/src/core/tools/edit-diff.ts:132-173`).

**Lesson** — A normalization used to find text must never become the bytes you write; confine any rewrite to the touched span.

Related: [[fuzzy-edit-matching]] · [[search-replace-edit]] · [[edit-invisible-character-mismatch]] · [[pi--fuzzy-edit-matching|pi]]
