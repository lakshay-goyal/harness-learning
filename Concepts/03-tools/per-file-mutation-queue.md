---
type: concept
stage: tools
tier: candidate
aliases: [withFileMutationQueue, file-mutation-queue, fileMutationQueues, registrationQueue]
harnesses: [pi]
---
Serialize mutating file tools per canonical file path (wrapping the whole read-modify-write window) while calls on other files keep running in parallel.

## Why
- With parallel tool execution, two edits of the same file read the same old content; the last write silently drops the other's change ([[concurrent-file-mutation-interleave]]).
- Symlinked aliases of one file must share the lock; async path canonicalization can reorder same-file requests.

## Design space
- Make mutating tools globally sequential (simple, loses cross-file concurrency).
- **In-process promise chain per canonical path** (pi), keyed by realpath with request order preserved by a registration queue.
- Namespace keys by filesystem identity for multi-environment agents (pi durable: `env.id + canonicalPath`).
- Missing-file key = canonical parent + name (pi durable) vs unresolved path (pi coding-agent).
- OS file locks / lockfiles to cover other processes and the shell (not in pi — explicitly "not a lock against bash").
- Optimistic concurrency: re-read and compare before write (not in pi; durable read has a related concurrent-writer guard → [[file-read-tool]]).

## Implementations
- [[pi--per-file-mutation-queue|pi]] — `withFileMutationQueue` wraps edit/write, exported to extensions; durable copy keyed by fs namespace.

## Failures
- [[concurrent-file-mutation-interleave]]

## Related
[[parallel-tool-execution]] · [[search-replace-edit]] · [[pluggable-tool-backends]] · [[plugin-tools]]
