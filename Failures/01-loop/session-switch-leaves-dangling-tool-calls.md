---
type: failure
concepts: [abort-propagation, transcript-replay-repair, session-tree]
harnesses: [pi, codex]
---
**Symptom** — Switching session or navigating the tree during an active response left assistant tool calls without results in the persisted transcript.

**Root cause** — The session was detached mid-turn; the in-flight turn (incl. tool results) never reached the outgoing session file.

**Fix · [[pi]]** — `cefa40ed8` 2026-07-28 "guard tree navigation during responses" (#7022): abort and persist the outgoing turn first (`packages/coding-agent/src/core/agent-session.ts:2899` at that commit; HEAD `agent-session-runtime.ts:167-178`: "so the aborted turn (including tool results) is persisted to the outgoing session"). `/tree` rejects while streaming/compacting (`agent-session.ts:3963-4155`). Replay backstop: orphaned calls get synthetic "No result provided" results (`packages/ai/src/api/transform-messages.ts:158-185`).

**Fix · [[codex]]** (variant: resume panics on an unpaired call)
- Symptom: a saved session containing a tool call with no output (user cancelled, or rollout error) made Codex panic on resume (`1e3cad95c0` body, issue #7990).
- `1e3cad95c0` 2025-12-15 "Do not panic when session contains a tool call without an output (#8048)" — history normalization tolerates/synthesizes missing outputs (`codex-rs/core/src/context_manager/normalize.rs`). Prevention side: aborted tools get synthetic "aborted by user" outputs (`e95abcdf49:codex-rs/core/src/tools/parallel.rs:363-380`) and items are recorded as they complete (`f59978ed3d`).

**Lesson** — Never detach a session mid-turn; settle (abort + persist) first — and still normalize at load, because persisted transcripts will contain orphaned calls.

Related: [[abort-propagation]] · [[transcript-replay-repair]] · [[session-tree]] · [[orphaned-tool-calls-and-results]] · [[pi--abort-propagation|pi]] · [[turn-items-lost-on-abort]] · [[codex--abort-propagation|codex]]
