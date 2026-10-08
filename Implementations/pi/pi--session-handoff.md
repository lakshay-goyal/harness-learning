---
type: implementation
harness: pi
concept: session-handoff
commit: b30a6dd77
files: [packages/coding-agent/examples/extensions/handoff.ts:20, packages/coding-agent/examples/extensions/handoff.ts:57, packages/coding-agent/examples/extensions/handoff.ts:131, packages/coding-agent/examples/extensions/handoff.ts:177, packages/durable/src/harness/harness.ts:110, packages/durable/src/harness/generation.ts:618]
---
[[session-handoff]] in [[pi]] — example extension (stable) + first-class `reset(handoff)` / tool `control:{handoff}` (durable).

## Mechanism
- `/handoff <goal>` command (`packages/coding-agent/examples/extensions/handoff.ts:80-189`): rationale "Instead of compacting (which is lossy), handoff extracts what matters for your next task and creates a new session with a generated prompt" (`:1-13`) → alternative to [[auto-compaction]].
- Guards: TUI mode only, model selected, non-empty goal (`:84-98`).
- Context gathering `getHandoffMessages(branch)` (`:57-78`): if branch compacted → [latest compaction summary as `compactionSummary` msg] + entries from `firstKeptEntryId` up to compaction + all after; else whole branch (message + compaction entries only) (`2fa7c2ef2` "use compacted context for handoff").
- Serialize: `convertToLlm(messages)` → `serializeConversation()` (tagged text) (`:110-111`) → [[transcript-serialization-for-summary]].
- Side LLM call `ctx.modelRegistry.complete(ctx.model, {systemPrompt: SYSTEM_PROMPT, messages:[user]}, {signal: loader.signal, cacheRetention:"none", sessionId: uuidv7()})` (`:131-139`) → [[cache-retention-control]] (fresh session id + no cache write so the one-off request doesn't pollute the main cache, `241431c69` #6618). User msg = `## Conversation History\n\n<text>\n\n## User's Goal for New Thread\n\n<goal>` (`:125`). Aborted → null → "Cancelled".
- **SYSTEM_PROMPT** (`:20-40`): "You are a context transfer assistant… generate a focused prompt that: 1. Summarizes relevant context (decisions made, approaches taken, key findings) 2. Lists any relevant files… 3. Clearly states the next task… 4. Is self-contained - the new thread should be able to proceed without the old conversation… Do not include any preamble"; example format `## Context` (decisions, files) + `## Task`.
- **Human in the loop**: generated prompt opened in `ctx.ui.editor("Edit handoff prompt", …)` (`:167`); cancel → abort handoff.
- New session `ctx.newSession({parentSession: currentSessionFile, withSession})` — lineage via header `parentSession` ([[session-fork]]); prompt placed as **editor draft**, not auto-sent ("Handoff ready. Submit when ready.") using the replacement ctx because the original ctx is stale after replacement (`:174-183`; `f0cf8a59d`) → [[runtime-plugin-loading]].
- Subagent analogue: scout role writes for "an agent who has NOT seen the files you explored" (`examples/extensions/subagent/agents/scout.md:10`) → [[pi--subagent-as-subprocess|subagent-as-subprocess]].

## Constants
| name | value | path:line |
|---|---|---|
| side-call cache retention | `"none"` | `handoff.ts:136` |
| side-call session id | fresh `uuidv7()` | `handoff.ts:137` |

## Evolution
- 2026-01-02 `ace0063a0` handoff example hook; `91c52de8b` use `serializeConversation`.
- 2026-01-05 `c6fc08453` hooks → unified extensions (#454); 2026-01-12 `783aa0d6d` fix for PR #642.
- 2026-03-27 `7a786d88a` resolve models.json auth per request (#1835).
- 2026-04-23 `f0cf8a59d` stale extension contexts; 2026-04-29 `2fa7c2ef2` compacted context.
- 2026-06-01 `e56521e32` extension mode context (`ctx.mode`).
- 2026-07-23 `241431c69` no cache write for compaction/branch summaries (#6618); 2026-07-31 `ab366ebe9` custom compaction via model runtime `complete` (#7325); 2026-08-04 `e741cb05c` preserve extension auth endpoints.

## Evidence commits
`ace0063a0` `91c52de8b` `c6fc08453` `783aa0d6d` `7a786d88a` `f0cf8a59d` `2fa7c2ef2` `e56521e32` `241431c69` `ab366ebe9` `e741cb05c`

## Quirks
- Compaction itself was framed as a handoff: v1 compaction prompt "Create a handoff summary for another LLM that will resume the task" (`6c2360af2`, #92) → [[structured-compaction-summary]].
- Non-TUI modes unsupported (`handoff.ts:84-87`).

## Durable variant (packages/durable)
- `reset(handoff?)` starts a new context: older entries stay in storage but leave model view; appends `pi.reset` entry `head:"self"` with the handoff text as a user message (`packages/durable/src/harness/harness.ts:110-114`; `packages/durable/README.md:325-334`; `spec.md:873`, `3337`). While busy, a reset is queued like a write; placed during a tool round it ends the run (`unanswered/reset`, `generation.ts:633-637`).
- Tool result `control:{handoff:"…"}` ends the run the same way; last handoff in call order wins (`spec.md:3081-3089`; `generation.ts:618-628`) — a model-callable handoff, unlike stable pi's user command.

## Failures
none recorded.
