---
type: implementation
harness: codex
concept: path-normalization
commit: 622e9e3696
files: [codex-rs/apply-patch/src/parser.rs:85-91, codex-rs/apply-patch/src/invocation.rs:190-206, codex-rs/core/src/tools/handlers/view_image.rs:145-156, codex-rs/core/src/tools/handlers/shell_spec.rs:41-44]
---
[[path-normalization]] in [[codex]] — **partial**: paths are resolved against the *selected environment's* cwd via a `PathUri` abstraction, and batch edits are de-aliased; there is no quirk-normalization layer (`@`, `~`, unicode spaces, screenshot names) like pi's — none found in the tool handlers (unverified beyond these call sites).

## Mechanism
- `apply_patch` hunks: `Hunk::resolve_path(cwd) = cwd.join(path)` on `PathUri` (`codex-rs/apply-patch/src/parser.rs:85-91`); a leading `cd dir &&` in the shell form becomes `workdir`, and `effective_cwd = cwd.join(workdir)` (`codex-rs/apply-patch/src/invocation.rs:190-196`).
- Duplicate-resolved-path rejection: two hunks resolving to the same `PathUri` (e.g. `duplicate.txt` vs `./duplicate.txt`) → "multiple operations target {path}" (`invocation.rs:198-206`; `a1c88e865d`) → [[duplicate-path-ops-in-one-patch]].
- Move destination resolved against the effective `cd <worktree>` cwd, not the main repo (`316352be94` 2025-11-06, issue #5485).
- `view_image`: `turn_environment.cwd().join(path)`; failure "unable to resolve image path `{path}` against environment cwd `{cwd}`" (`codex-rs/core/src/tools/handlers/view_image.rs:145-156`).
- `exec_command.workdir` "Working directory for the command. Defaults to the turn cwd." (`codex-rs/core/src/tools/handlers/shell_spec.rs:41-44`); legacy `shell_command` description told the model "Always set the `workdir` param… Do not use `cd` unless absolutely necessary." (`916fdc2a37`, removed `8a40095ea3`).
- `PathUri` keeps environment-native paths so remote environments are not projected onto the app-server host ("`cwd` must identify an absolute environment-native path so relative patch paths can be resolved without projecting them onto the app-server or exec-server host", `invocation.rs:138-140`) → [[pluggable-tool-backends]].
- Sandbox side: writable/readable roots compared on canonical paths after symlink bugs ([[symlinked-roots-escape-sandbox-policy]]); `apply_patch` permission derivation no longer widens write roots from a target's parent ([[apply-patch-path-and-permission-hazards]]); `follow_symlinks` option on patch application (`codex-rs/apply-patch/src/lib.rs:446`).

## Evolution
- 2025-11-06 `316352be94` rename move path resolution. 2026-03-14..04-10 `9060dc7557`, `db7e02c739`, `b114781495` symlinked root fixes (07). 2026-08-10 `a1c88e865d` duplicate resolved paths. 2026-08-20 `530c1aed58` no write-permission widening.

## Versus pi
pi normalizes model/user quirks (`@`, `~`, `file://`, unicode spaces, MSYS/WSL drives, macOS screenshot variants) before resolving against `ctx.cwd` ([[pi--path-normalization]]). codex invests instead in canonical-path checks for sandbox policy and patch de-aliasing.
