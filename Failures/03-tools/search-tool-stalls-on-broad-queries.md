---
type: failure
concepts: [search-tools, abort-propagation]
harnesses: [pi]
---
**Symptom** — `grep` stalled on broad searches (many matches); `find` could not be cancelled while discovering ignore files.

**Root cause** — grep re-read each matching file synchronously to format the line; ignore discovery in find was not abort-aware.

**Fix · [[pi]]** — `e9ba9e2eb` 2026-04-16 (#3148, #3205): format matches from ripgrep's `--json` line text when `context=0` (file reads only with context>0, cached) and make find's I/O abortable (`packages/coding-agent/src/core/tools/grep.ts:147-160,197-215`).

**Lesson** — Use the backend's streamed data instead of re-reading files per match, and make every I/O step abortable.

Related: [[search-tools]] · [[abort-propagation]] · [[pi--search-tools|pi]]
