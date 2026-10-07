---
type: concept
stage: architecture
tier: candidate
aliases: ["AgentEvent", "AgentSessionEvent", "--mode json", "json-event-stream", "session.subscribe", "message_update", "/event SSE", "/global/event", server.heartbeat, sessions.events]
harnesses: [pi, opencode]
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
- **Remote transport**: SSE per project instance plus a global SSE across instances, 10 s heartbeat (opencode); permission/question prompts travel as events.
- **Two-stream contract**: durable per-session replay stream with sequence cursor vs live instance stream without replay; neither auto-reconnects (opencode v2).
- **Payload safety**: clone payloads at publish (opencode `structuredClone`, [[event-payload-aliases-mutable-state]]).

## Implementations
- [[pi--agent-event-stream|pi]] — `AgentEvent` (11 types) → `AgentSessionEvent` (+ settle/retry/compaction/queue/entry events); JSON mode strips cumulative snapshots.
- [[opencode--agent-event-stream|opencode]] — bus events over `/event` and `/global/event` SSE drive TUI, web, `run`, ACP, share sync and workspace replication; v2 adds durable `sessions.events({after})`.

## Failures
- [[quadratic-event-stream-output]]
- [[headless-protocol-stream-corruption]]
- [[event-payload-aliases-mutable-state]]
- [[update-before-snapshot-on-subscribe]]

## Related
[[headless-rpc-mode]] · [[sdk-embedding]] · [[extension-event-hooks]] · [[run-settlement]] · [[turn-loop]] · [[unified-provider-api]] · [[replicated-state]] · [[differential-tui-rendering]] · [[client-server-session-split]]
