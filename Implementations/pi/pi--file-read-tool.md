---
type: implementation
harness: pi
concept: file-read-tool
commit: b30a6dd77
files: [packages/coding-agent/src/core/tools/read.ts:14, packages/coding-agent/src/core/tools/read.ts:96, packages/coding-agent/src/core/tools/read.ts:110, packages/coding-agent/src/core/tools/read.ts:153, packages/coding-agent/src/utils/mime.ts:3, packages/coding-agent/src/utils/image-process.ts:72, packages/coding-agent/src/core/tools/path-utils.ts:86, packages/durable/src/tools/read.ts:39, packages/durable/src/tools/read.ts:84]
---
[[file-read-tool]] in [[pi]].

`read` is default-active ([[pi--minimal-default-toolset|minimal-default-toolset]]): plain text (no line numbers) with 1-indexed `offset`/`limit` paging and head truncation, plus image attachments. Abbrev `packages/coding-agent/src/` = `packages/coding-agent/src/`.

## Mechanism

### Schema / text
- **Schema** (`packages/coding-agent/src/core/tools/read.ts:14-18`): `path` — "Path to the file to read (relative or absolute)"; `offset?` — "Line number to start reading from (1-indexed)"; `limit?` — "Maximum number of lines to read".
- **Description** (`read.ts:96`): "Read the contents of a file. Supports text files and images (jpg, png, gif, webp, bmp). Images are sent as attachments. For text files, output is truncated to 2000 lines or 50KB (whichever is hit first). Use offset/limit for large files. When you need the full file, continue with offset until complete." (limits interpolated from `DEFAULT_MAX_LINES/BYTES`, `truncate.ts:11-12`; see [[pi--tool-description-design|tool-description-design]]).
- **Snippet** "Read file contents"; **guideline** "Use read to examine files instead of cat or sed." (`read.ts:20-23`).
- `constrainedSampling: {type:"json_schema", strict:"prefer"}` (`read.ts:101`).
- **outputSchema** (`read.ts:33-38`): `string | {type:"image", data, mimeType, note}` — for codemode callers; `structuredContent = toReadOutput(content)` (`read.ts:72-77,216`; `021eae60a` #10251: codemode `read` on images resolves to image blocks). See [[structured-tool-output]].

### Execution (`read.ts:107-217`)
1. Abort: pre-aborted → reject "Operation aborted"; listener rejects immediately, but the in-flight fs op continues (`read.ts:111-121`) — acceptable because read doesn't mutate (contrast edit/write queue discipline).
2. `resolveReadPathAsync(path, ctx?.cwd || cwd)` (`read.ts:124`) — resolved path then macOS screenshot variants (narrow NBSP before AM/PM, NFD, curly apostrophe, NFD+curly) — see [[pi--path-normalization|path-normalization]].
3. `ops.access`; `ops.detectImageMimeType` (`read.ts:127-129`).
4. **Text path** (`read.ts:153-205`):
   - whole file read into memory, `toString("utf-8")`, `split("\n")`.
   - `startLine = offset ? max(0, offset-1) : 0`; beyond EOF → throw `Offset N is beyond end of file (M lines total)` (`:163-165`).
   - explicit `limit` applied first, then `truncateHead` ([[tool-output-truncation]]; head direction = keep first N complete lines, never partial).
   - first line > 50KB: `[Line N is X, exceeds 50.0KB limit. Use bash: sed -n 'Np' path | head -c 51200]` — line content **not** shown (`:179-183`).
   - truncated by lines: `[Showing lines a-b of T. Use offset=N to continue.]`; by bytes: `[Showing lines a-b of T (50.0KB limit). Use offset=N to continue.]` (`:184-194`).
   - user limit stopped early: `[K more lines in file. Use offset=N to continue.]` (`:195-199`).
   - otherwise plain content. `details = {truncation}` when truncated.
5. **Image path** (`read.ts:133-151`): `processImage(buffer, mime, {autoResizeImages, resizeOptions: ctx.model.inputLimits.images.resize ?? fallback})` → content = text note `Read image file [mime]` + hints (+ non-vision note) + image block; on failure text-only note with message. Non-vision model: `[Current model does not support images. The image will be omitted from this request.]` (`read.ts:79-84`; `2f4f283cc` #3429).
   - MIME sniffed from first `IMAGE_TYPE_SNIFF_BYTES = 4100` bytes by magic (`packages/coding-agent/src/utils/mime.ts:3-23`): JPEG (rejects `FF D8 FF F7` JPEG-LS), PNG (rejects animated `acTL`), GIF87a/89a (full signature, `47a18e37b` #9755 — text files starting "GIF" were treated as images), WEBP (RIFF….WEBP), BMP (validated header).
   - `processImage` (`packages/coding-agent/src/utils/image-process.ts:72-119`): BMP → PNG with hint `[Image converted from X to Y.]` (`:69`); conversion failure `[Image omitted: could not be converted to a supported inline image format.]` (`:82`); resize failure `[Image omitted: could not be resized below the inline image size limit.]` (`:91`; `a78882d8b` #2055 — previously fell back to the oversized original).
   - Resize via Photon (Rust/WASM) in a worker thread with in-process fallback (`packages/coding-agent/src/utils/image-resize.ts:84-109`); defaults 2000×2000, 4.5 MB base64 ("headroom below Anthropic's 5MB limit"), JPEG quality 80 then {85,70,55,40}, EXIF orientation, then ×0.75 shrink to 1×1 (`image-resize-core.ts:31-39,63-166`); coordinate hint `[Image: original WxH, displayed at wxh. Multiply coordinates by S to map to original image.]` (`image-resize.ts:115-122`). Worker replies tagged `pi:image-resize-response`, Node `--watch` messages ignored (`b30a6dd77`, HEAD).
   - Full pipeline (incl. images from any tool, `blockImages`) is [[image-normalization]] (05-context).
6. Result `.then(... structuredContent)` (`read.ts:216`).

### Deliberate design / absences
- **No line numbers** in output (since `c7a73d4f8` "Plain text output (no line numbers)") — edit uses exact text, not line anchors.
- **No binary detection** for non-image files: any non-image decoded as UTF-8 and returned; no NUL sniff, no size cap before reading (`read.ts:153-157`) → [[no-binary-detection-in-read]].
- **No read-before-edit tracking / staleness**; nothing records reads.
- Reads whole file into memory (no streaming) in coding-agent; OOM reports not searched (open question).
- No cwd confinement ([[no-cwd-confinement]]).
- Pluggable `ReadOperations {readFile, access, detectImageMimeType?}` ([[pluggable-tool-backends]]).

## Constants
| name | value | path:line |
|---|---|---|
| `DEFAULT_MAX_LINES` | 2000 | `packages/coding-agent/src/core/tools/truncate.ts:11` |
| `DEFAULT_MAX_BYTES` | 50 × 1024 | `truncate.ts:12` |
| bash fallback hint | `head -c 51200` | `packages/coding-agent/src/core/tools/read.ts:182` |
| `IMAGE_TYPE_SNIFF_BYTES` | 4100 | `packages/coding-agent/src/utils/mime.ts:3` |
| image max dimension | 2000 × 2000 | `packages/coding-agent/src/utils/image-resize-core.ts:35-36` |
| image max encoded bytes | 4.5 MiB | `image-resize-core.ts:32` |
| JPEG quality ladder | 80; 85,70,55,40 | `image-resize-core.ts:38,132` |
| durable `READ_CHUNK` | 64 KiB | `packages/durable/src/tools/read.ts:39` |

## Evolution
- 2025-10-17 `ffc9be886`: "Read the contents of a file. Returns the full file content as text."
- 2025-11-12 `84dcab219`: image support (jpg, png, gif, webp, bmp, svg) "sent as attachments to the model".
- 2025-11-12 `c7a73d4f8`: line limits — "defaults to first 2000 lines. Use offset/limit for large files."; max line length 2000 chars (later replaced); plain text, no line numbers.
- 2025-11-12 `9e3e319f1`: SVG + BMP dropped from list (provider-unsupported — inferred).
- 2025-12-07 `de77cd141` / `b813a8b92` (#134): byte+line head truncation, actionable notices model-visible; first-line-too-long → bash `sed` hint; limits interpolated.
- 2025-12-13 `9a7bbb283` (#181): macOS screenshot unicode spaces. 2025-12-17 `d70edf571` (#205): MIME via file-type (later own sniffer).
- 2026-01-02/03 `4a32af253`, `69dc6b078` (#424): auto-resize, progressive JPEG quality.
- 2026-01-06 `1fc2a912d`: `blockImages` setting.
- 2026-01-16 `012319e15`: `pi-internal://` scheme resolving to pi package dir (read pi docs) → **removed** next day `4068bc556` (2026-01-17), replaced by absolute doc paths in the `<docs>` system-prompt section (`packages/coding-agent/src/core/system-prompt.ts:162-170`; [[self-documentation-pointer]]).
- 2026-01-24 `89636cfe6`: "When you need the full file, continue with offset until complete." — models stopped after the first chunk → [[partial-file-read-acted-on]].
- 2026-01-29 `4edb506df` (#1078): NFD + curly-quote path variants.
- 2026-03-22 `235b247f1`: guideline "Use read to examine files instead of cat or sed." (was "Use read to examine files before editing. You must use this tool instead of cat or sed." from `42d7d9d9b`); `a78882d8b` (#2055) no oversized fallback.
- 2026-04-15 `d22c120b8` (#3194): lowercase am/pm. 2026-04-20 `2f4f283cc` (#3429): non-vision note.
- 2026-06-25 `4cc339f58` (#6047): BMP re-added via PNG conversion.
- 2026-08-03 `b0e05b442` (#7330): images from any tool normalized at history entry.
- 2026-08-06 `9ab91fb93` (#7671): `readToolSystemPromptContribution` exported (reused by durable prompt).
- 2026-09-01 `62835ea81` (#8627): `ctx.cwd`. 2026-09-05 `fcff255b0`: strict-prefer sampling.
- 2026-09-20 `f5c946480` (#9631): per-model resize profile `model.inputLimits.images.resize`. 2026-09-21 `47a18e37b` (#9755): full GIF signature.
- 2026-10-05 `021eae60a` (#10251): image `structuredContent`. 2026-10-07 `b30a6dd77` (HEAD): ignore Node's own worker messages in image resize (hang under `node --watch`).
- Durable: 2026-09-29 `445770e03` (Package 16) read tool; 2026-10-04 `a19c09d9b` bounded read via `BinaryReader.scanLines`; 2026-10-05 `cd60a5b99` growing-file reads.

## Evidence commits
`ffc9be886` `84dcab219` `c7a73d4f8` `9e3e319f1` `de77cd141` `b813a8b92` `9a7bbb283` `d70edf571` `4a32af253` `69dc6b078` `1fc2a912d` `012319e15` `4068bc556` `89636cfe6` `4edb506df` `42d7d9d9b` `235b247f1` `a78882d8b` `d22c120b8` `2f4f283cc` `4cc339f58` `b0e05b442` `9ab91fb93` `62835ea81` `fcff255b0` `f5c946480` `47a18e37b` `021eae60a` `b30a6dd77` `445770e03` `a19c09d9b` `cd60a5b99`

## Quirks
- `totalFileLines` counts the empty string after a final `\n`, so a 10-line file with trailing newline reports 11 lines (from `split("\n")`, `read.ts:155-156`) (observed from code).
- `offset: 0` silently treated as 1 (`offset ? … : 0`).
- First-line-too-long result shows **no** content, only a sed hint (durable shows first 50KB instead).
- Abort resolves the promise early while the fs read keeps running.
- Line-too-long hint uses `head -c 51200` bytes, may cut a UTF-8 char.

## Durable variant (packages/durable)
- `packages/durable/src/tools/read.ts`: same schema text (`:26-30`); description "Read the contents of a **text** file. Output is truncated to 2000 lines or 50KB (whichever is hit first). Use offset/limit for large files. When you need the full file, continue with offset until complete." (`:76`).
- **Images refused**: `isError`, diagnostic `unsupported_image` "{path} is an image ({mime}); reading images is not supported" (`:114-131`).
- **Bounded read**: opens a `BinaryReader` via `ExecutionEnv` ([[pluggable-tool-backends]]); one `scanLines` pass counts lines and locates the selection; then decodes only the head (stop at > 50 KiB+1 bytes or 2000 newlines, 64 KiB `READ_CHUNK` reads, `rangeDecoder` ignoreBOM) (`:39-70,133-174`). Same algorithm ported to Rust daemon (`packages/env` `scan.rs`) so a remote read transfers only scan result + head ([[remote-execution-env]]). Contract: result equals decoding whole file + `split("\n")` + `truncateHead` (`:102-105`).
- **Concurrent-writer guard**: compare `reader.info()` before/after; growth (`after.size > before.size`) or unchanged size+mtime accepted; shrink/rewrite → re-read once; second change → "{path} changed while it was read" (`:84-94`; growing-log case fixed `cd60a5b99` → [[read-fails-on-growing-file]]).
- Over-long first line: shows its **first 50KB** cut on a character boundary + warn diagnostic `Line N is X, exceeds the 50.0KB limit; showing its first Y. Use bash: sed -n 'Np' path | tail -c +K` (`:178-191`).
- Truncation/continuation remarks are **diagnostics**, never content (`:192-209`) → rendered in `<harness>` block ([[harness-diagnostics-channel]]).
- Path: unicode spaces + `@` normalized, screenshot variants (`packages/durable/src/tools/path-utils.ts:4-30`).
- **Differential test** `packages/durable/test/tools-read-differential.test.ts`: old whole-file implementation kept as `referenceRead` oracle (`:13-16`); 400 seeded random files × 4 random (offset, limit) trials must match (`:155-178`); inputs incl. BOM, CRLF, invalid UTF-8, split multibyte, lines > 50KB, offsets NaN/-4/2.5/1e20 (`:100-130`).
- No `replay` declared → `unsafe` default even though side-effect free (open question; [[crash-safe-tool-replay]]).

## Failures
- [[read-path-unicode-variants]]
- [[partial-file-read-acted-on]]
- [[read-fails-on-growing-file]]
- [[image-content-poisoning]] (05-context)
