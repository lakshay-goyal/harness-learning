---
type: concept
stage: messages
tier: must-have
aliases: [SYSTEM.md, APPEND_SYSTEM.md, "--system-prompt", "--append-system-prompt", before_agent_start, forceSystemPrompt, systemPromptOptions, model_instructions_file, developer_instructions, "config.base_instructions", include_permissions_instructions, include_environment_context]
harnesses: [pi, opencode, codex]
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
- **Replace the model's catalog prompt** via config (`base_instructions`, `model_instructions_file` — "Users are STRONGLY DISCOURAGED from using this field") while all dynamic fragments (permissions, AGENTS.md, environment) still arrive as separate messages ✔ codex.
- **Append as a separate developer-role message** (`developer_instructions`) rather than into the system prompt ✔ codex → [[message-role-layering]].
- Per-block kill switches for harness-injected fragments (`include_permissions_instructions`, `include_apps_instructions`, `include_collaboration_mode_instructions`, `include_environment_context`) ✔ codex.
- Side-prompt overrides: compaction prompt (`compact_prompt` / `experimental_compact_prompt_file`) ✔ codex; personality section removal (`personality = "none"`) ✔ codex.
- Internal replacement for sub-tasks: review child gets `base_instructions = REVIEW_PROMPT` ✔ codex → [[review-subagent]].
- Agent definition replaces the family prompt while harness sections are still appended (opencode) → [[agent-profiles]].
- Per-message `system` field appended (opencode).

## Implementations
- [[pi--system-prompt-override|pi]] — three tiers; hook chained; forced prompt projected onto request head; project files trust-gated.
- [[codex--system-prompt-override|codex]] — config replaces the catalog prompt wholesale; `developer_instructions` appended as developer message; per-fragment include toggles; compaction prompt override; no plugin hook.
- [[opencode--system-prompt-override|opencode]] — agent `prompt` replaces the family prompt (env, instructions, skills still appended); per-message `system` appended; plugin system transform may mutate, re-joined to ≤ 2 blocks.

## Failures
- [[forced-system-prompt-applied-as-late-update]]
- [[prompt-hook-chain-sees-stale-prompt]]
- [[markdown-boundaries-ingested-inconsistently]]

## Related
[[minimal-system-prompt]] · [[transcript-carried-system-prompt]] · [[extension-event-hooks]] · [[project-trust-gate]] · [[context-file-hierarchy]] · [[harness-evals]] · [[single-vs-per-model-system-prompt]]
