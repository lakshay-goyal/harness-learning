---
type: failure
concepts: [client-server-session-split]
harnesses: [pi]
---
**Symptom** — After an error the Radius session relay never reconnected; also pi couldn't exit while the relay was reconnecting.

**Root cause** — Code called `socket.close(1002/1011)`; undici's browser-style WebSocket only allows sending 1000 or 3000–4999, so `close()` threw and the reconnect path never ran. Codes you may *receive* (RFC 1002/1011) aren't codes you may *send*.

**Fix · [[pi]]** — `35c49350f` 2026-08-28: local errors use 4000 (protocol) / 4001 (transport), `close()` wrapped in try/catch (`packages/coding-agent/src/experimental/radius-relay.ts:21-25,683-689`); `f55da4a9d` 2026-08-28 allow exit while reconnecting. Reconnect: host 1 s → 30 s backoff; client re-attaches last session (`radius-relay.ts:16-20,343-401`).

**Lesson** — Use the application close-code range for your own errors and make cleanup paths unable to throw.

Related: [[client-server-session-split]] · [[pi--client-server-session-split|pi]]
