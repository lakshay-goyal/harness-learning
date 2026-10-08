---
type: implementation
harness: pi
concept: extension-ui-primitives
commit: b30a6dd77
files: [packages/coding-agent/src/core/extensions/types.ts:149-304, packages/coding-agent/src/core/extensions/types.ts:323-364, packages/coding-agent/src/core/extensions/types.ts:1653, packages/coding-agent/src/core/extensions/types.ts:1685-1694, packages/coding-agent/src/core/extensions/runner.ts:96-138, packages/coding-agent/src/core/extensions/runner.ts:324, packages/coding-agent/src/core/extensions/runner.ts:673-716, packages/coding-agent/src/modes/rpc/rpc-mode.ts:138-309]
---
[[extension-ui-primitives]] in [[pi]].

## Mechanism
- `ExtensionContext` (`core/extensions/types.ts:325-364`): `ui`, `mode: "tui"|"rpc"|"json"|"print"` (`:323`, `e56521e32`), `hasUI` (true in TUI and RPC, `:330`), `cwd`, read-only `sessionManager`, `modelRegistry`, `model`, `scopedModels`, `thinkingLevel`, `isIdle`, `isProjectTrusted`, `signal`, `abort`, `hasPendingMessages`, `shutdown`, `getContextUsage`, `compact`, `getSystemPrompt`.
- `ExtensionUIContext` (`types.ts:149-304`):
  - dialogs `select/confirm/input/editor` with `timeout` + `signal` (example `timed-confirm.ts`);
  - `notify`; `onTerminalInput` (raw key interception);
  - `setStatus` (footer), `setWorkingMessage/Visible/Indicator`, `setHiddenThinkingLabel`;
  - `setWidget` (string[] or component, above/below editor), `setFooter`, `setHeader`, `setTitle`;
  - `custom<T>(factory, {overlay, overlayOptions, onHandle})` — focused component/overlay resolving via `done`;
  - `pasteToEditor/setEditorText/getEditorText`, `addAutocompleteProvider` (stacked), `setEditorComponent` (e.g. vim editor via `CustomEditor`);
  - theme get/set; `get/setToolsExpanded`.
- Blocking dialogs emit `ui_prompt_start/end` to other extensions (`runner.ts:582-609`; `ccfe79ed2`).
- Renderer registries (`types.ts:1685-1694`): `registerMessageRenderer(customType)`, `registerMarkdownTransformer`, `registerEntryRenderer` (TUI-only `appendEntry` entries), `registerToolRenderer((toolName, next) => …)` resolver chain — can render tools that aren't registered yet, e.g. MCP tools before server connects (`docs/extensions.md:190`; `11449730c`). Custom renderers reused by HTML export ([[pi--session-export-share]]).
- Shortcuts: `registerShortcut(KeyId, {handler})` (`types.ts:1653`); extensions can't override reserved app keys (`app.interrupt`, `app.exit`, `tui.input.submit`, …), win over non-reserved built-ins with a diagnostic (`runner.ts:96-138,673-716`; `54c33f2ad`; `packages/coding-agent/CHANGELOG.md:2319`).
- Per-mode implementations: print/json → no-op UI (`runner.ts:324`); RPC forwards `select|confirm|input|editor|notify|setStatus|setWidget(strings only)|setTitle|set_editor_text` over `extension_ui_request/response`, agent-side timeout auto-resolves; component factories, `custom()`, theme switching stubbed (`modes/rpc/rpc-mode.ts:138-309`; `rpc-types.ts:252-297`) → [[pi--headless-rpc-mode]].
- `ExtensionCommandContext` adds `waitForIdle`, `newSession/fork/navigateTree/switchSession` (with `withSession`), `reload` — "only safe in user-initiated commands" (`types.ts:398-439`; `docs/extensions.md:214-215`).
- UI-heavy examples: `custom-footer`, `custom-header`, `modal-editor`, `rainbow-editor`, `widget-placement`, `overlay-test`/`overlay-qa-tests`, `doom-overlay/` (35 FPS wasm), `snake`, `space-invaders`, `tic-tac-toe`, `mac-system-theme`, `hidden-thinking-label`, `minimal-mode`, `built-in-tool-renderer`, `message-renderer`, `entry-renderer`, `border-status-editor`, `github-issue-autocomplete`, `qna`, `question`, `questionnaire`, `timed-confirm`, `titlebar-spinner`, `working-indicator`, `bookmark` (`setLabel`), `rpc-demo` (`packages/coding-agent/examples/extensions/`).

## Constants
- none beyond TUI ([[pi--differential-tui-rendering]]).

## Evolution
- 2025-12-09 `04d59f31e` `HookUIContext` per mode (with hooks system).
- 2026-01-12 `a4ccff382` overlay positioning; 2026-01-18 `54c33f2ad` reserved keybindings respected.
- 2026-06-01 `e56521e32` `ctx.mode`.
- 2026-08-27 `ccfe79ed2` UI prompt events (#8355).
- 2026-10-03 `11449730c` tool renderer chain renders MCP calls pre-connect.

## Evidence commits
`04d59f31e` `a4ccff382` `54c33f2ad` `e56521e32` `ccfe79ed2` `11449730c`

## Quirks
- `hasUI` true in RPC even though only a subset works; plugins must check `mode` before rich components.
- `permission-gate.ts` blocks dangerous commands when no UI is available (fail-closed by design of the example).

## Failures
[[plugin-shortcut-shadows-core-keys]]
