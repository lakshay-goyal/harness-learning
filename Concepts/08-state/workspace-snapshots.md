---
type: concept
stage: state
tier: candidate
aliases: [Snapshot.track, Snapshot.capture, shadow git repo, "snapshot: false", "snapshots: false", patch part, SessionRevert, "/undo", "/redo", unrevert, workspace-snapshot-revert, transcript-file-revert]
harnesses: [opencode]
---
Snapshot the working tree into a private git store at each step, with per-step patches on messages. This makes revert and unrevert to a message boundary possible, separate from the user's VCS.

## Why
- Conversation rewind without file rewind leaves the model reasoning about edits that are still on disk (or gone); users want "undo that turn" to mean both.
- Using the user's own git (stash/commits) pollutes their history and index and fights their workflow; a private git dir with `--work-tree` gets content-addressed trees for free.
- Snapshotting a real repo is a scale and isolation problem: binaries, huge untracked dirs, host git config (signing, external diff, autocrlf, excludes) all leak into a naive implementation ([[host-vcs-config-leaks-into-shadow-repo]]).
- Revert must distinguish "created by the agent" from "absent from the snapshot", or undo deletes user files ([[revert-deletes-preexisting-files]]).

## Design space
- **Store**: private git dir per project+worktree under the data dir (opencode `<data>/snapshot/<projectID>/<hash(worktree)>`) · user's repo (stash/branches; pi example extension `git-checkpoint.ts`, see [[session-fork]]) · file copies · none ([[no-checkpoints-undo]], pi core).
- **Granularity**: tree hash per model step + `patch` part listing changed files (opencode legacy) · start/end capture per provider turn recorded on the step event (opencode v2).
- **Capture point**: before the stream starts, because the SDK may execute tools before emitting step-start (opencode legacy `processor.ts:99-102`); also at stream end when no step-finish arrived (`d4b7f75ce3`).
- **Exclusions**: untracked files > 2 MiB, respect `.gitignore` and `info/exclude`, skip when not a git repo (opencode both runtimes).
- **Seeding**: reuse the source repo's objects to avoid re-hashing (opencode `51891d56e7`, v2 `seed: source`).
- **Revert semantics**: soft marker (files restored, messages hidden) reversible via unrevert until the next prompt commits it (opencode legacy `cleanup`, v2 `stage`/`clear`/`commit` events) · hard delete.
- **Retention**: `git gc --prune=7.days` hourly (opencode legacy).
- **Failure policy**: best-effort, capture failure only logs (opencode v2 `snapshot.ts:141-143`).

## Implementations
- [[opencode--workspace-snapshots|opencode]] — shadow git dir with `--work-tree`; `track` = add + `write-tree` per step; `patch` parts; `SessionRevert.revert/unrevert/cleanup` (legacy) and `stage/clear/commit` durable events (v2); 2 MiB untracked cap; hourly gc.

## Failures
- [[revert-deletes-preexisting-files]]
- [[host-vcs-config-leaks-into-shadow-repo]]

## Tradeoffs
- [[undo-vs-none]]

## Related
[[session-fork]] · [[no-checkpoints-undo]] · [[git-worktree-isolation]] · [[event-sourced-session-store]] · [[partial-message-persistence]] · [[session-export-share]]
