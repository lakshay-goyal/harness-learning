---
type: concept
stage: tools
tier: must-have
aliases: [read tool, read.ts, "offset/limit", bounded-line-scan-read, streaming-decode-parity, concurrent-writer-read-guard, isBinaryFile, "End of file - total N lines", "<entries>", view_image, read_file indentation mode]
harnesses: [pi, opencode, codex]
---
The model's file reader: line-window paging (offset/limit) with a size cap and continuation notice, image detection and attachment, encoding handling, and (optionally) bounded-memory scanning and consistency checks against concurrent writers.

## Why
- Large files blow the context window; without an explicit continuation protocol the model acts on the first chunk only ([[partial-file-read-acted-on]]).
- Images must be sniffed, resized, and capped before entering history or the provider rejects the whole conversation ([[image-content-poisoning]]).
- Reading a file that is being appended/rewritten mid-read yields inconsistent content ([[read-fails-on-growing-file]]).
- User/model-typed paths differ from on-disk names by invisible characters ([[read-path-unicode-variants]]).

## Design space
- Line numbers in output (cat -n style) vs **plain text** (pi, since 2025-11).
- Truncation by lines AND bytes, whichever first, head direction (pi 2000/50KB) → [[tool-output-truncation]].
- Over-long single line: show nothing + `sed` hint (pi coding-agent) vs first 50KB + hint (pi durable).
- Whole-file read into memory then slice (pi coding-agent) vs single streaming scan that only decodes the shown head (pi durable `scanLines`).
- Binary detection (NUL sniff) vs decode everything as UTF-8 (pi; [[no-binary-detection-in-read]]).
- Images: attach (pi coding-agent) vs refuse (pi durable) → [[image-normalization]].
- Read tracking for read-before-edit enforcement (absent in pi).
- Concurrent-writer guard: accept growth, re-read once on rewrite (pi durable).
- Special URI schemes for harness docs (pi tried `pi-internal://` for one day → absolute doc paths in prompt, [[self-documentation-pointer]]).
- **No text reader; read via shell** (`cat`, `sed -n`, `rg`) with intent recovered by a parser (✔ codex) → [[minimal-vs-rich-toolset]], [[shell-command-intent-parsing]].
- Image-only viewer tool (`view_image(path, detail?)`, resolved against the selected environment) (✔ codex).
- Structural read: return the enclosing block by indentation levels around an anchor line (codex experimental `read_file` indentation mode `0026b12615`, removed `14c35a16a8` 2026-03-25).
- Explicit end-of-file marker with total line count (opencode) vs silence.
- Directory listing inside the read tool, replacing ls (opencode).
- Binary detection: extension denylist + NUL / >30% non-printable sniff on 4 KiB (opencode).
- Side effects of reading: attach nested instruction files, warm the language server (opencode) → [[context-file-hierarchy]], [[lsp-diagnostics-feedback]].

## Implementations
- [[pi--file-read-tool|pi]] — `read(path, offset?, limit?)`, plain text, head-truncated 2000 lines/50KB with "Use offset=N to continue", magic-number image sniffing + resize, macOS path variants; durable bounded scan + writer guard.
- [[codex--file-read-tool|codex]] — partial: `view_image` only; experimental `read_file` (slice/indentation, `L{n}:` lines, 2000-line default) existed 2025-10 → 2026-03.
- [[opencode--file-read-tool|opencode]] — 2000 lines / 2000 chars / 50 KB, `N: ` prefixes, 1-indexed offset, explicit EOF line, directories as `<entries>`, extension + byte-sample binary check, images/PDF attached, nested AGENTS.md appended as `<system-reminder>`.

## Failures
- [[shell-cat-instead-of-read-tool]]
- [[partial-file-read-acted-on]]
- [[read-path-unicode-variants]]
- [[read-fails-on-growing-file]]
- related: [[image-content-poisoning]]
- [[shell-cat-instead-of-read-tool]] (04-prompting) — Model read files with cat/sed via bash instead of the read tool — losing read's truncation/paging contract,…
- [[silent-eof-causes-paging-loop]]
- [[binary-content-poisons-transcript]]
- [[read-offset-index-mismatch]]
- [[tiny-repeated-read-slices]]

## Related
[[tool-output-truncation]] · [[image-normalization]] · [[path-normalization]] · [[tool-description-design]] · [[pluggable-tool-backends]] · [[search-replace-edit]] · [[shell-command-intent-parsing]] · [[minimal-vs-rich-toolset]] · [[no-file-read-write-tools]]
