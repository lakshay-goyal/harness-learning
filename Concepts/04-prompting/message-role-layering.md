---
type: concept
stage: messages
tier: variant
aliases: [developer message, contextual user message, ContextualUserFragment, RenderedFragment, build_initial_context_with_world_state, separate_developer_sections, PromptSlot::DeveloperPolicy, PromptSlot::DeveloperCapabilities, role-authority-layering]
harnesses: [codex]
---
Each kind of harness text gets a fixed message role and position so the model can weigh authority: harness rules (base instructions) > host policy and capabilities (developer role) > repo/user context (user role, fenced) > the user's actual request.

## Why
- Injected repo instructions in a plain user message were read as part of the user's request ([[markdown-boundaries-ingested-inconsistently]], codex `063083af15`).
- Policy text (sandbox, approvals, modes) must outrank anything the user or a repo file says, and must be changeable mid-session without rewriting the cached prefix → developer messages appended over time.
- Mode/policy changes need a role the model treats as authoritative *and* a statement of who may change it ([[mode-state-confusion]]).

## Design space
- Everything in one system prompt (sections) ✔ pi ([[minimal-system-prompt]]).
- Base prompt in the API's top-level `instructions` field (codex until 2026-10-05) vs **a leading `developer` input message with a stable id** ✔ codex (`c9253c4977`).
- **Policy/capability fragments as developer messages** (permissions, collaboration mode, multi-agent, skills/apps/plugins, model switch, time reminder, token budget, persistent mode, memory, user `developer_instructions`) ✔ codex.
- **Context fragments as user messages with markers** (AGENTS.md, `<environment_context>`, `<user_shell_command>`, `<turn_aborted>`, compaction summary, subagent notifications, hook context) ✔ codex.
- Aggregation: one bundled developer message + one bundled contextual-user message per initial context, with listed exceptions kept separate (guardian policy "so the guardian subagent sees a distinct, easy-to-audit instruction block") ✔ codex.
- Markers double as classifier keys so the harness can tell its own user-role injections from real user input (rollback boundaries, compaction keep-set, UI) ✔ codex → [[xml-prompt-boundaries]].

## Implementations
- [[codex--message-role-layering|codex]] — base instructions as `developer` message `msg_<uuidv5>`; ~60 fragment types with role + markers + `ContentItemKind`; fixed initial-context assembly order.

## Failures
- [[markdown-boundaries-ingested-inconsistently]]
- [[mode-state-confusion]]

## Related
[[xml-prompt-boundaries]] · [[world-state-diff-injection]] · [[transcript-carried-system-prompt]] · [[context-file-hierarchy]] · [[plan-mode]] · [[per-model-system-prompt]] · [[current-time-reminder]] · [[cache-stable-prompt-prefix]] · [[single-vs-per-model-system-prompt]]
