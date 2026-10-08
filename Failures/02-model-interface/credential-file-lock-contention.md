---
type: failure
concepts: [subscription-oauth-auth, credential-resolution]
harnesses: [pi]
---
**Symptom** — Shared `auth.json` misbehaved across several pi instances:
- During parallel startup, users saw "No API key found".
- A compromised lock crashed the process.
- Long sessions used stale credentials after another process updated `auth.json`.
- Once every read took the lock, credential reads were serialized and startup was slow.
- `/login` claimed success while `auth.json` was locked.

**Root cause**
- `lockSync` failed immediately on ELOCKED, and the failure was read as an empty file.
- proper-lockfile's `onCompromised` callback threw asynchronously.
- The in-memory cache never reloaded.
- Lock-per-read caused lock convoys.
- Save failures were swallowed.

**Fix · [[pi]]**
- `ddd5a65c7` 2026-02-06 — capture the compromised flag and throw at checkpoints (before read, after fn, after write) (#1322) (`packages/coding-agent/src/core/auth-storage.ts:157-200`).
- `58f8fcd8f` 2026-03-06 — `lockSync` retried 10× with a 20 ms busy-wait (#1871) (`auth-storage.ts:69-94`).
- `135fb545f` 2026-06-03 — `0o600` applied only on creation, "so administrator-managed modes and ACLs remain intact" (`:24-25,63-67`).
- `f8bec25f3` 2026-07-01 — surface save failures to `/login` (#6223).
- `57cde8690` 2026-07-31 — reload under the lock before reads (#7319).
- `d2be68dbe` 2026-07-31 — stat-revision fast path `dev:ino:size:mtimeNs:ctimeNs` (`packages/coding-agent/src/utils/paths.ts:36-43`).
- `2c79ce453` 2026-08-05 — single-flight reload shared by readers, aborted when the last reader leaves (`auth-storage.ts:39,401-439`).
- `7cf90c1d1` 2026-08-05 — same pattern for `models-store.json` (`packages/coding-agent/src/core/models-store.ts:23-118`).
- `4d68d9355` 2026-08-05 — async backoff `min(10·2^n, 1000)·(1+rand)`, stale 30 s, deadline 30 s (`auth-storage.ts:116-155`).

**Lesson** — Lock contention must never be read as "no data". Use cheap revision checks plus single-flight reloads instead of lock-per-read, and never report a write you didn't verify.

Related: [[subscription-oauth-auth]] · [[credential-resolution]] · [[pi--credential-resolution|pi]] · [[oauth-refresh-token-rotation-lost]]
