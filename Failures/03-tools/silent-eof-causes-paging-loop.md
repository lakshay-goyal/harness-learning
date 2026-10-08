---
type: failure
concepts: [file-read-tool]
harnesses: [opencode]
---
**Symptom** — When a read reached the end of a file the output said nothing about it, so the model kept paging with larger offsets ("infinite loops").

**Root cause** — Absence of a continuation hint was the only end-of-file signal; models do not infer termination from silence.

**Fix · [[opencode]]** — `7ec32f834e` 2025-11-13 "improve read tool end-of-file detection to prevent infinite loops": output ends with `(End of file - total N lines)`; partial windows end with "(Showing lines a-b of N. Use offset=N to continue.)" (`packages/opencode/src/tool/read.ts:345-349`).

**Lesson** — Paginated tools must state termination positively, with the total.

Related: [[file-read-tool]] · [[harness-diagnostics-channel]] · [[partial-file-read-acted-on]] · [[opencode--file-read-tool|opencode]]
