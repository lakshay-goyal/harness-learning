---
type: concept
stage: tools
tier: candidate
aliases: ["!cmd", Op::RunUserShellCommand, UserShellCommandTask, "UserShellCommandMode::{StandaloneTurn, ActiveTurnAuxiliary}", user bash, bang command]
harnesses: [codex]
---
The user (not the model) runs a shell command from the prompt with `!cmd`; the harness executes it outside the model's tool path and records command + exit code + truncated output into history as a tagged context fragment the model sees on its next request.

## Why
- Users want to show the agent real state (`git status`, a failing test) without spending a model turn asking it to run it.
- The command is the user's, so model-tool policy (sandbox, approvals) is the wrong gate — but its output still has to be bounded like tool output ([[tool-output-truncation]]).
- Output recorded mid-turn must not split a tool call from its result ([[out-of-band-message-deferral]]).

## Design space
- Excluded from context (pi `!!cmd`) vs **recorded as model-visible context** (pi `!cmd` → user message "Ran `cmd`"; codex `<user_shell_command>` contextual-user fragment).
- While a turn is running: queued until the run ends (pi buffers) vs **run concurrently as an auxiliary of the active turn and injected as pending input** (codex `ActiveTurnAuxiliary` + `inject_no_new_turn`).
- Isolation: same sandbox as model tools vs **unsandboxed user command** (codex `PermissionProfile::Disabled`, `SandboxType::None`).
- Timeout: none vs long default (codex 1 h).
- Output shaping: strip ANSI/binary (pi) vs model's truncation policy (codex).
- Display/telemetry via intent parsing → [[shell-command-intent-parsing]].

## Implementations
- [[codex--user-shell-escape|codex]] — `Op::RunUserShellCommand` → `UserShellCommandTask` (standalone turn or auxiliary of the active turn), unsandboxed, 1 h timeout, output truncated by model policy into `<user_shell_command>`.
- pi: user `!cmd` / `!!cmd` path documented inside [[pi--shell-execution]] (bash-executor) and [[pi--out-of-band-message-deferral]].

## Failures
- (none recorded)

## Related
[[out-of-band-message-deferral]] · [[shell-execution]] · [[single-active-task-slot]] · [[tool-output-truncation]] · [[shell-command-intent-parsing]] · [[xml-prompt-boundaries]] · [[message-role-layering]]
