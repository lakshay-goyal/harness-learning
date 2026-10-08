---
type: implementation
harness: codex
concept: file-read-tool
commit: 622e9e3696
files: [codex-rs/core/src/tools/handlers/view_image_spec.rs:18-48, codex-rs/core/src/tools/handlers/view_image.rs:48-160, codex-rs/core/src/tools/spec_plan.rs:1358-1362, 14c35a16a8^:codex-rs/core/src/tools/handlers/read_file.rs:18-484, 14c35a16a8^:codex-rs/core/src/tools/spec.rs:2049-2170]
---
[[file-read-tool]] in [[codex]] — **partial**: today codex has only an *image* viewer (`view_image`); text files are read through `exec_command` (`cat`, `sed -n`, `rg`) and their intent is recovered by [[shell-command-intent-parsing]]. A dedicated `read_file` existed experimentally (2025-10 → 2026-03).

## Mechanism (current: `view_image`)
- Spec (`codex-rs/core/src/tools/handlers/view_image_spec.rs:18-48`): "View a local image file from the filesystem when visual inspection is needed. Use this for images already available on disk."; `path` "Local filesystem path to an image file."; `detail?` enum `high|original` "Image detail level. Defaults to `high`; use `original` to preserve exact resolution." (only when the model can request original detail); `environment_id?` for multi-environment turns; output schema with `image_url` for code mode.
- Path resolved against the selected environment's cwd (`PathUri` join), error "unable to resolve image path `{path}` against environment cwd `{cwd}`: {err}" (`codex-rs/core/src/tools/handlers/view_image.rs:145-156`) → [[path-normalization]], [[pluggable-tool-backends]].
- Errors: "view_image is not allowed because you do not support image inputs" / "unable to process image: invalid or unsupported image data" (`view_image.rs:54-56`); legacy `detail` values tolerated, others rejected with the allowed set (`:133-142`) → [[tool-argument-repair]].
- Image processing deferred to history insertion (`260261ed8f` 2026-08-11) → [[image-normalization]]; image detail preserved through app-server inputs (`8543e39885`).
- Gate: environment attached && `Feature::ViewImage` (Stable, on) (`codex-rs/core/src/tools/spec_plan.rs:1358-1362`); parallel-safe (`view_image.rs:81`).

## Removed design: `read_file` (experimental, per-model)
- `read_file(file_path, offset=1, limit=2000, mode="slice"|"indentation", indentation{anchor_line, max_levels=0, include_siblings=false, include_header=true, max_lines})` — "Reads a local file with 1-indexed line numbers, supporting slice and indentation-aware block modes."; output lines `L{n}: …`; `MAX_LINE_LENGTH = 500`, `TAB_WIDTH = 4`, `COMMENT_PREFIXES = ["#","//","--"]`; errors "file_path must be an absolute path", "offset must be a 1-indexed line number", "offset exceeds file length" (`14c35a16a8^:codex-rs/core/src/tools/handlers/read_file.rs:18-484`; spec `14c35a16a8^:codex-rs/core/src/tools/spec.rs:2049-2170`).
- Indentation mode = return the enclosing code block around an anchor line by indentation levels (+ doc comments/attributes above) — a structural read without a parser.
- Exposure was per model via `experimental_supported_tools` (`e0b38bd7a2` 2025-10-03); dropped for gpt-5-codex two days later (`f3b4a26f32` 2025-10-05); handler deleted (611 deletions) `14c35a16a8` 2026-03-25 "chore: remove read_file handler".

## Prompt-side reading rules
- Restored after the GPT-5 rewrite dropped them (`90d892f4fd`): prefer `rg`, "Read files in chunks with a max chunk size of 250 lines" — later the output-limit literals were removed (`570eb5fe78`, `26d0d822a2`), leaving "Do not use python scripts to attempt to output larger chunks of a file." (`codex-rs/protocol/src/prompts/base_instructions/default.md:265`) → [[prompt-states-stale-harness-limits]].
- Skills: "open and read its `SKILL.md` completely… If a read is truncated or paginated, continue until EOF." (`codex-rs/ext/skills/src/catalog_prompt.rs:28-30`; `56554904ba`) → [[partial-file-read-acted-on]].

## Constants
| name | value | path:line |
|---|---|---|
| `view_image.detail` values | `high` (default), `original` | `codex-rs/core/src/tools/handlers/view_image_spec.rs:25-27` |
| removed `read_file` default limit / max line length | 2000 lines / 500 chars | `14c35a16a8^:codex-rs/core/src/tools/handlers/read_file.rs:18,470-472` |

## Evolution
- 2025-10-03 `e0b38bd7a2` `beta_supported_tools` (per-model tool lists).
- 2025-10-05 `f3b4a26f32` "drop read-file for gpt-5-codex". 2025-10-09 `0026b12615` "indentation mode for read_file".
- 2026-03-25 `14c35a16a8` read_file handler removed.
- 2026-05-15 `8543e39885` preserve image detail. 2026-08-11 `260261ed8f` deferred view_image processing.

## Versus pi
pi's `read(path, offset?, limit?)` is a default tool: plain text, head-truncated 2000 lines/50 KB with "Use offset=N to continue", image sniffing + resize ([[pi--file-read-tool]]). codex ships no text reader; models read via shell, and [[no-file-read-write-tools]] records the decision. Axis: [[minimal-vs-rich-toolset]].
