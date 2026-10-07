---
type: failure
concepts: [credential-resolution]
harnesses: [pi]
---
**Symptom** — Credential discovery from the environment picked the wrong key or found nothing:
- Generic `GH_TOKEN`/`GITHUB_TOKEN` were used as Copilot credentials.
- An unrelated `ANTHROPIC_AUTH_TOKEN` was sent as a bearer token to Anthropic-compatible providers (e.g. Xiaomi).
- Bun-compiled binaries in Linux sandboxes saw an empty `process.env`, so pi reported no keys.

**Root cause** — Env discovery was too broad (generic names shared across tools) and assumed `process.env` is always populated.

**Fix · [[pi]]**
- `5b8deef2f` 2026-04-27 — Bun sandbox fallback reads `/proc/self/environ`. It is duplicated in pi-ai on purpose for direct consumers (#3801). Lookup order is scoped `env` → `process.env` → `/proc` (`packages/ai/src/utils/provider-env.ts:5-52`).
- `a8af0b5e9` 2026-05-15 — Copilot env key is only `COPILOT_GITHUB_TOKEN` (#4485) (`packages/ai/src/env-api-keys.ts:74-76`).
- `b256ac7d7` 2026-05-17 — one-line `authToken: null` in the Anthropic client so the SDK stops auto-reading `ANTHROPIC_AUTH_TOKEN` from the environment (#4342; that this scopes the token to direct Anthropic is inferred from later code). HEAD: `getEnvApiKey` skips it because it must be sent as `Authorization: Bearer` (`env-api-keys.ts:78-81,153-158`; `packages/ai/src/providers/anthropic.ts:34-41`).
- Whitespace-only env values are treated as unset (`packages/ai/src/auth/context.ts:25-28`).

**Lesson** — Env discovery must be provider-specific and conservative. Generic token names belong to other tools.

Related: [[credential-resolution]] · [[pi--credential-resolution|pi]] · [[process-env-mutated-for-auth]]
