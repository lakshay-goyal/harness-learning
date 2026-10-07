---
type: tradeoff
concepts: [search-replace-edit, fuzzy-edit-matching, patch-envelope-edit, model-specific-toolset, per-file-mutation-queue]
harnesses: [pi, opencode]
---
# edit-tool-variants

**Axis**: what edit format does the model get, and how forgiving is matching?

| dimension | pi | opencode legacy | opencode v2 | evidence |
|---|---|---|---|---|
| Edit schema | `edit {path, edits[{oldText,newText}]}`: several disjoint replacements per call, matched against the original | `edit {filePath, oldString, newString, replaceAll?}`: one replacement per call (`multiedit` removed) | exact `edit` + sequential `apply_patch` | pi `packages/coding-agent/src/core/tools/edit.ts:21-41`; legacy `packages/opencode/src/tool/edit.ts:49-54` → [[search-replace-edit]] |
| Format chosen per model | one format for all (Codex bridge said "APPLY_PATCH DOES NOT EXIST", `1650041a6`) | `apply_patch` (Codex `*** Begin Patch` envelope) **replaces** `edit`/`write` when model id has `gpt-` and not `oss`/`gpt-4` | same tool exists in Core | `packages/opencode/src/tool/registry.ts:297-300`; `b7ad6bd839` → [[patch-envelope-edit]], [[model-specific-toolset]] |
| Fuzzy matching | normalization only: NFKC, trailing whitespace, smart quotes, dashes, odd spaces; all-or-nothing switch to normalized space | 9-stage replacer cascade (line-trimmed, block-anchor with Levenshtein, indentation-flexible, escape-normalized …); thresholds 0.0/0.3 → 0.65/0.65 plus a size-proportionality veto | **none**: "Richer V1 fuzzy edit behavior remains intentionally deferred" | pi `packages/coding-agent/src/core/tools/edit-diff.ts:34-55,316-318`; legacy `packages/opencode/src/tool/edit.ts:217-221,244-640,682-737`; v2 `specs/v2/schema-changelog.md:275` → [[fuzzy-edit-matching]] |
| Destructive-match failure | — | block anchors replaced arbitrary spans; fixed `236cfcbbc3` 2026-06-05 | n/a | → [[fuzzy-edit-rewrites-untouched-lines]] |
| Concurrency | per-file mutation queue | per-path `Semaphore(1)` on `edit` only (`8bc4f91fd9`); `write`/`apply_patch` unlocked | compare-and-swap between approval and write under a per-path lock | → [[per-file-mutation-queue]], [[concurrent-file-mutation-interleave]] |
| After write | diff in result | formatter + LSP diagnostics (when configured) | — | → [[lsp-feedback-vs-none]] |
| Read-before-write rule | never | removed (`76a141090e`) | stale check instead | → [[no-read-before-write-guard]] |

**When each wins**
- **Multi-edit + light normalization (pi)**: fewer round-trips; typographic drift (quotes, dashes, NBSP) forgiven without guessing at structure. Fails loudly on real mismatch, so the model re-reads.
- **Aggressive fuzzy cascade (opencode legacy)**: weaker models that misremember indentation or whitespace succeed more often. Needs a similarity floor *and* a span-size veto, or it silently rewrites code the model never quoted.
- **Patch envelope for GPT models**: matches Codex training; multi-file add/delete/move in one call. Costs a second edit dialect to maintain and test.
- **Exact only (opencode v2)**: honest failure semantics first, fuzz later; pays in more failed edits on small models.

Related: [[minimal-vs-rich-toolset]] · [[search-replace-edit]] · [[pi]] · [[opencode]] · [[Tradeoffs]]
