---
name: loki-storyline
description: Compile a dump.md into a frozen presentation storyline (beat sheet + action-title ladder, edited for readability with the clarity skill) before any slide is built. Use this whenever the user wants to start a deck, presentation, or slides from a dump, mentions "storyline", "narrative", "title ladder", "loki", or asks why a deck doesn't flow. This is ALWAYS the first loki step — if a user asks to "build the deck" and no frozen storyline.md exists, run this skill first, even if they didn't ask for it.
---

# loki-storyline

Turn `dump.md` into `storyline.md` + `STYLE.md`. The storyline is the compiler:
it converts unordered knowledge into an argument. Every downstream failure mode
— bland titles, missing chart emphasis, broken flow — is cheapest to fix here
and most expensive to fix in slides. Do not let anyone (including yourself)
build a single slide until the gate at Step 5 passes.

## Contract

- **Input:** `dump.md` (required). Refuse politely if absent — the muscle
  memory is brainstorm → dump → loki, never loki from a blank page.
- **Output:** `storyline.md` (frozen, versioned) and `STYLE.md`.
- **Gate:** the memo test AND the connective test (Step 5) both pass, run on
  the clarity-edited ladder (Step 4b).
- **Depends on:** the `clarity` skill. Loki writes the claims; clarity makes
  the words carrying them readable. If clarity is not installed, stop and say
  so — a ladder that skipped it is unfinished.

## Step 0 — Dump gate

A deck dump needs fields a code dump doesn't. Check for all six; if any are
missing, ask the user targeted questions (max 5) before proceeding:

1. **Audience** — who decides, and what do they believe *today*?
2. **The ask** — the specific decision/action wanted, with owner and date.
3. **Budget** — slide count ceiling and speaking time.
4. **Data inventory** — what numbers/sources actually exist. List them as
   `[D1]`, `[D2]`, … for later citation.
5. **Constraints** — brand template, compliance flags (e.g., OJK for banking),
   confidentiality level.
6. **Stakes** — what happens if the audience does nothing.
7. **References** — if reference files accompany the dump, loki-mobius has
   run: `refs/manifest.md` exists with confirmed roles. Where
   `refs/house-style.md` exists, calibrate title *voice and register* to
   its imitate-list — voice from the house, discipline from the lint.

Why: an action title without a known audience is a guess, and a storyline
without a data inventory will make claims the exhibits can't cash.

## Step 1 — Frame

Write three things at the top of `storyline.md`:

- **Belief shift:** "Audience believes X today → must believe Y by the end."
  One line. If you can't state X, you don't know the audience yet.
- **Controlling idea:** ONE falsifiable sentence, ≤25 words, that the whole
  deck argues. Pyramid Principle: this is the apex; every act supports it.
  If it isn't falsifiable, it's a slogan, and slogans don't move decisions.
- **The ask:** decision, owner, date. Verbatim what you want said "yes" to.

## Step 2 — Beat sheet (Western skin over SCQA)

Structure is non-negotiable; the skin is cosmetic. The five beats map to
Minto's SCQA, so consultants recognize the bones while the audience feels a
story:

| Beat | SCQA | Job | Share of deck |
|---|---|---|---|
| **Cold Open** | Situation | Open on the wound: one number or one scene that makes the stakes visceral. No agenda slide before it — earn attention first, orient second. | ~5–10% |
| **The Stranger Rides In** | Complication | What changed. The threat/opportunity named precisely, with evidence. | ~15–20% |
| **Raising the Stakes** | (implied Question) | Cost of inaction, quantified. Why the default/easy answers fail — steelman them, then show the data that breaks them. | ~25–30% |
| **High Noon** | Answer | The decision moment: real options side by side (including "do nothing", priced), decision criteria, recommendation. | ~25–30% |
| **The New Order** | Resolution | Life after "yes": plan, first 90 days, risks with mitigations, the ask restated. | ~15–20% |

Rules that keep it honest:

- Acts II–III must be MECE against the controlling idea — no orphan slides
  that support nothing.
- High Noon options must be *real*. A strawman option is a lie the audience
  will smell; steelmanning is both ethics and persuasion.
- State decision criteria BEFORE revealing the recommendation (pre-commitment
  — decision-analysis discipline). Add one footnote to the recommendation:
  *"What would change our mind: ___."* Grounded persuasion means showing the
  falsifier.

**Theme intensity** is separate from structure and lives in STYLE.md (Step 6).
The beats always run; whether the audience ever sees a literal frontier is a
dial.

## Step 3 — Title ladder

Write EVERY slide's action title, in order, one row per slide:

```
S## | Act | TITLE (action sentence) | message detail | evidence [D#/E##] | visual: exhibit/figure/none | FACT/EST/BELIEF | transition-out
```

### Action-title rules (the lint)

1. **Complete sentence with a verb.** "Q2 Performance" is a label; "One-off
   recoveries drove the entire Q2 beat" is a claim.
2. **Falsifiable.** Someone could disagree. If no one could, it's decoration.
3. **Carries the so-what,** not just the what: data + implication.
4. **Specific:** contains the number or the noun. "Significant pressure" fails;
   "NIM compressed 35bps" passes.
5. **≤ ~14 words** — must fit the title bar on one line.
6. **No slop lexicon.** Run the no-ai-slop list: unlock, leverage, journey,
   robust, holistic, empower, transformative, landscape, delve, synergy…
   Slogans love to masquerade as action titles; this rule is the costume check.
   The clarity pass (Step 4b) catches the subtler patterns the list misses.
7. **Tagged for epistemic status:** FACT (in the data inventory), EST
   (estimate — show the method), BELIEF (judgment — disclose it as such).
   The High Noon recommendation must rest on FACT+EST; BELIEFs may appear but
   never disguised.

### The connective test (this is the flow fix)

Between every pair of consecutive titles, you must be able to insert
**"therefore"** or **"but"** — never "and then". "And then" means the slides
are adjacent, not connected; that is precisely the from-one-slide-to-the-next
incoherence loki exists to kill. Write the connective into the
`transition-out` column so it's explicit, not vibes.

**Passing ladder (excerpt):**

```
S04 | II  | Deposit growth stalled at 2% while funding costs rose 60bps
     ...but
S05 | III | Therefore NIM compresses ~35bps by Q4 unless the mix shifts
     ...but
S06 | III | CASA campaigns alone recovered only a third of past gaps
     ...therefore
S07 | IV  | Repricing the top decile via RBPE protects ~20bps at low attrition risk
```

**Failing ladder (same content, no spine):**

```
S04 | Deposit Overview
S05 | NIM Trends            ← "and then"
S06 | CASA Campaign Results ← "and then"
S07 | RBPE Proposal         ← "and then"
```

The failing version is what per-slide generation produces from a dump. Every
title is defensible; the sequence argues nothing.

## Step 4 — Evidence audit

Walk the ladder. Every FACT title must cite `[D#]`. Any claim with no evidence
gets one of three fates: find the data (add to dump), demote to EST/BELIEF
with disclosure, or cut the slide. A persuasive deck that isn't grounded is a
loan against your credibility at a terrible rate.

## Step 4b — Clarity pass

Invoke the **clarity** skill in **rewrite** mode on the frame (belief shift,
controlling idea, ask) and the full title ladder, as ONE piece in slide
order — never title by title. Coherence is a property of the sequence: one
term per concept, one register, each title picking up where the last left
off. A per-title pass optimizes each line and loses the thread.

Brief clarity with:

- **Medium:** speech, talk, slides (its `references/medium.md` row) —
  listening comprehension, memorable evidence, oral rhythm.
- **Register:** Argument. The ladder owes a supported position.
- **Reader and outcome:** the audience and belief shift from Step 1.
- **Voice sample:** `refs/house-style.md` where it exists, else the
  author's own phrasing in `dump.md`. The sample outranks clarity's defaults.
- **Hard constraints (claims are not clarity's to change):** every number,
  noun, `[D#]`/`E##` citation, FACT/EST/BELIEF tag, and connective survives
  verbatim in meaning; ≤ ~14 words per title; complete sentence with a verb.

Clarity may tighten wording, unify terminology, swap a vague verb for a
precise one, and cut throat-clearing. It may not strengthen a claim, soften
an EST into a FACT, or invent a specific. When a better title needs a fact
only the author has, clarity marks `[TK: question]` — that becomes a dump-gate
question for the human, never a guess. One pass, one self-review, stop:
clarity's own rule, and repeated passes flatten titles into sameness.

Re-run the Step 3 lint on the output (clarity can lengthen a title while
making it clearer), then proceed to the memo test.

## Step 5 — The memo test (GATE)

Strip everything except the titles. Read them top to bottom, aloud if
possible. Three checks:

1. **Persuades alone?** The title sequence should work as a standalone memo a
   skeptic could follow — because for the exec who flips the printout, it *is*
   the deck.
2. **Connective test passes** at every junction (Step 3).
3. **Nothing load-bearing is missing:** if any title needs its slide to make
   sense, the title is underwritten — rewrite it.

If any check fails, fix the ladder and re-run. Do not proceed to exhibits or
build with a failing memo test; that is the entire point of this skill.

## Step 6 — STYLE.md

Emit the deck's visual contract, consumed by loki-figure and loki-build:

```markdown
# STYLE.md
palette: primary #__, ink #__, muted #__, accent #__ (accent = message carrier ONLY)
type: display __ / body __ / min sizes: title 28pt, body 14pt, source 9pt
chart tokens: grid #E5E5E5 0.5pt, direct labels > legends, accent-vs-grey emphasis
figure style block: "<subject slot>, <medium/style>, <palette anchors>,
  <lighting>, <composition incl. negative space for title overlay>, <quality tags>"
figure negative: "text, words, letters, charts, graphs, logos, watermarks"
theme intensity: 0–3
  0 = structure only (CEO/board decks: beats invisible, pure McKinsey surface)
  1 = tonal (golden-hour photography, frontier metaphors in speaker notes)
  2 = visible motif (act dividers carry the Western imagery)
  3 = full skin (townhall/internal: lean in)
```

Default to intensity 0–1 for external or C-suite audiences. The structure
persuades; the skin only ever decorates.

## Step 7 — Freeze

Mark `storyline.md` as `v1 FROZEN` with date. Downstream stages treat titles
as immutable. Changes go through a `## Changelog` delta in storyline.md (like
an ADR), which triggers a re-touch list: which exhibits, figures, and slides
must be revisited. Ad-hoc title edits during build are how decks rot.

## Anti-patterns

- Agenda slide before the Cold Open (orient after you've earned attention).
- Topic titles that survive because "the chart explains it" — the memo test
  exists to catch exactly this.
- A High Noon with one real option and two strawmen.
- BELIEF stated in FACT's clothing — the fastest way to lose a room that
  includes one person who knows the number.
- Skipping Step 0 because the dump "feels complete".
