---
type: implementation
harness: opencode
concept: per-file-mutation-queue
commit: ecc4916b5a
files: [packages/opencode/src/tool/edit.ts:35-45, packages/core/src/file-mutation.ts:32-82, packages/core/src/tool/edit.ts:112-117]
---
[[per-file-mutation-queue]] in [[opencode]].

## Mechanism
### Legacy runtime
- `locks: Map<resolvedPath, Semaphore>`; `lock(filePath)` keys by `FSUtil.resolve(filePath)` and creates `Semaphore.makeUnsafe(1)` (`packages/opencode/src/tool/edit.ts:35-45`); edit's whole read-modify-write runs inside it.
- Only `edit` takes it. `write` and `apply_patch` call no `lock(` (grep at HEAD), so an `edit` and a `write` on the same path can still interleave.
- Map entries are never evicted (process-lifetime growth; minor).
### v2 runtime
- `FileMutation` service: process-local keyed mutex per canonical path plus **optimistic check**: `writeIfUnchanged` compares bytes read at permission time with bytes at commit, else `StaleContentError` (`packages/core/src/file-mutation.ts:32,61-63`).
- Edit maps it to "File changed after permission approval. Read it again before editing." (`packages/core/src/tool/edit.ts:115`).
- Spec: "V2 exact edits must fail rather than stale-clobber a concurrent cooperating write after permission approval" (`specs/v2/schema-changelog.md:541`).

## Constants
| name | value | path:line |
|---|---|---|
| permits per path | 1 | `packages/opencode/src/tool/edit.ts:42` |

## Evolution
- 2025-12-15 `5cf126d489` "add per-file lock to prevent read-before-write race (#4388)" (then via `FileTime.withLock`).
- 2026-04-16 `76a141090e` FileTime deleted; lock moved into edit.
- 2026-04-20 `8bc4f91fd9` "parallel edits sometimes would override each other (#23483)" → [[concurrent-file-mutation-interleave]].

## Quirks / drift
- Lock scope is one tool, not "mutating tools"; pi wraps both edit and write.

Contrast: pi chains promises per realpath for edit and write with a registration queue preserving order → [[pi--per-file-mutation-queue|pi]].
