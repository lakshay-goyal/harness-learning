---
type: absence
harnesses: [codex]
---
# no-cache-miss-detection

(unverified — grep evidence only) Cache read/write tokens are reported, but no per-turn cache-miss detector or warning exists.

**What's missing**
- `cached_input_tokens` / cache-write tokens reported to telemetry/analytics (`codex-rs/otel/src/events/session_telemetry.rs:1139`); searching `cache_miss|cache_hit` across `codex-rs` finds no detector.

**Evidence of decision**
- Absence by omission; effort goes into *preventing* misses instead — stable ids, pinned effort, session-id affinity, append-only world-state diffs ([[cache-stable-prompt-prefix]], [[session-affinity-cache-routing]], [[world-state-diff-injection]]).

**Implication**
- Prefix-breaking regressions surface only as cost/latency, found by reading telemetry ([[nondeterministic-tool-order-breaks-cache]], [[permission-context-reinjected-repeatedly]]).

Related: [[cache-stable-prompt-prefix]] · [[cache-strategy]] · [[install-telemetry]] · [[Absences]]
