# pstack-loki

> McKinsey rigor for the argument, Western-series storytelling for the arc.
> A skill toolkit that compiles rough notes into persuasive, grounded
> PowerPoint decks — one human gate, then an autonomous run.

Loki is a set of eight `SKILL.md` files for [Claude Code](https://docs.claude.com/en/docs/claude-code)
(portable to any harness that reads skills). You bring a dump of notes,
data, and reference files; loki brings a storyline compiler, chart specs
with element checklists, Seedream-generated figures, a conductor/worker
build, and a six-pass review that inspects the rendered pixels.

## Why this exists

Agent-built decks fail in three predictable ways:

1. **Bland titles** — an action title requires knowing the whole arc;
   per-slide agents default to descriptive labels ("Q2 Performance").
2. **Broken exhibits** — a chart with no thesis has nothing to prove, so
   annotations, emphasis, and reference lines go missing.
3. **No flow** — slides built straight from unordered notes are locally
   fine and globally incoherent.

One root cause: the pipeline went `dump → slides`. A dump is unordered
knowledge; a deck is an argument. Code has compilers and tests as free
feedback loops — decks have none, so loki manufactures them: a storyline
gate before any slide exists, exhibit specs before any chart renders, and
pixel-level review after the build.

## How it works

```
reference files ──► [loki-mobius] ──► house-style · [D#] fragments · visual anchors
(exemplars · materials ·      │
 visual anchors)              ▼
dump.md ◄─────────────────────┘
   │
   ▼                                COMPOSE
[loki-storyline] ──► storyline.md (frozen) + STYLE.md
   ├──► [loki-exhibit] ──► exhibits/E##.spec.md + data/E##.csv
   └──► [loki-figure]  ──► hero candidates
   │
   ▼                                ONE HUMAN GATE
[triage] ladder read · spec digest · hero pick · policy · the ask
   │
   ▼                                EXECUTE (autonomous)
[loki-figure] batch ──► [loki-build] waves ──► deck.pptx
                              │  (per-wave loki-uatu fast loop)
                              ▼
                      [loki-deckreview] six passes + uatu full sweep
                              │
                              ▼
                      SHIPPED  or  BLOCKED (one decision needed)
```

`loki-tva` orchestrates the whole thing: it derives pipeline state from
the files on disk, drives compose, holds the triage session, then runs to
a terminal state with zero interruptions.

### The Five Laws

1. **The storyline is the compiler.** Nothing builds until the title
   ladder reads as a persuasive memo on its own.
2. **Titles freeze at triage.** Workers render titles verbatim; execution
   fixes *renders* (pixels, layouts, transcriptions) but never *claims*
   (titles, messages, spec'd numbers).
3. **Charts prove, figures evoke.** Data visuals come only from specs with
   real CSVs. Image models never draw charts — they are good enough at
   fake infographics to be dangerous.
4. **Every number has a home.** Each figure on a slide traces to a CSV
   cell or a dump reference; unsourced numbers block the ship.
5. **Fixes route upstream.** A flow problem is a storyline bug, not a
   slide patch.

## The skills

| Skill | Job |
|---|---|
| `loki-mobius` | Ingest reference files by role — exemplars, materials, visual anchors — before compose |
| `loki-storyline` | Compile dump → beat sheet + action-title ladder; memo test and connective test as hard gates |
| `loki-exhibit` | One spec per chart: message, form, data CSV, 10-point element checklist, neutrality test |
| `loki-figure` | Seedream 5.0 Pro / 5.0 Lite / 4.5 imagery with a locked style block and reference-chained consistency |
| `loki-build` | Conductor/worker build in waves; title immutability; semantic shape names; overflow = error |
| `loki-uatu` | Visual QA: deterministic geometry lint + region-crop vision rubric + deck gestalt |
| `loki-deckreview` | Six passes: memo/flow, visual (uatu), grounding, slop, story beats, cold read of the whole deck |
| `loki-tva` | Orchestrator: compose → one triage gate → autonomous execute → SHIPPED/BLOCKED |

## Installation

**1. Prerequisites**

- [Claude Code](https://docs.claude.com/en/docs/claude-code) (or another
  harness that reads `SKILL.md` skills)
- Python 3.10+ with `python-pptx` (uatu's geometry lint, title extraction)
- LibreOffice + poppler (`pdftoppm`) — slide rendering for review
- ImageMagick — crops, thumbnails, contact sheets
- A pptx engine: the Anthropic `pptx` skill if you have it, else
  `pptxgenjs` / `python-pptx`

```bash
# Debian/Ubuntu
sudo apt install libreoffice poppler-utils imagemagick

# Arch/CachyOS
sudo pacman -S libreoffice-fresh poppler imagemagick

# macOS
brew install --cask libreoffice && brew install poppler imagemagick

# Python dep (uv or pip)
uv pip install python-pptx        # or: pip install python-pptx
```

**2. Install the skills**

```bash
git clone https://github.com/<you>/pstack-loki
cd pstack-loki

# globally, for all projects
cp -r skills/* ~/.claude/skills/

# or per-project
cp -r skills/* <your-deck-project>/.claude/skills/
```

**3. Seedream key** (only needed for generated figures)

Get an API key from [BytePlus ModelArk](https://www.byteplus.com/en/product/modelark),
then:

```bash
export ARK_API_KEY=...
# optional overrides
export ARK_BASE_URL=https://ark.ap-southeast.bytepluses.com/api/v3
export LOKI_MODEL=dola-seedream-5-0-pro-260628   # 5.0 Pro (default)
# alternatives: seedream-5-0-lite (drafts, 4K) · seedream-4-5-251128 (edits)
```

An all-exhibit deck needs no key at all.

**4. Verify**

```bash
ls ~/.claude/skills | grep -c '^loki-'   # → 8
soffice --version && pdftoppm -v && magick -version
python -c "import pptx"
```

## Quick start

```bash
mkdir q3-alco-deck && cd q3-alco-deck
$EDITOR dump.md          # notes, numbers, half-thoughts — rough is fine
cp ~/old-decks/*.pptx .  # optional: reference files
claude
```

Then, inside the session:

```
> loki triage        # compose + one review sitting
  … answer role/dump-gate questions, read the ladder, pick a hero …
> closed
> loki run           # walk away
  … SHIPPED — deck.pptx + review.md + TIMELINE.md
```

Two commands and one conversation.

## Usage guide

### 1. Start a deck project

Write `dump.md` — audience, the ask, slide budget, the data you have,
constraints, stakes. Missing pieces are fine; triage will ask. Drop any
reference files in the folder; you never invoke mobius directly, the
orchestrator does when it sees them.

### 2. `loki triage`

TVA runs everything up to the gate and speaks to you at most twice before
the packet: to confirm reference **roles** (exemplar / material /
visual-anchor per file) and to fill dump-gate gaps. Then it composes
silently — storyline, specs with CSVs attached, hero candidates on Lite —
and lands the **triage packet**:

1. the title ladder, connectives inline, read as a skeptic would;
2. one-line exhibit digests with full spec + data underneath;
3. STYLE, theme-intensity dial (0 = pure consulting surface, 3 = full
   Western skin), hero candidates;
4. the disposition policy for review findings (defaults: SEV-1 fix,
   SEV-2 fix, SEV-3 accept);
5. the ask, verbatim.

Object in plain language ("S07 overclaims — CSV says 32"); TVA re-lints
and re-prints. Say **closed** when the memo reads true. Two minutes of
real reading here is the best-spent time in the whole pipeline.

### 3. `loki run`

Fully autonomous: figure batch reference-chained from your hero, build
waves with a per-wave visual fast loop, six-pass review under your
pre-committed policy, one bounded fix cycle. It cannot ask questions —
it can only finish or fail loudly:

- **SHIPPED** — `deck.pptx`, clean `review.md`, a `TIMELINE.md` entry.
- **BLOCKED** — the exact finding and the one decision needed. Answer it
  (that's a triage delta), say `run` again; only what your answer touched
  gets rebuilt.

Long runs suit a headless box: close triage at your desk, execute on a
server, find the terminal state in the repo at breakfast.

### 4. Manual mode

Every stage answers directly when you want to drive:

| Say | Runs |
|---|---|
| `continue` | Next stage only, then stop and report |
| `run loki-storyline` | Just the storyline compiler |
| `spec the exhibits` | Just loki-exhibit |
| `build the deck` | Just loki-build (refuses without a frozen storyline) |
| `review the deck` | Full six-pass deckreview |
| `check the visuals` | Standalone uatu sweep |

First deck? Take one lap on `continue` to watch each stage's output before
going fire-and-forget.

### 5. Reference files

| Role | You mean | Loki does |
|---|---|---|
| **exemplar** | "learn how we present" | Distills title voice, rhythm, anatomy, real theme palette into `refs/house-style.md` — imitates the register, never the diseases (topic titles stay banned) |
| **material** | "part of this story, not yet narrative" | Decomposes into claims and `[D#]` numbers with provenance and as-of dates; real photos placed as-is, never regenerated |
| **visual-anchor** | "carry this look" | Chart conventions → STYLE tokens; doodles/figures → Seedream references seeding the hero itself |

One file can hold two roles; declare both.

### 6. Standalone visual QA

Uatu needs no blueprint. Point it at any deck — including ones other
people send you:

```
> check the visuals on vendor-proposal.pptx
```

Geometry lint, squint test, and contact-sheet gestalt run in full; spec
assertions engage only when specs exist.

### 7. What lands where

```
your-deck-project/
├── dump.md                 # input (mobius appends fragments)
├── refs/                   # manifest, house-style, anchors, assets
├── storyline.md  STYLE.md  # the frozen blueprint
├── exhibits/     data/     # chart specs + the only numbers allowed on slides
├── figures/                # generated imagery + prompts.md log
├── deck.pptx               # the deliverable
├── review.md   TIMELINE.md # findings + append-only run log
└── slides/                 # rendered PNGs, thumbnails, contact sheet
```

## Design decisions

- **One gate, not many.** All judgment front-loads into a single triage
  sitting; execution never negotiates with a signed-off blueprint. When
  the bet loses, the run blocks loudly instead of drifting.
- **Claims vs. renders.** Slide says 35, CSV says 32 → transcription
  fault, auto-fixed. Title *claims* 35, CSV says 32 → BLOCKED for you.
- **The connective test.** Between consecutive titles you must be able to
  insert *therefore* or *but* — never *and then*. This single lint kills
  most flow problems before a slide exists.
- **Structure and skin are decoupled.** The five beats (Cold Open → The
  Stranger Rides In → Raising the Stakes → High Noon → The New Order) are
  SCQA underneath and always run; how visibly Western the deck looks is a
  0–3 dial per audience.
- **Review reads like the audience.** The final pass is a cold read of
  the rendered deck, front to back, speaker notes off — because decks
  circulate as pre-reads and must argue with their author absent.

## Status

Young toolkit, opinions fully loaded. The skill logic is exercised; the
Seedream script is a template — dry-run it once in your environment before
trusting a batch. Build mechanics delegate to whatever pptx engine your
harness provides. Issues and PRs welcome; keep skills lean and slop-free.

## Lineage

Barbara Minto (Pyramid Principle, SCQA) · Gene Zelazny (*Say It With
Charts*) · Trey Parker & Matt Stone (the but/therefore rule) · Peter
Yang's [no-ai-slop](https://github.com/petergyang/no-ai-slop) ·
ByteDance Seedream.

Composes with the core **pstack** toolkit (dump discipline, no-ai-slop,
ADR habits) but runs standalone — no pstack required.

## Naming

| Skill | Lore |
|---|---|
| loki | God of stories. Decks are honest illusions: the trick is staged, but every prop is real. |
| tva | Keeps the Sacred Timeline (the frozen storyline) and prunes variants (drift). |
| mobius | The analyst who studies the case files before anyone touches the timeline. |
| uatu | The Watcher. Sees everything; fixes nothing; routes precisely. |

## License

MIT — see [LICENSE](LICENSE).
