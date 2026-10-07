---
type: failure
concepts: [summary-validation, transcript-serialization-for-summary]
harnesses: [pi]
---
**Symptom** — The summarization model answered with tool calls instead of a summary (the serialized transcript is full of tool calls); forcing `toolChoice:"none"` then broke providers/gateways that reject `tool_choice` without tools.

**Root cause** — Summary requests inherited tool-ish context; prevention via `tool_choice` is not portable across providers.

**Fix · [[pi]]**
- `90305d90a` 2026-08-17 "disable tools during summarization": `toolChoice:"none"` + reject responses containing a `toolCall`.
- `fe37e9f9b` 2026-08-25 (#8607) adapter silently omitted `tool_choice` without tools → reverted by `6b36eb592` 2026-08-26 (#8649, #8638), which removed the explicit tool choice from compaction callers instead and kept post-hoc rejection: "Summarization attempted to call a tool" (`packages/coding-agent/src/core/compaction/compaction.ts:759-761`, `1111-1113`; `branch-summarization.ts:363-365`). No tools are sent at all.

**Lesson** — Don't rely on `tool_choice` portability; send no tools and validate the output.

Related: [[summary-validation]] · [[transcript-serialization-for-summary]] · [[empty-payload-rejections]] · [[compaction-request-shape-mismatch]]
