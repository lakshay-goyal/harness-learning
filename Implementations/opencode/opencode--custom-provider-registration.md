---
type: implementation
harness: opencode
concept: custom-provider-registration
commit: ecc4916b5a
files: [packages/opencode/src/provider/provider.ts:148-193, packages/opencode/src/provider/provider.ts:1785-1888, packages/opencode/src/plugin/index.ts:71-83]
---
[[custom-provider-registration]] in [[opencode]].

## Mechanism

### Legacy runtime
- Providers are config + npm package: any provider/model can name `api.npm`; bundled SDK factories for common packages (`BUNDLED_PROVIDERS`, `packages/opencode/src/provider/provider.ts:148-193`), otherwise `Npm.add(model.api.npm)` installs at runtime and the first `create*` export is called (`provider.ts:1853-1888`). `file://` npm paths allowed.
- SDK instance cached by hash of `{providerID, npm, options}`; baseURL `${VAR}` placeholders filled from custom vars then env; OpenAI-compatible forced `includeUsage: true` (`provider.ts:1785-1830`).
- Built-in auth/provider plugins: Codex (ChatGPT), Copilot, Modal, GitLab, Poe, Cloudflare Workers + AI Gateway, Azure, DigitalOcean, Snowflake Cortex, xAI (`packages/opencode/src/plugin/index.ts:71-83`). Plugins load first so their `config()` hook can edit `cfg.provider`; `auth.loader` can supply options and a custom `fetch`.

### v2 runtime
- `PluginV2.HookSpec` `provider.update` / `model.update` / `account.*` with immutable input + Immer draft; transforms are scoped and replayable, disabling a plugin closes its scope and rebuilds (`specs/v2/catalog-config-plugin-lifecycle.md:191-324`).

## Constants
none.

## Evolution
- 2026-01-29 `b937fe9450` SDK cache key lacked `providerID` → two providers sharing npm + options shared one client.

## Quirks / drift
- Runtime `npm install` of provider SDKs is a supply-chain surface (owned by 10-platform).

Failures: [[connection-cache-shared-across-accounts]].

Contrast: [[pi--custom-provider-registration|pi]] registers providers from extension code (`pi.registerProvider`); opencode registers them declaratively from config plus an npm package name.
