---
type: concept
stage: messages
tier: candidate
aliases: [diagnostics, "<harness>", ToolDiagnostic, "api.diagnostic()"]
harnesses: [pi]
---
Harness/tool remarks to the model (truncation, spill path, corrected path, file changed) travel in a delimited, structured channel separate from the tool's data content.

## Why
- Inline notices inside tool output are indistinguishable from file/command bytes for both model and UI; placeholder text gets treated as data (cf. [[placeholder-text-misleads-model]]).
- UIs/code need the remarks structured, not parsed out of model text.

## Design space
- **Inline bracketed notices in content** ("[Showing lines X-Y of N. Use offset=K to continue.]") — stable pi.
- **Structured diagnostics list rendered as one trailing `<harness>[severity] message</harness>` block, stored separately** — pi-durable.
- Separate message role / metadata field never shown to the model (host-only diagnostics on provider errors in pi-ai).

## Implementations
- [[pi--harness-diagnostics-channel|pi]] — coding-agent inline notices; durable spec "Diagnostics are a channel, not text".

## Failures
- (none recorded)

## Related
[[xml-prompt-boundaries]] · [[tool-output-truncation]] · [[tool-output-spill]] · [[durable-execution]] · [[tool-error-as-result]]
