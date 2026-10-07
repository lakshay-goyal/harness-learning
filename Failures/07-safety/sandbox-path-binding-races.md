---
type: failure
concepts: [os-level-sandbox, protected-workspace-metadata]
harnesses: [codex]
---
**Symptom** — A sandboxed process could defeat path-based carve-outs: (a) rename a writable ancestor so protected descendants moved outside their carve-outs; (b) unlink/replace a writable root directory so the *next* sandbox policy built from that path pointed somewhere else; (c) swap symlinked path components between profile generation and enforcement; (d) reach a `.git` gitdir living in another writable root through a `.git` pointer file.

**Root cause** — Policies are compiled from paths at one moment and enforced later against a mutable filesystem (TOCTOU); carve-outs protected the leaf, not its ancestors or the root anchor.

**Fix · [[codex]]** — 2026-08-19 `52e387daca` "Prevent protected-path rename bypasses in macOS Seatbelt" (deny unlink on ancestors of protected paths, `codex-rs/sandboxing/src/seatbelt.rs:910-934`); 2026-08-19 `f6950546e5` "Protect macOS Seatbelt writable root anchors" ("A sandboxed process must not be able to replace an authority boundary that will be reused to build the next sandbox policy", `codex-rs/sandboxing/src/seatbelt.rs:518-521`); 2026-08-20 `02de49f718` "Harden Seatbelt writable root path binding" (literal grants for files, keep mutable components until bind); 2026-09-25 `a92ccbde53` "Preserve Git directory protections across writable roots"; earlier symlink fixes 2026-03-14 `9060dc7557`, 2026-03-16 `db7e02c739`, 2026-04-10 `b114781495` ([[symlinked-roots-escape-sandbox-policy]]). Linux masks symlink-in-path / missing protected components with `/dev/null` (`codex-rs/linux-sandbox/README.md:78-80`); Windows opens dirs without traversing reparse points (`codex-rs/windows-sandbox-rs/src/no_reparse_dir.rs:1-3`).

**Lesson** — Path-based sandbox policies are TOCTOU-prone: protect the ancestors and anchors of every carve-out, and bind paths as late and as literally as possible.

Related: [[os-level-sandbox]] · [[protected-workspace-metadata]] · [[path-normalization]] · [[codex--os-level-sandbox|codex]] · [[agent-writes-its-own-escalation-config]]
