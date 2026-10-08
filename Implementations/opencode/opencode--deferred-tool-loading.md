---
type: implementation
harness: opencode
concept: deferred-tool-loading
commit: ecc4916b5a
files: [packages/codemode/codemode.md:47-50, packages/codemode/src/tool-runtime.ts:86-87, packages/codemode/src/tool-runtime.ts:554-560, packages/opencode/src/session/tools.ts:388, packages/opencode/src/permission/index.ts:204-219, packages/core/src/tool/registry.ts:106-135]
---
[[deferred-tool-loading]] in [[opencode]]. Partial: only via code mode; no `tool_search` for directly declared tools.

## Mechanism
### Legacy runtime
- Default: every allowed tool (built-in, custom, MCP) is declared on every request.
- With `OPENCODE_EXPERIMENTAL_CODE_MODE`, MCP tools are undeclared (`packages/opencode/src/session/tools.ts:388`) and reachable only as `tools.<server>.<tool>()` inside `execute` → [[code-mode]].
- Inside code mode: inline catalog under a 2 000-token budget, round-robin across namespaces; header says COMPLETE or PARTIAL with counts; `$codemode.search` (default 10 hits) finds the rest (`packages/codemode/codemode.md:47-50`; `packages/codemode/src/tool-runtime.ts:86-87,554-560`). Catalog lines cut to the first description line ≤ 120 chars.
- Hiding by permission is all-or-nothing (deny `*` → undeclared) (`packages/opencode/src/permission/index.ts:204-219`) → [[permission-ruleset]].
### v2 runtime
- "Definition filtering is catalog visibility, not execution authorization": a tool is hidden only when the last matching rule is `resource:"*" effect:"deny"`; leaves still authorize at execution (`packages/core/src/tool/registry.ts:106-135`; `packages/core/src/tool/AGENTS.md:45-47`).
- codemode.md's v2 design groups deferred tools into codemode namespaces behind one `execute`; not wired on dev.

## Constants
| name | value | path:line |
|---|---|---|
| `defaultCatalogBudget` | 2 000 tokens (chars/4) | `packages/codemode/src/tool-runtime.ts:86` |
| `defaultSearchLimit` | 10 | `packages/codemode/src/tool-runtime.ts:87` |

## Evolution
- 2026-07-02 `cb93114424` codemode; 2026-07-06 `d4f7039932` unified catalog signatures.
- 2026-06-06 `4814ab3a3d` v2 tool definitions filtered by agent permissions (were not before).

## Quirks / drift
- No discovery path for built-ins or custom tools; deferral exists only for MCP under an experimental flag.

Contrast: pi has five exposure tiers and a BM25 `tool_search` for any tool → [[pi--deferred-tool-loading|pi]].
