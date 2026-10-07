---
type: group
group: 02-model-interface
---
Failures whose first concept is in [[Model Interface]].

## Handoff, reasoning replay, ids
- [[thinking-tag-mimicry]] — After a model switch, the model starts emitting literal `<thinking>`/`</thinking>` tags, copied from tagged replayed reasoning.
- [[gemini-unsigned-tool-call-replay]] — Gemini 3 rejects history whose function calls came from other providers without a thoughtSignature. Four fix strategies were tried.
- [[tool-call-id-requirement-drift]] — Tool-call ids stripped for Gemini 3 lose the pairing between calls and responses.
- [[signed-empty-reasoning-dropped]] — Signed reasoning blocks whose text is empty are dropped on replay. Results: Anthropic errors, Gemini Flash stops mid-task, Responses replay breaks.
- [[empty-signature-semantics-vary]] — "Compatible" endpoints send and accept empty signatures, but pi downgraded their reasoning to text.
- [[aborted-reasoning-signature-invalid]] — A turn aborted mid-thinking, when resubmitted, gets a 400 "Invalid signature in thinking block".
- [[foreign-reasoning-signature-replayed]] — Thought signatures from another provider or model, or in an invalid format, are replayed and rejected.
- [[model-relabel-breaks-same-model-check]] — A relay reports a different response model, so signed thinking is treated as coming from another model and downgraded.
- [[stale-thinking-signature-after-prefix-change]] — After the system prompt or tools change, there are persistent 400s for signed thinking.
- [[opaque-reasoning-payload-lost]] — Redacted or encrypted reasoning payloads are dropped, mis-merged or never captured, so continuity breaks.
- [[responses-reasoning-item-pairing]] — OpenAI Responses returns 400s because the replayed fc_/rs_/ctc_ item ids violate server-side pairing.
- [[reasoning-not-replayed-degrades-tool-args]] — Dropping prior reasoning makes Qwen and similar models emit `{}` args, or makes DeepSeek/Kimi/MiMo return 400s.
- [[orphaned-tool-calls-and-results]] — After an interruption, history contains calls with no results, or results with no calls, and providers reject it.
- [[failed-turns-replayed]] — Empty or errored assistant turns are replayed and break tool-call/result chains, causing "reasoning without following item".
- [[missing-optional-fields-crash-replay]] — `null` content or undefined tool arguments crash replay or are rejected.
- [[cross-provider-tool-call-id-normalization]] — Tool-call ids that are too long, contain pipes or trailing `_`, or use invalid characters are rejected after a provider switch.
- [[tool-call-id-collision]] — Truncated or missing ids collide, giving "duplicate tool_call_id".
- [[placeholder-text-misleads-model]] — Placeholder text for empty tool results ("see attached image") makes the model hallucinate images. Images dropped silently for non-vision models leave the model unaware they existed.
- [[assistant-content-shape-misread]] — Assistant content sent as an array: Copilot Claude re-answers the history, and DeepSeek mirrors the nesting.

## Streaming, payload, stop reasons
- [[streamed-tool-call-fragmentation]] — One tool call is split into several, or calls are merged, or calls that were never finalized are executed.
- [[stream-delta-assembly-errors]] — Out-of-order items, empty deltas, duplicate reasoning fields or start-event content all corrupt the assembled blocks.
- [[sse-framing-errors]] — The terminal frame without a trailing blank line is lost. Proxy junk events crash the parser.
- [[malformed-tool-json-crashes]] — An SDK's strict JSON parsing of tool deltas kills the stream.
- [[stream-scratch-state-persisted]] — `partialJson` scratch buffers are persisted into tool calls and corrupt resumed sessions.
- [[stop-reason-mapping-gaps]] — Unknown, refusal, safety or `end` stop reasons crash, look like normal stops, or a truncation is masked as toolUse.
- [[empty-payload-rejections]] — Empty tools arrays, text parts, content, beta headers or instructions get 400s.
- [[endpoint-rejects-request-field]] — Temperature, betas, display, too-small max_output_tokens or cache params are rejected by specific models or endpoints.

## Usage and cost
- [[usage-double-counting]] — Reasoning tokens or cached tokens are counted twice, or cache reads are under-reported.
- [[streamed-usage-misread]] — Streamed usage patches reset counts to 0, crash, or zero out aborted streams.
- [[usage-priced-at-wrong-rate]] — 1h cache writes, long-context tiers or fast/priority service tiers are priced at the base rate.
- [[billed-call-lost-on-parse-error]] — A classifier call that was billed loses its usage when the answer fails to parse.
- [[fallback-model-output-misattributed]] — Server fallback responses are priced as the requested model, or two models' output could be spliced together.

## Overflow and output budget
- [[rate-limit-misread-as-overflow]] — 429s or Bedrock throttling ("Too many tokens") trigger compaction.
- [[overflow-message-not-recognized]] — A new backend's phrasing of context overflow is not detected.
- [[silent-overflow-undetected]] — The provider accepts or truncates oversized input with no error.
- [[output-token-cap-misbudgeted]] — Static, missing or oversized max_tokens causes truncation or "max_tokens + input > context" 400s.
- [[thinking-consumes-answer-budget]] — Reasoning uses the shared max_tokens and no answer is produced.

## Errors
- [[provider-error-body-hidden]] — A gateway error collapses to "403 (no body)" / "Unknown: UnknownError", or is replaced by garbage.
- [[error-text-breaks-retry-classification]] — Transient 5xx/429 errors are not auto-retried because the error text lacks the keyword the classifier looks for.
- [[foreign-sdk-error-shape-skips-retry]] — The Google SDK's ApiError shape bypasses the shared retry, so pre-token 429/5xx are terminal.
- [[unpaired-surrogate-breaks-json]] — A lone UTF-16 surrogate in tool output breaks request serialization.

## Stream runtime and packaging
- [[quadratic-event-queue-drain]] — Draining a large event queue takes quadratic CPU.
- [[private-fields-break-duck-typed-streams]] — Adding `#private` fields breaks hand-rolled EventStream mocks and implementations.
- [[node-only-imports-break-browser-bundle]] — Node-only imports in the provider graph break browser and Bun bundles.

## Thinking, catalog, resolution
- [[thinking-off-not-honored]] — "Off" still thinks, disables tools, or gets a 400 on models that can't disable thinking.
- [[thinking-config-per-model-drift]] — Hardcoded tables of thinking capability per model id misconfigure new models.
- [[capability-sniffing-misses-opaque-ids]] — Application inference-profile ARNs and new model ids miss caching or adaptive thinking.
- [[strict-tool-schema-rejections]] — Strict tool schemas or rejected keywords make every request 400.
- [[catalog-layer-precedence-errors]] — A stale overlay masks a newer bundle; disabled models still appear; upstream drift deletes models; custom models are dropped.
- [[availability-snapshot-races]] — Startup picks the wrong provider, refresh gets stuck, or login hangs on an availability snapshot.
- [[catalog-hot-path-quadratic]] — Model lookups and prompt submission slow down as the catalog or the session grows.
- [[provider-reregistration-replaces-config]] — Re-registering a provider with only overrides loses its models; overrides or baseUrl are ignored for plugin providers.
- [[model-reference-ambiguity]] — `--model` picks an unauthenticated provider, a slashed gateway id, or mis-splits a colon id.
- [[unusable-default-model-selected]] — The saved default model has no credentials and blocks a usable local model.

## Auth and credentials
- [[tool-name-mapping-not-invertible]] — Subscription tool-name mapping does not round-trip (find→Glob), so the model calls unknown tools.
- [[credential-expires-mid-run]] — An OAuth token expires during long tool phases.
- [[oauth-refresh-token-rotation-lost]] — Concurrent or cancelled refreshes lose the rotated refresh token, so users are silently logged out.
- [[credential-refresh-on-availability-path]] — A refresh failure bricks startup; `/model` is slow because of refreshes.
- [[credential-file-lock-contention]] — Lock contention reads as "No API key"; a compromised lock crashes; credentials go stale; reads are serialized.
- [[oauth-loopback-callback-failures]] — Redirect_uri mismatch, a reserved port, a busy fixed port, or headless login fails.
- [[device-code-polling-hang]] — Device login hangs after authorization under clock drift.
- [[login-side-effects-rate-limited]] — Copilot login trips rate limits with bursts of policy POSTs.
- [[env-credential-discovery-misfires]] — Generic or unrelated env tokens hijack auth; a sandboxed Bun sees an empty env.
- [[process-env-mutated-for-auth]] — Deleting an env API key for one request removes it for the whole process.
- [[async-init-race-caches-wrong-value]] — An async import race caches "no ADC" or a "browser" UA.
- [[placeholder-sent-as-api-key]] — Sentinel or placeholder strings, or SDK placeholder keys, are sent as real credentials.
- [[config-value-indirection-ambiguity]] — Literal keys are treated as env names on Windows; command keys are cached forever.
- [[credential-scoped-config-dropped]] — An endpoint or account config carried by the credential is lost between layers.
- [[bedrock-credential-and-endpoint-precedence]] — Profile ignored, inference profiles broken, or duplicate Authorization header on Bedrock.

## Transport
- [[proxied-request-hang-after-upgrade]] — After a dependency upgrade, proxied HTTP requests hang or responses fail to decode.
- [[transport-defaults-kill-connections]] — Connects fail on high-latency links; an unhandled socket error crashes the process.
- [[persistent-connection-lifetime-exceeded]] — Long Codex WebSocket sessions hit server age or connection limits, or lose continuation state.
- [[connection-cache-shared-across-accounts]] — A cached WebSocket connection is reused across different accounts.
- [[transport-fallback-after-partial-output]] — A WebSocket failure after partial output, or repeated failures, has no safe fallback.
- [[stream-stall-without-header-timeout]] — An SSE request shows "Working…" forever with zero events.
- [[server-limited-identifier-rejected]] — Session ids or cache keys over 64 characters, or UUIDv4 request ids, are rejected.

## Elsewhere (other groups, related)
- [[truncated-stream-accepted-as-success]] (01-loop) — Streams with no terminal event are treated as successful partial answers.
- [[tool-result-image-routing]] (05-context) — Images in tool results are dropped or flaky per provider wire shape.
- [[deterministic-5xx-retried]] (01-loop) — A deterministic edge 504 on huge classifier inputs is retried pointlessly.
- [[tool-arg-coercion-breaks-unions]] (03-tools) — Lenient argument coercion corrupts nullable unions or fails under CSP.
- 01-loop retry failures: [[retry-classifier-regex-sprawl]] · [[retry-backoff-hygiene]] · [[hidden-sdk-retries-double-retry]] · [[length-stop-recovery]] · [[length-truncated-tool-calls-executed]]
- [[api-key-overrides-subscription-auth]] — Config API key silently overrides stored subscription OAuth, so users get billed pay-as-you-go.

Back: [[Model Interface]]
