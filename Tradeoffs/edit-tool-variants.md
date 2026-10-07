---
type: tradeoff
concepts: [search-replace-edit, fuzzy-edit-matching, patch-envelope-edit, model-specific-toolset, per-file-mutation-queue, tool-wire-kinds, constrained-tool-sampling]
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

## Also: [[codex]] (folded from `edit-format`)
**Axis** — how the model expresses a file change: exact old→new string pairs in a JSON tool call (search/replace) vs a self-delimiting multi-file patch language in a raw-text tool (patch envelope).

| dimension | pi — [[search-replace-edit]] | codex — [[patch-envelope-edit]] |
|---|---|---|
| tool surface | `edit(path, edits[{oldText,newText}])` + `write(path, content)` ([[pi--search-replace-edit]]) | single freeform `apply_patch`; no write tool (`codex-rs/core/src/tools/handlers/apply_patch_spec.rs:9-27`) |
| wire | JSON function args (strict-prefer constrained sampling) | Responses `custom` tool + Lark grammar (`codex-rs/core/assets/tools/apply_patch.lark`) → [[tool-wire-kinds]] |
| files per call | one | many: Add / Delete / Update / Move hunks |
| anchor | exact string, must be unique (counted in normalized space) | context lines + optional `@@ <line>` anchor; first match at/after cursor, no uniqueness check (`codex-rs/apply-patch/src/seek_sequence.rs`) |
| fuzzy fallback | NFKC, trailing ws, quotes, dashes, unicode spaces; line-preserving write ([[pi--fuzzy-edit-matching]]) | exact → trim_end → trim → punctuation/space→ASCII ("mirrors `git apply`"); no NFKC ([[codex--fuzzy-edit-matching]]) |
| multi-edit semantics | all edits matched against the original, overlap = error | hunks applied in order within a file; all files verified in memory before any write |
| atomicity | per file | per hunk sequence, no rollback across files; exact partial delta reported (`codex-rs/apply-patch/src/lib.rs:438-465`) |
| line endings / BOM | CRLF/BOM normalized for matching, restored on write | line endings preserved per line since `685270a56a`; no BOM handling found |
| format spec location | tool description + schema | grammar + per-model system prompt; 75-line prose doc deleted (`8d637ae398`) |
| model fit | any provider / model | GPT-5-family trained on the format ("Well-suited for GPT-5 models"); Bedrock needed a function fallback (`0db6811b7c`) |
| characteristic failures | [[edit-tool-dual-mode-confusion]], [[tool-arg-shape-drift]], [[edit-invisible-character-mismatch]], [[fuzzy-edit-rewrites-untouched-lines]], [[concurrent-file-mutation-interleave]] | [[patch-wrapped-in-heredoc-by-model]], [[tool-name-training-artifact]], [[patch-body-executed-as-shell]], [[edit-rewrites-line-endings]], [[duplicate-path-ops-in-one-patch]], [[partial-multi-file-patch-untracked]], [[apply-patch-path-and-permission-hazards]], [[malformed-tool-json-crashes]], [[grammar-constrained-tool-instability]] |
| cross-harness friction | Codex-trained models call `apply_patch` in pi → bridge prompt "APPLY_PATCH DOES NOT EXIST" ([[foreign-harness-tool-hallucination]]) | owns the trained name; aliases `applypatch`; still saw ~22 % of edits bypass the tool via sed/python ([[edits-bypass-patch-tool]]) |

**When each wins**
- Search/replace: provider-agnostic harnesses serving many model families; small targeted edits; when uniqueness guarantees matter more than batching; when JSON-schema-constrained decoding is the only constraint mechanism available.
- Patch envelope: a model family trained on the format; multi-file refactors, renames and creates in one call; when a grammar-constrained raw-text tool is available (no JSON escaping); when the harness can afford parser leniency, shell-side interception and partial-write accounting.
- Either way: normalize-for-matching but write original bytes outside the edit; canonicalize paths before validating a batch.
