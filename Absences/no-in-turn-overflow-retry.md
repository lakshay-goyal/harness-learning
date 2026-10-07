---
type: absence
harnesses: [codex]
---
# no-in-turn-overflow-retry

A request that overflows the context window in a normal session is not compacted-and-retried; the turn fails, usage is pinned to full, and the next turn compacts first.

**What's missing**
- `codex-rs/core/src/session/turn.rs:1688-1691`, `codex-rs/core/src/session/turn.rs:816-831`. Only guardian reviews retry once (`codex-rs/core/src/session/turn.rs:758-787`).

**Evidence of decision**
- Hardening attempt reverted: `15e79f3c26` → `69f3183a8e` 2026-05-11.

**Implication**
- Proactive thresholds (90% auto-compact, 95% effective window, optional post-turn compaction) replace reactive recovery; contrast pi's [[overflow-recovery]] compact-and-retry-once.

Related: [[overflow-recovery]] · [[auto-compaction]] · [[context-overflow-detection]] · [[Absences]]
