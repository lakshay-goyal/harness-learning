---
type: failure
concepts: [shell-execution, tool-argument-repair]
harnesses: [pi]
---
**Symptom** — A model-chosen bash `timeout` that was huge (e.g. hours in seconds → > 2^31-1 ms) or non-positive/NaN fired **immediately**, killing the command at once with a misleading timeout.

**Root cause** — Node's `setTimeout` treats delays > 2,147,483,647 ms (int32) or invalid values as ~1 ms; the value was passed through unvalidated/clamped.

**Fix · [[pi]]**
- `cbcf4e04c` 2026-06-30 (#6181) — reject > `MAX_TIMEOUT_MS = 2_147_483_647` with "Invalid timeout: maximum is 2147483.647 seconds" (`packages/coding-agent/src/core/tools/bash.ts:22-38`).
- `85b7c2474` 2026-07-01 — reject non-finite / ≤0: "Invalid timeout: must be a finite number of seconds".
- Durable mirrors: `MAX_TIMEOUT_SECONDS = 2_147_483_647/1000` (`packages/durable/src/tools/bash.ts:8,46-54`).

**Lesson** — Validate model-supplied numbers against the runtime's real limits and reject with an explanation rather than silently clamping.

Related: [[shell-execution]] · [[tool-argument-repair]] · [[no-bash-default-timeout]] · [[pi--shell-execution|pi]]
