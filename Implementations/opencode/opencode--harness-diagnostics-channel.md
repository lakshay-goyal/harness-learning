---
type: implementation
harness: opencode
concept: harness-diagnostics-channel
commit: ecc4916b5a
files: [packages/opencode/src/lsp/diagnostic.ts:20-28, packages/opencode/src/tool/edit.ts:193-199, packages/opencode/src/tool/read.ts:345-356, packages/opencode/src/tool/shell.ts:560-578]
---
[[harness-diagnostics-channel]] in [[opencode]]. Inline only; no separate structured channel.

## Mechanism
### Legacy runtime
- Harness remarks are appended to tool content, delimited ad hoc:
  - LSP: "LSP errors detected in this file, please fix:" + `<diagnostics file="…">ERROR [l:c] msg</diagnostics>` (`packages/opencode/src/lsp/diagnostic.ts:20-28`; `packages/opencode/src/tool/edit.ts:193-199`) → [[lsp-diagnostics-feedback]].
  - Read paging: parenthesized "(Showing lines a-b of N. Use offset=N to continue.)" / "(End of file - total N lines)" (`packages/opencode/src/tool/read.ts:345-349`).
  - Nested instruction files: `<system-reminder>` in read output (`read.ts:356`) → [[ephemeral-reminder-injection]].
  - Shell: timeout/abort notes ("shell tool terminated command after exceeding timeout…", "User aborted the command") plus truncation/spill notice around the tail (`packages/opencode/src/tool/shell.ts:560-578`).
- Framing matters: neutral "LSP errors detected" + leading "Edit applied successfully." replaced "This file has errors, please fix" so the model stops reading success as failure (`66f9bdab32` 2026-01-12) → [[empty-success-output-read-as-failure]].
- Only actionable severities: ERROR-only since `46c7a41d5f` (2025-12-26) → [[stale-diagnostics-after-edit]].

## Constants
| name | value | path:line |
|---|---|---|
| diagnostics per file | 20 | `packages/opencode/src/lsp/diagnostic.ts:3` |

## Evolution
- 2025-11-13 `7ec32f834e` explicit EOF marker.
- 2025-12-16 `ef78fd8bae` debounce; 2025-12-26 `46c7a41d5f` errors only; 2026-01-12 `66f9bdab32` neutral framing.

## Quirks / drift
- Each tool uses its own delimiter style (parentheses, XML tag, `<system-reminder>`), so the model sees three conventions for harness remarks.

Contrast: pi-durable renders a structured `<harness>[severity] …</harness>` block stored apart from content → [[pi--harness-diagnostics-channel|pi]].
