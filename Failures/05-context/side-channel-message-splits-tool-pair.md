---
type: failure
concepts: [out-of-band-message-deferral, transcript-replay-repair, message-conversion-layer]
harnesses: [pi, opencode]
---
**Symptom** — Extension messages sent with `triggerTurn: false` while the agent was running landed between an assistant tool call and its tool result; providers that validate message order rejected the replayed history (every later request). Separately, `triggerTurn: false` still steered the active run.

**Root cause** — The message was pushed into agent state and the session tree immediately, mid-turn.

**Fix · [[pi]]**
- `47b5119d0` 2026-08-12 (#8022) "trigger turn false should not start turn".
- `240eb29c4` 2026-08-25 (#8537) "Queue them during the run and append at turn_end, mirroring recordBashResult": `_pendingCustomMessages` flushed at `turn_end` (`packages/coding-agent/src/core/agent-session.ts:1183-1194`, `2319-2324`, `2347-2355`). User `!cmd` output already deferred via `_pendingBashMessages` (`agent-session.ts:3891-3898`).

**Fix · [[opencode]]** Guard at the provider boundary (v2, `76ee87ead8` 2026-06-03): a chronological system update positioned between a local tool call and its result is rejected — "Anthropic Messages system updates cannot split a local tool call from its tool result" (`splitsLocalToolResults`, `packages/llm/src/protocols/anthropic-messages.ts:375-385`, `:410-411`); updates are only admitted at a Safe Provider-Turn Boundary after settled tool results (`CONTEXT.md:99`).

**Lesson** — tool_use → tool_result adjacency is a provider invariant: anything injected mid-run waits for the turn boundary.

Related: [[out-of-band-message-deferral]] · [[transcript-replay-repair]] · [[orphaned-tool-calls-and-results]] · [[steering-queue]]
