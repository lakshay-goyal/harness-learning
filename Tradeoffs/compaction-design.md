---
type: tradeoff
concepts: [auto-compaction, compaction-cut-point, structured-compaction-summary, transcript-serialization-for-summary, tool-output-pruning, overflow-recovery, split-turn-summary]
harnesses: [pi, opencode]
---
# compaction-design

**Axis**: when the context fills, how is history shrunk: when, how much is kept verbatim, what the summarizer sees, and whether cheap pruning runs first?

| dimension | pi | opencode | evidence |
|---|---|---|---|
| Trigger | `ctx > window − 16384` (fixed headroom) | legacy `count ≥ limit.input − min(20000, maxOutput)` (else `context − maxOutput`); v2 pre-request estimate `> context − max(output, 20000)` | pi `packages/coding-agent/src/core/compaction/compaction.ts:267-270`; opencode `packages/opencode/src/session/overflow.ts:8-34`; `packages/core/src/session/compaction.ts:232-241` |
| Unknown context window | assume 128000 / 16384 | `limit.context ?? 0` → compaction disabled, rely on overflow error | pi `packages/coding-agent/src/core/provider-composer.ts:242-243`; opencode `packages/opencode/src/session/overflow.ts:29` |
| Kept verbatim | `keepRecentTokens = 20000` (fixed), cut may split a turn (prefix summarized separately) | legacy `clamp(0.25 × usable, 2k, 15k)`, may split; v2 `8000` | pi `packages/coding-agent/src/core/compaction/compaction.ts:129` → [[split-turn-summary]]; opencode `packages/opencode/src/session/compaction.ts:32-33,115-119`; `packages/core/src/session/compaction.ts:13` |
| Summarizer input | serialized transcript, tool results capped 2000 chars | same: tagged text lines, 2000 chars (`TOOL_OUTPUT_MAX_CHARS`) | pi `packages/coding-agent/src/core/compaction/utils.ts:94`; opencode `packages/opencode/src/session/compaction.ts:30,51-85` → [[transcript-serialization-for-summary]] |
| Summary format | structured checkpoint template | anchored template (Objective / Important Details / Work State / Next Move / Relevant Files) + `<prior-summary>` merge | → [[structured-compaction-summary]], [[iterative-summary-update]] |
| Summary budget | 0.8 × reserve (13107) | legacy: normal output (≤ 32k); v2 4096 | pi `packages/coding-agent/src/core/compaction/compaction.ts:712-713`; opencode `packages/core/src/session/compaction.ts:15` |
| Cheap pruning before summarizing | none in core (context-edit overlay is a different mechanism) | legacy prune: keep newest 40k tool output, act only if > 20k freed, never `skill`; **off by default** since April 2026; v2: not built | `packages/opencode/src/session/compaction.ts:28-31,275,290-308` → [[tool-output-pruning]] |
| Overflow recovery | once per user turn | legacy replays the last user turn after compaction; v2 once, never after durable output | pi `6b4b92042` → [[overflow-recovery]]; v2 `specs/v2/session.md:121` |
| Where the summary lives | `compaction` entry in the session tree | legacy: user message with `compaction` part + assistant `summary: true`; v2: checkpoint user message "historical context, not as new instructions" | `packages/core/src/session/runner/to-llm-message.ts:153` |

**When each wins**
- **Fixed keep window (pi)**: predictable cost per compaction; fine on 200k windows. On 1M windows it compacts at ~98% and keeps the same 20k.
- **Scaled keep window (opencode legacy)**: proportional to the model, capped at 15k so the summary stays the main carrier. v2 shrinks further (8k + 4k summary) to keep the post-compaction prefix small.
- **Pruning first**: cheap and lossy-but-recoverable, yet it rewrites old tool results and invalidates the cache prefix; opencode turned it off by default (reason unverified). Summarize-only (pi) keeps the prefix stable until the one big cut.

Related: [[auto-compaction]] · [[prompt-cache-strategy]] · [[Constants]] · [[pi]] · [[opencode]] · [[Tradeoffs]]
