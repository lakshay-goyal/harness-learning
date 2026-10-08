---
type: tradeoff
concepts: [auto-compaction, compaction-cut-point, structured-compaction-summary, transcript-serialization-for-summary, tool-output-pruning, overflow-recovery, split-turn-summary, iterative-summary-update, summary-validation, model-requested-context-reset]
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

## Also: [[codex]] (folded from `compaction-locus`)
Where and how history gets compressed: a harness-authored local summary, an opaque provider-side compaction, or a model-requested reset with no summary.

| dimension | pi — local structured summary | codex — local handoff summary (fallback) | codex — remote server compaction (preferred) | codex — model-requested reset (token budget) |
|---|---|---|---|---|
| Who summarizes | session model, harness prompt ([[pi--auto-compaction]]) | session model; previous model on model switch ([[codex--auto-compaction]]) | provider: history + `CompactionTrigger` → one encrypted `Compaction` item (`codex-rs/core/src/compact_remote_v2_attempt.rs:84-103`) | nobody ([[codex--model-requested-context-reset]]) |
| Prompt | 6-section EXACT checkpoint template ([[pi--structured-compaction-summary]]) | 9-line free-form "CONTEXT CHECKPOINT COMPACTION … handoff summary" (`codex-rs/prompts/templates/compact/prompt.md:1-9`) | none client-side | n/a |
| Summarizer input | serialized transcript in `<conversation>`, tool results ≤2000 chars, anti-continuation system prompt ([[pi--transcript-serialization-for-summary]]) | full chat history + prompt as synthetic user message | full normalized history, tool outputs pre-trimmed if over window | n/a |
| Kept verbatim | recent mixed tail (20k tokens) at a legal cut ([[pi--compaction-cut-point]]) | user messages only, 20k tokens ([[codex--compaction-cut-point]]) | user/hook/selected agent messages, 64k, images budgeted | fresh initial context only |
| Summary framing | user message with preamble | third-person "Another language model started to solve this problem…" (`summary_prefix.md`) | opaque item | n/a |
| Repeated compaction | iterative update prompt + `<previous-summary>` ([[pi--iterative-summary-update]]) | old summaries excluded from kept set, re-summarized as history | server-side (opaque) | n/a |
| Validation | reject error/length/tool calls ([[pi--summary-validation]]) | empty → "(no summary available)"; post-turn requires non-empty ([[codex--summary-validation]]) | exactly one compaction item | n/a |
| Continuity aids | `<read-files>`/`<modified-files>` ([[pi--file-op-tracking]]) | canonical context re-injected | canonical context re-injected | notes + searchable history tools |
| Trigger | window − 16384 before every request | 90 % of window; PreTurn/MidTurn/PostTurn/model switch | same | model call `new_context` or budget exhausted |
| Overflow | compact + retry once ([[pi--overflow-recovery]]) | fail turn, compact next turn ([[codex--overflow-recovery]]) | same | fallback prompt then forced reset |
| Inspectability | summary text stored and readable | readable | **not readable by client or user** | n/a |

**When local wins** — any provider, auditable summaries, harness controls what survives (exact paths, decisions), works offline/with third-party models; cost: prompt engineering per failure mode ([[summarizer-continues-conversation]], [[summary-template-drops-goals]]).

**When remote wins** — the vendor also trains the model and the compaction format (encrypted, model-native state), fewer client prompt failures, cache-friendly; cost: opaque, provider-locked, needs retained-context contracts for harness state ([[compaction-drops-harness-state]]).

**When model-requested reset wins** — long autonomous sessions where the model knows when a window is spent and can persist state itself; cost: needs backend notes/history tools and model training for the protocol (codex: under development, ChatGPT plans only).
