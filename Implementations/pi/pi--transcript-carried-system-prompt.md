---
type: implementation
harness: pi
concept: transcript-carried-system-prompt
commit: b30a6dd77
files: [packages/ai/src/types.ts:528-544, packages/ai/src/utils/transcript.ts:10-237, packages/ai/src/utils/text.ts:23-40, packages/agent/src/agent-loop.ts:323-375, packages/coding-agent/src/core/system-prompt.ts:212-229, packages/coding-agent/src/core/agent-session.ts:1737-1749, packages/ai/src/api/anthropic-messages.ts:1136-1230, packages/ai/src/api/anthropic-messages.ts:1316-1354, packages/durable/src/harness/prompt.ts:54-158]
---
[[transcript-carried-system-prompt]] in [[pi]].

## Mechanism
- **Data model** — `SystemMessage {content, sections?: Record<name, string|null>, toolsAdded?: Tool[], toolsRemoved?: ToolReference[], timestamp}` (`packages/ai/src/types.ts:528-544`). Leading one = base prompt + initial tools; later ones = deltas. Doc: "later messages replace sections by name, and `null` removes one. Keep each section self-delimiting (a tag, a heading) so the model can relate an update to the original. Avoid integer-like names; JSON objects reorder those." (`:532-537`).
- **Entry** `normalizeContext`: folds legacy `Context.systemPrompt`/`tools` into a leading system message; empty prompt + no tools ⇒ no message ("an empty transcript stays empty") (`packages/ai/src/utils/transcript.ts:10-34`).
- **Replay** `getCurrentSystemMessage`: later `content` appended, `sections` patched by name (null deletes), tools resolved by ordered remove/add (`getCurrentTools`) (`transcript.ts:57-102`). A changed tool definition = removal + addition (`getToolStateChanges`, `:150-167`); equality via normalized JSON (`declarationsEqual`, `:122-142`).
- **Per-request transport choice** `resolveTranscript(ctx, supportsMidConvoSystemMessages)`: in-place if the model supports mid-conversation system messages, else `collapseSystemMessages` (replayed head, later system messages dropped) (`:104-120`). Tools: `resolveTranscriptTools` keeps initial tools top-level and anchors additions in place only if no removals/redeclarations (`hasNonAdditiveToolChanges`), else sends the current list (`:199-237`).
- **Rendering of updates**: `Updated system prompt section "<name>":\n\n<text>` / `Removed system prompt section "<name>".` — "request-time only and may change between versions" (`text.ts:23-40`).
- **Who writes deltas**:
  - Prompt: coding-agent `_preparePromptAndToolLoadout` diffs transcript-replayed sections vs freshly built sections → sections-only system message (`agent-session.ts:1737-1749`; `diffSystemPromptSections`, `system-prompt.ts:212-229`), on each prompt (`:2104-2106`) and before each next turn (`:898-932`).
  - Tools: agent-core `declareToolChanges` diffs executable `context.tools` vs transcript-declared tools and inserts/patches a system message before each request, "so replay always yields exactly `context.tools`" (`packages/agent/src/agent-loop.ts:323-375`).
- **Provider transports**:
  - Anthropic (`supportsMidConvoSystemMessages && supportsMidConvoToolChanges && initialTools.length>0`): request `tools` fixed = initial tools + `__pi_deferred_placeholder__`; later additions `tool_addition{tool_definition}`, removals `tool_removal{tool_reference}` (skipped when same name redefined) (`anthropic-messages.ts:1136-1230, 1339-1353`); system updates rendered as `role:"system"` messages **held back** until before the next assistant message because `tool_result` must immediately follow `tool_use` — "an update placed before a user message in the transcript lands after it on the wire" (`:1316-1328`). Beta `inline-tools-2026-09-15` (`:195, 1120`).
  - OpenAI Responses: mid-convo developer messages + `additional_tools` developer item / tool search for capable ids (`openai-responses-shared.ts:189-190`; compat `types.ts:894, 903-906`).
  - Mistral: first system → full text, later → `renderSystemMessageUpdate` (`mistral-conversations.ts:798-801`).
  - Google/Vertex: no mid-convo system → collapsed, initial text → `systemInstruction` (`google-shared.ts:192-193`).
  - Compat flags default **false**; generated catalog enables for verified models (`types.ts:867-869, 894, 966-968, 987`; generator `OPENAI_MID_CONVO_SYSTEM_MESSAGE_MODEL_IDS = OPENAI_TOOL_SEARCH_MODEL_IDS`, `generate-models.ts:385`).
- **Session integration**: compaction excludes system messages from the summarized range ("System messages are prompt state, not conversation; the compaction entry carries their replay", `compaction.ts:98-102`) and snapshots `getCurrentSystemMessage` into `CompactionEntry.systemMessage` ("Complete prompt and tool state at this compaction boundary", `session-manager.ts:102-103, 1270-1283`); projection of a compaction = `[systemMessage?, compactionSummary]` → [[auto-compaction]], [[context-projection]]. Resume restores the tool loadout from transcript declarations (`_restoreToolsFromTranscript`, `agent-session.ts:1803+`). `Agent.reset()` keeps the replayed system baseline (`agent.ts:355-368`). Extension `context` event sees messages **without** system messages, `context_with_system` sees all (`extensions/types.ts:866-884`; `aef5fc429`) → [[context-transform-hook]]. Forced prompts are projected, not recorded ([[system-prompt-override]]).
- **Why** (commit `9e05370b2`): "makes system prompt text and tool changes part of the transcript rather than silently rewriting its starting conditions… preserve cached prompt prefixes where the upstream supports it"; also restore exact instructions after resume/branch.

## Constants
| name | value | path:line |
|---|---|---|
| update framing | `Updated system prompt section "<name>":` / `Removed system prompt section "<name>".` | `text.ts:35-36` |
| Anthropic beta | `inline-tools-2026-09-15` | `anthropic-messages.ts:195` |
| compat default | `supportsMidConvo*` = false | `types.ts:867-869, 966-968` |

## Evolution
- `9e05370b2` 2026-09-16 (#9548, Armin Ronacher) "Mid conversation system messages": `SystemMessage.sections/toolsAdded/toolsRemoved`, section diffs, Anthropic placeholder, collapse fallback.
- `e4c75a732` 2026-09-16: forced prompt replaces (was a late update on native models) → [[forced-system-prompt-applied-as-late-update]].
- `16292398a` 2026-09-17: forced prompts sent without recording (a full prompt was written on every change).
- `5a3a03a7f` 2026-09-17 (#9706): evals validate system prompt from transcript (`getCurrentSystemPrompt`).
- `aef5fc429` 2026-09-21 (#9846, #9789, #9822): `context` handlers slicing messages dropped the transcript system message → requests without built-in tools, Codex emitted raw tool-call text; fix hides system messages from `context`, adds `context_with_system`.
- `466db0fec` 2026-09-21: canonical session projections authoritative for requests.
- `4e69b0c28` 2026-09-02 / `69f0be6f0` 2026-10-02 (#10324): signed thinking bound to old prompt/tools → drop stale blocks (Anthropic, Bedrock) → [[signed-reasoning-replay]].
- `b271b0a52` 2026-10-02: inline tool definitions; deprecated `hasToolRedefinitions()` (`transcript.ts:179-197`).
- `92216fa15` 2026-10-06 (#10542): durable `leadWithSystem`.

## Evidence commits
`9e05370b2`, `e4c75a732`, `16292398a`, `5a3a03a7f`, `aef5fc429`, `466db0fec`, `4e69b0c28`, `69f0be6f0`, `b271b0a52`, `92216fa15`.

## Quirks
- Wire order ≠ transcript order on Anthropic (held-back system updates).
- On non-supporting models every prompt change still rewrites the head (collapse) — caching benefit only where compat flags are on.
- Section names must not be integer-like (JSON key reordering).
- `hasToolRedefinitions()` deprecated but kept for API compatibility.

## Durable variant (packages/durable)
- No stored prompt; extension `sections` re-rendered each request and only differences appended as positional `pi.system` entries (`spec.md:3127-3252`). `planSystemEntries` (`packages/durable/src/harness/prompt.ts:54-93`): after a head marker with no later `pi.system`, write one **complete baseline** with `ContextEdit` omissions for every retained earlier `pi.system` entry ("A PR #9548 `SystemMessage` is always a patch, not a reset", `spec.md:3223-3234`); otherwise if a minimal patch would change order, remove-all then re-add-all (`planSections`, `:121-142`); else minimal patch. Tools: kept-in-place + appended additions, or remove-all/re-add-all when order differs (`planTools`, `:100-119`). A throwing renderer keeps its previously shown text (`:25-48`). `leadWithSystem` normalizes request order (`context.ts:170-180`; `92216fa15`).

## Failures
- [[late-tool-change-rewrites-cache]]
- [[forced-system-prompt-applied-as-late-update]]
- [[mcp-startup-blocks-and-description-churn]]
- [[context-handler-drops-system-state]]
