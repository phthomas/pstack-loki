---
name: loki-build
description: Build the actual .pptx from a frozen storyline, exhibit specs, and approved figures, using a conductor/worker wave model where every worker receives the FULL storyline. Use whenever the user says "build the deck", "generate the slides", "assemble the pptx", or when storyline + exhibits + figures exist and slides don't. Refuses to run if storyline.md is not frozen or the memo test hasn't passed — route back to loki-storyline instead.
---

# loki-build

Assemble `deck.pptx` from frozen artifacts. Build is deliberately the *dumb*
stage: all thinking happened upstream, so build's job is fidelity — render
exactly what the storyline and specs say, flag anything that resists, invent
nothing.

## Preconditions (hard gate)

Refuse to build until ALL hold, and say which failed:

1. `storyline.md` is marked FROZEN and the memo test passed.
2. Every `visual: exhibit` row has a spec + CSV from loki-exhibit.
3. Every `visual: figure` row has an approved asset from loki-figure.
4. `STYLE.md` exists.
5. The `clarity` skill is installed — every slide's words pass through it.

Building on a soft storyline is how the old failure mode returns; the gate is
the point.

## Conductor plan

- **Wave per act** (Cold Open → Stranger → Stakes → High Noon → New Order).
  Acts are the natural parallelism boundary because transitions *within* an
  act are tightest.
- **Worker packet:** the FULL `storyline.md` (never just their slides — a
  worker who can't see the whole arc reproduces the flow bug), `STYLE.md`,
  their exhibit specs + CSVs, their figure assets + placement notes.
- **Title immutability:** workers render titles verbatim from the ladder.
  A worker who believes a title is wrong writes a `FLAG:` to the conductor
  with reasoning; the conductor routes it to loki-storyline, which issues a
  changelog delta. Workers never edit titles inline — silent title drift is
  how ladders rot mid-build.
- **Numbers only from CSVs.** If a worker needs a number that isn't in
  `data/`, that's a spec gap → FLAG, don't improvise. (Law 4.)

## Slide grammar

Consistent anatomy per slide type; deviations need a reason in the buildlog.

**Exhibit slide**
```
[title bar: action title verbatim, ≤1 line, 28pt+]
[optional kicker: ≤8 words restating the so-what]
[exhibit: ~65–75% of canvas, built to spec checklist]
[source line: bottom-left, 8–9pt]     [tracker: act marker, bottom-right]
```

**Divider** — full-bleed figure, act name, one-line promise of the act.
Theme intensity from STYLE.md governs how loud.

**High Noon options matrix (mandated)** — criteria as rows (pre-committed in
the storyline, listed BEFORE the reveal), options as columns including
"do nothing" priced, recommendation column highlighted in accent, footnote:
*"What would change our mind: ___"*. This slide is the deck's showdown;
its structure is not a style choice.

**Ask slide** — decision / owner / date, verbatim from the frame. Nothing
else competes for attention on it.

## Words on the slide — clarity pass

Titles, kickers, and annotations arrive already clarity-edited and frozen;
render them verbatim. Everything else a worker writes — divider promise
lines, body bullets, options-matrix cells, footnotes, speaker notes — goes
through **clarity** before it is rendered:

1. **Worker draft.** Write the slide's text from the storyline row and spec.
2. **Clarity rewrite, per slide, with the full storyline as context.**
   Medium: speech, talk, slides (notes are speech; slide text is read in a
   glance). Register from the act: Argument for Stakes and High Noon,
   Guide for The New Order's plan, Message for the ask. Constraints: the
   frozen title is the thesis the text must serve — reuse its nouns; every
   number stays a CSV transcription; FACT/EST/BELIEF framing is preserved.
3. **Conductor coherence read, per wave.** Concatenate the wave's slide
   text in order and run clarity in **review** mode against the previous
   waves' text: terminology drift (one concept, one name, deck-wide),
   register breaks, and repeated openers across slides. `revise`/`cut`
   verdicts on renders are fixed in-wave; `ask-author` verdicts are claim
   gaps → FLAG.

Clarity never edits a title, kicker, or annotation in build. If its review
says one reads badly, that is a FLAG to the conductor → loki-storyline or
loki-exhibit, the same as any other belief that a claim is wrong. And
clarity's output still has to fit: shorter is the usual result, but if a
rewrite overflows, split or FLAG — never shrink.

## Engine mechanics

Delegate the *how* to whatever pptx engine the environment provides:

- If the Anthropic `pptx` skill is present, read and follow it — its
  pptxgenjs gotchas (layout before slides, hex colors without `#`, native
  `addChart()` for anything PowerPoint can chart, validate.py after write)
  are hard-won and non-obvious. Keep charts NATIVE via `addChart()` so
  numbers stay editable and review can extract them; only chart types
  PowerPoint has no native form for go in as images.
- Otherwise use `pptxgenjs` or `python-pptx` directly with STYLE.md tokens.

Loki-build governs *what* is on each slide; the engine skill governs *how*
XML gets written. Don't duplicate engine documentation here.

Build rules regardless of engine:

- **Overflow is a build error, not a formatting choice.** Never shrink-to-fit
  below STYLE.md minimums; split content or FLAG the spec. Autofit hides
  problems that review then has to find visually.
- **Name every shape semantically** — `title`, `kicker`, `exhibit`, `source`,
  `tracker`, `fig-*`, `scrim-*`, `bg-*`. loki-uatu's deterministic geometry
  checks key off these names (intentional `fig`/`scrim`/`bg` overlaps are
  allowlisted); a deck of `TextBox 14`s degrades visual QA to guesswork.
- **Per-wave uatu fast loop.** When a wave's slides render, invoke
  loki-uatu's fast loop on just those slides — geometry lint plus clipping,
  collision, and squint checks. Render faults are fixed in-wave while the
  worker still has context; spec or claim faults FLAG to the conductor.
  Visual QA shifted left is minutes; the same defect found at deckreview is
  a reroute cycle.
- Accent color appears ONLY on message carriers (exhibit emphasis, the
  recommendation column, the ask). An accent used decoratively dilutes every
  slide where it matters.
- Figures: apply scrim per placement notes when title contrast demands it.
- Real brand logos from brand assets only — never generated, never stretched.

## Speaker notes = the voiceover

Each worker writes 2–4 sentences of speaker notes per slide (clarity-edited
as speech, above): what the presenter says, INCLUDING the spoken transition that begins with the ladder's
connective ("…but here's what the campaigns couldn't fix" / "therefore the
question becomes…"). Two jobs: the presenter's script, and review's flow
evidence — if the voiceover transition feels forced, the ladder junction is
weak, and that finding routes to loki-storyline.

## Output

- `deck.pptx`
- `buildlog.md`: wave map, per-slide status, all FLAGs and their resolutions,
  engine validate results.

Hand both to loki-deckreview. Build does not self-certify.
