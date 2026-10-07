---
type: implementation
harness: opencode
concept: nested-tool-calls
commit: ecc4916b5a
files: [packages/opencode/src/tool/code-mode.ts:59, packages/opencode/src/tool/code-mode.ts:134-185, packages/opencode/src/tool/code-mode.ts:239-286, packages/codemode/codemode.md:88-92]
---
[[nested-tool-calls]] in [[opencode]]. Only through code mode.

## Mechanism
### Legacy runtime
- `execute` scripts call MCP tools as `tools.<server>.<tool>(input)`; each child call goes through `tool.execute.before` → `ctx.ask` permission on the MCP tool key (`always: ["*"]`) → MCP client call (abort signal, `resetTimeoutOnProgress`) → `tool.execute.after` (`packages/opencode/src/tool/code-mode.ts:134-185`) → [[tool-call-gate]], [[tool-result-rewriting]].
- Built-in tools (read, shell, edit…) are not callable from scripts; only MCP.
- No call budget (`maxToolCalls` unset) and no per-call record on the parent result beyond the script's output; the outer `execute` result is truncated once by normal tool truncation (`code-mode.ts:239`).
- Cancellation interrupts the interpreter fiber and its supervised children (`code-mode.ts:274-286`).
### v2 design (not wired on dev)
- Nested calls check registration currency and skip registry bounding; the outer `execute` is the single bounding boundary (`packages/codemode/codemode.md:88-110`).

## Constants
| name | value | path:line |
|---|---|---|
| nested calls per script | unlimited | `packages/codemode/src/codemode.ts:13` |
| concurrent nested calls | 8 | `packages/codemode/src/stdlib/promise.ts:6` |

## Evolution
- 2026-07-02 `cb93114424` codemode; 2026-07-03 `ed6dc879be` execute gated behind flag.

## Quirks / drift
- No provenance ids linking child calls to the parent call in the transcript (none found; unverified).

Contrast: pi routes `ctx.executeTool` through the full pipeline for any tool and records bounded nested-call summaries → [[pi--nested-tool-calls|pi]].
