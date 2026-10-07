---
type: failure
concepts: [thinking-level-abstraction, model-catalog]
harnesses: [pi]
---
**Symptom** — Thinking was misconfigured on each new model release:
- Opus 4.7 adaptive thinking was misconfigured, and Sonnet 4.6 had no adaptive mode or xhigh mapping.
- The interleaved-thinking beta was sent to adaptive models, where it is redundant or deprecated.
- Anthropic-compatible aliases missed adaptive thinking.
- Google thinking level maps were ignored, and unsupported Gemini levels were sent.
- GLM 5.3 on Mistral got `prompt_mode`, and GLM 5.2 got no max effort.
- Gemini 2.5-flash-lite got "thinking budget 128 is invalid… between 512 and 24576".
- OpenAI models on Bedrock always ran at the default effort.

**Root cause** — Hardcoded per-model tables and id-substring checks in the adapters (`includes("2.5-flash")` matched flash-lite first). The tables rot with every model release and get duplicated across sibling adapters.

**Fix · [[pi]]**
- `9825c13f5` 2026-02-27 — skip the interleaved beta for adaptive models (#1665). HEAD gate: `!forceAdaptiveThinking` (`packages/ai/src/api/anthropic-messages.ts:1108-1115`).
- `e9d0074fa` 2026-02-26 — adaptive thinking for Sonnet 4.6; clamp xhigh (#1548).
- `b48d80293` 2026-04-09 — most-specific row first for 2.5-flash-lite (#2838/#2861) (`packages/ai/src/api/google-generative-ai.ts:441-466`). **Still live on Vertex**: the copy at `packages/ai/src/api/google-vertex.ts:524-554` lacks the flash-lite row (`:543` `includes("2.5-flash")`). Whether the Vertex catalog contains flash-lite is unverified, because the catalog JSON is gitignored (`.gitignore:11`).
- `d1c6cb1e0` 2026-04-16 — Opus 4.7 adaptive config (#3286).
- `d801d88a1` 2026-05-22 — adaptive thinking for Anthropic-compatible aliases.
- `6184307c3` 2026-06-23 — "require explicit anthropic compat metadata": id sniffing moved out of the adapter into the generator (`isAnthropicAdaptiveThinkingModel` `packages/ai/scripts/generate-models.ts:626-643`; `getAnthropicMessagesCompat` `:1220`).
- `af2c35223` 2026-08-17 — honor Google thinking level maps (#8135).
- `16235fd93` 2026-09-17 — removed hardcoded Gemini level tables in favour of the generated `thinkingLevelMap` (#9455) (`packages/ai/src/api/google-shared.ts:48-65`).
- `dc84c1ac0` 2026-09-28 — Mistral reasoning is derived from `thinkingLevelMap` instead of an id list (#9678) (`packages/ai/src/api/mistral-conversations.ts:186-215`).
- `2989eb581` 2026-10-06 — OpenAI on Bedrock (#10142): `gpt-oss` gets a flat `reasoning_effort` (low/medium/high); other `gpt-` models get nested `reasoning.effort`, with minimal→low because Bedrock rejects minimal (`packages/ai/src/api/bedrock-converse-stream.ts:1311-1348`).

**Quirk (live divergence)** — Anthropic `mapThinkingLevelToEffort` silently maps xhigh/max to "high" when they are not in `thinkingLevelMap` (`anthropic-messages.ts:908-926`). Bedrock checks `supportsNativeXhighEffort` first (`bedrock-converse-stream.ts:780-828`). Custom models therefore behave differently on the two paths.

**Lesson** — Per-model reasoning capability belongs in generated catalog data. Adapters should read maps, not sniff ids. Duplicated tables drift.

Related: [[thinking-level-abstraction]] · [[model-catalog]] · [[pi--thinking-level-abstraction|pi]] · [[thinking-off-not-honored]] · [[capability-sniffing-misses-opaque-ids]]
