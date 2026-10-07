---
type: implementation
harness: pi
concept: out-of-band-message-deferral
commit: b30a6dd77
files: [packages/coding-agent/src/core/agent-session.ts:398, packages/coding-agent/src/core/agent-session.ts:1191, packages/coding-agent/src/core/agent-session.ts:2292, packages/coding-agent/src/core/agent-session.ts:2347, packages/coding-agent/src/core/agent-session.ts:3891, packages/coding-agent/src/core/agent-session.ts:1842, packages/coding-agent/src/core/agent-session.ts:2088]
---
[[out-of-band-message-deferral]] in [[pi]].

## Mechanism
- Three buffers (`packages/coding-agent/src/core/agent-session.ts:398-421`): `_pendingNextTurnMessages`, `_pendingCustomMessages`, `_pendingBashMessages`.
- **`sendCustomMessage` delivery matrix** (`packages/coding-agent/src/core/agent-session.ts:2292-2328`):
  - `deliverAs:"nextTurn"` → `_pendingNextTurnMessages`, injected alongside the next user prompt (`packages/coding-agent/src/core/agent-session.ts:2088-2091`).
  - streaming + `triggerTurn !== false` → `agent.followUp()` / `agent.steer()` (model input at next boundary, [[steering-queue]]).
  - not streaming + `triggerTurn` → start a run (deferred if emitted during `agent_settled`).
  - **streaming + `triggerTurn:false`** → `_pendingCustomMessages`; comment: "Appending now would put the message between an assistant tool call and its result, which providers that validate message order reject on replay. Defer to the end of the turn. Nothing is emitted yet: message events must not describe messages the session tree does not contain."
  - else append immediately.
- **Flush points**: custom messages at `turn_end` — "the first point in the run where a context-only custom message can be inserted without landing between a tool call and its result"; flushed after extension/listener dispatch so `turn_end` handlers' messages are included (`packages/coding-agent/src/core/agent-session.ts:1183-1194`, `_flushPendingCustomMessages` `packages/coding-agent/src/core/agent-session.ts:2347-2355`). Both buffers flushed again in the run's `finally` before `agent_settled` (`packages/coding-agent/src/core/agent-session.ts:1842-1849`).
- **User shell `!cmd`** during streaming: `bashExecution` message queued in `_pendingBashMessages` "to avoid breaking tool_use/tool_result ordering" (`packages/coding-agent/src/core/agent-session.ts:3891-3898`), flushed when the run settles (`packages/coding-agent/src/core/agent-session.ts:3924-3932`); not streaming → appended immediately. `!!cmd` sets `excludeFromContext` (never reaches the model).
- Boundary previews (`agent_before_settle`) count pending custom messages as context that lets a run continue (`packages/coding-agent/src/core/agent-session.ts:994-1012`).
- Provider-boundary safety net: pi-ai `transformMessages` holds mid-transcript system messages that fall between a call and its results until results are emitted (`packages/ai/src/api/transform-messages.ts:163-166`, `216-221`) → [[transcript-replay-repair]].

## Evolution
- 2026-08-12 `47b5119d0` (#8022) `triggerTurn:false` no longer starts/steers a turn.
- 2026-08-25 `240eb29c4` (#8537) "append run-time custom messages after the turn's tool results" — `_pendingCustomMessages` flushed at `turn_end`.
- Bash deferral predates (comment "flushed on agent_end"; now flushed at settle by the driver loop `32bcdc973`/`c685b2736`).

## Evidence commits
`47b5119d0` `240eb29c4` `32bcdc973` `9e05370b2`

## Quirks
- `!cmd` output typed during a long run is invisible to the model until the run settles (by design: flushed at settle, not turn end) — the model can't react to it mid-run (inferred).
- Queued custom messages are not mirrored in the UI steering/follow-up string lists (01-loop quirk).

## Failures
[[side-channel-message-splits-tool-pair]]
