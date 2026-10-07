---
type: concept
stage: loop
tier: candidate
aliases: [prepareRequest, prepareNextTurn, prepareNextTurnWithContext, finishTurn, turn-hooks, onYield, afterResponse, afterTools, chat.params, chat.headers]
harnesses: [pi, opencode]
---
Low-level loop callbacks that rewrite the next request (context/model/thinking), append messages before a turn, or end/force a turn — the seam where session concerns plug into a stateless loop.

## Why
- Without a per-request hook, context is computed once per user prompt; tool/loadout changes and compaction thresholds go stale mid-run ([[tool-loadout-stale-within-run]]).
- A per-turn decision point lets extensions end runs or demand one more response without patching the loop.
- Keeps the loop provider- and storage-agnostic: projection from a persistent log, model routing, compaction all live in hooks.

## Design space
- **Granularity**: before every request incl. first (`prepareRequest`) · only before continuing turns (`prepareNextTurn`) · after turn (`finishTurn`).
- **Decision vocabulary**: end / continue / default (pi); continue is satisfied by existing scheduling, not additive.
- **Composition**: decorator chain wrapping previous hook (pi coding-agent) · named hook chains in a registry (pi-durable).
- **Queue interaction**: hooks do not poll queues (pi `prepareRequest`); re-poll after long `prepareNextTurn` work.
- **Safety**: unconditional continue = endless loop; pairs with [[no-turn-cap]].
- **Rewrite-only hooks**: plugins mutate messages, sampling params, headers and tool I/O per request but cannot end or continue a turn (opencode).

## Implementations
- [[pi--turn-lifecycle-hooks|pi]] — `prepareRequest`/`prepareNextTurn`/`finishTurn` in agent-core; coding-agent installs projection, threshold compaction, tool-loadout refresh and extension `turn_end` boundary through them.
- [[opencode--turn-lifecycle-hooks|opencode]] — plugin hooks `experimental.chat.messages.transform` (every step), `chat.params`, `chat.headers`, `tool.execute.before/after`.

## Failures
- [[threshold-check-misses-post-tool-request]]
- [[tool-loadout-stale-within-run]]

## Tradeoffs
- [[turn-cap-vs-none]]

## Related
[[turn-loop]] · [[context-projection]] · [[auto-compaction]] · [[transcript-carried-system-prompt]] · [[virtual-model-router]] · [[extension-event-hooks]] · [[tool-call-gate]]
