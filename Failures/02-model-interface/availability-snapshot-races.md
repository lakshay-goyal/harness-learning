---
type: failure
concepts: [model-catalog]
harnesses: [pi]
---
**Symptom**
- A forced availability refresh got stuck behind a stalled earlier refresh.
- Login hung after the credentials were already saved, because the post-login catalog refresh stalled.
- A hung pi.dev catalog request consumed the whole refresh deadline.
- At startup, pi picked the wrong provider or reported "no models available" for a native extension provider that had stored credentials.

**Root cause**
- Refreshes were chained on a promise.
- Network freshness was coupled to committing local credential state.
- One timeout covered all attempts.
- The synchronous snapshot was updated only after an async auth check, while startup selection read it first.

**Fix · [[pi]]**
- `8f9e76974` 2026-08-01 — independent rebuilds with a monotonic `availabilityRefreshSeq`; stale passes are dropped (#7301) (`packages/coding-agent/src/core/model-runtime.ts:349,366-379`).
- `c6eb6281a` 2026-08-03 — bound the post-login catalog refresh; `synchronizeCredentialState` refreshes with `allowNetwork:false` and wraps errors in `CredentialSynchronizationError` (#7027) (`model-runtime.ts:136-153,592-612`).
- `fed6009cc` 2026-08-04 — caller-owned cancellation; generation-checked `context.publish` (`packages/coding-agent/src/core/remote-catalog-provider.ts:80-86,150-155`).
- `df018b602` 2026-08-17 — per-attempt timeout `attemptTimeoutMs` 4 s plus retries, separate from the overall budget (`remote-catalog-provider.ts:14,103-114`; `packages/coding-agent/src/utils/management-http.ts:27-35`).
- `fddc968b9` 2026-09-29 — `markProvisionallyConfigured` on registration (#9962) (`model-runtime.ts:898-919`).

**Lesson** — Use generation counters, not promise chains, for refresh pipelines. Commit local state first and treat network freshness as bounded, cancellable best-effort. Synchronous snapshots need provisional updates at registration.

Related: [[model-catalog]] · [[pi--model-catalog|pi]] · [[credential-resolution]] · [[catalog-layer-precedence-errors]]
