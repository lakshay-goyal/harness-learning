---
type: implementation
harness: pi
concept: turn-lifecycle-hooks
commit: b30a6dd77
files: [packages/agent/src/types.ts:146, packages/agent/src/types.ts:255, packages/agent/src/agent-loop.ts:185, packages/agent/src/agent-loop.ts:219, packages/agent/src/agent-loop.ts:286, packages/coding-agent/src/core/agent-session.ts:787, packages/coding-agent/src/core/agent-session.ts:886, packages/coding-agent/src/core/agent-session.ts:898]
---
[[turn-lifecycle-hooks]] in [[pi]].

## Mechanism
- Three low-level `AgentLoopConfig` hooks (`packages/agent/src/types.ts:255-283`), documented lifecycle: `selected input events → prepareRequest → provider response → tool results → finishTurn → turn_end → existing continuation scheduling or agent_end` (`packages/agent/README.md:157-167`).
- **`prepareRequest(request, signal)`** — immediately before every conversational provider request **including the first**; pending msgs already appended; returned context/model/thinkingLevel replace runtime values for this and later requests; does **not** poll queues (`types.ts:180-189`, `266-271`; loop `agent-loop.ts:219-239`; README `:126-134`).
- **`prepareNextTurn(lastCompletedTurn)`** — after `turn_end` only when the loop will continue, before next `turn_start`; may replace context/model/thinking and append messages (`AgentLoopTurnUpdate`, `types.ts:160-170`, `273-283`; loop `agent-loop.ts:185-200`). `"off"` thinking maps to `undefined` reasoning (`:193-198`). Agent variant `prepareNextTurnWithContext` receives the completed turn + signal (`agent.ts:130`, `485-488`).
- **`finishTurn(turn, signal)`** — after assistant + all tool results, before `turn_end`; returns `{action:"end"}` (stop now, no queue poll, no `prepareNextTurn`) | `{action:"continue"}` (ensure one more request; satisfied by tools/steering/follow-up if they already cause one, else one context-only turn) | undefined (`types.ts:146-158`, `257-264`; loop `:280-298`, `:310-314`). Error/aborted responses: called but decision ignored (`agent-loop.ts:245-256`). README: unconditional continue = endless loop (`packages/agent/README.md:146`) → [[no-turn-cap]].
- **Coding-agent installs decorators** that wrap any previously set hook (chain pattern):
  - `_installAgentRequestProjection` (`agent-session.ts:787-844`): swaps `context.messages` for `sessionManager.buildSessionProjection().messages` + current executable tools (`:793-799`) → [[context-projection]]; virtual model: routes via `_modelRuntime.resolveModel` with `reason: failed ? "retry" : userTurn ? "user" : "continuation"` and `failed` response (`:816-828`) → [[virtual-model-router]]; threshold compaction against the **routed** physical model, then re-prepare (`:838-841`).
  - `_installAgentBoundaryHooks` → `finishTurn` dispatches extension `turn_end` boundary (drafts committed, may request continuation; rejected if context can't continue) then defers to previous decision; `end` wins (`agent-session.ts:846-896`).
  - `_installAgentNextTurnRefresh` → `prepareNextTurnWithContext`: `_compactBeforeNextAssistantResponse` (threshold compaction between turns, skipped for virtual models, `:776-785`) → previous hook → rebuild system-prompt options with current active tools/snippets/guidelines → `_preparePromptAndToolLoadout` emits a sections/tools delta system message → returns fresh tools/model/thinking (`:898-932`) → [[transcript-carried-system-prompt]], [[auto-compaction]].
- Plus per-tool hooks `beforeToolCall`/`afterToolCall` (`types.ts:319-341`) → [[tool-call-gate]], [[tool-result-rewriting]].

## Constants
none.

## Evolution
- 2026-05-09 `322759a3f` "snapshot harness turn state" — first `prepareNextTurn` in `types.ts` (git `-S`).
- 2026-06-30 `e547bb9f4` refresh session state before next turn; `fd6659dd5` preserve run prompt during tool refresh (#6162) → [[tool-loadout-stale-within-run]].
- 2026-08-28 `56700d42e` compact before post-tool model requests (#8782); `prepareNextTurn` now only runs when another turn actually happens.
- 2026-09-21 `466db0fec` "canonical session context boundaries": `prepareRequest` + `finishTurn` added (git `-S` on `types.ts`); projections authoritative; "preserve existing queue scheduling during continuation and recovery".

## Evidence commits
`322759a3f` `e547bb9f4` `fd6659dd5` `56700d42e` `466db0fec`

## Quirks
- `prepareRequest` steering blind spot: steering queued while it runs (e.g. during a routed-model compaction) waits for the next normal poll (`packages/agent/README.md:134`).
- `prepareNextTurn` re-polls steering afterwards only if the earlier poll was empty (`agent-loop.ts:201-206`).

## Durable variant (packages/durable)
- Hooks are named chains on the extension registry: `beforeTool`/`afterTool`, `afterResponse` (`generation.ts:450`), `afterTools` (`:612`), `onYield` (continue after stop/length, `spec.md:3536-3566`), `beforeCompact` (`harness/compaction.ts:128-133`). Prompt sections re-rendered per `prepare` phase; only diffs appended as `pi.system` (`spec.md:3487-3495`).

## Failures
[[tool-loadout-stale-within-run]]
