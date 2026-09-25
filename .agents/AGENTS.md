# harness-learning — Obsidian vault + reverse-engineering lab

This repo is an **Obsidian vault** for learning how coding-agent harnesses work by
reverse engineering their source and git history.

## Vault

- **Vault root is this repo root.** Open `/Users/lakshay.goyal/Desktop/Learning/harness-learning`
  in Obsidian ("Open folder as vault").
- The `harness-atlas` skill ships with a default vault of `~/Desktop/harness-atlas`.
  **Always override it** when running the skill here: treat the repo root as the vault
  so notes land in `Concepts/`, `Failures/`, `Digests/`, etc. instead of a sibling folder.
- Vault layout and note frontmatter: `.meta/schema.md` (copy of the skill's
  `references/schema.md`). Obsidian-side folders are dot-prefixed so they stay out of the
  vault UI; the skill's own `_meta` / `_templates` names map to `.meta` / `.templates`.

## Rules

- Do **not** edit the skill files under `.agents/skills/harness-atlas/`. They are
  intentionally byte-identical to the upstream package (SKILL.md, references/, README.md).
  Config and vault instructions live here, not in the skill.
- Every non-obvious claim in a note needs evidence: `path:line`, commit hash, or spec path.
  No evidence → mark `unverified` or drop it.
- Merge into existing concept notes (match by name **and** `aliases`) rather than
  creating near-duplicates.
- Harness repos are studied **read-only**; clone them outside the vault (e.g. `~/code/<slug>`)
  so mining output never lands in the vault.

## Typical run

```
/harness-atlas ~/code/opencode — vault is this repo
```
