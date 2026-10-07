---
type: implementation
harness: codex
concept: protected-workspace-metadata
commit: 622e9e3696
files: [codex-rs/protocol/src/permissions.rs:40, codex-rs/protocol/src/permissions.rs:2346, codex-rs/protocol/src/protocol.rs:1161, codex-rs/sandboxing/src/seatbelt.rs:565, codex-rs/sandboxing/src/seatbelt.rs:592, codex-rs/sandboxing/src/seatbelt.rs:910, codex-rs/linux-sandbox/README.md:50]
---
[[protected-workspace-metadata]] in [[codex]].

## Mechanism
- Type: `WritableRoot{root, read_only_subpaths, protected_metadata_names}`; `is_path_writable` rejects paths under a read-only subpath or whose first component under root is a protected name (`codex-rs/protocol/src/protocol.rs:1161-1210`). Doc: protects "`.codex`, `.git`, notably `.git/hooks`" because they "could be modified to escalate the privileges of the agent" (`codex-rs/protocol/src/protocol.rs:1163-1167`).
- List: `PROTECTED_METADATA_PATH_NAMES = [".git", ".agents", ".codex", ".aws"]` (`codex-rs/protocol/src/permissions.rs:40-51`).
- Per-name rules in `default_read_only_subpaths_for_writable_root` (`codex-rs/protocol/src/permissions.rs:2346-2390`):
  - `.git` protected whether directory or file; a `.git` pointer file (worktree/submodule) also protects the resolved gitdir; bare repo where the gitdir is the root itself handled.
  - `.codex` protected even when missing for the workspace root "so first-time creation still goes through the protected-path approval flow".
  - `.agents`, `.aws` only if they exist; `.aws` because "AWS profiles can select credential helpers that the application executes".
- Enforcement per backend ([[codex--os-level-sandbox]]):
  - Seatbelt: excluded subpaths get both `(require-not (literal …))` and `(require-not (subpath …))` — "`subpath` alone leaves a gap for first-time creation of the protected directory itself, such as `mkdir .codex`" (`codex-rs/sandboxing/src/seatbelt.rs:565-576`); metadata names by regex `^{root}/{name}(/.*)?$` (`codex-rs/sandboxing/src/seatbelt.rs:592-604`); ancestors of read-only paths denied unlink/rename (`codex-rs/sandboxing/src/seatbelt.rs:910-934`); writable-root anchors denied unlink (`codex-rs/sandboxing/src/seatbelt.rs:518-521`).
  - Linux bwrap: protected subpaths (`.git`, resolved `gitdir:`, `.codex`) re-applied `--ro-bind`; missing protected paths / symlink components masked with `/dev/null` (`codex-rs/linux-sandbox/README.md:50-52`, `:78-80`).
  - Windows: `.git` file entries denied under writable roots (`f2de920185`); MXC receives protected-metadata carve-outs from the canonical profile (`codex-rs/mxc-sandbox/README.md:21-23`).
- Writes beneath denied paths require fresh approval rather than reusing a broader grant (`d68b85a097`).

## Constants
| name | value | path:line |
|---|---|---|
| protected metadata names | `.git`, `.agents`, `.codex`, `.aws` | `codex-rs/protocol/src/permissions.rs:40-51` |

## Evolution
- 2025-08-01 `80555d4ff2` "make .git read-only within a writable root when using Seatbelt (#1765)".
- 2025-08-15 `26c8373821` "tighten up checks against writable folders for SandboxPolicy (#2338)".
- 2025-12-15 `bef36f4ae7` `.codex` sub-folder of writable root read-only (#8088).
- 2026-01-20 `f2de920185` Windows: deny `.git` file entries under writable roots.
- 2026-02-03 `9a487f9c18` "make $PWD/.agents read-only like $PWD/.codex (#10524)".
- 2026-03-26 `86764af684` "Protect first-time project .codex creation across Linux and macOS sandboxes (#15067)".
- 2026-04-28 `0156b1e61f` generic "Enforce protected workspace metadata paths (#19846)", `0670d8971a` Seatbelt; 2026-04-29 `74f06dcdfb` Linux.
- 2026-08-18 `d68b85a097` "Require fresh approval beneath denied permission paths".
- 2026-08-19 `52e387daca` "Prevent protected-path rename bypasses in macOS Seatbelt (#39623)".
- 2026-09-25 `a92ccbde53` "Preserve Git directory protections across writable roots (#47974)" (gitdir pointer into another writable root); `1d804e91b7` "Protect `.aws` directories under sandbox writable roots (#48176)".

## Quirks
- The list grows one vector at a time (hooks → agent config → skills → credential helpers); each addition was a separate fix ([[agent-writes-its-own-escalation-config]]).
- `.aws` is protected only as a top-level child of a writable root; `~/.aws` outside writable roots is already non-writable.

## Versus pi
- pi has no write confinement; its `protected-paths.ts` example blocks `write`/`edit` on `.env`, `.git/`, `node_modules/` by substring but bash can still write them ([[pi--tool-call-gate]]).
