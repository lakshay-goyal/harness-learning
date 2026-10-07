---
type: implementation
harness: codex
concept: shell-environment-snapshot
commit: 622e9e3696
files: [codex-rs/shell-command/src/shell_snapshot_capture.rs:1-140, codex-rs/core/src/shell_snapshot.rs:60-107, codex-rs/core/src/tools/runtimes/mod.rs:263-300, codex-rs/features/src/lib.rs:1060-1085, codex-rs/core/src/shell.rs:24-50]
---
[[shell-environment-snapshot]] in [[codex]].

## Mechanism
- **Capture** once per session: run the user's shell (zsh / bash / sh; PowerShell and cmd unsupported) with a generated script that prints `unalias -a`, `functions`, `setopt`s, `alias -L`, exports and optionally `env -0` (`codex-rs/shell-command/src/shell_snapshot_capture.rs:1-140`). Timeout `SNAPSHOT_TIMEOUT = 10 s`, files under `$CODEX_HOME/shell_snapshots` (`SNAPSHOT_DIR`), `SNAPSHOT_RETENTION = 3 days` (`codex-rs/core/src/shell_snapshot.rs:105-107`).
- **Use** (`maybe_wrap_shell_lc_with_snapshot`, POSIX only, `codex-rs/core/src/tools/runtimes/mod.rs:267-300`): commands of the form `[shell, "-lc", script]` (or brokered `[shell, "-c", script]`) become one non-login shell: `user_shell -c ". SNAPSHOT (best effort); exec shell -c <script>"`. No-op for non-matching commands or when command cwd ≠ snapshot cwd.
- After sourcing, re-apply `explicit_env_overrides` (policy-driven shell env) and runtime-only vars such as `CODEX_THREAD_ID`; replay Codex-owned PATH prepends unless the user overrides `PATH` (`runtimes/mod.rs:287-296`).
- **Credential brokerage**: when the network credential broker is active, snapshot env values are swapped for dummies; brokered Bash/Zsh run in the session shell process "to preserve captured shell functions"; brokered zsh wrappers use `-f` "so startup files cannot overwrite credentials after the proxy has replaced them with dummies"; `ZDOTDIR=/dev/null` set (`runtimes/mod.rs:263,278-281`; `codex-rs/core/src/shell_snapshot.rs:60-100`) → [[egress-policy-proxy]], [[secret-handling]].
- Base argv without snapshot: `[shell, "-lc"|"-c", cmd]` for zsh/bash/sh; PowerShell `-NoProfile -Command` when not login; cmd `/c` (`codex-rs/core/src/shell.rs:24-50`). `exec_command.login` param toggles `-l/-i` ("Defaults to true", `codex-rs/core/src/tools/handlers/shell_spec.rs:75-77`).
- Gates: `Feature::ShellSnapshot` Stable, default on; `Feature::ShellSnapshotV2` (executor-side snapshots) UnderDevelopment; neighbours `LoginShellPackagePath` Experimental off, `PowerShellShellVersion` UnderDevelopment (`codex-rs/features/src/lib.rs:1060-1085`).

## Constants
| name | value | path:line |
|---|---|---|
| `SNAPSHOT_TIMEOUT` | 10 s | `codex-rs/core/src/shell_snapshot.rs:105` |
| `SNAPSHOT_RETENTION` | 3 days | `codex-rs/core/src/shell_snapshot.rs:106` |
| `SNAPSHOT_DIR` | `shell_snapshots` | `codex-rs/core/src/shell_snapshot.rs:107` |

## Evolution
- 2025-12-09 `7836aeddae` "feat: shell snapshotting (#7641)".
- 2026-07-20 `2deed3fb9c` preserve zsh tied PATH exports.
- 2026-08-18 `681c82f497` capture scripts moved to `codex-shell-command`.
- 2026-09-08 `2e220af1f6` protect snapshots under credential brokerage; 2026-09-09 `808b3411fd` cache protected snapshots + harden capture cleanup; `ec512d2347` harden credential handling in snapshots and replay.

## Quirks
- Snapshot is skipped silently (best-effort `.`) — a failed capture degrades to the plain login-less shell without telling the model (inferred from "best effort" wrapper text).
- cwd-equality condition means `workdir` changes bypass the snapshot (`runtimes/mod.rs:284-285` comment).

## Versus pi
pi spawns `bash -c` with `process.env` + its bin dir and a user-configured `shellCommandPrefix` ([[pi--shell-execution]]); no capture of login-shell state.
