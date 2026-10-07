---
type: concept
stage: state
tier: candidate
aliases: [SessionManager, JSONL, leaf, leafId, /tree, labels, parentId, branch(), branchWithSummary, deferred-session-file, torn-tail-repair, _hasConversation]
harnesses: [pi]
---
Persistent conversation log as an append-only file of typed entries linked by `id`/`parentId`; a leaf pointer selects the active branch, so rewinds/branches never delete history; with crash rules for when the file is created and how a torn last line is repaired.

## Why
- Linear transcripts force destructive edits for "go back and try differently"; a tree keeps every attempt for audit/export while the model sees one path.
- Persistence ordering and crash timing determine what survives: tool results written before their call ([[tool-result-persisted-before-tool-call]]), prompt lost if file is created too late ([[session-lost-before-first-response]]), torn last line fusing the next append ([[torn-log-tail-fuses-next-entry]]).
- Long sessions make every path walk hot ([[quadratic-long-session-operations]]).

## Design space
- **Shape**: linear list (pi ≤ v1) vs id/parentId tree in one file (pi v2+, `c58d5f20a`) vs one file per branch vs DB tables (pi-durable SQLite/JSONL with conversations + fork points).
- **Active branch**: in-memory leaf only, leaf = last appended entry on load (pi) vs persisted leaf pointer.
- **Metadata as tree nodes** (labels, model/thinking changes, summaries) vs side tables — pi uses nodes (labels move the leaf; forks must re-chain).
- **Entry ids**: short random (8 hex of UUID, collision-checked) vs UUIDv7 vs global numeric (durable).
- **File creation**: eager vs deferred until conversation exists (pi: at first user message).
- **Write path**: sync append per entry, no fsync (pi) vs fsync / WAL / commit markers (durable).
- **Torn tail**: skip malformed lines + append `\n` on load (pi) vs truncate (removed harness) vs commit-marker recovery (durable JSONL).
- **What enters context**: `custom` (extension state, not in context) vs `custom_message` (in context) entry types.
- **No tree at all**: linear message/part rows in one global SQLite database written through event projectors; rewinds are soft revert markers committed on the next prompt, branches are row-copy forks (opencode, not an implementation of this concept) → [[event-sourced-session-store]], [[workspace-snapshots]], [[opencode--session-fork|opencode fork]].

## Implementations
- [[pi--session-tree|pi]] — `~/.pi/agent/sessions/--<cwd>--/<ts>_<uuidv7>.jsonl`; header v3 + typed entries; leaf = last entry; deferred `wx` create at first user message; `\n` torn-tail repair.

## Failures
- [[repeated-compaction-drops-kept-messages]]
- [[fork-boundary-loss]]
- [[session-switch-leaves-dangling-tool-calls]]
- [[tool-result-persisted-before-tool-call]]
- [[torn-log-tail-fuses-next-entry]]
- [[session-lost-before-first-response]]
- [[session-config-not-restored-on-resume]]
- [[quadratic-long-session-operations]]
- [[time-ordered-id-prefix-collision]]

## Tradeoffs
- [[undo-vs-none]]
- [[session-store-format]]

## Related
[[session-fork]] · [[context-projection]] · [[context-edit-overlay]] · [[partial-message-persistence]] · [[session-migration]] · [[branch-scoped-extension-state]] · [[branch-summary]] · [[auto-compaction]] · [[session-export-share]] · [[durable-execution]] · [[no-checkpoints-undo]]
