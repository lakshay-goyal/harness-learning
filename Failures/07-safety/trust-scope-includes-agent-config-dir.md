---
type: failure
concepts: [project-trust-gate]
harnesses: [pi]
---
**Symptom** — Running pi from `$HOME` treated the user's own global state (`~/.pi/agent`, `~/.agents/skills`) as project-local trust input and prompted to "trust" the home directory; `pi update` prompted for trust; the subagent example re-prompted for repo-agent confirmation even in already-trusted projects.

**Root cause** — Trust triggers were path-pattern based (`<cwd>/.pi/…`, ancestor `.agents/skills`) without excluding the agent's own config locations, which coincide with project paths when cwd = `$HOME`; plugin-level confirmations didn't consult the core trust decision.

**Fix · [[pi]]**
- 0.79.2 (#5619, hash unverified) — ignore global `~/.pi/agent` state from `$HOME`; `pi update` uses only saved/explicit trust (`packages/coding-agent/CHANGELOG.md:1619`). HEAD: `hasTrustRequiringProjectResources` skips `~/.agents/skills` "even when cwd is $HOME" (`packages/coding-agent/src/core/trust-manager.ts:180-208`).
- `8af7690c4` 2026-08-18 (#8261) — subagent example skips its repo-agent prompt in trusted projects (`examples/extensions/subagent/README.md:65`).

**Lesson** — A trust boundary must exclude the agent's own configuration directories, and plugins should reuse the core trust decision instead of re-asking.

Related: [[project-trust-gate]] · [[subagent-as-subprocess]] · [[pi--project-trust-gate|pi]]
