---
type: concept
stage: architecture
tier: candidate
aliases: ["AgentEvent", "AgentSessionEvent", "--mode json", "json-event-stream", "session.subscribe", "message_update"]
harnesses: [pi]
---
Typed lifecycle event protocol (run/turn/message/tool/queue/compaction/retry events) that every presentation — TUI, SDK subscriber, JSON stream, RPC client, plugins — consumes as the single source of UI truth.

## Why
- One event vocabulary keeps all front-ends consistent and lets plugins and hosts observe the same things.
- Ordering bugs between event dispatch and persistence corrupt sessions (tool results persisted before their tool call — see [[run-settlement]]).
- Streaming deltas that carry cumulative snapshots make output O(n²) ([[quadratic-event-stream-output]]).
- "Run ended" is ambiguous when retries/compaction may follow; a distinct settled event is required ([[run-settlement]]).

## Design space
- **Layering**: low-level loop events (`AgentEvent`) wrapped by session-level events adding retry/compaction/queue/settle (pi).
- **Delta vs snapshot**: cumulative partial message per update (pi in-process; cheap by reference) vs deltas only on the wire (pi JSON/RPC since `a4475344f`) vs snapshot-on-overflow (pi durable/Chord replicated state, see [[replicated-state]]).
- **Delivery**: awaited listeners (pi plugins) vs synchronous fire (pi public SDK listeners) vs buffered with backpressure.
- **Order across consumers**: plugins first, then public listeners (pi).
- **Wire framing**: JSONL header + events (pi `--mode json`) vs typed RPC events.
- **Durable alternative**: events derived from committed storage (`watchEvents()` in pi-durable) so remote UIs never see un-persisted state.

## Implementations
- [[pi--agent-event-stream|pi]] — `AgentEvent` (11 types) → `AgentSessionEvent` (+ settle/retry/compaction/queue/entry events); JSON mode strips cumulative snapshots.

## Failures
- [[quadratic-event-stream-output]]
- [[headless-protocol-stream-corruption]]

## Related
[[headless-rpc-mode]] · [[sdk-embedding]] · [[extension-event-hooks]] · [[run-settlement]] · [[turn-loop]] · [[unified-provider-api]] · [[replicated-state]] · [[differential-tui-rendering]]
