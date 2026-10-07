---
type: concept
stage: state
tier: candidate
aliases: [appendEntry, tool-result details, codemode store/load, codemode-store, todo.ts, getBranch, custom entry, branch-scoped-tool-state, todo-state-in-tool-details, defineDoc]
harnesses: [pi]
---
Plugin/tool state is persisted *inside the session log* (tool-result `details` or custom entries) and reconstructed by replaying the active branch, so after a rewind/branch/fork each branch sees exactly the state it had at that point.

## Why
- State in external files or process memory diverges from the conversation the moment the user rewinds or forks (a todo list "remembers" items from an abandoned branch).
- Makes plugin state crash-safe and exportable for free.

## Design space
- **External file / process memory** (diverges on branch) vs **tool-result `details`** (follows branch, not in model context) vs **custom entries** (`appendEntry`, durable, not in context) vs **custom messages** (in context) — pi docs table.
- **Rebuild**: replay `getBranch()` on `session_start`/`session_tree` (pi), not all file entries ("abandoned branches represent alternative histories").
- **Transactional writes**: only persist if the script succeeds (codemode `store`).
- **Typed documents with history/fork policy** (pi-durable `defineDoc({scope, history: latest|rewindable, fork: initial|current|asOf})`).
- **Size limits** for stored values (codemode 256Ki chars/value, 1Mi total).
- codex: not applicable as such (no in-file branching). Closest: harness state persisted in the log as world-state merge-patch snapshots (`WorldStateItem`, `codex-rs/protocol/src/protocol.rs:3330-3347`) so resume/fork keep diffing; goal state lives outside the log in `goals_1.sqlite` (`codex-rs/state/src/sqlite.rs:34-39`) — whether goals follow forks is unverified.

## Implementations
- [[pi--branch-scoped-extension-state|pi]] — `todo.ts` stores state in tool `details`, rebuilds from `getBranch()`; codemode `codemode-store` custom entries; `pi.appendEntry`; durable `defineDoc` scopes/fork policies.

## Failures
- none recorded

## Related
[[session-tree]] · [[session-fork]] · [[plugin-tools]] · [[code-mode]] · [[extension-event-hooks]] · [[durable-execution]] · [[no-todo-tool]] · [[structured-tool-output]]
