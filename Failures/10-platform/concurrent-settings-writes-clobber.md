---
type: failure
concepts: [layered-settings]
harnesses: [pi]
---
**Symptom** — An invalid `settings.json` was overwritten with empty settings (#1054); saving settings from one pi instance reverted external edits or changes made by another concurrent pi process (#527).

**Root cause** — Save wrote the full in-memory settings object back to disk (last-writer-wins on the whole file), and a parse failure was treated as "no settings".

**Fix · [[pi]]**
- 0.50.4 (#1054) — invalid JSON no longer overwritten (`packages/coding-agent/CHANGELOG.md:3859`).
- `9dfe5bf89` 2026-01-07 (PR #527) preserve settings on save → HEAD: writes only fields (and nested keys) modified this session, merged onto the current on-disk content, under `proper-lockfile` 10 × 20 ms (`packages/coding-agent/src/core/settings-manager.ts:327-351,728-757`).
- Same pattern for credentials and trust: `auth.json` / `trust.json` locked writes (see [[credential-file-lock-contention]], [[oauth-refresh-token-rotation-lost]]).

**Lesson** — Shared config files edited by several processes (and humans) need lock + read-merge-write of only modified fields; never overwrite a file you couldn't parse.

Related: [[layered-settings]] · [[credential-file-lock-contention]] · [[secret-handling]]
