---
type: failure
concepts: [session-tree, context-projection]
harnesses: [pi]
---
**Symptom** — Long sessions got progressively slower or crashed: deep-branch path construction quadratic (600k-entry session: ~20.3 s); prompt submission slowed with session length because model selection looked up the catalog once per assistant message (#10198); `--resume` listing OOM on large histories (#4583); opening very large JSONL files materialized the whole file as one string (#5231); EventStream queue drain quadratic (#9055).

**Root cause** — Per-turn walks written as O(n²) (`Array.unshift` per step, per-message lookups) or whole-file buffering; acceptable at 100 entries, pathological at 10⁵.

**Fix · [[pi]]**
- `a1da88aed` 2026-06-20 (refs #5804/#5909) "make session path traversal linear": `push()` + `reverse()` — ~20.3 s → ~35 ms (`packages/coding-agent/src/core/session-manager.ts:390-416`).
- 0.80.x (#5231) — line-by-line loading in 1 MiB chunks (`session-manager.ts:604,626-670`; `packages/coding-agent/CHANGELOG.md:1738`).
- (#4583) — cap in-flight session metadata loads, `MAX_CONCURRENT_SESSION_INFO_LOADS = 10` / discovery 64 (`:891-892`; `packages/coding-agent/CHANGELOG.md:1996`).
- (#10198) — model selection no longer resolved per assistant message (`packages/coding-agent/CHANGELOG.md:207`).
- `b2602be77` 2026-09-07 (#9055) — two-stack FIFO EventStream queue (see [[quadratic-event-queue-drain]]).
- pi-durable analogue: [[per-request-projection-rescans-log]].

**Lesson** — Agent sessions grow to 10⁴–10⁶ entries; every per-turn or per-request walk must be O(n) or cached, and loaders must stream.

Related: [[session-tree]] · [[context-projection]] · [[quadratic-event-queue-drain]] · [[pi--session-tree|pi]]
