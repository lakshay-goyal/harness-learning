---
type: concept
stage: tool-design
tier: candidate
aliases: [edit tool, "edits[]", oldText/newText, "Found N occurrences", edit-uniqueness-requirement, multi-edit-against-original, line-ending-bom-preservation, write tool, oldString/newString, replaceAll, FileTime, "Edit applied successfully."]
harnesses: [pi, opencode]
---
File editing by exact old→new string replacement: each anchor must be unique, several disjoint replacements are matched against the original file in one call, line endings and BOM are normalized for matching and restored on write, and a diff is produced for UI/approval.

## Why
- Line-number edits drift as the file changes; whole-file rewrites waste tokens and clobber concurrent changes.
- Non-unique anchors silently edit the wrong place → must be rejected with "provide more context".
- Sequentially applied multi-edits shift offsets and make later anchors stale; overlapping edits are ambiguous.
- Invisible differences (CRLF, BOM) make exact anchors fail on otherwise correct edits ([[edit-invisible-character-mismatch]]).
- Two input shapes (single vs multi) confuse models ([[edit-tool-dual-mode-confusion]]).

## Design space
- Exact string replace, single edit per call (pi v0) vs **multi-edit array matched against original, overlap = error** (pi since 2026-03).
- Uniqueness required (pi) vs replace-all flag (absent in pi) vs line-number anchors (absent).
- Unified diff / patch input (apply_patch style; pi rejected — its Codex bridge told models "APPLY_PATCH DOES NOT EXIST", [[foreign-harness-tool-hallucination]]).
- Normalize CRLF/BOM for matching, restore on write (pi) vs byte-exact.
- Fallback fuzzy match → [[fuzzy-edit-matching]].
- Read-before-edit enforcement / staleness tracking (absent in pi) vs none.
- Write tool: overwrite with mkdir -p (pi) vs atomic temp+rename/backup (absent).
- Diff shown to model vs only to UI/approval (pi: only UI/extensions; preview computed before execution for permission prompts).
- `replaceAll` flag (opencode).
- Read-before-edit guard with mtime/size staleness (opencode until `76a141090e` 2026-04-16, then deleted without stated reason) vs none (pi).
- Swap the edit tool for a patch envelope per model family (opencode) → [[patch-envelope-edit]].
- Post-write formatting and diagnostics in the result (opencode) → [[post-edit-formatting]], [[lsp-diagnostics-feedback]].

## Implementations
- [[pi--search-replace-edit|pi]] — `edit(path, edits[{oldText,newText}])`, uniqueness counted in fuzzy-normalized space, reverse-order apply, CRLF/BOM restore, diff+patch in details; `write(path, content)`.
- [[opencode--search-replace-edit|opencode]] — `edit(oldString,newString,replaceAll)` with a 9-step fuzzy cascade, per-path lock, permission diff, format + LSP diagnostics after write; read-before-write guard deleted 2026-04-16; hidden for GPT ids (apply_patch instead); v2 exact-only with stale check.

## Failures
- [[fuzzy-edit-rewrites-untouched-lines]]
- [[edit-tool-dual-mode-confusion]]
- [[tool-arg-shape-drift]]
- [[edit-invisible-character-mismatch]]
- [[concurrent-file-mutation-interleave]]
- [[ambiguous-tool-error-causes-retry-loop]]
- [[edit-oldstring-drops-lines]]
- [[alternate-edit-tool-skips-edit-pipeline]]
- [[tool-description-drifts-from-implementation]]

## Tradeoffs
- [[edit-tool-variants]]

## Related
[[fuzzy-edit-matching]] · [[tool-argument-repair]] · [[per-file-mutation-queue]] · [[tool-description-design]] · [[path-normalization]] · [[tool-call-gate]]
