---
type: failure
concepts: [http-transport-hardening]
harnesses: [pi, opencode]
---
**Symptom**
- Codex rejected `session-id` headers longer than 64 chars (#6630).
- OpenAI rejected `prompt_cache_key` longer than 64.
- Codex rejected UUIDv4 request ids for some models.
- The underscore `session_id` header was ignored on Codex.

**Root cause** — Session ids are user-customizable and were forwarded unclamped. Request-id format and header spelling differed from the reference client's.

**Fix · [[pi]]**
- `7be75bade` 2026-05-19 — clamp `prompt_cache_key` to 64 code points (`packages/ai/src/api/openai-prompt-cache.ts:1-8`).
- `26f1e00f7` 2026-05-27 — hyphenated `session-id` header for Codex (`packages/ai/src/api/openai-codex-responses.ts:1663-1681`).
- `dcfe36c79` 2026-07-14 — clamp Codex session-id to 64 (#6653).
- `d2f8dafb0` 2026-07-19 — shared UUIDv7 for sessionless WebSocket request ids (#6834) (`:282,865,1683-1699`).

**Fix · [[opencode]]** `747b8daafc` 2026-06-06 (#31062) "bound prompt cache session keys": v2 external session ids (`ses_` + 64 hex) exceeded OpenAI's 64-char `prompt_cache_key`; the runner strips the `ses_` prefix for that shape (`packages/core/src/session/runner/llm.ts:204`, sent at `:216`). Legacy keys on the session id via `promptCacheKey` (`packages/opencode/src/provider/transform.ts:1323-1334`, unverified for length).

**Lesson** — Clamp every server-limited identifier at the edge, and match the reference client's id formats and header spelling.

Related: [[http-transport-hardening]] · [[pi--http-transport-hardening|pi]] · [[tool-call-id-normalization]]

**Fix · [[pi]]** (cache-affinity angle, appended by 06-caching writer) — `6af10c9c7` 2026-04-23 (#3579) "make OpenAI Responses session_id header optional" (some routes/proxies reject the header and `prompt_cache_retention`); OpenCode Responses models omit the unsupported `session-id` header "while preserving other cache-affinity data" (#6645, `packages/coding-agent/CHANGELOG.md:1251`); related issues cited in fix mining #7676, #5702, #6941 (content unverified). Affinity header matrix in [[pi--session-affinity-cache-routing]].
