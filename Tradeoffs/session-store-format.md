---
type: tradeoff
concepts: [session-tree, event-sourced-session-store, durable-execution, context-projection, partial-message-persistence, session-fork, sqlite-session-index, session-migration]
harnesses: [pi, opencode]
---
# session-store-format

**Axis**: how is a session persisted: an append-only file per session shaped as a tree, or rows in a shared database written through events?

| option | pi | opencode | evidence |
|---|---|---|---|
| JSONL file per session, entries form a tree (`id`/`parentId`, leaf pointer) | ✅ stable: `~/.pi/agent/sessions/--<cwd>--/<ts>_<id>.jsonl`, `CURRENT_SESSION_VERSION = 3` | ❌ (legacy JSON files only migrated) | pi: `packages/coding-agent/src/core/session-manager.ts:41,57-62,589-594` → [[session-tree]] |
| One SQLite DB (WAL), JSON blobs per message/part | durable package only (experimental) | ✅ legacy: `MessageTable`, `PartTable`; `busy_timeout = 5000` | opencode: `packages/core/src/database/database.ts:27-30`; `packages/core/src/session/sql.ts:22-98`; "sqlite again" `6d95f0d14c` 2026-02-13 → [[pi--durable-execution|pi durable]] |
| Typed versioned events + in-transaction projectors | — | ✅ v2: `publish` allocates seq, runs projectors, writes event rows in one immediate transaction; deltas live-only | `packages/core/src/event.ts:205-351`; `fb43c15f88` → [[event-sourced-session-store]] |
| Branching | in-file tree: `/tree`, `/fork`, `/clone`, branch summaries | linear; fork copies messages into a new session with id remap | pi `docs/sessions.md:20-32` → [[session-fork]], [[branch-summary]] |
| Streamed deltas persisted? | assistant persisted at `message_end` | no: a part is written empty at start and full at end; crash mid-stream loses text | opencode `packages/opencode/src/session/processor.ts:500-546` → [[partial-message-persistence]] |
| Ordering authority | file order | legacy: `time.created` then id (after reorder bugs `94564f3588`, `db581e47a3`); v2: durable `seq` | → [[timestamp-ordered-history-pagination]] |

**When each wins**
- **JSONL tree (pi)**: one human-readable file per session, trivially portable, `grep`-able; branching is native. Concurrency is per-file; torn tails are repaired on load.
- **SQLite + events (opencode)**: many sessions, one server serving several clients, queries across sessions (cost totals, lists), and a replayable log that feeds the UI and the model context from the same projections. Costs: schema migrations, a DB lock shared by every process, and an ordering authority that must be designed (timestamps failed).

Related: [[session-tree]] · [[client-server-vs-single-process]] · [[undo-vs-none]] · [[pi]] · [[opencode]] · [[Tradeoffs]]

## Also: [[codex]] (folded from `session-log-shape`)
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
