---
type: implementation
harness: codex
concept: replaceable-builtin-extension
commit: 622e9e3696
files: [codex-rs/ext/extension-api/src/contributors.rs:83, codex-rs/ext/extension-api/src/lib.rs:10, codex-rs/features/src/lib.rs:982, codex-rs/ext/items/src/lib.rs:1]
---
[[replaceable-builtin-extension]] in [[codex]].

Partial match: in-box features are moved out of `codex-core` onto a typed extension API, but the extensions are **compiled-in Rust crates**, not runtime-loadable or replaceable by third parties. Disable/enable is by feature flag, not by name-based displacement.

## Mechanism
- Extension crates at HEAD (`codex-rs/ext/`): agent, agent-message-board, connectors, git-attribution, goal, guardian-reviewer, guardian-v2, history-notes, image-generation, items, mcp, memories, queue, skills, web-search, plus `extension-api`.
- API surface: contributor traits for thread/turn/tool/approval/world-state lifecycle (`codex-rs/ext/extension-api/src/contributors.rs:83-430`); capabilities `ConversationHistorySnapshot`, `ExtensionEventSink`, `ExtensionMetrics`, `ResponseItemInjectionFuture`; `ToolPolicy`; `SessionIsolation` / `IsolatedSessionExtensions` (`codex-rs/ext/extension-api/src/lib.rs:10-20`) → [[codex--extension-event-hooks|extension-event-hooks]].
- Toggle path: each feature gated by a `Feature` with a lifecycle stage in the central registry (`codex-rs/features/src/lib.rs:982`) → [[feature-flag-stages]]; admins pin via `feature_requirements`; agent roles may only turn a small set off ([[agent-profiles]]).
- Third-party extension is declarative instead: plugin bundles (skills + MCP + apps + hooks) → [[codex--harness-package-distribution|harness-package-distribution]], [[no-executable-plugins]].
- Built-in MCP servers were tried and dropped: `ca257b6ce5` 2026-05-06 → `32b1ae7099` 2026-05-11 "Drop something that was never used"; memories MCP dropped `d579dafb70` 2026-05-26.

## Evolution
- 2026-03-19 → 2026-04-02 mass extraction from core into crates (features, sandboxing, rollout, plugin, analytics, instructions, tools, codex-mcp, models-manager, model-provider-info).
- 2026-05-11 `d2c3ebac1f` typed extension API; extensions born: guardian `d996f5366f` (2026-05-12), memories `8ba6749932`, goal `a80f07ec4a`, web-search `a22706dfae` (2026-05-26), image-generation `ecb41fcb64`, skills `2d385e166c`, mcp `4ec3b8eeea`, agent `6629e08702`, queue `bc8b25ea02`, history-notes `daa48072f4` (2026-08-21), guardian-v2 `fe614a6304` (2026-08-13; ext/guardian consolidated `e741cd9ace` 2026-08-19), agent-message-board `abbdde95b5` (2026-09-21).

## Versus pi
- [[pi--replaceable-builtin-extension]]: pi's built-ins (`mcp`, `codemode`, `tool-search`, `llama.cpp`) are written on the *public* plugin API and step aside when a third-party plugin registers the same name; codex's ext crates use an internal API only first-party code can implement, so "proves the API is sufficient for third parties" does not hold. Both converge on "features out of core". See [[extensibility-model]].
