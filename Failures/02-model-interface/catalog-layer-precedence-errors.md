---
type: failure
concepts: [model-catalog]
harnesses: [pi]
---
**Symptom**
- After an upgrade, a stale cached remote overlay masked the newer bundled catalog.
- Models an org owner disabled on Radius were still listed.
- models.dev dropped `workers-ai/*` from cloudflare-ai-gateway, which deleted every openai-completions model there and narrowed the inferred API union.
- Custom models for built-in providers were silently dropped, and `list-models` swallowed errors.

**Root cause** — Layered catalogs (bundled → remote overlay → models.json → extension → overrides) without recency or authority rules:
- the overlay always won;
- the fetched Radius catalog was merged instead of replacing the baseline;
- the API union was inferred from a volatile upstream;
- custom model definitions did not inherit required api/baseUrl/apiKey.

**Fix · [[pi]]**
- `ec4d413fc` 2026-04-14 — custom models inherit defaults from the builtin: same id, else same api, else first openai-completions, else first (`packages/coding-agent/src/core/provider-composer.ts:251-259`).
- `c8c3cd499` 2026-07-20 — validate generated model data before builds (`scripts/check-model-data.ts`).
- `54fad505b` 2026-07-21 — remote models are used only if `entry.lastModified > localGeneratedAt` (`packages/coding-agent/src/core/remote-catalog-provider.ts:52-58`).
- `7ddbac282` 2026-08-25 — derive gateway models from the Workers AI catalog and pin the API map to anthropic-messages + openai-completions + openai-responses (`packages/ai/src/providers/cloudflare-ai-gateway.ts:11-26`).
- `43d376399` 2026-10-06 — a fetched or cached Radius catalog **replaces** the bundled baseline (`packages/ai/src/providers/radius.ts:26-29`).

**Lesson** — Overlays need a recency check against the base they override. Authoritative remote catalogs replace rather than merge. Don't infer supported transports from a volatile upstream.

Related: [[model-catalog]] · [[pi--model-catalog|pi]] · [[availability-snapshot-races]]
