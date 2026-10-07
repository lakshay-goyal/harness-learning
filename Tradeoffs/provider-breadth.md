---
type: tradeoff
concepts: [unified-provider-api, custom-provider-registration, model-catalog, cross-provider-handoff, signed-reasoning-replay, thinking-level-abstraction, context-overflow-detection, usage-cost-accounting]
---
**Axis:** how many wire protocols and vendors the harness speaks natively. One option normalizes many vendor APIs behind a neutral layer. The other uses one vendor protocol as the internal transcript format and requires third parties to speak it.

| dimension | pi | codex |
|---|---|---|
| wire protocols | 10 chat APIs (`KnownApi`, `packages/ai/src/types.ts:17-27`), open `Api` string for plugins ([[pi--unified-provider-api]]) | 1: OpenAI Responses (`WireApi::Responses`, `codex-rs/model-provider-info/src/lib.rs:104-111`). Chat Completions was deleted (`d2394a2494` 2026-02-03; [[no-chat-completions-wire]]) |
| built-in providers | 42 (`packages/ai/src/providers/all.ts:136-181`) | 5: openai, amazon-bedrock, amazon-bedrock-runtime, ollama, lmstudio (`codex-rs/model-provider-info/src/lib.rs:653-690`). "We do not want to be in the business of adjucating which third-party providers are bundled" (`:667-670`) |
| internal message model | neutral `AssistantMessage` with a live partial | the vendor's `ResponseItem` stored as-is ([[codex--unified-provider-api]]) |
| extension point | `pi.registerProvider` / native `Provider` with a custom `streamSimple` ([[pi--custom-provider-registration]]) | declarative `config.toml [model_providers]` only; the endpoint must be Responses-compatible ([[codex--custom-provider-registration]]) |
| catalog | built at generation time from public aggregators; prices and compat flags ([[pi--model-catalog]]) | pushed by the vendor per account and client version; prompts, tool shapes and compaction limits, no prices ([[codex--model-catalog]]) |
| reasoning replay | six signature formats, same-model scoping, stale-signature handling ([[pi--signed-reasoning-replay]]) | one format (`encrypted_content`), always requested and replayed ([[codex--signed-reasoning-replay]]) |
| thinking control | neutral scale mapped to about 11 wire formats plus budgets ([[pi--thinking-level-abstraction]]) | vendor effort enum, with catalog-declared levels per model ([[codex--thinking-level-abstraction]]) |
| overflow detection | 25-regex catalogue, silent overflow, length stops ([[pi--context-overflow-detection]]) | one error code, `context_length_exceeded` ([[codex--context-overflow-detection]]) |
| handoff | cross-vendor `transformMessages` ([[pi--cross-provider-handoff]]) | capability projection within one family ([[codex--cross-provider-handoff]]) |
| cost | local price tables, tiers, cache TTL buckets ([[pi--usage-cost-accounting]]) | server-priced turn-cost endpoint ([[codex--usage-cost-accounting]]) |
| failure tail | [[stop-reason-mapping-gaps]], [[streamed-tool-call-fragmentation]], [[foreign-reasoning-signature-replayed]], [[thinking-off-not-honored]] and many more per adapter | Chat-era quirks ([[stop-reason-mapping-gaps]]) were removed by deletion. What remains is capability drift *within* the family: [[endpoint-rejects-request-field]], [[model-switch-replays-unsupported-content]] |

**When each wins**
- **Breadth (pi):**
  - users bring their own subscriptions or local models;
  - vendor lock-in is unacceptable;
  - the harness is a neutral product.

  The cost is a permanent per-adapter quirk tail and lowest-common-denominator features. Adapters must be maintained as each vendor ships.
- **Single protocol (codex):**
  - the harness is the vendor's own client;
  - features depend on server cooperation, such as WebSocket deltas, sticky routing, server compaction, configuration-update items and catalog-pushed prompts.

  Simplicity, and depth in vendor-specific features, beat reach. Third parties can still join through Responses-compatible endpoints (Ollama ≥ 0.13.4, LM Studio, Bedrock Mantle).
- Codex's history is the evidence. It *had* breadth (Chat Completions 2025-05 → 2026-02), paid the quirk tail ([[stop-reason-mapping-gaps]]: proxy `finish_reason:""`, the `[DONE]` sentinel, LiteLLM tool names), and chose deletion over maintenance.

Related: [[cache-strategy]] · [[prompt-ownership]] · [[unified-provider-api]] · [[custom-provider-registration]] · [[model-catalog]]
