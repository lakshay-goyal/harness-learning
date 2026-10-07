---
type: concept
stage: failure-handling
tier: candidate
aliases: [effect sandwich, "replay: safe/unsafe", ToolTaskCheckpoint, interrupted result, effect-sandwich, tool-replay-policy]
harnesses: [pi]
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

## Implementations
- [[pi--crash-safe-tool-replay|pi]] — `pi.tool` task phases `call`/`execute`; intent `{phase:"execute", arguments, replay}`; recovery reruns only safe tools, else "Tool X was interrupted and may have partially run".

## Failures
- [[non-idempotent-tool-replayed-after-crash]]

## Related
[[durable-execution]] · [[task-owned-subagent]] · [[tool-error-as-result]] · [[parallel-tool-execution]] · [[transcript-replay-repair]]
