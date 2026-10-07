---
type: failure
concepts: [cache-warming]
harnesses: [pi]
---
**Symptom** — Idle cache-warming refreshes sometimes fired after the provider cache had already expired, so the "warm" request paid a full cache write (rebuilding the entry) instead of a cheap cache read — the keep-alive cost more than it saved.

**Root cause** — Refresh timers scheduled at `min(0.9·TTL, TTL−10s)` can run late after laptop sleep, event-loop blockage, or a slow async `cache_warming_decision` extension hook; nothing checked whether the cache was still alive before sending.

**Fix · [[pi]]** — `3390bd936` 2026-09-20 "skip late cache warming refreshes" (one day after the feature `c596d09d9` #9668): `refreshDeadlineAt = nextWarmAt + floor((TTL − delay)/2)` — "Keep half of the planned pre-expiry margin for that delay and request dispatch; a late refresh is likely a full-price cache write, not a cache warm" (HEAD `packages/coding-agent/src/core/cache-warmer.ts:284-298`); checked before and after the extension decision, missed → stop with reason "cache refresh deadline missed" (`:300-303, 318, 357-361`).

**Lesson** — A keep-alive that fires after its deadline is a full write, not a warm; check the deadline at send time and stop rather than refresh late.

Related: [[cache-warming]] · [[cache-retention-control]] · [[usage-cost-accounting]] · [[pi--cache-warming|pi]]
