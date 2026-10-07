---
type: failure
concepts: [auto-retry-backoff, terminal-event-required, errors-as-stream-events]
harnesses: [pi, opencode]
---
**Symptom** — Transient failures ended headless runs ("waiting for a manual nudge") because the error text did not match the retryable list.

**Root cause** — Retryability is a regex over provider `errorMessage`; every provider/transport/runtime words transient failures differently, so the list grows forever.

**Fix · [[pi]]** (each adds a pattern; HEAD list `packages/ai/src/utils/retry.ts:30-107`)
- 0.25.0 "connection error"; `bb445d24f` 2025-12-10 initial overloaded/rate limit/5xx (#157).
- `fb6d464ed` 2026-01-14 "fetch failed"; `9b84857b8` 2026-01-22 "terminated"; 0.47.0 `upstream connect`/`connection refused`/`reset before headers` (#733, Codex).
- `2501a053e` 2026-03-14 `server_error|internal_error` (#2117); `8e3bb4ff5` 2026-03-17 "provider returned error" (#2264).
- `f10cce943` 2026-04-08 "ended without sending any chunks" (#2892); `c15e4d491` 2026-04-17 "Network connection lost" (#3317); `2394f3a98` 2026-04-23 Bedrock http2 no-response (#3594).
- `5ac874c84` 2026-05-12 Anthropic missing `message_stop` (#4433).
- `371adcf37` 2026-06-24 list moved from `agent-session.ts` to pi-ai `utils/retry.ts` + explicit "please retry" texts (#6019).
- `4285712ba` 2026-07-09 Bun "socket connection was closed" (#6431); `b0c2a90e5` 2026-07-17 Responses early EOF (#6727); `33e40c3e1` 2026-07-22 DNS `ENOTFOUND/EAI_AGAIN` (#6946); `d53b56760` CF 524 (#6239); `e5d18382a` CF 520 (#9627); `57d96d72e` gRPC ResourceExhausted (#6449); `fe10558eb` upstream buffer limit; `e98f287ee` Azure peak load (#9669).
- `3874b3e98` 2026-10-02 "model is at capacity" (#10278); `5b6c792b4` 2026-10-05 HTTP/2 "pending stream has been canceled" (#10379); `8b5708dbb` 2026-10-06 `server_busy` (#10543); `7fb59f995` 2026-10-07 Mistral `finish_reason:"error"` (#10487).
- ~52 changelog bullets mention retry classification.

**Fix · [[opencode]]** a steady stream of "retry X too" commits: `12ee9d51c3` 2025-12-23 socket closed unexpectedly; `7123aad5a8` 2026-03-25 Bun `ZlibError`; `81bdffc81c` 2026-04-07 Alibaba "rate increased too quickly"; `25ecf0af6b` 2026-05-05 `server_is_overloaded`; `61aefc0759` 2026-08-05 regex list over message **and** response body; `71d08e94d5` 2026-08-20 xAI "at capacity"; `40282c1d4d` 2026-08-21 `network[-_\s]error` variants. HEAD: structured codes first (`packages/opencode/src/provider/error.ts:102-139`), then 7 regexes (`packages/opencode/src/session/retry.ts:33-41`). v2 classifies by typed reason, but its `content[-_ ]?policy|content_filter|safety` body regex runs before the status check (`packages/llm/src/route/executor.ts:233-235`), so a 5xx mentioning "safety" is non-retryable (inference).

**Lesson** — Centralize retry classification in the provider layer, default-to-retry on transport-level errors, and expect the list to grow; prefer structured error kinds over text.

Related: [[auto-retry-backoff]] · [[terminal-event-required]] · [[error-text-breaks-retry-classification]] · [[foreign-sdk-error-shape-skips-retry]] · [[truncated-stream-accepted-as-success]] · [[pi--auto-retry-backoff|pi]] · [[opencode--auto-retry-backoff|opencode]]
