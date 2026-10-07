---
type: implementation
harness: codex
concept: file-op-tracking
commit: 622e9e3696
files: [codex-rs/core/src/turn_diff_tracker.rs:12, codex-rs/core/src/turn_diff_tracker.rs:17, codex-rs/core/src/turn_diff_tracker.rs:44, codex-rs/core/src/turn_diff_tracker.rs:92, codex-rs/rollout/src/policy.rs:203]
---
[[file-op-tracking]] in [[codex]] — partial match: a per-turn diff for clients, not a compaction-carried file list.

## Mechanism
- **Operation-backed**: `TurnDiffTracker` accumulates baseline/current contents from committed `apply_patch` deltas (`AppliedPatchDelta`) per (environment, path), never re-reading the filesystem; renders a git-style unified diff (blob OIDs, `/dev/null`, mode 100644) with a 100 ms diff timeout falling back to a coarse content-exact diff (`codex-rs/core/src/turn_diff_tracker.rs:12-17`, `:44-60`).
- A non-exact delta invalidates the turn diff (`:92-112`); exact deltas of partially failed patches are still recorded (`:92-105`).
- Shell-made changes (`sed -i`, scripts, formatters) are not tracked.
- **Purpose is UI**: emitted as `TurnDiff` events (`turn/diff/updated`), not persisted to the rollout (`codex-rs/rollout/src/policy.rs:203`), not shown to the model, not carried across compaction.

## Constants
| name | value | path:line |
|---|---|---|
| `DIFF_TIMEOUT` | 100 ms (coarse fallback) | `codex-rs/core/src/turn_diff_tracker.rs:17` |

## Evolution
- 2026-05-07 `f7e8ff8e50` "Make turn diff tracking operation backed (#21180)" — replaced filesystem snapshots.
- 2026-05-07 `9b6c6f7a01` "preserve exact turn diffs after partial apply_patch failures (#21518)": "a move can write the destination file before failing to remove the source. Treating the whole call as unknowable then drops a change that Codex actually knows happened" → [[turn-diff-drops-known-change]].
- 2026-06-10 `b389b950e1` 100 ms render timeout.
- 2026-04-27 `4e05f3053c` removed ghost snapshots (git-based undo) → [[undo-clobbers-user-git-state]].

## Versus pi
- [[pi--file-op-tracking]] extracts read/modified file lists from tool calls and carries them across compactions as `<read-files>`/`<modified-files>` for the *model*. codex tracks exact diffs for the *user* and gives the model no file working-set after compaction (only user messages + summary survive).
