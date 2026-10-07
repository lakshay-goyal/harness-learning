---
type: implementation
harness: opencode
concept: extension-event-hooks
commit: ecc4916b5a
files: [packages/plugin/src/index.ts:220-334, packages/opencode/src/plugin/index.ts:160-200, packages/opencode/src/plugin/index.ts:284-297, packages/opencode/src/session/llm/request.ts:69-146, packages/opencode/src/session/prompt.ts:1255, packages/opencode/src/session/processor.ts:530-538, packages/opencode/src/session/compaction.ts:373-379, packages/opencode/src/session/compaction.ts:500-517, packages/llm/DESIGN.md:738-804]
---
[[extension-event-hooks]] in [[opencode]].

## Mechanism
### Legacy runtime (`@opencode-ai/plugin`)
- **Plugin** = async function `(ctx: {client, project, directory, worktree, serverUrl, $, …}) => Hooks` (`packages/plugin/src/index.ts:57-60`, `packages/plugin/src/index.ts:220-334`).
- **Hooks** (`packages/plugin/src/index.ts:223-334`):
  - lifecycle/registration: `dispose`, `event` (every bus event), `config` (merged config), `tool` (register tools, [[plugin-tools]]), `auth`, `provider`.
  - chat: `chat.message`, `chat.params`, `chat.headers`.
  - tools: `tool.execute.before` (mutate args, [[tool-call-gate]]), `tool.execute.after` (mutate output), `tool.definition` (rewrite description/schema), `shell.env` (inject env).
  - commands/permissions: `command.execute.before`, `permission.ask` (declared, never triggered since `2fc06c5a17`, [[dead-hook-in-public-api]]).
  - experimental: `experimental.chat.messages.transform` (in-memory history per request, `packages/opencode/src/session/prompt.ts:1255`), `experimental.chat.system.transform` (`packages/opencode/src/session/llm/request.ts:69-73`), `experimental.session.compacting` (inject context or replace the summary prompt, `packages/opencode/src/session/compaction.ts:373-379`), `experimental.compaction.autocontinue` (suppress the synthetic "Continue…", `packages/opencode/src/session/compaction.ts:500-517`), `experimental.text.complete` (rewrite final assistant text before persistence, `packages/opencode/src/session/processor.ts:530-538`).
- **Shape**: every hook is `(input, output) => Promise<void>`; plugins **mutate `output` in place**. No return-value chaining, no `{block}` result.
- **Composition**: `Plugin.trigger` runs hooks sequentially in load order over the same `output` object, so later plugins see earlier mutations (`packages/opencode/src/plugin/index.ts:284-297`).
- **Failure policy**: no try/catch around hook calls; `Effect.promise` turns a rejection into a defect, failing the surrounding operation (inferred). Only `config` hooks are individually guarded (`814a515a8a`) ([[hook-error-fails-open]]).
- **Timeouts**: none.
- **Loading order**: internal plugins first (skipped by `OPENCODE_DISABLE_DEFAULT_PLUGINS`), then config/project plugins (skipped by `OPENCODE_PURE`) (`packages/opencode/src/plugin/index.ts:170-200`) ([[runtime-plugin-loading]]).

### v2 runtime
- Direction: "Move behavior out of large application services and into plugins. Core services should become small, typed containers that own state, expose simple operations, and trigger hooks" (`specs/v2/instructions.md:7`); built-in agents are themselves a core plugin (`packages/core/src/plugin/agent.ts:97`).
- `packages/llm/DESIGN.md:738-804` (status "Discussion draft"): five hook stages (canonical request, provider-native body, prepared transport request, normalized event, error); hooks "may not secretly short-circuit execution, synthesize a response, retry, or redirect control flow"; customization ladder generation controls → typed provider options → hooks → raw HTTP overlays → protocol patching.
- `CONTEXT.md:225` flags `experimental.chat.system.transform` ("can mutate the assembled baseline system prompt arbitrarily") as incompatible with an immutable cache baseline (unresolved).

## Constants
| name | value | path:line |
|---|---|---|
| hook timeout | none | `packages/opencode/src/plugin/index.ts:284-297` |
| TUI plugin dispose timeout | `DISPOSE_TIMEOUT_MS = 5000` | `packages/opencode/src/plugin/tui/runtime.ts:122` |

## Evolution
- 2025-11-02 `894cbaa51e` duplicate plugin subscriptions per instance.
- 2025-12-10 `070ced0b3f` reverted a try/catch around hooks "that surpressed errors".
- 2026-01-04 `c3fd3c8656` plugin exported under two names initialized twice.
- 2026-03-14 `2fc06c5a17` `permission.ask` trigger deleted with the legacy permission module.
- 2026-03-24 `814a515a8a` per-hook try/catch for `config` only, two-phase init; 2026-03-28 `55895d0663` sync hooks returning non-promises.
- 2026-05-09 `5e49029e70` plugins mutating the provider model object leaked across providers → public copy.

## Quirks / drift
- Mutation-in-place means a plugin can hold and mutate the object after the hook returns (inferred; `5e49029e70` is one such leak).

pi contrast: 41 typed `pi.on` events with chained return-value transforms and fail-closed `tool_call` ([[pi--extension-event-hooks|pi]]).
