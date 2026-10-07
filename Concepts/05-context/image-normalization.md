---
type: concept
stage: context
tier: candidate
aliases: [processImage, normalizeToolResultImages, images.autoResize, images.blockImages, "Image reading is disabled.", inputLimits.images.resize, image-ingest-normalization, image-context-normalization, multimodal-tool-result-routing]
harnesses: [pi]
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
- Resize policy: ≤2000×2000, ≤4.5 MB base64 (headroom under Anthropic 5 MB), PNG vs JPEG, JPEG quality ladder, then shrink dims ×0.75 ✔ pi; per-model profile from catalog `inputLimits.images.resize` ✔ pi (`f5c946480`).
- On failure: replace with text "[Image omitted: …]" (read) vs keep original block (tool results — may be missing backend) ✔ pi (both, by door).
- Coordinate hint after downscale ("Multiply coordinates by S") ✔ pi.
- Global kill switch: `blockImages` replaces images with "Image reading is disabled." at conversion time, history keeps them ✔ pi.
- Non-vision models: text placeholder per image at provider boundary ✔ pi (→ [[cross-provider-handoff]]).
- Worker thread for resizing (Photon WASM) with in-process fallback ✔ pi.

## Implementations
- [[pi--image-normalization|pi]] — `processImage` + `image-resize-core` (2000px / 4.5 MB / q80 ladder) applied to prompt images, read tool, and all tool results after extension hooks; `blockImages` convertToLlm wrapper; durable read refuses images.

## Failures
- [[image-content-poisoning]]
- [[tool-result-image-routing]]
- Cross-group: [[placeholder-text-misleads-model]] (02)

## Related
[[message-conversion-layer]] · [[file-read-tool]] · [[cross-provider-handoff]] · [[token-estimation]] · [[tool-result-rewriting]] · [[code-mode]] · [[model-catalog]]
