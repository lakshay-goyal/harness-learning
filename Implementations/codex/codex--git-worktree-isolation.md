---
type: implementation
harness: codex
concept: git-worktree-isolation
commit: 622e9e3696
files: [codex-rs/worktree/src/lib.rs:62, codex-rs/worktree/src/git.rs:27, codex-rs/worktree/src/metadata.rs:19, codex-rs/worktree/src/settings.rs:12, codex-rs/exec/src/cli.rs:169, codex-rs/core/src/agent/role.rs:372]
---
[[git-worktree-isolation]] in [[codex]].

## Mechanism
- **Create**: `git worktree add --detach --no-checkout <root> <sha>` with sha resolved from `base` or HEAD, then per-worktree config only — no `extensions.worktreeConfig` change on the source repo (`codex-rs/worktree/src/lib.rs:62-120`).
- **Hooks disabled**: git invoked with `core.hooksPath=/dev/null` (NUL on Windows) (`codex-rs/worktree/src/git.rs:27`, `:171`).
- **Ownership**: `codex-thread.json` version 1 binds a checkout to a thread (`codex-rs/worktree/src/metadata.rs:19-20`).
- **Cleanup**: `DEFAULT_WORKTREE_KEEP_COUNT = 15` with auto-cleanup setting; settings keys shared with the Desktop app's config (`codex-rs/worktree/src/settings.rs:12-16`, `:28-45`).
- **Entry points**: exec global `--worktree` arg (`codex-rs/exec/src/cli.rs:169`); TUI `codex-rs/tui/src/app/managed_worktree_creation.rs` and `/worktree`.
- **Not used for sub-agents**: V2 children share the parent's cwd (`codex-rs/core/src/agent/child_config.rs`, `config.cwd = turn_cwd`); the `worker` role tells the parent to assign file ownership because workers are "not alone in the codebase" (`codex-rs/core/src/agent/role.rs:372-383`) → [[no-subagent-workspace-isolation]].

## Constants
| name | value | path:line |
|---|---|---|
| `DEFAULT_WORKTREE_KEEP_COUNT` | 15 | codex-rs/worktree/src/settings.rs:16 |
| metadata file / version | `codex-thread.json` / 1 | codex-rs/worktree/src/metadata.rs:19-20 |

## Evolution
- 2026-08-25 `f832b2fe7b`, `23cedf4802` worktree crate added.

## Versus pi
- pi core has no managed worktrees; its failure [[shared-worktree-agents-clobber-each-other]] came from several sessions in one checkout.
