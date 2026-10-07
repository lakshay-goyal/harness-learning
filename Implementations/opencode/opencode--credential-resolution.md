---
type: implementation
harness: opencode
concept: credential-resolution
commit: ecc4916b5a
files: [packages/opencode/src/provider/provider.ts:1491-1497, packages/opencode/src/provider/provider.ts:1633-1705, packages/opencode/src/auth/index.ts:10, packages/core/src/session/runner/model.ts:83-88]
---
[[credential-resolution]] in [[opencode]].

## Mechanism

### Legacy runtime — layered passes, config last
1. Env vars listed by models.dev for the provider (`source: "env"`; `key` kept only when exactly one env var) (`packages/opencode/src/provider/provider.ts:1633-1645`).
2. Stored auth `<data>/auth.json` (mode 0600, `packages/opencode/src/auth/index.ts:10,79-88`) → `source: "api"` (`provider.ts:1646-1656`).
3. Plugin `auth.loader` (may supply options and a custom `fetch`), then built-in custom loaders.
4. Config re-applied last → **config wins** (`provider.ts:1698-1705`).
- `enabled_providers` allow-list / `disabled_providers` deny-list applied throughout (`provider.ts:1496-1497`).

### v2 runtime
- Runner credentials: key → value, OAuth access token, else model body `apiKey` (`packages/core/src/session/runner/model.ts:83-88`).
- `Auth.config(name)` reads a redacted Effect `Config` env var, so a missing key is a typed Authentication error (`packages/llm/src/route/auth.ts:85-117`).

## Constants
| name | value | path:line |
|---|---|---|
| auth file | `<data>/auth.json`, mode 0600 | `packages/opencode/src/auth/index.ts:10` |

## Evolution
- 2026-01-12 `bf37a88f7f` `auth.set` not awaited → first request used the old key.
- 2026-01-29 `b937fe9450` SDK cache key now includes `providerID`.

Failures: [[credential-expires-mid-run]] · [[connection-cache-shared-across-accounts]].

Contrast: [[pi--credential-resolution|pi]] lets a stored credential own its provider (request override → stored → ambient); opencode applies config last so config beats stored auth.
