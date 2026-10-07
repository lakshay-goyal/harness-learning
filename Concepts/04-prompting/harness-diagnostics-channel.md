---
type: concept
stage: messages
tier: candidate
aliases: [diagnostics, "<harness>", ToolDiagnostic, "api.diagnostic()", "<image_resize_notice>", ImageResizeNotice, "Warning: truncated output"]
harnesses: [pi, codex]
---
Harness/tool remarks to the model (truncation, spill path, corrected path, file changed) travel in a delimited, structured channel separate from the tool's data content.

## Why
- Inline notices inside tool output are indistinguishable from file/command bytes for both model and UI; placeholder text gets treated as data (cf. [[placeholder-text-misleads-model]]).
- UIs/code need the remarks structured, not parsed out of model text.

## Design space
- **Inline bracketed notices in content** ("[Showing lines X-Y of N. Use offset=K to continue.]") — stable pi.
- **Structured diagnostics list rendered as one trailing `<harness>[severity] message</harness>` block, stored separately** — pi-durable.
- Separate message role / metadata field never shown to the model (host-only diagnostics on provider errors in pi-ai).
- **Harness remark as its own developer-role message placed right after the content it describes, kept grouped with it through compaction** ✔ codex (`<image_resize_notice>` "Image i of n in the preceding user message was resized from WxH to wxh pixels.").
- **Fixed plain-text header/prefix lines in tool output** (`Chunk ID / Wall time / Process exited with code N / Original token count: N / Output:`; `Warning: truncated output (original token count: N)`) ✔ codex; structured JSON with `output_schema` for code-mode callers instead.
- UI-only warnings never sent to the model ("Heads up: Long threads and multiple compactions can cause the model to be less accurate…") ✔ codex.

## Implementations
- [[pi--harness-diagnostics-channel|pi]] — coding-agent inline notices; durable spec "Diagnostics are a channel, not text".
- [[codex--harness-diagnostics-channel|codex]] — inline headers/prefixes in tool output; separate `<image_resize_notice>` developer message; model-visible placeholders for omitted media.

## Failures
- (none recorded)

## Related
[[xml-prompt-boundaries]] · [[tool-output-truncation]] · [[tool-output-spill]] · [[durable-execution]] · [[tool-error-as-result]] · [[image-normalization]] · [[message-role-layering]]
