---
type: failure
concepts: [tool-error-as-result, search-replace-edit]
harnesses: [opencode]
---
**Symptom** — The edit tool returned one combined error for "oldString not found" and "oldString found multiple times". The model could not tell which remedy applied and re-issued near-identical edits in a "doom editing loop".

**Root cause** — One error string covered two failures with opposite fixes (add exactness vs add context).

**Fix · [[opencode]]**
- `900fe5ca04` 2025-09-05 "separate edit tool error message with clearer guidance to avoid llm doom editing loop (#2051)": two errors, each naming its remedy; mirrored in `edit.txt`.
- `624dd94b5d` 2026-02-12 current wording: "Could not find oldString in the file. It must match exactly, including whitespace, indentation, and line endings." / "Found multiple matches for oldString. Provide more surrounding context to make the match unique." (`packages/opencode/src/tool/edit.ts:723-728`).
- `236cfcbbc3` 2026-06-05 adds a third distinct error for disproportionate fuzzy matches ("Re-read the file and provide the full exact oldString", `edit.ts:709-713`).

**Lesson** — One error string per remedy; the error text is the model's only debugging signal.

Related: [[tool-error-as-result]] · [[search-replace-edit]] · [[repeated-tool-call-detection]] · [[opencode--search-replace-edit|opencode]]
