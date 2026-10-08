---
type: implementation
harness: opencode
concept: context-transform-hook
commit: ecc4916b5a
files: [packages/plugin/src/index.ts:282-308, packages/opencode/src/session/prompt.ts:1255, packages/opencode/src/session/compaction.ts:372-391, packages/opencode/src/session/llm/request.ts:68-78, specs/v2/session.md:142]
---
[[context-transform-hook]] in [[opencode]].

## Mechanism

### Legacy runtime — plugin hooks on stored messages and on the system array
- `experimental.chat.messages.transform(input: {}, output: {messages: {info, parts}[]})` (`packages/plugin/src/index.ts:282-290`): fired every loop step on the in-memory `msgs` (stored `MessageV2` messages with parts) **before** conversion to provider messages (`packages/opencode/src/session/prompt.ts:1255`, then `toModelMessagesEffect` at `:1262`). History is re-read from SQLite each iteration, so mutations stay request-local (inference from the loop shape).
- Same hook runs on a `structuredClone` of the compaction head before serialization (`packages/opencode/src/session/compaction.ts:378-379`; `4cb29967f6` 2026-03-16 "apply message transforms during compaction") → [[compaction-request-shape-mismatch]].
- `experimental.chat.system.transform({sessionID, model}, {system: string[]})` mutates the joined system array; if the plugin kept the header and pushed more, the tail is re-joined to keep ≤ 2 system blocks for caching (`packages/opencode/src/session/llm/request.ts:68-78`, `72ebaeb8f7`) → [[opencode--cache-stable-prompt-prefix]].
- `experimental.session.compacting({sessionID}, {context, prompt?})` adds context strings or replaces the summarizer prompt (`index.ts:305-308`; `compaction.ts:372-391`).
- Whole-request hooks `chat.params` / `chat.headers` mutate sampling options and headers (not messages).

### v2 runtime
- No plugin message, system, parameter or header transforms yet: parity table row "Plugin message, system, parameter, and header transforms — missing — Design V2 plugin hooks and lifecycle semantics" (`specs/v2/session.md:142`).

## Constants
| name | value | path:line |
|---|---|---|
| max system blocks after system transform | 2 | `packages/opencode/src/session/llm/request.ts:74-78` |

## Evolution
- 2025-12-11 `2a9269c347` add `experimental.chat.messages.transform` (#5207).
- 2025-12-15 `72ebaeb8f7` rejoin system after the system hook to preserve caching.
- 2026-03-16 `4cb29967f6` apply message transforms during compaction too.

## Quirks / drift
- The hook receives storage-shaped messages (info + parts), not provider messages, so a plugin can only prune/inject in opencode's own schema; tool-call/result adjacency is preserved by construction (parts of one assistant message).
- No error isolation: `Plugin.trigger` awaits each hook via `Effect.promise` with no catch, so a throwing hook becomes a defect for the whole step (`packages/opencode/src/plugin/index.ts:284-297`).

Contrast: [[pi--context-transform-hook|pi]] offers two phases (`context` conversation-only and `context_with_system` full transcript) with harness projections chained after; opencode legacy has one storage-level message hook plus a separate system-array hook, and v2 has none yet.
