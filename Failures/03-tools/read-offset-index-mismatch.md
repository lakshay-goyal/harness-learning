---
type: failure
concepts: [file-read-tool]
harnesses: [opencode]
---
**Symptom** — The read tool displayed 1-based line numbers but its `offset` parameter was 0-based, so follow-up reads were off by one.

**Root cause** — Parameter indexing differed from the indexing the output showed.

**Fix · [[opencode]]** — `006d673ed2` 2026-02-11 "make read tool offset 1 indexed instead of 0 to avoid confusion"; output lines `N: content` (`packages/opencode/src/tool/read.ts:339`), continuation "Use offset=N to continue" (`:345-347`).

**Lesson** — Parameter indexing must match what the output shows.

Related: [[file-read-tool]] · [[tool-description-design]] · [[opencode--file-read-tool|opencode]]
