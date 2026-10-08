---
type: failure
concepts: [patch-envelope-edit, permission-ruleset, model-specific-toolset]
harnesses: [opencode]
---
**Symptom** — When GPT models edited through `apply_patch`, the edit-only machinery did not run: the edit permission filter missed it, edited events were not emitted (no LSP/snapshot refresh), and empty move paths were emitted.

**Root cause** — Edit behaviour was wired to a list of tool names (`edit`, `write`, `patch`); a second edit tool added later for one model family was not on every list.

**Fix · [[opencode]]**
- `74bd52e8a7` 2026-01-20 "ensure apply patch tool emits edited events".
- `4b4dd2b882` 2026-03-17 "Add apply_patch to EDIT_TOOLS filter (#18009)".
- `f7da00f35e` 2026-08-31 omit empty apply_patch move path.
- Residual at HEAD: `edit` takes a per-path lock, `apply_patch` and `write` do not (`packages/opencode/src/tool/edit.ts:35-45`; no `lock(` in `packages/opencode/src/tool/apply_patch.ts`) → [[concurrent-file-mutation-interleave]].

**Lesson** — Classify tools by capability (mutates files) and attach shared side effects to the capability, not to a name list.

Related: [[patch-envelope-edit]] · [[model-specific-toolset]] · [[permission-ruleset]] · [[per-file-mutation-queue]] · [[opencode--patch-envelope-edit|opencode]]
