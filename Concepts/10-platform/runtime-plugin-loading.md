---
type: concept
stage: architecture
tier: candidate
aliases: ["jiti", "virtualModules", "/reload", "ctx.reload()", "facets", "FacetHost.reload", "stale ctx", "withSession", "commit/discard", "ts-plugin-loading-jiti", "transactional-plugin-registration", "manual-hot-reload", "hot-extension-reload", "hot-reload-shape-preserving", "vm-loaded-plugin-generations", "facet-plugin-model", "stable-service-facade", "stale-context-invalidation"]
harnesses: [pi]
---
Loading uncompiled plugin code into a running harness: host-module aliasing so plugins share host singletons, transactional registration, explicit reload, and invalidation of handles that outlive their generation.

## Why
- Plugins written in TS must run without a build step, including inside single-file compiled binaries with no `node_modules` ([[plugins-fail-to-load-in-compiled-binary]]).
- A plugin importing its own copy of host packages gets duplicate classes/registries ([[duplicate-host-module-instances]]).
- A factory that throws half-way leaves live subscriptions/providers ([[plugin-registrations-leak-after-failure-or-reload]]).
- Handles captured before reload/session switch silently target the wrong session ([[stale-plugin-context-after-session-replacement]]).
- Node's ESM cache retains every module generation forever ([[module-cache-retains-plugin-generations]]).
- "Drain in-flight calls before swap" is often unimplementable ([[reload-cannot-drain-in-flight-calls]]).

## Design space
- **Loader**: precompiled JS only vs runtime TS transpile (pi: jiti; Babel lazily) vs bundler-per-plugin (pi experimental: esbuild → content-addressed CJS bundles).
- **Host deps**: plugin bundles its own vs host injects aliases/virtual modules (pi) vs peer deps enforced at install (pi, too).
- **Registration**: immediate side effects vs staged record committed on success (pi `commit()`/`discard()`).
- **Reload trigger**: file watching (rejected by pi, [[no-auto-hot-reload]]) vs explicit `/reload` (pi) vs per-plugin generation swap preserving service shape (pi Chord facets).
- **Old-generation handling**: drain admitted calls (pi Chord tried, reversed `5dd8c0132`) vs cut over, no rollback after cutover (pi Chord now) vs full teardown + `session_shutdown`/`session_start` (pi stable).
- **Stale handles**: silently keep working vs throw explicit error (pi) vs stable facade proxy that re-binds (pi Chord `use()` facades).
- **Memory**: ESM import (leaks generations) vs `node:vm` `compileFunction` CJS (pi Chord).
- **Placement**: one process vs facets per environment (worker / TUI / web) with dependency graph validation (pi Chord).

## Implementations
- [[pi--runtime-plugin-loading|pi]] — jiti + virtualModules, two-phase transactional factory load, `/reload` full rebuild, stale-ctx invalidation; experimental Chord facets with shape-preserving generation swap and `node:vm` bundles.

## Failures
- [[plugins-fail-to-load-in-compiled-binary]]
- [[duplicate-host-module-instances]]
- [[plugin-registrations-leak-after-failure-or-reload]]
- [[stale-plugin-context-after-session-replacement]]
- [[module-cache-retains-plugin-generations]]
- [[reload-cannot-drain-in-flight-calls]]
- [[settled-draft-then-probe-throws]]

## Related
[[extension-event-hooks]] · [[harness-package-distribution]] · [[replaceable-builtin-extension]] · [[client-server-session-split]] · [[project-trust-gate]] · [[sdk-embedding]] · [[no-auto-hot-reload]]
