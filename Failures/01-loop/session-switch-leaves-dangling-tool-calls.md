---
type: failure
concepts: [abort-propagation, transcript-replay-repair, session-tree]
harnesses: [pi]
---
**Symptom** — Switching session or navigating the tree during an active response left assistant tool calls without results in the persisted transcript.

**Root cause** — The session was detached mid-turn; the in-flight turn (incl. tool results) never reached the outgoing session file.

**Fix · [[pi]]** — `cefa40ed8` 2026-07-28 "guard tree navigation during responses" (#7022): abort and persist the outgoing turn first (`packages/coding-agent/src/core/agent-session.ts:2899` at that commit; HEAD `agent-session-runtime.ts:167-178`: "so the aborted turn (including tool results) is persisted to the outgoing session"). `/tree` rejects while streaming/compacting (`agent-session.ts:3963-4155`). Replay backstop: orphaned calls get synthetic "No result provided" results (`packages/ai/src/api/transform-messages.ts:158-185`).

**Lesson** — Never detach a session mid-turn; settle (abort + persist) first.

Related: [[abort-propagation]] · [[transcript-replay-repair]] · [[session-tree]] · [[orphaned-tool-calls-and-results]] · [[pi--abort-propagation|pi]]
