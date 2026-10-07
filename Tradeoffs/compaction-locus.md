---
type: tradeoff
concepts: [auto-compaction, structured-compaction-summary, iterative-summary-update, compaction-cut-point, transcript-serialization-for-summary, summary-validation, overflow-recovery, model-requested-context-reset]
---
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
