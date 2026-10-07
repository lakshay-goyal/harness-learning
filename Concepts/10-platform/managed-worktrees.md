---
type: concept
stage: architecture
tier: candidate
aliases: [codex-worktree, ManagedWorktree, WorktreeSettings, worktree-keep-count, codex-thread.json, --worktree, /worktree]
harnesses: [codex]
---
Harness-created git worktrees (detached, repo hooks disabled, owner metadata) so a thread can run in its own isolated checkout, with automatic cleanup beyond a keep count.

## Why
- Several agent sessions in one checkout clobber each other's files and commits ([[shared-worktree-agents-clobber-each-other]]).
- Users want to try work in parallel without stashing their own changes.
- Harness-run git commands in a fresh checkout must not trigger repo-controlled hooks.

## Design space
- **Isolation unit**: none / shared cwd (pi; codex sub-agents) vs **per-thread managed worktree** (codex threads) vs container/VM.
- **Creation**: `git worktree add --detach --no-checkout` at a base sha (codex) vs branch-per-worktree.
- **Repo config mutation**: per-worktree config only, no `extensions.worktreeConfig` on the source repo (codex).
- **Hooks**: inherit repo hooks vs disabled `core.hooksPath=/dev/null` (codex).
- **Ownership / cleanup**: owner metadata file + keep count with auto-cleanup (codex 15) vs manual.
- **Scope**: user threads only (codex) vs also sub-agents ([[no-subagent-workspace-isolation]]).

## Implementations
- [[codex--managed-worktrees|codex]] — `codex-rs/worktree`: detached no-checkout worktrees, hooks disabled, `codex-thread.json` v1, keep count 15; exec `--worktree`, TUI `/worktree`.

## Failures
- [[shared-worktree-agents-clobber-each-other]] (motivating class; codex sub-agents still share cwd)

## Related
[[os-level-sandbox]] · [[in-process-subagent-threads]] · [[spec-driven-agentic-development]] · [[no-subagent-workspace-isolation]] · [[isolation-strategy]]
