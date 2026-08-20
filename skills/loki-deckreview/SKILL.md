---
name: loki-deckreview
description: Render every slide of a built deck to images and run six review passes (memo/flow, visual via loki-uatu, number grounding, slop lexicon, story beats, and a cold read of the entire rendered deck in sequence), producing severity-tagged findings routed to the responsible upstream stage. Use whenever a deck.pptx exists and needs QA, whenever the user says "review the deck", "check the slides", "QA", "does the narrative hold", or before any deck ships. This is the mandatory final loki gate — a deck that hasn't passed deckreview is a draft, not a deliverable.
---

# loki-deckreview

Decks have no compiler, so review manufactures one: render to pixels, assert
against the specs, read the argument end to end. The single most important
rule: **findings route upstream.** A flow problem is a storyline bug; a
missing annotation is an exhibit-spec or build bug. Patching symptoms in the
slides leaves the artifact that produced them wrong, and the next build
regresses.

## Step 0 — Render

```bash
soffice --headless --convert-to pdf deck.pptx     # or the pptx skill's soffice.py wrapper
rm -f slides/S-*.png && mkdir -p slides
pdftoppm -png -r 150 deck.pdf slides/S
```

Inspect the PNGs fresh — after staring at generating code you see what you
expect, not what rendered. Also extract the title ladder as-built:

```python
from pptx import Presentation
p = Presentation("deck.pptx")
for i, s in enumerate(p.slides, 1):
    t = s.shapes.title.text if s.shapes.title else "(NO TITLE)"
    print(f"S{i:02d} | {t}")
```

## The six passes

**Pass 1 — Memo & flow.** Diff extracted titles against the frozen ladder:
any mismatch is SEV-1 (title drift). Re-run the memo test on the *as-built*
titles: read top to bottom; insert "therefore/but" at every junction. Read
the speaker-notes voiceover straight through — a transition that needs
forcing marks a weak junction. Route failures to **loki-storyline**.

**Pass 2 — Visual (delegated to loki-uatu).** Invoke **loki-uatu** on the
built deck: Lens 1 geometry (deterministic XML checks — overlaps,
off-canvas, alignment rails, title drift, font minimums), Lens 2 region
eyes (spec checklists asserted crop by crop: title band, exhibit canvas,
figures, footer), Lens 3 gestalt (contact sheet, squint test, accent
audit). Uatu's findings merge into review.md with the same schema and
routes. Overflow/clipped/shrunk text stays the first thing checked — most
common defect.

**Pass 3 — Grounding.** Extract every number visible on every slide
(markitdown or the title/body dump). Trace each to a cell in `data/E##.csv`
or a `[D#]` dump reference. Rounding must match the spec's stated precision.
**Any number with no home is SEV-1** — this is the pass that keeps
"persuasive" honest, and it is never skipped for time. Also verify FACT/EST/
BELIEF framing survived the build: an EST presented with FACT confidence is
a grounding failure even if the number itself is right.

**Pass 4 — Slop.** Scan all slide text and notes against the no-ai-slop
lexicon (unlock, leverage, journey, robust, holistic, empower,
transformative, delve, synergy, "in today's fast-paced…"). Titles were
linted at storyline; this catches slop that crept in through body text,
kickers, and notes during build.

**Pass 5 — Story.** Judgment pass against the beat sheet: Does the Cold Open
wound within the first slide, or does throat-clearing precede it? Does each
act land its job? Are High Noon options real (steelmanned) with criteria
shown *before* the recommendation and the "what would change our mind"
footnote present? Does The New Order restate the ask with owner and date?
Does theme intensity match STYLE.md's dial for this audience?

**Pass 6 — The cold read (the deck in its entirety).** Only after the
decomposed passes: read the rendered slides front to back, in order, full
PNGs — not extracted text — the way a silent pre-read audience will,
because bank decks circulate without a presenter and the deck must argue
with its author absent. Track three things:

1. **Title↔body coherence, per slide.** The body must prove *its own*
   title. A slide whose chart proves an adjacent claim can pass Pass 2
   (elements present) and Pass 3 (numbers sourced) and still fail here —
   this is the only pass positioned to catch it. Mismatch is SEV-1; route
   to exhibit (wrong message spec'd) or storyline (title over-claims).
2. **The question chain.** Write down the question each slide leaves in a
   reader's head; the next slide must engage it. This is the reader-side
   twin of the connective test — connectives verify the author's logic,
   the chain verifies the reader's experience, and they can diverge when a
   slide technically follows but answers something nobody was asking.
3. **Pacing and accumulation.** Where does attention sag (three dense
   exhibits in a row with no relief)? Does the argument *accumulate*
   toward High Noon, or reset? Stop at each act boundary and state the
   argument so far — if it can't be stated, the act didn't land in the
   artifact, whatever the beat sheet says.

The speaker-notes voiceover is explicitly OFF during this pass: notes are
for the room, and a deck that only works with the voiceover fails the
cold read.

## Findings

`review.md`:

```
| ID | Slide | Pass | SEV | Finding | Route |
|----|-------|------|-----|---------|-------|
| R01| S07   | 3    | 1   | "35bps" not in data/E03.csv (CSV says 32) | loki-exhibit |
| R02| S04→05| 1    | 2   | junction needs "but"; voiceover forces it  | loki-storyline |
| R03| S09   | 2    | 1   | annotation missing on peak datum per spec  | loki-build |
```

- **SEV-1 (blockers):** unsourced/mismatched number, title drift from ladder,
  missing mandatory checklist element, overflow/clipped text, chart-like
  content inside a generated figure, fake/strawman option at High Noon,
  title↔body mismatch from the cold read.
- **SEV-2:** emphasis or neutrality failures, weak junctions, contrast
  issues, EST/BELIEF framing softness.
- **SEV-3:** polish — spacing, wording, figure grading.

## Exit criteria

Ship when: **SEV-1 = 0**, memo test passes on as-built titles, the cold
read's question chain holds unbroken, and every
SEV-2 has an accept/fix decision recorded — by a human triaging, or by the
disposition policy pre-committed at the TVA triage gate when running
autonomously (defaults: SEV-1 fix, SEV-2 fix, SEV-3 accept). Findings that
route to *claims* — storyline titles/messages, spec'd numbers, missing data
— are never auto-fixed under policy: they terminate an autonomous run as
BLOCKED for the human. After any upstream fix,
re-render and re-run the affected passes — the PDF must be regenerated before
`pdftoppm` reflects changes, and a fixed storyline junction means Pass 1 runs
again in full.

One review cycle should converge. If a second full cycle still finds SEV-1s,
stop patching: the storyline or the specs are unstable, and the honest move
is a stage rollback, not a third lap of whack-a-mole.
