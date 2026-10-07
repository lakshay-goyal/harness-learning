---
type: implementation
harness: opencode
concept: http-transport-hardening
commit: ecc4916b5a
files: [packages/opencode/src/provider/provider.ts:35-127, packages/opencode/src/provider/provider.ts:249, packages/opencode/src/plugin/openai/ws-pool.ts:26-38, packages/opencode/src/plugin/openai/ws-pool.ts:167, packages/llm/src/route/transport/websocket.ts:56-199]
---
[[http-transport-hardening]] in [[opencode]].

## Mechanism

### Legacy runtime — `timeoutFetch` around every SDK
- Every provider SDK gets a wrapped `fetch` combining caller abort + header timeout + SSE idle timeout + optional overall `timeout` via `AbortSignal.any`; Bun's own fetch timeout disabled (`timeout: false`) (`packages/opencode/src/provider/provider.ts:94-127`).
- **Header timeout**: aborts before the first response byte; `headerTimeout ?? 300_000`, `false` disables (`provider.ts:99,104-105`); OpenAI explicitly `OPENAI_HEADER_TIMEOUT_DEFAULT` (`provider.ts:35,249`).
- **SSE idle (chunk) timeout**: `wrapSSE` re-arms a timer per read; on expiry aborts with retryable `ResponseStreamError("SSE read timed out")` and cancels the reader (`provider.ts:37-58,98`).
- **OpenAI Responses WebSocket pool** (`packages/opencode/src/plugin/openai/ws-pool.ts`): connect 15 s, idle 5 min, max age 55 min; WS stream failures retried up to `streamRetries ?? 5`, then the session is pinned to HTTP (`ws-pool.ts:26-38,167`). Post-first-event failures surface as retryable instead of replaying partial output.

### v2 runtime
- No request or idle timeout in `packages/llm/src/route/transport/http.ts` or `route/client.ts` — deliberate: "Do not impose a universal provider-stream inactivity or absolute timeout" (`specs/v2/schema-changelog.md:557`; `specs/v2/session.md:153`).
- WebSocket transport closes with 1000 on cleanup and fails explicitly on early close (`packages/llm/src/route/transport/websocket.ts:56-99,170,195-199`).
- Request headers: `x-opencode-session-id`, `x-session-affinity`, `X-Session-Id` (+ parent ids) (`packages/core/src/session/runner/llm.ts:207-215`).

## Constants
| name | value | path:line |
|---|---|---|
| `chunkTimeout` default | 300 000 ms | `packages/opencode/src/provider/provider.ts:98` |
| `headerTimeout` default | 300 000 ms | `packages/opencode/src/provider/provider.ts:99` |
| `OPENAI_HEADER_TIMEOUT_DEFAULT` | 300 000 ms | `packages/opencode/src/provider/provider.ts:35` |
| WS connect / idle / max age | 15 s / 5 min / 55 min | `packages/opencode/src/plugin/openai/ws-pool.ts:26-28` |
| WS `streamRetries` | 5 | `packages/opencode/src/plugin/openai/ws-pool.ts:37` |

## Evolution — the idle-timeout reversals
- 2026-03-10 `69ddc91c35` per-chunk idle timeout added (2 min).
- 2026-03-14 `8c53b2b470` raised to 5 min (long reasoning pauses tripped it).
- 2026-03-19 `d69962b0f7` **disabled by default**.
- 2026-05-26 `f965db9e13` `headerTimeout` added, on only for OpenAI, **10 s**.
- 2026-05-29 `c7e1fc5e42` SSE stall made a retryable stream error.
- 2026-07-19 `67caf894e0` OpenAI header timeout raised (to 5 min).
- 2026-09-02 `4eb29a64f0` chunk timeout default 5 min again; `b04697366f` header timeout default 5 min for all providers; `69c172e8a7` unhandled `reader.cancel(err)` rejection.
- WS: 2026-05-27 `62da1e7682` transport; 2026-05-28 `14e0b9b17f` retry WS failures, `913659890d` `unref()`ed idle timer never fired; 2026-06-03 `7f8412ec3e` idle state preserved.

## Quirks / drift
- `specs/v2/schema-changelog.md:563` says "V1 had no universal processor inactivity watchdog" — true for the processor, but legacy has had a fetch-level 5-min idle timeout by default since 2026-09-02.
- 10 s was too aggressive for reasoning models; long-but-finite won.

Failures: [[stream-stall-without-header-timeout]] · [[server-limited-identifier-rejected]] · [[transport-fallback-after-partial-output]] · [[connection-cache-shared-across-accounts]].

Contrast: [[pi--http-transport-hardening|pi]] owns a dispatcher with idle timeouts and sticky WS→SSE fallback (same 15 s / 5 min / 55 min WS constants); opencode v2 deliberately has no timeouts at all.
