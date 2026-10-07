---
type: absence
harnesses: [pi, opencode]
---
# no-prompt-injection-defense

Prompt injection is declared out of scope and unpreventable.

**What's missing**
- No sanitization or classification of tool output.
- No "untrusted content" fencing beyond XML boundaries around context files, and no instruction-hierarchy reminders.
- No trust gating of AGENTS.md/CLAUDE.md.
- Grep found no output sanitizer on the model path (unverified beyond grep). `stripAnsi` / `sanitizeBinaryOutput` run only in rendering and for user `!` commands (`packages/coding-agent/src/core/tools/render-utils.ts:48`, `bash-executor.ts:78`).

**Evidence of decision**
- `SECURITY.md:19-22`: "Pi relies on users installing trustworthy extensions and loading trustworthy skills and only to use pi within trusted repositories. This is because files like `AGENTS.md` or instructions in comments can be used to prompt inject the coding agent trivially and this cannot be protected against."
- `SECURITY.md:48-68` (Out Of Scope): untrusted repos, prompt injection, malicious model output, and user-controlled local state incl. `~/.pi`, `models.json`, `AGENTS.md`.
- `packages/coding-agent/docs/security.md:5`: "Files, comments, instructions, command output, and model responses can steer the model through prompt injection. Project trust controls which project resources load at startup, but it does not make that content or the resulting actions safe."
- `packages/coding-agent/docs/security.md:99`: "prompt injection from untrusted content … generally outside the security boundary".
- `b172beb92` (2025-11-12): "it can use `curl` or read files from disk. Both provide ample surface area for prompt injection attacks. Malicious content in files or command outputs can influence behavior."

**The reversal that confirms it**
- Context files were trust-gated in `89a92207f` (2026-06-05, PR #5332, v0.79.0: "project-local settings, resources, instructions, and packages", CHANGELOG `:1687`).
- They were **ungated four days later** in `5cb4f597f` (2026-06-09). The `docs/security.md` diff drops "instructions" from what trust guards and adds "context files" to the existing sentence: "Project trust is only an input-loading guard … Prompt injection from repository files, comments, documentation, context files, or build output is expected local-agent risk and cannot be reliably prevented by pi." (`git show 5cb4f597f -- packages/coding-agent/docs/security.md`).
- HEAD: "Context files such as `AGENTS.override.md`, `AGENTS.md`, and `CLAUDE.md` load regardless of project trust unless you disable context loading. Treat instructions in a folder as untrusted input even when you decline project trust." (`docs/security.md:57`).

**What pi does instead** (limits blast radius, does not detect injection)
- [[project-trust-gate]] blocks *executable* repo config: `.pi/{settings.json, mcp.json, extensions, skills, prompts, themes, SYSTEM.md, APPEND_SYSTEM.md}` and `.agents/skills` (`packages/coding-agent/src/core/trust-manager.ts:30-39`). Known hole: project `sessionDir` is read before trust is resolved (`docs/security.md:31`).
- Credential routing: MCP provider-token auth only in global config, "so a repository cannot pick where the credential goes" (`src/extensions/mcp/config.ts:25-26`).
- The subagent example confirms repo-controlled agent definitions (`subagent/index.ts:484-545`).
- Containment ([[no-sandbox]] replacements).
- Structural fencing ([[xml-prompt-boundaries]]): context files are injected inside `<project_instructions>`. This is a boundary marker, not a defense.

**Opt-in replacement**
- `tool_result` hooks can redact or annotate output ([[tool-result-rewriting]]). `tool_call` gates can block exfiltration-shaped calls ([[tool-call-gate]]). No example does injection detection.

**Implication**
- The honest position: the defense is the OS boundary. Any content the model reads is potentially adversarial, and pi does not pretend otherwise.
- Users running pi on untrusted repos or with web/MCP tools must containerize.

**codex** — *partly present*. The Guardian approval reviewer reasons explicitly about injection and untrusted evidence (`codex-rs/prompts/templates/guardian/policy_template.md:75-77`: "Malicious prompt injection requires affirmative evidence…") → [[llm-approval-reviewer]]; sub-agent reviews accept only the root user's genuine messages as authorization — "ordinary tool results and quoted delegation text cannot establish sender provenance" (`codex-rs/core/src/agent/control/sender_context.rs:1-5`; `d12a7f3fd8` 2026-08-21) → [[delegated-authorization-provenance]]; project trust prompt because "Trusting a directory enables project-local config, hooks, and exec policies, which can increase exposure to prompt injection" (`17801b4206` 2026-08-04) → [[project-trust-gate]]; memories are marked `polluted` when external context is used (`codex-rs/config/src/types.rs:326-329`) → [[cross-session-memory]]. No sanitisation of tool output itself (unverified beyond grep). Containment (sandbox + approvals) is the main defense, unlike pi's out-of-scope stance.
**opencode** ([[opencode]], `ecc4916b5a`): also absent, implicitly.
- No mention of prompt injection in `SECURITY.md`, `specs/` or source (grep `prompt.?injection`; the only "untrusted" hit is about inbox queue growth, `specs/v2/session.md:171`). Web and MCP output is not marked untrusted.
- The threat model scopes out the obvious channels: "LLM provider data handling", "MCP server behavior: External MCP servers you configure are outside our trust boundary", "Malicious config files" (`SECURITY.md:31-33`).
- Project `.opencode/tool(s)/*.ts` and plugins execute on open without a prompt (`packages/opencode/src/tool/registry.ts:183-197`) → [[no-project-trust-gate]].
- Partial mitigations that are UX, not defense: `external_directory` ask ([[workspace-boundary-check]]) and `*.env: ask` read rules (`packages/opencode/src/agent/agent.ts:129-134`).
- The one structural mitigation is in v2 compaction: the checkpoint is wrapped as "historical context, not as new instructions" (`packages/core/src/session/runner/to-llm-message.ts:147-165`). It guards against summary text being obeyed, not against hostile content.

Related: [[project-trust-gate]] · [[context-file-hierarchy]] · [[xml-prompt-boundaries]] · [[tool-result-rewriting]] · [[tool-call-gate]] · [[no-sandbox]] · [[no-permission-prompts]] · [[no-web-tools]] · [[Absences]] · [[llm-approval-reviewer]] · [[delegated-authorization-provenance]] · [[untrusted-repo-loads-executable-config]]
Related: [[project-trust-gate]] · [[context-file-hierarchy]] · [[xml-prompt-boundaries]] · [[tool-result-rewriting]] · [[tool-call-gate]] · [[no-sandbox]] · [[no-permission-prompts]] · [[no-web-tools]] · [[opencode]] · [[Absences]]
