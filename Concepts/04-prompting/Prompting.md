---
type: group
group: 04-prompting
---
Scope: everything the harness *says* to the model outside the conversation — system prompt construction and its contributors (tools, docs, instruction files, skills, templates, overrides), identity, structure/fencing, wording register, and side channels for session facts and harness remarks.

## Concepts
- [[minimal-system-prompt]] — Keep harness-authored prompt tiny; push behavior into tool contracts, docs-on-demand and env.
- [[dynamic-tool-guidelines]] — Each declared tool contributes its own prompt snippet + guidelines; rules generated from the active toolset; deduped.
- [[context-file-hierarchy]] — Instruction files (AGENTS.md/CLAUDE.md) discovered global → ancestors → cwd, injected with explicit boundaries.
- [[skill-progressive-disclosure]] — Only skill name/description/path in prompt; body read on demand.
- [[prompt-template-expansion]] — User-side slash macros with args expanding into the user message.
- [[system-prompt-override]] — Replace vs append base prompt (SYSTEM.md / APPEND_SYSTEM.md / --system-prompt) and plugin hook mutating sections.
- [[self-documentation-pointer]] — Prompt carries paths to harness docs, read on demand when user asks about the harness.
- [[harness-identity]] — Tell the model which harness it runs in; never override its model identity.
- [[xml-prompt-boundaries]] — Fence injected content with XML tags instead of markdown headings.
- [[guideline-softening]] — Downgrade imperative rules to permissive wording to stop over-compliance.
- [[env-vars-as-context]] — Expose session facts via env vars readable by tools rather than prompt text.
- [[harness-diagnostics-channel]] — Out-of-band harness remarks to the model in a delimited block separate from tool content.
- [[per-model-system-prompt]] — Several base prompts shipped; one picked per request from model id or provider.
- [[ephemeral-reminder-injection]] — Harness `<system-reminder>` text attached to the newest user message for mode changes, leaving the prefix stable.
- [[mention-expansion]] — `@file`/`@agent`/MCP-resource mentions resolved at message creation and stored as synthetic parts.

Key pi artifact: the full system-prompt timeline (45 changes, 19 removed rules, current full text) lives in [[pi--minimal-system-prompt]].

Failures: [[Prompting Failures]] · Neighbors: [[Caching]] (prompt prefix stability), [[Tools]] (tool descriptions), [[Context]] (compaction prompts).
