---
type: failure
concepts: [http-transport-hardening]
harnesses: [pi]
---
**Symptom**
- Valid connections failed on high-latency routes (satellite, remote links).
- The process crashed when a stream terminated mid-flight.
- Long thinking pauses tripped dead-socket timeouts.

**Root cause** — Node and undici defaults are tuned for LANs:
- Happy-eyeballs `autoSelectFamilyAttemptTimeout` is 250 ms.
- An undici Client/Pool `error` event with no listener is fatal.
- The default body and header timeouts are too short for long thinking pauses.

**Fix · [[pi]]**
- `849f9d9c5` 2026-05-20 — configurable HTTP idle timeout: `bodyTimeout = headersTimeout`, default 300 s, choices 30s/1m/2m/5m/off (#4759) (`packages/coding-agent/src/core/http-dispatcher.ts:4,8-14,81-99`).
- `2117b61c6` 2026-06-30 — no-op `error` listener on the undici Client/Pool (#6133) (`:52-62`).
- `14551e769` 2026-08-02 — attempt timeout 2000 ms (#7315/#7435) (`:5-6`).

**Fix · [[pi]]** (earlier steps, from fix-mining `10-platform`) — long local-LLM SSE streams (vLLM buffering a large tool call) aborted at 5 min with `UND_ERR_BODY_TIMEOUT`; Node 26 idle streams timed out; undici HTTP/2 races crashed the CLI:
- `ea90a6783` 2026-04-27 — `bodyTimeout:0, headersTimeout:0` on the global dispatcher (`packages/coding-agent/src/cli.ts:20` at `ea90a6783`; cli.ts is now a 6-line shim, dispatcher setup lives in `packages/coding-agent/src/core/http-dispatcher.ts:81-99`) (#3715).
- `e26fbb3d4` 2026-05-17 — route global `fetch` through the undici dispatcher (`packages/coding-agent/src/cli.ts:20-26` at `e26fbb3d4`; HEAD `installedGlobalFetch` in `http-dispatcher.ts:16-17`) (#4519).
- `967fb4d8b` 2026-05-18 — disable undici HTTP/2 (#4681); HEAD `allowH2: false` (`http-dispatcher.ts:88`).
- 0.78.1 (2026-06-04): `httpIdleTimeoutMs` now also applies as the SDK request timeout for non-Codex providers; "disabled" sends max int32 instead of 0 because SDKs treat 0 as immediate timeout (#5294, `packages/coding-agent/CHANGELOG.md:1730`). Undici side: `bodyTimeout`/`headersTimeout` = setting (HEAD `packages/coding-agent/src/core/http-dispatcher.ts:91,95`).
- Lesson echo: model streams can be silent for minutes (reasoning, buffered tool calls) — audit every timeout in the HTTP stack.

**Lesson** — Defaults tuned for LANs break agent traffic; own the HTTP client config. In Node, an EventEmitter "error" event with no listener is fatal.

Related: [[http-transport-hardening]] · [[pi--http-transport-hardening|pi]] · [[proxied-request-hang-after-upgrade]] · [[stream-stall-without-header-timeout]]
