---
type: concept
stage: loop
tier: must-have
aliases: [prepareRequest, prepareNextTurn, prepareNextTurnWithContext, finishTurn, turn-hooks, onYield, afterResponse, afterTools, chat.params, chat.headers, codex-hooks, ClaudeHooksEngine, HookEventName, Stop hook, SubagentStop, UserPromptSubmit, SessionStart, SessionEnd, stop_hook_active, "decision:block", hook-prompt message, after_agent]
harnesses: [pi, opencode, codex]
---
Low-level loop callbacks that rewrite the next request (context/model/thinking), append messages before a turn, or end/force a turn — the seam where session concerns plug into a stateless loop.

## Why
- Without a per-request hook, context is computed once per user prompt; tool/loadout changes and compaction thresholds go stale mid-run ([[tool-loadout-stale-within-run]]).
- A per-turn decision point lets extensions end runs or demand one more response without patching the loop.
- External hook processes need concurrent I/O and an all-encompassing timeout, or a hook that never reads stdin hangs the turn ([[plugin-hook-wall-clock-timeout]]); a Stop hook that always blocks re-opens the turn forever ([[unbounded-hook-continuation-loop]]).
- Keeps the loop provider- and storage-agnostic: projection from a persistent log, model routing, compaction all live in hooks.

## Design space
- **Granularity**: before every request incl. first (`prepareRequest`) · only before continuing turns (`prepareNextTurn`) · after turn (`finishTurn`).
- **Decision vocabulary**: end / continue / default (pi); continue is satisfied by existing scheduling, not additive.
- **Composition**: decorator chain wrapping previous hook (pi coding-agent) · named hook chains in a registry (pi-durable).
- **Queue interaction**: hooks do not poll queues (pi `prepareRequest`); re-poll after long `prepareNextTurn` work.
- **Safety**: unconditional continue = endless loop; pairs with [[no-turn-cap]]. Codex Stop-hook continuations are uncapped, relying on the hook honouring `stop_hook_active`; managed memory-consolidation sessions turn block/stop into an error.
- **Hook substrate**: in-process callbacks (✔ pi) · external processes (command / MCP tool) speaking a Claude-Code-compatible JSON stdin/stdout + exit-code protocol (✔ codex `ClaudeHooksEngine`) · LLM prompt/agent hooks (declared in codex schema, rejected at load).
- **Event set**: request-level (`prepareRequest`/`prepareNextTurn`/`finishTurn`, pi) · lifecycle events SessionStart/UserPromptSubmit (incl. every steer)/Stop/SubagentStop/PreCompact/PostCompact/Interrupt/SessionEnd (+ tool events, [[tool-call-gate]]) (✔ codex).
- **Continue mechanism**: return `continue` (pi) · Stop hook `decision:block` / exit 2 → reason injected as continuation prompt, `stop_hook_active=true` (✔ codex); `{"continue":false}` wins over block.
- **Async hooks**: fire-and-forget behind a semaphore, results injected at turn start or after each sampling step (✔ codex).
- **Rewrite-only hooks**: plugins mutate messages, sampling params, headers and tool I/O per request but cannot end or continue a turn (opencode).

## Implementations
- [[pi--turn-lifecycle-hooks|pi]] — `prepareRequest`/`prepareNextTurn`/`finishTurn` in agent-core; coding-agent installs projection, threshold compaction, tool-loadout refresh and extension `turn_end` boundary through them.
- [[codex--turn-lifecycle-hooks|codex]] — `codex-hooks` crate: user-configured command/MCP-tool hooks with Claude-Code protocol at SessionStart, UserPromptSubmit, Stop/SubagentStop, Interrupt, SessionEnd…; per-step `StepContext` re-capture instead of request-rewrite callbacks.
- [[opencode--turn-lifecycle-hooks|opencode]] — plugin hooks `experimental.chat.messages.transform` (every step), `chat.params`, `chat.headers`, `tool.execute.before/after`.

## Failures
- [[threshold-check-misses-post-tool-request]]
- [[tool-loadout-stale-within-run]]
- [[plugin-hook-wall-clock-timeout]] (10-platform)
- [[unbounded-hook-continuation-loop]] (10-platform)
- [[threshold-check-misses-post-tool-request]] (05-context) — A large tool result pushed context over the threshold, but the follow-up provider request was sent before…

## Related
[[turn-loop]] · [[context-projection]] · [[auto-compaction]] · [[transcript-carried-system-prompt]] · [[virtual-model-router]] · [[extension-event-hooks]] · [[tool-call-gate]] · [[mid-turn-settings-switch]] · [[world-state-diff-injection]]

## Tradeoffs
- [[turn-cap-vs-none]]
