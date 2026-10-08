---
type: failure
concepts: [replicated-state]
harnesses: [pi]
---
**Symptom** — (a) Proxy bookkeeping after a full traversal of a 139.5 MiB document retained ~3.5 GiB; (b) a dissimilar "layer swap" exceeding the 4,096-op limit turned a small logical change into a ~66 MB whole-root replacement broadcast to every replica.

**Root cause** — (a) strong per-node proxy caches kept every visited node alive; (b) op-count fallback (`MAX_DELTA_OPERATIONS` 4096 → root `r`, `packages/chord/src/delta/tracker.ts:112,1707`) is not byte-aware.

**Fix · [[pi]]** — (a) `58541ee72` 2026-09-18 weak proxy caches + compact clones: retained 3,526 → 204 MiB, but cold traversal slowed 2.4 s → 6.3 s (`packages/durable/docs/chord-delta-findings.md:228-237`); ID-addressed graph tracker rejected (ready heap 430 vs 139.5 MiB; import 1,119 ms vs 0.02 ms, `:182-192`). (b) **open at HEAD** (`chord-delta-findings.md:84-87`). Conclusion: "No measured design … simultaneously achieved low retained and transient memory, cheap ordinary reads, cheap large reference reassignment, and the desired broad mutable-JavaScript semantics" (`:356-358`).

**Lesson** — Benchmark exhaustive reads and post-GC retention, not just mutation/flush; size fallbacks need byte awareness, not op counts.

Related: [[replicated-state]] · [[pi--replicated-state|pi]]
