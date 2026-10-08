---
type: implementation
harness: codex
concept: user-shell-escape
commit: 622e9e3696
files: [codex-rs/core/src/session/handlers.rs:99-131, codex-rs/core/src/tasks/user_shell.rs:47-77, codex-rs/core/src/tasks/user_shell.rs:110-230, codex-rs/core/src/tasks/user_shell.rs:440-466, codex-rs/core/src/user_shell_command.rs:9-40, codex-rs/core/src/context/user_shell_command.rs:7-44, codex-rs/core/src/exec.rs:63-94]
---
[[user-shell-escape]] in [[codex]].

## Mechanism
- `Op::RunUserShellCommand` (`codex-rs/core/src/session/handlers.rs:99-131`): if a turn is active the command runs **concurrently** as `UserShellCommandMode::ActiveTurnAuxiliary` (shares the turn's cancellation token, no second TurnStarted/TurnComplete pair); otherwise `StandaloneTurn`, a task in the [[single-active-task-slot]] with `kind() = TaskKind::Regular` (`codex-rs/core/src/tasks/user_shell.rs:48-57,75-77`).
- Execution (`user_shell.rs:160-230`): user's environment shell (snapshot-wrapped, [[shell-environment-snapshot]]), `inject_session_env` + `inject_apply_patch_env`, `permission_profile = PermissionProfile::Disabled`, `sandbox: SandboxType::None` — the user's command is not sandboxed; expiration `timeout_ms.unwrap_or(USER_SHELL_TIMEOUT_MS)` = 1 h; runs on the legacy `codex-rs/core/src/exec.rs` path (constants below), not unified exec.
- `parse_command` of the display command for begin/end events (`user_shell.rs:187`) → [[shell-command-intent-parsing]].
- Recording (`persist_user_shell_output`, `user_shell.rs:440-466`): output formatted with `format_exec_output_str` under the **model's truncation policy** (`codex-rs/core/src/user_shell_command.rs:9-40`) into a contextual-user fragment `<user_shell_command>…</user_shell_command>` with command, exit code, duration; role `user`, content kind `shell.user_command` (`codex-rs/core/src/context/user_shell_command.rs:7-44`) → [[xml-prompt-boundaries]], [[message-role-layering]].
  - Standalone: `record_conversation_items` + `ensure_rollout_materialized` (shell turns can precede any regular turn).
  - Auxiliary: `session.inject_no_new_turn(items)` — queued as pending input of the active turn, delivered at the next model-request boundary, so it never splits a tool call/result ([[out-of-band-message-deferral]]).
- TODO in source: after TurnStarted, emit model-visible turn-context diffs for standalone lifecycle tasks (`user_shell.rs:120-123`).

## Constants
| name | value | path:line |
|---|---|---|
| `USER_SHELL_TIMEOUT_MS` | 3_600_000 (1 h) | `codex-rs/core/src/tasks/user_shell.rs:47` |
| legacy exec SIGKILL / timeout code / signal exit | 9 / 64 / 128+n | `codex-rs/core/src/exec.rs:67-69` |
| legacy exec read chunk | 8192 bytes | `codex-rs/core/src/exec.rs:74` |
| legacy exec max output delta events | 10_000 per call | `codex-rs/core/src/exec.rs:85` |
| legacy exec IO drain timeout | 2000 ms | `codex-rs/core/src/exec.rs:94` |

## Evolution
- 2025-10-29 `89591e4246` "feature: Add "!cmd" user shell execution (#2471)".
- 2025-11-10 `980886498c` user command event types.
- 2026-02-05 `97582ac52d` "Allow user shell commands to run alongside active turns (#10513)" (`ActiveTurnAuxiliary`).

## Versus pi
pi's `!cmd` strips ANSI/binary, spills > 50 KB, becomes a user message "Ran `cmd`", and `!!cmd` is excluded from context; output arriving during a run is buffered to the turn boundary ([[pi--shell-execution]], [[pi--out-of-band-message-deferral]]). codex has no context-excluded variant (none found) and runs user commands unsandboxed while model commands are sandboxed.
