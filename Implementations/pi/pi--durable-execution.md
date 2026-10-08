---
type: implementation
harness: pi
concept: durable-execution
commit: b30a6dd77
files: [packages/durable/README.md:5, packages/durable/docs/spec.md:43, packages/durable/docs/spec.md:1609, packages/durable/src/session/session.ts:431, packages/durable/src/harness/generation.ts:53, packages/durable/src/harness/tool.ts:37, packages/durable/src/storage/jsonl/storage.ts:441, packages/durable/src/storage/sqlite/node.ts:190, packages/coding-agent/src/experimental/session-worker.ts:517]
---
[[durable-execution]] in [[pi]].

## Mechanism
- **Package**: `@earendil-works/pi-durable` ("Pico5", `docs/spec.md:1-3`), "**Experimental.** The API changes without notice" (`packages/durable/README.md:3`), v1.0.4 (`packages/durable/package.json:3`). Goal: "Conversations, model turns, tool calls, and your own state are committed to storage before anything is shown. If the process dies mid-turn, reopening the storage picks the work up where it stopped." (`packages/durable/README.md:5`). Core rule: "A Session atomically commits immutable entries, full task records, and Chord-tracked documents. Only committed state is observable." (`docs/spec.md:43-44`). **Not used by stable pi** — only `packages/coding-agent/src/experimental/**` imports it (`session-worker.ts:15`, `client-tui.ts:12`), gated `PI_EXPERIMENTAL=1`.
- **Spec invariants** (`spec.md:63-78`): commit atomic across records + docs; doc update published only after storage commit; "All visible progress is durable. There is no volatile publication path."; external effects never inside the mutation transaction; ids immutable/never reused; drafts revoked when callback settles, strict JSON; mutation line held through storage settlement; "An uncertain storage failure is fatal" → session **poisoned** unless `StorageRejected` (`src/session/session.ts:431-440,545-547`). Table reads only before first write (`ReadAfterWrite`, `src/session/transaction.ts:924`).
- **Tasks** (`spec.md:1609-1630`): `pending | running | waiting(on, policy) | completing(outcome) | terminal(outcome)`; outcomes `completed | failed | aborted | orphaned | faulted`. Scheduler serially reserves, runs handlers off the session line (`spec.md:1929-1931`). Abort: commit `abortRequested` → signal+join → wait owned work → abort handler commits terminal (`spec.md:1935-1941`). Missing/incompatible code ⇒ `blocked: missing_task | task_too_old | migration_failed`, never terminalized; `orphaned` = "no task code ran, so external effects the task started may remain uncleaned" (`spec.md:2011-2039`; `src/harness/scheduler.ts:57`). Memos: small first-writer-wins values (`spec.md:1885-1888`). Hot reload hands a task over at the next phase boundary (`spec.md:2077-2087`).
- **Effect sandwich** (`spec.md:1872-1883`): "commit intent phase → perform external effect → commit outcome or next phase"; reopening in an intent phase "may have happened" → retry safely, poll a handle, or record interruption → [[crash-safe-tool-replay]].
- **Loop decomposed into tasks** (`spec.md:3297-3301`): the `while` loop becomes `pi.generation` → `pi.tool`×n (waiting `allSettled`) → next `pi.generation`, handing off the `pi.live.run` baton (`spec.md:2208-2217,3669-3673`).
  - `pi.generation` phases `prepare` (`src/harness/generation.ts:53`), `request` (`:61`), `retry` (`:71`), `poll` (`:73`), `tools` (`:83`); starts `{phase:"prepare", attempt:1}` (`:118`). `prepare` renders sections/tools, appends `pi.system` delta only if changed, pins model/thinking/stream options/`cutoff`; above `contextWindow − reserveTokens` blocks on owned compaction, above `− backgroundTokens` starts background compaction (`:317-321`). Overflow error compacts **once** then retries (`:464-471`); retryable errors backoff (`:484-492`); `deferred` → `poll` after `pollAfterMs ?? 5000` (`:109,432`) → [[deferred-responses]].
  - `pi.tool` phases `call`, `execute` (`src/harness/tool.ts:37-38`): call validates, `beforeTool` chain, re-validates, commits `{phase:"execute", arguments, replay: tool.replay ?? "unsafe"}` (`:55-90`); `execute` reached only by recovery: reruns only if stored **and** current policy are `safe`, else error result "Tool ${name} was interrupted and may have partially run" (`:93-110`). pi-durable's `CodingTools` (`packages/durable/src/tools/*.ts`: read/bash/edit/write) declare no `replay` → `unsafe`; only the experimental `subagent` tool (`packages/coding-agent/src/experimental/durable/subagent.ts:33`) and the vacation demo search (`experimental/vacation/vacation.ts:44`) declare `safe`.
  - `pi.compaction` phases `select`, `summarize` (`src/harness/compaction.ts:103-200`) → [[background-compaction]].
- **Submissions/inbox**: `submit()` durably admits input; `requestId` dedups within a conversation so a post-restart retry returns the same submission (`packages/durable/README.md:115-124`; `src/harness/submissions.ts:155-160`); busy ⇒ `pi.inbox` with `whenBusy: steer|followUp|reject`; boundaries `postTools` / `final` (`spec.md:2253-2256`) → [[steering-queue]].
- **Storage** (`spec.md:4418-4443`): `commit(writes)`, `mintId`, keyed lookups, cursor scans, `findLatestHeadMarker`, `document(id, at)`; one global numeric id namespace, sequences increase with gaps.
  - **Memory**: "the reference semantics", copies everything to simulate serialization (`spec.md:4521-4525`).
  - **JSONL**: `main.jsonl` (writes + one commit marker per commit) + `doc-<id>.jsonl` + `task-<id>.jsonl` sidecars; publish = append sidecars → append marker `{format, type:"commit", seq, writes}` (`src/storage/jsonl/storage.ts:441-447`) → publish in memory. `fsync` option default **false** (`:77-78,256`); with it, sidecars flushed before marker (`:294-301`). Recovery removes torn final lines, ignores unconfirmed sidecar tails, missing confirmed data = corruption, open fails (`spec.md:4579-4587`; `storage.ts:542-710`); uncertain append poisons backend (`:90,292,301`); `main.jsonl` never compacted (`spec.md:4593-4594`).
  - **SQLite**: tables `durable_metadata, record_ids, conversations, entries(head, commit_seq), tasks, submissions(request_id), documents, document_revisions(base|delta)` (`src/storage/sqlite/migrations.ts:9-87`); `next_id` TEXT (node:sqlite int limit, `:8`); WAL, `synchronous=NORMAL`, configurable autocheckpoint (`src/storage/sqlite/node.ts:190-192`) — "commits survive process crashes; the newest may be lost on power or host failure" (`packages/durable/README.md:541`).
  - **Cloudflare Durable Object SQLite** (`da866ada1`); storage cores avoid Node APIs (`packages/durable/README.md:545`).
  - **Single writer**: "One process owns a storage at a time; there is no cross-process locking" (`packages/durable/README.md:545`); non-goals: CRDT/offline multi-writer, forced termination of non-cooperative extension code (`spec.md:4743-4747`).
- **Replay on open**: re-materialize docs (base + delta tail), reopen tasks; `running` shows as `pending` until rerun (`packages/durable/README.md:502`); "a crash loses at most" one 100 ms throttle window (`packages/durable/README.md:304`).
- **Conformance**: exported `createStorageConformance` (`src/testing/storage-conformance.ts:94-1661`): ID 1 = root conversation, mint from 2; atomic multi-table commit with rollback incl. secondary indexes; detached records; `__proto__` keys safe; global id namespace, `MAX_SAFE_INTEGER` ⇒ sticky "ID space is exhausted"; all ops rejected after close. Run against Memory, JSONL (incl. reopen), SQLite (incl. reopen), DO SQLite. Validated benchmarks with correctness oracles + fresh fixture pools (`src/testing/storage-benchmark.ts`, `test/storage.bench.ts`).
- **Hosts**: experimental durable TUI (single process owns runtime + SQLite + TUI; sessions under `~/.pi/agent/experimental/durable-sessions/<sha256(cwd)[:24]>/<ms>-<uuid>/session.sqlite`, `proper-lockfile` 12 × 1 s, stale 10 s — `experimental/durable/sessions.ts:20-25,40,47-51`; "Nothing in the TUI handles recovery"); session-worker process per session takes `proper-lockfile` (stale 2 s, update 1 s), opens Harness, calls `harness.resume()` ("Recovered work from an interrupted turn continues now"); "Every live task, background work included, keeps the worker and its Session open." (`session-worker.ts:517-522,591-594`) → [[client-server-session-split]].

## Constants
| name | value | path:line |
|---|---|---|
| retry | maxRetries 3, base 2000 ms, maxAgentDelay 60 000 ms | packages/durable/src/harness/agent.ts:19-24 |
| compaction | reserve 16 384, keepRecent 20 000, background 32 768 | packages/durable/src/harness/agent.ts:26-31 |
| progress | partial 100 ms / output 100 ms | packages/durable/src/harness/agent.ts:34-35 |
| `contextRetentionMs` | 600 000 | packages/durable/src/harness/agent.ts:52-53 |
| deferred poll default | 5000 ms | packages/durable/src/harness/generation.ts:109 |
| SQLite `DEFAULT_WAL_AUTO_CHECKPOINT_PAGES` / `DEFAULT_BUSY_TIMEOUT_MS` | 1000 / 5000 | packages/durable/src/storage/sqlite/node.ts:16-17 |
| `SCAN_PAGE_SIZE` | 256 | packages/durable/src/harness/harness.ts:57 |
| JSONL `FORMAT_VERSION` | 1 | packages/durable/src/storage/jsonl/storage.ts:28 |
| session-worker lock | stale 2 s, update 1 s | packages/coding-agent/src/experimental/session-worker.ts:517-522 |

## Evolution
- 2026-05-03 `a5b27367d` first `AgentHarness` inside pi-agent-core; 2026-05-14 `b7ea82105`, `846906e4d` result-based env.
- 2026-08-25 `5c6655e76` "make durable assistant framing burst-safe" — first "durable" wording; harness `runtime2/` renamed to `runtime/` (44 files). 2026-09-02 `e26afb63a` settled tool calls retained in the lane snapshot until placement; `tool_end` moved after the staging commit → [[settled-tool-vanishes-before-placement]]. Both in the since-removed agent-core harness (`7fd478a2e`).
- 2026-08-26 `353c990f4` "mini" three-process agent to learn what an RPC presentation needs.
- 2026-09-07 `73f3257dd` Pico spec-first redesign; 2026-09-14 `46b66c59a` pico3 kernel; 2026-09-15–17 Pico5 spec set (`729d5cb74` −40.5k lines); 2026-09-18 `080160162` `packages/durable`.
- 2026-09-22–23 `8158b0321` versioned docs, `5901c9b9e` SQLite, `898ab8040` JSONL; 2026-09-24 `19a0361be` transactional sessions/docs, `5d4de953c` checkpoints + migrations.
- 2026-09-29 Packages 15–18 (`837d8a16b` first chat turn, `445770e03` first tool turn, `47f65f1a3` inbox/reset/usage/view/events, `2532a0bef` ownership + subagents), each preceded by a `docs(durable): specify …` commit.
- 2026-09-30 `03180653c` structured concurrency, `ed0d6b91b` compaction/overflow, `f3e68e8ea` async SQLite, `ed391c4f0` queue/close hardening.
- 2026-10-01 `5609b0d6c` durable TUI, `48dd1e2f0` client/server ported, `7fd478a2e` old harness removed from pi-agent-core (105k lines), v1.0.0.
- 2026-10-04 `70eceaade` persisted provider session UUIDv7; 2026-10-06 `68ccef176`/`da866ada1` context caching + DO SQLite, `92216fa15`, `4dd2af42c`, `36a686ee8`, `76f6c06da`.
- Built by agents from an ordered handoff: "Implement this list in order. After every package: run its tests, run `npm run check`, and stop for user review." (`docs/pico-v5-handoff.md:3-5`) → [[spec-driven-agentic-development]].

## Evidence commits
`a5b27367d`, `73f3257dd`, `080160162`, `5901c9b9e`, `898ab8040`, `19a0361be`, `5d4de953c`, `2532a0bef`, `03180653c`, `ed391c4f0`, `48dd1e2f0`, `7fd478a2e`, `70eceaade`, `da866ada1`

## Quirks
- SQLite backend is written against a portable async core (`SqliteValue = null|number|bigint|string|Uint8Array`; `exec/run/get/all` shared by db and transaction handles; adapters may cache prepared statements by SQL text) so node/bun/Cloudflare DO drivers plug in (`packages/durable/src/storage/sqlite/database.ts:1-12`).
- Even side-effect-free `read` is `replay:"unsafe"` → interrupted reads become "may have partially run" errors (open question).
- JSONL `fsync` off by default — durability against process crash only, not power loss; structural footguns documented as contracts, not guarded (`spec.md:4596-4731`).
- A `blocked` task keeps a worker alive forever (worker retires only with no live tasks) — unverified.
- Missing pieces vs stable pi in the durable TUI: session picker, forks/tree navigation, extensions, prompt templates, images, `/login` (`experimental/durable/README.md:66`).

## Failures
- [[storage-queue-close-races]] · [[per-request-projection-rescans-log]] · [[output-window-depends-on-commit-cadence]] · [[settled-tool-vanishes-before-placement]]
