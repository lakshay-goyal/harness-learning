---
type: failure
concepts: [usage-cost-accounting, errors-as-stream-events]
harnesses: [pi]
---
**Symptom**
- Aborted Anthropic streams showed 0 tokens.
- The input token count reset to 0 behind Portkey.
- A `message_delta` without `usage` crashed or did nothing.
- Moonshot usage was missing.

**Root cause**
- Usage was read only at the end of the stream.
- `message_delta` counts are cumulative but were added with `+=`.
- Delta fields are nullable, and Portkey omits `input_tokens`, but the code overwrote unconditionally.
- Moonshot reports usage on `choice.usage`.

**Fix · [[pi]]**
- `bc8d994a7` 2025-10-26: capture usage at `message_start`, so aborted streams keep their input counts, and assign with `=` rather than `+=` (`packages/ai/src/api/anthropic-messages.ts:680-686`).
- `8b5c81f21` 2026-01-29: overwrite only non-null fields.
- `0e6909f05` 2026-07-13: skip the block when `usage` is absent (`anthropic-messages.ts:826-853`).
- `453541530` 2026-03-10: fall back to `choice.usage` (`packages/ai/src/api/openai-completions.ts:568-579`).

**Lesson** Treat streamed usage as partial cumulative patches. Capture it at the earliest event, so aborts and errors still account for spend.

Related: [[usage-cost-accounting]] · [[errors-as-stream-events]] · [[usage-double-counting]] · [[pi--usage-cost-accounting|pi]]
