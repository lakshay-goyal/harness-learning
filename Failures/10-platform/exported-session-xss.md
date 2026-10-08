---
type: failure
concepts: [session-export-share]
harnesses: [pi]
---
**Symptom** — Exported/shared HTML transcripts executed attacker-influenced content: `javascript:` links and injected markup from session content (tool output, fetched pages), image data/metadata injection (#3532, #3819, #3883).

**Root cause** — Transcript content was rendered as trusted markdown/HTML in the viewer.

**Fix · [[pi]]**
- 0.31.0 — sanitize user HTML (hash unverified).
- `ee462dd70` 2026-04-22 (#3532) — escape in export template; follow-ups escape image data/metadata (#3819, #3883).
- `6cb23f9b5` 2026-06-02 — scheme allow-list after stripping control chars: `sanitizeMarkdownUrl` removes C0 controls then permits only `https?|mailto|tel|ftp` for links and images (`packages/coding-agent/src/core/export-html/template.js:619-624,1615-1631`).

**Lesson** — Transcripts contain untrusted content; render exports as untrusted input, with scheme allow-lists applied after browser-equivalent normalization.

Related: [[session-export-share]] · [[no-prompt-injection-defense]] · [[pi--session-export-share|pi]]
