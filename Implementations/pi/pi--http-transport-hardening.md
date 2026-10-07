---
type: implementation
harness: pi
concept: http-transport-hardening
commit: b30a6dd77
files: [packages/coding-agent/src/core/http-dispatcher.ts:4, packages/ai/src/api/openai-codex-responses.ts:57, packages/ai/src/api/openai-codex-responses.ts:866, packages/ai/src/api/bedrock-converse-stream.ts:220, packages/ai/src/api/bedrock-converse-stream.ts:460, packages/ai/src/api/anthropic-messages.ts:1080, packages/coding-agent/src/core/provider-attribution.ts:40, packages/ai/src/api/cloudflare.ts:1, packages/ai/src/utils/headers.ts:11, packages/ai/src/utils/node-http-proxy.ts:117]
---
[[http-transport-hardening]] in [[pi]].

## Mechanism

### Global HTTP client (coding-agent `http-dispatcher.ts`)
- Global undici `EnvHttpProxyAgent` honouring `HTTP_PROXY/HTTPS_PROXY`; setting `httpProxy` sets them if unset (`packages/coding-agent/src/core/http-dispatcher.ts:45-50`; `910508595`).
- `allowH2: false`; `proxyTunnel: true` "Keep HTTP origins on CONNECT tunnels as before Undici 8.7" (`:86-91`; `23842b1e6` #8134 proxied plain-HTTP requests hung after a tool call).
- `bodyTimeout = headersTimeout = idle timeout`, default 300 s, choices 30s/1m/2m/5m/disabled (`:4, 8-14, 81-99`; `849f9d9c5` #4759) — covers long thinking pauses.
- `autoSelectFamilyAttemptTimeout: 2000` vs Node's 250 ms that kills valid connects on high-latency routes (`:5-6`; `14551e769` #7315/#7435).
- No-op `error` listener on undici Client/Pool so mid-stream termination errors don't crash the process (`:52-62`; `2117b61c6` #6133).
- `undici.install()` replaces global fetch so fetch + dispatcher come from one undici (Node 26 bundled fetch + npm dispatcher skipped decompression → `response.json()` failures), unless caller already replaced fetch (`:101-112`; `a93f06660`).
- SDK request options (`packages/coding-agent/src/core/sdk.ts:340-366`): `timeoutMs` = provider retry setting ?? HTTP idle timeout (0 → 2147483647); `websocketConnectTimeoutMs`; `maxRetries`, `maxRetryDelayMs` from `settings.retry.provider` (retry policy → [[auto-retry-backoff]]); `transformHeaders` = attribution headers + extension `before_provider_headers`.
- pi-ai proxy resolution: `resolveHttpProxyUrlForTarget` honors `<proto>_proxy`/`all_proxy` + `NO_PROXY`, provider env override; SOCKS/PAC rejected with explicit message (`packages/ai/src/utils/node-http-proxy.ts:117-161`).
- Default `User-Agent: pi (<platform> <release>; <arch>)` or `pi (browser)` (`packages/ai/src/utils/pi-user-agent.ts:17-19`), merged first so callers override (`87af49dec` #8361); `node:os` read synchronously via `process.getBuiltinModule` (`a3cc169d9` — async import raced first Codex request → "pi (browser)").
- Session resource registry `registerSessionResourceCleanup/cleanupSessionResources` for per-session cleanup (WS) (`packages/ai/src/session-resources.ts:1-24`).

### Header rules
- Merge **case-insensitive**; `null` value deletes a lower-level default (`packages/ai/src/models.ts:374-388`; `utils/headers.ts:11-23`; `packages/ai/src/types.ts:128,170`). Order (OpenAI family): pi UA → `model.headers` → Copilot dynamic → session-affinity → caller `options.headers` (`openai-completions.ts:766-790`, `openai-responses.ts:267-291`); Anthropic: base → session-affinity → model → dynamic → options (`anthropic-messages.ts:303-305, 1037-1051`). Deletion markers survive coding-agent layers (`a24fb9e96` #7030 — placeholder OpenAI key through Cloudflare AI Gateway). Extension `before_provider_headers` mutates in place, `null` deletes (`244f1deaf`).
- **Beta header management** (`getBetaFeatures`, `anthropic-messages.ts:1080-1122`): user/model `anthropic-beta` header **replaces** computed list; `null` suppresses all (`:1087-1103`; tests `anthropic-auth-token.test.ts:219,228`); empty header previously 400'd (`b0217ca6f`). Computed: OAuth betas; `fine-grained-tool-streaming-2025-05-14` only if eager streaming unsupported & tools present; `interleaved-thinking-2025-05-14` iff reasoning && thinking && !forceAdaptive (`9825c13f5`); `server-side-fallback-2026-07-01` iff fallbacks; managed effort → `mid-conversation-output-config-2026-07-01` + `thinking-binding-controls-2026-08-01`; native tool changes → `inline-tools-2026-09-15` (`:190-195, 1108-1120`). Sent as SDK `betas` param (`:1164`). Bedrock: `anthropic_beta` inside `additionalModelRequestFields` (thinking-binding, interleaved) (`bedrock-converse-stream.ts:1254-1306`).
- **Provider attribution headers** (`packages/coding-agent/src/core/provider-attribution.ts`), only when install telemetry enabled (`:40-42`; `PI_TELEMETRY`; [[install-telemetry]]): OpenRouter `HTTP-Referer: https://pi.dev`, `X-OpenRouter-Title: pi`, `X-OpenRouter-Categories: cli-agent` (`:44-50`); NVIDIA NIM `X-BILLING-INVOKE-ORIGIN: Pi` (`:52-56`); Cloudflare `User-Agent: pi-coding-agent` (`:58-62`); Vercel attribution removed (`83cbfc652`). OpenCode session headers regardless of telemetry: `x-opencode-session: <sessionId>`, `x-opencode-client: pi` (`:67-77`); pi-ai also injects `x-opencode-session` unless set (`packages/ai/src/providers/opencode-headers.ts:3-25`). Merge: session < attribution < caller (`:79-97`).
- Session-affinity headers (cache routing; detail in [[session-affinity-cache-routing]] / caching notes): openrouter `x-session-id`; openai `session_id` + `x-client-request-id` + `x-session-affinity`; `openai-nosession`; Anthropic-compat `sendSessionAffinityHeaders` for OpenRouter (`bbb61e34a`) / Fireworks (`99dc6fcec`); Mistral `x-affinity` unless caller set any-case (`mistral-conversations.ts:341-358`).

### Bedrock transport (`bedrock-converse-stream.ts`)
- Proxy: replaces default HTTP/2 handler with `NodeHttpHandler` + `HttpProxyAgent/HttpsProxyAgent` (HTTP/2 handler lacks agent support since SDK v3.798.0); `AWS_BEDROCK_FORCE_HTTP1=1` for HTTP/1.1-only endpoints (`:220-232`).
- **Signed-header injection**: caller headers added by a Smithy **build-step** middleware (after serialization, before SigV4 → signed); reserved `x-amz-*`, `authorization`, `host` silently skipped (`:460-494`; `7619aaefa`). Raw response headers captured by **deserialize-step** middleware for `onResponse` (`:507-529`; `10acee604` #8243); fallback `onResponse` with `x-amzn-requestid` (`:289-296`). Bearer via SDK-native token auth (`454b9619c` fixed duplicate Authorization from custom middleware).
- Endpoint pinning: non-standard baseUrl (VPC/proxy) always pinned; standard `bedrock-runtime(-fips)?.<region>.amazonaws.com(.cn)?` pinned only when no region and no ambient profile (`:168-180, 1217-1242`). Arc `5a889ef58` (pass baseUrl as endpoint) broke `us.*`/`eu.*` profiles → `a0a16c776` restored regional resolution. Browser: region fallback us-east-1 (`:233-238`). Abort via `client.send(command,{abortSignal})` (`:288`).
- Bedrock lazy wrapper imports via variable specifier so bundlers can't follow into Node-only AWS SDK; `setBedrockProviderModule()` for Bun binary static import (`bedrock-converse-stream.lazy.ts:4-24`; `bedrock-provider.ts:1-6`).

### Codex transports (`openai-codex-responses.ts`)
- `transport: "sse" | "websocket" | "websocket-cached" | "auto"`, default `"auto"` (`:294`; `types.ts:124`; settings `transport`, `sdk.ts:443`). WS attempted unless `sse` or session has sticky SSE fallback (`:296-301, 967-986`).
- WS headers: base + `OpenAI-Beta: responses_websockets=2026-02-06`, `x-client-request-id` & `session-id` = clamped session or fresh **UUIDv7** (models reject v4; `d2f8dafb0`) (`:282, 865, 1683-1699`); request = one JSON frame `{type:"response.create", ...body}` uncompressed (`:1542`). Bun WebSocket subclass injects proxy from env (Bun ignores proxy env for WS) (`:994-1025`; `8c2e3edde`).
- **Connection cache** `Map<sessionId, Map<accountId, entry>>` (account-scoped, `cfe6b6a05` #7284) (`:910`); idle TTL 5 min, **max age 55 min** below backend 60-min limit (`:866-867, 1052-1073`; `23d146261` #6268); busy cached socket → one-off uncached socket (`:1206-1215`); release keeps socket only if requested and OPEN (`:1193-1203, 1235-1246`); `registerSessionResourceCleanup(closeOpenAICodexWebSocketSessions)` (`:965`).
- **Continuation** (auto / websocket-cached, `:1518`): store `{lastRequestBody, lastResponseId, lastResponseItems}` after success (`:1565-1580`); next request with JSON-identical body minus `input`/`previous_response_id` AND input prefixed by `lastInput + lastResponseItems` → send `previous_response_id` + delta items only (`:1425-1477`); `store` stays false ("ChatGPT Codex Responses rejects store:true", `:1519-1520`); any error clears continuation (`:1581-1586`). Debug stats per session (requests, connections created/reused, delta vs full, failures, sseFallbacks) (`:893-947`).
- **Retry/fallback loop** (`:301-374`): `previous_response_not_found` → retry once with full context (`:344-348`; `c5dcb2600` #6931); `websocket_connection_limit_reached` before start → retry once (`:349-352`; `d0e0b84cb` #5973); aborted / `CodexApiError` / `CodexProtocolError` / `ProviderStreamEventCallbackError` → rethrow, no fallback (`:353-355, 714-720`); other transport failure → diagnostic `provider_transport_failure {configuredTransport, fallbackTransport, eventsEmitted, phase, requestBytes}` (`:356-365`), **sticky SSE for session** (`:366, 978-986`), fall back only if **no events emitted** else throw (`:367-371`; `370fdae6f` #4133). `start` emitted lazily on first WS event so fallback is invisible (`:1479-1491`).
- WS parsing: queue-based iterator, completion on `response.completed|done|incomplete`, close-before-completion → error, idle timeout = `timeoutMs` (`:1307-1423`); close 1009 → "message too big" (`:62, 1278-1280`); connect timeout 15 s (`:57, 1075-1151`).
- **SSE path**: body **zstd level 3** + `content-encoding: zstd` when `node:zlib.zstdCompressSync` available via `process.getBuiltinModule`, else plain JSON (browser) (`:58-60, 204-231, 376-383`; `0ac3cfe09`). Response-header timeout `AbortSignal.timeout(timeoutMs)` (`:396-413`; `7c02a5563` 10 s → `be7d5cf58` 20 s → `54113731b` configured HTTP timeout, #4945). Parser flushes decoder at EOF and treats residual buffer as a frame (`:799-859`; `64eeb82a4` #9047); abort cancels reader (`:805-808`; `a36a132c7`). Headers: `OpenAI-Beta: responses=experimental`, `accept: text/event-stream`, `session-id` (hyphenated, `26f1e00f7`) + `x-client-request-id`, clamped to 64 (`dcfe36c79` #6630/#6653) (`:1663-1681`). SSE retry `DEFAULT_MAX_RETRIES=0`, base 1000 ms (`:54-56, 388-461`).

### Templated endpoints (Cloudflare)
- Base URLs with `{CLOUDFLARE_ACCOUNT_ID}`/`{CLOUDFLARE_GATEWAY_ID}` placeholders: Workers AI `/ai/v1` (OpenAI-compatible), REST `/ai` (`run`), AI Gateway `/compat`, `/openai` passthrough ("until /compat supports /v1/responses"), `/anthropic` passthrough (`packages/ai/src/api/cloudflare.ts:1-19`). Substituted at dispatch from resolved provider env by `cloudflareStreams`/`cloudflareClassifier` (`providers/cloudflare-stream.ts:6-36`); unresolved placeholders stay literal (→ 404, `bdd5c53bc`). Gateway API map pinned to anthropic-messages + openai-completions + openai-responses (`providers/cloudflare-ai-gateway.ts:11-26`; `7ddbac282`).
- Worker binding transport: `createAiBindingFetch(env.AI)` passthrough fetch for `https://workers-binding.ai/ai-gateway/gateways/{gateway}/{provider}/…`; validates `binding.fetch` at construction (`api/cloudflare-ai-binding.ts:1-28, 80-90`); replaced a 192-line envelope shim (`55adba4f2`).
- Vertex: catalog `{location}` template baseUrl ignored; else `baseUrlResourceScope: COLLECTION`, `apiVersion:""` if path already versioned (`google-vertex.ts:388-422`; `65a6472bd`). Google: custom baseUrl → `httpOptions.baseUrl` + `apiVersion:""` (`google-generative-ai.ts:353-357`; `6ff405a97`, `aac68ba35`); custom `fetch` unsupported (`:87-89`). Azure host normalization `*.openai.azure.com|*.cognitiveservices.azure.com|*.ai.azure.com` path → `/openai/v1`, query stripped (`azure-openai-config.ts:37-66`).

### Other per-adapter transport limits
- Mistral: native `fetch`, SDK removed (`9dd90a497`); `AbortSignal.timeout(options.timeoutMs ?? 60_000)` over **whole** request incl. body (`mistral-conversations.ts:307-308, 326`) — only adapter with a hard default timeout.
- pi-messages: no retries, no timeout (`pi-messages.ts:401`).
- Identifier clamps: OpenAI `prompt_cache_key` 64 code points (`openai-prompt-cache.ts:1-8`; `7be75bade`).

## Constants
| name | value | path:line |
|---|---|---|
| `DEFAULT_HTTP_IDLE_TIMEOUT_MS` | 300000 (choices 30s/1m/2m/5m/off) | packages/coding-agent/src/core/http-dispatcher.ts:4,8-14 |
| `DEFAULT_AUTO_SELECT_FAMILY_ATTEMPT_TIMEOUT_MS` | 2000 | packages/coding-agent/src/core/http-dispatcher.ts:6 |
| `DEFAULT_WEBSOCKET_CONNECT_TIMEOUT_MS` | 15000 | packages/ai/src/api/openai-codex-responses.ts:57 |
| `SESSION_WEBSOCKET_CACHE_TTL_MS` | 5 min | packages/ai/src/api/openai-codex-responses.ts:866 |
| `SESSION_WEBSOCKET_MAX_AGE_MS` | 55 min | packages/ai/src/api/openai-codex-responses.ts:867 |
| `REQUEST_COMPRESSION_ZSTD_LEVEL` | 3 | packages/ai/src/api/openai-codex-responses.ts:60 |
| Codex WS beta | responses_websockets=2026-02-06 | packages/ai/src/api/openai-codex-responses.ts:865 |
| Codex `DEFAULT_MAX_RETRIES` / `BASE_DELAY_MS` | 0 / 1000 | packages/ai/src/api/openai-codex-responses.ts:54-55 |
| Mistral request timeout | 60000 ms | packages/ai/src/api/mistral-conversations.ts:307 |
| `OPENAI_PROMPT_CACHE_KEY_MAX_LENGTH` | 64 | packages/ai/src/api/openai-prompt-cache.ts:1 |
| `timeoutMs` disabled sentinel | 2147483647 | packages/coding-agent/src/core/sdk.ts:340-366 |

## Evolution
- 2026-02-13 `a26a9cfab` configurable transport + Codex WS session caching.
- 2026-04-18 `454b9619c` Bedrock SDK token auth; 2026-04-19 `5a889ef58` → 2026-04-21 `a0a16c776` Bedrock endpoint pin arc; 2026-04-21 `b0217ca6f` empty beta header fix.
- 2026-05-03 `370fdae6f` WS→SSE fallback (#4133); 2026-05-10 `8c2e3edde` Bun WS proxy; 2026-05-19 `7be75bade` cache-key clamp; 2026-05-20 `849f9d9c5` idle timeout (#4759); 2026-05-27 `7c02a5563` header timeout, `26f1e00f7` hyphenated header; 2026-05-29 `7619aaefa` Bedrock custom headers, `a36a132c7` abort SSE reads.
- 2026-06-12 `be7d5cf58` relax header timeout; 2026-06-16 `910508595` httpProxy setting, `a93f06660` fetch overrides; 2026-06-23 `d0e0b84cb` reconnect on connection limit; 2026-06-28 `54113731b` configured timeout; 2026-06-30 `2117b61c6` undici error listener, `0ac3cfe09` zstd, `a3cc169d9` UA race.
- 2026-07-03 `23d146261` rotate stale WS (#6268), `83cbfc652` drop Vercel attribution; 2026-07-14 `dcfe36c79` session-id clamp; 2026-07-19 `d2f8dafb0` UUIDv7; 2026-07-22 `c5dcb2600` previous_response_not_found; 2026-07-31 `cfe6b6a05` account-scoped sockets.
- 2026-08-02 `14551e769` 2 s connect attempt; 2026-08-03 `a24fb9e96` header deletion markers; 2026-08-17 `10acee604` Bedrock smithy headers; 2026-08-19 `87af49dec` pi UA everywhere.
- 2026-09-02 `23842b1e6` proxy tunnel; 2026-09-03 `64eeb82a4` SSE EOF flush, `55adba4f2` binding fetch; 2026-09-10 `bbb61e34a` OpenRouter affinity.

## Evidence commits
a26a9cfab, 454b9619c, 5a889ef58, a0a16c776, b0217ca6f, 370fdae6f, 8c2e3edde, 7be75bade, 849f9d9c5, 7c02a5563, 26f1e00f7, 7619aaefa, a36a132c7, be7d5cf58, 910508595, a93f06660, d0e0b84cb, 54113731b, 2117b61c6, 0ac3cfe09, a3cc169d9, 23d146261, 83cbfc652, dcfe36c79, d2f8dafb0, c5dcb2600, cfe6b6a05, 14551e769, a24fb9e96, 10acee604, 87af49dec, 23842b1e6, 64eeb82a4, 55adba4f2, bbb61e34a, 99dc6fcec, 9dd90a497, 65a6472bd

## Quirks
- `connectWebSocket` deletes `wsHeaders["OpenAI-Beta"]` from a lowercased record (`openai-codex-responses.ts:1087-1088` vs `utils/headers.ts:3-9`) → likely no-op (unverified).
- `previous_response_not_found` retry `continue`s even if events emitted — duplication possible? (unverified; error probably arrives before output).
- Mistral 60 s whole-request default cuts long generations as `error` (not `aborted`) unless coding-agent always passes `timeoutMs` (it passes idle timeout via `sdk.ts:340-366`; whether all paths do — unverified).
- Azure Responses sends `prompt_cache_key` even when retention none (`azure-openai-responses.ts:202`) — inconsistent (unverified intent).
- Bedrock bypasses pi's abortable `retryProviderRequest`; AWS SDK default retry applies (unverified).

## Failures
- [[proxied-request-hang-after-upgrade]]
- [[transport-defaults-kill-connections]]
- [[persistent-connection-lifetime-exceeded]]
- [[connection-cache-shared-across-accounts]]
- [[transport-fallback-after-partial-output]]
- [[stream-stall-without-header-timeout]]
- [[server-limited-identifier-rejected]]
- [[sse-framing-errors]]
- [[async-init-race-caches-wrong-value]]
- [[placeholder-sent-as-api-key]]
- [[bedrock-credential-and-endpoint-precedence]]
- [[empty-payload-rejections]] (empty beta header)
