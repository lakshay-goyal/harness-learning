---
type: implementation
harness: codex
concept: dangerous-command-heuristics
commit: 622e9e3696
files: [codex-rs/shell-command/src/command_safety/is_dangerous_command.rs:34, codex-rs/shell-command/src/command_safety/is_dangerous_command.rs:49, codex-rs/shell-command/src/command_safety/windows_dangerous_commands.rs:6, codex-rs/core/src/exec_policy.rs:787]
---
[[dangerous-command-heuristics]] in [[codex]].

## Mechanism
- POSIX (`codex-rs/shell-command/src/command_safety/is_dangerous_command.rs:49-150`): `rm` with a force option (`DangerousCommandMatch::ForcedRm`); `sudo`, `env`, `trap` are unwrapped and the inner command re-checked; `bash/zsh -lc` scripts parsed so nested literals are checked; recursion cap `MAX_DANGEROUS_COMMAND_WRAPPER_DEPTH = 8`, exceeding it ⇒ dangerous (`:34`).
- Windows (`codex-rs/shell-command/src/command_safety/windows_dangerous_commands.rs:6-305`): `Start-Process` / `Invoke-Item` / ShellExecute, `mshta` / `explorer` with a URL, `cmd /c del|erase|rd|rmdir` with force/recursive/quiet flags, PowerShell `Remove-Item -Force`; splits embedded cmd operators (`echo hi&del`).
- Use in policy (`codex-rs/core/src/exec_policy.rs:787-806`): for commands no rule matched, a dangerous hit → `Prompt`; under `AskForApproval::Never` → `Forbidden`. Windows with no platform sandbox but managed FS restrictions → every unmatched command treated as dangerous.
- Is a denylist only; the known-safe allowlist companion (`is_known_safe_command`, 536 lines + 503-line Windows variant) was deleted in `942af8447b` ([[no-safe-command-allowlist]]).

## Constants
| name | value | path:line |
|---|---|---|
| MAX_DANGEROUS_COMMAND_WRAPPER_DEPTH | 8 | `codex-rs/shell-command/src/command_safety/is_dangerous_command.rs:34` |

## Evolution
- 2025-09-26 `55801700de` "reject dangerous commands for AskForApproval::Never (#4307)".
- 2026-08-19 `942af8447b` allowlist removed; denylist kept.

## Versus pi
- pi's equivalent is the opt-in `permission-gate.ts` regex list over `bash` only ([[pi--tool-call-gate]]); nothing in core.
