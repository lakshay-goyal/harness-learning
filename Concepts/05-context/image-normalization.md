---
type: concept
stage: context
tier: must-have
aliases: [processImage, normalizeToolResultImages, images.autoResize, images.blockImages, "Image reading is disabled.", inputLimits.images.resize, image-ingest-normalization, image-context-normalization, multimodal-tool-result-routing, Image.normalize, supportsMediaInToolResult, SYNTHETIC_ATTACHMENT_PROMPT, "Attached media from tool result:", prepare_conversation_items_for_history, PromptImageMode, ImagePreparationMode, UnifiedImageBudget, ImageReference, AttachmentStore, UnsupportedMedia]
harnesses: [pi, opencode, codex]
---
Every image is sniffed, converted, resized and size-capped (or blocked/replaced by text) at the point it enters history, because one bad image block persisted in the transcript makes every later request fail.

## Why
- Provider limits are per-request, not per-turn: an oversized or corrupt image in history gets the *whole conversation* rejected on every subsequent call — the session is bricked ([[image-content-poisoning]]).
- Images arrive from many doors: user attachments, the read tool, plugin/MCP tools, script `image()` calls — normalizing only one door is not enough (pi fixed read first, tool results 8 months later).
- Images cost ~1–1.5k tokens each in estimates; they must be counted ([[estimator-undercounts-context]]).
- Wire shape of images inside tool results differs per provider (`tool-result-image-routing`, 02 group).

## Design space
- Sniff by magic bytes vs extension ✔ pi (magic, 4100 bytes; reject JPEG-LS, animated PNG; full GIF signature).
- Convert unsupported formats (BMP→PNG) with a text hint ✔ pi; SVG never supported.
- Resize policy: ≤2000×2000, ≤4.5 MB base64 (headroom under Anthropic 5 MB), PNG vs JPEG, JPEG quality ladder, then shrink dims ×0.75 ✔ pi; per-model profile from catalog `inputLimits.images.resize` ✔ pi (`f5c946480`); **patch-based**: high/auto ≤2048 px & 2,500 patches, original ≤6000 px & 10,000 patches (32-px patches), one unified budget for original-capable models ✔ codex.
- Single choke point: all user-message and tool-output images prepared centrally before insertion into history ✔ codex (pi normalized door by door).
- Remote URLs: fetched by provider vs **rejected with model-visible text** ✔ codex (`7153affa0f`); detail `low` rejected ✔ codex.
- Upload to an attachment store and reference by file id, inline fallback ✔ codex.
- Resize notice as a separate developer message grouped with its image through compaction ✔ codex.
- Invalid image error from provider: auto-remove/retry (codex tried, removed `8431dc590a`) vs **end turn asking the user to remove it** ✔ codex.
- Replay re-projected for the receiving model: strip images for text-only models, downgrade `detail: original` → `high` ✔ codex.
- On failure: replace with text "[Image omitted: …]" (read) vs keep original block (tool results — may be missing backend) ✔ pi (both, by door).
- Coordinate hint after downscale ("Multiply coordinates by S") ✔ pi.
- Global kill switch: `blockImages` replaces images with "Image reading is disabled." at conversion time, history keeps them ✔ pi.
- Non-vision models: text placeholder per image at provider boundary ✔ pi (→ [[cross-provider-handoff]]).
- Worker thread for resizing (Photon WASM) with in-process fallback ✔ pi.
- Cap at the provider's hard limit (5 MB base64, opencode) vs with headroom (4.5 MB, pi).
- Tool-result media routing by SDK allow-list: inline where the SDK accepts media in tool results, else hoist into a synthetic user message "Attached media from tool result:" (opencode legacy); structured blocks per native protocol (opencode v2).
- Per-provider format drop (xAI rejects GIF etc. → dropped, opencode).

## Implementations
- [[pi--image-normalization|pi]] — `processImage` + `image-resize-core` (2000px / 4.5 MB / q80 ladder) applied to prompt images, read tool, and all tool results after extension hooks; `blockImages` convertToLlm wrapper; durable read refuses images.
- [[codex--image-normalization|codex]] — central `prepare_conversation_items_for_history`; 2048 px / 2,500 patches (high), 6000 px / 10,000 (original); remote URLs → placeholder; attachment-store upload; modality-aware `for_prompt` stripping.
- [[opencode--image-normalization|opencode]] — `Image.normalize` (Photon, 2000×2000 / 5 MB, JPEG ladder) on tool-result attachments at completion, un-resizable images omitted with a note; `supportsMediaInToolResult` routing with synthetic-user hoisting.

## Failures
- [[image-content-poisoning]]
- [[tool-result-image-routing]]
- [[compaction-loses-modality-or-structure]]
- Cross-group: [[placeholder-text-misleads-model]] (02) · [[model-switch-replays-unsupported-content]] (02)

## Related
[[message-conversion-layer]] · [[file-read-tool]] · [[cross-provider-handoff]] · [[token-estimation]] · [[tool-result-rewriting]] · [[code-mode]] · [[model-catalog]]
