---
type: tradeoff
concepts: [workspace-snapshots, session-tree, session-fork]
harnesses: [pi, opencode]
---
# undo-vs-none

**Axis**: can the user roll file changes back to a point in the conversation, or is recovery left to git?

| option | pi | opencode | evidence |
|---|---|---|---|
| No file undo; conversation branching only | ✅ `/tree` and `/fork` never restore files | — | pi: "Use snapshots, backups, or version control before substantial changes." (`packages/coding-agent/docs/security.md:89`) → [[no-checkpoints-undo]] |
| Opt-in git stash per turn | ✅ example `git-checkpoint.ts` (offers restore on fork); `auto-commit-on-exit.ts` | — | `examples/extensions/git-checkpoint.ts:4,20-45` |
| Shadow git repo per project/worktree, snapshot each step | — | ✅ `track` = add + `write-tree` at step start; `patch` part at step finish; files > 2 MiB skipped; `gc --prune=7.days` | `packages/opencode/src/snapshot/index.ts:23-24,280-347`; `11d042be25` 2025-07-01 → [[workspace-snapshots]] |
| Revert + unrevert to a message | — | ✅ soft marker; reverted messages deleted only on the next prompt | `packages/opencode/src/session/revert.ts:38-124`; `packages/opencode/src/session/prompt.ts:1056` |
| Requires git | user's own | yes (only when project vcs is git) | organs-legacy |

**When each wins**
- **None (pi)**: users who already commit often; no hidden `.git` store growing in the data dir; no surprise restores. Cost: after `/tree` navigation the model's context can disagree with the working tree.
- **Snapshots (opencode)**: casual or exploratory use, where "undo that last change" is the most common request; conversation and files rewind together. Costs: disk and CPU on big repos (2 MiB exclusion added after blow-ups, `0a80ef4278`), and only harness-visible changes are tracked correctly when other processes also write.

Related: [[no-checkpoints-undo]] · [[session-store-format]] · [[pi]] · [[opencode]] · [[Tradeoffs]]
