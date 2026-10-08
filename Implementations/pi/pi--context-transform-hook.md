---
type: implementation
harness: pi
concept: context-transform-hook
commit: b30a6dd77
files: [packages/agent/src/agent-loop.ts:388, packages/coding-agent/src/core/sdk.ts:436, packages/coding-agent/src/core/extensions/runner.ts:285, packages/coding-agent/src/core/extensions/runner.ts:1298, packages/coding-agent/src/core/extensions/types.ts:866, packages/coding-agent/src/core/agent-session.ts:1766, packages/coding-agent/src/core/agent-session.ts:787]
---
[[context-transform-hook]] in [[pi]].

## Mechanism
- **Core hook** `transformContext(messages, signal)` applied before `convertToLlm` on every request (`packages/agent/src/agent-loop.ts:388-391`); doc examples: "Context window management (pruning old messages)", "Injecting context from external sources"; must not throw.
- **Request replacement hook** `prepareRequest` (runs immediately before every provider request, may replace context/model/thinking; `packages/agent/src/agent-loop.ts:219-239`). AgentSession installs it to swap `context.messages` for `sessionManager.buildSessionProjection().messages` (the session log is the source of truth; `packages/coding-agent/src/core/agent-session.ts:787-844`, `466db0fec`) and to route virtual models.
- **Coding-agent chain** (each wrapper calls the previous):
  1. sdk: extension `runner.emitContext(messages)` (`packages/coding-agent/src/core/sdk.ts:436-440`; returns messages unchanged when no runner).
  2. `_installHiddenDeclarationsProjection`: strips tool declarations hidden by `prepareLoadout` from system deltas (`packages/coding-agent/src/core/agent-session.ts:1766-1784`).
  3. `_installAgentForcedPromptProjection`: collapses system messages into one head with forced text; not recorded in the transcript (`packages/coding-agent/src/core/agent-session.ts:1786-1801`; `16292398a`).
- **`emitContext` two phases** (`packages/coding-agent/src/core/extensions/runner.ts:1298-1360`), on a `structuredClone` of the messages (history never mutated):
  - `context` handlers see the conversation **without system messages**; may return a new list or mutate `event.messages` in place; Pi restores prompt/tool state after each via `restoreSystemMessages` (unchanged → keep current; changed → prepend `getCurrentSystemMessage(current)` as one leading system message "so pruning, windowing, or slicing from a compaction summary cannot drop them", `packages/coding-agent/src/core/extensions/runner.ts:276-293`).
  - `context_with_system` handlers then see the full transcript and their output is used as returned; if they remove the leading system message Pi reports "Handler removed the leading system message; the request has no prompt or initial tool declarations…" but honors the output.
  - Handler throws → `emitError`, continue with previous messages.
- Event types documented in `packages/coding-agent/src/core/extensions/types.ts:866-884`.
- Other request-time hooks: `before_provider_request` (replace wire payload), `before_agent_start` (system prompt mutation) — see [[extension-event-hooks]], [[system-prompt-override]].

## Evolution
- 2026-09-16 `9e05370b2` (#9548) prompt + tools moved into transcript system messages → slicing handlers could now delete them.
- 2026-09-17 `16292398a` forced system prompts sent without recording.
- 2026-09-21 `466db0fec` canonical projection installed via `prepareRequest`; `aef5fc429` (#9846) `context` handlers see conversation only, `context_with_system` added.

## Evidence commits
`9e05370b2` `16292398a` `466db0fec` `aef5fc429` `e4c75a732`

## Quirks
- When a `context` handler changes anything, a model with native mid-conversation system messages gets the collapsed head instead of in-place deltas → cached prefix may change (comment `packages/coding-agent/src/core/extensions/runner.ts:276-284`; inferred cache cost).
- No built-in pruning beyond compaction uses this hook in core (01-loop open question; none found).

## Failures
[[context-handler-drops-system-state]]
