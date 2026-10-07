---
type: concept
stage: failure-handling
tier: must-have
aliases: [effect sandwich, "replay: safe/unsafe", ToolTaskCheckpoint, interrupted result, effect-sandwich, tool-replay-policy]
harnesses: [pi, opencode]
---
Commit the intent (final arguments + replay policy) before performing a tool's external effect and the outcome after; on recovery from a crash in between, re-run only tools declared replay-safe and give every other interrupted call an explicit "interrupted, may have partially run" result.

## Why
- A durable agent that resumes from storage cannot know whether an in-flight side effect happened.
- Blindly re-running non-idempotent tools duplicates effects or clobbers newer work ([[non-idempotent-tool-replayed-after-crash]]); silently dropping them leaves orphan tool calls.

## Design space
- No durability: crash loses the turn (stable pi coding-agent; orphan calls repaired at replay → [[transcript-replay-repair]]).
- **Effect sandwich with per-tool replay policy** (pi durable): rerun only when stored *and* current policy are safe; default unsafe.
- Poll an external handle instead of rerunning (pi durable for deferred provider responses).
- Idempotency keys (`requestId`) so a rerun finds prior work (pi durable subagent/vacation research).
- codex: absent — no intent/outcome sandwich or replay policy; persisted calls without outputs get a synthetic "aborted" output at prompt build and are never re-run (`codex-rs/core/src/context_manager/normalize.rs:21-115`; `1e3cad95c0` 2025-12-14 after resume panics, [[session-switch-leaves-dangling-tool-calls]]); interrupted calls get "aborted by user after Xs" results that warn "they may have partially executed" (`e95abcdf49:codex-rs/core/src/tools/parallel.rs:363-381`; `codex-rs/core/src/context/turn_aborted.rs:10`).
- **Fail-all, never rerun** (opencode v2): durably project each call before execution; on the next drain mark every still-running call `Tool execution interrupted`; retry policy deferred.

## Implementations
- [[pi--crash-safe-tool-replay|pi]] — `pi.tool` task phases `call`/`execute`; intent `{phase:"execute", arguments, replay}`; recovery reruns only safe tools, else "Tool X was interrupted and may have partially run".
- [[opencode--crash-safe-tool-replay|opencode]] — v2 `failInterruptedTools` fails every `pending`/`running` call before each drain; legacy answers dangling calls with "[Tool execution was interrupted]" at replay.

## Failures
- [[non-idempotent-tool-replayed-after-crash]]
- (01, codex variant: resume panicked on an unpaired call) [[session-switch-leaves-dangling-tool-calls]]
- [[settled-tool-vanishes-before-placement]] (08-state) — In the experimental agent-core harness (removed in 7fd478a2e), a tool call whose result was already committed…

## Related
[[durable-execution]] · [[task-owned-subagent]] · [[tool-error-as-result]] · [[parallel-tool-execution]] · [[transcript-replay-repair]]
