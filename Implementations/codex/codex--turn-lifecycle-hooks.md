---
type: implementation
harness: codex
concept: turn-lifecycle-hooks
commit: 622e9e3696
files: [codex-rs/hooks/src/engine/dispatcher.rs:115, codex-rs/protocol/src/protocol.rs:1624, codex-rs/hooks/src/engine/discovery.rs:637, codex-rs/hooks/src/events/stop.rs:257, codex-rs/core/src/session/turn.rs:622, codex-rs/core/src/hook_runtime.rs:677, codex-rs/hooks/src/engine/command_runner.rs:54, codex-rs/hooks/src/events/session_end.rs:20]
---
[[turn-lifecycle-hooks]] in [[codex]].

## Mechanism
1. **Substrate**: user-configured external-process hooks with a Claude-Code-compatible protocol — engine type is literally `ClaudeHooksEngine` (`codex-rs/hooks/src/engine/dispatcher.rs:115`). Events: PreToolUse, PermissionRequest, PostToolUse, PreCompact, PostCompact, SessionStart, SessionEnd, UserPromptSubmit, SubagentStart, SubagentStop, Stop, Interrupt (`codex-rs/protocol/src/protocol.rs:1624-1637`). Handler types Command, McpTool, Prompt, Agent (`:1641-1646`) — Prompt and Agent skipped at load with "prompt hooks are not supported yet" / "agent hooks are not supported yet" (`codex-rs/hooks/src/engine/discovery.rs:637-656`) → [[no-prompt-and-agent-hooks]].
2. **Where they fire in the loop**: SessionStart pending hooks before the first sample and after mid-turn compaction (`codex-rs/core/src/session/turn.rs:321`, `:600`); UserPromptSubmit on initial and every steered input — a stop result ends the turn before sampling (`codex-rs/core/src/session/turn.rs:363`, `:438`; `codex-rs/core/src/hook_runtime.rs:677-713`); Stop/SubagentStop when the model wants to finish (`codex-rs/core/src/session/turn.rs:622-668`); Interrupt in `handle_task_abort` (`codex-rs/core/src/tasks/mod.rs:993-995`); legacy after-agent hook (`codex-rs/core/src/hook_runtime.rs:609`). Tool-side PreToolUse/PostToolUse → [[tool-call-gate]].
3. **Stop semantics** (`codex-rs/hooks/src/events/stop.rs:257-445`): exit 0 + JSON `{"continue":false}` ⇒ should_stop (ends turn, wins over block); JSON `{"decision":"block","reason":…}` or exit 2 with stderr reason ⇒ should_block → reason injected as a hook-prompt message, loop continues with `stop_hook_active = true` (`codex-rs/core/src/session/turn.rs:645-668`); block without reason = error, not block. Multiple handlers aggregate: any stop ⇒ stop; else any block ⇒ block with all reasons as continuation fragments (`codex-rs/hooks/src/events/stop.rs:417-440`).
4. **No continuation cap**: nothing in `codex-rs/core/src/session/turn.rs:618-725` bounds Stop-hook re-opens; relies on the hook honouring `stop_hook_active` (unverified that no other guard exists) → [[unbounded-hook-continuation-loop]]. Exception: memory-consolidation sessions turn block/stop into an error "Do not feed managed rejections back into an unattended memory loop" (`codex-rs/core/src/session/turn.rs:629-637`).
5. **Concurrency**: synchronous handlers for one event run concurrently (`FuturesUnordered`), results re-ordered by config order (`codex-rs/hooks/src/engine/dispatcher.rs:115-160`); async handlers fire-and-forget through a semaphore (`codex-rs/hooks/src/engine/command_runner.rs:54`, `:116-140`); their results drained at turn start (recorded as context before the user prompt) or after each sampling step (injected into the running turn) (`codex-rs/core/src/hook_runtime.rs:782-818`, `codex-rs/core/src/session/turn.rs:177`, `:556`).
6. **Timeouts**: timeout outcome "hook timed out after {n}s" (`codex-rs/hooks/src/engine/command_runner.rs:276-318`); stdin written concurrently with draining stdout/stderr, timeout covers both (`885113aa1d`) → [[plugin-hook-wall-clock-timeout]].
7. **Request-rewrite seam**: codex has no `prepareRequest`-style callback; the loop itself re-captures a `StepContext` (tools, MCP binding, settings, AGENTS.md) before every sampling request and appends world-state diffs (`codex-rs/core/src/session/turn.rs:460-518`) → [[mid-turn-settings-switch]], [[world-state-diff-injection]], [[tool-loadout-stale-within-run]]. In-process compile-time extension contributors (turn-start contributors, `on_turn_abort`, `on_thread_idle`) are the code-level seam (`codex-rs/core/src/tasks/regular.rs:37-124`, `codex-rs/core/src/tasks/mod.rs:1010`) → [[extension-event-hooks]].

## Constants
| name | value | path:line |
|---|---|---|
| default command-hook timeout | 600 s | `codex-rs/hooks/src/engine/discovery.rs:764` |
| SessionEnd hook timeout default / max | 1 s / 3 s (clamped) | `codex-rs/hooks/src/events/session_end.rs:20-23`; `codex-rs/hooks/src/engine/discovery.rs:752-762` |
| concurrent async hooks | `MAX_CONCURRENT_ASYNC_HOOKS` = 8 | `codex-rs/hooks/src/engine/command_runner.rs:54` |
| Stop-hook continuation cap | none | `codex-rs/core/src/session/turn.rs:618-725` |

## Evolution
- 2026-02-10 `d735df1f50` "Extract hooks into dedicated crate (#11311)".
- 2026-03-09 `244b2d53f4` "start of hooks engine (#13276)".
- 2026-08-15 `85fc4def35` MCP tool hook handlers (#38705).
- 2026-08-25 `cbfd999db7` "Add hooks for interrupted turns (#40511)"; 2026-08-28 `c2abf869d5` "Run executor hooks for interrupted turns (#41432)".
- 2026-09-09 `885113aa1d` "Prevent command hooks from hanging on blocked stdin (#44288)".

## Quirks
- Protocol compatibility is the design goal: users can reuse Claude Code hook scripts (exit 2 / `decision:block` / `continue:false`).
- Steers re-run UserPromptSubmit, so a policy hook sees every mid-turn message, not just the opener.

## Versus pi
- [[pi--turn-lifecycle-hooks]]: pi's hooks are in-process loop callbacks that rewrite the next request (`prepareRequest`/`prepareNextTurn`/`finishTurn`); codex's user hooks are shell/MCP processes with JSON stdin/stdout and exit-code semantics, while request rewriting is built into the loop (per-step snapshot + appended diffs).
