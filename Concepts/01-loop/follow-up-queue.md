---
type: concept
stage: loop
tier: candidate
aliases: ["followUp()", followUpMode, getFollowUpMessages, streamingBehavior followUp, deliverAs nextTurn, whenBusy followUp, final boundary, codex-ext-queue, QueuedItemService, start_turn_if_idle, "TurnInputMode::StartIfIdle", "turn_trigger queue", "MailboxDeliveryPhase::NextTurn", maybe_start_turn_for_pending_work, thread queue APIs, DeferMailboxPreemption]
harnesses: [pi, codex]
---
Queue of messages delivered only when the agent would otherwise stop (no tool calls, no steering), starting another turn.

## Why
- Lets users (or background workers) line up the next instruction without interrupting current work.
- Every "agent would stop" exit must re-check it, or queued work strands until the next user message ([[queued-messages-stranded-at-run-end]]).
- Background subagents need a non-interrupting channel to report back (pi-durable research reports; codex queue-only agent mail, [[subagent-result-mailbox]]).
- "Input is pending" must not override "turn failed terminally", or the queue becomes an unbounded retry loop ([[pending-input-restarts-failed-turn]]).

## Design space
- **Check point**: inner loop natural stop only (pi `runLoop`) · plus session driver after end-of-run hooks (pi `_handlePostAgentRun`).
- **Batching**: one-at-a-time (pi default) · all.
- **Failure semantics**: dropped on failed run · kept queued until next submission (pi-durable).
- **Variants**: attach to next user prompt (`deliverAs:"nextTurn"`, pi) · trigger a new run when idle (✔ codex `ext/queue` on `on_thread_idle` → `start_turn_if_idle`).
- **Locus**: in-loop queue (✔ pi) · client-side (TUI Tab queue submits on idle, ✔ codex) · persisted extension dispatching on thread idle, survives restarts (✔ codex `codex-ext-queue`).
- **After interrupt**: drain · suppress dispatch on interrupted idle (✔ codex `ThreadIdleCause::Interrupted`).
- **Agent mail at a final-answer boundary**: reopen sampling · defer queue-only mail to next turn unless the turn is reopened anyway (✔ codex `MailboxDeliveryPhase::{CurrentTurn,NextTurn}`).
- **Leftover pending input at task end**: drop · record into history via hooks for the next request (✔ codex).

## Implementations
- [[pi--follow-up-queue|pi]] — second `PendingMessageQueue` drained at natural stop + session-level `hasQueuedMessages()` re-check; durable follow-ups placed at `final` boundary.
- [[codex--follow-up-queue|codex]] — no in-loop user follow-up queue; TUI Tab queue, durable `ext/queue` dispatch on thread idle (not after interrupt), agent-mail `NextTurn` delivery + `maybe_start_turn_for_pending_work`.

## Failures
- [[queued-messages-stranded-at-run-end]]
- [[side-phase-input-lost]]
- [[pending-input-restarts-failed-turn]]

## Related
[[steering-queue]] · [[run-settlement]] · [[turn-loop]] · [[task-owned-subagent]] · [[subagent-result-mailbox]] · [[persistent-goal-continuation]] · [[single-active-task-slot]]
