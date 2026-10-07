---
type: failure
concepts: [remote-host-trust, client-server-session-split]
harnesses: [pi, opencode]
---
**Symptom** — Arbitrary internal exceptions from session services (messages, stack-derived details) were serialized into protocol `response {ok:false, error}` frames sent to remote clients.

**Root cause** — The server mapped any thrown error to a wire error verbatim; no distinction between contract errors and internal failures.

**Fix · [[pi]]** — `dab70b2cd` 2026-08-05 (#7644) "sanitize service failures": `toProtocolError` passes only `ServerError`/`RemoteServiceError` codes, maps `ProtocolValidationError` → `invalid_request`, everything else → `{code:"internal_error", message: INTERNAL_SERVER_ERROR_MESSAGE}` and reports the cause locally (`packages/server/src/server.ts:512-521`; `packages/server/src/errors.ts:11,14-23`). Codec errors truncated to 500 chars (`packages/protocol/src/codec.ts:34-37`).

**Fix · [[opencode]]** — variant: internal **catalog records** (provider headers, bodies, API settings, possibly credentials) were serialized as public HTTP responses. v2 replaced internal catalog response schemas with explicit public DTOs: "Internal catalog records may contain credentials or provider-specific request material and must not cross the public HTTP serialization boundary" (`specs/v2/schema-changelog.md:395-405`). Provider error text is redacted by sensitive-name regex and bodies capped at 500 chars (`packages/llm/src/route/executor.ts:39-56`, `packages/llm/src/route/executor.ts:203-206`).

**Lesson** — At a trust boundary only stable, allow-listed error codes cross; internal causes stay in local logs.

Related: [[remote-host-trust]] · [[client-server-session-split]] · [[pi--remote-host-trust|pi]] · [[opencode--secret-handling|opencode]]
