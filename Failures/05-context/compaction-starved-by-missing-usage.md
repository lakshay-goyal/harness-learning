---
type: failure
concepts: [token-estimation, auto-compaction]
harnesses: [pi]
---
**Symptom** — Sessions stuck on persistent 529/overloaded errors never compacted; providers that don't stream usage (or return all-zero usage) never reached the threshold, so the session grew until a hard overflow.

**Root cause** — The threshold check read usage from the *latest* assistant message; error and zero-usage messages carry none, so the check returned false.

**Fix · [[pi]]**
- `b8910f13a` 2026-03-06 (#1834): for error messages estimate from the last successful usage + trailing chars/4.
- `4495469a5` 2026-08-19 (#8328): no usage at all → pure message-size estimate (previously `return false`) (`packages/coding-agent/src/core/agent-session.ts:3048-3079` at HEAD; comment (`:3050`) "sessions that hit persistent API errors (e.g. 529) or malformed zero-usage responses can still compact").

**Lesson** — Context accounting needs an estimator fallback that never depends on the newest response having usage.

Related: [[token-estimation]] · [[auto-compaction]] · [[stale-usage-drives-compaction]] · [[streamed-usage-misread]]
