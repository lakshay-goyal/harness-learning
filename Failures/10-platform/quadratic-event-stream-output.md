---
type: failure
concepts: [agent-event-stream, headless-rpc-mode]
harnesses: [pi]
---
**Symptom** — `--mode json` / RPC output grew O(n²) with response length: every `message_update` carried the full cumulative partial message, so long answers produced gigabytes of JSON (#7290).

**Root cause** — The in-process event shape (cheap: partial message passed by reference) was serialized verbatim to the wire.

**Fix · [[pi]]** — `a4475344f` 2026-08-03 (#7394): JSON streaming strips cumulative `partial`/`message` snapshots from `message_update`, keeping deltas + constant-size usage (`packages/coding-agent/src/modes/json-event.ts:40-61`).

**Lesson** — In-process event shapes are not wire shapes; stream deltas over process boundaries and send snapshots only on (re)sync.

Related: [[agent-event-stream]] · [[headless-rpc-mode]] · [[replicated-state]] · [[quadratic-event-queue-drain]] · [[pi--agent-event-stream|pi]]
