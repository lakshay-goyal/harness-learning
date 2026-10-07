---
type: concept
stage: architecture
tier: candidate
aliases: ["pi.on", "41 extension events", "hooks (pre-2026)", "ExtensionAPI", "ExtensionEvent", "provider-payload-hook", "compaction-extension-hook", "before_provider_request", "session_before_compact"]
harnesses: [pi]
---
Typed lifecycle event bus where plugins observe, transform or veto harness behavior at fixed points (input, prompt build, per-request context, provider request/stream, tool call/result, session transitions, settle boundary).

## Why
- Lets a minimal core push every opinionated feature (permissions, plan mode, sub-agents, sandboxes, compaction strategy) out to plugins — pi's whole product stance rests on it ([[replaceable-builtin-extension]], [[no-permission-prompts]], [[no-plan-mode]], [[no-subagents-core]]).
- Without composition rules, multiple plugins silently clobber each other ([[tool-result-hook-patches-lost]]).
- Without fail-closed semantics, a crashing policy hook becomes a bypass ([[hook-error-fails-open]]).
- Without hiding harness-owned state, a well-meaning transform deletes invariants (system prompt / tool declarations) — see [[context-handler-drops-system-state]].
- Wall-clock timeouts on plugin code break legitimate human/LLM waits ([[plugin-hook-wall-clock-timeout]]).

## Design space
- **Hook shape**: notify-only events vs transform (return replaces value) vs veto (`{block}`/`{cancel}`) — pi mixes all three per event, typed per overload.
- **Composition**: last-handler-wins (pi pre-2026-02, rejected) vs **chained** (each handler sees the previous result; pi's choice) vs first-wins short-circuit (pi for `tool_call` block, `project_trust`, `input:"handled"`).
- **Mutation model**: return-value replacement (most pi events) vs in-place mutation (`tool_call.input`, `before_provider_headers`).
- **Failure policy**: swallow+report (pi default for notify events) vs **fail-closed** (pi for `tool_call`, `user_bash`) vs fail-open (pi pre-`509ee2bd0` for `user_bash`, rejected).
- **Timeouts**: per-hook wall clock (pi 2025-12, removed) vs none + user abort (pi now).
- **Granularity of raw access**: only semantic events vs also provider-payload/header/raw-stream taps below the model abstraction (pi exposes both).
- **Ordering vs other subscribers**: plugins before public SDK listeners (pi) vs same bus.
- **Process model**: in-process with full permissions (pi; no sandbox) vs out-of-process hooks (shell-command hooks, RPC).

## Implementations
- [[pi--extension-event-hooks|pi]] — 41 typed `pi.on` events, chained transforms, fail-closed `tool_call`/`user_bash`, in-process, no timeouts.

## Failures
- [[plugin-hook-wall-clock-timeout]]
- [[tool-result-hook-patches-lost]]
- [[pre-tool-hook-sees-stale-state]]
- [[hook-error-fails-open]]
- [[unbounded-hook-continuation-loop]]
- [[context-handler-drops-system-state]]
- [[side-door-input-bypasses-hooks]]
- [[hook-throw-aborts-parallel-batch]]

## Related
[[plugin-tools]] · [[runtime-plugin-loading]] · [[extension-ui-primitives]] · [[replaceable-builtin-extension]] · [[tool-call-gate]] · [[tool-result-rewriting]] · [[context-transform-hook]] · [[system-prompt-override]] · [[run-settlement]] · [[turn-lifecycle-hooks]] · [[agent-event-stream]] · [[project-trust-gate]] · [[custom-provider-registration]]
