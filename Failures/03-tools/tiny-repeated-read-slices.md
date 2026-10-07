---
type: failure
concepts: [file-read-tool]
harnesses: [opencode]
---
**Symptom** — The model paged files in ~30-line chunks, spending many calls on one file.

**Root cause** — Description gave no guidance on window size (it had said "it's recommended to read the whole file", which some models ignored); small slices felt cheaper.

**Fix · [[opencode]]** — `6b4d617df0` 2026-02-11 read description: "Avoid tiny repeated slices (30 line chunks)"; the whole-file recommendation removed in the same commit (`packages/opencode/src/tool/read.txt:13`). beast.txt goes further: "Always read 2000 lines of code at a time" (`packages/opencode/src/session/prompt/beast.txt:83`).

**Lesson** — Name the inefficient pattern with a concrete number.

Related: [[file-read-tool]] · [[partial-file-read-acted-on]] · [[opencode--file-read-tool|opencode]]
