---
type: implementation
harness: opencode
concept: tool-output-spill
commit: ecc4916b5a
files: [packages/opencode/src/tool/truncate.ts:12-17, packages/opencode/src/tool/truncate.ts:127-148, packages/opencode/src/tool/truncation-dir.ts:4, packages/opencode/src/agent/agent.ts:296-310, packages/core/src/tool-output-store.ts:13-17, packages/core/src/tool-output-store.ts:129-136, packages/core/src/tool-output-store.ts:158-159, packages/core/src/tool-output-store.ts:176-205, packages/core/src/plugin/agent.ts:11, packages/core/src/session/runner/llm.ts:320-324, CONTEXT.md:192-198, specs/v2/tools.md:157]
---
[[tool-output-spill]] in [[opencode]].

## Mechanism

### Legacy runtime
- On truncation the full text is written to `TRUNCATION_DIR` = `<data>/tool-output` (`packages/opencode/src/tool/truncation-dir.ts:4`) and the path is named in the hint "Full output saved to: <file>" (`packages/opencode/src/tool/truncate.ts:127-131`); `metadata.outputPath` carries it structurally (`packages/opencode/src/tool/tool.ts:142`).
- Every agent is auto-allowed `external_directory` on `Truncate.GLOB` unless a rule explicitly denies it (`packages/opencode/src/agent/agent.ts:296-310`), so read/grep can open the file.
- Retention 7 days: cleanup every 1 h after a 1-min delay (`truncate.ts:12`, `:143-148`); cleanup by file mtime since `d468201952`.

### v2 runtime — `ToolOutputStore`, "Managed Tool Output File"
- `write`: `<data>/tool-output/tool_<ascending id>`, created with `flag: "wx"` (exclusive) (`packages/core/src/tool-output-store.ts:129-136`); marker "... output truncated; full content saved to <path> ..." placed between head and tail (`:158-159`); the path is also returned as typed `outputPaths`.
- Flat shared directory, globally unique names; "Their absolute paths are readable and searchable by ordinary tools" (`CONTEXT.md:198`); default permissions whitelist the glob (`packages/core/src/plugin/agent.ts:11`).
- "The bounded Model Tool Output, not the file, is the durable replayable record" (`CONTEXT.md:193`); files expire after 7 days via one global hourly cleanup (`tool-output-store.ts:176-205`).
- **Spill failure**: `CONTEXT.md:194` says retention failure "does not change a successful tool operation into a failed one. The Session records an explicitly lossy bounded output without a path"; `specs/v2/tools.md:157` says "if complete retention fails, settlement fails operationally rather than publishing lossy success". Code follows tools.md: `StorageError` from `write` propagates out of `bound` → settlement fiber fails → runner `failUnsettledTools("Tool execution failed: Failed to write tool output…")` (`packages/core/src/session/runner/llm.ts:320-324`) for a tool whose side effect already happened → [[spill-failure-reports-successful-side-effect-as-failed]].

## Constants
| name | value | path:line |
|---|---|---|
| `RETENTION` | 7 days | `packages/opencode/src/tool/truncate.ts:12`; `packages/core/src/tool-output-store.ts:15` |
| cleanup cadence | every 1 h (legacy: first after 1 min) | `packages/opencode/src/tool/truncate.ts:143-148`; `packages/core/src/tool-output-store.ts:203` |
| `MANAGED_DIRECTORY` | `"tool-output"` | `packages/core/src/tool-output-store.ts:17` |

## Evolution
- 2026-01-18 `e2f1f4d81e` scheduler + cleanup module; 2026-03-17 `5dfe86dcb1` effectified `TruncateService`, scheduler deleted.
- 2026-06-05 `a9094fd059` v2 bounded output with managed spill files.
- 2026-08-07 `d468201952` cleanup by file mtime instead of id timestamp.

## Quirks / drift
- The legacy hint tells task-capable agents not to read the spill file themselves but to delegate to the explore subagent — spill as a delegation trigger.
- Spec contradiction on spill failure is unresolved at HEAD.

Contrast: [[pi--tool-output-spill|pi]] writes private 0600 temp files per tool and kills the command when spill fails (durable); opencode uses one shared 7-day directory whitelisted for reads, and v2 fails the settlement after the side effect.
