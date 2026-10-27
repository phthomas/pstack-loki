---
name: loki-tva
description: Orchestrate the loki deck pipeline as two phases split by ONE human gate - COMPOSE drafts the full blueprint (storyline, STYLE, exhibit specs, hero candidates), the TRIAGE GATE closes everything in a single human session, then EXECUTE runs autonomously to SHIPPED or BLOCKED with zero interruptions. Use whenever the user says "run loki", "loki triage", "close the triage", "execute the deck", "loki status", "take it from here", or wants a whole deck from a dump. If the user wants the whole pipeline rather than one stage, use THIS skill, not the stage skills individually.
---

# loki-tva

Two phases, one human sitting between them:

```
dump ──► COMPOSE ─────────────► TRIAGE GATE ──► EXECUTE ──────────► terminal
         storyline + STYLE       one human       figure batch        SHIPPED
         exhibit specs + CSVs    session:        build waves           or
         hero candidates         close ALL       review + fix loop   BLOCKED
```

TVA is the Dormammu-equivalent for the loki universe: macro-conductor across
stages, while loki-build's conductor handles workers within its stage. Two
standing rules: **TVA never does stage work** (it invokes stage skills,
enforces the gate, prunes, logs — the moment it writes a title, it *is* the
bug), and **the filesystem is the only state** (any session, any machine,
cold-starts by scanning the repo).

## The claims/renders line

This one distinction is what makes autonomy safe:

- **Claims** live in the blueprint: titles, messages, spec'd numbers and
  their CSVs, the hero's look, the theme dial, the ask. Human-owned. A
  signed-off claim changes only through a human reopening triage.
- **Renders** are what execution produces: pixels, layouts, chart elements,
  transcriptions of CSV numbers onto slides. Machine-owned. Execution fixes
  render faults without asking.

Slide says 35, CSV says 32 → transcription fault → auto-fix to 32. Title
*claims* 35 but the CSV says 32 → claim vs. evidence conflict → **BLOCKED**.
Execution may correct how a claim is rendered; it may never revise the claim.

## Commands

| User says | TVA does |
|---|---|
| "loki triage" | Drive COMPOSE to completion, then open the triage session and iterate deltas until the human closes it. |
| "loki run" / "execute" | **Refuse unless triage is closed.** Then run to a terminal state with zero interruptions. |
| "loki status" | Scan artifacts; report phase, open triage items or blockers, next action. Never mutates. |
| "continue" | Manual mode: advance one stage, stop, report — for when you *want* to babysit. |

## State detection

First match wins:

| Condition on disk | State | Action |
|---|---|---|
| `clarity` skill not installed (`.claude/skills/clarity/SKILL.md` or `~/.claude/skills/clarity/SKILL.md`) | S0 | Refuse; loki depends on it. Point to `npx skills add addyosmani/clarity`. |
| No `dump.md` | S0 | Refuse; offer to interview for one. |
| Blueprint incomplete — no frozen storyline, or a `visual:` row without spec/CSV/hero candidates | S1 | **COMPOSE:** loki-mobius first when reference files are present (roles confirmed with the human), then loki-storyline (its dump gate may interrogate the human — the only other pre-gate interaction), then loki-exhibit ∥ loki-figure phase A as parallel subagents. |
| Blueprint complete, triage not closed | S2 | Open/resume the **TRIAGE GATE**. |
| Triage closed; deck absent or stale | S3 | **EXECUTE:** loki-figure phase B (batch) ∥ loki-build. |
| Deck built; review absent or older than deck | S4 | loki-deckreview under the disposition policy. |
| Findings outstanding | S5 | Fix loop — renders only, one full cycle. |
| SEV-1 = 0, policy satisfied | S6 | **SHIPPED.** Log and stand down. |
| Any finding routes to claims, or 2nd cycle still has SEV-1s | S6′ | **BLOCKED.** Report the exact decision needed. |

## The triage packet (what the human closes)

1. **The ladder read** — titles only, connectives inline, exactly as a
   skeptic sees it, after storyline's clarity pass. Any `[TK]` clarity
   raised is listed here as an open item: the human supplies the fact or
   the title stays plainer. This is the highest-leverage two minutes in the
   pipeline.
2. **Exhibit digest** — one line per exhibit (message → form → emphasis →
   clarity-edited kicker and annotation),
   full spec + CSV attached underneath. Claims get reviewed *with* their
   evidence, because approving a title while its data sits unread is how
   BLOCKED runs happen later.
3. **Look** — STYLE.md, theme intensity dial, hero candidates. Pick one hero;
   it seeds the reference chain for every figure in the batch. Where
   references were ingested, the mobius manifest and house-style digest
   appear here — confirm what was learned before anything is built on it.
4. **Disposition policy** — pre-committed review handling. Defaults:
   SEV-1 fix, SEV-2 fix, SEV-3 accept. Edit at triage, never mid-run.
5. **The ask, verbatim** — the sentence the deck exists to get a yes to.

Closure is explicit and total: TVA records `TRIAGE CLOSED` in storyline.md
and TIMELINE.md only when every item is dispositioned. A packet with an
unpicked hero or an unread ladder is *open*, and TVA lists what remains
rather than proceeding half-closed. Deltas loop inside the session — change
a title, TVA re-runs the connective test and re-prints the read — until
closed. There is no `--force`; a run built on an unclosed triage is the
failure mode this toolkit exists to prevent.

## Execution (uninterrupted by construction)

Figure batch reference-chains from the approved hero; build runs its waves;
review executes the disposition policy instead of asking; the fix loop
handles render faults and re-renders. The run cannot ask questions — it can
only finish or block loudly:

- **SHIPPED:** deck.pptx + clean review.md + TIMELINE entry. Deliverable.
- **BLOCKED:** the exact finding, its route, and the one human decision
  needed. Resume = reopen triage, close the delta, "run" again — the prune
  cascade regenerates everything downstream of what changed.

Blocking conditions: a finding routes to claims; a data gap surfaces that
compose missed; the second full fix cycle still yields SEV-1s (unstable
blueprint — TVA's job is to say so, not to lap a third time).

The design bet, stated plainly: human attention front-loaded at triage makes
claim-routed findings rare, so most runs terminate SHIPPED. When the bet
loses, TVA fails loudly rather than negotiating with a signed-off ladder.

## Staleness and pruning

```
dump ──(human judgment only)──► storyline + STYLE
storyline ─► specs ─► CSVs ─┐
STYLE + storyline ─► figures ├─► deck ─► review
```

Anything downstream of a change is a variant: mark stale → regenerate via
the owning skill → cascade. The re-touch list from storyline's changelog is
the precise source; mtime is the safety net. A dump edit never
auto-invalidates a closed triage — TVA flags the divergence and asks whether
to reopen; pruning human judgment is above its clearance.

## TIMELINE.md (append-only)

```
| when             | phase   | event                          | human |
| 2026-08-17 09:02 | compose | storyline v1 + 6 specs + hero×3| —     |
| 2026-08-17 09:31 | triage  | CLOSED (hero F01-b, policy dflt)| ✓    |
| 2026-08-17 10:44 | execute | shipped: SEV-1=0, 1 cycle      | —     |
```

One human row per deck is the goal. The log is the ADR habit applied to a
run: who closed which blueprint, when, and what the machine did with it.

## Machines

Triage from the desktop or the phone; execute overnight on a headless box.
Fire-and-forget is real precisely because the run cannot ask — it can only
finish or block, and both states are waiting in the repo at breakfast.
