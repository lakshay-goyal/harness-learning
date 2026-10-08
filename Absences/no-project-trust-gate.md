---
type: absence
harnesses: [opencode]
---
# no-project-trust-gate

Opening a repo executes its config. No "trust this folder?" prompt.

**What's missing**
- Project `opencode.json`, `.opencode/plugin(s)/*.{ts,js}` and `.opencode/tool(s)/*.{ts,js}` are imported and run when a session starts in that directory, with no prompt:
  - tools: `Glob.scanSync("{tool,tools}/*.{js,ts}")` over every config dir, then `import(pathToFileURL(match))` (`packages/opencode/src/tool/registry.ts:183-197`);
  - plugins: `Glob.scan("{plugin,plugins}/*.{ts,js}")` (`packages/opencode/src/config/plugin.ts:21`).
- The only lever is a global env kill switch, `OPENCODE_DISABLE_PROJECT_CONFIG` (`packages/opencode/src/config/config.ts:420`).

**Evidence of decision**
- `SECURITY.md:33` (Out of Scope): "**Malicious config files**: Users control their own config; modifying it is not an attack vector".
- `SECURITY.md:32`: "External MCP servers you configure are outside our trust boundary". Project MCP config is not gated either.
- Threat model written in `207a59aad4` (2026-01-14).

**Contrast**
- pi gates `.pi/{settings.json, mcp.json, extensions, skills, prompts, …}` behind a trust prompt (`packages/coding-agent/src/core/trust-manager.ts:30-39`) → [[pi--project-trust-gate|pi]]. pi's gate guards **input loading**, not actions ([[no-permission-prompts]]); opencode guards actions ([[permission-ruleset]]) but not input loading. Each harness has the half the other lacks.

**Implication**
- Cloning a hostile repo and running `opencode` in it runs the repo's TypeScript with the user's privileges, before any permission rule applies. The permission system cannot help, because the plugin code itself is the attacker.
- Related accepted risk: AGENTS.md / instruction files load without trust too → [[no-prompt-injection-defense]].

Related: [[project-trust-gate]] · [[runtime-plugin-loading]] · [[plugin-tools]] · [[supply-chain-pinning]] · [[no-sandbox]] · [[no-prompt-injection-defense]] · [[opencode]] · [[Absences]]
