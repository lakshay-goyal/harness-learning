---
type: implementation
harness: opencode
concept: plugin-tools
commit: ecc4916b5a
files: [packages/opencode/src/tool/registry.ts:121-215, packages/opencode/src/tool/registry.ts:318, packages/opencode/src/tool/registry.ts:355, packages/plugin/src/index.ts:226-228, packages/core/src/tool/registry.ts:45-125, specs/v2/tools.md:151]
---
[[plugin-tools]] in [[opencode]].

## Mechanism
### Legacy runtime
- **Two sources**: (1) every config dir's `{tool,tools}/*.{js,ts}` module, each export a tool (`default` → file name, else `<file>_<export>`); (2) a plugin's `tool` hook map (`packages/opencode/src/tool/registry.ts:183-205`; hook `packages/plugin/src/index.ts:226-228`). Files are `import()`ed directly as `file://` URLs after `config.waitForDependencies()`.
- **Schema**: plugin `ToolDefinition` uses Zod `args`; all-Zod args → `z.object(args)` → JSON Schema, otherwise legacy JSON-Schema args; missing `args` normalized to `{}` ("pre-1.14.49 … Zod silently tolerated undefined (#27451, #27630)") (`packages/opencode/src/tool/registry.ts:125-136`) ([[plugin-tool-without-schema-breaks-requests]]).
- **Output**: plugin tool output goes through the same `Truncate` service as built-ins.
- **Per-request rewrite**: `tool.definition` hook may change any tool's description or schema (`packages/plugin/src/index.ts:334`).
- **Visibility**: plugin tools are subject to the [[permission-ruleset]] like built-ins (`*` deny hides them).
- **Name conflicts**: later registrations shadow earlier ones by id (inferred: map keyed by id).

### v2 runtime
- **Scoped overlay registration** (`packages/core/src/tool/registry.ts:45-125`): application-scope tools (`ApplicationTools`) plus Location-scope registrations stacked per name; latest wins; closing the registering Scope removes exactly that registration via a finalizer and reveals the previous one.
- **Materialization**: each provider turn snapshots definitions filtered by agent permissions and records a registration identity per advertised name; at settlement a call whose identity no longer matches settles as `Stale tool call: <name>` (`packages/core/src/tool/registry.ts:50-61`; `specs/v2/tools.md:151`). A running call keeps its captured handler even if removed later.

## Constants
| name | value | path:line |
|---|---|---|
| custom tool glob | `{tool,tools}/*.{js,ts}` | `packages/opencode/src/tool/registry.ts:185` |

## Evolution
- 2026-02-04 `556adad67b` custom tools loaded before their deps were installed → wait.
- 2026-04-02 `81d3ac3bf0` `Tool.define()` wrapper accumulation on object-defined tools ([[tool-wrapper-accumulation]]).
- 2026-04-16 `7b3bb9a761` plugin tool metadata dropped from results.
- 2026-05-18 `12ae22378f` plugin `ask` returned an Effect instead of a Promise.
- 2026-05-19 `c79a9634d3` plugin tool defs with missing `args` crashed the registry.
- 2026-06-06 `660a00d317` unified v2 tool architecture.

## Quirks / drift
- Project `.opencode/tool(s)/*.ts` files execute on open with no trust prompt ([[untrusted-repo-loads-executable-config]]).

pi contrast: `pi.registerTool` with TypeBox params, renderers and exposure tiers; registration validated at load ([[pi--plugin-tools|pi]]).
