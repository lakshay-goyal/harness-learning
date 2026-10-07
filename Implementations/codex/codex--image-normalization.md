---
type: implementation
harness: codex
concept: image-normalization
commit: 622e9e3696
files: [codex-rs/core/src/session/mod.rs:3368, codex-rs/core/src/session/mod.rs:3464, codex-rs/core/src/image_preparation.rs:31, codex-rs/core/src/image_preparation.rs:122, codex-rs/core/src/image_preparation.rs:350, codex-rs/core/src/image_preparation.rs:393, codex-rs/utils/image/src/lib.rs:24, codex-rs/utils/image/src/lib.rs:73, codex-rs/core/src/context/image_resize_notice.rs:37, codex-rs/core/src/context_manager/normalize.rs:330, codex-rs/core/src/context/unsupported_media.rs:11, codex-rs/core/src/session/turn.rs:793]
---
[[image-normalization]] in [[codex]].

## Mechanism
- **One door**: images in user messages and function/custom tool outputs are prepared centrally before insertion into history (`record_conversation_items` → `prepare_conversation_items_for_history`, `codex-rs/core/src/session/mod.rs:3368`, `:3464-3475`; loop `codex-rs/core/src/image_preparation.rs:122-215`).
- **Rejections → model-visible placeholders**: remote http(s) URLs → "image content omitted because remote image URLs are not supported"; detail `low` rejected; processing failure / too large → placeholders (`codex-rs/core/src/image_preparation.rs:31-37`, `:338-342`, `:404-406`).
- **Resize limits** (`codex-rs/utils/image/src/lib.rs:24-31`, `:73-84`): high/auto detail → max 2048 px and 2,500 patches; original detail → 6000 px and 10,000 patches (32-px patches). Feature `UnifiedImageBudget` forces one 6000 px / 10k-patch policy for models that support original detail and rewrites detail to `original` "to preserve accurate context-window accounting" (`codex-rs/core/src/image_preparation.rs:45-50`, `:393-424`). Sanity cap 1 GiB input; 64 MiB prompt image cache (`codex-rs/utils/image/src/lib.rs:30-32`).
- **Attachment store**: prepared images may be uploaded and replaced by `ImageReference::File{file_id}`; upload failure falls back to an inline data URL (`codex-rs/core/src/image_preparation.rs:350-385`).
- **Resize notice** (feature `image_resize_notice`): developer message `<image_resize_notice>` after the message/tool output — "Image i of n in the preceding user message was resized from WxH to wxh pixels." (`codex-rs/core/src/context/image_resize_notice.rs:37-75`); grouped with its source through remote compaction (`codex-rs/core/src/compact_remote_history.rs:55-66`).
- **Receiving-model projection**: `for_prompt` strips images/audio for models without that input modality, substituting "image content omitted because you do not support image input" (`codex-rs/core/src/context_manager/normalize.rs:330-380`; `codex-rs/core/src/context/unsupported_media.rs:11-19`); request copies downgrade `detail: original` → `high` for models that don't support it (`6eecd04fc1`).
- **Provider rejects an image**: turn emits "Invalid image in your last message. Please remove it and try again." and ends; no automatic removal (`codex-rs/core/src/session/turn.rs:793-815`).
- **Token cost**: 7,373 bytes per resized image or patch count for original detail → [[codex--token-estimation]]; remote compaction charges retained images atomically → [[codex--compaction-cut-point]].

## Constants
| name | value | path:line |
|---|---|---|
| `MAX_DIMENSION` (high/auto) | 2048 px | `codex-rs/utils/image/src/lib.rs:26` |
| high-detail max patches | 2_500 | `codex-rs/utils/image/src/lib.rs:77` |
| original detail | 6000 px / 10_000 patches | `codex-rs/utils/image/src/lib.rs:80-83` |
| `MAX_IMAGE_CACHE_BYTES` | 64 MiB | `codex-rs/utils/image/src/lib.rs:32` |
| input sanity cap | 1 GiB | `codex-rs/utils/image/src/lib.rs:30-32` |

## Evolution
- 2026-02-10 `5e01450963` "Strip unsupported images from prompt history to guard against model switch (#11349)" → [[image-content-poisoning]].
- 2026-04-17 `120bbf46c1` high-detail 2048 px / 2,500 patches.
- 2026-06-22 `7153affa0f` "[codex] replace remote images with model-visible error text (#29417)" ("The HTTP image url pathway … is slow and not recommended").
- 2026-07-20 `8431dc590a` (#34380) removed "replace bad image with 'Invalid image' text and retry"; now a hard bad-request error.
- 2026-08-04 `4bd5b9fd09` "Keep image resize notices attached during remote compaction (#36956)"; 2026-08-05 `fa5d5ae047` (#37134) resize notice.
- 2026-08-06 `0a0ebb8535` original detail 6000 px / 10k patches.
- 2026-08-23 `6677fd827d` retained images budgeted in remote compaction.
- 2026-09-09 `6eecd04fc1` "Normalize image detail for the receiving model" → [[model-switch-replays-unsupported-content]].

## Versus pi
- [[pi--image-normalization]]: byte/dimension caps tuned for Anthropic (2000 px / 4.5 MB, JPEG quality ladder), door-by-door normalization, `[Image omitted: …]` placeholders. codex: OpenAI patch-based budgets, one central door, attachment upload, separate resize-notice message, and no auto-repair of provider-rejected images.
