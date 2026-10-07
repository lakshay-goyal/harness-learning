---
type: concept
stage: tools
tier: must-have
aliases: [apply_patch, "*** Begin Patch", "*** Update File:", seekSequence, Codex patch format, patch tool, ApplyPatchHandler, codex_apply_patch, verify_apply_patch_args, intercept_apply_patch, freeform apply_patch, apply_patch.lark, patch DSL]
harnesses: [opencode, codex]
---
A model-specific edit tool. It takes a `*** Begin Patch` envelope that adds, updates, deletes or moves several files, and applies each hunk by seeking its context lines.
Edit files through one self-delimiting, multi-file patch language (Add / Delete / Update [+ Move] hunks with `@@` context anchors and `+`/`-`/` ` lines) that the harness parses, verifies against current files entirely in memory, then applies — instead of a search/replace or write tool.

- Models trained inside Codex-style harnesses emit this envelope natively. Given only `edit(oldString,newString)` they still reach for `apply_patch` ([[foreign-harness-tool-hallucination]]).
- One call can touch many files and create/delete/move them, so there are fewer round trips than per-file string replacement.
- Context-seeking hunks tolerate small drift (trailing whitespace, unicode punctuation) without needing a unique full anchor.
## Why
- One call can create, delete, rename and edit many files; context lines make anchors robust to line-number drift (contrast [[search-replace-edit]]).
- A free-text DSL inside a JSON string needs escaping and invites malformed arguments ([[malformed-tool-json-crashes]]); a grammar-constrained raw-text ("custom") tool removes the escaping layer ([[tool-wire-kinds]], [[constrained-tool-sampling]]).
- Models trained on the format emit it everywhere — wrapped in heredocs ([[patch-wrapped-in-heredoc-by-model]]), under misspelled names ([[tool-name-training-artifact]]), as a shell script body ([[patch-body-executed-as-shell]]) — so the harness must sniff and route it.
- A patch is a mini-transaction: aliasing paths, partial multi-file failure and byte rewriting outside hunks are its specific hazards ([[duplicate-path-ops-in-one-patch]], [[partial-multi-file-patch-untracked]], [[edit-rewrites-line-endings]], [[apply-patch-path-and-permission-hazards]]).

- **Envelope parser + per-hunk context seek with escalating normalization** (opencode legacy: exact → `trimEnd` → `trim` → unicode-punctuation-to-ASCII).
- Alternative to string-replace edit, never both on one request (opencode legacy, chosen by model id → [[model-specific-toolset]]) vs alongside it.
- Atomicity: preflight every target then commit sequentially, earlier ops stay applied on a later failure, no moves, no rollback (opencode v2) vs all-or-nothing.
- Permission: one ask for the whole patch (opencode) vs per file.
- JSON function tool carrying the patch string (opencode) vs freeform/grammar tool input (Codex upstream; [[constrained-tool-sampling]]).
- Reject it and tell the model it doesn't exist (pi's short-lived Codex bridge, [[foreign-harness-tool-hallucination]]).
## Design space
- Patch format: own stripped-down envelope (codex `*** Begin Patch … *** End Patch`) vs unified diff / `git apply` vs search/replace blocks ([[search-replace-edit]], pi) → [[edit-tool-variants]].
- Transport: JSON function argument (codex until `e783341b70` 2026-05-08) vs **freeform tool with Lark grammar** (codex) vs a shell command `apply_patch <<'EOF'` intercepted by the harness (codex, still accepted).
- Format spec location: long prose in the system prompt / a tool-instructions file (codex until `8d637ae398` 2026-08-13) vs **grammar only + one-line description** (codex now) — [[tool-description-design]].
- Parse strictness: strict vs **lenient (strip heredoc wrappers) for all models** (codex `PARSE_IN_STRICT_MODE = false`).
- Verify-all-then-write (codex: every hunk resolved in memory first) vs apply-as-you-go.
- Atomicity: transactional multi-file write with rollback (absent in codex; observed, not stated as a decision — `codex-rs/apply-patch/src/lib.rs:438-465`) vs sequential writes + exact report of the committed prefix (codex).
- Context matching: exact vs normalized fallback → [[fuzzy-edit-matching]].
- Line-ending policy: normalize to LF vs **preserve per line** (codex since `685270a56a`).
- Safety: patch runs through approval + sandbox like a command; writable-root check per path ([[tool-call-gate]], [[os-level-sandbox]]).
- Streaming preview of the patch while the model is still writing it (codex `ApplyPatchStreamingEvents`).

- [[opencode--patch-envelope-edit|opencode]] — `apply_patch` exposed only to `gpt-*` (not `oss`, not `gpt-4`) ids in place of edit/write; 4-pass `seekSequence`; format + LSP diagnostics per file.
## Implementations
- [[codex--patch-envelope-edit|codex]] — freeform `apply_patch` (Lark grammar, one-line description), lenient parser, in-memory verification + unified diffs, sequential apply via environment filesystem with orchestrator approval/sandbox, shell-heredoc interception, line endings preserved, streaming diff events.

- [[alternate-edit-tool-skips-edit-pipeline]]
- [[tool-description-drifts-from-implementation]]
- [[prompt-names-unavailable-tools]]
## Failures
- [[patch-wrapped-in-heredoc-by-model]]
- [[tool-name-training-artifact]]
- [[patch-body-executed-as-shell]]
- [[edit-rewrites-line-endings]]
- [[partial-multi-file-patch-untracked]]
- [[duplicate-path-ops-in-one-patch]]
- [[apply-patch-path-and-permission-hazards]]
- (02) [[malformed-tool-json-crashes]] · [[grammar-constrained-tool-instability]] · [[endpoint-rejects-request-field]]
- (04) [[edits-bypass-patch-tool]] · [[formatter-corrupts-prompt-file]]
- [[edit-invisible-character-mismatch]] (03-tools) — Edit failed with "Could not find the exact text" although the model's oldText was visually identical to the…

[[search-replace-edit]] · [[fuzzy-edit-matching]] · [[model-specific-toolset]] · [[per-model-system-prompt]] · [[foreign-harness-tool-hallucination]] · [[lsp-diagnostics-feedback]] · [[post-edit-formatting]]
## Related
[[search-replace-edit]] · [[fuzzy-edit-matching]] · [[tool-wire-kinds]] · [[constrained-tool-sampling]] · [[tool-argument-repair]] · [[shell-execution]] · [[file-op-tracking]] · [[path-normalization]] · [[tool-call-gate]] · [[edit-tool-variants]] · [[foreign-harness-tool-hallucination]] · [[minimal-vs-rich-toolset]] · [[no-transactional-multi-file-patch]] · [[no-standalone-patch-format-doc]]

## Tradeoffs
- [[edit-tool-variants]]
