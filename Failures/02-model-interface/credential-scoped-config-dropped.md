---
type: failure
concepts: [credential-resolution]
harnesses: [pi]
---
**Symptom**
- Extension calls dropped credential-resolved endpoints (Copilot Business), so requests went to the default host.
- Cloudflare requests hit a literal `{CLOUDFLARE_ACCOUNT_ID}` URL and returned 404, because a key-only stored credential short-circuited the ambient env lookup for the account id.

**Root cause**
- Only the secret (apiKey/headers) was forwarded, not the `baseUrl` that the auth result carries.
- Credential and ambient config were merged all-or-nothing instead of per field.

**Fix · [[pi]]**
- `bdd5c53bc` 2026-07-11 — per-field merge of credential → ambient env for `CLOUDFLARE_API_KEY/ACCOUNT_ID/GATEWAY_ID` (#6021/#6292) (`packages/ai/src/providers/cloudflare-auth.ts:10-29`). Unresolved placeholders stay literal (`packages/ai/src/providers/cloudflare-stream.ts:6-36`).
- `e741cb05c` 2026-08-04 — preserve extension auth endpoints. The credential-supplied `baseUrl` overrides the model baseUrl (`packages/coding-agent/src/core/model-runtime.ts:679`; pi-ai `packages/ai/src/models.ts:869`; Copilot `toAuth` derives it from token `proxy-ep`, `packages/ai/src/auth/oauth/github-copilot.ts:64-87`).

**Lesson** — An auth result can carry endpoint and config, not just a secret. Merge credential and ambient config per field.

Related: [[credential-resolution]] · [[pi--credential-resolution|pi]] · [[placeholder-sent-as-api-key]] · [[http-transport-hardening]]
