---
type: concept
stage: architecture
tier: must-have
aliases: [Worktree.Service, "opencode/<name> branch", "sandbox (opencode: a git worktree)", startCommand, worktree workspace adapter, codex-worktree, ManagedWorktree, WorktreeSettings, worktree-keep-count, codex-thread.json, --worktree, /worktree, managed-worktrees]
harnesses: [opencode, codex]
---
Create a separate git worktree and branch for each parallel work stream, so agents do not share a checkout.

## Why
- Parallel agents in one checkout stage, stash, reset and switch branches under each other ([[shared-worktree-agents-clobber-each-other]]).
- Worktrees share the object store, so creation is cheap compared with clones, and each stream ends as a reviewable branch.
- Gives a directory-level unit that higher layers (workspaces, session moves) can target without process isolation.
- Several agent sessions in one checkout clobber each other's files and commits ([[shared-worktree-agents-clobber-each-other]]).
- Users want to try work in parallel without stashing their own changes.
- Harness-run git commands in a fresh checkout must not trigger repo-controlled hooks.

## Design space
- **None, policy only**: standing git rules in agent instructions (pi `AGENTS.md`) vs harness-created worktrees (opencode).
- **Location**: inside the repo vs harness data dir per project (opencode `<data>/worktree/<projectID>/<name>`).
- **Branching**: new branch per worktree (`opencode/<name>`) vs detached HEAD (opencode both).
- **Setup**: bare checkout vs run a configured start command after creation (opencode `startCommand`).
- **Bookkeeping**: project tracks its worktrees as "sandboxes" (opencode; the word does not mean process isolation).
- **Layering**: worktree as one adapter of a pluggable workspace system whose other adapters may be remote ([[remote-execution-env]], opencode control plane).
- **Isolation strength**: directory only; processes, network and credentials stay shared ([[tool-only-isolation]]).
- **Isolation unit**: none / shared cwd (pi; codex sub-agents) vs **per-thread managed worktree** (codex threads) vs container/VM.
- **Creation**: `git worktree add --detach --no-checkout` at a base sha (codex) vs branch-per-worktree.
- **Repo config mutation**: per-worktree config only, no `extensions.worktreeConfig` on the source repo (codex).
- **Hooks**: inherit repo hooks vs disabled `core.hooksPath=/dev/null` (codex).
- **Ownership / cleanup**: owner metadata file + keep count with auto-cleanup (codex 15) vs manual.
- **Scope**: user threads only (codex) vs also sub-agents ([[no-subagent-workspace-isolation]]).
- (folded from `managed-worktrees`, codex framing) Harness-created git worktrees (detached, repo hooks disabled, owner metadata) so a thread can run in its own isolated checkout, with automatic cleanup beyond a keep count.

## Implementations
- [[opencode--git-worktree-isolation|opencode]] — legacy `Worktree.Service` (`git worktree add --no-checkout -b opencode/<name>`), list/remove/reset, `startCommand`; built-in `worktree` workspace adapter returns `{type: "local", directory}`.
- [[codex--git-worktree-isolation|codex]] — `codex-rs/worktree`: detached no-checkout worktrees, hooks disabled, `codex-thread.json` v1, keep count 15; exec `--worktree`, TUI `/worktree`.

## Failures
- [[shared-worktree-agents-clobber-each-other]]
- [[shared-worktree-agents-clobber-each-other]] (motivating class; codex sub-agents still share cwd)

## Related
[[remote-execution-env]] · [[location-scoped-runtime]] · [[client-server-session-split]] · [[workspace-snapshots]] · [[workspace-boundary-check]] · [[no-checkpoints-undo]] · [[opencode]]
[[os-level-sandbox]] · [[in-process-subagent-threads]] · [[spec-driven-agentic-development]] · [[no-subagent-workspace-isolation]] · [[isolation-strategy]]
