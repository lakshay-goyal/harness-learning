---
type: implementation
harness: pi
concept: branch-scoped-extension-state
commit: b30a6dd77
files: [packages/coding-agent/docs/extensions.md:222, packages/coding-agent/examples/extensions/todo.ts:1, packages/coding-agent/src/extensions/codemode/execute.ts:232, packages/codemode/src/runtime/prelude-source.ts:32, packages/coding-agent/src/core/extensions/types.ts:1717, packages/durable/src/documents.ts:39]
---
[[branch-scoped-extension-state]] in [[pi]].

## Mechanism
- **Doctrine table** (`packages/coding-agent/docs/extensions.md:222-235`): tool state that follows the active branch → tool-result `details`; durable data excluded from model context → `pi.appendEntry()`; custom content stored *and* sent to the model → `pi.sendMessage()`; data outside one session → external storage. "Reconstruct branch-sensitive state from `ctx.sessionManager.getBranch()` during `session_start`. Do not rebuild it from every file entry because abandoned branches represent alternative histories."
- **Entry kinds**: `custom` entries (extension state, not in context) vs `custom_message` (in context, `display` flag) ([[session-tree]], `packages/coding-agent/src/core/session-manager.ts:64-194`); `appendEntry<T>(customType, data?)` (`packages/coding-agent/src/core/extensions/types.ts:1717`). `ToolResultMessage.details` is JSON-typed and compile-time checked (`IsJsonCompatible`) and never sent to the model (`packages/ai/src/types.ts:585-624`).
- **`todo.ts` example** — "State is stored in tool result details (not external files), which allows proper branching" (`packages/coding-agent/examples/extensions/todo.ts:1-11`): every `todo` call returns full `{action, todos, nextId}` in `details` (`:154-161`); `reconstructState` replays `getBranch()` taking the last details (`:118-126`) on `session_start` and `session_tree` (`:132-133`). Pi's answer to [[no-todo-tool]] ("They confuse models").
- **Codemode `store`/`load`**: script KV writes kept only if the script succeeds, appended as `custom` entry `codemode-store` `{set, delete}` (`packages/coding-agent/src/extensions/codemode/execute.ts:500-507`); `load()` = replay of `codemode-store` entries on the branch from root (`readCodemodeStore`, `:232-243`). Limits `MAX_STORE_VALUE_CHARS = 256Ki`, `MAX_STORE_TOTAL_CHARS = 1Mi` JSON chars (`packages/codemode/src/runtime/prelude-source.ts:32-33,302-311`) → [[code-mode]].
- Other examples: `bookmark.ts` (labels), `entry-renderer.ts` (TUI-only custom entries via `appendEntry` + `registerEntryRenderer`), `plan-mode/` reuses todo-style tracking, `tools.ts` (`/tools` enable/disable persisted).
- Tool loadout itself is branch state: restored from the transcript's replayed system message on resume (`packages/coding-agent/src/core/agent-session.ts:1808-1815`); MCP tools loaded by `tool_search` re-activated when servers register (`c662ec7e3`).

## Constants
| name | value | path:line |
|---|---|---|
| `MAX_STORE_VALUE_CHARS` | 262 144 | packages/codemode/src/runtime/prelude-source.ts:32 |
| `MAX_STORE_TOTAL_CHARS` | 1 048 576 | packages/codemode/src/runtime/prelude-source.ts:33 |

## Evolution
- 2025-11-12 `9066f58ca` "pi does not and will not support built-in to-dos … make it stateful by writing to a file: TODO.md".
- 2026-01-05 `c6fc08453` unified extensions; `todo.ts` example with state in tool details.
- 2026-09-29 `8562bcf66` codemode `store/load` as branch-aware `codemode-store` entries.
- 2026-10-01 `c662ec7e3` restore tool_search-loaded MCP tools on resume/reload.

## Evidence commits
`9066f58ca`, `c6fc08453`, `8562bcf66`, `c662ec7e3`

## Quirks
- Todo snapshots repeat the whole list in every result (O(n) per call, grows the log).
- `appendEntry` entries are tree nodes → move the leaf like any entry.

## Durable variant (packages/durable)
- Typed documents: `defineDoc({kind, version, scope: session|conversation|task, history: latest|rewindable, fork: initial|current|asOf, initial, checkpointWhen})` (`packages/durable/src/documents.ts:39-75`; `packages/durable/README.md:511-518`), stored as Chord base + delta chains (`docs/spec.md:4377-4387`); built-ins `pi.agent` (rewindable, fork `asOf`), `pi.provider`, `pi.live`, `pi.inbox`, `pi.usage`. `HarnessOptions.conversationCreated(tx, conversation)` runs in every commit that creates/forks a conversation so every conversation gets plugin docs (`packages/durable/README.md:526`). `pi.` prefix reserved by convention only — "Nothing enforces it" (`spec.md:4695-4698`). Extensions stored by name: uninstalled extension ⇒ conversations "just stop getting it" (`packages/durable/README.md:210`).

## Failures
- none recorded
