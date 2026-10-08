---
type: implementation
harness: opencode
concept: lsp-diagnostics-feedback
commit: ecc4916b5a
files: [packages/opencode/src/lsp/diagnostic.ts:3-28, packages/opencode/src/lsp/client.ts:13-18, packages/opencode/src/lsp/lsp.ts:150-153, packages/opencode/src/lsp/lsp.ts:437, packages/opencode/src/lsp/server.ts:152, packages/opencode/src/tool/edit.ts:195-199, packages/opencode/src/tool/write.ts:18, packages/opencode/src/tool/write.ts:74-89, packages/opencode/src/tool/apply_patch.ts:269-292, packages/opencode/src/tool/read.ts:119, packages/opencode/src/tool/lsp.ts:11-21, packages/opencode/src/tool/lsp.txt:3-22]
---
[[lsp-diagnostics-feedback]] in [[opencode]].

## Mechanism
### Legacy runtime
- After a successful write, `edit` calls `lsp.touchFile(path, "document")`, then `lsp.diagnostics()`, and appends `LSP.Diagnostic.report(...)` under "LSP errors detected in this file, please fix:" (`packages/opencode/src/tool/edit.ts:195-199`). Output always starts "Edit applied successfully." → [[empty-success-output-read-as-failure]].
- `write`: same for the written file, plus up to 5 other files under "LSP errors detected in other files:" (`packages/opencode/src/tool/write.ts:18,74-89`). `apply_patch`: per patched file (`packages/opencode/src/tool/apply_patch.ts:269-292`).
- `report` keeps only severity 1 (ERROR), ≤ 20 per file plus "... and N more", wrapped in `<diagnostics file="…">` (`packages/opencode/src/lsp/diagnostic.ts:3,20-28`) → [[harness-diagnostics-channel]].
- Waits: 150 ms debounce on pushed diagnostics, 5 s document wait, 10 s full wait, 3 s per request, 45 s initialize (`packages/opencode/src/lsp/client.ts:13-18`); pull diagnostics supported since `e383df4b17`.
- `read` warms the server with a background `touchFile` (`packages/opencode/src/tool/read.ts:119`).
- Server table: 38 `Info` entries in `packages/opencode/src/lsp/server.ts`, many auto-downloaded unless `OPENCODE_DISABLE_LSP_DOWNLOAD` (`server.ts:152`; `packages/opencode/src/effect/runtime-flags.ts:22`).
- **Opt-in**: `if (!cfg.lsp)` → "all LSPs are disabled" (`packages/opencode/src/lsp/lsp.ts:150-153`).
- Experimental `lsp` tool (flag `OPENCODE_EXPERIMENTAL_LSP_TOOL`, `runtime-flags.ts:45`): goToDefinition, findReferences, hover, documentSymbol, workspaceSymbol, goToImplementation, call hierarchy (`packages/opencode/src/tool/lsp.ts:11-21`); workspace symbols capped `.slice(0, 10)` (`packages/opencode/src/lsp/lsp.ts:437`).
- `lsp` tool description lists the nine operations and fixes coordinates as 1-based line and character "as shown in editors"; for `workspaceSymbol` the `filePath` only selects which server to start and is not sent in the request (`packages/opencode/src/tool/lsp.txt:3-22`).
### v2 runtime
- Not ported: core built-ins TODO lists "LSP" among deliberately unported leaves (`packages/core/src/tool/builtins.ts:26-29`).

## Constants
| name | value | path:line |
|---|---|---|
| `MAX_PER_FILE` | 20 | `packages/opencode/src/lsp/diagnostic.ts:3` |
| `MAX_PROJECT_DIAGNOSTICS_FILES` | 5 | `packages/opencode/src/tool/write.ts:18` |
| `DIAGNOSTICS_DEBOUNCE_MS` | 150 | `packages/opencode/src/lsp/client.ts:13` |
| document / full / request wait | 5 000 / 10 000 / 3 000 ms | `packages/opencode/src/lsp/client.ts:14-16` |
| `INITIALIZE_TIMEOUT_MS` | 45 000 | `packages/opencode/src/lsp/client.ts:18` |

## Evolution
- 2025-08-21 `aa4dba1541` LSP spawn failure no longer injected as edit diagnostics.
- 2025-11-18 `81ebf56cf1` top-level `lsp: false` / `formatter: false`.
- 2025-12-14 `aedb5550a8` limit diagnostics "to prevent context window waste".
- 2025-12-16 `ef78fd8bae` debounce to get complete results; 2025-12-26 `46c7a41d5f` block only when errors exist → [[stale-diagnostics-after-edit]].
- 2025-12-21 `345f4801e8` experimental `lsp` tool; 2025-12-26 `1e2ef07c97` `lsp-hover`/`lsp-diagnostics` tools killed as unused.
- 2026-01-12 `66f9bdab32` neutral framing + explicit success line.
- 2026-04-16 `220e3e9a2b` "make formatter config opt-in" also flips `lsp` to opt-in (touches `packages/opencode/src/lsp/lsp.ts`, `packages/opencode/src/config/lsp.ts`); docs `6b68b1020e` 2026-05-03. Rationale not in the commit body (unverified).
- 2026-04-23 `e383df4b17` pull diagnostics + init timeout.

## Quirks / drift
- Opt-in default means most sessions get no diagnostics at all, while prompts (e.g. default.txt lint/typecheck rules) still assume a feedback loop.

Contrast: pi has no LSP client → [[no-lsp]].
