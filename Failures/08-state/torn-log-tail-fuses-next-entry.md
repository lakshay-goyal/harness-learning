---
type: failure
concepts: [session-tree, session-migration]
harnesses: [pi]
---
**Symptom** — After a crash mid-write left a session JSONL without a trailing newline, the next appended entry was concatenated onto the torn line, corrupting *two* entries (#8345). Related: interrupted fork/repair writes left partial files (#7707); loading an empty/invalid file via `--session` overwrote it (#932, #6002).

**Root cause** — Append-only log assumed every prior write completed; no repair step at open; rewrite paths wrote in place.

**Fix · [[pi]]**
- `0b5ee5d8b` 2026-08-26 "repair unterminated session files": after a successful header check, `loadEntriesFromFile` appends `"\n"` when the last line lacks one; malformed lines are skipped (`packages/coding-agent/src/core/session-manager.ts:615-623,661-668`).
- `a838c069e` 2026-08-06 (#7707) — atomic writes for forks + torn-tail truncation in the (since removed, `7fd478a2e`) agent-core harness JSONL storage.
- 0.50.0 (#932) and `543710f64` 2026-06-25 (#6002) — empty file initialized; non-empty invalid file rejected without overwrite (`session-manager.ts:1028-1046`; `packages/coding-agent/CHANGELOG.md:1383,4023`).
- pi-durable JSONL: commit markers; recovery removes torn final lines and ignores unconfirmed sidecar tails (`packages/durable/docs/spec.md:4579-4587`).

**Lesson** — Append-only logs need torn-tail repair at open and atomic (temp + rename) rewrites; never overwrite a file you failed to parse. (Coding-agent still rewrites in place during migration — `_rewriteFile`, `session-manager.ts:1124-1134`.)

Related: [[session-tree]] · [[session-migration]] · [[durable-execution]] · [[pi--session-tree|pi]]
