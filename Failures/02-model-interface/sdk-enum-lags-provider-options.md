---
type: failure
concepts: [thinking-level-abstraction, unified-provider-api]
harnesses: [opencode]
---
**Symptom** — Typed provider SDKs validated options against enums and allow-lists frozen at release time, so values the provider already accepted were rejected client-side or silently stripped: new reasoning efforts (Groq, xAI, Mistral, Bedrock `none`) and explicit OpenAI `service_tier` for newer models.

**Fix · [[opencode]]** vendored SDK patches (`patches/*`, root `package.json` `patchedDependencies`):
- `6fea419feb` 2026-08-12 Groq `reasoningEffort` enum → `z.string()`; same widening in the xAI and Mistral patches.
- `1542195217` 2026-09-01 Bedrock SDK accepts `reasoningEffort: none`.
- `ea2d59d7ca` 2026-09-06 `@ai-sdk/openai` model allow-list for flex/priority `service_tier` removed.

**Lesson** — A typed SDK in front of fast-moving providers is itself a failure source; pass unknown option values through and let the provider validate.

Related: [[thinking-level-abstraction]] · [[unified-provider-api]] · [[thinking-config-per-model-drift]] · [[capability-sniffing-misses-opaque-ids]] · [[opencode--unified-provider-api|opencode]]
