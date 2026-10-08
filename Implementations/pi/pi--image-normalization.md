---
type: implementation
harness: pi
concept: image-normalization
commit: b30a6dd77
files: [packages/coding-agent/src/utils/image-resize-core.ts:31, packages/coding-agent/src/utils/image-process.ts:72, packages/coding-agent/src/utils/tool-result-images.ts:24, packages/coding-agent/src/utils/mime.ts:3, packages/coding-agent/src/core/agent-session.ts:704, packages/coding-agent/src/core/agent-session.ts:1935, packages/coding-agent/src/core/tools/read.ts:138, packages/coding-agent/src/core/sdk.ts:296, packages/ai/src/api/transform-messages.ts:12]
---
[[image-normalization]] in [[pi]].

## Mechanism
- **Sniffing** `packages/coding-agent/src/utils/mime.ts:3-23`: first `IMAGE_TYPE_SNIFF_BYTES = 4100` bytes by magic numbers — JPEG (rejects `FF D8 FF F7` JPEG-LS), PNG (rejects **animated PNG** `acTL`), GIF87a/89a (full signature required, `47a18e37b` #9755: text files starting "GIF" were treated as images), WEBP (RIFF…WEBP), BMP (validated header).
- **`processImage`** (`packages/coding-agent/src/utils/image-process.ts:72-119`): non-native formats (BMP) converted to PNG with hint `[Image converted from X to Y.]`; conversion failure → `[Image omitted: could not be converted to a supported inline image format.]`; resize failure → `[Image omitted: could not be resized below the inline image size limit.]` (`a78882d8b` #2055: previously fell back to the original oversized image).
- **Resize** (`packages/coding-agent/src/utils/image-resize-core.ts:31-166`, Photon Rust/WASM in a worker thread, in-process fallback, `packages/coding-agent/src/utils/image-resize.ts:84-109`): defaults 2000×2000, `maxBytes = 4.5 MiB` of base64 ("Provides headroom below Anthropic's 5MB limit"), JPEG q80; EXIF orientation; resize to max dims, try PNG and JPEG, then JPEG quality steps {80,85,70,55,40}, then shrink dims ×0.75 down to 1×1; null on failure. Coordinate hint `[Image: original WxH, displayed at wxh. Multiply coordinates by S to map to original image.]` (`packages/coding-agent/src/utils/image-resize.ts:115-122`). Worker replies tagged `pi:image-resize-response` so Node's own `{"watch:require":…}` messages under `node --watch` are ignored (`packages/coding-agent/src/utils/image-resize-core.ts:21-29`; HEAD `b30a6dd77`).
- **Per-model profile**: `model.inputLimits.images.resize` from the generated catalog (`f5c946480` #9631; generator default `DEFAULT_IMAGE_RESIZE` 2000/2000/4.5 MiB/80 "cache-safe", `packages/ai/scripts/generate-models.ts:424-429`; Anthropic `maxRequestBytes 32 MiB`, `maxPerRequest` 100/600, Bedrock `maxPerMessage 20`, `:994-1020`).
- **Doors** (`images.autoResize`, default true, `packages/coding-agent/src/core/settings-manager.ts:1427`):
  1. User prompt images: `_normalizePromptImages` — failures become text hints, image dropped (`packages/coding-agent/src/core/agent-session.ts:1935-1955`).
  2. `read` tool images (`packages/coding-agent/src/core/tools/read.ts:138`, text note block + image block; non-vision model note `[Current model does not support images. The image will be omitted from this request.]` `:79-84`, `2f4f283cc`).
  3. **All tool results** (extensions, MCP, screenshot tools) via `normalizeToolResultImages` in `afterToolCall`, *after* the extension `tool_result` hook so injected images are normalized too (`packages/coding-agent/src/core/agent-session.ts:704-711`; `packages/coding-agent/src/utils/tool-result-images.ts:24-67`, `b0e05b442` #7330). Doc: "Oversized images make the provider reject the whole conversation, not just the offending turn, so normalize them once as they enter history." On processing failure the **original block is kept** (may be a missing image backend).
  4. codemode `image()` validates base64 + magic signature before accepting (`packages/codemode/src/runtime/prelude-source.ts:392-427`, `d2931ad3d` #10215).
- **Blocking**: `images.blockImages` (default false) — sdk `convertToLlm` wrapper replaces images with "Image reading is disabled." (deduped), checked per request; history keeps them (`packages/coding-agent/src/core/sdk.ts:296-331`; `1fc2a912d`).
- **Non-vision downgrade** at provider boundary: user images → "(image omitted: model does not support images)", tool-result images → "(tool image omitted: model does not support images)", consecutive collapsed (`packages/ai/src/api/transform-messages.ts:12-57`, `2f4f283cc` #3429) → [[cross-provider-handoff]].
- **Accounting**: 4800 chars/image in both estimators (`96f0edd02` #4983 counts user images); images dropped from summarizer input (text only) ([[pi--token-estimation]], [[pi--transcript-serialization-for-summary]]).
- Provider wire shape of tool-result images (Gemini nested `functionResponse.parts`, Responses `function_call_output`) handled in adapters → [[tool-result-image-routing]].

## Constants
| name | value | path:line |
|---|---|---|
| max dims | 2000 × 2000 | `packages/coding-agent/src/utils/image-resize-core.ts:35-36` |
| `DEFAULT_MAX_BYTES` | 4.5 MiB base64 | `packages/coding-agent/src/utils/image-resize-core.ts:32` |
| JPEG quality ladder | 80 → 85, 70, 55, 40 | `packages/coding-agent/src/utils/image-resize-core.ts:38`, `:132` |
| dimension shrink | ×0.75 per step | `packages/coding-agent/src/utils/image-resize-core.ts` (`Math.floor(w·0.75)`) |
| `IMAGE_TYPE_SNIFF_BYTES` | 4100 | `packages/coding-agent/src/utils/mime.ts:3` |
| `ESTIMATED_IMAGE_CHARS` | 4800 | `packages/coding-agent/src/core/compaction/compaction.ts:276` |

## Evolution
- 2025-11-12 `9e3e319f1` SVG + BMP support dropped from read.
- 2026-01-02 `4a32af253` automatic image resizing; 2026-01-03 `69dc6b078` (#424) more attempts to get under 5 MB (quality ladder); 2026-01-06 `1fc2a912d` `blockImages`.
- 2026-03-22 `a78882d8b` (#2055) enforce safe limits, no oversized fallback.
- 2026-04-20 `2f4f283cc` (#3429) non-vision placeholders.
- 2026-05-26 `96f0edd02` (#4983) user image tokens counted.
- 2026-06-25 `4cc339f58` (#6047) BMP via PNG conversion.
- 2026-08-03 `b0e05b442` (#7330) resize images returned by any tool.
- 2026-09-20 `f5c946480` (#9631) per-model image input limits; 2026-09-21 `47a18e37b` (#9755) full GIF signature.
- 2026-09-30 `d2931ad3d` (#10215) codemode `image()` validation; 2026-10-04 `d677d0ee7` codemode images to temp files.
- `021eae60a` (#10251) read `structuredContent` carries `{type:"image",data,mimeType,note}` for script callers.
- 2026-10-07 `b30a6dd77` (HEAD) ignore Node's own worker messages in image resize.

## Evidence commits
`9e3e319f1` `4a32af253` `69dc6b078` `1fc2a912d` `a78882d8b` `2f4f283cc` `96f0edd02` `4cc339f58` `b0e05b442` `f5c946480` `47a18e37b` `d2931ad3d` `d677d0ee7` `b30a6dd77` `021eae60a`

## Quirks
- Inconsistent failure policy by door: read/prompt drop the image with a text note; tool results keep the unprocessed original (could still poison history if oversized — inferred).
- SVG never supported; no OCR/text extraction fallback.
- Durable `read` refuses images ("unsupported_image" diagnostic, `packages/durable/src/tools/read.ts:114-131`); durable TUI has no images.

## Failures
[[image-content-poisoning]] (incl. node --watch worker-message drop, `b30a6dd77`) · [[tool-result-image-routing]] · cross-group: [[placeholder-text-misleads-model]]
