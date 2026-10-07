---
type: tradeoff
concepts: [session-tree, event-sourced-session-store, durable-execution, context-projection, partial-message-persistence]
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
