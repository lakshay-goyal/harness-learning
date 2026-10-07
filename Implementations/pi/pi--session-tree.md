---
type: implementation
harness: pi
concept: session-tree
commit: b30a6dd77
files: [packages/coding-agent/src/core/session-manager.ts:41, packages/coding-agent/src/core/session-manager.ts:57, packages/coding-agent/src/core/session-manager.ts:276, packages/coding-agent/src/core/session-manager.ts:626, packages/coding-agent/src/core/session-manager.ts:1103, packages/coding-agent/src/core/session-manager.ts:1160, packages/coding-agent/src/core/agent-session.ts:1143, packages/coding-agent/docs/session-format.md:10]
---
[[session-tree]] in [[pi]].

## Mechanism
- **Location**: `~/.pi/agent/sessions/--<cwd with / \ : → ->--/<ISO-ts>_<sessionId>.jsonl` (`packages/coding-agent/src/core/session-manager.ts:589-594,1079-1080`; `docs/session-format.md:10-14`). Session id = UUIDv7 (`:264-266`); custom ids must match `^[A-Za-z0-9](?:[A-Za-z0-9._-]*[A-Za-z0-9])?$` (`:268-274`; `52dc08c1f` #5076).
- **Header** line `{type:"session", version, id, timestamp, cwd, parentSession?}`, not a tree node (`:43-50`); `CURRENT_SESSION_VERSION = 3` (`:41`) → [[session-migration]].
- **Entry base** `{type, id, parentId, timestamp(ISO)}` (`:57-62`); id = first 8 hex of `randomUUID()`, ≤100 collision retries, fallback full UUID (`:276-284`; short ids `95312e00b`). Message timestamps ms epoch vs entry ISO (`docs/session-format.md:47`).
- **Entry types** (`:64-194`): `message` (any AgentMessage incl. system/bashExecution/custom roles), `thinking_level_change`, `model_change`, `usage` (non-context billing e.g. `cache_warm`), `compaction{summary, firstKeptEntryId, tokensBefore, details?, usage?, fromHook?, systemMessage?}`, `branch_summary{fromId, summary…}`, `custom` (extension state, **not** in context), `custom_message` (in context, `display` flag), `context_edit` ([[context-edit-overlay]]), `label{targetId, label|undefined}`, `session_info{name}`.
- **Append**: `_appendEntry` pushes, indexes, sets `leafId = entry.id`, persists (`:1191-1196`) → every append is a child of the current leaf.
- **Navigation**: `branch(id)` moves only the in-memory leaf (`:1579-1584`); `resetLeaf()` → next append is a new root (`:1591-1593`); `branchWithSummary(fromId|null,…)` appends a `branch_summary` under the target recording old leaf in `fromId` (`:1600-1625`; `d711bd5f0` fixed recording the destination) ([[branch-summary]]). `getTree()` treats orphans/self-parent as roots, sorts children by timestamp iteratively (no recursion — `544814875` stack overflow fix) (`:1529-1567`).
- **`/tree`** = `AgentSession.navigateTree` (`packages/coding-agent/src/core/agent-session.ts:3963-4155`): rejects while streaming (`cefa40ed8` #7022, `:3967-3969`) or compacting (`e687434a6` #9179, `:3970-3973`); optional LLM summary of abandoned path to common ancestor; selecting a user/custom message sets leaf to its parent and returns text to the editor.
- **Leaf on load = last entry in file order** (`_buildIndex`, `:1103-1122`) → a `/tree` move that appends nothing is not persisted; resume returns to the most recently appended entry (inferred).
- **Labels are tree entries** under the current leaf (they move the leaf) (`:1441-1462`; `bookmark.ts` example uses `setLabel`).
- **Persistence timing**: every `message_end` → `appendMessage` (`agent-session.ts:1143-1163`); `_persist` (`:1172-1189`): until `_hasConversation()` (a user or assistant message exists, `:1160-1170`) entries stay in memory; first write opens with `openSync(…, "wx")` (exclusive create) and flushes all buffered entries; then `appendFileSync` per entry. Synchronous, **no fsync** (no fsync/rename in file). Streaming partials never written ([[partial-message-persistence]]).
- **Adjacency deferral**: user `!` bashExecution and run-time custom messages are buffered while streaming so they never land between a tool call and its result (`agent-session.ts:3891-3898,3924-3932,2319-2324`; `240eb29c4` #8537) → [[out-of-band-message-deferral]].
- **Ordering**: listeners awaited; assistant `message_end` persisted before tool preflight (barrier, `packages/agent/src/agent.ts:565-612`); history in [[tool-result-persisted-before-tool-call]].
- **Load / torn-tail repair** `loadEntriesFromFile` (`:626-670`): stream 1 MiB chunks (`SESSION_READ_BUFFER_SIZE`, `:604`) through `StringDecoder`; malformed lines skipped (`:615-623`); header must be `type:"session"` with string id else `[]`; if the file lacks a trailing newline append `"\n"` so the next append can't fuse (`:668`; `0b5ee5d8b` #8345). Torn line itself stays on disk, skipped each load. Empty file → initialized; non-empty non-session file → error, not overwritten (`:1028-1046`; `543710f64`).
- **Header discovery** bounded: 4 KiB reads, ≤1 MiB scan, `SessionHeaderScanLimitError` (`:605-614,685-727`).
- **Resume**: `--continue` = newest `.jsonl` by mtime in the cwd dir (`findMostRecentSession` `:749`, `continueRecent` `:1793-1801`); missing stored cwd → `MissingSessionCwdError` / prompt (`packages/coding-agent/src/core/session-cwd.ts:14-59`; `080af6fc0`). New sessions append `model_change` + `thinking_level_change` at creation; resume restores model/thinking from entries (`packages/coding-agent/src/core/sdk.ts:210-277,448-459`) and tool loadout from replayed system message (`agent-session.ts:1808-1815`).
- **Session replacement** (`/new`, `/resume`, `/fork`) calls `session.abort()` first "so the aborted turn (including tool results) is persisted to the outgoing session" (`agent-session-runtime.ts:167-178`; `cefa40ed8` #7022).
- **Interactive entry points**: Esc with an empty editor twice within 500 ms opens `/tree` (default) or the `/fork` user-message picker, per `doubleEscapeAction: "tree"|"fork"|"none"` (`modes/interactive/interactive-mode.ts:3059-3074`; `core/settings-manager.ts:1469-1471`) — Esc first aborts streaming / running `!` bash / exits bash mode (`:3050-3058`). `/tree` opens with `treeFilterMode` (`default|no-tools|user-only|labeled-only|all`, `settings-manager.ts:1479-1483`; `interactive-mode.ts:5541`); filters toggle in the selector via `app.tree.filter.*` keys (Ctrl+D/T/U/L/A, Ctrl+O cycles; `docs/keybindings.md:176-182`). `/import <path.jsonl>` asks for confirmation, then replaces the current session through `runtimeHost.importFromJsonl` (`interactive-mode.ts:6540-6580`) → [[pi--session-fork|session-fork]].

## Constants
| name | value | path:line |
|---|---|---|
| `CURRENT_SESSION_VERSION` | 3 | packages/coding-agent/src/core/session-manager.ts:41 |
| entry id | 8 hex, ≤100 retries | packages/coding-agent/src/core/session-manager.ts:276-284 |
| `SESSION_READ_BUFFER_SIZE` | 1 MiB | packages/coding-agent/src/core/session-manager.ts:604 |
| `SESSION_HEADER_READ_BUFFER_SIZE` | 4096 | packages/coding-agent/src/core/session-manager.ts:605 |
| `MAX_SESSION_HEADER_SCAN_BYTES` | 1 MiB | packages/coding-agent/src/core/session-manager.ts:607 |
| `MAX_CONCURRENT_SESSION_INFO_LOADS` / `…DISCOVERY_LOADS` | 10 / 64 | packages/coding-agent/src/core/session-manager.ts:891-892 |
| session-list publish batch | 10 / 100 | packages/coding-agent/src/core/session-manager.ts:893-894 |
| session file size cap | none (unverified: no cap found) | — |

## Evolution
- 2025-11-12 `812f2f43c` defer session creation until first user+assistant exchange.
- 2025-11-14 `8ae236f95` `/branch` (linear copy) — precursor of fork.
- 2025-12-22 `184c64833` flush all buffered entries on first assistant message.
- 2025-12-25 `c58d5f20a` **tree**: id/parentId (v2); `95312e00b` 8-char ids; 2025-12-29 `4958271dd` `/tree`, `544814875` iterative sort; 2025-12-30 `98c85bf36` save initial model/thinking.
- 2026-01-05 `c6fc08453` v3 (`hookMessage`→`custom`).
- 2026-03-02 `dfc779faa` → 2026-03-30 `9022a5b5e` → 2026-05-19 `32bcdc973`: persistence ordering via queue then awaited listeners.
- 2026-06-20 `a1da88aed` linear path traversal (#5909).
- 2026-06-25 `543710f64` reject invalid session files instead of overwriting.
- 2026-07-28 `cefa40ed8` guard tree navigation during responses (#7022).
- 2026-08-19 `d711bd5f0` branch summary source leaf.
- 2026-08-26 `0b5ee5d8b` repair unterminated session files (#8345).
- 2026-09-25 `ff72faba2` create file at first **user** message (#10000).

## Evidence commits
`812f2f43c`, `184c64833`, `c58d5f20a`, `95312e00b`, `4958271dd`, `544814875`, `c6fc08453`, `dfc779faa`, `9022a5b5e`, `32bcdc973`, `a1da88aed`, `543710f64`, `cefa40ed8`, `d711bd5f0`, `0b5ee5d8b`, `ff72faba2`

## Quirks
- No fsync, no atomic rename on rewrite (`_rewriteFile` opens `"w"` and rewrites in place, `:1124-1134`) → a crash during migration rewrite can truncate the log (inferred).
- Leaf not persisted: pure navigation lost on resume (open question, `:1103-1122`).
- Session files grow unbounded (append-only; compaction never deletes).

## Durable variant (packages/durable)
- Conversations + immutable entries (`pi.user/assistant/tool-result/system/reset/compaction`, `packages/durable/src/entries.ts:15-34`) in storage tables; one global numeric id namespace (ID 1 = root conversation, `mintId` from 2, exhaustion sticky) (`src/testing/storage-conformance.ts:98-105,1634-1652`); forks are conversations with a fork point, not copies; JSONL backend recovery removes torn final lines and ignores unconfirmed sidecar tails (`docs/spec.md:4579-4587`). See [[durable-execution]].

## Failures
- [[tool-result-persisted-before-tool-call]] · [[torn-log-tail-fuses-next-entry]] · [[session-lost-before-first-response]] · [[session-config-not-restored-on-resume]] · [[quadratic-long-session-operations]] · [[time-ordered-id-prefix-collision]]
