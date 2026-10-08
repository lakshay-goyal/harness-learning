---
type: implementation
harness: opencode
concept: runtime-plugin-loading
commit: ecc4916b5a
files: [packages/opencode/src/config/plugin.ts:18-25, packages/opencode/src/plugin/index.ts:160-200, packages/opencode/src/plugin/loader.ts:139, packages/opencode/src/plugin/shared.ts:200-216, packages/core/src/npm.ts:86-100, packages/opencode/src/effect/instance-state.ts:30-49, specs/v2/instructions.md:13]
---
[[runtime-plugin-loading]] in [[opencode]].

## Mechanism
### Legacy runtime
- **Discovery**: config `plugin: [spec | [spec, options]]` (npm package, `file://` or path) plus every config dir's `{plugin,plugins}/*.{ts,js}` (`packages/opencode/src/config/plugin.ts:18-25`), including project `.opencode/` — no trust gate ([[untrusted-repo-loads-executable-config]]).
- **Install**: npm specs installed with arborist, `ignoreScripts: true`, under a file lock (`packages/core/src/npm.ts:86-100`); `engines.opencode` range checked (`packages/opencode/src/plugin/shared.ts:200-211`).
- **Load**: Bun runtime `import(entry)` of TS/JS (`packages/opencode/src/plugin/loader.ts:139`); no transpiler aliasing layer, plugins import `@opencode-ai/plugin` types.
- **Order**: internal plugins (Codex/Copilot/GitLab auth, provider plugins) unless `OPENCODE_DISABLE_DEFAULT_PLUGINS`; then external ones unless `OPENCODE_PURE` (`packages/opencode/src/plugin/index.ts:170-200`). Install/load errors are published as session errors, not fatal.
- **Scope / reload**: plugin state lives in a per-directory `InstanceState` (`ScopedCache` keyed by directory, `packages/opencode/src/effect/instance-state.ts:30-49`); reload = disposing the instance and rebuilding it. No file watching.

### v2 runtime
- Goal: "Services are hot-reloadable by design: updates are granular, observable, and do not require tearing down the whole process" (`specs/v2/instructions.md:13`); plugin registrations are Scope-bound overlays that disappear when the Scope closes ([[plugin-tools]]); location graphs evicted after 60 min idle ([[location-scoped-runtime]]).
- Config transforms vs catalog transforms: "a transform can mutate any part of config, a transform change cannot safely trigger only `Catalog.reload()`" → catalog transforms chosen (`specs/v2/catalog-config-plugin-lifecycle.md:3`, `specs/v2/catalog-config-plugin-lifecycle.md:40`).

## Constants
| name | value | path:line |
|---|---|---|
| plugin file glob | `{plugin,plugins}/*.{ts,js}` | `packages/opencode/src/config/plugin.ts:21` |

## Evolution
- 2025-08-11 `1c83ef75a2` lazy dynamic import inside the plugin client hung the compiled binary ([[plugins-fail-to-load-in-compiled-binary]]).
- 2025-11-02 `894cbaa51e` duplicate subscriptions; 2026-01-04 `c3fd3c8656` duplicate init.
- 2026-02-04 `556adad67b` wait for dependency install before loading.
- 2026-03-24 `814a515a8a` two-phase init, async error handling.
- 2026-04-02 `81d3ac3bf0` `Tool.define()` wrapper accumulation on re-init ([[tool-wrapper-accumulation]]).

## Quirks / drift
- A plugin exported under two names was initialized twice until `c3fd3c8656`: identity is by function, not by name.

pi contrast: jiti with virtual host modules, transactional registration, explicit `/reload` with stale-context invalidation ([[pi--runtime-plugin-loading|pi]]).
