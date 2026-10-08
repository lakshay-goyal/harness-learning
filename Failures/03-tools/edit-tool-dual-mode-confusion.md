---
type: failure
concepts: [tool-description-design, search-replace-edit, tool-argument-repair]
harnesses: [pi]
---
**Symptom** — After the edit tool gained a multi-edit mode alongside the original single `oldText/newText` mode, models produced repeated invalid edit calls and retries that mixed the two shapes.

**Root cause** — One schema offering two alternative input shapes (top-level `oldText/newText` *or* `edits[]`, with prose explaining when to use which) is itself a prompt bug: the model has to choose a mode and frequently fills both or neither.

**Fix · [[pi]]**
- `20a57e759` 2026-03-27 — multi-edit added as a dual-mode schema (`oldText` "Exact text to replace for one contiguous change…", `edits` "Use this when changing multiple separate, disjoint regions…", 5 guidelines).
- `e773527b3` 2026-03-28 (#2639), one day later — `edits[]` is the only shape (`packages/coding-agent/src/core/tools/edit.ts:21-41`); CHANGELOG `packages/coding-agent/CHANGELOG.md:2804`: "eliminating the mixed single-edit and multi-edit modes that caused repeated invalid tool calls and retries".
- `b5f425ad1` 2026-03-29 — `prepareArguments` silently folds legacy top-level `oldText/newText` into `edits` (`edit.ts:124-133`) so resumed sessions and habit-driven calls still work without re-exposing the old shape (CHANGELOG `:2780`).

**Lesson** — One canonical argument shape per tool; absorb legacy or variant shapes in code before validation, never in the public schema.

Related: [[tool-description-design]] · [[search-replace-edit]] · [[tool-argument-repair]] · [[pi--search-replace-edit|pi edit]] · [[pi--tool-argument-repair|pi repair]]
