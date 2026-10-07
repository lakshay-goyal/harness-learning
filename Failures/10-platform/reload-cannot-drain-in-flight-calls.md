---
type: failure
concepts: [runtime-plugin-loading]
harnesses: [pi]
---
**Symptom** — Chord's facet reload contract promised to "drain admitted calls" on the old generation before cutover; the contract couldn't be honored (calls could hang the reload or race the swap), and partial rollback after cutover left mixed generations.

**Root cause** — Draining requires knowing when arbitrary plugin work ends; rollback after providers were replaced cannot restore consumers that already rebound.

**Fix · [[pi]]**
- `c4b0e35ab` 2026-09-01 — services stay available during reloads: candidates activate while old providers stay routed, then `provision.replace()` per singleton (no unavailable gap) (`packages/chord/src/facets/host.ts:423-511`).
- `5dd8c0132` 2026-09-01 — "every singleton switches directly… any failure after replacement begins terminates the host"; running work in retired facets is not drained (`PLANNING.md:227,255`; `host.ts:506-508` "Facet reload failed after cutover").
- Stable pi sidesteps the problem: `/reload` is a full `session_shutdown` → rebuild → `session_start` (`packages/coding-agent/src/core/agent-session.ts:3659-3697`).

**Lesson** — Prefer an explicit cutover point with fail-fast terminal state over drain-then-swap or partial rollback.

Related: [[runtime-plugin-loading]] · [[module-cache-retains-plugin-generations]] · [[no-auto-hot-reload]] · [[pi--runtime-plugin-loading|pi]]
