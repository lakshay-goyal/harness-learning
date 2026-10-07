---
type: concept
stage: loop
tier: candidate
aliases: ["steer()", steeringMode, getSteeringMessages, streamingBehavior steer, pi.inbox, whenBusy, durable-inbox, postTools boundary, queueMode]
harnesses: [pi]
---
Mid-run user messages queued and injected at the next turn boundary (after the current tool batch) without cancelling in-flight tools.

## Why
- Users type while the agent works; without a queue input is dropped, rejected, or forces an abort that discards work ([[side-phase-input-lost]]).
- Delivering the interrupt mid-batch leaves model-issued tool calls unexecuted with fake results → model confusion ([[steering-skips-pending-tool-calls]]).
- Injecting between a tool call and its result breaks provider adjacency rules ([[out-of-band-message-deferral]]).

## Design space
- **Delivery point**: after each tool call, skipping the rest (pi 2025-12, reverted) · after the whole batch (pi HEAD) · only at end of run (= follow-up).
- **Batching**: one-at-a-time (pi default) · all at once.
- **Mid-run hard cut**: separate abort that returns queued text to the editor (pi Esc).
- **Persistence**: in-memory queue (pi stable) · durable inbox document with requestId dedupe, deterministic placement by item ID (pi-durable).
- **API**: single queueMessage (pi early) · split `steer()`/`followUp()` (pi `d0a4c3702`) · `whenBusy: steer|followUp|reject` on submit (pi-durable).
- **Side phases** (compaction, summaries, navigation): reject input · queue and drain after (pi, after ≥9 fixes).

## Implementations
- [[pi--steering-queue|pi]] — `PendingMessageQueue` polled at run start, after `prepareNextTurn`, after each tool batch; durable `pi.inbox` with `postTools`/`final` boundaries.

## Failures
- [[steering-skips-pending-tool-calls]]
- [[side-phase-input-lost]]
- [[queued-messages-stranded-at-run-end]]
- [[reentrant-prompt-corrupts-state]]

## Related
[[follow-up-queue]] · [[turn-loop]] · [[abort-propagation]] · [[out-of-band-message-deferral]] · [[run-settlement]] · [[auto-compaction]]
