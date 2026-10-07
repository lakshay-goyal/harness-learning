---
type: failure
concepts: [usage-cost-accounting]
harnesses: [pi]
---
**Symptom** Costs were mispriced in several ways:
- 1h cache writes were priced at the 5-minute rate: Anthropic (#5738), then Bedrock (#9457) and Vercel AI Gateway streaming (#9210).
- Long-context requests were billed at base rates. Bedrock under-costed requests over 272k tokens (#10326).
- OpenAI Fast mode was billed at 1× (#10034).
- Codex priority/flex requests were mispriced when the response said `default`.

**Root cause**
- Cache TTL tiers, input-size tiers and service tiers are separate price buckets.
- Adapters did not report the TTL split: it lives in Bedrock `cacheDetails` and in Vercel deltas, while the SDK types it only on `message_start`.
- The generator dropped models.dev tiers.
- The pricing tables are duplicated, so they drift.

**Fix · [[pi]]**
- `0be5bb6c9` 2026-06-15: `Usage.cacheWrite1h`, priced at 2× base input (`packages/ai/src/models.ts:1211-1217`) (#5738).
- `8a7b0c03d` 2026-09-15: Bedrock reads Σ`cacheDetails[ttl==ONE_HOUR]` (`packages/ai/src/api/bedrock-converse-stream.ts:705-722`) (#9457).
- `667fc3dd3` 2026-09-23: read `cacheWrite1h` from deltas (`anthropic-messages.ts:841-847`) (#9210).
- `a9ecf301f` 2026-07-09: request-wide input tiers, where the highest `inputTokensAbove` threshold applies to the WHOLE request (`models.ts:1201-1209`).
- `4665fafb4` 2026-10-02: generator keeps Bedrock tiers (#10326).
- `a6ca86102` 2026-09-25: `fast` multiplier (×2) in `openai-responses.ts:391-404` (#10034). The Codex copy still lacks it (`packages/ai/src/api/openai-codex-responses.ts:602-614`).
- `2cdac7382` 2026-04-17: Codex trusts the requested tier when the response says `default` (`openai-codex-responses.ts:631-639`).

**Lesson** Every price bucket (TTL, size tier, service tier) must be reported and priced separately. Duplicated pricing tables drift.

Related: [[usage-cost-accounting]] · [[usage-double-counting]] · [[fallback-model-output-misattributed]] · [[pi--usage-cost-accounting|pi]]
