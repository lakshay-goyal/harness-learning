---
type: concept
stage: caching
tier: must-have
aliases: [cache_control, cachePoint, applyAnthropicCacheControl, cacheControlFormat, supportsCacheControlOnTools, AWS_BEDROCK_FORCE_CACHE, applyCaching, CachePolicy, applyCachePolicy, "cache: auto", ANTHROPIC_BREAKPOINT_CAP, BEDROCK_BREAKPOINT_CAP, copilot_cache_control]
harnesses: [pi, opencode]
---
Where explicit prompt-cache markers go in a request (system, last tool, last conversation message) for providers that require them, and on which models they may be sent at all.

## Why
- Unmarked stable segments are re-billed: tool schemas missed cache whenever the transcript changed until they got their own breakpoint; string-shaped user messages never got a marker ([[cache-breakpoints-miss-stable-segments]]).
- Markers sent to models that don't support them → API errors; models hidden behind aliases/ARNs never get caching ([[capability-sniffing-misses-opaque-ids]]).
- Providers cap markers (Anthropic ~4), so placement is a budget.

## Design space
- Implicit provider caching only (OpenAI/Mistral/Codex: keyed by `prompt_cache_key`; Google: pi sends nothing).
- **System + last tool + last message** (Anthropic in **pi**); **system + last user message** (Bedrock in **pi**); **first system + last tool + last user/assistant/tool message** (OpenAI-compat with Anthropic format, e.g. OpenRouter `anthropic/*`, in **pi**).
- Normalize content shape before marking (string → text block).
- Capability detection by id/name substring + user override env (pi Bedrock) vs explicit catalog metadata.
- codex: absent. No `cache_control` or breakpoint field anywhere in `codex-rs` (`git grep cache_control` at `622e9e3696` → no hits). Caching relies on OpenAI's automatic prefix caching keyed by `prompt_cache_key` ([[codex--session-affinity-cache-routing|codex]]).
- **First two system + last two non-system messages, marker written under every SDK dialect key at once** (opencode legacy `applyCaching`); message-level for Anthropic/Bedrock, last-content-part otherwise.
- **Policy object** `auto | none | {tools, system, messages: latest-user-message | latest-assistant | {tail:n}, ttlSeconds}` applied before lowering, manual per-part hints preserved (opencode v2 `CachePolicy`, default auto = last tool + last system + latest user message).
- **Cap enforcement**: lowering counts markers, allocates tools → system → messages, silently drops the excess with a warning (opencode v2, cap 4 for Anthropic and Bedrock).
- Defer to provider-side automatic caching when the user opts in (opencode legacy skips its markers if `options.cacheControl` is set; Vercel gateway `caching: "auto"`).

## Implementations
- [[pi--cache-breakpoint-placement|pi]] — per-adapter placement in pi-ai; marker on last *initial* tool + deferred placeholder under native tool changes.
- [[opencode--cache-breakpoint-placement|opencode]] — legacy fixed positions (first 2 system, last 2 messages) under all SDK keys; v2 `cache-policy.ts` auto placement (last tool, last system, latest user) with a 4-breakpoint cap in Anthropic/Bedrock lowering.

## Failures
- [[cache-breakpoints-miss-stable-segments]]
- [[capability-sniffing-misses-opaque-ids]]
- [[late-tool-change-rewrites-cache]]
- [[cache-marker-namespace-mismatch]]
- [[nondeterministic-tool-order-busts-cache]]

## Related
[[cache-retention-control]] · [[cache-stable-prompt-prefix]] · [[transcript-carried-system-prompt]] · [[model-catalog]] · [[unified-provider-api]] · [[prompt-cache-strategy]]

## Tradeoffs
- [[prompt-cache-strategy]]
