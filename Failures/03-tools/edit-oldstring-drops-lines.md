---
type: failure
concepts: [search-replace-edit, fuzzy-edit-matching, per-model-system-prompt]
harnesses: [opencode]
---
**Symptom** — For multi-line edits the model wrote a `newString` that silently omitted lines present in `oldString`, deleting code it meant to keep.

**Root cause** — The model rewrote a block from memory instead of diffing old vs new; the edit tool applies exactly what it is given.

**Fix · [[opencode]]**
- Model-side rule, `5a8ee27254` 2026-07-20 (Meta prompt rewrite): "Before calling `edit` with a multi-line `oldString`, compare it to `newString`: every omitted line is a deletion" and re-read the region after edits with preservation constraints (`packages/opencode/src/session/prompt/meta.txt:33-34`).
- Tool-side guard for the related fuzzy case: `236cfcbbc3` 2026-06-05 refuses matches much larger than `oldString` → [[fuzzy-edit-rewrites-untouched-lines]]. No guard exists for an exact match with a shorter `newString` (by design).

**Lesson** — Large replace edits need either a model-side diff check or a harness-side deletion warning.

Related: [[search-replace-edit]] · [[fuzzy-edit-matching]] · [[per-model-system-prompt]] · [[opencode--fuzzy-edit-matching|opencode]]
