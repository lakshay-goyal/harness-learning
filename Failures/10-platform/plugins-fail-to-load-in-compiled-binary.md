---
type: failure
concepts: [runtime-plugin-loading]
harnesses: [pi]
---
**Symptom** — User TypeScript extensions failed to load in the single-file Bun binary (#681) and later in Node SEA hosts (#8237): imports of host packages couldn't resolve and Babel wasn't available.

**Root cause** — Compiled binaries have no `node_modules`; jiti's normal resolution and lazy Babel loading assume a filesystem install.

**Fix · [[pi]]**
- `1919fd7c9` / `843f23525` 2026-01-13 — fork `@mariozechner/jiti` with `virtualModules` mapping host packages to the binary's own instances.
- `50993d743` 2026-05-07 — back to upstream jiti 2.7 once `virtualModules` landed upstream (#4244).
- `c06132898` 2026-08-18 — Node SEA hosts: `jiti/static` so bundlers embed Babel (`packages/coding-agent/src/core/extensions/jiti-static-loader.ts:1-3`; `packages/coding-agent/src/core/extensions/loader.ts:41-54`); virtual modules registered under both `@earendil-works/*` and legacy `@mariozechner/*` (`packages/coding-agent/src/core/extensions/virtual-modules.ts:18-38`).

**Lesson** — If plugins are uncompiled source, the plugin loader must resolve host packages from the host itself, not from disk — test the compiled distribution, not just the dev install.

Related: [[runtime-plugin-loading]] · [[duplicate-host-module-instances]] · [[pi--runtime-plugin-loading|pi]]
