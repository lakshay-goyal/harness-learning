---
type: concept
stage: subagents
tier: variant
aliases: [Mailboxes, InterAgentCommunication, InterAgentCompletionMessage, "Message Type: FINAL_ANSWER", notify_parent_of_terminal_turn, "MessageDeliveryMode::QueueOnly", "MessageDeliveryMode::TriggerTurn", trigger_turn, WaitOutcome, PENDING_MAILBOX_MESSAGES, has_pending_mailbox_items]
harnesses: [codex]
---
A child's terminal outcome (and any agent-to-agent message) is delivered asynchronously as a typed message into the recipient's mailbox — queue-only by default, not starting a turn; the parent picks it up at its next step boundary or by blocking on a generic "wait for mailbox activity" tool whose output carries no content.

## Why
- Non-blocking delegation needs a channel back that doesn't interrupt the parent's current step or break tool-call/result adjacency ([[out-of-band-message-deferral]]).
- Waits must check already-queued mail (lost wakeup) and must yield to the user ([[wait-misses-already-queued-result]], [[blocking-wait-ignores-user-steer]]); models busy-poll waits with tiny timeouts ([[model-polls-background-work]]).
- If workers can be unloaded for capacity, their inbox must outlive them ([[subagent-eviction-loses-mail]]).

## Design space
- **Return channel**: tool result of a blocking call (pi [[subagent-as-subprocess]]) · follow-up message (pi-durable background) · typed mailbox message, assistant-role envelope (✔ codex).
- **Delivery mode**: queue-only, picked up at next boundary (✔ codex `send_message`, child completion) · trigger-turn, wakes an idle target (✔ codex `followup_task`).
- **Boundary**: next sampling step · mid-stream preemption after reasoning/commentary (✔ codex, opt-out flag) · deferred to next turn after a final answer ([[follow-up-queue]]).
- **Wait tool**: returns content · returns only "completed / interrupted by new input / timed out", content arrives via mailbox (✔ codex V2); floor-clamped timeout reported in result (✔ codex).
- **Payload limits**: truncate everything (pi 50 KiB example) · final answer untruncated, only error text capped + next-action hint (✔ codex).
- **Mailbox residency**: inside the worker session · runtime-held, survives unload (✔ codex).

## Implementations
- [[codex--subagent-result-mailbox|codex]] — `notify_parent_of_terminal_turn` → `InterAgentCommunication{trigger_turn:false}`; `FINAL_ANSWER` envelope; `wait_agent` waits for mailbox activity / steer / timeout (30 s default, 10 s floor, 1 h max).

## Failures
- [[wait-misses-already-queued-result]]
- [[blocking-wait-ignores-user-steer]]
- [[subagent-eviction-loses-mail]]
- [[queued-messages-stranded-at-run-end]] (01-loop)

## Related
[[in-process-subagent-threads]] · [[task-owned-subagent]] · [[follow-up-queue]] · [[steering-queue]] · [[out-of-band-message-deferral]] · [[agent-message-board]] · [[run-settlement]] · [[delegated-authorization-provenance]] · [[builtin-subagents-vs-none]]
