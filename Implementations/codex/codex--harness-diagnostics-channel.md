---
type: implementation
harness: codex
concept: harness-diagnostics-channel
commit: 622e9e3696
files: [codex-rs/core/src/tools/context.rs:524, codex-rs/core/src/tools/context.rs:186, codex-rs/utils/output-truncation/src/lib.rs:23, codex-rs/core/src/context/image_resize_notice.rs:37, codex-rs/core/src/compact_remote_history.rs:55, codex-rs/core/src/context/unsupported_media.rs:11, codex-rs/core/src/compact.rs:419]
---
[[harness-diagnostics-channel]] in [[codex]].

## Mechanism
- **Inline headers in tool output** (no separate channel): exec result text `Chunk ID: x\nWall time: 0.1234 seconds\nProcess exited with code N | Process running with session ID N\nOriginal token count: N\nOutput:\n…` (`codex-rs/core/src/tools/context.rs:524-548`); MCP `Wall time: X seconds\nOutput:` (`:186-200`); truncation prefix `Warning: truncated output (original token count: N)\nTotal output lines: L` and in-body `…N tokens truncated…` (`codex-rs/utils/output-truncation/src/lib.rs:23-41`) → [[codex--tool-output-truncation]]. Code-mode callers get a structured JSON object instead (`codex-rs/core/src/tools/handlers/shell_spec.rs:197-226`).
- **Separate developer message**: feature `image_resize_notice` appends `<image_resize_notice>` after the user message / tool output: "Image i of n in the preceding user message was resized from WxH to wxh pixels." (`codex-rs/core/src/context/image_resize_notice.rs:37-75`); remote compaction keeps it grouped with its source (`codex-rs/core/src/compact_remote_history.rs:39-66`) → [[codex--image-normalization]].
- **Placeholders as diagnostics**: "image content omitted because you do not support image input" (`codex-rs/core/src/context/unsupported_media.rs:11-19`); "image content omitted because remote image URLs are not supported"; "Output exceeded the available model context and was truncated" (remote compaction pre-trim, `codex-rs/core/src/compact_remote_history.rs:16-17`).
- **UI-only**: after compaction a Warning event "Heads up: Long threads and multiple compactions can cause the model to be less accurate. Start a new thread when possible…" (`codex-rs/core/src/compact.rs:419-423`) — not model-visible; transient events (exec deltas, warnings, stream errors) are not persisted to the rollout (`codex-rs/rollout/src/policy.rs:156-215`).

## Evolution
- 2026-05-18 `82061660ae` removed legacy JSON-structured shell output — "Current shell and apply_patch responses are already plain text for model consumption".
- 2026-08-04 `4bd5b9fd09` "Keep image resize notices attached during remote compaction (#36956)"; 2026-08-05 `fa5d5ae047` (#37134) resize notice.

## Versus pi
- [[pi--harness-diagnostics-channel]]: coding-agent inline bracketed notices; durable package moves remarks into a trailing `<harness>[severity] …</harness>` block. codex: mostly inline headers, but uses a *separate message* for image-resize remarks — a role-level channel rather than an in-content block.
