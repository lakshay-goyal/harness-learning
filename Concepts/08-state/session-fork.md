---
type: concept
stage: state
tier: candidate
aliases: [/fork, /clone, createBranchedSession, parentSession, forkFrom, AgentSessionRuntime.fork, "conversation.fork()"]
harnesses: [pi]
---
Create a new session identity by copying the root→target path of an existing session into a fresh log linked back to its parent — versus in-file branching (tree navigation), which keeps one file.

## Why
- Users want an independent copy (separate file, separate resume entry, separate provider cache identity) from a point in history without dragging abandoned branches along.
- Copying a path is not copying entries: derived references (compaction `firstKeptEntryId`, labels as tree nodes, summary source leaves) break ([[fork-boundary-loss]]); writing to the wrong file or twice duplicates headers ([[fork-writes-wrong-file]]).
- Conversation fork ≠ workspace fork: files on disk are not restored unless something snapshots them ([[no-checkpoints-undo]]).

## Design space
- **In-file branch** (`/tree`) vs **new file with path copy** (`/fork`, `/clone`) vs **copy whole file elsewhere** (`forkFrom` across cwd) — pi has all three.
- **Fork point**: before a user message (text returned to editor) vs at an entry (pi `position: "before"|"at"`).
- **Lineage**: header `parentSession` link (pi) vs none.
- **Derived-state handling**: re-chain parents around dropped nodes, remap references (pi) vs copy verbatim.
- **Workspace state**: none (pi core) vs git-stash checkpoints offered on fork (pi example) vs filesystem snapshots.
- **Durable**: `conversation.fork(entryId)` sees parent entries up to `entryId` without copying; fork gets the parent agent config as of that entry and a fresh provider session id; documents choose `fork: initial|current|asOf`.

## Implementations
- [[pi--session-fork|pi]] — `AgentSessionRuntime.fork` → `createBranchedSession` writes a new JSONL with only the path (labels re-chained, compaction remapped), `parentSession` header; `/clone` copies active branch; `git-checkpoint.ts` example.

## Failures
- [[fork-boundary-loss]]
- [[fork-writes-wrong-file]]

## Related
[[session-tree]] · [[branch-summary]] · [[auto-compaction]] · [[session-handoff]] · [[durable-execution]] · [[session-affinity-cache-routing]] · [[no-checkpoints-undo]]
