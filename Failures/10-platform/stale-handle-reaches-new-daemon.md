---
type: failure
concepts: [remote-execution-env, client-server-session-split]
harnesses: [pi]
---
**Symptom** — (design flaw) After a dropped SSH connection and lazy restart, a request carrying a file handle from the dead daemon could reach the fresh daemon, which never opened that handle number — potentially operating on a different file.

**Root cause** — Handles were numeric per connection with no connection identity on requests.

**Fix · [[pi]]** — `46d0ff936` 2026-10-05: every daemon start is a numbered `Session`; handle-bound requests carry `session` and fail after reconnect; in-flight requests on a lost session fail `{code:"unknown", lost:true}`; mutation outcome documented as unknown (`packages/env/src/connection.ts:85-115,164-167,190-200,307-320`; `docs/protocol.md:62-65`). Server-side analogue: attachment-fenced routing rejects delayed frames after re-attach (`packages/server/src/session-router.ts:224-232`).

**Lesson** — Tag replies and resource handles with a connection/attachment generation; fail rather than misroute after reconnect.

Related: [[remote-execution-env]] · [[client-server-session-split]] · [[pi--remote-execution-env|pi]]
