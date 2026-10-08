---
type: implementation
harness: codex
concept: elicitation-pause
commit: 622e9e3696
files: [codex-rs/core/src/elicitation.rs:1, codex-rs/core/src/session/mod.rs:1421, codex-rs/core/src/unified_exec/process_manager.rs:644, codex-rs/core/src/codex_thread.rs:1218, codex-rs/core/src/tools/code_mode/execute_handler.rs:162]
---
[[elicitation-pause]] in [[codex]].

## Mechanism
- `ElicitationService` "Coordinates user elicitations that pause tool-result delivery for a session. Registrations are counted so concurrent elicitations keep the session paused until all of them finish." (`codex-rs/core/src/elicitation.rs:6-10`). `register()` increments; first registration flips a `watch<bool>` to paused; dropping the last `ElicitationRegistration` clears it (`:29-75`).
- **Consumers**: unified exec subscribes for its timeouts (`codex-rs/core/src/unified_exec/process_manager.rs:644`, `:1172`; `codex-rs/core/src/session/mod.rs:1421-1423`) → [[shell-execution]]. Code mode waits `wait_until_clear()` before returning a cell result (`codex-rs/core/src/tools/code_mode/execute_handler.rs:162`, `codex-rs/core/src/tools/code_mode/wait_handler.rs:190`) → [[code-mode]].
- **Out-of-band elicitations**: clients increment/decrement a per-thread counter; 0→1 registers with the service, →0 drops the registration; decrement at zero = `InvalidRequest` (`codex-rs/core/src/codex_thread.rs:1218-1242`).

## Evolution
- 2026-03-09 `c6343e0649` "Implemented thread-level atomic elicitation counter for stopwatch pausing (#12296)".
- 2026-07-06 `84fe70c30e` "elicitations: Move to shared ElicitationService (#30627)".
- 2026-10-07 `a6baf8867c` "Signal abandonment of unanswered MCP elicitations (#51611)": MCP `ElicitationRequestRouter` keys pending responders by `(server_name, request_id)`; dropping the native waiter (`ElicitationRequestGuard`) removes the responder and emits `EventMsg::ElicitationAbandoned{server_name, id}` on the original unbounded event channel, so clients can retire the prompt; router phases Open → Closing → Closed wait for in-flight deliveries on shutdown (`codex-rs/codex-mcp/src/elicitation.rs:100-180`).

## Versus pi
- pi has no approval prompts ([[no-permission-prompts]]), so no pause concept exists.
