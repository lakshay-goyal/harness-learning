---
type: failure
concepts: [summary-validation, transcript-serialization-for-summary]
harnesses: [pi, opencode]
---
**Symptom** — The summarization model answered with tool calls instead of a summary (the serialized transcript is full of tool calls); forcing `toolChoice:"none"` then broke providers/gateways that reject `tool_choice` without tools.

**Root cause** — Summary requests inherited tool-ish context; prevention via `tool_choice` is not portable across providers.

**Fix · [[pi]]**
- `90305d90a` 2026-08-17 "disable tools during summarization": `toolChoice:"none"` + reject responses containing a `toolCall`.
- `fe37e9f9b` 2026-08-25 (#8607) adapter silently omitted `tool_choice` without tools → reverted by `6b36eb592` 2026-08-26 (#8649, #8638), which removed the explicit tool choice from compaction callers instead and kept post-hoc rejection: "Summarization attempted to call a tool" (`packages/coding-agent/src/core/compaction/compaction.ts:759-761`, `1111-1113`; `branch-summarization.ts:363-365`). No tools are sent at all.

**Fix · [[opencode]]** Legacy compaction sends `tools: {}` and the processor throws "Tool call not allowed while generating summary: <name>" on any tool input/call (`packages/opencode/src/session/processor.ts:316-334`). Variant: LiteLLM/Bedrock/Copilot reject histories with tool calls but no `tools` field, so a dummy `_noop` tool ("Do not call this tool…") is injected — and summarizers called it; `196a03caff` 2026-03-30 (#18539) added "Do not call any tools. Respond only with the summary text." to the compaction prompt (no longer in `compaction.txt` at HEAD; `_noop` now only for GitHub Copilot, `packages/opencode/src/session/llm/request.ts:159-175`, `f9d99f044d` 2026-04-15). v2 sends `tools: []` (`packages/core/src/session/compaction.ts:207`).

**Lesson** — Don't rely on `tool_choice` portability; send no tools and validate the output.

Related: [[summary-validation]] · [[transcript-serialization-for-summary]] · [[empty-payload-rejections]] · [[compaction-request-shape-mismatch]]
