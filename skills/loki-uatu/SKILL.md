---
name: loki-uatu
description: Deep visual QA of built decks. Render every slide, then inspect through three lenses in order - deterministic geometry checks computed from the pptx XML (overlaps, off-canvas, alignment rails, cross-slide title drift, font minimums), region-by-region pixel inspection of charts, figures, and element placement using crops, and deck-level gestalt (contact sheet, squint test, accent audit). Use whenever the user asks to check the visuals, inspect the slides, verify charts/figures/placement look right, or run visual QA on any .pptx - loki-built or not. Also invoked by loki-deckreview as its Pass 2. Uatu only FINDS and ROUTES; it never fixes slides in place.
---

# loki-uatu

The Watcher. Uatu sees everything and intervenes in nothing: findings route
to the owning stage (build / exhibit / figure / storyline), because a
reviewer who patches slides in place leaves the artifact that produced the
defect still wrong, and the next build regresses. Inspection and repair stay
separated on purpose.

Three lenses, strict order. Geometry runs first because it is cheap,
deterministic, and catches most placement bugs before any judgment is spent;
eyes are expensive and unreliable for a 0.1" misalignment, and math isn't.

## Lens 1 — Geometry (computed from XML, zero judgment)

Extract every shape's bounding box and check placement as arithmetic:

```python
#!/usr/bin/env python3
"""uatu_geo.py deck.pptx — deterministic placement checks."""
import sys
from itertools import combinations
from pptx import Presentation

EMU, TOL = 914400, 0.03                      # inches
SKIP = ("bg-", "fig-", "scrim-")             # intentional-overlap allowlist

p = Presentation(sys.argv[1])
W, H = p.slide_width / EMU, p.slide_height / EMU

def rects(slide):
    for sh in slide.shapes:
        if sh.left is None or not sh.width:
            continue
        yield sh.name, (sh.left/EMU, sh.top/EMU, sh.width/EMU, sh.height/EMU)

def overlap(a, b):
    ax, ay, aw, ah = a; bx, by, bw, bh = b
    ox = max(0, min(ax+aw, bx+bw) - max(ax, bx))
    oy = max(0, min(ay+ah, by+bh) - max(ay, by))
    return ox * oy

title_y = []
for i, s in enumerate(p.slides, 1):
    R = list(rects(s))
    for n, (x, y, w, h) in R:
        if x < -TOL or y < -TOL or x+w > W+TOL or y+h > H+TOL:
            print(f"S{i:02d} OFF-CANVAS  {n}  ({x:.2f},{y:.2f} {w:.2f}×{h:.2f})")
    for (n1, r1), (n2, r2) in combinations(R, 2):
        if any(n.startswith(SKIP) for n in (n1, n2)):
            continue
        if overlap(r1, r2) > 0.05:
            print(f"S{i:02d} OVERLAP     {n1} × {n2}")
    t = s.shapes.title
    if t is not None and t.top is not None:
        title_y.append((i, t.top / EMU))

ys = [y for _, y in title_y]
if ys and max(ys) - min(ys) > 0.05:
    print(f"TITLE-DRIFT y-range {max(ys)-min(ys):.2f}\" — "
          + ", ".join(f"S{i:02d}@{y:.2f}" for i, y in title_y))
```

Extend the same pattern for: **alignment rails** (left edges of body content
within TOL of each other), **footer furniture** (source line and tracker at
identical positions on every exhibit slide), **minimum gaps** (<0.3" between
sibling shapes), and a **font scan** (walk text frames; any run below
STYLE.md minimums, or autofit shrink detected, is SEV-1 — shrunk text is
overflow wearing makeup).

This lens depends on loki-build's semantic shape names (`title`, `exhibit`,
`source`, `fig-*`, `scrim-*`…). Unnamed shapes degrade geometry QA to
guesswork; flag a deck full of `TextBox 14` back to build as its own finding.

## Lens 2 — Region eyes (pixels, cropped so attention can't skate)

Render fresh, then inspect CROPS, not full slides — full-slide viewing is
how small defects survive, because you see the layout you expect:

```bash
soffice --headless --convert-to pdf deck.pptx
pdftoppm -png -r 150 deck.pdf slides/S        # 16:9 @150dpi → 2000×1125
S=slides/S-07.png
magick $S -crop 2000x190+0+0     crops/S07-title.png    # title band
magick $S -crop 2000x150+0+975   crops/S07-footer.png   # source + tracker
magick $S -crop 1500x720+250+210 crops/S07-canvas.png   # exhibit zone
magick crops/S07-canvas.png -resize 200% crops/S07-canvas@2x.png  # label detail
```

**Title band:** one line, no wrap, no ellipsis; matches the ladder verbatim
(deckreview Pass 1 diffs the text — here you confirm the *rendering*).

**Exhibit canvas — chart checklist (assert against `E##.spec.md`):**
- Every spec checklist element visibly present: kicker, units, time basis,
  annotation, reference line, direct labels, n=, axis start.
- **Label collisions:** data labels overlapping each other, their bars, or
  axis text — check at the 2× crop; collisions invisible at fit-to-width are
  exactly the "elements missing/off" class of defect.
- **Emphasis is real:** exactly one accent element; sample it —
  `magick $S -format "%[pixel:p{x,y}]" info:` — and match STYLE.md's accent
  hex; everything else muted grey. Multiple accents = no accent.
- Annotation sits ON its datum and its leader doesn't cross data ink.
- Legend present where the spec said direct labels → FAIL to build.
- Axis text not truncated ("Categor…") or rotated past ~45°; if labels can't
  fit horizontally, route to exhibit for shorter labels, don't rotate.
- Number formatting uniform (separators, decimals) and equal to the CSV at
  the spec's stated precision.

**Figures:**
- Artifact hunt at 100% crop (never only fit-to-width): warped geometry,
  pseudo-text textures, banding in gradients, cloned patches.
- **Overlay zone:** crop the region where the title sits — it must be the
  low-detail negative space that was prompted. Busy zone → route: figure
  (regenerate) or build (scrim).
- **Contrast, measured not eyeballed:**
  `magick $S -crop WxH+X+Y -colorspace gray -format "%[fx:mean]" info:`
  → relative luminance of the zone; compute the ratio against the title
  color; assert ≥ 4.5:1, else build adds scrim. Never solve contrast by
  shrinking the title.
- **Law 3 enforcement:** any chart-like content inside a generated figure —
  axes, bars, plausible "data" — is SEV-1, even as background texture.
- Resolution: image px width ≥ 150 × display inches (full-bleed 13.33" →
  ≥2000px). No watermarks, no generated logos.

**Footer band:** source line present, bottom-left, legible at 100%; tracker
bottom-right; page number where STYLE says.

## Lens 3 — Gestalt (the deck as one object)

```bash
magick montage slides/S-*.png -tile 4x -geometry +8+8 contact.png
for f in slides/S-*.png; do magick "$f" -resize 320x "thumbs/$(basename $f)"; done
```

- **Contact sheet:** does it read as ONE deck — consistent furniture,
  divider cadence marking the acts, visual weight distributed (not five
  dense slides then a desert)?
- **Squint test:** at 320px, the message carrier on each exhibit slide must
  still be findable in ~1 second. Emphasis that dies at thumbnail size was
  decoration, not emphasis — this is the neutrality test's pixel twin.
- **Accent audit:** accent color appears ONLY on message carriers, the High
  Noon recommendation, and the ask. Count decorative uses; each one dilutes
  every slide where the accent matters.
- **Figure family:** montage all figures side by side; grade, palette, and
  lighting must read as one world. The outlier regenerates with the hero
  reference — route to figure.

## Findings

Same schema and routing as review.md (uatu's rows merge straight in):

```
| ID  | Slide | Lens | SEV | Finding                                | Route   |
| U01 | S07   | 1    | 1   | OVERLAP exhibit × annotation-box       | build   |
| U02 | S07   | 2    | 2   | accent #7A9E9F ≠ STYLE #0B6E6E         | build   |
| U03 | S02   | 2    | 1   | chart-like texture in hero background  | figure  |
| U04 | all   | 3    | 2   | emphasis fails squint on S09, S12      | exhibit |
```

SEV-1: overflow/clipped/shrunk text, content-shape overlap, missing
mandatory spec element, chart-in-figure, contrast < 4.5:1 unfixed,
off-canvas. SEV-2: alignment/title drift, label collisions, accent
mismatch or dilution, squint failures, figure-family outliers. SEV-3:
gaps, grade, polish.

**The Watcher's oath:** report, route, stop. After fixes land, re-render the
PDF (pdftoppm reads nothing until it's regenerated), re-run Lens 1 in full
(it's free), and re-eye only the changed slides.

## Two modes

- **Fast loop** — called by loki-build after each wave renders: Lens 1 in
  full (it's free), plus Lens 2's clipping and collision checks and the
  squint thumbnail, on that wave's slides only. Render faults get fixed
  in-wave while context is hot; the same defect surfacing at full review
  costs an entire reroute cycle.
- **Full sweep** — called by loki-deckreview as Pass 2, or standalone: all
  three lenses, contact sheet included, findings merged into review.md.

Under an autonomous TVA run, uatu's build-routed findings are render faults
and auto-fix under the disposition policy; exhibit-, figure-spec-, or
storyline-routed findings are claims-side, so uatu reports them and the run
terminates BLOCKED rather than negotiating with the blueprint.

## Standalone use

Uatu needs no blueprint. Pointed at any .pptx — including decks other
people send — Lens 1 and Lens 3 run in full, and Lens 2 runs its general
checks with spec assertions skipped. Inside the loki pipeline, deckreview
invokes uatu as Pass 2 and merges its findings.
