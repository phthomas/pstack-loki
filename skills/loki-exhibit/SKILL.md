---
name: loki-exhibit
description: Write exhibit specs (message, chart form, data table, mandatory element checklist) for every data visual in a deck BEFORE any chart is rendered. Use this whenever a storyline.md exists and slides need charts, graphs, tables, or data visualization; whenever the user mentions "exhibits", "chart specs", or complains that deck charts are missing elements or don't land. Runs after loki-storyline and before loki-build. Never let a chart be built without a spec from this skill.
---

# loki-exhibit

Every data visual gets a spec before it gets pixels. The spec states what the
chart must PROVE and enumerates every element it must contain — so the build
stage has a blueprint and the review stage has assertions. Charts without
specs are how elements go missing: nobody wrote down that the annotation had
to exist, so nobody noticed it didn't.

## Contract

- **Input:** frozen `storyline.md` (every ladder row with `visual: exhibit`).
- **Output:** `exhibits/E##.spec.md` + `data/E##.csv` per exhibit.
- **Gate:** every spec names its message, form, data, and passes the
  neutrality test (below) — on paper, before build.
- **Depends on:** the `clarity` skill, for the words on the chart (Step 3b).

## Step 1 — One message per exhibit

Copy the slide's action title into the spec. The exhibit exists to prove that
sentence and nothing else. If a chart seems to need two messages, it's two
exhibits or the ladder needs a split — route that upstream to loki-storyline,
don't cram.

## Step 2 — Choose the form (Zelazny logic)

The message's comparison type dictates the form — never the reverse:

| Message says… | Comparison | Form |
|---|---|---|
| "X is bigger/smaller than Y" | Item | Horizontal bar |
| "X grew/fell/flat over time" | Time series | Line (trend) or column (few periods) |
| "X is __% of the whole" | Component | Stacked bar / waterfall; pie only if ≤3 slices and you can defend it |
| "We got from A to B because of…" | Bridge | Waterfall |
| "X and Y move together" | Correlation | Scatter (+ trendline only with the r² shown) |
| "Distribution is skewed/bimodal" | Frequency | Histogram |
| "Market splits by segment × share" | Two-dimensional composition | Marimekko |
| "Rank shifted between two points" | Rank change | Slope / dumbbell |
| "Same pattern across many groups" | Multiples | Small multiples, shared axes |
| Qualitative comparison | Judgment | Harvey balls / 2×2 — spec these too |

When numbers beat charts: fewer than ~5 values the audience must *read*, not
compare → big-number layout or a clean table. A chart of three numbers is
decoration wearing a lab coat.

## Step 3 — The spec

```markdown
# E## — <slide S##>
title_verbatim: "<action title from storyline — immutable>"
message: <one sentence the chart must prove; usually = title>
form: <from table above> because <comparison type>
emphasis: <which series/bar/point carries the message>

## data
source: [D#] <origin, as-of date, FX basis if any>
file: data/E##.csv        # the ONLY numbers allowed on the slide

## element checklist  (review will assert each line against the render)
- [ ] kicker or chart title states the message (≤8 words)
- [ ] axis labels with UNITS (IDR bn vs USD mm vs % — never ambiguous)
- [ ] time basis labeled (YoY/QoQ/YTD; fiscal vs calendar)
- [ ] source line, bottom-left, 8–9pt: "Source: <[D#]>, as of <date>"
- [ ] annotation ON the key datum stating the so-what (≤8 words)
- [ ] emphasis: message-carrier in accent color; ALL other series muted grey
- [ ] reference line if the claim is vs. a benchmark/target/peer
- [ ] direct series labels (legend only if direct labeling is impossible)
- [ ] n= / sample basis disclosed if survey or subset
- [ ] axis starts at 0 for bars; any truncation disclosed on the axis

## banned (exceptions require a note in this spec)
dual axes · 3D · rainbow palettes · pie >3 slices · smoothing that hides
reversals · undisclosed axis truncation
```

## Step 3b — Clarity pass on chart words

The kicker and the key-datum annotation are the only prose a chart carries,
and readers hit them before the axes. Write them into the spec as literal
strings (`kicker: "..."`, `annotation: "..."`), then invoke **clarity** in
**rewrite** mode on ALL specs' kickers and annotations together, in slide
order, with the frozen title ladder as context:

- **Medium:** speech, talk, slides. **Register:** Argument.
- Each string must prove its own title in the ladder's terms — same nouns,
  same units, no synonym drift between title, kicker, and annotation.
- Numbers and units are untouchable: every figure must still equal a CSV cell.
- ≤8 words each; clarity marks `[TK]` rather than invent a qualifier.

Axis labels, units, and source lines are reference text — exact and
consistent beats readable; clarity does not touch them.

Because kickers and annotations live in the spec, they are claims: the
human reads them at triage, and build renders them verbatim.

## Step 4 — The neutrality test

Mentally strip the annotation and the accent color. If the remaining chart
reads *neutral* — if a stranger couldn't tell what it's arguing — the
emphasis layer is load-bearing and must be in the spec explicitly. A chart
that only works with its caption hasn't earned its slide. Conversely: if the
chart argues something the data doesn't support once you look at the CSV,
that's a grounding failure — fix the claim, not the chart.

Emphasis doctrine in one line: ~90% of the ink is quiet grey; exactly one
element is loud; the annotation says in ≤8 words why it's loud.

## Step 5 — Data contract

`data/E##.csv` is the single source of truth. Every number that appears on
the rendered slide must equal a cell in this file (rounding stated in the
spec, applied consistently — never mixed precision on one slide).
loki-deckreview diffs slide text against these CSVs; a number with no home
is a SEV-1. This is Law 4, and it is what "grounded" means mechanically.

## Integrity rules (banking-grade)

- Bar/column axes start at 0. Line charts may zoom, but say so on the axis.
- No dual axes by default — they exist to imply correlations the data won't
  claim. If truly needed, the exception and its reason go in the spec.
- Currency and units on every axis; note the FX rate and date if converted.
- Percentages: state the base ("% of NII", "% of accounts").
- Peer comparisons: same period, same definition, or a footnote saying why not.

## Routing

- Message unclear or doesn't match the data → **loki-storyline** (ladder bug).
- Data doesn't exist → dump gap; add `[D#]` or demote the claim.
- Form/element issues → fix here, re-spec, notify loki-build.
