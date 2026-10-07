---
type: absence
harnesses: [pi]
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

**opencode contrast**: implements it: shadow-git snapshot per step, `patch` parts, `/undo` + `unrevert` (`packages/opencode/src/snapshot/index.ts:23-24,280-347`; `packages/opencode/src/session/revert.ts:38-124`) — see [[workspace-snapshots]] / [[undo-vs-none]].

Related: [[session-tree]] · [[session-fork]] · [[branch-summary]] · [[extension-event-hooks]] · [[no-permission-prompts]] · [[no-sandbox]] · [[Absences]]
