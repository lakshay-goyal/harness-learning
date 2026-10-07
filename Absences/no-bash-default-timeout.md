---
type: absence
harnesses: [pi, codex]
---
# no-bash-default-timeout

**What's missing**
- A bash/powershell call with no `timeout` runs until it exits or the user aborts. There is no implicit 30s/2min/10min ceiling.
- Schema: `timeout: Type.Optional(Type.Number({ description: "Timeout in seconds (optional, no default timeout)" }))` (`packages/coding-agent/src/core/tools/bash.ts:40-43`).
- Description: "…Optionally provide a timeout in seconds." (`packages/coding-agent/src/core/tools/bash.ts:255`).

**Evidence of decision**
- `29900ce64` (2025-11-12, Mario Zechner), "feat: make bash tool timeout optional and configurable". It replaced "Commands run with a 30 second timeout." with "Optionally provide a timeout in seconds" and changed `setTimeout` to "Set timeout if provided". Rationale per the constants census: "commands run until completion unless specified".
- Same day `271810c80`: long-running commands belong in tmux, not in the agent ([[no-background-bash]]).
- The durable harness copied the stance: "Timeout in seconds (optional, no default timeout)" (`packages/durable/src/tools/bash.ts:12,17`).

**Guards that do exist** → [[shell-execution]], [[process-tree-kill]]
- Model-supplied values are validated: finite and > 0 (`85b7c2474`, 2026-07-01), and ≤ `MAX_TIMEOUT_MS = 2_147_483_647` ms, the int32 setTimeout max (`packages/coding-agent/src/core/tools/bash.ts:22`; `cbcf4e04c`, 2026-06-30, #6181). Before this, huge values fired immediately or NaN'd. Durable mirrors it with `MAX_TIMEOUT_SECONDS` (`packages/durable/src/tools/bash.ts:8,46-54`).
- On timeout: kill the process tree, then throw "Command timed out after N seconds" with partial output prepended (`packages/coding-agent/src/core/tools/bash.ts:152-154,377-380`).
- Abort (Esc) kills the process group ([[abort-propagation]]).

**Contrast within pi** (where defaults *do* exist)
| Executor | Default | Location |
|---|---|---|
| bash / powershell tool | none | `packages/coding-agent/src/core/tools/bash.ts:42` |
| codemode script | none (`Number.POSITIVE_INFINITY` unless `timeout_ms`) | `packages/coding-agent/src/extensions/codemode/execute.ts:484` |
| codemode library | 300s | `packages/codemode/src/runtime/host.ts:22` |
| MCP tool call | 60s, reset by progress notifications | `packages/coding-agent/src/core/mcp-servers.ts:75-76`; `extensions/mcp/runtime.ts:48` |
| `pi.exec` (extensions) | SIGTERM → SIGKILL after 5s, when a timeout is set | `packages/coding-agent/src/core/exec.ts:61` |
| Mistral HTTP request | 60s | `packages/ai/src/api/mistral-conversations.ts:307` |

**Opt-in replacement**
- The model passes `timeout`. Extensions can inject one via `spawnHook` / `tool_call` argument mutation ([[tool-call-gate]]; `packages/coding-agent/src/core/tools/bash.ts:184,211`, `86b43c8ea`). See `examples/extensions/bash-spawn-hook.ts`.

**History**
- 30s (`ffc9be886`, 2025-10-17 era) → none (`29900ce64`, 2025-11-12). Validation hardening followed in 2026-06/07. Never reverted.

**Implication**
- Builds, test suites and installs finish without the model guessing a duration. The cost is that a hung command (server, watcher, prompt waiting on stdin, though stdin is ignored) blocks the turn until Esc. Unattended runs need an external watchdog.
- Together with [[no-turn-cap]], pi has no wall-clock bound anywhere in the default path.

**codex** — *partial; holds for model commands*. The one-shot exec path has `DEFAULT_EXEC_COMMAND_TIMEOUT_MS = 10_000` (`codex-rs/core/src/exec.rs:63`) and a post-exit pipe drain `IO_DRAIN_TIMEOUT_MS = 2_000` (`codex-rs/core/src/exec.rs:94`; grandchild-pipe hang fix `73ed30d7e5` 2025-11-12). But model commands now run through unified exec, which has **no wall-clock kill timeout**: a long command yields a session id and keeps running until the session ends (`codex-rs/core/src/session/handlers.rs:305-312`); a "no timeout mode" experiment was added and reverted the same day (`9719dc502c` / `928be5f515` 2026-02-19) → [[no-kill-timeout-in-unified-exec]]. Different reason from pi: the yield model returns control to the model instead of blocking, so a kill timeout is unnecessary for responsiveness. Elicitation time is excluded from unified-exec timeouts ([[elicitation-pause]]).
**opencode contrast**: implements it: 2-min default (`packages/opencode/src/tool/shell.ts:347`), no max in legacy, 10-min max in v2 (`packages/core/src/tool/bash.ts:19-20`) — see [[shell-execution]] / [[bash-timeout-default-vs-none]].

Related: [[shell-execution]] · [[process-tree-kill]] · [[abort-propagation]] · [[no-background-bash]] · [[no-turn-cap]] · [[Constants]] · [[Absences]] · [[codex--shell-execution|codex]] · [[no-kill-timeout-in-unified-exec]]
