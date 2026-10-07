---
type: implementation
harness: codex
concept: self-documentation-pointer
commit: 622e9e3696
files: [codex-rs/skills/src/lib.rs:55, codex-rs/skills/src/lib.rs:57, codex-rs/skills/src/assets/samples/openai-docs/SKILL.md]
---
[[self-documentation-pointer]] in [[codex]].

## Mechanism
- Five sample skills compiled into the binary (`include_dir!` of `codex-rs/skills/src/assets/samples`, `codex-rs/skills/src/lib.rs:55`) and materialized under a `.system` skills dir with a salted marker file (`codex-rs/skills/src/lib.rs:57-60`): imagegen (19 KB), openai-docs (5.4 KB + references incl. `codex-self-knowledge.md`), review-agent, skill-creator (15 KB), skill-installer.
- No docs section in the system prompt: the docs are reached through the ordinary skills catalog ([[codex--skill-progressive-disclosure]]), so routing depends entirely on the skill *description* matching the user's phrasing.
- Current openai-docs description: "Use for Codex models/pricing, scheduled tasks, skills, settings, setup, troubleshooting, customization, automations, and self-knowledge—including 'you,' 'your,' 'this app,' or 'this coding agent' when they refer to Codex"; references split "with at most one primary route loaded per request" (`codex-rs/skills/src/assets/samples/openai-docs/SKILL.md` frontmatter).
- The skill is also the vehicle for post-cutoff product knowledge: commits bump the bundled latest-model guidance (`6bb2fa3fd4` GPT-5.5, `37c6f2916f` GPT-5.6, `12fe9f822c` GPT-6 Astra).

## Evolution
- 2026-03-10 `b7f8e9195a` openai-docs only "how to build with OpenAI products or APIs".
- 2026-05-28 `a4ed6c5aa0` added "asks about Codex itself or choosing Codex surfaces… use the Codex manual helper first for broad Codex self-knowledge".
- 2026-07-14 `83a4187837` review-agent system skill (detached reviews) → [[review-subagent]].
- 2026-07-29 `a5082373f1` description rewritten to enumerate deictic forms ("you", "your", "this app", "this coding agent"); one primary reference route per request → [[self-reference-not-routed-to-docs]].

## Versus pi
- [[pi--self-documentation-pointer]] puts absolute doc paths + a 13-topic map into the prompt (~45 % of it) gated by "only when the user asks about pi". codex keeps the prompt free of docs and relies on skill-description matching, which needed explicit pronoun forms to trigger.
