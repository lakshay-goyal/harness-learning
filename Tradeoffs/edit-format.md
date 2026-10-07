---
type: tradeoff
concepts: [search-replace-edit, patch-envelope-edit, fuzzy-edit-matching, tool-wire-kinds, constrained-tool-sampling]
---
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
