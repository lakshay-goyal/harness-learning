---
type: failure
concepts: [image-normalization, token-estimation, transcript-replay-repair]
harnesses: [pi, opencode, codex]
---
**Symptom** — One bad image block in history bricked the session: every later request was rejected (HTTP 400 / size errors) and switching models didn't help. Variants:
- Oversized images exceeded Anthropic's 5 MB / per-image dimension limits (the per-image cap drops from 8000px to 2000px once a request carries many images).
- Resize fallback kept the original oversized image.
- Images produced by extensions/MCP/screenshot tools bypassed resizing entirely.
- Codemode `image()` accepted invalid base64, which "made every later provider request fail with HTTP 400" (`packages/coding-agent/CHANGELOG.md` 0.99.2).
- Text files beginning with "GIF" were sniffed as images and omitted.
- Under `node --watch` (Node 24.19+/26.x), Node's own `{"watch:require": …}` worker message was taken as the resize reply → image silently dropped as "could not be resized below the inline image size limit" (resize appeared to hang/fail).

**Root cause** — Provider limits apply per request, and history is replayed on every request; images entered history through several doors, only some of them normalized, with non-strict sniffing and an untagged worker protocol.

**Fix · [[pi]]**
- `4a32af253` 2026-01-02 automatic resize; `69dc6b078` 2026-01-03 (#424) PNG/JPEG + quality ladder to stay under 4.5 MB base64 (`packages/coding-agent/src/utils/image-resize-core.ts:31-38`, `132`).
- `a78882d8b` 2026-03-22 (#2055) enforce safe limits: unresizable → `[Image omitted: could not be resized below the inline image size limit.]` instead of the original (`packages/coding-agent/src/utils/image-process.ts:91`).
- `b0e05b442` 2026-08-03 (#7330) normalize tool-result images in `afterToolCall`, after the extension `tool_result` hook (`packages/coding-agent/src/core/agent-session.ts:704-711`; `packages/coding-agent/src/utils/tool-result-images.ts:13-67`): "Oversized images make the provider reject the whole conversation, not just the offending turn".
- `f5c946480` 2026-09-20 (#9631) per-model `inputLimits.images.resize`.
- `47a18e37b` 2026-09-21 (#9755) require the complete GIF signature (`packages/coding-agent/src/utils/mime.ts:13`).
- `d2931ad3d` 2026-09-30 (#10215) codemode `image()` validates base64 + magic signature (`packages/codemode/src/runtime/prelude-source.ts:392-427`).
- `b30a6dd77` 2026-10-07 (#10527, HEAD) worker replies tagged `pi:image-resize-response`; untagged messages ignored (`packages/coding-agent/src/utils/image-resize-core.ts:21-29`).
- Related: `39b1bf7b6` (#2734) Anthropic 413 `request_too_large` recognized as overflow (see [[overflow-message-not-recognized]]); `96f0edd02` (#4983) image tokens counted.

**Fix · [[codex]]**
- Symptom variant (model switch): switching from a multimodal to a text-only model sent historic `input_image` parts and every request failed.
- `5e01450963` 2026-02-10 "Strip unsupported images from prompt history to guard against model switch (#11349)": modality-aware `for_prompt` replaces images with "image content omitted because you do not support image input" (`codex-rs/core/src/context_manager/normalize.rs:330-380`); `6eecd04fc1` 2026-09-09 downgrade `detail: original` for receiving models that lack it ([[model-switch-replays-unsupported-content]]).
- All user/tool images prepared centrally before entering history; remote URLs replaced by model-visible text (`7153affa0f`); oversize/failed → placeholders (`codex-rs/core/src/image_preparation.rs:31-37`, `:338-342`).
- Provider-rejected image: `8431dc590a` 2026-07-20 removed the "replace with 'Invalid image' and retry" path; the turn now ends with "Invalid image in your last message. Please remove it and try again." (`codex-rs/core/src/session/turn.rs:793-815`) — the bad block stays in history (no auto-removal).
**Fix · [[opencode]]** `9eefcd1b41` 2025-12-15 (#5521) reading an empty image file produced an Anthropic error on every request → replaced by a text error; `563177c6ac` 2026-05-01 (#25241) tool result with image + empty text caused API errors; `85ce6a5f95` 2026-05-10 (#26401) auto-resize to 2000×2000 / 5 MB with a JPEG quality ladder, un-resizable images dropped with "[N images omitted: could not be resized below the image size limit.]" (`packages/opencode/src/image/image.ts:10-14`; `packages/opencode/src/session/processor.ts:390-412`).

**Lesson** — One bad content block persisted in history bricks the session: validate and normalize media at every door before it enters the transcript, re-project history for the *receiving* model's capabilities on every request, and tag cross-thread protocol messages.

Related: [[image-normalization]] · [[tool-result-image-routing]] · [[placeholder-text-misleads-model]] · [[code-mode]] · [[estimator-undercounts-context]] · [[codex--image-normalization|codex]]
