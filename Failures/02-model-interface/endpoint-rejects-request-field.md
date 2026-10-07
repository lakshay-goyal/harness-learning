---
type: failure
concepts: [model-catalog, unified-provider-api]
harnesses: [pi, opencode]
---
**Symptom** 400s on otherwise valid requests:
- Anthropic: `temperature` sent alongside extended thinking. Opus 4.7+ rejects any non-default temperature. The deprecated `interleaved-thinking` beta was sent to adaptive models.
- Bedrock GovCloud: `display` and `block_binding` rejected.
- OpenAI Responses: `max_output_tokens < 16` rejected (#6265), and some Codex-protocol gateways reject `max_output_tokens` entirely.
- Azure Foundry: 400 on prompt-cache params.
- Sign in with ChatGPT tokens: rejected `prompt_cache_retention`, `prompt_cache_options`, `max_output_tokens` and `temperature`.

**Root cause** The request builder assumed one wire schema per API. In practice the accepted fields vary by model generation, region and credential type.

**Fix · [[pi]]**
- `9825c13f5` 2026-02-27: drop temperature when thinking, and skip the interleaved beta for adaptive models.
- `40832d195` 2026-05-31: suppress temperature for Opus 4.7+ via the `supportsTemperature` compat flag (`packages/ai/src/api/anthropic-messages.ts:1194-1202`; generator `packages/ai/scripts/generate-models.ts:645-657`).
- `454b9619c` 2026-04-18: GovCloud omits `display` and `block_binding` (`packages/ai/src/api/bedrock-converse-stream.ts:1244-1252,1263-1270`).
- `2e4ad6a09` 2026-07-06: `max_output_tokens = max(maxTokens, 16)` (`packages/ai/src/api/openai-responses.ts:32-33,340-342`; Azure `azure-openai-responses.ts:18-19`) (#6265).
- `b8b873b98` 2026-09-01: gate on the `supportsMaxOutputTokens` compat flag.
- `a37306d43` 2026-10-05: the Azure Foundry catalog omits prompt-cache params.
- The ChatGPT-token path detects provider `openai` with a non-`sk-` key and strips those fields (`openai-responses.ts:40-47,328-346`).

**Fix · [[opencode]]** `20088a87b0` 2026-01-13 / `5e650fd9e2` 2026-04-16 Cloudflare AI Gateway rejects `max_tokens` for OpenAI reasoning models; `4a56491e42` 2026-01-30 / `3a4c253969` 2026-08-21 `textVerbosity` and `c2403d0f15` 2026-04-13 `reasoningSummary` guarded for `@ai-sdk/openai-compatible`; `0c32afbc35` 2026-01-30 snake_case `thinking.budget_tokens` for OpenAI-compatible; `81b7b58a5e` 2026-04-17 Copilot Haiku rejected `eager_input_streaming`, then `025a6392ce` / `c361c2953f` tool streaming off for Vertex-Anthropic and non-Claude models (`packages/opencode/src/provider/transform.ts:1227-1232`).

**Lesson** Treat request fields as per-model capabilities declared in catalog metadata. Respect provider minimums when clamping.

Related: [[model-catalog]] · [[unified-provider-api]] · [[thinking-level-abstraction]] · [[thinking-config-per-model-drift]] · [[output-token-cap-misbudgeted]] · [[pi--model-catalog|pi]] · [[opencode--thinking-level-abstraction|opencode]]
