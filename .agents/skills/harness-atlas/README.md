# harness-atlas — Claude Code skill

Studies an open-source coding-agent harness (opencode, pi, codex, aider, cline, goose, openhands, crush, continue, …) and merges the findings into a cross-harness **Obsidian knowledge graph**. It mines prompt and tool-description git history as a failure log, maps the five organs (loop, message builder, tools, provider, session store), pulls out hardcoded constants and deliberate absences, then writes concept, implementation, failure and tradeoff notes with `file:line` and commit evidence. Each run ends with a dated digest.

## Contents

```
harness-atlas/
  SKILL.md               main instructions (Claude reads this)
  references/
    schema.md            vault layout, note types, frontmatter templates
    mining.md            git commands used to mine a repo
```

## Install

Copy the `harness-atlas` folder into your Claude Code skills directory.

**For your user (works in every project):**
```bash
mkdir -p ~/.claude/skills
cp -R harness-atlas ~/.claude/skills/
```

**For one project only:**
```bash
mkdir -p .claude/skills
cp -R harness-atlas .claude/skills/
```

Restart Claude Code (or start a new session). Then check that `harness-atlas` appears when you type `/`.

## Use

```
/harness-atlas ~/code/opencode
```
You can also just ask: *"Study this harness repo and add it to my atlas: https://github.com/sst/opencode"*

- **Vault location:** `~/Desktop/harness-atlas` by default. To use another folder, name it in your prompt, e.g. *"…use ~/notes/atlas as the vault"*, or change the `Vault:` line in `SKILL.md`.
- The vault is plain Markdown. Open the folder in [Obsidian](https://obsidian.md) to browse the graph. The Dataview plugin is optional and only powers the `Home.md` dashboards.
- Needs `git`. A repo URL gets cloned with `git clone --filter=blob:none`.
- Runs its passes in parallel subagents when they are available.
