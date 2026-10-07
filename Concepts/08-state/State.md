---
type: group
group: 08-state
---
How conversation and plugin state is persisted, branched, projected into model context, migrated, made crash-durable and replicated to UIs.

## Concepts
- [[session-tree]] — Append-only JSONL of id/parentId entries; leaf selects active branch; deferred file creation; torn-tail repair.
- [[session-fork]] — Copy root→target path into a new session linked to its parent (vs in-file branching).
- [[context-projection]] — Model context rebuilt from the log on every request (latest compaction/head + kept + edits), not trusted from memory.
- [[context-edit-overlay]] — Append-only edits omitting/replacing an earlier entry's model-visible content (e.g. failed attempts).
- [[partial-message-persistence]] — Persist interrupted/errored assistant output but skip it on replay; persistable frames; durable partial commits.
- [[session-migration]] — Versioned on-load migration of the log format and stored typed state.
- [[branch-scoped-extension-state]] — Plugin/tool state stored in tool-result details or custom entries so each branch sees its own.
- [[durable-execution]] — Every loop step a checkpointed task committed before shown; crashed process resumes from storage.
- [[replicated-state]] — Single-writer sequence-numbered state published as snapshot + delta ops; overflow collapses to snapshot.
- [[sqlite-session-index]] — JSONL transcripts stay source of truth; a disposable SQLite mirror indexes thread metadata (+ paginated item projection) for listing/search/jobs, rebuildable from the files.

## Absences
[[no-checkpoints-undo]] · [[no-todo-tool]] · [[no-partial-history-fork]] · [[no-server-stored-conversation]] · [[no-tool-output-spill-file]]

Tradeoffs: [[session-log-shape]]

Failures: [[State Failures]]
