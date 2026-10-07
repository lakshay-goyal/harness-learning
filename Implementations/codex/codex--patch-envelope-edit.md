---
type: implementation
harness: codex
concept: patch-envelope-edit
commit: 622e9e3696
files: [codex-rs/core/assets/tools/apply_patch.lark:1-19, codex-rs/core/src/tools/handlers/apply_patch_spec.rs:9-27, codex-rs/core/src/tools/handlers/apply_patch.rs:61, codex-rs/core/src/tools/handlers/apply_patch.rs:286-416, codex-rs/apply-patch/src/parser.rs:37-53, codex-rs/apply-patch/src/parser.rs:225-252, codex-rs/apply-patch/src/invocation.rs:141-252, codex-rs/apply-patch/src/invocation.rs:254-403, codex-rs/apply-patch/src/lib.rs:52-57, codex-rs/apply-patch/src/lib.rs:438-465, codex-rs/core/src/tools/runtimes/apply_patch.rs:1-5, codex-rs/core/src/tools/spec_plan.rs:1344-1347]
---
[[patch-envelope-edit]] in [[codex]].

`apply_patch` is codex's **only** file-mutation tool (no write/edit tool; [[minimal-default-toolset]], [[dedicated-vs-shell-tools]]). Freeform (Responses "custom") tool constrained by a Lark grammar; patch parsed by the `codex-rs/apply-patch` crate, verified in memory, applied through the environment filesystem under approval + sandbox.

## Mechanism

### Wire / model-facing
- Spec `create_apply_patch_freeform_tool` (`codex-rs/core/src/tools/handlers/apply_patch_spec.rs:9-27`): `ToolSpec::Freeform{name:"apply_patch", format:{type:"grammar", syntax:"lark", definition}}` ([[tool-wire-kinds]]). Comment: "Well-suited for GPT-5 models" (`:7`).
- **Description is one line**: "The `apply_patch` tool can be used to edit files. This is a FREEFORM tool, so do not wrap the patch in JSON." (`apply_patch_spec.rs:20`). Format knowledge = grammar + per-model system prompt ([[per-model-system-prompt]]), not prose in the tool ([[tool-description-design]]).
- Grammar `codex-rs/core/assets/tools/apply_patch.lark:1-19`:
  - `start: begin_patch hunk+ end_patch`; `"*** Begin Patch" LF` … `"*** End Patch" LF?`
  - `add_hunk: "*** Add File: " filename LF add_line+` (every line `+…`)
  - `delete_hunk: "*** Delete File: " filename LF`
  - `update_hunk: "*** Update File: " filename LF change_move? change?` — `change_move: "*** Move to: " filename LF`; `change` optional ⇒ rename-only update allowed
  - `change: (change_context | change_line)+ eof_line?`; `change_context: ("@@" | "@@ " text) LF`; `change_line: ("+" | "-" | " ") text LF`; `eof_line: "*** End of File" LF`.
- Multi-environment turns: grammar rewritten to `start: begin_patch environment_id? hunk+ end_patch` with `environment_id: "*** Environment ID: " filename LF` (`apply_patch_spec.rs:10-16`) — environment selection travels **inside** the patch ([[pluggable-tool-backends]]).
- Gate: environment attached && `model_info.apply_patch_tool_type.is_some()` (`codex-rs/core/src/tools/spec_plan.rs:1344-1347`); `ApplyPatchToolType` has a single variant `Freeform` (`codex-rs/protocol/src/openai_models.rs:324-326`).
- Handler accepts only `ToolPayload::Custom` (`codex-rs/core/src/tools/handlers/apply_patch.rs:389-391`); not parallel-safe (default false → exclusive write lock, [[parallel-tool-execution]]).

### Parse (lenient for every model)
- Markers `codex-rs/apply-patch/src/parser.rs:37-45`; `PARSE_IN_STRICT_MODE = false` — comment: "the only OpenAI model that knowingly requires lenient parsing is gpt-4.1 … we resign ourselves allowing lenient parsing for all models" (`parser.rs:47-53`).
- `ParseMode::Lenient` strips `<<EOF` / `<<'EOF'` / `<<"EOF"` heredoc wrappers around the patch (`parser.rs:165-190,225-252`) → [[patch-wrapped-in-heredoc-by-model]], [[tool-argument-repair]].
- Paths: `Hunk::resolve_path` = `cwd.join(path)` on a `PathUri` (`parser.rs:85-91`); `cd dir &&` prefix becomes `workdir` → effective cwd (`invocation.rs:192-196`) — [[path-normalization]].

### Verify before any write
- `verify_apply_patch_args` computes full new content + unified diff for **every** file in memory (`codex-rs/apply-patch/src/invocation.rs:168-252`); context located by `seek_sequence` ([[fuzzy-edit-matching]]).
- Two hunks resolving to the same path → `"multiple operations target {path}"` (`invocation.rs:198-206`; `a1c88e865d`) — [[duplicate-path-ops-in-one-patch]].
- Any miss → `"apply_patch verification failed: {err}"` returned to model, **no file touched** (`codex-rs/core/src/tools/handlers/apply_patch.rs:322-374`); other texts: "failed to parse apply_patch: {err}" (`:136`), "apply_patch handler received invalid patch input" (`:374`), "apply_patch is unavailable in this session" (`:337`), "failed to prepare patch permissions" / "failed to check patch permissions" (`:511,516`).

### Apply
- Runtime goes through the tool orchestrator (approval + sandbox) using `ExecutorFileSystem` with an explicit sandbox context — local or remote (`codex-rs/core/src/tools/runtimes/apply_patch.rs:1-5,170-181`). Safety decision `assess_patch_safety`: empty patch rejected; UnlessTrusted → ask; confined to writable roots + sandbox available → auto-approve **but still run sandboxed** because "paths in the patch are hard links to files outside the writable roots" (`codex-rs/core/src/safety.rs:67-125`) — see [[tool-call-gate]], [[os-level-sandbox]].
- `apply_hunks_to_files` writes **sequentially, no rollback**; `ApplyPatchOptions{follow_symlinks}`; empty hunk list → "No files were modified."; any failed write sets `delta.exact = false` ("a failed write can still have modified the target … truncating before ENOSPC") (`codex-rs/apply-patch/src/lib.rs:438-465`; `9b6c6f7a01`) — [[partial-multi-file-patch-untracked]].
- Committed `AppliedPatchDelta` feeds the operation-backed turn diff (no filesystem re-read) → [[file-op-tracking]] (`f7e8ff8e50`, W4/W6 note).
- Line endings: always preserved — original endings kept on untouched/context lines, inserted lines use the file's first line ending; env var `CODEX_APPLY_PATCH_PRESERVE_LINE_ENDINGS` forced to `1` for older standalone executables (`lib.rs:54-57`; `21aa552e87`, `685270a56a`) — [[edit-rewrites-line-endings]].
- Output to model is plain text (JSON-structured shell/patch output removed `82061660ae` 2026-05-18: "Current shell and apply_patch responses are already plain text for model consumption").

### Shell-side interception (the model's old habit still works)
- `exec_command` whose argv is `apply_patch <<'EOF' … EOF` under bash/zsh/sh `-lc`/`-c` (tree-sitter-bash query, optional leading `cd dir &&`) is detected and routed to the apply_patch path instead of executed (`codex-rs/apply-patch/src/invocation.rs:64,254-403`; `codex-rs/core/src/tools/handlers/unified_exec/exec_command.rs:372`).
- Accepted command names `APPLY_PATCH_COMMANDS = ["apply_patch", "applypatch"]` (`invocation.rs:27,309`; `5f8984aa7d`, `1a1516a80b`) → [[tool-name-training-artifact]].
- A raw patch passed as the whole command or as a shell script body → `ApplyPatchError::ImplicitInvocation`: `patch detected without explicit call to apply_patch. Rerun as ["apply_patch", "<patch>"]` — never executed (`invocation.rs:141-157`; `codex-rs/apply-patch/src/lib.rs:86-90`; `5f6e95b592`) → [[patch-body-executed-as-shell]].
- The codex binary itself acts as `apply_patch`: argv1 `--codex-run-as-apply-patch` (`codex-rs/apply-patch/src/lib.rs:52`); arg0 dispatch also maps the misspelled `applypatch`, via a per-session temp PATH dir of symlinks (`codex-rs/arg0/src/lib.rs:21-25,327-340`).

### Streaming preview & hooks
- `ApplyPatchArgumentDiffConsumer` incrementally parses the streaming custom-tool input (`StreamingPatchParser`, `codex-rs/apply-patch/src/streaming_parser.rs`) and emits `PatchApplyUpdatedEvent` at most every `APPLY_PATCH_ARGUMENT_DIFF_BUFFER_INTERVAL = 500 ms` under `Feature::ApplyPatchStreamingEvents` (`apply_patch.rs:61,77-125`; `7995c66032` 2026-04-16).
- PreToolUse hooks see `{command: <patch>}` under hook name `apply_patch` and may rewrite the patch (`apply_patch.rs:397-428`) → [[tool-call-gate]], [[tool-result-rewriting]].

### Prompt-side rules (per-model instructions)
- "Use the `apply_patch` tool to edit files (NEVER try `applypatch` or `apply-patch`, only `apply_patch`)" (`81b148bda2`, `codex-rs/protocol/src/prompts/base_instructions/default.md:132`).
- "Do not waste tokens by re-reading files after calling `apply_patch` on them. The tool call will fail if it didn't work." (`default.md:143`).
- gpt-5-codex: "Try to use apply_patch for single file edits, but it is fine to explore other options … Do not use apply_patch for changes that are auto-generated … or when scripting is more efficient" (`codex-rs/core/gpt_5_codex_prompt.md:11`; `0ad1b0782b`) — [[edits-bypass-patch-tool]].
- Bundled catalog text: "Use `apply_patch` for manual code edits. Do not create or edit files with `cat` or other shell write tricks" (`codex-rs/models-manager/models.json:1408`).

## Constants
| name | value | path:line |
|---|---|---|
| `PARSE_IN_STRICT_MODE` | `false` (lenient for all models) | `codex-rs/apply-patch/src/parser.rs:53` |
| `APPLY_PATCH_ARGUMENT_DIFF_BUFFER_INTERVAL` | 500 ms | `codex-rs/core/src/tools/handlers/apply_patch.rs:61` |
| `APPLY_PATCH_COMMANDS` | `apply_patch`, `applypatch` | `codex-rs/apply-patch/src/invocation.rs:27` |
| `CODEX_CORE_APPLY_PATCH_ARG1` | `--codex-run-as-apply-patch` | `codex-rs/apply-patch/src/lib.rs:52` |
| `CODEX_APPLY_PATCH_PRESERVE_LINE_ENDINGS` | forced `1` for old executables | `codex-rs/apply-patch/src/lib.rs:56-57` |
| tool description length | 1 sentence (+ grammar) | `codex-rs/core/src/tools/handlers/apply_patch_spec.rs:20` |

## Evolution
- 2025-04-24 `31d0d7a305` apply-patch crate born with the Rust import; TS-era prompt carried the patch grammar inline.
- 2025-04-25 `15bf5ca971` unicode punctuation seek pass ([[edit-invisible-character-mismatch]]).
- 2025-06-03 `6fcc528a43` lenient mode for GPT-4.1 heredocs ("the error is predictable").
- 2025-08-04 `a6139aa003` restore literal `*** Begin Patch` markers that prettier mangled; 2025-08-21 `e4c275d615` grammar removed from `prompt.md`, shipped only with the tool definition ([[formatter-corrupts-prompt-file]]).
- 2025-08-07 `81b148bda2` "NEVER try `applypatch` or `apply-patch`" prompt rule; 2025-08-11 `5f8984aa7d` / 2025-08-20 `1a1516a80b` accept `applypatch`.
- 2025-08-22 `236c4f76a6` freeform (grammar) apply_patch → disabled by default 2025-08-24 `4157788310` ("issues"); grammar fix 2025-09-02 `6f75114695` ([[grammar-constrained-tool-instability]]).
- 2025-09-12 `5f6e95b592` refuse to execute patch bodies as shell.
- 2025-09-30 `f6a152848a` "you MUST use apply_patch" (0 % bypass in eval) → reverted 2025-10-01 `5d78c1edd3` → permissive carve-outs 2025-10-04 `0ad1b0782b`.
- 2025-10-04 `4764fc1ee7` "Freeform apply_patch with simple shell output" (description "This is a FREEFORM tool, so do not wrap the patch in JSON").
- 2025-11-06 `316352be94` move destination resolved against `cd <worktree>` effective cwd (#5485).
- Before 2026-02-09 (`a1abd53b6a`): models without native apply_patch got `BASE_INSTRUCTIONS_WITH_APPLY_PATCH` = base prompt + grammar (`a1abd53b6a^:codex-rs/core/src/models_manager/model_info.rs:20-21`).
- 2026-03-27 `307e427a9b` no redundant write roots; 2026-08-20 `530c1aed58` prevent widening write permissions ([[apply-patch-path-and-permission-hazards]]).
- 2026-04-16 `7995c66032` streaming patch events. 2026-04-24 `0db6811b7c` function-style apply_patch for Bedrock (Bedrock rejected `custom` tools; [[endpoint-rejects-request-field]]; current Bedrock handling after the JSON variant's deletion unverified).
- 2026-05-07 `9b6c6f7a01` exact deltas after partial failure. 2026-05-08 `cce059467a` freeform on by default; same day `e783341b70` deleted function-style JSON apply_patch ("left another way for models and tests to invoke `apply_patch`, which made the tool surface harder to reason about").
- 2026-05-18 `82061660ae` legacy JSON shell/patch output formatting removed.
- 2026-08-10 `21aa552e87` line-ending preservation opt-in; `a1c88e865d` reject duplicate resolved paths. 2026-10-05 `685270a56a` preservation unconditional.
- 2026-08-13 `8d637ae398` deleted the 75-line `apply_patch_tool_instructions.md` ("Use the `apply_patch` shell command to edit files. Your patch language is a stripped‑down, file‑oriented diff format…", `8d637ae398^:codex-rs/prompts/templates/apply_patch_tool_instructions.md`) as unused.

## Quirks
- No standalone format doc any more: a provider without custom-tool grammar support gets no patch spec from the tool itself ([[no-standalone-patch-format-doc]]).
- Parser leniency is global, keyed to one model (gpt-4.1) that is no longer in the catalog.
- Two invocation paths (freeform tool, intercepted shell heredoc) converge on the same verification; the shell path still exists after the JSON path was deleted "to simplify the surface".
- Unrelated to the model tool: `codex-rs/git-utils/src/apply.rs:1-7` wraps system `git apply` (temp diff file, dry-run preflight via `ApplyGitRequest::preflight`) for client features; the old `git-apply` crate was merged into git-utils `fa92cd92fa` 2025-10-29.
- Not transactional across files; the harness reports what landed instead of rolling back ([[no-transactional-multi-file-patch]]).

## Versus pi
pi edits with JSON `edit(path, edits[{oldText,newText}])` + `write` ([[pi--search-replace-edit]]); its Codex bridge once told models "APPLY_PATCH DOES NOT EXIST" ([[foreign-harness-tool-hallucination]]). codex owns the trained name and aliases its misspellings. Axis: [[edit-format]].

## Failures
[[patch-wrapped-in-heredoc-by-model]] · [[tool-name-training-artifact]] · [[patch-body-executed-as-shell]] · [[edit-rewrites-line-endings]] · [[partial-multi-file-patch-untracked]] · [[duplicate-path-ops-in-one-patch]] · [[apply-patch-path-and-permission-hazards]] · [[malformed-tool-json-crashes]] · [[grammar-constrained-tool-instability]] · [[edits-bypass-patch-tool]] · [[formatter-corrupts-prompt-file]]
