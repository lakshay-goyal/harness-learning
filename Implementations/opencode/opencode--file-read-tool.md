---
type: implementation
harness: opencode
concept: file-read-tool
commit: ecc4916b5a
files: [packages/opencode/src/tool/read.ts:13-19, packages/opencode/src/tool/read.ts:94, packages/opencode/src/tool/read.ts:119, packages/opencode/src/tool/read.ts:182-226, packages/opencode/src/tool/read.ts:235-236, packages/opencode/src/tool/read.ts:264-292, packages/opencode/src/tool/read.ts:300-328, packages/opencode/src/tool/read.ts:339-356, packages/opencode/src/session/instruction.ts:179-221, packages/core/src/tool/read-filesystem.ts:11-14]
---
[[file-read-tool]] in [[opencode]].

## Mechanism
### Legacy runtime
- Paths: relative → `path.resolve(instance.directory, …)` (`packages/opencode/src/tool/read.ts:235-236`); missing file → "File not found … Did you mean one of these?" with siblings (`:94`).
- Window: 2000 lines default, 2000 chars per line + "... (line truncated to 2000 chars)", 50 KB total (`read.ts:13-16`).
- Output lines `N: content`, `offset` 1-indexed (`read.ts:339`).
- Terminators stated positively: "(Output capped at 50 KB. Showing lines a-b. Use offset=N to continue.)", "(Showing lines a-b of N. Use offset=N to continue.)", "(End of file - total N lines)" (`read.ts:345-349`) → [[silent-eof-causes-paging-loop]].
- Directories: `<entries>` listing with trailing `/`, same offset/limit paging, "(Showing X of Y entries…)" (`read.ts:264-292`); replaced the `list` tool.
- Binary: extension denylist (zip, exe, wasm, pyc, docx, …) then a 4096-byte sample: any NUL or >30% non-printable → binary error (`read.ts:182-226,327`) → [[binary-content-poisons-transcript]].
- Images (jpeg/png/gif/webp) and PDF returned as `data:` attachments; MIME sniffed from the same sample (`read.ts:19,300-321`) → [[image-normalization]].
- Nested instructions: `Instruction.resolve` walks from the file's dir to the project root and appends unseen AGENTS.md/CLAUDE.md as `<system-reminder>` (`read.ts:355-356`; `packages/opencode/src/session/instruction.ts:179-221`) → [[context-file-hierarchy]], [[ephemeral-reminder-injection]].
- Background LSP warm-up `lsp.touchFile` (`read.ts:119`) → [[lsp-diagnostics-feedback]].
- External paths go through `external_directory` permission → [[workspace-boundary-check]]; `*.env` reads ask by default → [[permission-ruleset]].
### v2 runtime
- `MAX_READ_LINES = 2_000`, `MAX_READ_BYTES = 50 KiB`, `MAX_MEDIA_INGEST_BYTES = 20 MiB`, `MAX_LINE_LENGTH = 2_000` (`packages/core/src/tool/read-filesystem.ts:11-14`); rejects absolute paths, escapes and symlink escapes (`specs/v2/session.md:197`).

## Constants
| name | value | path:line |
|---|---|---|
| `DEFAULT_READ_LIMIT` | 2000 lines | `packages/opencode/src/tool/read.ts:13` |
| `MAX_LINE_LENGTH` | 2000 chars | `packages/opencode/src/tool/read.ts:14` |
| `MAX_BYTES` | 50 KB | `packages/opencode/src/tool/read.ts:16` |
| `SAMPLE_BYTES` | 4096 | `packages/opencode/src/tool/read.ts:18` |
| non-printable ratio | > 0.3 → binary | `packages/opencode/src/tool/read.ts:226` |
| v2 `MAX_MEDIA_INGEST_BYTES` | 20 MiB | `packages/core/src/tool/read-filesystem.ts:13` |

## Evolution
- 2025-07-30 `1b3d58e791` refuse binary files ("corrupting session"); 2025-08-17 `ebd1b18b70` extension list + printable ratio; 2026-01-19 `38c641a2fc` `.fbs` not an image; 2026-02-19 `8ebdbe0ea2` text files misclassified as binary.
- 2025-08-13 `7d54f893c9` description stops claiming image support; 2025-10-09 `225adc46ba` real image support.
- 2025-11-13 `7ec32f834e` explicit end-of-file marker "to prevent infinite loops".
- 2025-12-25 `f397c92ddf` `list` tool removed; 2026-02-11 `6b4d617df0` read handles directories, description "Avoid tiny repeated slices (30 line chunks)" → [[tiny-repeated-read-slices]].
- 2026-02-11 `006d673ed2` `offset` 1-indexed → [[read-offset-index-mismatch]].
- 2026-01-26 `39a73d4894` nested AGENTS.md resolution on read.
- 2026-06-05 `83dca45dd5` v2 reads media-aware and binary-safe.

## Quirks / drift
- The `list` permission key survives on the explore agent after the tool's removal (`packages/opencode/src/agent/agent.ts:204`).

Contrast: pi has no binary detection and plain-text output without line numbers → [[pi--file-read-tool|pi]], [[no-binary-detection-in-read]].
