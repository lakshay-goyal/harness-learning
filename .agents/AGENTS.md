# harness-learning — Obsidian vault + reverse-engineering lab

This repo is an **Obsidian vault** for learning how coding-agent harnesses work by
reverse engineering their source and git history.

## Vault

- **Vault root is this repo root.** Open `/Users/lakshay.goyal/Desktop/Learning/harness-learning`
  in Obsidian ("Open folder as vault").
- The `harness-atlas` skill ships with a default vault of `~/Desktop/harness-atlas`.
  **Always override it** when running the skill here: treat the repo root as the vault
  so notes land in `Concepts/`, `Failures/`, `Digests/`, etc. instead of a sibling folder.
- Vault layout and note frontmatter: `.meta/schema.md` (a copy of the skill's
  `references/schema.md` plus the vault-local **Color convention** section below —
  the upstream skill has no CSS snippet, so that section is this vault's own).
  Obsidian-side folders are dot-prefixed so they stay out of the
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

## Color convention

Reading rule for the whole vault: **red = a harness topic, blue = a concept.**

- **Red** — `[[pi]]`, `[[opencode]]`, `[[codex]]` and the implementation notes
  belonging to one (`[[pi--turn-loop|pi]]`).
- **Blue** — cross-harness concept notes (`[[turn-loop]]`, `[[os-level-sandbox]]`)
  and their group overviews (`[[Loop]]`).
- Default color — failures, tradeoffs, absences, digests, constants. Not a gap;
  two colors only, so the eye separates "who" from "what".

This is **rendered, not written**. Never add inline HTML or color markup to a note —
colors come from link targets, so notes stay plain markdown and a normal
`[[wikilink]]` is always correct.

### Inside notes — CSS snippet

- `.obsidian/snippets/harness-colors.css` — generated, enabled in
  `.obsidian/appearance.json`. Edit the four hex values in its `:root` /
  `body.theme-dark` blocks to change the palette; nothing else.
- `.meta/generate-link-colors.py` — regenerates the snippet from note frontmatter.

After adding, renaming, or deleting any note, run:

```
python3 .meta/generate-link-colors.py
```

It reads `type: harness` notes (red) and `type: concept` / `type: group` notes under
`Concepts/` (blue), plus any concept alias actually used as a link target. New harness
studied = one new red note, and the script picks it up. The snippet is committed, so a
fresh clone renders correctly without running anything.

### In the graph view — native color groups, not CSS

**CSS can never color the graph.** Graph nodes are SVG circles: no `.internal-link`
class, no `data-href`, so the snippet's selectors cannot match them. Node color comes
from Obsidian's Groups feature in `.obsidian/graph.json` → `colorGroups`, keyed by
search query:

| Query | Color | Matches |
|---|---|---|
| `path:"Harnesses"` | red `#d1242f` | the 3 harness topic notes |
| `path:"Concepts"` | blue `#0969da` | 215 concepts + group overviews |

Caveats:

- `collapse-color-groups` is `false` on purpose. With it `true`, every matching note
  collapses into one blob instead of staying individually colored.
- Groups only render after Groups is switched on in the graph filter pane.
- `graph.json` is Obsidian-owned state that Obsidian rewrites from memory. **Quit
  Obsidian before hand-editing it**, or the edit is lost on exit. Re-apply these two
  queries if they get dropped; keep the same hexes as the CSS palette so both views
  agree.

Known limit on both views: CSS cannot color plain prose, so harness mentions that are
*not* links — the `(✔ codex ...)` asides in **Design space** bullets — stay default
color. Leave them as plain text; do not wrap them in spans to force a color.

## Typical run

```
/harness-atlas ~/code/opencode — vault is this repo
```
