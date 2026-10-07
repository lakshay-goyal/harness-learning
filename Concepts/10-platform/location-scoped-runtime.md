---
type: concept
stage: architecture
tier: candidate
aliases: [Location, "Location.Ref", LocationServiceMap, locationServices, InstanceState, "?directory=", x-opencode-directory, "Instance (opencode legacy)"]
harnesses: [opencode]
---
One process hosts many directories or workspaces. Each one lazily gets its own cached service graph: config, catalog, plugins, tools, permissions and runner.

## Why
- A server that serves several projects and worktrees must not leak one project's config, plugins, permissions or approvals into another.
- Building every project's graph up front is slow; tearing the whole process down to switch projects loses other sessions.
- Session execution must route by session alone: the session row records its location, so the right graph is selected without callers passing directories (`specs/v2/session.md:39-48`).

## Design space
- **Key**: working directory (opencode legacy) vs (directory, workspace) location ref (opencode v2; workspace ID "reserved for future placement semantics").
- **Granularity of caching**: per service keyed by directory (opencode legacy `InstanceState` = one `ScopedCache` per service) vs one compiled layer graph per location (opencode v2 `LocationServiceMap`, ~36 nodes).
- **Eviction**: explicit dispose of a directory (opencode legacy) vs idle TTL (opencode v2 `idleTimeToLive: "60 minutes"`).
- **What stays global**: session store, event store and run coordinator (opencode v2: "No layer takes a Session ID").
- **Request routing**: `?directory=` → `x-opencode-directory` header → `process.cwd()` (opencode HTTP server).
- **Alternative**: one process per project (pi: one session worker process per session, [[client-server-session-split]]).

## Implementations
- [[opencode--location-scoped-runtime|opencode]] — legacy per-directory `InstanceState` caches selected by request directory; v2 `buildLocationServiceMap` = `LayerMap` of compiled location graphs with 60-minute idle TTL.

## Failures
- [[stale-runner-recreates-context-after-move]]

## Tradeoffs
- [[client-server-vs-single-process]]

## Related
[[client-server-session-split]] · [[git-worktree-isolation]] · [[layered-settings]] · [[runtime-plugin-loading]] · [[event-sourced-session-store]] · [[opencode]]
