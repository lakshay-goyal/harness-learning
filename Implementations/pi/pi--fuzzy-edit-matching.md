---
type: implementation
harness: pi
concept: fuzzy-edit-matching
commit: b30a6dd77
files: [packages/coding-agent/src/core/tools/edit-diff.ts:34, packages/coding-agent/src/core/tools/edit-diff.ts:132, packages/coding-agent/src/core/tools/edit-diff.ts:207, packages/coding-agent/src/core/tools/edit-diff.ts:247, packages/coding-agent/src/core/tools/edit-diff.ts:300, packages/durable/src/tools/edit-diff.ts:203]
---
[[fuzzy-edit-matching]] in [[pi]].

Fallback layer under the exact-match `edit` tool ([[pi--search-replace-edit|search-replace-edit]]): exact `indexOf` first; on miss, match in a normalized view; write back only the lines the replacement touched.

## Mechanism
- **Pre-normalization (always, not "fuzzy")**: BOM stripped (`edit.ts:191`), CRLF/lone CR → LF on file and on every `oldText/newText` (`edit-diff.ts:19-21,305-308`). These are restored on write (`edit.ts:197`).
- **`normalizeForFuzzyMatch(text)`** (`packages/coding-agent/src/core/tools/edit-diff.ts:34-55`), applied in order:
  1. `.normalize("NFKC")` — compatibility forms (full-width ＡＢＣ１２３, full-width CJK punctuation `，（）`, composed vs decomposed `é`) (`700bcf345` #2044).
  2. per line `trimEnd()` — trailing whitespace ignored.
  3. smart single quotes U+2018/2019/201A/201B → `'`.
  4. smart double quotes U+201C/201D/201E/201F → `"`.
  5. dashes U+2010 hyphen, U+2011 NB hyphen, U+2012 figure, U+2013 en, U+2014 em, U+2015 bar, U+2212 minus → `-`.
  6. spaces U+00A0 NBSP, U+2002–200A, U+202F narrow NBSP, U+205F, U+3000 ideographic → ASCII space.
- **`fuzzyFindText(content, oldText)`** (`:207-245`): exact `content.indexOf(oldText)` → `{usedFuzzyMatch:false}`; else normalize both and `indexOf` in normalized space; offsets returned are in normalized space with `contentForReplacement = fuzzyContent`.
- **All-or-nothing switch** (`applyEditsToNormalizedContent`, `:316-318`): run `fuzzyFindText` for every edit; if **any** needed fuzzy, `replacementBaseContent = normalizeForFuzzyMatch(whole file)` and **all** edits are re-matched against it (so offsets for every edit share one coordinate space).
- **Uniqueness always in fuzzy space**: `countOccurrences` normalizes both sides regardless of whether the exact path matched (`:247-251`, used `:328-331`) → error "Found N occurrences…".
- **Overlap check** after sorting by offset, in the same space (`:341-350`).
- **Write-back: `applyReplacementsPreservingUnchangedLines(original, base, replacements)`** (`:132-173`):
  - split original into lines with endings; compute line spans of normalized base; require equal line count else throw "Cannot preserve unchanged lines because the base content has a different line count." (`:137-141`).
  - each replacement widened to the base lines it touches (`getReplacementLineRange` `:84-109`); overlapping line ranges merged into groups (`:143-154`).
  - result = original lines before group (byte-for-byte) + group lines from **normalized base** with replacements applied + … + original tail. "The actual replacement ranges drive preservation so duplicate normalized lines cannot be aligned to the wrong occurrence." (`:122-131`).
- Exact-only edits never touch normalization: `applyReplacements` on the LF-normalized original (`:353-355`).

## Constants
| name | value | path:line |
|---|---|---|
| Unicode normalization form | NFKC | `packages/coding-agent/src/core/tools/edit-diff.ts:37` |
| dash set | U+2010–2015, U+2212 | `edit-diff.ts:49` |
| space set | U+00A0, U+2002–200A, U+202F, U+205F, U+3000 | `edit-diff.ts:53` |
| quote sets | U+2018–201B → `'`; U+201C–201F → `"` | `edit-diff.ts:43,45` |

## Evolution
- 2025-12-29/30 `c214a3340` (#360) / `8c43a9fbc` (#355): CRLF normalize+restore (first "invisible char" fix).
- 2026-01-02 `d9adf659c`: BOM strip/restore.
- 2026-01-14 `0c135d014` (#713, external contributor Danila Poyarkov): `fuzzyFindText` + `normalizeForFuzzyMatch` (trailing ws, quotes, dashes, special spaces); fuzzy result written by applying the replacement to the **fully normalized file**.
- 2026-03-15 `700bcf345` (#2044): `.normalize("NFKC")` prepended (tests: Chinese full-width punctuation, `ＡＢＣ１２３`, `café`).
- 2026-06-19 `128330e36` (#5899, Armin Ronacher): "preserve untouched lines in fuzzy edit" — before, any fuzzy match rewrote the **whole file** through normalized content (trailing whitespace stripped and smart quotes/dashes converted everywhere) → [[fuzzy-edit-rewrites-untouched-lines]].
- 2026-08-19 `1355cd36e` (#8337): BOM normalization extended to text inputs.
- 2026-09-29 `445770e03`: same algorithm copied into `packages/durable/src/tools/edit-diff.ts:203-244`.

## Evidence commits
`c214a3340` `8c43a9fbc` `d9adf659c` `0c135d014` `700bcf345` `128330e36` `1355cd36e` `445770e03`

## Quirks
- Touched lines are still rewritten from normalized text: on a line where the replacement lands, unrelated trailing whitespace / smart quotes / NFKC-changed chars on that same line are normalized too (by construction, `:160-167`).
- Exactly-unique `oldText` can be rejected as duplicate when another region differs only by normalized chars (uniqueness counted in fuzzy space).
- Error text on miss still claims "must match exactly including all whitespace and newlines" (`:253-262`) — understates the fallback.
- No indentation-insensitive matching (leading whitespace is not trimmed), no line-number anchors, no similarity/Levenshtein matching (observed absence).
- If one of several edits needs fuzzy, the exact-matched edits are also re-located in normalized space; their written lines also become normalized (consequence of `:316-318` + line overlay).

## Durable variant (packages/durable)
- `packages/durable/src/tools/edit-diff.ts:203-244` exact-then-fuzzy, same normalization; `edit.ts:116-125` strip BOM → detect CRLF → apply → restore. Identical behavior except BOM helper: diffing the two `edit-diff.ts` files shows only fs imports and a local `stripBom` (durable `edit-diff.ts:243-246`) differ in the matching code (verified at HEAD).

## Failures
- [[edit-invisible-character-mismatch]]
- [[fuzzy-edit-rewrites-untouched-lines]]
