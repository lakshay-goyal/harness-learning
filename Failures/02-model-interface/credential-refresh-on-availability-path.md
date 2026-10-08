---
type: failure
concepts: [subscription-oauth-auth, model-catalog]
harnesses: [pi]
---
**Symptom**
- The `/model` selector was slow because the availability check triggered an OAuth token refresh for each provider.
- pi failed to start when one provider's OAuth refresh failed during model discovery.

**Root cause** — "Is this provider configured?" was conflated with "get a fresh token", and a refresh failure propagated as a throw.

**Fix · [[pi]]**
- `17b3a14bf` 2026-01-03 — the availability check reads config without refreshing; the refresh happens at use time.
- `d2f3b42de` 2026-01-06 — a refresh failure returns undefined, so the app starts and the user can `/login`.
- HEAD: a refresh failure raises `ModelsError("oauth")` with the stored credential preserved for re-login (`packages/ai/src/models.ts:302-305`). `getAvailable()` uses only the auth check (`models.ts:702-739`).

**Lesson** — Separate "configured" from "fresh token". An auth failure for one provider must not brick startup or block the picker.

Related: [[subscription-oauth-auth]] · [[model-catalog]] · [[availability-snapshot-races]] · [[pi--subscription-oauth-auth|pi]]
