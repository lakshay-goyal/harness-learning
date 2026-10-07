---
type: failure
concepts: [runtime-plugin-loading, harness-package-distribution]
harnesses: [pi]
---
**Symptom** — Installed pi packages that listed `pi-tui`/`pi-ai`/`pi-coding-agent`/`typebox` as regular dependencies got their own copies; classes and registries were duplicated (instanceof failures, components/providers registered in the wrong registry), producing duplicate extension runtimes (#9863).

**Root cause** — npm installed the package's own host deps; module resolution then picked the nested copy instead of the host instance.

**Fix · [[pi]]** — `8d897edaa` 2026-09-23: managed installs disable peer installs (npm `--legacy-peer-deps`, bun/pnpm `--omit=peer`); host packages must be `peerDependencies: "*"`; pi warns when they appear in `dependencies` (`packages/coding-agent/src/core/package-manager.ts:1823-1852`; `resource-loader.ts:53-88`; `docs/packages.md:80-90`). Runtime aliasing to host copies via `virtualModules`/`alias` (`packages/coding-agent/src/core/extensions/loader.ts:570-575`).

**Lesson** — A plugin ecosystem needs one canonical instance of every host library; enforce it at install time and at resolve time.

Related: [[runtime-plugin-loading]] · [[harness-package-distribution]] · [[plugins-fail-to-load-in-compiled-binary]] · [[pi--harness-package-distribution|pi]]
