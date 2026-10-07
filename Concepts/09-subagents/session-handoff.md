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

## Implementations
- [[pi--session-handoff|pi]] — `examples/extensions/handoff.ts` (compaction-aware context → generated prompt → editor → new session); durable `reset(handoff)` / tool `control.handoff`.

## Failures
none recorded.

## Related
[[auto-compaction]] · [[structured-compaction-summary]] · [[transcript-serialization-for-summary]] · [[session-fork]] · [[subagent-as-subprocess]] · [[cache-retention-control]]
