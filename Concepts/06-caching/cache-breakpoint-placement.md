---
type: concept
stage: caching
tier: candidate
aliases: [cache_control, cachePoint, applyAnthropicCacheControl, cacheControlFormat, supportsCacheControlOnTools, AWS_BEDROCK_FORCE_CACHE]
harnesses: [pi]
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

## Implementations
- [[pi--cache-breakpoint-placement|pi]] — per-adapter placement in pi-ai; marker on last *initial* tool + deferred placeholder under native tool changes.

## Failures
- [[cache-breakpoints-miss-stable-segments]]
- [[capability-sniffing-misses-opaque-ids]]
- [[late-tool-change-rewrites-cache]]

## Related
[[cache-retention-control]] · [[cache-stable-prompt-prefix]] · [[transcript-carried-system-prompt]] · [[model-catalog]] · [[unified-provider-api]]
