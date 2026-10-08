---
type: implementation
harness: opencode
concept: tool-argument-repair
commit: ecc4916b5a
files: [packages/opencode/src/session/llm.ts:296-317, packages/opencode/src/tool/invalid.ts:9-21, packages/opencode/src/tool/tool.ts:18-34, packages/opencode/src/tool/tool.ts:64, packages/opencode/src/tool/tool.ts:99-129, packages/core/src/tool/tool.ts:93-108, packages/llm/src/tool-runtime.ts:23-76]
---
[[tool-argument-repair]] in [[opencode]].

## Mechanism
### Legacy runtime
- **Name repair** via the AI SDK `experimental_repairToolCall`: if the lowercased name is a registered tool, rename (e.g. Claude-Code-style `Read` → `read`); otherwise rewrite the call to tool `invalid` with `{tool, error}` (`packages/opencode/src/session/llm.ts:296-312`) → [[foreign-harness-tool-hallucination]].
- `invalid` is registered ("Do not use") but excluded from `activeTools`, so it is never offered; it returns "The arguments provided to the tool are invalid: <error>" (`llm.ts:317`; `packages/opencode/src/tool/invalid.ts:12-17`).
- **Argument decode**: `Tool.define` decodes args with the tool's Effect Schema (decoder hoisted once per init); failure → `InvalidArgumentsError`: "The X tool was called with invalid arguments: …\nPlease rewrite the input so it satisfies the expected schema." (`packages/opencode/src/tool/tool.ts:24-34,120-129`); optional per-tool `formatValidationError` (`:64`).
- No shape coercion shim (no stringified-JSON or legacy-field repair found); leniency comes from loosening schemas instead → [[tool-arg-shape-drift]].
### v2 runtime
- Decode failure → `ToolFailure("Invalid tool input: …")`, tool never invoked; invalid output → "Tool returned an invalid value for its output schema" (`packages/core/src/tool/tool.ts:93,108`).
- `ToolRuntime.dispatch` in `packages/llm`: unknown tool / invalid input / `ToolFailure` → error result the model can correct (`packages/llm/src/tool-runtime.ts:23-76`).

## Constants
| name | value | path:line |
|---|---|---|
| tool-name pattern (v2) | `^[A-Za-z][A-Za-z0-9_-]{0,63}$` | `packages/core/src/tool/tool.ts:135` |

## Evolution
- 2025-08-04 `0a42068fbb` "hack to return tool call errors back to model" (the `invalid` sink).
- 2025-10-28 `fc8db6cdf9` negative bash timeout rejected.
- 2026-01-24 `397ee419d1` question `label`/`header` `.max(30)` moved to description "to avoid tool call failures".
- 2026-02-05 `64e2bf8bf0` task param renamed to `task_id` with a resume-only description, for GPT call failures (unverified symptom detail).
- 2026-05-20 `26008696e1` question schema failures surfaced as friendly tool errors.
- 2026-06-22 `d29f5eba92` required bash `description` param removed.

## Quirks / drift
- Schema tool definitions go through provider-specific lowering before sending → [[tool-schema-lowering]].

Contrast: pi repairs shapes per tool (`prepareArguments`) and coerces generically; opencode repairs only names and otherwise asks the model to rewrite → [[pi--tool-argument-repair|pi]].
