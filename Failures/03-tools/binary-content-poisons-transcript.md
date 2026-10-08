---
type: failure
concepts: [file-read-tool]
harnesses: [opencode]
---
**Symptom** — Reading a binary file dumped NUL bytes and garbage into the transcript; the session became unusable (provider rejections or a derailed model).

**Root cause** — The read tool decoded every file as text; MIME-from-extension misclassified files in both directions.

**Fix · [[opencode]]**
- `1b3d58e791` 2025-07-30 "prevent read tool from opening binary files and corrupting session (#1425)": NUL check on the first bytes.
- `ebd1b18b70` 2025-08-17 extension denylist + >30% non-printable ratio on a 4096-byte sample (`packages/opencode/src/tool/read.ts:182-226`).
- `38c641a2fc` 2026-01-19 `.fbs` treated as text, not image; `8ebdbe0ea2` 2026-02-19 text files misclassified as binary.
- `83dca45dd5` 2026-06-05 v2 reads media-aware and binary-safe, 20 MiB media cap (`packages/core/src/tool/read-filesystem.ts:13`).

**Lesson** — Sniff bytes before ingesting; extension-based MIME is wrong both ways, so keep a text allow-list as well as a binary deny-list.

Related: [[file-read-tool]] · [[no-binary-detection-in-read]] · [[bash-output-integrity]] · [[image-normalization]] · [[opencode--file-read-tool|opencode]]
