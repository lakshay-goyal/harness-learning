---
type: failure
concepts: [per-file-mutation-queue, parallel-tool-execution]
harnesses: [pi]
---
**Symptom** — When a turn contained two `edit`/`write` calls for the same file, one change was silently lost; later, same-file mutations could run in a different order than the model issued them.

**Root cause** — Parallel tool execution became default (`63ac2df24`, 2026-03-14): both calls read the same old content, the last write wins. The first queue fix resolved keys with async `realpath`, which could complete out of order and reorder requests.

**Fix · [[pi]]**
- `74a46fc7e` 2026-03-20 (#2327) — `withFileMutationQueue` around the whole read-modify-write in edit and write, keyed by realpath (`packages/coding-agent/src/core/tools/file-mutation-queue.ts:32-61`); doc rule for extensions (`packages/coding-agent/docs/extensions.md:143`).
- `d38ad0cd6` 2026-03-21 — preserve ordering (sync `realpathSync.native`).
- `e9146a5ff` 2026-05-23 — back to async I/O with a global `registrationQueue` serializing key resolution (`file-mutation-queue.ts:5,33-49`).
- Durable copy namespaces keys by filesystem id and canonicalizes missing files via the parent (`packages/durable/src/tools/file-mutation-queue.ts:4-55`).

**Lesson** — Parallel tool execution requires per-resource serialization of read-modify-write windows; key by canonical identity and keep request order. (Still not a lock against `bash`.)

Related: [[per-file-mutation-queue]] · [[parallel-tool-execution]] · [[pi--per-file-mutation-queue|pi]]
