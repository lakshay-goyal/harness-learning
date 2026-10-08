---
type: implementation
harness: opencode
concept: tool-output-truncation
commit: ecc4916b5a
files: [packages/opencode/src/tool/truncate.ts:12-17, packages/opencode/src/tool/truncate.ts:85-141, packages/opencode/src/tool/tool.ts:130-144, packages/opencode/src/tool/shell.ts:579, packages/opencode/src/tool/read.ts:13-18, packages/core/src/tool-output-store.ts:13-15, packages/core/src/tool-output-store.ts:74-104, packages/core/src/tool-output-store.ts:138-174, packages/core/src/tool/registry.ts:75, CONTEXT.md:189-199, specs/v2/tools.md:153-159]
---
[[tool-output-truncation]] in [[opencode]].

## Mechanism

### Legacy runtime — one generic wrapper, tools may pre-truncate
- `Tool.define` wrapper runs `Truncate.output` on every tool result unless the tool already set `metadata.truncated` (`packages/opencode/src/tool/tool.ts:130-144`); also applied to plugin, MCP and MCP-resource outputs (`0d49df46ef`).
- `Truncate.output` (`packages/opencode/src/tool/truncate.ts:85-141`): lines OR UTF-8 bytes, whichever first; `direction` head (default) or tail; full text spilled ([[opencode--tool-output-spill]]); notice `...N lines|bytes truncated...` + hint.
- **Hint depends on agent**: if the agent may use `task`: "Use the Task tool to have explore agent process this file with Grep and Read (with offset/limit). Do NOT read the full file yourself - delegate to save context."; else "Use Grep … or Read with offset/limit" (`:129-131`).
- Config override `tool_output.{max_lines, max_bytes}` (`:80-81`).
- Shell does its own tail truncation with "...output truncated..." + saved path (`packages/opencode/src/tool/shell.ts:579`); read has its own 2000 lines / 2000 chars per line / 50 KB (`packages/opencode/src/tool/read.ts:13-18`).
- LSP diagnostics appended to edit/write results capped at 20 per file and 5 other files (`aedb5550a8`).

### v2 runtime — tools return complete output; registry bounds once
- "Tools return complete validated domain output. They do not truncate model-facing output" (`specs/v2/tools.md:155`); `ToolRegistry` calls `resources.bound(...)` at the single settlement boundary (`packages/core/src/tool/registry.ts:75`).
- One aggregate limit per settlement over text parts only (or JSON of the structured value when content is empty); media and structured value untouched (`packages/core/src/tool-output-store.ts:138-174`). "The limit is provider-independent; token pressure belongs to context assembly and compaction" (`CONTEXT.md:190`).
- **Head + tail preview** with the marker in the middle (`tool-output-store.ts:74-104`), multi-byte safe prefix/suffix (`:50-72`).
- Bounding + publishing the settlement is an interruption-safe region (`CONTEXT.md:195`); provider-executed results are not bounded (`CONTEXT.md:199`).
- Producer capture limits are separate (bash `MAX_CAPTURE_BYTES = 1 MiB`, `packages/core/src/tool/bash.ts:21`): "A producer cannot claim a complete retained output after it has already discarded bytes" (`specs/v2/tools.md:159`).

## Constants
| name | value | path:line |
|---|---|---|
| `MAX_LINES` | 2000 | `packages/opencode/src/tool/truncate.ts:14`; `packages/core/src/tool-output-store.ts:13` |
| `MAX_BYTES` | 50 × 1024 | `packages/opencode/src/tool/truncate.ts:15`; `packages/core/src/tool-output-store.ts:14` |
| read `DEFAULT_READ_LIMIT` / `MAX_LINE_LENGTH` | 2000 / 2000 | `packages/opencode/src/tool/read.ts:13-14` |
| v2 bash `MAX_CAPTURE_BYTES` | 1 MiB | `packages/core/src/tool/bash.ts:21` |

## Evolution
- 2025-12-14 `aedb5550a8` cap LSP diagnostics to stop context waste.
- 2026-01-19 `0d49df46ef` truncation applied to MCP outputs too → [[tool-output-bypasses-truncation]].
- 2026-05-14 `e26abd8da9` close shell truncation stream; 2026-05-16 `77db212a0a` read stream byte cap.
- 2026-06-05 `a9094fd059` (#30999) v2 bounding (v2 shipped with none).
- 2026-06-21 `69f1ec22e3` bound web tool bodies (`MAX_RESPONSE_BYTES = 5 MiB`, `packages/core/src/tool/webfetch.ts:17`).

## Quirks / drift
- Legacy keeps the head by default (tail only when a tool asks); v2 always keeps head + tail.
- The delegate-to-explore hint couples truncation to the agent's permission set: the same output reads differently per agent.

Contrast: [[pi--tool-output-truncation|pi]] truncates inside each tool with a per-tool direction (head/tail/middle); opencode legacy truncates at a generic wrapper, v2 moves the single bound to the tool registry and forbids tools from truncating.
