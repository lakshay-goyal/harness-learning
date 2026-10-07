---
type: concept
stage: tools
tier: candidate
aliases: [apply_patch, "*** Begin Patch", "*** Update File:", seekSequence, Codex patch format, patch tool]
harnesses: [opencode]
---
A model-specific edit tool. It takes a `*** Begin Patch` envelope that adds, updates, deletes or moves several files, and applies each hunk by seeking its context lines.

## Why
- Models trained inside Codex-style harnesses emit this envelope natively. Given only `edit(oldString,newString)` they still reach for `apply_patch` ([[foreign-harness-tool-hallucination]]).
- One call can touch many files and create/delete/move them, so there are fewer round trips than per-file string replacement.
- Context-seeking hunks tolerate small drift (trailing whitespace, unicode punctuation) without needing a unique full anchor.

## Design space
- **Envelope parser + per-hunk context seek with escalating normalization** (opencode legacy: exact → `trimEnd` → `trim` → unicode-punctuation-to-ASCII).
- Alternative to string-replace edit, never both on one request (opencode legacy, chosen by model id → [[model-specific-toolset]]) vs alongside it.
- Atomicity: preflight every target then commit sequentially, earlier ops stay applied on a later failure, no moves, no rollback (opencode v2) vs all-or-nothing.
- Permission: one ask for the whole patch (opencode) vs per file.
- JSON function tool carrying the patch string (opencode) vs freeform/grammar tool input (Codex upstream; [[constrained-tool-sampling]]).
- Reject it and tell the model it doesn't exist (pi's short-lived Codex bridge, [[foreign-harness-tool-hallucination]]).

## Implementations
- [[opencode--patch-envelope-edit|opencode]] — `apply_patch` exposed only to `gpt-*` (not `oss`, not `gpt-4`) ids in place of edit/write; 4-pass `seekSequence`; format + LSP diagnostics per file.

## Failures
- [[alternate-edit-tool-skips-edit-pipeline]]
- [[tool-description-drifts-from-implementation]]
- [[prompt-names-unavailable-tools]]

## Tradeoffs
- [[edit-tool-variants]]

## Related
[[search-replace-edit]] · [[fuzzy-edit-matching]] · [[model-specific-toolset]] · [[per-model-system-prompt]] · [[foreign-harness-tool-hallucination]] · [[lsp-diagnostics-feedback]] · [[post-edit-formatting]]
