---
type: failure
concepts: [session-tree, tool-output-truncation]
harnesses: [codex]
---
**Symptom** — "just 3 MCP tool calls that each were multi-hundred MBs" produced enormous rollout JSONL files (records `mcp_tool_call_end` and `function_call_output`), slowing resume/listing and filling disk.

**Root cause** — Model-visible truncation applied only to live history; the transcript persisted full payloads with no size policy of its own.

**Fix · [[codex]]**
- `3516cb9751` 2026-04-30 (#20260) — persisted ItemCompleted MCP results and command output capped at 64 KiB with marker `"\n... command output truncated for persistence ...\n"` (`codex-rs/rollout/src/policy.rs:16-19`, `:250-275`).
- `bee28e8a06` 2026-10-02 — paginated-history command output cap 64 KiB (from subject).
- Still open: `function_call_output` ResponseItems are persisted untruncated — live history comment says the rollout keeps full payloads (`codex-rs/core/src/context_manager/history.rs:514-515`); whether they are capped at persistence time is unverified.

**Lesson** — Transcript persistence needs its own size policy, separate from model-visible truncation.

Related: [[session-tree]] · [[tool-output-truncation]] · [[no-tool-output-spill-file]] · [[codex--session-tree|codex]] · [[codex--tool-output-truncation|codex]]
