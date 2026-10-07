---
type: concept
stage: subagents
tier: candidate
aliases: [durable subagent tool, "ownership: {kind: task}", completing hold, structured-concurrency-ownership, background task, failFast, allSettled, keyed-service-generations, research task, scanConversations ownerTaskId]
harnesses: [pi]
---
A subagent is a child conversation owned by the tool task that spawned it, under structured concurrency: abort flows down, the owner cannot finish before owned work drains, background tasks are explicit boundaries.

## Why
- Subprocess subagents are invisible to the harness's lifecycle: aborts, idle, crash recovery and accounting must be reimplemented per tool.
- Without an ownership tree, cancellations race (children outlive or miss parent aborts) ([[ownership-cancellation-races]]).
- Crash-resumable runtimes need idempotent spawn/submit so a replayed tool call finds the same child, and must not replay non-idempotent control operations ([[non-idempotent-tool-replayed-after-crash]]).
- Usage accounting double counts if the tool also reports the child's spend.

## Design space
- **Ownership**: task-owned child conversation (pi-durable) · ownerless + manual bookkeeping · subprocess ([[subagent-as-subprocess]]).
- **Foreground vs background**: tool waits for child answer (holds the run) · background anchor task that reports back as a follow-up message (pi-durable vacation / example 23).
- **Idempotence**: ownership index lookup + per-task `requestId` on submissions → `replay:"safe"` · not replay-safe for stop/kill.
- **Join policy**: `failFast` (abort siblings on first failure) · `allSettled`.
- **Abort order**: bottom-up so compensating handlers see final child outcomes.
- **Recursion**: child config removes the subagent extension.
- **Interrogation**: child conversation survives the call so users can switch to it.
- **Exposure to remote UIs**: keyed service instances per conversation with generation fencing (planned).

## Implementations
- [[pi--task-owned-subagent|pi]] — pi-durable ownership tree (Package 18/19) + experimental TUI `subagent` tool and vacation `research` background subagent.

## Failures
- [[ownership-cancellation-races]]
- [[non-idempotent-tool-replayed-after-crash]]

## Related
[[subagent-as-subprocess]] · [[durable-execution]] · [[crash-safe-tool-replay]] · [[abort-propagation]] · [[follow-up-queue]] · [[run-settlement]] · [[client-server-session-split]] · [[replicated-state]] · [[no-subagents-core]]
