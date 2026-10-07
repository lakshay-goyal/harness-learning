---
type: concept
stage: subagents
tier: candidate
aliases: [handoff.ts, "/handoff", "reset(handoff)", "control: {handoff}", pi.reset, context transfer]
harnesses: [pi]
---
Transfer distilled context into a fresh session/context (new thread seeded by a generated, self-contained prompt) instead of compacting the current one.

## Why
- Compaction is lossy and keeps the agent anchored to the old goal; a goal-directed transfer extracts only what the next task needs.
- Same problem as subagent result handoff: the receiver has not seen the history, so the transfer text must be self-contained.

## Design space
- **Trigger**: user command (pi example `/handoff <goal>`) · model-callable tool result control (pi-durable `control:{handoff}`) · automatic at threshold.
- **Generator**: side LLM call over serialized transcript with goal (pi) · user-written note (`reset(note)`).
- **Review**: draft into editor for human edit before sending (pi) · auto-submit.
- **Lineage**: new session with `parentSession` link (pi stable) · same conversation with a `head` reset entry, history kept in storage (pi-durable).
- **Cache hygiene**: one-off side call with no cache write + fresh session id.
- codex: no user-facing "handoff into a new session" command found in findings (unverified absence). Adjacent mechanisms: compaction prompt framed as "a handoff summary for another LLM that will resume the task" (`codex-rs/prompts/templates/compact/prompt.md:1-9`, [[auto-compaction]]); model-callable `new_context_window` reset carried by model notes ([[model-requested-context-reset]]); turn suspension "Handoff intentionally drops" in-process pending input for recovery by another worker (`codex-rs/core/src/session/turn_suspension.rs:1-119`, [[durable-execution]]); realtime end-of-session handoff of the transcript tail to the agent ([[voice-frontend-delegation]]).

## Implementations
- [[pi--session-handoff|pi]] — `examples/extensions/handoff.ts` (compaction-aware context → generated prompt → editor → new session); durable `reset(handoff)` / tool `control.handoff`.

## Failures
none recorded.

## Related
[[auto-compaction]] · [[structured-compaction-summary]] · [[transcript-serialization-for-summary]] · [[session-fork]] · [[subagent-as-subprocess]] · [[cache-retention-control]] · [[model-requested-context-reset]] · [[voice-frontend-delegation]]
