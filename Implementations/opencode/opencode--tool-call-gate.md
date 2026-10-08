---
type: implementation
harness: opencode
concept: tool-call-gate
commit: ecc4916b5a
files: [packages/plugin/src/index.ts:261-269, packages/opencode/src/session/tools.ts:100-130, packages/opencode/src/session/prompt.ts:307-311, packages/opencode/src/tool/code-mode.ts:141-145, packages/opencode/src/plugin/index.ts:284-297, specs/v2/tools.md:131]
---
[[tool-call-gate]] in [[opencode]].

## Mechanism
### Legacy runtime
- **Two layers**: the declarative [[permission-ruleset]] is the built-in gate (each tool calls `ctx.ask` with its own patterns before acting); the plugin hook `tool.execute.before` is the programmable seam.
- **Hook shape**: `"tool.execute.before"(input: {tool, sessionID, callID}, output: {args})` (`packages/plugin/src/index.ts:266-269`). Plugins mutate `output.args` in place; no return value, no `{block}` result (docs: "mutate `output` in place; return `void`", `packages/core/src/plugin/skill/customize-opencode.md:345`).
- **Coverage**: built-in and plugin tools (`packages/opencode/src/session/tools.ts:100-130`), MCP tools (`packages/opencode/src/session/tools.ts:259`, `packages/opencode/src/session/tools.ts:339`), user-invoked subtasks (`packages/opencode/src/session/prompt.ts:307-311`) and code-mode nested MCP calls (`packages/opencode/src/tool/code-mode.ts:141-145`). Runs before `item.execute`, i.e. before the tool's own permission ask.
- **Blocking** = throwing: `Plugin.trigger` runs hooks sequentially via `Effect.promise(async () => fn(input, output))` (`packages/opencode/src/plugin/index.ts:284-297`), so a rejection becomes a defect that fails the tool call (inferred from code; no explicit fail-closed contract documented).
- **Mutated args are not re-validated** against the tool schema (inferred: validation happens before `execute`).
- **Dead decision hook**: `"permission.ask"(input, output: {status: ask|deny|allow})` is still declared (`packages/plugin/src/index.ts:261`) but has no trigger since the legacy permission module was deleted (`2fc06c5a17`, 2026-03-14) ([[dead-hook-in-public-api]]).
- **Side door**: command templates' `` !`cmd` `` expansion runs a shell with no `tool.execute.before` and no `bash` permission ([[side-door-input-bypasses-hooks]]).

### v2 runtime
- No registry-level gate: "Trusted tools formulate and sequence permission requests. … The registry does not inject an `assertPermission` helper" (`specs/v2/tools.md:131`). No `tool.execute.before` trigger exists under `packages/core/src` (grep).
- Stale registrations are rejected at settlement ("Stale tool call: <name>", `packages/core/src/tool/registry.ts:50-61,116-120`).

## Constants
| name | value | path:line |
|---|---|---|
| hook timeout | none | `packages/opencode/src/plugin/index.ts:284-297` |

## Evolution
- 2025-12-10 `070ced0b3f` reverted a try/catch around hooks "that surpressed errors" ([[hook-error-fails-open]]).
- 2026-03-14 `2fc06c5a17` legacy permission module deleted; `permission.ask` trigger gone.
- 2026-03-24 `814a515a8a` per-hook try/catch only for `config` hooks; two-phase plugin init.
- 2026-03-28 `55895d0663` sync hooks returning non-promises broke `Effect.promise`.

## Quirks / drift
- Policy plugins cannot veto with a reason; a thrown error's message becomes the tool error text (inferred).
- The v2 rebuild removes the plugin seam for tool calls entirely so far; policy lives only in permissions.

pi contrast: pi's single seam is `tool_call` returning `{block, reason}`, fail-closed on throw, no built-in rules ([[pi--tool-call-gate|pi]]).
