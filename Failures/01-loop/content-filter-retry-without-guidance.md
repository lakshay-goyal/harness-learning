---
type: failure
concepts: [auto-retry-backoff]
harnesses: [codex]
---
**Symptom** — A content-filter stop (`response.incomplete` reason `content_filter`) is classified retryable; retrying the identical prompt reproduces the same block.

**Root cause** — A semantic refusal treated like a transient transport error: the retry did not change the input.

**Fix · [[codex]]**
- Before the retry decision, record a model-specific content-filter guidance fragment into history, then retry (`codex-rs/core/src/responses_retry.rs:67-82`); moved into the shared retry handler `a9118edae8` 2026-09-29. Guidance text is model-owned: `ModelMessages.content_filter_guidance`, ≤ 512 bytes, else bundled guidance (`codex-rs/protocol/src/openai_models.rs:543-548`).

**Lesson** — A retry for a semantic refusal must change the prompt; otherwise classify it terminal.

Related: [[auto-retry-backoff]] · [[truncated-tool-call-guard]] · [[per-model-system-prompt]] · [[codex--auto-retry-backoff|codex]]
