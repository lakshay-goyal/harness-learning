---
type: concept
stage: messages
tier: candidate
aliases: [diagnostics, "<harness>", ToolDiagnostic, "api.diagnostic()", "<diagnostics file>"]
harnesses: [pi, opencode]
---
Harness/tool remarks to the model (truncation, spill path, corrected path, file changed) travel in a delimited, structured channel separate from the tool's data content.

## Why
- Inline notices inside tool output are indistinguishable from file/command bytes for both model and UI; placeholder text gets treated as data (cf. [[placeholder-text-misleads-model]]).
- UIs/code need the remarks structured, not parsed out of model text.

## Design space
- **Inline bracketed notices in content** ("[Showing lines X-Y of N. Use offset=K to continue.]") — stable pi.
- **Structured diagnostics list rendered as one trailing `<harness>[severity] message</harness>` block, stored separately** — pi-durable.
- Separate message role / metadata field never shown to the model (host-only diagnostics on provider errors in pi-ai).
- Lead with an explicit success line, then neutral diagnostics (opencode).

## Implementations
- [[pi--harness-diagnostics-channel|pi]] — coding-agent inline notices; durable spec "Diagnostics are a channel, not text".
- [[opencode--harness-diagnostics-channel|opencode]] — inline only: `<diagnostics file>` blocks, parenthesized paging notes, `<system-reminder>` in read output; explicit success line first, neutral diagnostics after.

## Failures
- [[tool-output-bypasses-truncation]]
- (none recorded)
- [[empty-success-output-read-as-failure]]
- [[stale-diagnostics-after-edit]]

## Related
[[xml-prompt-boundaries]] · [[tool-output-truncation]] · [[tool-output-spill]] · [[durable-execution]] · [[tool-error-as-result]]
