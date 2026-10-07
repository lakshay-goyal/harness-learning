---
type: concept
stage: tools
tier: candidate
aliases: [read tool, read.ts, "offset/limit", bounded-line-scan-read, streaming-decode-parity, concurrent-writer-read-guard]
harnesses: [pi]
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

## Implementations
- [[pi--file-read-tool|pi]] — `read(path, offset?, limit?)`, plain text, head-truncated 2000 lines/50KB with "Use offset=N to continue", magic-number image sniffing + resize, macOS path variants; durable bounded scan + writer guard.

## Failures
- [[partial-file-read-acted-on]]
- [[read-path-unicode-variants]]
- [[read-fails-on-growing-file]]
- related: [[image-content-poisoning]]

## Related
[[tool-output-truncation]] · [[image-normalization]] · [[path-normalization]] · [[tool-description-design]] · [[pluggable-tool-backends]] · [[search-replace-edit]]
