---
name: harness-atlas
description: Study an open-source coding-agent harness (opencode, pi, codex, aider, cline, goose, openhands, crush, continue, or similar) and merge the findings into a cross-harness Obsidian knowledge graph at ~/Desktop/harness-atlas. Mines prompt/tool-description git history as a failure log, maps the five organs (loop, message builder, tools, provider, session store), extracts hardcoded constants and deliberate absences, then writes dense concept / implementation / failure / tradeoff notes with file:line and commit evidence, and ends with a dated digest. Use whenever the user points at an agent harness repo and wants to learn it, compare it with other harnesses, add it to the atlas, find must-haves vs harness-specific ideology, or asks "what can I learn from this harness".
---

# Harness Atlas

Turn one harness repo into evidence-backed notes merged into a shared vault. The vault compounds: every new harness strengthens existing concept notes instead of creating parallel ones.

- Vault: `~/Desktop/harness-atlas` (override if the user gives another path)
- Schema, templates, naming rules: `references/schema.md` — read before writing any note
- Mining commands: `references/mining.md`

## Inputs

1. Repo path (clone first if given a URL; `git clone --filter=blob:none` keeps history cheap)
2. Harness slug, kebab-case (`opencode`, `pi`, `codex`)
3. Record `git rev-parse --short HEAD` — every note cites it

Scope check: a harness drives a model in a loop with tools. Model libraries, inference servers, and agent frameworks-for-others are out of scope; say so and stop.

## Pipeline

Run passes 1–5 as parallel subagents when available (each returns conclusions with file:line + hashes, not dumps). Then do 6–8 yourself.

1. **Organs** — locate the loop, message builder, tool registry, provider layer, session store. Per organ: key files, entry functions, termination conditions, cancellation path end to end.
2. **Prompt archaeology** (highest yield) — every system prompt, tool description, compaction/title/summary prompt. For the most-edited files, `git log -p --follow`. Each added sentence = a model failure hit in production; each removed sentence = a behavior the model outgrew or a rule that cost more than it saved. Classify into failure notes.
3. **Fix commits** — grep history for compaction, cache, truncat, overflow, retry, abort, interrupt, doom, loop, reasoning, tool call, permission. Keep ones that reveal a failure mode + fix.
4. **Constants census** — every limit, threshold, timeout, cap, TTL, retry count with file:line. Each is a tradeoff.
5. **Absences** — no embeddings? no sandbox? no planner? removed tools? Find the commit/issue/spec explaining why. Rejected designs are knowledge.
6. **Resolve concepts** — for each finding, look up existing `Concepts/` notes by name AND `aliases`. Reuse if it matches; create only if genuinely new. Put the harness's own term into `aliases`. Never create a near-duplicate.
7. **Write / merge notes** per `references/schema.md`:
   - `Harnesses/<slug>.md` (new or refreshed)
   - New concepts go into a `Concepts/<NN-group>/` folder + a line in its overview note
   - `Implementations/<slug>/<slug>--<concept>.md` per concept the harness implements
   - Append the harness to each concept's `harnesses:` list
   - `Failures/` — new failure, or append this harness's evidence to an existing one
   - `Tradeoffs/` when two harnesses choose differently
   - `Absences/` for deliberate non-features from pass 5
   - Update `Constants.md` rows
8. **Digest** — `Digests/YYYY-MM-DD-<slug>.md`: 3 most surprising findings, 1 thing to build, 1 question to argue, links to new/changed notes. This is what the user reads; the graph is reference.

## Writing rules

- Dense: fragments, tables, bullets. No intros, no restating the title, no "this note describes".
- Every non-obvious claim carries evidence: `path:line`, commit hash, or spec path. No evidence → don't write it, or mark `unverified`.
- Neutral concept names (`tool-output-spill`, not opencode's term). Harness vocabulary → aliases.
- Link liberally with `[[wikilinks]]`; link failures ↔ concepts ↔ implementations.
- Mark `tier` on concepts after merging: `must-have` if ≥ 60% of studied harnesses implement it (min 3 harnesses studied), otherwise `variant`; before 3 harnesses, use `candidate`.
- Small repos with thin history: lean on current prompts, issues, and README rationale; say the history was thin in the digest.

## Validate before finishing

- Every new note has frontmatter matching the schema.
- No concept duplicated under another name (grep aliases).
- Every implementation note links its concept; every failure links ≥1 concept.
- Digest exists and links what changed.
