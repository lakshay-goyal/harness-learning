---
type: failure
concepts: [task-owned-subagent, tool-description-design]
harnesses: [opencode, codex]
---
**Symptom** — With a `task_status(task_id, wait, timeout_ms)` tool for background subagents, the model slept, polled for progress and re-checked status, burning turns instead of doing other work.

**Root cause** — A status tool invites busy-waiting; the model has no other way to learn completion.

**Fix · [[opencode]]** — `dabf2dc013` 2026-05-25 "remove the need for polling from experimental background agents" (#29179) deleted `task_status` (179 lines; `dabf2dc013^:packages/opencode/src/tool/task_status.txt`); completion is pushed as a synthetic user message "Background task completed: …" that starts a new parent turn (`packages/opencode/src/tool/task.ts:227-264`). `cc9b73b0bd` 2026-06-04 (#30790) added to the tool result "Do not poll for progress, ask the task for status, or duplicate this task's work"; models still slept between checks, so `b9131aa69c` 2026-06-06 (#31162) "lets kill this sleep behavior" escalated it to "DO NOT sleep, poll for progress, …" (`packages/opencode/src/tool/task.ts:31-35`).

**Lesson** — Async work should notify the model by injecting a message, not offer a poll tool.

Related: [[task-owned-subagent]] · [[follow-up-queue]] · [[delegated-work-duplicated]] · [[opencode--task-owned-subagent|opencode]]

## Also: [[codex]] (folded from `orchestrator-busy-polls-subagents`)
**Symptom** — With collab (multi-agent) enabled, a prompt like "Examine the code at <deeply nested project>…" made the top-level agent "busy wait on subagents" via rapid short-timeout `wait` calls → high CPU and token burn (`375a5ef051` body).

**Root cause** — Models poll with the smallest allowed timeout; the wait tool had no floor.

**Fix · [[codex]]**
- `375a5ef051` 2026-01-26 "fix: attempt to reduce high cpu usage when using collab (#9776)" — `MIN_WAIT_TIMEOUT_MS` = 10 000 clamp ("Minimum wait timeout to prevent tight polling loops from burning CPU", `codex-rs/core/src/tools/handlers/multi_agents_common.rs:1-4`) + prompt "Do not busy-poll `wait` with very short timeouts. Prefer waits measured in seconds (or minutes)" (handler then in `af434b4f71:codex-rs/core/src/tools/handlers/collab.rs`).
- Later description: "Call wait_agent very sparingly… Do not repeatedly wait by reflex." (`codex-rs/core/src/tools/handlers/multi_agents_spec.rs:692-770`); `8a1c941439` 2026-07-27 "Recommend longer waits in the v2 wait_agent schema"; `4d7e3e90d9` 2026-08-07 clamp (instead of reject) sub-minimum timeouts and report the adjustment in the result. HEAD: `DEFAULT_MULTI_AGENT_V2_MIN_WAIT_TIMEOUT_MS` = 10 000 (`codex-rs/core/src/config/mod.rs:259`).

**Lesson** — Blocking-wait tools need a minimum timeout enforced in code, stated in the description, and reported in the result — prose alone doesn't stop polling.

Related: [[task-owned-subagent]] · [[tool-description-design]] · [[subagent-result-mailbox]] · [[codex--task-owned-subagent|codex]]
