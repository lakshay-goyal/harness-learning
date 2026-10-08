---
type: failure
concepts: [cache-breakpoint-placement, unified-provider-api]
harnesses: [opencode]
---
**Symptom** — Prompt caching was silently off for whole provider routes (cost spike, no cache-read tokens): Anthropic via OpenRouter, Anthropic-format models on OpenAI-compatible gateways, Claude on Bedrock, and markers that clobbered other provider options.

**Root cause** — Each AI SDK provider reads cache markers from its own `providerOptions` key and at its own level (message vs last content part), and Bedrock spells the marker differently. A marker under the wrong key or level is ignored without error.

**Fix · [[opencode]]**
- `0e3458b112` 2025-06-16 "fix cache-control" (bare `cacheControl` vs namespaced option).
- `969ad80ed2` 2025-07-05 "fix openrouter caching with anthropic, should be a lot cheaper".
- `827469c725` 2025-07-25 (#1305) content-level (last part) markers for non-Anthropic providers; message-level for Anthropic/Bedrock.
- `087d7da14d` 2026-01-25 (#10323) deep-merge `providerOptions` instead of overwriting.
- `ca5e85d6ea` 2026-02-01 (#11664) Bedrock `cachePoint.type` must be `"default"`, not `"ephemeral"`.
- HEAD writes the marker under every dialect at once — `anthropic`, `openrouter`, `bedrock`, `openaiCompatible`, `copilot`, `alibaba` (`packages/opencode/src/provider/transform.ts:362-381`). v2 lowers markers per native protocol instead (`packages/llm/src/protocols/anthropic-messages.ts:234-241`, `packages/llm/src/protocols/utils/bedrock-cache.ts:19`).

**Lesson** — Cache markers are per-SDK dialect and fail silently; assert cache-read tokens per provider route in tests rather than trusting that a marker was "sent".

Related: [[cache-breakpoint-placement]] · [[capability-sniffing-misses-opaque-ids]] · [[cache-breakpoints-miss-stable-segments]] · [[cache-miss-accounting]] · [[opencode--cache-breakpoint-placement|opencode impl]]
