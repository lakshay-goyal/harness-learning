---
type: implementation
harness: pi
concept: runtime-plugin-loading
commit: b30a6dd77
files: [packages/coding-agent/src/core/extensions/loader.ts:41-54, packages/coding-agent/src/core/extensions/loader.ts:131-150, packages/coding-agent/src/core/extensions/loader.ts:157-160, packages/coding-agent/src/core/extensions/loader.ts:192-200, packages/coding-agent/src/core/extensions/loader.ts:244-268, packages/coding-agent/src/core/extensions/loader.ts:567-578, packages/coding-agent/src/core/extensions/loader.ts:736-876, packages/coding-agent/src/core/extensions/virtual-modules.ts:14-38, packages/coding-agent/src/core/agent-session.ts:3659-3697, packages/coding-agent/src/core/resource-loader.ts:501-521, packages/chord/src/facets/host.ts:423-511, packages/chord/src/node/bundle-loader.ts:192]
---
[[runtime-plugin-loading]] in [[pi]].

## Mechanism
### Stable coding-agent (jiti)
- Loader = jiti (`packages/coding-agent/src/core/extensions/loader.ts:2`). Node: `jiti` with Babel loaded only if native load fails (`extensions/jiti-loader.ts:1-3`); compiled binaries (Bun `--compile`, Node SEA, bundled Node) use `jiti/static` so bundlers embed Babel (`extensions/jiti-static-loader.ts:1-3`, `loader.ts:41-54`). Loader deps deferred until needed (`40c256ccc`).
- Host-module identity: embedded runtimes → `virtualModules` holding the host's own copies of `typebox` (+ `@sinclair/typebox` alias), `pi-ai` (compat entry, + `oauth`, `providers/all`), `pi-agent-core`, `pi-tui`, `pi-coding-agent` under both `@earendil-works/*` and legacy `@mariozechner/*` scopes, `tryNative:false`; TS-source runtime → virtualModules + tsconfig paths; unbundled Node → `alias` map (`virtual-modules.ts:14-38`, `loader.ts:567-578`). Goal: plugin imports resolve to the SAME instances as the host (no duplicate classes/registries).
- `moduleCache:false` per jiti instance (`loader.ts:577`) + explicit factory cache keyed by cwd+generation, cleared on `/reload` (`loader.ts:131-150`, `resource-loader.ts:509-513`): same-dir session switches reuse imported modules but run fresh factories (`5505316ea`, `packages/coding-agent/CHANGELOG.md:1474`).
- **Two-phase transactional load**: factory runs against throwing action stubs ("Action methods cannot be called during extension loading", `loader.ts:157-160`); registrations go to the `Extension` record; runtime changes queued while `state==="loading"`; `commit()` on success, `discard()` on throw rolls back subscriptions/providers/flags (`loader.ts:244-268,532-548,613-632`, `a69bef789`). Runner `bindCore()` replaces stubs with real actions (shared `ExtensionRuntime`, `cb3ac0ba9`).
- **Discovery order** (`loader.ts:828-876`): (1) project `<cwd>/.pi/extensions/` (2) global `~/.pi/agent/extensions/` (3) configured paths (settings `extensions`, `-e`). Per dir: `*.ts|*.js`, or subdir with `package.json` `pi.extensions` manifest, else `index.ts|index.js`; one level deep (`loader.ts:736-822`); dedup by resolved path. Built-ins load after file/package extensions ([[pi--replaceable-builtin-extension]]).
- **Trust bootstrap**: first pass loads only global + CLI extensions with project settings forced untrusted; `project_trust` handlers decide; then full load (`resource-loader.ts:501-521`; `89a92207f` 2026-06-05) → [[project-trust-gate]].
- **Conflicts**: same tool/flag from two extensions → load error (`resource-loader.ts:1246-1281`); reserved keys can't be overridden ([[plugin-shortcut-shadows-core-keys]]).
- **Reload = manual teardown/rebuild, no file watching**: `/reload` (or `ctx.reload()`) → `session_shutdown{reason:"reload"}` → invalidate old runner → reload settings → reset API providers → reload resources → rebuild runtime preserving active tools + flag values → `session_start{reason:"reload"}` + `resources_discover` (`agent-session.ts:3659-3697`; builtin command `packages/coding-agent/src/core/slash-commands.ts:42`). Only the active user theme file is fs-watched (`docs/themes.md:68`, `modes/interactive/theme/theme.ts:808`) → [[no-auto-hot-reload]].
- **Stale-context invalidation**: after reload/session replacement captured `pi`/`ctx` throw an explanatory error ("This extension ctx is stale after session replacement or reload… move post-replacement work into withSession…", `loader.ts:192-200`); invalidation also unsubscribes event-bus listeners (`loader.ts:196-198`, `6ca423447`). `ExtensionCommandContext.newSession/fork/switchSession` take `withSession(ctx)` callbacks giving a fresh `ReplacedSessionContext` (`packages/coding-agent/src/core/extensions/types.ts:398-439`; `1cc303d05`, `f0cf8a59d`).
- Load timings via `PI_TIMING=1` (`core/timings.ts:3-6`).
- Host deps must be `peerDependencies: "*"`; pi warns when they appear in `dependencies` (`resource-loader.ts:53-88`, `8d897edaa`) → [[pi--harness-package-distribution]].

### Experimental Chord facets (`packages/chord`, `PI_EXPERIMENTAL=1`)
- Plugin = facets, one per environment (session worker, TUI, …): `Facet {id; setup(env)}` (`packages/chord/src/types.ts:303-306`); `FacetEnvironment` = `use/observe/provide/provideMany/replicatedState/own/onActivate/onDeactivate` (`packages/chord/src/types.ts:281-301`).
- Setup must be synchronous ("Facet ${id} setup must be synchronous", `facets/host.ts:379-386`); host records calls in a ledger — no parallel `requires/provides` manifest (`PLANNING.md:132`).
- Lifecycle `setting_up|prepared|active|disposing|dead` (`host.ts:47`); activation: setup all → assembling (resolve sources, validate graph, topological order) → connecting (await `ready()`) → activating (providers before consumers) → active; failure → terminate with `AggregateError` (`host.ts:388-421`). Graph validation rejects duplicate providers, singleton/keyed mix, double provision, multi-source offers, cycles (`host.ts:607,624,822-847,876-877`).
- Co-located remotely-exposable services still go through a loopback binding so replacement semantics don't depend on placement (`services/loopback.ts:5-16`, `host.ts:663`; `PLANNING.md:301`).
- **Shape-preserving reload** `FacetHost.reload` (`host.ts:423-511`): stage + setup candidates → must preserve requirements/provisions (`host.ts:441-443`) → activate candidates while old providers stay routed → `provision.replace(provider)` per singleton (no unavailable gap) → dispose retired facets in reverse → connect keyed provisions. Failure after cutover → abort "Facet reload failed after cutover" (`host.ts:506-508`); "There is no rollback after cutover" (`PLANNING.md:227`). Running work in a retired facet is NOT drained (`PLANNING.md:255`; reversed `5dd8c0132`). Structural add/remove still planned (`PLANNING.md:229-251`).
- **Stable facades**: consumer handles are stable Proxies over lazily created member slots (`services/consumer.ts:142-173,26-66`); `then` returns `undefined` so facades aren't thenable (`consumer.ts:62`); closed binding → `service_stale_instance` (`consumer.ts:123-127`); keyed instances fenced by host-owned generation (`services/provider.ts:183-212,377-410`).
- **Bundling**: esbuild → one content-addressed CJS file per facet entry + `chord-facets.json`; peer deps externalized; never installs deps or runs lifecycle scripts (`chord/README.md:183-226`); `FACET_BUNDLE_FORMAT="chord.facet-bundle"` v2 (`node/manifest.ts:1-5`); integrity `sha256-<base64>` (`node/bundle.ts:152`) verified before evaluation (`node/bundle-loader.ts:25,233-240`).
- **GC-able generations**: loader compiles via `node:vm` `compileFunction` (`bundle-loader.ts:7,192`) to stay outside Node's ESM cache, which "retains every imported module generation" (`PLANNING.md:187`; `65f77ecd6`). Server builds `src/session.ts` + `src/tui.ts` facets per plugin package, ships TUI artifacts to client; `/reload` rebuilds and cuts over via `FacetHost.reload()` (`packages/coding-agent/src/experimental/services/README.md:18-19,32`). Example: `examples/plugins/pi-example-plugin` (`contract.ts`, `session.ts`, `tui.ts`).

## Constants
| name | value | path:line |
|---|---|---|
| jiti module cache | `moduleCache:false` | `loader.ts:577` |
| discovery depth | 1 level | `loader.ts:736-822` |
| facet bundle format | `chord.facet-bundle` v2 | `packages/chord/src/node/manifest.ts:1-5` |

## Evolution
- 2026-01-05 `c6fc08453` hooks + custom tools unified into extensions (#454); 2026-01-07 `cb3ac0ba9` shared runtime with throwing stubs.
- 2026-01-13 `1919fd7c9` / `843f23525` jiti fork `@mariozechner/jiti` with `virtualModules` for Bun binary (#681) — compiled binary has no `node_modules`.
- 2026-01-20 `b846a4bfc` ResourceLoader, package management, `/reload` (#645).
- 2026-04-22/23 `1cc303d05` / `f0cf8a59d` `withSession` callbacks + stale ctx errors (#2860, #3606).
- 2026-05-07 `50993d743` back to upstream jiti 2.7 (#4244); `3e5ad67e0` `@mariozechner/*` → `@earendil-works/*`, both aliased.
- 2026-08-29 `34dc9d055` facet services moved into Chord; `024439d15` clarified Chord remote service APIs (consolidation before the shape-preserving reload work of 2026-09-01).
- 2026-06-05 `89a92207f` project-trust bootstrap pass; 2026-06-20 `5505316ea` factory cache for same-dir switches.
- 2026-08-05 `6ca423447` event-bus listeners unsubscribed on invalidation (#7656); 2026-08-18 `c06132898` load in Node SEA hosts (#8237); 2026-08-23 `a69bef789` transactional commit/discard (#8424).
- 2026-08-28 `28b49a6b3` Chord runtime foundation; 2026-08-30 `45d0174ee` facet-owned capability views, loopback, terminal failure after cutover; `0252dff88`/`429f4e756` bundled facet distribution + `/reload`, package plugins; 2026-08-31 `65f77ecd6` ESM → CJS + `node:vm`; 2026-09-01 `c4b0e35ab`/`5dd8c0132` services stay available during reload, stop draining admitted calls.
- 2026-09-19 `40c256ccc` defer loader deps; 2026-09-23 `8d897edaa` peer-install suppression + warnings (#9863).

## Evidence commits
`c6fc08453` `cb3ac0ba9` `1919fd7c9` `843f23525` `b846a4bfc` `1cc303d05` `f0cf8a59d` `50993d743` `3e5ad67e0` `89a92207f` `5505316ea` `6ca423447` `c06132898` `a69bef789` `28b49a6b3` `45d0174ee` `0252dff88` `429f4e756` `65f77ecd6` `c4b0e35ab` `5dd8c0132` `40c256ccc` `8d897edaa`

## Quirks
- Reload is all-or-nothing for the session runtime (shutdown → start); in-flight work is not preserved by stable `/reload`.
- Lifecycle handlers calling `ctx.reload()` can deadlock — only safe in user-initiated commands (`docs/extensions.md:214-215`).
- Extensions run with full process permissions; transactional loading is about consistency, not isolation (`docs/extensions.md:5`).
- Chord sequence-gap recovery: replica clears + reports, no automatic resubscribe (`PLANNING.md:839`, open) — see [[replicated-state]].

## Durable variant (packages/durable)
- `Registry` of named extensions; reload in place; running work keeps its snapshot; tasks hand over at the next phase boundary — if the definition object changed the task commits back to `pending` and a fresh invocation reserves it (`packages/durable/docs/spec.md:2077-2087`). Old and new Harness instances must never own the same Session concurrently (`spec.md:3288-3291`).
- Missing/incompatible task code never terminalizes a task: it stays `blocked: missing_task|task_too_old|migration_failed` (`spec.md:2011-2032`; `harness/scheduler.ts:57`).

## Failures
[[plugins-fail-to-load-in-compiled-binary]] · [[duplicate-host-module-instances]] · [[plugin-registrations-leak-after-failure-or-reload]] · [[stale-plugin-context-after-session-replacement]] · [[module-cache-retains-plugin-generations]] · [[reload-cannot-drain-in-flight-calls]]
