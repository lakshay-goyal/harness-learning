---
type: failure
concepts: [credential-resolution, http-transport-hardening]
harnesses: [pi]
---
**Symptom**
- The Vertex ADC check returned false at startup and cached that result forever. pi fell back to AI Studio, which has stricter 429s.
- The first Codex request reported `User-Agent: pi (browser)` (a 50/50 repro).

**Root cause** — Node builtins (`node:fs`, `node:os`) are loaded by async dynamic import for browser safety. A first call that raced the import saw "not available" and memoized it.

**Fix · [[pi]]**
- `cf656c169` 2026-02-25 — don't cache unless definitively in a browser (#1550) (`packages/ai/src/env-api-keys.ts:46-58`). Guard comment: "NEVER convert to top-level imports - breaks browser/Vite builds" (`:1-24`).
- `a3cc169d9` 2026-06-30 — synchronous `process.getBuiltinModule` for the UA (`packages/ai/src/utils/pi-user-agent.ts:17-19`).
- Related open quirk: `lazyOAuth` memoizes the load promise (`promise ??=`), so a rejected load is cached for the process lifetime (`packages/ai/src/auth/helpers.ts:46-50`) (unverified, untested).

**Lesson** — Don't memoize "unknown" as "no". Avoid async module init for values needed on the first request.

Related: [[credential-resolution]] · [[http-transport-hardening]] · [[unified-provider-api]] · [[node-only-imports-break-browser-bundle]]
