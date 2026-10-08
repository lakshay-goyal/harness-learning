---
type: implementation
harness: opencode
concept: transcript-carried-system-prompt
commit: ecc4916b5a
files: [packages/core/src/system-context/index.ts:21-39, packages/core/src/system-context/index.ts:150-180, packages/core/src/system-context/index.ts:197-320, packages/core/src/system-context/builtins.ts:12-42, packages/core/src/session/context-epoch.ts:31-174, packages/core/src/session/history.ts:24-53, packages/core/src/instruction-context.ts:71-87, packages/llm/src/protocols/anthropic-messages.ts:353-423, packages/llm/src/protocols/shared.ts:111-127, packages/core/src/event.ts:122-123, CONTEXT.md:7-37, CONTEXT.md:90-133]
---
[[transcript-carried-system-prompt]] in [[opencode]].

## Mechanism

### Legacy runtime
- Not implemented: the system prompt is rebuilt from current config/files every step and sent as ≤ 2 leading system blocks ([[opencode--cache-stable-prompt-prefix]]).

### v2 runtime — System Context, Context Sources, Context Epoch
- **Vocabulary** (`CONTEXT.md:7-37`): System Context (_Avoid_: system prompt), Context Source (_Avoid_: prompt fragment), Baseline System Context, Context Snapshot, Context Epoch, Mid-Conversation System Message (_Avoid_: system update, raw text diff), Safe Provider-Turn Boundary, Unavailable Context.
- **Context Source** (`packages/core/src/system-context/index.ts:21-39`): stable namespaced key (`^[a-z0-9][a-z0-9._-]*\/[a-z0-9][a-z0-9._/-]*$`), JSON codec, infallible `load` (may return the `unavailable` symbol), pure `baseline`/`update(previous, current)` renderers, optional `removed` renderer. `make` hides the value type; `combine` rejects duplicate keys (`packages/core/src/system-context/index.ts:175-180`). Comparison uses codec equivalence on decoded values, not rendered-text hashes (`packages/core/src/system-context/index.ts:154-166`). Empty renders throw (`packages/core/src/system-context/index.ts:309-312`).
- Built-ins: `core/environment` (`<env>` cwd, workspace root, git, platform; update "The environment you are running in is now:") and `core/date` (update "Today's date is now: …") (`packages/core/src/system-context/builtins.ts:16-39`); also `core/instructions`, `core/skill-guidance`, `core/reference-guidance` ([[opencode--project-references]]).
- **Epoch lifecycle** (`packages/core/src/session/context-epoch.ts`): first turn `initialize` renders baseline + snapshot into `session_context_epoch` at `baseline_seq`, *before* prompt promotion so an unavailable baseline leaves the prompt pending (`packages/core/src/session/context-epoch.ts:80-89`; `packages/core/src/session/runner/llm.ts:183`). Later turns `prepare` → `reconcile` returns exactly one of Unchanged / Updated / ReplacementReady / ReplacementBlocked (`packages/core/src/system-context/index.ts:217-226`).
- **Updated** → one `SessionEvent.ContextUpdated` with the combined text of all changed, new (baseline rendering once) and removed (pre-rendered removal) sources (`packages/core/src/system-context/index.ts:247-279`); the snapshot advances atomically via the event's local `commit` hook (`packages/core/src/session/context-epoch.ts:72-76`; `packages/core/src/event.ts:122-123`).
- **Replacement** only after a completed compaction newer than `baseline_seq` (`packages/core/src/session/context-epoch.ts:59-70`), or when a stored value no longer decodes / a removed source lacks a removal renderer (`packages/core/src/system-context/index.ts:239`, `packages/core/src/system-context/index.ts:244`).
- **History selection**: messages since the latest compaction, plus `system` messages only if `seq > baseline_seq` (`packages/core/src/session/history.ts:24-53`) — older updates leave model history but stay durable.
- **Unavailable Context**: stale-while-revalidate — unavailable sources keep their stored snapshot and emit nothing (`packages/core/src/system-context/index.ts:251-253`); initialization fails `InitializationBlocked` (`packages/core/src/system-context/index.ts:198-206`); replacement blocks while an admitted source is unavailable (`packages/core/src/system-context/index.ts:287-291`). Instruction loader returns unavailable when a discovered AGENTS.md cannot be read or on any error/defect (`packages/core/src/instruction-context.ts:71-72`, `packages/core/src/instruction-context.ts:86-87`).
- **Ordering & laziness**: promoted user input and settled tool results precede the update (`CONTEXT.md:99`); "Context source changes never wake idle sessions" (`CONTEXT.md:126`); admitted updates survive a failed provider attempt and replay unchanged (`CONTEXT.md:127`); content states only the new value (`CONTEXT.md:220-221`).
- **Provider lowering** (`packages/llm`): native mid-conversation `role: "system"` **only for Anthropic `claude-opus-4-8`**, and only when the previous message is user/tool/server-tool-use and the next is assistant or end (`packages/llm/src/protocols/anthropic-messages.ts:353-373`). Everything else: user text `<system-update>…</system-update>` with `& < >` XML-escaped so content cannot close the wrapper (`packages/llm/src/protocols/shared.ts:111-121`), merged into a preceding user message (`packages/llm/src/protocols/anthropic-messages.ts:416-421`). An update that would split a local tool call from its result is rejected as an invalid request (`packages/llm/src/protocols/anthropic-messages.ts:375-385`, `packages/llm/src/protocols/anthropic-messages.ts:410-411`). Policy: "Do not insert raw retrieved, tool, or web content into privileged updates" (`packages/llm/src/protocols/shared.ts:123-127`).

## Constants
| name | value | path:line |
|---|---|---|
| native system-update model | `claude-opus-4-8` only | `packages/llm/src/protocols/anthropic-messages.ts:356` |
| fallback wrapper | `<system-update>` (XML-escaped) | `packages/llm/src/protocols/shared.ts:120-121` |
| source key pattern | `^[a-z0-9][a-z0-9._-]*\/[a-z0-9][a-z0-9._/-]*$` | `packages/core/src/system-context/index.ts:22` |

## Evolution
- 2026-06-03 `76ee87ead8` v2 runtime + `supportsNativeSystemUpdates` / `wrapSystemUpdate`.
- 2026-06-04 `1af8dafd3e` (#30789) persist v2 session context epochs.
- 2026-06-05 `a261b55e43` snapshot decode `orDie` → typed `ContextSnapshotDecodeError` → [[strict-schema-rejects-legacy-records]].
- 2026-06-22 `c6ee511485` simplify epochs: model/agent switches no longer force a new baseline; migration `20260622142730_simplify_session_context_epoch`; same day `1787fa4261` adds `20260622170816_reset_v2_session_state`.
- 2026-08-13 `d8bf79225f` `Moved` projector no longer resets the epoch → [[projector-depends-on-transitional-table]].

## Quirks / drift
- Spec says a session move clears the epoch (`CONTEXT.md:118`, `specs/v2/schema-changelog.md:794`); at HEAD `SessionContextEpoch.reset` has no callers (`packages/core/src/session/context-epoch.ts:111`), so a moved session keeps its old baseline and gets an "environment is now" update instead (inference) → [[stale-runner-recreates-context-after-move]].
- The privileged update channel is user-visible text on every model but one, so its authority depends on prompt framing.

## Failures
[[late-tool-change-rewrites-cache]] · [[side-channel-message-splits-tool-pair]] · [[stale-runner-recreates-context-after-move]]

Contrast: [[pi--transcript-carried-system-prompt|pi]] stores prompt sections and tool deltas as transcript `SystemMessage`s and collapses them for models without mid-conversation system support; opencode v2 keeps a durable baseline per epoch, models sources as typed codecs with removal text, and lowers updates natively only for Opus 4.8.
