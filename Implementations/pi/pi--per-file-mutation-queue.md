---
type: implementation
harness: pi
concept: per-file-mutation-queue
commit: b30a6dd77
files: [packages/coding-agent/src/core/tools/file-mutation-queue.ts:32, packages/coding-agent/src/core/tools/edit.ts:163, packages/coding-agent/src/core/tools/write.ts:67, packages/coding-agent/src/core/tools/index.ts:20, packages/coding-agent/docs/extensions.md:143, packages/durable/src/tools/file-mutation-queue.ts:33]
---
[[per-file-mutation-queue]] in [[pi]].

## Mechanism
- `withFileMutationQueue(filePath, fn)` (`packages/coding-agent/src/core/tools/file-mutation-queue.ts:32-61`): module-level `Map<key, Promise<void>>` of per-file promise chains (`:4`). Each call appends a `nextQueue` promise to the file's chain, awaits the previous tail, runs `fn`, releases in `finally`, deletes the map entry if it is still the tail (chain drained) (`:51-60`).
- **Key** = `realpath(resolve(path))`; on `ENOENT`/`ENOTDIR` falls back to the resolved path (`getMutationQueueKey` `:16-26`) → symlink aliases of an existing file share a queue.
- **Global `registrationQueue`** (`:5,33-49`) serializes key resolution: async `realpath` calls could otherwise resolve out of order and reorder same-file mutations; errors swallowed so one failure doesn't jam registration (`:46-49`).
- Users: `edit` (`packages/coding-agent/src/core/tools/edit.ts:163`) and `write` (`write.ts:67`) wrap their **whole read-modify-write window**; exported for extensions (`packages/coding-agent/src/core/tools/index.ts:20`, `sdk.ts:39,136`, `src/index.ts:389`; example `examples/extensions/truncated-tool.ts:25`). Doc rule: "File-mutating tools should wrap the complete read-modify-write operation with `withFileMutationQueue()`." (`packages/coding-agent/docs/extensions.md:143`). Rationale in docs added by `74a46fc7e`: "tool calls run in parallel by default. Without the queue, two tools can read the same old file contents … whichever write lands last overwrites the other."
- Different files fully parallel; same file serialized in request order; no timeout, no lock file, in-process only.
- **Abort discipline inside the queue**: "Do not reject from an abort event listener here: that would release the mutation queue while an in-flight filesystem operation may still finish." → check `signal.aborted` after each await (`edit.ts:164-171`, `write.ts:68-74`).
- Chosen instead of `executionMode:"sequential"` on edit/write, so parallel batches keep concurrency across files ([[parallel-tool-execution]]).

## Constants
none (no timeouts/limits).

## Evolution
- 2026-03-20 `74a46fc7e` (#2327) "queue file mutations across edit and write" — parallel tool execution had just become default (`63ac2df24`, 2026-03-14).
- 2026-03-21 `d38ad0cd6` "preserve file mutation queue ordering" — switched to `realpathSync.native` so key resolution couldn't reorder calls.
- 2026-05-23 `e9146a5ff` "use async operations in tools" — back to async `realpath`, ordering kept by the global `registrationQueue`.
- 2026-10-01 `7fd478a2e` durable copy moved to `packages/durable/src/tools/file-mutation-queue.ts`.

## Evidence commits
`63ac2df24`, `74a46fc7e`, `d38ad0cd6`, `e9146a5ff`, `7fd478a2e`

## Quirks
- **Not a lock against `bash` or other processes** (stated explicitly in durable copy, `packages/durable/src/tools/file-mutation-queue.ts:28-32`); a parallel `bash sed -i` and `edit` on one file still race.
- Create-then-edit under a symlinked dir gap: for a missing file the coding-agent keys on the unresolved path, so `write` (creating) and a later `edit` through a different symlinked path may get different keys (noted as known gap in findings 10; durable fixes it).
- Module-global map: two `AgentSession`s in one process share queues (benign; inferred).

## Durable variant (packages/durable)
- Key = `${env.id}\0${canonicalPath}` (`packages/durable/src/tools/file-mutation-queue.ts:7-10`): namespaced by file-system id (`node:local` for all local envs; each container/remote host its own), so remote and local files with the same path never block each other.
- Missing file → canonical **parent** joined with name, recursively (`canonical` `:16-26`) → create and later edit share a key even under symlinked dirs; `not_supported` canonicalization falls back to absolute path (`:19`).
- Slot taken **without awaiting** after key resolution ("so no other call can take it in between", `:40-47`); no global registration queue → "Concurrent calls on one file run in the order their keys resolve" (`:29-31`), i.e. weaker request-order guarantee than coding-agent.

## Failures
- [[concurrent-file-mutation-interleave]]
