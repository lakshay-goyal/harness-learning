---
type: absence
harnesses: [opencode]
---
# no-hardcoded-secret-refusal

Removed: the hardcoded refusal to read `.env` files. Replaced by overridable permission rules.

**What's missing**
- No code path that unconditionally refuses `.env`. Default rules instead (`packages/opencode/src/agent/agent.ts:129-134`, "mirrors github.com/github/gitignore Node.gitignore pattern for .env files"):
  - `read: {"*.env": "ask", "*.env.*": "ask", "*.env.example": "allow"}`.

**Evidence of decision**
- `3611260405` (2026-01-04) "core: remove hardcoded .env read block and use new permissions model instead". Lands right after the permission rework `351ddeed91` (2026-01-01, #6319).

**Implication**
- Policy as data: users can loosen or tighten it, and one engine evaluates it ([[permission-ruleset]]).
- Gaps observed (not verified as exploits): the rule binds only the `read` permission, so `bash cat .env` is judged as a bash pattern; `grep` passes `--hidden` (`packages/core/src/ripgrep.ts:222-228`); an "always" reply on any read approves `read:*`. A per-tool secret rule does not cover other tools that read the same bytes.
- pi: the same idea lives only in the `protected-paths.ts` example (substring `.env`, write/edit only) → [[no-permission-prompts]].

Related: [[secret-handling]] · [[permission-ruleset]] · [[tool-call-gate]] · [[opencode]] · [[Absences]]
