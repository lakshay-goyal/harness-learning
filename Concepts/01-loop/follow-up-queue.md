---
type: concept
stage: loop
tier: candidate
aliases: ["followUp()", followUpMode, getFollowUpMessages, streamingBehavior followUp, deliverAs nextTurn, whenBusy followUp, final boundary]
harnesses: [pi]
---
Queue of messages delivered only when the agent would otherwise stop (no tool calls, no steering), starting another turn.

## Why
- Lets users (or background workers) line up the next instruction without interrupting current work.
- Every "agent would stop" exit must re-check it, or queued work strands until the next user message ([[queued-messages-stranded-at-run-end]]).
- Background subagents need a non-interrupting channel to report back (pi-durable research reports).

## Design space
- **Check point**: inner loop natural stop only (pi `runLoop`) · plus session driver after end-of-run hooks (pi `_handlePostAgentRun`).
- **Batching**: one-at-a-time (pi default) · all.
- **Failure semantics**: dropped on failed run · kept queued until next submission (pi-durable).
- **Variants**: attach to next user prompt (`deliverAs:"nextTurn"`, pi) · trigger a new run when idle.

## Implementations
- [[pi--follow-up-queue|pi]] — second `PendingMessageQueue` drained at natural stop + session-level `hasQueuedMessages()` re-check; durable follow-ups placed at `final` boundary.

## Failures
- [[queued-messages-stranded-at-run-end]]
- [[side-phase-input-lost]]

## Related
[[steering-queue]] · [[run-settlement]] · [[turn-loop]] · [[task-owned-subagent]]
