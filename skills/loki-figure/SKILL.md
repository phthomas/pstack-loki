---
name: loki-figure
description: Generate non-data deck imagery (hero images, act dividers, metaphor visuals, backgrounds) with ByteDance Seedream 5.0 Pro / 5.0 Lite / 4.5 via the ModelArk API, with a locked style block and reference-chaining for cross-slide consistency. Use whenever a deck needs generated images, whenever the user mentions Seedream, figures, hero images, dividers, or visual theme for slides. Runs in parallel with loki-exhibit after loki-storyline. NEVER use this skill (or any image model) to produce charts, graphs, or data visuals — those belong exclusively to loki-exhibit.
---

# loki-figure

Figures carry mood and metaphor; exhibits carry proof. This skill generates
the former with Seedream and enforces the wall between the two.

## The two bans (non-negotiable)

1. **No charts from image models.** Seedream 5.0 Pro renders convincing
   infographics and "data" — which is precisely the hazard: plausible-looking
   numbers with no source. Every data visual goes through loki-exhibit with a
   CSV behind it. A generated image containing chart-like content is a SEV-1
   in review, even as background texture.
2. **No text in figures** (default). Words live in the PPTX text layer:
   editable, brand-font, spell-checkable, translatable. Seedream's multilingual
   text rendering is genuinely strong — and irrelevant here, because a typo
   baked into pixels at 11pm before the board meeting cannot be fixed in
   PowerPoint. Exception: display-art typography as the *subject* of a divider,
   with explicit user opt-in, proofread at full zoom.

## Contract

- **Input:** `storyline.md` rows with `visual: figure`, plus `STYLE.md`
  (style block + theme intensity dial).
- **Output:** `figures/F##-<slug>.png` + `figures/prompts.md` (the log).
- **Gate:** hero approved at the TVA triage gate BEFORE any batch generation
  — candidates are a COMPOSE deliverable; the batch is an EXECUTE task.

## Step 1 — Build the style block once

Compose from STYLE.md and reuse verbatim on every generation. Consistency
comes from the prompt being boring, not creative:

```
STYLE_BLOCK = "<medium: cinematic photograph | matte painting | …>,
<palette anchors: e.g. muted sepia and desaturated teal #0B6E6E accents>,
<lighting: golden hour, long shadows, volumetric dust>,
<composition: minimal, single subject, generous negative space in upper third
for title overlay>, 35mm anamorphic, film grain, high detail"

NEGATIVE = "text, words, letters, numbers, charts, graphs, diagrams,
logos, watermarks, UI elements"
```

Respect the **theme intensity** dial from STYLE.md: at 0–1 the Western skin
is lighting and mood only (golden hour, open space — reads as premium
photography, not cowboys); at 2–3 the frontier imagery is explicit. A Group
CEO deck at intensity 3 is a self-inflicted wound; a townhall at intensity 0
wastes the story you built.

## Step 2 — Consistency protocol

1. **Compose phase:** generate the **hero** (Cold Open image) first — 2–4
   candidates into the triage packet; the human picks at the gate. Where
   `refs/visual-anchors/` exists (loki-mobius), fold each anchor's
   descriptor line into STYLE_BLOCK and pass the anchor images as
   references for the hero candidates themselves — the user's doodle is
   the first ancestor of the whole visual family.
2. **Execute phase — reference-chain:** pass the approved hero as a reference
   image for every subsequent figure (Seedream 4.5/5.0 accept up to 10
   reference images; budget hero + 2–3 anchors). Same STYLE_BLOCK + hero
   reference = one visual world across the deck. This batch runs unattended.
3. Log every generation in `figures/prompts.md`: figure id, model id, full
   prompt, seed/params, chosen output. Reproducibility is the grounded ethos
   applied to pixels — six weeks later, "make one more divider like the
   others" should be a lookup, not archaeology.

## Step 3 — Model selection

| Model | ModelArk ID (intl / BytePlus) | Use for | Notes |
|---|---|---|---|
| Seedream 5.0 Pro | `dola-seedream-5-0-pro-260628` | Finals: hero, dividers, anything full-bleed | 1K/2K output; strongest layout/lighting control; ~2× Lite's price |
| Seedream 5.0 Lite | `seedream-5-0-lite` | Drafts, candidate exploration, volume | Faster/cheaper; supports up to 4K |
| Seedream 4.5 | `seedream-4-5-251128` | Fallback; region-precise edits on existing figures | Legacy but capable editor |

On Volcano Engine (CN) the Pro ID is `doubao-seedream-5-0-pro-260628` — same
model, regional prefix. Third-party hosts (fal.ai, Runware, etc.) expose the
same models under their own endpoint shapes if you'd rather not hold an Ark
key.

Working economics: explore wide on Lite, commit narrow on Pro. List pricing
is roughly $0.03/image Lite vs $0.045 (1K) / $0.09 (2K) Pro — at a dozen
figures per deck this is coffee money; optimize for approval speed, not spend.

## Step 4 — Generation script

BytePlus ModelArk, OpenAI-compatible images endpoint:

```python
#!/usr/bin/env python3
"""loki-figure generator. Env: ARK_API_KEY, optional ARK_BASE_URL, LOKI_MODEL."""
import os, sys, json, base64, pathlib, datetime, urllib.request

BASE  = os.environ.get("ARK_BASE_URL",
        "https://ark.ap-southeast.bytepluses.com/api/v3")
MODEL = os.environ.get("LOKI_MODEL", "dola-seedream-5-0-pro-260628")

def generate(fig_id: str, subject: str, style_block: str, negative: str,
             size: str = "2K", refs: list[str] | None = None) -> pathlib.Path:
    prompt = f"{subject}, {style_block}. Avoid: {negative}"
    body = {"model": MODEL, "prompt": prompt, "size": size,
            "response_format": "b64_json", "watermark": False}
    if refs:                      # reference-chain the approved hero
        body["image"] = refs      # list of URLs or base64 data URIs, max 10
    req = urllib.request.Request(
        f"{BASE}/images/generations",
        data=json.dumps(body).encode(),
        headers={"Authorization": f"Bearer {os.environ['ARK_API_KEY']}",
                 "Content-Type": "application/json"})
    with urllib.request.urlopen(req, timeout=180) as r:
        data = json.load(r)
    out = pathlib.Path(f"figures/{fig_id}.png")
    out.parent.mkdir(exist_ok=True)
    out.write_bytes(base64.b64decode(data["data"][0]["b64_json"]))
    with open("figures/prompts.md", "a") as log:   # Law: log every generation
        log.write(f"\n## {fig_id} — {datetime.date.today()}\n"
                  f"model: {MODEL} | size: {size} | refs: {refs or '—'}\n"
                  f"prompt: {prompt}\n")
    return out
```

Ask for 16:9-compatible sizing (2K, or explicit width×height where the
provider supports it). Full-bleed on a 13.33in slide wants ≥2000px width;
2K clears that. Upscale Lite drafts only if they're being promoted to finals.

## Step 5 — Placement notes for loki-build

For each figure, record in the spec row: bleed (full/half/inset), overlay
zone (where the title sits — must match the negative-space composition you
prompted), and a scrim rule (if title contrast < WCAG-ish 4.5:1 against the
image, build adds a gradient scrim; never solve contrast by shrinking the
title).

## Guardrails

- No real-person likenesses; no client/competitor logos generated into images
  (real logos come from brand assets in the build stage).
- Bank compliance: external-facing decks get a human review of imagery;
  figures depicting money, customers, or products follow brand guidelines,
  and OJK-sensitive material never gets illustrated in ways that imply
  guarantees.
- If a figure is tempting you to communicate a *quantity* — a growing pile,
  a rising line in the clouds — stop: that's an exhibit wearing a costume.
  Route to loki-exhibit.
