---
type: failure
concepts: [signed-reasoning-replay, transcript-replay-repair]
harnesses: [pi]
---
**Symptom** — Anthropic returned 400 `Invalid signature in thinking block` when the user resubmitted after aborting a turn.

**Root cause** — The stream was aborted mid-thinking, so the persisted block had no signature or only part of one. Anthropic validates signatures on replay. An aborted stream's artifacts are unsigned or incomplete by construction.

**Fix · [[pi]]**
- `387cc97ba` (2025-11-18): convert unsigned thinking to text at the adapter level. This later led to [[thinking-tag-mimicry]] because the text was wrapped in tags.
- `2d27a2c72` (2026-01-19), #838 superseded it: errored and aborted assistant messages are skipped entirely at replay (`packages/ai/src/api/transform-messages.ts:195-203`) → [[failed-turns-replayed]].
- The aborted partial is still persisted and shown, but never replayed (packages/ai/README.md:1162-1185 vs `transform-messages.ts:201-203`).

**Lesson** — Persisted partial reasoning cannot be replayed. Drop incomplete turns from the provider projection, or downgrade them to text when the signature is missing.

Related: [[signed-reasoning-replay]] · [[transcript-replay-repair]] · [[partial-message-persistence]] · [[pi--transcript-replay-repair|pi]]
