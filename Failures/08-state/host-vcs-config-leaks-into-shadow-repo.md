---
type: failure
concepts: [workspace-snapshots]
harnesses: [opencode]
---
**Symptom** — The private snapshot repo behaved like the user's git: snapshot commits hung on GPG signing prompts, external diff tools hijacked diff output, line endings were rewritten, ignored or excluded files were snapshotted (or binaries froze the UI), and huge repos took minutes to snapshot.

**Root cause** — Shelling out to `git` with a private `--git-dir` still inherits global/system git config, the user's `.gitignore`/`info/exclude` semantics are partly bypassed by explicit staging, and default git settings are tuned for human-sized commits, not per-step snapshots of whole worktrees.

**Fix · [[opencode]]**
- `f45deb37f0` 2025-07-16 (#1046) don't sign snapshot commits (HEAD uses `write-tree`, no commits).
- `9637d70407` 2025-11-09 (#4109) binary files froze the UI; `0e08655407` 2025-11-26 `--no-ext-diff` on every diff.
- `ac0b37a7b7` 2026-02-20 (#13495) respect `info/exclude`; `113304a058` / `264418c0cd` 2026-04-12 respect `.gitignore` for previously tracked files.
- `0a80ef4278` 2026-03-25 (#19043) skip files > 2 MB; `a992d8b733` 2026-04-15 NUL-separated stdin pathspecs (ENAMETOOLONG); `51891d56e7` 2026-06-10 reuse source objects instead of re-hashing.
- HEAD init pins `core.autocrlf=false`, `core.fsmonitor=false`, `feature.manyFiles`, `index.version 4`, `untrackedCache` on the shadow repo (`packages/opencode/src/snapshot/index.ts:324-337`).

**Lesson** — A shadow VCS must run with an explicit, isolated config (no signing, no external diff, fixed line endings) and inherit only the user's ignore rules — never their tooling.

Related: [[workspace-snapshots]] · [[revert-deletes-preexisting-files]] · [[opencode--workspace-snapshots|opencode impl]]
