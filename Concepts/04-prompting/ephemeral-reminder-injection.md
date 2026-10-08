---
type: concept
stage: messages
tier: variant
aliases: ["<system-reminder>", SessionReminders.apply, PROMPT_PLAN, BUILD_SWITCH, PLAN_MODE, "synthetic: true", system-reminder-injection]
harnesses: [opencode]
---
Harness-authored `<system-reminder>` text is attached to the newest user message rather than the system prompt. It signals mode changes or nearby context while the stable prefix stays untouched.

## Why
- Mode state (plan vs build, last step) changes mid-session; editing the system prompt for it busts the cache and rewrites history ([[cache-stable-prompt-prefix]]).
- A restrictive reminder left in history keeps restricting after the mode ends unless a lifting reminder follows (opencode build-switch).
- The model must know how much authority the tag carries; prompts frame it differently.

## Design space
- **Re-attached in memory to the last user message every step, never persisted** (opencode legacy plan reminder).
- **Persisted as a synthetic part only on the transition step** (opencode experimental plan mode, build-switch).
- Inside a tool result (opencode read tool: nested AGENTS.md as `<system-reminder>`) → [[context-file-hierarchy]].
- Client-originated reminders (opencode TUI: opened file, cwd change).
- Trailing assistant-role message instead of a user part (opencode max-steps `MAX_STEPS_PROMPT`) → [[step-budget-limit]].
- Authority framing in the system prompt: "bear no direct relation" (anthropic.txt) vs "authoritative system directives that you MUST follow" (kimi.txt) vs "harness instructions, not user-authored content" (origin/v2) vs not mentioned (gpt.txt, codex.txt).
- Append prompt-delta sections to the transcript instead (pi, [[transcript-carried-system-prompt]]).

## Implementations
- [[opencode--ephemeral-reminder-injection|opencode]] — `SessionReminders.apply` pushes `plan.txt` / `build-switch.txt` / `plan-mode.txt` as `synthetic: true` text parts onto the last user message before each request.

## Failures
- [[step-limit-enforced-only-by-prompt]]
- [[stale-mode-reminder-persists]]
- [[prompt-names-unavailable-tools]]
- [[tool-description-drifts-from-implementation]]

## Related
[[plan-mode]] · [[step-budget-limit]] · [[transcript-carried-system-prompt]] · [[cache-stable-prompt-prefix]] · [[harness-diagnostics-channel]] · [[mention-expansion]] · [[context-file-hierarchy]]
