---
type: failure
concepts: [model-catalog, unified-provider-api, constrained-tool-sampling, custom-provider-registration]
harnesses: [pi, codex]
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

**Fix · [[codex]]** — A custom tool type was unsupported by the serving provider.
- Symptom: Bedrock-served `openai.gpt-5.4-cmb` reused the bundled `gpt-5.4` metadata (`apply_patch_tool_type = freeform`), so Codex sent a Responses `custom` tool, which Bedrock's validator rejects: "even heavily disabled sessions can fail before the model runs" (`0db6811b7c` body).
- `0db6811b7c` 2026-04-24 (#19416): use the function `apply_patch` tool for Bedrock models. Superseded by `e783341b70` 2026-05-08, which deleted function-style `apply_patch` and kept freeform "including Bedrock catalog metadata".
- Related: `d19de6d150` 2026-04-24 Bedrock reasoning levels; `966932124c` 2026-05-30 Bedrock GPT models limited to the default service tier. Today the request filters `service_tier` to tiers Bedrock advertises (`codex-rs/core/src/client.rs:979-988`).
- The pi-style field rejections cannot occur for `max_output_tokens`/`temperature`, because Codex never sends those fields (`codex-rs/codex-api/src/common.rs:279-304`).

**Lesson** — Treat request fields, including tool wire types and service tiers, as capabilities of a given model on a given serving provider, declared in catalog metadata. Metadata reused across providers must be re-validated per provider. Respect provider minimums when clamping.

Related: [[model-catalog]] · [[unified-provider-api]] · [[thinking-level-abstraction]] · [[thinking-config-per-model-drift]] · [[output-token-cap-misbudgeted]] · [[pi--model-catalog|pi]] · [[codex--model-catalog|codex]] · [[constrained-tool-sampling]]
