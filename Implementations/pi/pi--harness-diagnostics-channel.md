---
type: implementation
harness: pi
concept: harness-diagnostics-channel
commit: b30a6dd77
files: [packages/durable/src/harness/tool.ts:440-485, packages/durable/docs/spec.md:2827-2848, packages/durable/src/tools/read.ts:72, packages/durable/src/tools/read.ts:175-214, packages/durable/src/tools/bash.ts:107, packages/coding-agent/src/core/tools/read.ts:188-199]
---
[[harness-diagnostics-channel]] in [[pi]].

## Mechanism
- **Stable pi (coding-agent): no separate channel.** Harness/tool remarks are appended inline to tool content as bracketed notices: "[Showing lines X-Y of N. Use offset=K to continue.]", "[N more lines in file. Use offset=K to continue.]" (`packages/coding-agent/src/core/tools/read.ts:188-199`), bash "[Showing lines X-Y of N. Full output: <tmp path>]" (`bash.ts:348-352`), grep "Some lines truncated to 500 chars. Use read tool to see full lines" (`b813a8b92`; HEAD `packages/coding-agent/src/core/tools/grep.ts:299`) → [[tool-output-truncation]], [[tool-output-spill]].
- **Durable harness (pi-durable): explicit channel** — see section below.

## Constants
| name | value | path:line |
|---|---|---|
| render format | `<harness>\n[severity] message\n</harness>` | `packages/durable/src/harness/tool.ts:483-485` |
| harness-owned code | `truncated` (warn) | `packages/durable/src/harness/tool.ts:448` |
| tool codes | `full_output` (info), `unsupported_image` (error), `tool_unavailable` (error) | `durable/src/tools/bash.ts:107`, findings 10, `spec.md:778` |

## Evolution
- `b813a8b92` 2025-12-07 (#134): coding-agent inline "actionable notices" for truncation.
- `e045ed2f3` 2026-09-09 → `729d5cb74` 2026-09-17: Pico design drafts → finalized spec incl. "Diagnostics are a channel, not text".
- `445770e03` 2026-09-29 (Package 16 "first coding-agent tool turn"): `renderDiagnostics` / `appendToolResult` implemented.

## Evidence commits
`b813a8b92`, `e045ed2f3`, `729d5cb74`, `445770e03`.

## Quirks
- In the durable model the *rendering* is still text appended to the result (model sees it), but the boundary is unambiguous (`<harness>` tag) and the structured list is stored separately (`data: {diagnostics}`), so UIs never parse model text.
- Provider-side diagnostics (e.g. Codex `provider_transport_failure`, pi-messages `pi_messages_rewrite`) are attached to assistant messages for hosts; whether they are ever model-visible is (unverified) — they are a separate, host-facing channel.

## Durable variant (packages/durable)
- Spec: "Diagnostics are a channel, not text. Remarks about a call, such as truncated output, a spill path, a corrected path, a file changed on disk, or a capped search, go through `api.diagnostic()` or `ToolExecutionResult.diagnostics`, never into the content the model reads as the tool's data. … Every diagnostic is model-visible; information only for UIs belongs in `details`." (`packages/durable/docs/spec.md:2827-2833`).
- Order at settlement: those recorded via `api`, then result's, then Harness's (`spec.md:2834-2835`); harness adds `warn`/`truncated` when bounding drops text ("stating the dropped lines and bytes", `spec.md:2824-2825`; `harness/tool.ts:448`).
- `appendToolResult` appends one text item `renderDiagnostics(...)` at the end of content so "the stored message is exactly what the model sees; `data` keeps the structured list" (`harness/tool.ts:453-480`). A `warn` diagnostic does not set `isError` (`spec.md:2848`).
- Tools: durable read "Remarks about truncation and continuation are diagnostics; the content is only file text." (`durable/src/tools/read.ts:72, 175-214`; e.g. "Use offset=N to continue", oversized-line `sed -n 'Np' | tail -c +K` hint); bash spill path `info`/`full_output` "Full output: <path>" (`durable/src/tools/bash.ts:107`); read refuses images with `unsupported_image` error diagnostic.

## Failures
- (none recorded; motivation is preventing harness remarks from being mistaken for file/command data — cf. [[placeholder-text-misleads-model]])
