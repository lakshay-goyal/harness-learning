---
type: concept
stage: messages
tier: candidate
aliases: [SYSTEM.md, APPEND_SYSTEM.md, "--system-prompt", "--append-system-prompt", before_agent_start, forceSystemPrompt, systemPromptOptions]
harnesses: [pi]
---
User and plugin control over the base prompt: append, replace the harness base while keeping environment sections, or fully force a prompt — plus a structured hook to mutate named sections.

## Why
- Users/teams need to change tone, workflow or domain without forking the harness.
- Full-text replacement hooks destroy structure (section patching, caching, tool guidance); chained plugins need to see each other's edits ([[prompt-hook-chain-sees-stale-prompt]]).
- A forced prompt delivered as a late update leaves the original instructions in charge on models with native mid-conversation system messages ([[forced-system-prompt-applied-as-late-update]]).
- Repo-supplied prompt replacement is an injection vector → trust gating.

## Design space
- Append-only file/flag (**pi** APPEND_SYSTEM.md / `--append-system-prompt` → `addendum` section).
- Replace base but keep context/skills/cwd (**pi** SYSTEM.md / `--system-prompt` → preamble only).
- Replace everything (**pi** `before_agent_start` returning `systemPrompt` → `forceSystemPrompt`, projected not recorded).
- **Structured mutable options with named sections, chained across handlers** (**pi** `systemPromptOptions`).
- Project-local overrides gated by trust (**pi**) vs always honored.

## Implementations
- [[pi--system-prompt-override|pi]] — three tiers; hook chained; forced prompt projected onto request head; project files trust-gated.

## Failures
- [[forced-system-prompt-applied-as-late-update]]
- [[prompt-hook-chain-sees-stale-prompt]]
- [[markdown-boundaries-ingested-inconsistently]]

## Related
[[minimal-system-prompt]] · [[transcript-carried-system-prompt]] · [[extension-event-hooks]] · [[project-trust-gate]] · [[context-file-hierarchy]] · [[harness-evals]]
