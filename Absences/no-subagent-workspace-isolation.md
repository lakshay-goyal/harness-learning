---
type: absence
harnesses: [codex]
---
# no-subagent-workspace-isolation

Sub-agents share the parent's cwd/checkout; no per-child worktree even though managed worktrees exist for threads.

**What's missing**
- `codex-rs/core/src/agent/child_config.rs` (`config.cwd = turn_cwd`); `codex-rs/worktree` used only by exec/TUI threads ([[git-worktree-isolation]]).

**Evidence of decision**
- Mitigation is prompt-level: the `worker` role tells the parent to assign file ownership because workers are "not alone in the codebase" (`codex-rs/core/src/agent/role.rs:372-383`).

**Implication**
- Concurrent children can clobber each other's edits — the class of [[shared-worktree-agents-clobber-each-other]]; isolation is by OS sandbox only, not by checkout.

Related: [[git-worktree-isolation]] · [[in-process-subagent-threads]] · [[shared-worktree-agents-clobber-each-other]] · [[agent-profiles]] · [[Absences]]
