---
type: tradeoff
concepts: [session-tree, session-fork, context-projection, sqlite-session-index, session-migration, durable-execution]
---
**Axis** — How the persistent conversation log is shaped: one file holding a branchable tree vs one linear file per thread (new file per fork/rewind) with a separate index.

| aspect | pi | codex |
|---|---|---|
| shape | id/parentId **tree** in one JSONL; leaf = last appended entry (`packages/coding-agent/src/core/session-manager.ts:1103-1122`) | **linear** rollout JSONL per thread (`codex-rs/history/src/lib.rs:357-367`) |
| rewind | move leaf in-file (`/tree`), history kept | `thread/revert` writes a new immutable rollout file, same thread id (`4343b2bdc4`); earlier: append `ThreadRolledBack` markers (`8b7ec31ba7` → removed `3052bbcf8c`) |
| fork | new file with root→leaf path, `parentSession` header ([[pi--session-fork]]) | new thread with truncated prefix, `forked_from_id` (`codex-rs/protocol/src/protocol.rs:3192-3196`); also the sub-agent spawn primitive ([[codex--session-fork]]) |
| request context | projected from the log on every request ([[pi--context-projection]]) | live in-memory history; projection only at resume/fork, reverse scan to newest `replacement_history` compaction ([[codex--context-projection]]) |
| compaction record | entry with summary + `firstKeptEntryId` | `CompactedItem` with full `replacement_history` + token usage (`codex-rs/history/src/lib.rs:286-306`) |
| listing/search | scan headers with bounded reads (`session-manager.ts:605-614`) | SQLite mirror `state_5.sqlite` + paginated item projection, rebuildable ([[sqlite-session-index]]) |
| durability | sync append per entry, no fsync | `PersistContext` fences, per-thread writer lock (`codex-rs/thread-store/src/store.rs:68-94`, `codex-rs/rollout/src/writer_lock.rs:17-18`) |
| payload size | no cap (unverified) | persistence-only 64 KiB caps for MCP/command output (`codex-rs/rollout/src/policy.rs:16-17`); `.zst` after 7 days |
| format change | eager v1→v3 migration, in-place rewrite | staged + verified + atomic publish with `.pending` journal (`codex-rs/thread-store/src/local/rollout_migration.rs:1-9`) |

**When each wins**
- **Tree in one file (pi)**: interactive exploration — users branch and compare attempts without file sprawl; exports/branch summaries can see every alternative; cheap for single-user CLI.
- **Linear file + index (codex)**: many clients/hosts and thousands of threads — immutable files per revert/fork are simple to replicate, lock and migrate; the index gives fast listing/pagination for IDE/desktop sidebars; resume cost is bounded by the last compaction checkpoint.
- Shared lesson: both repair torn tails before append ([[torn-log-tail-fuses-next-entry]]); an index must stay derivable from the log ([[index-db-corruption]]).
