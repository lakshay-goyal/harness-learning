---
type: concept
stage: architecture
tier: candidate
aliases: ["ctx.ui", "ctx.hasUI", "ctx.mode", "ExtensionUIContext", "registerMessageRenderer", "registerEntryRenderer", "registerToolRenderer", "custom-message-renderers", "registerShortcut"]
harnesses: [pi]
---
Mode-portable UI API for plugins — dialogs, notifications, status/widgets, overlays, editor replacement, custom message/entry/tool renderers, shortcuts — that degrades per front-end (TUI full, RPC forwarded subset, print/JSON no-op).

## Why
- Plugins implementing permission gates, plan mode, questionnaires or sub-agent dashboards need user interaction; without a portable API they only work in one front-end.
- Headless modes must not hang waiting for a dialog nobody can answer (pi: `hasUI` false in print/json; RPC dialogs carry timeouts).
- Plugin keybindings can shadow core keys like submit/interrupt ([[plugin-shortcut-shadows-core-keys]]).

## Design space
- **Capability probing**: `hasUI` boolean + `mode` enum (pi) vs feature flags per primitive.
- **Degradation**: no-op UI (pi print/json) vs forward over protocol (pi RPC: select/confirm/input/editor/notify/status/widget-strings/title/editor-text) vs throw.
- **Rich components**: arbitrary TUI components/overlays (pi `custom()`, `setEditorComponent`) — not portable over RPC (stubbed).
- **Custom transcript content**: plugin-defined message/entry types with own renderers (pi) — also reused in HTML export ([[session-export-share]]).
- **Renderer resolution**: by registered tool only vs resolver chain that can render not-yet-registered tools (pi `registerToolRenderer`).
- **Shortcut conflicts**: last wins vs reserved core list + diagnostics (pi).
- **Blocking dialog visibility**: emit `ui_prompt_start/end` so other plugins can react (pi).

## Implementations
- [[pi--extension-ui-primitives|pi]] — `ExtensionUIContext` (dialogs, notify, status, widgets, header/footer, overlays, editor swap, autocomplete, themes), renderer registries, RPC extension-UI forwarding.

## Failures
- [[plugin-shortcut-shadows-core-keys]]

## Related
[[extension-event-hooks]] · [[plugin-tools]] · [[differential-tui-rendering]] · [[headless-rpc-mode]] · [[branch-scoped-extension-state]] · [[session-export-share]]
