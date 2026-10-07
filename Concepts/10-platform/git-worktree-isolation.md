---
type: concept
stage: architecture
tier: candidate
aliases: [Worktree.Service, "opencode/<name> branch", "sandbox (opencode: a git worktree)", startCommand, worktree workspace adapter]
harnesses: [opencode]
---
Create a separate git worktree and branch for each parallel work stream, so agents do not share a checkout.

## Why
- Parallel agents in one checkout stage, stash, reset and switch branches under each other ([[shared-worktree-agents-clobber-each-other]]).
- Worktrees share the object store, so creation is cheap compared with clones, and each stream ends as a reviewable branch.
- Gives a directory-level unit that higher layers (workspaces, session moves) can target without process isolation.

## Design space
- **None, policy only**: standing git rules in agent instructions (pi `AGENTS.md`) vs harness-created worktrees (opencode).
- **Location**: inside the repo vs harness data dir per project (opencode `<data>/worktree/<projectID>/<name>`).
- **Branching**: new branch per worktree (`opencode/<name>`) vs detached HEAD (opencode both).
- **Setup**: bare checkout vs run a configured start command after creation (opencode `startCommand`).
- **Bookkeeping**: project tracks its worktrees as "sandboxes" (opencode; the word does not mean process isolation).
- **Layering**: worktree as one adapter of a pluggable workspace system whose other adapters may be remote ([[remote-execution-env]], opencode control plane).
- **Isolation strength**: directory only; processes, network and credentials stay shared ([[tool-only-isolation]]).

## Implementations
- [[opencode--git-worktree-isolation|opencode]] — legacy `Worktree.Service` (`git worktree add --no-checkout -b opencode/<name>`), list/remove/reset, `startCommand`; built-in `worktree` workspace adapter returns `{type: "local", directory}`.

## Failures
- [[shared-worktree-agents-clobber-each-other]]

## Related
[[remote-execution-env]] · [[location-scoped-runtime]] · [[client-server-session-split]] · [[workspace-snapshots]] · [[workspace-boundary-check]] · [[no-checkpoints-undo]] · [[opencode]]
