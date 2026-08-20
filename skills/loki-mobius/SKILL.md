---
name: loki-mobius
description: Ingest reference files that accompany a dump - past presentations, raw material decks/images, and visualization style examples - and distill them into blueprint-usable artifacts before compose runs. Use whenever the user uploads or mentions reference files alongside a deck task, says "use this as reference/example", "match this style", "like this chart", "here are past decks", or drops .pptx/.pdf/images with the dump. Runs BEFORE loki-storyline. If reference files exist and refs/manifest.md does not, run this skill first.
---

# loki-mobius

The analyst. Mobius reads the case files before anyone touches the timeline.
Reference files carry three different intents that look identical on disk —
a .pptx can be a style exemplar, a sack of raw material, or a chart-style
sample — and the role, not the file type, determines what gets extracted
and where it flows. Guessing the role silently is how a materials deck
quietly becomes the storyline template; Mobius never guesses.

## The three roles

| Role | The user means | Mobius extracts | Flows to |
|---|---|---|---|
| **exemplar** | "learn how we present" — style, storytelling, how slides are written | Title grammar, narrative rhythm, slide anatomy, register, palette/fonts | `refs/house-style.md` → storyline voice, STYLE.md, build conventions |
| **material** | "part of the story to be told, not yet in narrative form" | Claims, numbers with provenance, reusable assets, speaker points — as FRAGMENTS | Appended to `dump.md` + `[D#]` inventory + `refs/assets/` |
| **visual-anchor** | "carry this look through the deck" — charts, figures, doodles | Chart styling tokens; reference images for Seedream | STYLE.md chart tokens; `refs/visual-anchors/` → loki-figure's reference chain |

One file can hold two roles (a past deck that is both exemplar and material)
— declare both; each extraction runs.

## Step 0 — The manifest

Write `refs/manifest.md`: one row per file — filename, declared role(s),
what it is, as-of date if knowable. If the user didn't declare roles, infer
and **confirm before extracting** (pre-gate interaction, like storyline's
dump gate). A wrong role poisons everything downstream of it.

## Step 1 — Exemplars → refs/house-style.md

Read the deck properly: extract all text (python-pptx via the pptx skill),
render slides to PNG (soffice) and *study the pages*, and pull the theme —
`unzip -p deck.pptx ppt/theme/theme1.xml` yields the actual palette and
font pairs, which seed STYLE.md with the house's real colors instead of
invented ones.

Distill into two lists:

- **Imitate** — register and vocabulary (formal/direct/bilingual?), title
  *voice* with 3–5 verbatim examples of the best titles found, narrative
  rhythm (how many slides an argument breathes across, where dividers
  fall), slide anatomy conventions (kicker use, source-line format, tracker
  style), recurring layouts that clearly work.
- **Observed but superseded** — patterns the house uses that violate loki's
  lints: topic titles, agenda-first openings, legend-heavy charts, dual
  axes. Name them explicitly so nobody imitates them by accident.

**The exemplar trap, stated plainly:** the past decks predate loki and
often carry the exact diseases loki exists to cure — that's *why* it
exists. Mobius learns the register, never the violations: voice from the
house, discipline from the lint. And house-style informs *how things are
said*, never *what this deck argues* — importing an old storyline skeleton
is compose's job to resist and Mobius's job never to offer.

## Step 2 — Materials → dump fragments

Decompose, never narrate. The user said it precisely: materials are part of
the story *not yet in cohesive or narrative form* — forming the narrative
is compose's monopoly, so Mobius outputs inventory, not prose:

- **Claims** found in the material, quoted or tightly paraphrased, each
  tagged with its source slide/page.
- **Numbers → `[D#]` entries** with full provenance:
  `D7: NIM 2.9% — Q3-2025 ALCO deck S12, as-of Sep 2025 — VERIFY`.
  A number inherited from an old deck is *someone's claim then*, not truth
  now: it enters the inventory as the weakest evidence class, carries its
  as-of date, and gets a VERIFY flag when load-bearing. Deckreview's
  grounding pass treats deck-sourced numbers accordingly and flags
  staleness against the new deck's date.
- **Reusable assets** (photos, existing diagrams worth keeping) copied to
  `refs/assets/` with captions. Real photographs and real diagrams are
  placed AS-IS by loki-build — never regenerated through Seedream; the
  figure skill is for generated imagery only.
- **Speaker points** worth preserving, as bullets with source refs.

Appended to `dump.md` under a `## From materials (mobius)` section so the
dump remains the single compose input.

## Step 3 — Visual anchors

Two sub-types, two destinations:

- **Chart examples** → distill into STYLE.md chart tokens: how the house
  draws waterfalls, gridline weight, label placement, emphasis treatment.
  Exhibit specs inherit these automatically because STYLE.md is already
  their token source. Anchor the *drawing conventions*, never the data.
- **Figure/doodle examples** → copy to `refs/visual-anchors/`, write one
  descriptor line each (what about it must carry: linework, palette, mood),
  and fold the descriptors into the Seedream STYLE_BLOCK. loki-figure
  passes the anchor images as references alongside the approved hero —
  budget the 10-reference cap as hero + 2–3 anchors. A doodle on paper is
  a legitimate anchor: photograph it, anchor it, let 5.0 Pro carry its
  character into every divider.

## Outputs and wiring

```
refs/manifest.md          → confirmed roles (triage packet shows the digest)
refs/house-style.md       → loki-storyline (voice), STYLE.md (palette/fonts/anatomy)
dump.md (+fragments,[D#]) → compose, grounding pass
refs/assets/              → loki-build (placed as-is)
refs/visual-anchors/      → loki-figure (reference chain), STYLE.md (chart tokens)
```

TVA runs Mobius first in COMPOSE whenever reference files are present; the
triage packet then includes the manifest and the house-style digest so the
human confirms what was learned before anything is built on it.

## Anti-patterns

- Extracting an exemplar's storyline structure as this deck's storyline.
- Imitating observed title diseases because "that's how the house writes".
- Treating a number lifted from an old deck as verified because it was
  once presented confidently — confidence is not provenance.
- Regenerating real photos or real diagrams through the image model.
- Inferring roles silently. Ask once; extract right.
