---
type: failure
concepts: [tool-only-isolation, path-normalization]
harnesses: [codex]
---
**Symptom** — Writable/readable roots given as symlinks were compared un-canonicalized, so policies with a symlinked writable root and a denied child behaved incorrectly; a symlinked Linux sandbox cwd broke enforcement.

**Root cause** — Policy evaluation used lexical paths while the kernel enforcement (Seatbelt, bwrap mounts) resolves real paths; the two disagreed. An early canonicalize attempt (TS era) was reverted the same day.

**Fix · [[codex]]** — 2025-04-18 `3356ac0aef` canonicalize Seatbelt writable paths → reverted `9a046dfcaa`; 2026-03-14 `9060dc7557` "fix symlinked writable roots in sandbox policies"; 2026-03-16 `db7e02c739` "canonicalize symlinked Linux sandbox cwd"; 2026-04-10 `b114781495` "fix symlinked writable roots in sandbox permissions". Linux masks symlink-in-path protected components with `/dev/null` (`codex-rs/linux-sandbox/README.md:78-80`); Windows deny-read ACEs applied to both lexical and canonical paths (`codex-rs/windows-sandbox-rs/src/deny_read_acl.rs:10-13`).

**Lesson** — Evaluate sandbox policies on canonical paths, and make the canonicalization match what the kernel enforcement sees (or apply rules to both lexical and resolved forms).

Related: [[tool-only-isolation]] · [[path-normalization]] · [[os-level-sandbox]] · [[codex--os-level-sandbox|codex]] · [[sandbox-path-binding-races]]
