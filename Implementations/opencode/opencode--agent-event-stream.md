---
type: implementation
harness: opencode
concept: agent-event-stream
commit: ecc4916b5a
files: [packages/opencode/src/server/routes/instance/httpapi/handlers/event.ts:40-75, packages/opencode/src/server/routes/instance/httpapi/handlers/global.ts:35-37, packages/opencode/src/session/llm.ts:208-222, CONTEXT.md:156-170, specs/v2/session.md:175-183]
---
[[agent-event-stream]] in [[opencode]].

## Mechanism
### Legacy runtime
- **Bus events drive every UI**: message updated, part updated, part delta, session status (busy/retry/idle), error, diff, compacted, `permission.asked`, `question.asked`, installation `UpdateAvailable`. No UI state lives in the loop.
- **Wire**: SSE `GET /event` per instance (opens with `server.connected`, heartbeat 10 s, closes on `server.instance.disposed`) and `GET /global/event` across instances as `{directory, payload}` (`packages/opencode/src/server/routes/instance/httpapi/handlers/event.ts:40-75`; `packages/opencode/src/server/routes/instance/httpapi/handlers/global.ts:24-45`).
- **Consumers**: TUI (worker RPC event source), web app, `opencode run` (prints parts, answers permissions), ACP (`sessionUpdate`), GitHub agent, share sync (subscribes to session/message/part/diff events, [[session-export-share]]), workspace replication.
- **Tracing**: AI SDK telemetry spans tagged with `session.id` behind `experimental.openTelemetry` (`packages/opencode/src/session/llm.ts:208-222`) ([[install-telemetry]]).

### v2 runtime
- Two public streams with different contracts: `sessions.events({sessionID, after})` = durable replay after an aggregate sequence then tail of newly committed events, excludes live fragments; `events.subscribe()` = instance-wide live stream with heartbeats, no replay (`CONTEXT.md:165-166`). Neither reconnects automatically; callers resume with `after` (`CONTEXT.md:156`, `CONTEXT.md:169`).
- Finite history pages (default 50, max 100); tail wake-up = one sliding-capacity-1 dirty signal per tail, subscribed before replay so no commit is missed (`specs/v2/session.md:175-183`) ([[unbounded-subscriber-buffering]]).

## Constants
| name | value | path:line |
|---|---|---|
| heartbeat | 10 s | `packages/opencode/src/server/routes/instance/httpapi/handlers/event.ts:63` |

## Evolution
- 2026-03-02 `fd6f7133c5` bus events carried the live part object; later mutation changed already-published token values → `structuredClone` ([[event-payload-aliases-mutable-state]]).
- 2026-05-18 `cb35493242` `/event` PubSub subscription acquired eagerly to close a connect/subscribe race ([[update-before-snapshot-on-subscribe]]).
- 2026-06-26 `65210f2d97` v2 finite durable session history pages.

## Quirks / drift
- Legacy events are in-memory bus fan-out; a client that disconnects must refetch state (no replay on `/event`).

pi contrast: typed in-process `AgentEvent` → `AgentSessionEvent` layers with JSONL/RPC framing; durable variant derives events from committed storage ([[pi--agent-event-stream|pi]]).
