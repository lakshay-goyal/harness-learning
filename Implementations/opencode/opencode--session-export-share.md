---
type: implementation
harness: opencode
concept: session-export-share
commit: ecc4916b5a
files: [packages/opencode/src/cli/cmd/export.ts:11-17, packages/opencode/src/cli/cmd/export.ts:163-234, packages/opencode/src/share/share-next.ts:23, packages/opencode/src/share/share-next.ts:124-146, packages/opencode/src/share/share-next.ts:206-262, packages/opencode/src/share/share-next.ts:274-356, packages/opencode/src/share/session.ts:26-45, packages/core/src/share/sql.ts:1-13, packages/opencode/src/cli/cmd/github.handler.ts:515-518]
---
[[session-export-share]] in [[opencode]].

## Mechanism
### Legacy runtime (`ShareNext`)
- **Create**: POST `/api/share` (or `/api/shares` for a console org account) → `{id, secret, url}` stored in `session_share` (`packages/opencode/src/share/share-next.ts:310-335`), then a **full sync** of session info, every message, every part (tool inputs and outputs, so file contents the agent read), `session_diff` (snapshot file diffs) and model metadata (`packages/opencode/src/share/share-next.ts:274-300`). No redaction pass (no `redact` in the file) ([[secret-handling]]).
- **Live mirror**: subscribes to session updated, message updated, part updated and session diff events; queues items per session in a Map keyed `session` / `message/<id>` / `part/<msgID>/<id>` / `session_diff` / `model` (later updates overwrite earlier: last-write-wins coalescing); first enqueue schedules a flush after 1000 ms that POSTs `{secret, data[]}` to `/api/share/<id>/sync` (`packages/opencode/src/share/share-next.ts:124-146`, `packages/opencode/src/share/share-next.ts:248-262`).
- **Target/auth**: `enterprise.url ?? "https://opncd.ai"`, authorized only by the per-share secret (`packages/opencode/src/share/share-next.ts:210`); with an active console org, `Authorization: Bearer <account token>` + `x-org-id`.
- **Privacy controls**: default manual (`/share`); `share: "auto"` (or deprecated `autoshare`, or `OPENCODE_AUTO_SHARE`) shares every new **root** session, never subagent children (`packages/opencode/src/share/session.ts:39-45`); `share: "disabled"` makes `share()` throw (`packages/opencode/src/share/session.ts:28`); `OPENCODE_DISABLE_SHARE=1` turns every path into a no-op (`packages/opencode/src/share/share-next.ts:23`). `unshare` and session deletion DELETE the remote share (`packages/opencode/src/share/share-next.ts:338-356`).
- **GitHub agent**: shares by default only on public repos (`share` unset and repo private → no share) and links the share in its comment footer (`packages/opencode/src/cli/cmd/github.handler.ts:515-518`) ([[ci-agent-integration]]).
- **Local export**: `opencode export [sessionID]` writes session info + messages + parts as JSON; `--sanitize` replaces every content-bearing field (text, reasoning, tool raw/output/title, file paths/URLs, subtask prompts, snapshots, patches, system prompt, cwd/root, session title/directory) with `[redacted:<kind>:<id>]` placeholders, keeping structure for bug reports (`packages/opencode/src/cli/cmd/export.ts:11-17`, `packages/opencode/src/cli/cmd/export.ts:163-214`, `packages/opencode/src/cli/cmd/export.ts:223-234`; `3695057bee` 2026-04-14). `opencode import <file>` reads it back (`packages/opencode/src/cli/cmd/import.ts:95`). No HTML viewer.

### v2 runtime
- `packages/core/src/share/sql.ts` defines only the table (cascade delete with the session); no v2 share sync yet.

## Constants
| name | value | path:line |
|---|---|---|
| flush delay | 1000 ms | `packages/opencode/src/share/share-next.ts:142` |
| default share host | `https://opncd.ai` | `packages/opencode/src/share/share-next.ts:210` |

## Evolution
- 2025-11-21 `49408c00e9` enterprise (#4617): `share-next` added.
- 2025-11-24 `69d1381ba3` share IDs separated from session IDs.
- 2026-04-14 `3695057bee` `opencode export --sanitize` (#22489).

## Quirks / drift
- A share is a continuously updated mirror, not a snapshot: everything the session does after `/share` is uploaded until `unshare`.

pi contrast: `/export` to self-contained HTML/JSONL, `/share` as a private gist or org artifact, both unredacted; opencode's only redaction is the opt-in `export --sanitize` ([[pi--session-export-share|pi]]).
