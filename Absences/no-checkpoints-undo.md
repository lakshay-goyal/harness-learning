---
type: absence
harnesses: [pi, codex]
---
# no-checkpoints-undo

No file checkpoints, no `/undo` of file changes, no git auto-commit.

**What's missing**
- The session tree branches the **conversation only**. `/tree`, `/fork` and `/clone` (`packages/coding-agent/docs/sessions.md:20-32`) never restore files.
- The write tool overwrites with no backup and no atomic temp+rename (`03-tools`; `packages/coding-agent/src/core/tools/write.ts:67-89`). Edit keeps no pre-image beyond the diff in tool-result details.
- No shadow git repo and no per-turn snapshot.

**Evidence of decision**
- No manifesto line; absence by omission. The official guidance is user-side: "Use snapshots, backups, or version control before substantial changes." (`packages/coding-agent/docs/security.md:89`).
- `b172beb92` (2025-11-12) mitigation: "Don't use pi on systems with sensitive data you can't afford to lose".
- The only "undo" in core is editor text undo, `Ctrl+-` in the TUI input (`bacf334bc`, 2026-01-18). It is unrelated to files.

**Opt-in replacement** (`packages/coding-agent/examples/extensions/`)
- `git-checkpoint.ts`: `git stash create` on every `turn_start`, keyed by entry id. On `session_before_fork` it offers "Restore code state?" → `git stash apply <ref>`, so "/fork can restore code state" (`git-checkpoint.ts:4,20-45`). It skips restore when `!ctx.hasUI`. `git stash create` does not capture untracked files (git semantics; unverified whether the example compensates).
- `auto-commit-on-exit.ts`: on `session_shutdown`, `git add -A` plus a commit whose message comes from the last assistant reply (`:11,42-43`).
- `dirty-repo-guard.ts` blocks session changes when uncommitted changes exist. `git-merge-and-resolve.ts` fetches and merges upstream after each turn.
- `confirm-destructive.ts` confirms clear/switch/fork.

**History**
- Never present in core.

**Implication**
- Conversation state and file state are decoupled. Rewinding the tree (`/tree`) leaves edits on disk, so the model's context can disagree with the working tree after navigation. [[branch-summary]] partially compensates for conversation context only.
- Recovery is delegated to git and the user, consistent with "fast iteration requires trust" (`b172beb92`).

**codex** — *absent now, by removal*. Codex shipped git ghost-commit snapshots + `/undo`: `e0fbc112c7` 2025-09-23 "feat: git tooling for undo (#3914)", `e92c4f6561`/`afc4eaab8b` 2025-10-27 async ghost commits + `/undo`, enabled by default `052b052832` 2025-11-11. Bug: `/undo` used `git restore --staged`, wiping the user's index (`014235f533` 2025-12-20, issue #8214) → [[undo-clobbers-user-git-state]]; two days later "chore: un-ship undo" `7a8407bbb6` 2025-12-22 and "drop undo from the docs" `45727b9ed3`; ghost snapshots removed from the Responses API surface 2026-04-27 `4e05f3053c` (#19481, "make undo a no-op that reports the feature is unavailable"). Today: `undo` is a `Stage::Removed` no-op flag (`codex-rs/features/src/lib.rs:425-428`, `:1006-1009`); `GhostSnapshotToml` fields "Legacy no-op setting retained for compatibility" (`codex-rs/config/src/config_toml.rs:804-814`). Rationale for removal not stated in commit bodies (unverified; timing suggests the staging data-loss bug). `thread/revert` explicitly leaves files alone: "local file changes are unaffected" (`4343b2bdc4`). Codex still tracks a per-turn diff for display (`codex-rs/core/src/turn_diff_tracker.rs:12-17`, [[file-op-tracking]]) and `codex apply` git-applies the last diff — neither restores state. Same end state as pi: conversation rewind and workspace state decoupled.

Related: [[session-tree]] · [[session-fork]] · [[branch-summary]] · [[extension-event-hooks]] · [[no-permission-prompts]] · [[no-sandbox]] · [[Absences]] · [[undo-clobbers-user-git-state]] · [[codex--session-fork|codex]] · [[file-op-tracking]]
