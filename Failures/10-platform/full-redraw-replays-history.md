---
type: failure
concepts: [differential-tui-rendering]
harnesses: [pi]
---
**Symptom** — The terminal transcript showed duplicated history: (a) full re-renders appended a second copy of the conversation into scrollback; (b) on Termux, toggling the soft keyboard changed terminal height, which triggered a full redraw that replayed the whole history each time (#2467).

**Root cause** — A main-screen differential renderer can't touch rows already scrolled into native scrollback; any forced full redraw rewrites everything below, and height changes were treated as redraw triggers.

**Fix · [[pi]]**
- `fb4893d8d` 2025-11-11 — clear scrollback (`\x1b[3J`) on full re-render (`packages/tui/src/tui-main-screen.ts:331-358`).
- `16937947b` 2026-03-20 — skip height-change redraws on Termux (`tui-main-screen.ts:344-347`).
- Structural answer: alternate-screen renderer with app-owned scrolling (`c13ffe187` 2026-07-30), default since `88ff80b98` 2026-10-01.

**Lesson** — With native scrollback, minimize full-redraw triggers and clear scrollback when you must; owning the screen (alt-screen) removes the class.

Related: [[differential-tui-rendering]] · [[pi--differential-tui-rendering|pi]]
