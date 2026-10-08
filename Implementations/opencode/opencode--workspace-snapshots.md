---
type: implementation
harness: opencode
concept: workspace-snapshots
commit: ecc4916b5a
files: [packages/opencode/src/snapshot/index.ts:23-27, packages/opencode/src/snapshot/index.ts:66-75, packages/opencode/src/snapshot/index.ts:165-170, packages/opencode/src/snapshot/index.ts:278-347, packages/opencode/src/snapshot/index.ts:425-443, packages/opencode/src/snapshot/index.ts:761-766, packages/opencode/src/session/processor.ts:99-102, packages/opencode/src/session/processor.ts:425-483, packages/opencode/src/session/revert.ts:38-124, packages/core/src/snapshot.ts:94-144, packages/core/src/session/revert.ts:60-121, packages/core/src/session/runner/llm.ts:226, packages/core/src/session/runner/llm.ts:325-330]
---
[[workspace-snapshots]] in [[opencode]].

## Mechanism

### Legacy runtime — shadow git dir per project/worktree
- Git dir `<data>/snapshot/<projectID>/<hash(worktree)>`, every command run with `--git-dir <shadow> --work-tree <user worktree>` (`packages/opencode/src/snapshot/index.ts:66-75`); enabled only when the project VCS is git and `snapshot !== false` (`packages/opencode/src/snapshot/index.ts:167-170`); serialized per git dir with a semaphore (`packages/opencode/src/snapshot/index.ts:165`).
- Init config isolates host settings and tunes for big repos: `core.autocrlf=false`, `longpaths`, `symlinks`, `fsmonitor=false`, `feature.manyFiles`, `index.version 4`, `index.threads`, `untrackedCache`; objects seeded from the source repo (`packages/opencode/src/snapshot/index.ts:324-337`).
- `track()` = stage allowed paths + `git write-tree` → tree hash (`packages/opencode/src/snapshot/index.ts:318-347`); untracked files > 2 MiB excluded (`packages/opencode/src/snapshot/index.ts:278-297`); diffs use `--no-ext-diff`.
- Processor: snapshot **before** the stream ("The AI SDK may execute tools internally before emitting start-step events", `packages/opencode/src/session/processor.ts:99-102`); on step-finish `track` again + `patch(start)` → a `patch` part `{hash, files}` on the assistant message (`packages/opencode/src/session/processor.ts:425-483`); `cleanup()` writes the patch if the stream ended without step-finish.
- **Revert** `SessionRevert.revert` (`packages/opencode/src/session/revert.ts:38-89`): refuse while busy; find target message/part; collect `patch` parts after it; remember the pre-revert snapshot (`rev.snapshot = existing ?? track()`), restore an earlier revert first, then `snap.revert(patches)`; store `revert` state + diff summary. Per-file revert: `checkout <hash> -- file`; on failure, `ls-tree` decides keep vs delete (`packages/opencode/src/snapshot/index.ts:425-443`).
- **Unrevert** restores the pre-revert snapshot and clears the marker (`packages/opencode/src/session/revert.ts:91-99`); the next prompt or shell calls `cleanup`, which deletes the hidden messages/parts permanently (`packages/opencode/src/session/revert.ts:101-124`). TUI `undo` / `redo` commands (`packages/tui/src/routes/session/index.tsx:615`, `packages/tui/src/routes/session/index.tsx:652`).
- Retention: `git gc --prune=7.days` every hour after a 1-min delay (`packages/opencode/src/snapshot/index.ts:23`, `packages/opencode/src/snapshot/index.ts:300-313`, `packages/opencode/src/snapshot/index.ts:761-766`).

### v2 runtime
- Same location, created by `git.repo.create({seed: source})` (`packages/core/src/snapshot.ts:94-122`); enabled only in git repos unless config `snapshots: false` (`packages/core/src/snapshot.ts:124-127`); `capture` is best-effort — failure logs and returns undefined (`packages/core/src/snapshot.ts:129-144`).
- Runner captures before the turn and after settlement, recording changed files on `Step.Ended` (`packages/core/src/session/runner/llm.ts:226`, `packages/core/src/session/runner/llm.ts:325-330`).
- Revert is event-sourced: `SessionRevert.stage` (restore files, publish `RevertEvent.Staged` with diff), `clear` (restore original, `Cleared`), `commit` (`Committed`) (`packages/core/src/session/revert.ts:60-121`).

## Constants
| name | value | path:line |
|---|---|---|
| gc prune age | `7.days`, hourly | `packages/opencode/src/snapshot/index.ts:23`, `packages/opencode/src/snapshot/index.ts:761-766` |
| untracked size limit | 2 MiB (both runtimes) | `packages/opencode/src/snapshot/index.ts:24`; `packages/core/src/snapshot.ts:138` |

## Evolution
- 2025-07-01 `11d042be25` snapshot functionality; 2025-07-15 `b5c85d3806` suppress in very large directories; 2025-07-16 `f45deb37f0` don't sign snapshot commits.
- 2025-09-07 `74469a0d3d`, 2025-09-12 `c02f58c2af`, 2026-01-26 `6b83b172ae` await revert cleanup before new messages (duplicates otherwise).
- 2025-10-01 `6a7eeb39c3` don't delete files that existed before → [[revert-deletes-preexisting-files]].
- 2025-11-09 `9637d70407` binary diffs froze the UI; 2025-11-26 `0e08655407` external diff tools; 2026-02-20 `ac0b37a7b7` `info/exclude`; 2026-04-12 `113304a058`/`264418c0cd` gitignore for previously tracked files → [[host-vcs-config-leaks-into-shadow-repo]].
- 2025-12-22 `d4b7f75ce3` snapshot when finish-step is not reached; 2025-12-26 `bfb9787361` `/compact` after revert clears revert state.
- 2026-03-25 `0a80ef4278` skip files > 2 MB; 2026-04-01 `48db7cf07a` batched revert without reordering; 2026-04-03 `6359d00fb4` restore earlier message in a reverted chain; 2026-04-15 `a992d8b733` NUL-separated stdin pathspecs (ENAMETOOLONG); 2026-06-10 `51891d56e7` reuse source objects.
- 2026-06-24 `9bb5370205` (#33226) v2 snapshot + revert system.

## Quirks / drift
- Only git projects get snapshots; non-git directories have no undo.
- Revert is reversible only until the next prompt, then message deletion is permanent.

Contrast: pi has no file checkpoints ([[no-checkpoints-undo]]; only an example git-stash extension); opencode treats file rollback as a core session feature with a private git store.
