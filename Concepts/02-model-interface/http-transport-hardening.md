---
type: concept
stage: model-interface
tier: candidate
aliases: [configureHttpDispatcher, SESSION_WEBSOCKET_MAX_AGE_MS, zstd, http-dispatcher.ts, connection-pool-max-age, transport-fallback-sticky, request-body-compression, header-deletion-marker, signed-header-injection, templated-endpoint-placeholders, beta-header-management, provider-attribution-headers, getBetaFeatures, anthropic-beta, transport, stream_idle_timeout_ms, websocket_connect_timeout_ms, force_http_fallback, codex-http-client]
harnesses: [pi, codex]
---
The harness owns its HTTP and WebSocket transport policy instead of inheriting runtime and SDK defaults. That policy covers:
- proxies and tunnels, idle and header timeouts, connect attempt timeouts;
- error listeners;
- pooled-connection lifetime and scoping;
- fast-transport to robust-transport fallback;
- body compression;
- header merge, deletion and signing rules, beta flags, attribution headers;
- server-limited identifiers.

## Why
- Runtime defaults are tuned for LAN traffic and short requests:
  - Node's 250 ms happy-eyeballs timeout kills high-latency connects.
  - A missing `error` listener crashes the process.
  - A dependency upgrade changes proxy semantics or fetch/dispatcher pairing.

  See [[transport-defaults-kill-connections]] and [[proxied-request-hang-after-upgrade]].
- Agent requests are long and stateful:
  - Pooled sockets outlive the server's hard limit.
  - Caches leak across accounts.
  - Falling back mid-stream would duplicate output.
  - Streams stall with no headers.

  See [[persistent-connection-lifetime-exceeded]], [[connection-cache-shared-across-accounts]], [[transport-fallback-after-partial-output]] and [[stream-stall-without-header-timeout]].
- Servers impose identifier formats: a 64-character cap, UUIDv7, header spelling ([[server-limited-identifier-rejected]]).
- Header layers must preserve deletion semantics, and signing must cover injected headers ([[placeholder-sent-as-api-key]], [[bedrock-credential-and-endpoint-precedence]]).
- Async-initialized values such as the UA race the first request ([[async-init-race-caches-wrong-value]]).

## Design space
- **Client**
  - SDK defaults.
  - A single global dispatcher owned by the harness: proxy env, `proxyTunnel:true`, `allowH2:false`, configurable idle timeout, 2 s family attempt, no-op error listener, `undici.install()`. *pi chose this.*
- **Pooled WebSockets**
  - Per request.
  - Cached per session and account, with a 5 min idle TTL and a 55 min max age below the server's 60 min limit. *pi chose this.*
  - One-shot stateless retry on lost continuation or connection limit.
- **Fallback**
  - None.
  - WebSocket to SSE only before the first event, sticky per session, with diagnostics. *pi chose this:* 370fdae6f.
- **Timeouts**
  - A fixed constant. *pi tried 10 s, then 20 s.*
  - Tie the header timeout to the user-configured timeout. *pi chose this:* 54113731b.
- **Headers**
  - Case-insensitive merge with precedence auth < model < dynamic < caller.
  - `null` deletes a header.
  - Beta flags computed per model generation, with user override or suppression.
  - Custom headers signed with SigV4, reserved headers skipped.
  - Attribution headers only when telemetry is enabled.
- **Body**
  - zstd compression where the runtime supports it.
- **Server lifetime handling**
  - Proactive rotation below the limit (55 min). ✔ pi
  - Reactive: map `websocket_connection_limit_reached` (60 min) to a retryable error and reopen. ✔ codex
- **Timeouts**
  - A separate 15 s connect timeout, plus one 300 s idle timeout applied to every receive *and* send. ✔ codex (`codex-rs/model-provider-info/src/lib.rs:66,71`)
- **Fallback trigger**
  - WebSocket → HTTPS as the last retry after stream-retry exhaustion or `426`, sticky per session, still honoring server Retry-After. ✔ codex
- **Offline tolerance**
  - Unbounded network-wait retries (5 s doubling to 60 s) that don't consume the retry budget. ✔ codex (feature `UnboundedConnectionRetries`)

## Implementations
- [[pi--http-transport-hardening|pi]] — coding-agent `http-dispatcher.ts` and `provider-attribution.ts`. Codex WS/SSE transport (connection cache, sticky fallback, zstd level 3), Anthropic `getBetaFeatures`, Bedrock Smithy build/deserialize middleware, Cloudflare endpoint placeholders.
- [[codex--http-transport-hardening|codex]] — own `codex-http-client` and `codex-websocket-client`; 300 s idle and 15 s connect timeouts; reactive 60-min socket limit; sticky WebSocket→HTTPS fallback; zstd for ChatGPT auth; unbounded network wait.

## Failures
- [[proxied-request-hang-after-upgrade]]
- [[transport-defaults-kill-connections]]
- [[persistent-connection-lifetime-exceeded]]
- [[connection-cache-shared-across-accounts]]
- [[transport-fallback-after-partial-output]]
- [[stream-stall-without-header-timeout]]
- [[server-limited-identifier-rejected]]
- [[async-init-race-caches-wrong-value]]
- [[placeholder-sent-as-api-key]]
- [[bedrock-credential-and-endpoint-precedence]]
- [[sse-framing-errors]]
- [[server-side-state-missing-on-continuation]]
- [[prewarm-blocks-turn-start]]
- [[server-retry-advice-ignored]]

## Related
[[unified-provider-api]] · [[credential-resolution]] · [[provider-identity-shim]] · [[install-telemetry]] · [[auto-retry-backoff]] · [[abort-propagation]] · [[extension-event-hooks]]
