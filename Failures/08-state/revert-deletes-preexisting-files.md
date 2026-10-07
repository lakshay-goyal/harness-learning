---
type: failure
concepts: [workspace-snapshots]
harnesses: [opencode]
---
**Symptom** — Undoing an agent turn deleted user files that had existed before the session, not just files the agent created.

**Root cause** — Per-file revert ran `git checkout <snapshot> -- <file>` in the shadow repo and treated any checkout failure as "file did not exist in the snapshot → the agent created it → delete". Files missing from the shadow tree for other reasons (ignored, too large, never staged) were deleted.

**Fix · [[opencode]]** — `6a7eeb39c3` 2025-10-01 "prevent file deletion when reverting changes to existing files": on checkout failure, `git ls-tree <hash> -- <rel>`; if the file is in the tree, keep it and log "file existed in snapshot but checkout failed, keeping"; delete only when absent (`packages/opencode/src/snapshot/index.ts:425-443`). Follow-ups: `48db7cf07a` 2026-04-01 batched revert without reordering, `6359d00fb4` 2026-04-03 restoring an earlier message in a reverted chain.

**Lesson** — "Absent from the checkpoint" is not "created by the agent"; verify against the base tree before any destructive undo step, and fail closed (keep the file).

Related: [[workspace-snapshots]] · [[host-vcs-config-leaks-into-shadow-repo]] · [[opencode--workspace-snapshots|opencode impl]] · [[no-checkpoints-undo]]
