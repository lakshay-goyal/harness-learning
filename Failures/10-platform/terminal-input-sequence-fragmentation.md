---
type: failure
concepts: [differential-tui-rendering]
harnesses: [pi]
---
**Symptom** — Keystrokes dropped or misread: over SSH, key presses batched with Kitty release events in one stdin chunk were lost (#538); split Alt+Enter arrived as lone ESC → interrupted the agent (#7876); Kitty printable input duplicated (#3780); WezTerm `ESC ESC[27;…u` parsed as legacy meta; Shift+Enter indistinguishable from Enter on Apple Terminal / Windows.

**Root cause** — stdin `data` events are arbitrary byte chunks, not key events; escape sequences can be split across chunks or several fused in one, and some terminals cannot encode modifiers at all.

**Fix · [[pi]]** — `f3b7b0b17` 2026-01-07: `StdinBuffer` (adapted from OpenTUI) splits/reassembles CSI/OSC/DCS/APC/paste so each `handleInput` gets one sequence (`packages/tui/src/stdin-buffer.ts:1-26,194`). `bdb416cbc` 2026-04-27 dedupe Kitty printable input. `7f30fb613` 2026-05-13 split `\x1b\x1b[` into ESC + new sequence. `c5181a266` 2026-05-23 / `73dd066ee` 2026-08-05 native Shift-state probe on `\r` (`packages/tui/src/terminal.ts:352-361`). `06ed87167` 2026-08-11 (#7899) longer ESC timeout over SSH + `PI_TUI_ESC_TIMEOUT`; `2a95ef70d` applies it only to lone ESC so Escape/mouse stay snappy (`packages/tui/src/terminal.ts:120-130`).

**Lesson** — Treat terminal input as a framed protocol: buffer to sequence boundaries, use separate timeouts for "lone ESC" vs "incomplete sequence", and scale them for network latency.

Related: [[differential-tui-rendering]] · [[pi--differential-tui-rendering|pi]]
