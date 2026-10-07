---
type: failure
concepts: [file-read-tool, pluggable-tool-backends]
harnesses: [pi]
---
**Symptom** — (durable) The bounded `read` tool failed with "changed while it was read" on an actively appended log file.

**Root cause** — The concurrent-writer guard compared file metadata before/after the read and treated any change as inconsistency — including pure growth, where the scanned bytes are unchanged.

**Fix · [[pi]]** — `cd60a5b99` 2026-10-05 (durable `CHANGELOG.md:42`): accept a file that only grew; a file that shrank or was rewritten is re-read once, and fails only on a second change (`packages/durable/src/tools/read.ts:84-94`). Guard itself arrived with bounded read `a19c09d9b` 2026-10-04.

**Lesson** — Distinguish append-only growth (safe) from rewrite (retry) when guarding reads against concurrent writers.

Related: [[file-read-tool]] · [[pluggable-tool-backends]] · [[pi--file-read-tool|pi]]
