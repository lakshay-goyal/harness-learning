---
type: concept
stage: context
tier: variant
aliases: [prune, "compaction.prune", SessionCompaction.prune, PRUNE_PROTECT, PRUNE_MINIMUM, PRUNE_PROTECTED_TOOLS, time.compacted, microcompact, "[Old tool result content cleared]", OPENCODE_DISABLE_PRUNE]
harnesses: [opencode]
---
Old tool-result bodies beyond a recent-token window are erased and replaced with a placeholder, as cheap relief before or instead of summarization.

## Why
- Tool outputs dominate long coding sessions; most old ones (file reads, grep dumps) are dead weight once acted on, and dropping them costs no LLM call.
- Summarization is lossy for everything; pruning is lossy only for bulky, re-fetchable tool output.
- Every prune rewrites earlier history, so it busts the provider prompt cache from the first pruned part onward; small gains are not worth a cache miss (opencode `PRUNE_MINIMUM`), and the trade turned negative enough that opencode switched it off by default.
- Some tool results are instructions, not data (loaded skill bodies); pruning them silently removes behavior (opencode `PRUNE_PROTECTED_TOOLS = ["skill"]`, `046e351140`).

## Design space
- **When**: after each run, off the critical path (opencode legacy, forked after the loop) · before each request · only under context pressure.
- **Window**: protect the newest N tokens of tool output (opencode 40k) and the last K user turns (opencode 2) · protect by count of results.
- **Minimum gain**: only prune when ≥ M tokens are freed (opencode 20k) to amortize the cache bust.
- **Stop condition**: stop at the latest summary or the first already-pruned part, so each run only scans new history (opencode).
- **Storage**: mark the part (`time.compacted`) and keep the bytes in the DB, render a placeholder at request build (opencode legacy) · delete the output · append an overlay edit ([[context-edit-overlay]], pi's mechanism for other edits).
- **Protected tools**: allow-list of results never pruned (opencode: `skill`).
- **Default**: on (opencode 2025-09 → 2026-04) · **off, opt-in** (opencode since 2026-04) · absent (pi: "OpenCode-style prune" considered in plan, not adopted — see [[auto-compaction]] design space) · deferred (opencode v2: "Deterministic old tool-result pruning remains a separate follow-up", `specs/v2/session.md:121`).
- **Provider-executed results**: cannot be pruned generically because some providers need exact round-trip payloads (opencode v2 `CONTEXT.md:199`).

## Implementations
- [[opencode--tool-output-pruning|opencode]] — legacy `SessionCompaction.prune`: newest-first walk skipping 2 user turns, protect 40k tokens of output, prune only if > 20k freed, `skill` exempt, sets `time.compacted`; rendered as "[Old tool result content cleared]"; default off since 2026-04; absent in v2.

## Failures
- (none recorded; the default flip `6eddf08244` has no stated rationale)

## Tradeoffs
- [[compaction-design]]

## Related
[[auto-compaction]] · [[context-edit-overlay]] · [[context-transform-hook]] · [[tool-output-truncation]] · [[cache-stable-prompt-prefix]] · [[transcript-serialization-for-summary]] · [[skill-progressive-disclosure]]
