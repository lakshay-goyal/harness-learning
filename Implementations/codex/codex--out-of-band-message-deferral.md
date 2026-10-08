---
type: implementation
harness: codex
concept: out-of-band-message-deferral
commit: 622e9e3696
files: [codex-rs/core/src/session/inject.rs:17, codex-rs/core/src/session/inject.rs:170, codex-rs/core/src/tasks/user_shell.rs:440, codex-rs/core/src/session/handlers.rs:99, codex-rs/core/src/context/user_shell_command.rs:7]
---
[[out-of-band-message-deferral]] in [[codex]].

## Mechanism
- **Generic path** `inject_no_new_turn` ("Injects items into active work, or records them without starting a turn", `codex-rs/core/src/session/inject.rs:169-187`): `inject_if_running` pushes the items into the active turn's pending-input queue as `PendingTurnInput::ResponseItem` (`:17-38`) — consumed at the next sampling boundary like steering input, so they never land between a tool call and its output; if no turn is active they are recorded directly with a default turn context.
- **User shell commands** (`!cmd`): during an active turn the command runs as an auxiliary of that turn (shares its cancellation, no extra TurnStarted/Complete, `codex-rs/core/src/session/handlers.rs:99-131`); its output — truncated with the model policy and wrapped as a `<user_shell_command>` contextual-user fragment with command, exit code, duration (`codex-rs/core/src/context/user_shell_command.rs:7-44`) — goes through `inject_no_new_turn`; standalone shell turns record directly and materialize the rollout (`codex-rs/core/src/tasks/user_shell.rs:440-467`) → [[user-shell-escape]].
- Hook additional context is injected into the running turn atomically by the same module (`codex-rs/core/src/session/inject.rs:40+`).
- Belt-and-braces: prompt-time normalization still synthesizes missing outputs / drops orphans ([[transcript-replay-repair]]).

## Versus pi
- [[pi--out-of-band-message-deferral]] keeps separate buffers per message kind (custom → `turn_end`, bash → run settle, next-turn → next prompt). codex routes every out-of-band item through one "pending input of the active turn, else record now" path.
