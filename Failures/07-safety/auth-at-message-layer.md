---
type: failure
concepts: [remote-host-trust, client-server-session-split]
harnesses: [pi]
---
**Symptom** — The experimental session protocol authenticated clients with a framed `auth {token}` message inside the CBOR protocol, coupling auth to protocol versions and every transport (Unix socket, relay) to one bearer scheme.

**Root cause** — Authentication placed at the message layer rather than delegated to the byte transport that already has an identity mechanism (filesystem permissions, WebSocket upgrade headers).

**Fix · [[pi]]** — `305c014dc` 2026-08-03 (#7551) "make session authentication transport-specific": framed auth removed, protocol version reset 2 → 1; transports must hand over "already-authorized bytes" (`packages/server/src/listener.ts:3-8`); Unix sockets `0o600` in `0o700` dirs (`packages/server/src/transports/unix/listener.ts:11,53`); Radius relay `Authorization: Bearer` on the WebSocket upgrade (`packages/coding-agent/src/experimental/radius-relay.ts:626-646`, `1d0d110ab`). Protocol README: "Peer authentication and authenticated service contexts are not implemented by the experimental transport" (`packages/protocol/README.md:39`).

**Lesson** — Authenticate at the transport; keep the message protocol auth-free so each transport uses its native identity mechanism.

Related: [[remote-host-trust]] · [[client-server-session-split]] · [[pi--remote-host-trust|pi]]
