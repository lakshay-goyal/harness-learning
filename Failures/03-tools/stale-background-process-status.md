---
type: failure
concepts: [shell-execution]
harnesses: [codex]
---
**Symptom** — `/ps` showed no background terminals while the status line still said it was waiting for one: `write_stdin` polled a process that exited before the response returned, the process manager correctly returned `process_id: None`, but the handler still emitted a `TerminalInteraction` event with the requested session id, so clients believed a dead process was being polled (issue #23214). Later: early output missing from completion events; cancellation could split an output chunk.

**Root cause** — Liveness and output for long-lived process sessions were tracked in more than one place (manager state vs handler-emitted events vs buffers updated non-atomically).

**Fix · [[codex]]**
- `e43a2e297f` 2026-05-19 "Fix stale background terminal poll events (#23231)".
- `d47b9a8c00` 2026-09-23 "Preserve early unified exec output in completion events (#47665)".
- `4891c4e35f` 2026-09-24 "Update unified exec output buffers atomically (#47712)".

**Lesson** — Long-lived process sessions need a single source of truth for liveness and output; derive events from it, never from the request.

Related: [[shell-execution]] · [[codex--shell-execution|codex]] · [[premature-backgrounding]] · [[agent-event-stream]]
