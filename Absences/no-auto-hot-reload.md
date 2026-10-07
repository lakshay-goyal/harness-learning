---
type: absence
harnesses: [pi]
---
# no-auto-hot-reload

**What's missing**
- Extensions, skills, prompt templates, settings, keybindings and context files are **not file-watched**. Edits take effect only after an explicit `/reload` (or `ctx.reload()`), or on a new session.
- Only exception: the active user theme file is fs-watched. "Pi hot-reloads the active user theme only from `<agent-dir>/themes/<name>.json`. Run `/reload` after adding or changing a theme from any other source." (`packages/coding-agent/docs/themes.md:68`; watcher `src/modes/interactive/theme/theme.ts:808`).

**Evidence of decision**
- `/reload` builtin: "Reload keybindings, extensions, skills, prompts, themes, and context files" (`packages/coding-agent/src/core/slash-commands.ts:42`). Introduced with ResourceLoader and package management in `b846a4bfc` (2026-01-20, #645).
- The mechanism is a full teardown/rebuild, not incremental (`packages/coding-agent/src/core/agent-session.ts:3659-3697`):
  - emit `session_shutdown{reason:"reload"}` and invalidate the old runner;
  - reload settings, reset API providers, reload resources;
  - rebuild the runtime, keeping active tools and flag values;
  - emit `session_start{reason:"reload"}` and `resources_discover`.
- Loader: jiti with `moduleCache: false` plus an explicit factory cache keyed by cwd+generation, cleared on `/reload` (`src/core/extensions/loader.ts:131-150,577`; `resource-loader.ts:509-513`; `5505316ea`).
- No stated rationale against watching (absence by omission). Implied reasons: reload runs `session_shutdown` handlers that "await cleanup", and `reload` is "only safe in user-initiated commands" because lifecycle handlers calling it can deadlock (`src/core/extensions/types.ts:398-439`; `docs/extensions.md:214-215`).

**Failures that shaped manual reload** → [[runtime-plugin-loading]]
- A captured `ctx` silently pointed at the old session after reload/new/fork. Fix: invalidation plus an explicit error, and `withSession` (`1cc303d05`, `f0cf8a59d`; `packages/coding-agent/src/core/extensions/loader.ts:192-200`).
- Event-bus listeners survived reload. Fix: tracked unsubscribers (`6ca423447`; `packages/coding-agent/src/core/extensions/loader.ts:162,193-211,523`).

**Opt-in replacement**
- `examples/extensions/reload-runtime.ts`: `/reload-runtime` plus an LLM-callable tool that queues a follow-up command to trigger `ctx.reload()` (`reload-runtime.ts:1-6`). The model can reload after writing an extension.
- `examples/extensions/file-trigger.ts` watches a file and injects its contents. This is a pattern for user-built watching.
- Runtime `registerTool` applies immediately without `/reload` (`packages/coding-agent/CHANGELOG.md:3296`).

**Durable / experimental variant** (reload is designed in, still not file-watch triggered)
- Durable spec: at every phase boundary a task whose definition object changed hands over. It commits back to `pending` and a fresh invocation reserves it under the new code (`packages/durable/docs/spec.md:2077-2087`). Old and new Harness instances never own the same Session (`spec.md:3288-3291`). Example: `packages/durable/test/examples/31-reload-and-restart.ts` ("Running work finishes on the code it started with").
- Chord facets: shape-preserving `FacetHost.reload()` swaps providers with no unavailable gap. "There is no rollback after cutover"; running work in a retired facet is not drained (`packages/chord/src/facets/host.ts:423-511`; `PLANNING.md:227,255`). The experimental services reload the configured Session facet generation and the TUI presentation plugins (`packages/coding-agent/src/experimental/services/README.md:18-19`).

**Implication**
- Deterministic: a half-saved extension file never loads mid-turn. The cost is a manual step in the extension dev loop.
- Because reload is a full session_shutdown → session_start cycle, plugins must be written to survive it. Stale-context invalidation exists for exactly that reason.

Related: [[runtime-plugin-loading]] · [[extension-event-hooks]] · [[replaceable-builtin-extension]] · [[harness-package-distribution]] · [[durable-execution]] · [[Absences]]
