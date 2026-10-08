---
type: implementation
harness: opencode
concept: fuzzy-edit-matching
commit: ecc4916b5a
files: [packages/opencode/src/tool/edit.ts:217-221, packages/opencode/src/tool/edit.ts:244-640, packages/opencode/src/tool/edit.ts:288-303, packages/opencode/src/tool/edit.ts:695-721, packages/opencode/src/tool/edit.ts:731-737, packages/core/src/tool/edit.ts:84]
---
[[fuzzy-edit-matching]] in [[opencode]].

## Mechanism
### Legacy runtime
- `type Replacer = (content, find) => Generator<string>`: each yields candidate spans that exist in the file (`packages/opencode/src/tool/edit.ts:217`).
- Order: `SimpleReplacer` → `LineTrimmedReplacer` → `BlockAnchorReplacer` → `WhitespaceNormalizedReplacer` → `IndentationFlexibleReplacer` → `EscapeNormalizedReplacer` → `TrimmedBoundaryReplacer` → `ContextAwareReplacer` → `MultiOccurrenceReplacer` (`edit.ts:695-705`; definitions `:244-640`).
- The candidate is the **file's** text (a substring found by `indexOf`), so the write replaces original bytes only inside the matched span; no normalized buffer is written back.
- `BlockAnchorReplacer`: first and last trimmed lines must match; middle lines scored by Levenshtein similarity; line-count delta allowed `max(1, floor(size × 0.25))` (`edit.ts:288-303`).
- Thresholds `SINGLE_CANDIDATE_SIMILARITY_THRESHOLD = 0.65`, `MULTIPLE_CANDIDATES_SIMILARITY_THRESHOLD = 0.65` (`edit.ts:220-221`).
- **Veto** `isDisproportionateMatch`: refuse when matched lines ≥ `max(old+3, 2×old)`, or (multi-line `oldString`) trimmed span > `max(old+500 chars, 4×old)` → "Refusing replacement because the matched span is much larger than oldString. Re-read the file…" (`edit.ts:709-713,731-737`).
- Uniqueness checked on the candidate in the real file (`index !== lastIndex` → next candidate) (`edit.ts:718-719`).
- The model is not told that a fuzzy replacer matched.
### v2 runtime
- Exact only; fuzzy port deliberately deferred (`packages/core/src/tool/edit.ts:84`; `specs/v2/schema-changelog.md:275` "Richer V1 fuzzy edit behavior remains intentionally deferred").

## Constants
| name | value | path:line |
|---|---|---|
| `SINGLE_CANDIDATE_SIMILARITY_THRESHOLD` | 0.65 (was 0.0) | `packages/opencode/src/tool/edit.ts:220` |
| `MULTIPLE_CANDIDATES_SIMILARITY_THRESHOLD` | 0.65 (was 0.3) | `packages/opencode/src/tool/edit.ts:221` |
| block line delta | `max(1, floor(n·0.25))` | `packages/opencode/src/tool/edit.ts:303` |
| veto | lines ≥ `max(n+3, 2n)`; chars > `max(c+500, 4c)` | `packages/opencode/src/tool/edit.ts:731-737` |

## Evolution
- 2025-07-24 `541a7a39d3` thresholds 0.0 (single candidate) / 0.3 (multiple) (`541a7a39d3:packages/opencode/src/tool/edit.ts:109-110`): any span between two matching anchor lines could be replaced.
- 2026-06-05 `236cfcbbc3` "fix(opencode): prevent destructive edit matches (#30932)": both 0.65, veto added, empty `oldString` rejected → [[fuzzy-edit-rewrites-untouched-lines]].
- 2026-07-20 `5a8ee27254` meta.txt adds the model-side check "Before calling `edit` with a multi-line `oldString`, compare it to `newString`: every omitted line is a deletion" → [[edit-oldstring-drops-lines]].

## Quirks / drift
- 9 replacers include similarity scoring, which pi's design space lists as "absent; riskier". opencode hit exactly that risk for ~11 months.

Contrast: pi normalizes a fixed character set and rewrites only touched lines; opencode scores similarity but never writes normalized text → [[pi--fuzzy-edit-matching|pi]].
