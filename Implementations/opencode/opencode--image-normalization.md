---
type: implementation
harness: opencode
concept: image-normalization
commit: ecc4916b5a
files: [packages/opencode/src/image/image.ts:10-14, packages/opencode/src/image/image.ts:75-130, packages/opencode/src/session/processor.ts:390-412, packages/opencode/src/session/message-v2.ts:46, packages/opencode/src/session/message-v2.ts:138-170, packages/opencode/src/session/message-v2.ts:300-312]
---
[[image-normalization]] in [[opencode]].

## Mechanism

### Legacy runtime
- `Image.normalize` (`packages/opencode/src/image/image.ts:75-130`): decode base64 with Photon (WASM), downscale to fit `attachment.image` config or defaults (2000×2000, 5 MB base64), Lanczos3 resize, JPEG quality ladder `[80, 85, 70, 55, 40]` (`:10-14`, `:111-127`). Typed errors: `ResizerUnavailableError`, `InvalidDataUrlError`, `DecodeError`, `SizeError`.
- Applied to tool-result attachments at completion time: un-resizable images are dropped and the output gets "[N images omitted: could not be resized below the image size limit.]"; resizer-unavailable keeps the original (`packages/opencode/src/session/processor.ts:390-412`).
- **Tool-result media routing per SDK** `supportsMediaInToolResult` (`packages/opencode/src/session/message-v2.ts:147-163`): inline for `@ai-sdk/anthropic`, `@ai-sdk/openai`, Bedrock mantle, Bedrock images only for Claude/Nova/Llama 4, xAI images, Vertex-Anthropic, Gemini 3; otherwise media is hoisted into a synthetic user message after the assistant turn prefixed `SYNTHETIC_ATTACHMENT_PROMPT = "Attached media from tool result:"` (`:46`).
- xAI non-png/jpeg/webp images dropped because xAI fails the whole request on them (`:165-169`).
- Pruned (`time.compacted`) tool parts drop their attachments too (`:306-309`) → [[opencode--tool-output-pruning]].

### v2 runtime
- Normalization lives in the native protocols of `packages/llm`; tool-result media emitted as structured blocks inside `tool_result` (Anthropic `9db90a0b76`, OpenAI Responses `700d012025`, both 2026-05-22); OpenAI Chat moves tool images to a following user message because the `tool` role is text-only (`pendingImages`, `packages/llm/src/protocols/openai-chat.ts:297-311`).
- `10d1e04e9b` 2026-06-06 isolate image normalization (shared state across requests).

## Constants
| name | value | path:line |
|---|---|---|
| `MAX_BASE64_BYTES` | 5 × 1024 × 1024 | `packages/opencode/src/image/image.ts:10` |
| `MAX_WIDTH` / `MAX_HEIGHT` | 2000 / 2000 | `packages/opencode/src/image/image.ts:11-12` |
| `AUTO_RESIZE` | true | `packages/opencode/src/image/image.ts:13` |
| `JPEG_QUALITIES` | [80, 85, 70, 55, 40] | `packages/opencode/src/image/image.ts:14` |

## Evolution
- 2025-12-15 `9eefcd1b41` empty image file crashed Anthropic → text error → [[image-content-poisoning]].
- 2026-01-16 `de2de099b4` stop hoisting, reverted same day `f5a6a4af7f`; redone 2026-01-21 `c2844697f3` → [[tool-result-image-routing]].
- 2026-02-05 `72de9fe7a6` Kimi via OpenAI-compatible; 2026-04-12 `8b9b9ad31e` hoisted images counted as user-initiated Copilot premium requests.
- 2026-05-01 `563177c6ac` image + empty text tool result → API error.
- 2026-05-10 `85ce6a5f95` auto resize & max size constraints.
- 2026-09-20 `c10134729d` Bedrock hoists except Claude/Nova/Llama 4; 2026-10-06 `83802d800e` xAI SDK bump so tool-result images reach xAI.

## Quirks / drift
- The 5 MB cap equals Anthropic's hard limit with no headroom for base64 framing (pi uses 4.5 MB).
- The allow-list is by npm package and id substring, the same pattern that missed Bedrock ARNs for caching ([[capability-sniffing-misses-opaque-ids]]).

Contrast: [[pi--image-normalization|pi]] normalizes at every door with a 4.5 MB cap and magic-byte sniffing; opencode resizes at tool completion and spends most of its fixes on per-provider routing of tool-result media.
