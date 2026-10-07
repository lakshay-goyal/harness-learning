---
type: implementation
harness: codex
concept: voice-frontend-delegation
commit: 622e9e3696
files: [codex-rs/core/src/realtime_conversation.rs:97, codex-rs/core/src/realtime_context.rs:30, codex-rs/prompts/templates/realtime/backend_prompt.md, codex-rs/prompts/templates/realtime/realtime_start.md, codex-rs/prompts/templates/realtime/realtime_end.md, codex-rs/voice-host/src/main.rs:1]
---
[[voice-frontend-delegation]] in [[codex]].

## Mechanism
- **Split**: a realtime speech model (default `gpt-realtime-1.5`; frameless `gpt-live-1-codex`) talks to the user and hands textual tasks to the Codex agent as a "background agent"; the agent's progress streams back prefixed `[BACKEND] ` (user transcript `[USER] `) and its final message prefixed `"Agent Final Message":` (`codex-rs/core/src/realtime_conversation.rs:97-122`). Surface: TUI `/voice`, app-server `thread/realtime/*`.
- **Back-channel**: handoff stream flushed every 200 ms, truncated with "\n…output truncated…\n"; acknowledgements "Background agent finished. Use the preceding [BACKEND] messages as the result." / "This was sent to steer the previous background agent task." (steer into the running task, [[steering-queue]]); end of session: "The user just ended their realtime session. Here is the remaining handoff/transcript tail…" (`codex-rs/core/src/realtime_conversation.rs:108-122`) → [[session-handoff]].
- **Voice-side prompt** `codex-rs/prompts/templates/realtime/backend_prompt.md` (4,913 bytes): "When interacting with the user, do not mention 'backend'. Present every work as done by you."; persona "playful collaborator: super fun, warm, witty"; `{{ user_first_name }}`.
- **Agent-side prompts**: developer `<realtime_conversation>` fragments — start: "You are operating as a backend executor behind an intermediary… When user text is routed from realtime, treat it as a transcript. It may be unpunctuated or contain recognition errors." (`codex-rs/prompts/templates/realtime/realtime_start.md`); end: "Do not assume recognition errors or missing punctuation once realtime has ended." (`realtime_end.md`) → [[message-role-layering]].
- **Startup context** for the voice model: recent threads + workspace layout under per-section token budgets, wrapped `<startup_context>` with header "This is background context about recent work and machine/workspace layout. It may be incomplete or stale." (`codex-rs/core/src/realtime_context.rs:30-44`).
- **Media**: `codex-rs/realtime-webrtc` (WebRTC session/client, Linux ALSA) and `codex-rs/voice-host` (same-build helper process owning devices, capture/playback; "Queued privacy controls take priority over starting another capture batch", `codex-rs/voice-host/src/main.rs:1-2`).
- Prompt wiring: `codex-rs/prompts/src/realtime.rs:1-3` embeds `codex-rs/prompts/templates/realtime/backend_prompt.md` (`BACKEND_PROMPT`), `realtime_start.md` (`START_INSTRUCTIONS`) and `realtime_end.md` (`END_INSTRUCTIONS`) via `include_str!`.

## Constants
| name | value | path:line |
|---|---|---|
| startup context / assistant output budgets | 5 300 / 1 000 tokens | `codex-rs/core/src/realtime_conversation.rs:102-103` |
| initial items cap | 128 items / 8 192 tokens; mode instructions 8 192 tokens | `codex-rs/core/src/realtime_conversation.rs:105-107` |
| handoff flush interval | 200 ms | `codex-rs/core/src/realtime_conversation.rs:108` |
| default realtime model | `gpt-realtime-1.5` (frameless `gpt-live-1-codex`) | `codex-rs/core/src/realtime_conversation.rs:112-113` |
| startup section budgets | current thread 1 200 / recent work 2 200 / workspace 1 600 / notes 300 tokens; per realtime turn 300 | `codex-rs/core/src/realtime_context.rs:33-37` |
| recent threads / work groups | 40 / 8; tree depth 2, 20 entries per dir | `codex-rs/core/src/realtime_context.rs:38-44` |
| queue capacities | audio in 256, text in 64, handoff out 64, output events 256 | `codex-rs/core/src/realtime_conversation.rs:97-100` |

## Evolution
- 2026-02-20 `6817f0be8a` realtime wired to core.
- 2026-04-08 `4c2a1ae31b` realtime prompts moved into core; `d90a348870` WebRTC.
- 2026-04-10 `029fc63d13` "Strengthen realtime backend delegation prompt (#17363)".

## Versus pi
- pi has no voice front-end or realtime model integration (not in findings).
