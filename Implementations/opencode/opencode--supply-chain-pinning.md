---
type: implementation
harness: opencode
concept: supply-chain-pinning
commit: ecc4916b5a
files: [packages/core/src/npm.ts:86-100, packages/opencode/src/plugin/shared.ts:200-211, packages/opencode/src/installation/index.ts:145-164, packages/opencode/src/installation/index.ts:265-277, packages/opencode/src/skill/discovery.ts:76-118]
---
[[supply-chain-pinning]] in [[opencode]].

## Mechanism
### Legacy runtime
- **Plugin installs**: npm plugins from config install through `@npmcli/arborist` with `ignoreScripts: true`, `savePrefix: ""`, under a cross-process file lock `npm-install:<dir>` (`packages/core/src/npm.ts:86-100`; lock = `EffectFlock`, `a147ad68e6`). Plugins declare `engines.opencode`, checked before load (`packages/opencode/src/plugin/shared.ts:200`).
- **Self-update** ([[self-update]]): npm/pnpm/bun paths run `<pm> install -g opencode-ai@<version>` with no `--ignore-scripts` (`packages/opencode/src/installation/index.ts:271-277`); the curl path downloads `https://opencode.ai/install` and pipes it to bash/sh with `VERSION` set, no checksum (`packages/opencode/src/installation/index.ts:145-164`).
- **Remote skills** (`skills.urls`): `index.json` lists files per skill; downloads staged into a temp dir and swapped atomically when the version changes; tracked by a `.opencode-version` file; no hash verification (`packages/opencode/src/skill/discovery.ts:76-118`).
- **Project-supplied code** loads without a trust gate (`.opencode/plugin(s)`, `.opencode/tool(s)`; "Malicious config files" out of scope, `SECURITY.md:33`) ([[untrusted-repo-loads-executable-config]]).

### v2 runtime
- v2: `PluginInternal` and the dynamic provider plugin take `Npm.Service` from `packages/core/src/npm.ts` (`packages/core/src/plugin/internal.ts:73,95`; `packages/core/src/plugin/provider/dynamic.ts:9`), so v2 package installs share the same `ignoreScripts: true` path.

## Constants
| name | value | path:line |
|---|---|---|
| plugin install scripts | disabled (`ignoreScripts: true`) | `packages/core/src/npm.ts:100` |
| self-update scripts | enabled (npm default) | `packages/opencode/src/installation/index.ts:272` |

## Evolution
- 2026-04-15 `a147ad68e6` Effect-idiomatic file lock used by the installer.
- 2026-02-04 `556adad67b` custom tools and plugins wait for dependency install before loading.

## Quirks / drift
- Asymmetry is the reverse of pi's: opencode disables scripts for third-party plugins but not for its own self-update; pi disables them for self-update but not for third-party packages.

pi contrast: exact pins, lockfile-pinned installer, script allowlist, content-addressed remote binaries ([[pi--supply-chain-pinning|pi]]).
