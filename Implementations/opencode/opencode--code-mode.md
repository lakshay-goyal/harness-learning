---
type: implementation
harness: opencode
concept: code-mode
commit: ecc4916b5a
files: [packages/codemode/codemode.md:16-27, packages/codemode/codemode.md:35, packages/codemode/codemode.md:47-50, packages/codemode/codemode.md:88-92, packages/codemode/codemode.md:116-145, packages/codemode/src/codemode.ts:9-17, packages/codemode/src/interpreter/runtime.ts, packages/codemode/src/tool-runtime.ts:86-87, packages/codemode/src/tool-runtime.ts:122, packages/codemode/src/tool-runtime.ts:554-560, packages/codemode/src/stdlib/promise.ts:6, packages/opencode/src/tool/code-mode.ts:59, packages/opencode/src/tool/code-mode.ts:142-154, packages/opencode/src/tool/code-mode.ts:239-286, packages/opencode/src/tool/registry.ts:118, packages/opencode/src/tool/registry.ts:288, packages/opencode/src/session/tools.ts:388]
---
[[code-mode]] in [[opencode]].

## Mechanism
### Package `@opencode-ai/codemode`
- Goals: shrink catalog context, avoid a round trip per dependent call, keep intermediates out of context, "Give generated code only the authority explicitly supplied by the host" (`packages/codemode/codemode.md:16-27`).
- Engine: TypeScript stripped (`transpileModule`), Acorn parse, **owned tree-walking interpreter** written as Effect generators (`packages/codemode/codemode.md:35`; `src/interpreter/runtime.ts`, 3465 lines). Not QuickJS, `node:vm` or WASM.
- Globals allowlist only (`tools`, restricted `Promise`, `Object`, `Math`, `JSON`, `Date`, `RegExp`, `Map`, `Set`, `URL`, …); no `fetch`, timers, fs, process, modules, `eval`; `__proto__`/`constructor`/`prototype` blocked at the data boundary (`src/tool-runtime.ts:152`) → [[codemode-sandbox-escape-to-host]].
- Catalog: token-budgeted (`defaultCatalogBudget = 2_000`, chars/4), round-robin across namespaces, `$codemode.search` always callable (`codemode.md:47-50`; `src/tool-runtime.ts:86-87`) → [[deferred-tool-loading]].
- Instructions to the model: "This is a restricted JavaScript language for calling tools, not a general-purpose runtime"; "Do not infer or normalize tool names; use only exact signatures shown below or returned by search" (`src/tool-runtime.ts:554-560`).
- `Promise.all` inside scripts ≤ 8 concurrent calls (`src/stdlib/promise.ts:6`).
- Limits `timeoutMs`, `maxToolCalls`, `maxOutputBytes` have **no defaults** ("absent means no timeout / unlimited / no truncation", `src/codemode.ts:9-17`); "Leave execution-limit defaults to hosts" (`codemode.md:142`).
- Failures returned as diagnostic data; interrupts stay interrupts.
### Legacy host (`packages/opencode`)
- Tool `execute`, flag `OPENCODE_EXPERIMENTAL_CODE_MODE`, lazily imported (`packages/opencode/src/tool/registry.ts:118`); hidden when the MCP catalog is empty (`:288`).
- Exposes **only MCP tools**, grouped by server namespace; direct MCP declarations removed (`packages/opencode/src/session/tools.ts:388`) → [[mcp-integration]].
- Each child call: `tool.execute.before` hook → permission ask on the MCP tool name (`always: ["*"]`) → MCP call with abort + `resetTimeoutOnProgress` → `tool.execute.after` (`packages/opencode/src/tool/code-mode.ts:142-154`) → [[nested-tool-calls]].
- `CodeMode.make` called with no limits (`code-mode.ts:239`); cancellation via `Effect.raceFirst(runtime.execute(code), abort)` → "Execution cancelled." (`:274-286`). Output bounded once by normal tool truncation.
### v2 runtime
- Design: deferred tools grouped into namespaces behind one `execute`; nested calls skip registry bounding (`codemode.md:88-92`). Not wired on dev: core built-ins list "Rune/code mode" as unported (`packages/core/src/tool/builtins.ts:28`).

## Constants
| name | value | path:line |
|---|---|---|
| script timeout | none (host decides; opencode passes none) | `packages/codemode/src/codemode.ts:11` |
| `TOOL_CALL_CONCURRENCY` | 8 | `packages/codemode/src/stdlib/promise.ts:6` |
| `defaultCatalogBudget` / `defaultSearchLimit` | 2 000 tokens / 10 | `packages/codemode/src/tool-runtime.ts:86-87` |
| `MAX_VALUE_DEPTH` | 32 | `packages/codemode/src/tool-runtime.ts:122` |

## Evolution
- Superseded a vendored interpreter at `packages/opencode/src/session/rune/` ("rune", deleted) (`cb93114424:packages/codemode/codemode.md:32-33`).
- 2026-07-02 `cb93114424` "experimental codemode (#34677)"; reverted same day `379adee35c`; 2026-07-03 `2409c7a3d5` confined execution package; `ed6dc879be` execute gated behind flag.
- 2026-07-04 `a8983bd2c7` OpenAPI tool adapter ("Incorrect parameter encoding … is worse than a precise `skipped` reason", `packages/codemode/codemode.md:143`).
- 2026-07-06 `d4f7039932` unified catalog signatures; `d5aa79c73a` v2 sync.

## Quirks / drift
- Design doc rule: "Never reference external prior-art implementations … in code, comments, commit messages, or docs" (`cb93114424:packages/codemode/codemode.md:51`).
- `Date.now()` is real host time, so scripts are not replay-deterministic.
- Namespace collisions are "last write wins" (`cb93114424:packages/codemode/codemode.md:59-61`) → [[mcp-tool-name-collision]].

Contrast: pi runs scripts in a fresh QuickJS WASM VM with memory/output caps; opencode owns the language and leaves budgets to the host → [[pi--code-mode|pi]].
