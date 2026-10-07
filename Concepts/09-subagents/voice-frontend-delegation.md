---
type: concept
stage: subagents
tier: candidate
aliases: [voice-frontend-delegation-prompt, realtime conversation, "/voice", "thread/realtime", realtime-webrtc, voice-host, "Background agent finished", REALTIME_BACKEND_TEXT_PREFIX, "[BACKEND]", backend_prompt.md, "<realtime_conversation>", "<startup_context>"]
harnesses: [codex]
---
A realtime speech model is the user-facing conversationalist; it hands textual tasks to the coding agent as a "background agent", receives streamed `[BACKEND]` progress plus the agent's final message, and both sides get prompts describing the split and the transcript nature of input.

## Why
- Speech needs low latency and conversational persona; coding needs a slow, tool-using agent — one model can't be both.
- The coding agent receives ASR transcripts (unpunctuated, misrecognized) and must be told so; the voice model must not expose the split ("do not mention 'backend'").
- The voice model starts cold: it needs a bounded, possibly stale summary of recent work and workspace.

## Design space
- **Front**: same model for voice and code · realtime model fronting, coding agent as background executor (✔ codex).
- **Back-channel**: final answer only · streamed progress flushed periodically with truncation marker + final message (✔ codex).
- **Context for the front model**: none · token-budgeted startup context of recent threads + workspace tree, flagged "may be incomplete or stale" (✔ codex).
- **End of session**: discard · hand transcript tail to the agent (✔ codex).
- **Media**: in-process audio · same-build helper process owning devices with privacy controls prioritized (✔ codex `voice-host`).

## Implementations
- [[codex--voice-frontend-delegation|codex]] — `codex-rs/core/src/realtime_conversation.rs` + `realtime_context.rs`; prompts `codex-rs/prompts/templates/realtime/*`; `codex-rs/realtime-webrtc`, `codex-rs/voice-host`.

## Failures
none recorded.

## Related
[[in-process-subagent-threads]] · [[session-handoff]] · [[message-role-layering]] · [[context-file-hierarchy]] · [[client-server-session-split]]
