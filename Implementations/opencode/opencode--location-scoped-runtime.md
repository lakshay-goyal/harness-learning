---
type: implementation
harness: opencode
concept: location-scoped-runtime
commit: ecc4916b5a
files: [packages/opencode/src/effect/instance-state.ts:30-66, packages/opencode/src/project/instance-store.ts:15-25, packages/opencode/src/server/routes/instance/httpapi/middleware/workspace-routing.ts:87, packages/core/src/location-services.ts:42-79, packages/core/src/location-services.ts:83-110, packages/core/src/session/execution/local.ts:15-21, specs/v2/session.md:39-48, specs/project.md:3]
---
[[location-scoped-runtime]] in [[opencode]].

## Mechanism
### Legacy runtime ("instances")
- Each service that holds per-project state wraps it in `InstanceState`: a `ScopedCache` keyed by directory, built lazily on first `get` (`packages/opencode/src/effect/instance-state.ts:30-49`); `has`/`invalidate` per directory (`packages/opencode/src/effect/instance-state.ts:61-66`).
- `InstanceStore` disposes one directory or all (`packages/opencode/src/project/instance-store.ts:15-25`); a registered disposer invalidates every service cache for that directory (`packages/opencode/src/effect/instance-state.ts:38`).
- HTTP routing picks the directory per request: `?directory=` → `x-opencode-directory` → `process.cwd()` (`packages/opencode/src/server/routes/instance/httpapi/middleware/workspace-routing.ts:87`). One `opencode serve` process therefore serves many projects and worktrees.
- Goal since 2025-09: "let a single instance of OpenCode run sessions for multiple projects and different worktrees per project" (`specs/project.md:3`; `f993541e0b`).
- Per-directory state includes config, agents, permissions' in-memory approvals, plugins, MCP clients, tool registry.

### v2 runtime ("Locations")
- `Location {directory, workspaceID?, project}`. `locationServices` lists ~36 nodes: Location, Policy, Config, AgentV2, CommandV2, Reference, Integration, Catalog, AISDK, PluginV2, PluginInternal, FileSystem, Watcher, Pty, SkillV2, SystemContextRegistry, LocationMutation, PermissionV2, ToolOutputStore, ToolRegistry, BuiltInTools, Snapshot, SessionRunnerLLM… (`packages/core/src/location-services.ts:42-79`).
- `buildLocationServiceMap` = `LayerMap.make(ref => compile(hoisted graph), {idleTimeToLive: "60 minutes"})` (`packages/core/src/location-services.ts:83-110`).
- Session execution routes from the session ID alone: `SessionStore.get` → `LocationServiceMap.get(session.location)` → `SessionRunner.run` (`packages/core/src/session/execution/local.ts:15-21`; `specs/v2/session.md:39-48`). "No layer takes a Session ID."

## Constants
| name | value | path:line |
|---|---|---|
| v2 idle TTL | `"60 minutes"` | `packages/core/src/location-services.ts:109` |

## Evolution
- 2025-09-01 `f993541e0b` multiple instances inside a single opencode process (#2360).
- 2026-05-30 `9583e08be4` location-scoped config loading.
- 2026-06-01 `9b815bcbd2` location-based permission service.
- 2026-06-26 `ecdfff5a42` location node functionality separated and integrated into v2 (#34119).

## Quirks / drift
- Legacy approvals ("always") are per directory instance, so they are shared by every session in that project and vanish when the instance is disposed ([[permission-ruleset]]).

pi contrast: one session worker process per session instead of many locations per process ([[pi--client-server-session-split|pi]]).
