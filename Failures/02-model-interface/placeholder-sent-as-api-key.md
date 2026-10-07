---
type: failure
concepts: [credential-resolution, http-transport-hardening]
harnesses: [pi]
---
**Symptom** — Fake credentials reached providers as real keys:
- Vertex received a literal `<placeholder>` template value or the `gcp-vertex-credentials` / `<authenticated>` marker as its API key.
- A placeholder OpenAI key travelled through Cloudflare AI Gateway, which treats `Authorization` as a BYOK provider key that overrides stored keys.

**Root cause** — Sentinel and placeholder strings occupied the API-key slot. The `null` header-deletion markers that should suppress SDK default auth were dropped by an intermediate layer (registry `getApiKeyAndHeaders`).

**Fix · [[pi]]**
- `ff1ea1232` 2026-03-18 — ignore `/^<[^>]+>$/` placeholder Vertex keys (#2335) (`packages/ai/src/api/google-vertex.ts:99-103`).
- `3a13fa80c` 2026-04-15 — treat the `gcp-vertex-credentials` marker as ADC (#3221) (`:429-439`).
- `850c210b7` 2026-07-11 — filter ambient auth markers (`"<authenticated>"` sentinel, `packages/ai/src/env-api-keys.ts:160-192`) in compat dispatch.
- `a24fb9e96` 2026-08-03 — preserve `null` header-deletion markers through `getApiKeyAndHeaders` (#7030/#7539) (`packages/coding-agent/src/core/model-registry.ts:33-41,89-118`).
- HEAD Cloudflare AI Gateway: auth goes in `cf-aig-authorization: Bearer <key>`, and `Authorization` / `x-api-key` are set to **null** (`packages/ai/src/providers/cloudflare-auth.ts:87-101`). Header merge is case-insensitive; `null` suppresses a lower-level default (`packages/ai/src/utils/headers.ts:11-23`).

**Lesson** — Sentinel values in a key slot leak. Use a typed "ambient credential" result, and make header-deletion semantics survive every layer.

Related: [[credential-resolution]] · [[http-transport-hardening]] · [[pi--credential-resolution|pi]] · [[credential-scoped-config-dropped]]
