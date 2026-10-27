---
name: loki-figure
description: Generate non-data deck imagery (hero images, act dividers, metaphor visuals, backgrounds) with ByteDance Seedream (BytePlus ModelArk) or Qwen-Image / Wan (Alibaba Cloud Model Studio, incl. the token plan), with a locked style block and reference-chaining for cross-slide consistency. Use whenever a deck needs generated images, whenever the user mentions Seedream, Qwen-Image, Wan, figures, hero images, dividers, or visual theme for slides. Runs in parallel with loki-exhibit after loki-storyline. NEVER use this skill (or any image model) to produce charts, graphs, or data visuals — those belong exclusively to loki-exhibit.
---

# loki-figure

Figures carry mood and metaphor; exhibits carry proof. This skill generates
the former with an image model (Seedream or Qwen-Image/Wan) and enforces the wall between the two.

## The two bans (non-negotiable)

1. **No charts from image models.** Modern image models (Seedream 5.0 Pro, Qwen-Image 3.0) render convincing
   infographics and "data" — which is precisely the hazard: plausible-looking
   numbers with no source. Every data visual goes through loki-exhibit with a
   CSV behind it. A generated image containing chart-like content is a SEV-1
   in review, even as background texture.
2. **No text in figures** (default). Words live in the PPTX text layer:
   editable, brand-font, spell-checkable, translatable. Their multilingual
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
   reference images; budget hero + 2–3 anchors. On Model Studio, chain
   with `wan2.7-image-pro` — see Step 3). Same STYLE_BLOCK + hero
   reference = one visual world across the deck. This batch runs unattended.
3. Log every generation in `figures/prompts.md`: figure id, model id, full
   prompt, seed/params, chosen output. Reproducibility is the grounded ethos
   applied to pixels — six weeks later, "make one more divider like the
   others" should be a lookup, not archaeology.

## Step 3 — Provider and model selection

Two providers, one script. The provider is picked from whichever key is set
(`LOKI_PROVIDER=ark|modelstudio` forces it).

| Provider | Key env var | Default host |
|---|---|---|
| BytePlus ModelArk (`ark`) | `ARK_API_KEY` | `https://ark.ap-southeast.bytepluses.com/api/v3` |
| Alibaba Cloud Model Studio (`modelstudio`) | `DASHSCOPE_API_KEY` | `https://token-plan.ap-southeast-1.maas.aliyuncs.com` (token plan) |

For Model Studio pay-as-you-go instead of the token plan, set
`LOKI_HOST=https://dashscope-intl.aliyuncs.com`. Override the host for either
provider with `LOKI_HOST`, the model with `LOKI_MODEL`.

### ModelArk — Seedream

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

### Model Studio — Qwen-Image / Wan

Verified 2026-10-27 on the international token plan (`ap-southeast-1`).

| Model | ID | Use for | With a reference image |
|---|---|---|---|
| Qwen-Image 3.0 Pro | `qwen-image-3.0-pro` | Hero candidates, standalone finals, edits | **Edits** the reference — keeps its subject and adds to it. Right for "fix this figure", wrong for chaining |
| Wan 2.7 Image Pro | `wan2.7-image-pro` | Every reference-chained figure after the hero | **Restyles** — new subject in the reference's light, palette and grade. The consistency engine |
| Wan 2.7 Image | `wan2.7-image` | Cheap drafts, candidate exploration | Same as Pro, lower fidelity |

The script defaults to Qwen with no refs and switches to Wan Pro whenever
refs are passed; pass `model="qwen-image-3.0-pro"` explicitly to edit.
Gotchas: sizes are `W*H` (asterisk); `1664*928` is the native ~16:9 frame.
The token plan's OpenAI-compatible `/images/generations` route returns 404 —
images go through the native multimodal endpoint. Output is a signed URL
that expires, so the script downloads it immediately.

## Step 4 — Generation script

```python
#!/usr/bin/env python3
"""loki-figure generator.
Env: ARK_API_KEY (ModelArk/Seedream) or DASHSCOPE_API_KEY (Model Studio);
optional LOKI_PROVIDER, LOKI_HOST, LOKI_MODEL. Reads ./.env if the key is not exported."""
import os, json, base64, pathlib, datetime, mimetypes, urllib.request

KEYS = {"ark": ("ARK_API_KEY",),
        "modelstudio": ("DASHSCOPE_API_KEY", "modelstudio_aliyun_token_plan")}
HOSTS = {"ark": "https://ark.ap-southeast.bytepluses.com/api/v3",
         "modelstudio": "https://token-plan.ap-southeast-1.maas.aliyuncs.com"}
MODELS = {"ark": "dola-seedream-5-0-pro-260628", "modelstudio": "qwen-image-3.0-pro"}

def _env() -> dict:
    env = {}
    if pathlib.Path(".env").exists():
        for line in pathlib.Path(".env").read_text().splitlines():
            k, _, v = line.partition("=")
            env[k.strip().removeprefix("export ")] = v.strip().strip("'\"")
    return env | dict(os.environ)

def _provider(env: dict) -> tuple[str, str]:
    want = env.get("LOKI_PROVIDER")
    for prov in ([want] if want else ["ark", "modelstudio"]):
        for name in KEYS[prov]:
            if env.get(name):
                return prov, env[name]
    raise SystemExit("no image key: set ARK_API_KEY or DASHSCOPE_API_KEY")

def _ref(r: str) -> str:            # local path → data URI; URLs pass through
    if r.startswith(("http://", "https://", "data:")):
        return r
    mime = mimetypes.guess_type(r)[0] or "image/png"
    return f"data:{mime};base64,{base64.b64encode(pathlib.Path(r).read_bytes()).decode()}"

def _post(url: str, key: str, body: dict) -> dict:
    req = urllib.request.Request(url, data=json.dumps(body).encode(),
        headers={"Authorization": f"Bearer {key}", "Content-Type": "application/json"})
    with urllib.request.urlopen(req, timeout=300) as r:
        return json.load(r)

def generate(fig_id: str, subject: str, style_block: str, negative: str,
             size: str | None = None, refs: list[str] | None = None,
             model: str | None = None) -> pathlib.Path:
    env = _env()
    prov, key = _provider(env)
    host = env.get("LOKI_HOST", HOSTS[prov])
    model = model or env.get("LOKI_MODEL") or (
        "wan2.7-image-pro" if prov == "modelstudio" and refs else MODELS[prov])
    prompt = f"{subject}, {style_block}. Avoid: {negative}"
    out = pathlib.Path(f"figures/{fig_id}.png")
    out.parent.mkdir(exist_ok=True)
    if prov == "ark":               # OpenAI-compatible images endpoint
        size = size or "2K"
        body = {"model": model, "prompt": prompt, "size": size,
                "response_format": "b64_json", "watermark": False}
        if refs:                    # reference-chain the approved hero, max 10
            body["image"] = [_ref(r) for r in refs]
        data = _post(f"{host}/images/generations", key, body)
        out.write_bytes(base64.b64decode(data["data"][0]["b64_json"]))
    else:                           # Model Studio native multimodal endpoint
        size = size or "1664*928"
        content = [{"image": _ref(r)} for r in refs or []] + [{"text": prompt}]
        body = {"model": model,
                "input": {"messages": [{"role": "user", "content": content}]},
                "parameters": {"size": size, "n": 1, "watermark": False,
                               "prompt_extend": False}}
        data = _post(f"{host}/api/v1/services/aigc/multimodal-generation/generation",
                     key, body)
        url = data["output"]["choices"][0]["message"]["content"][0]["image"]
        urllib.request.urlretrieve(url, out)        # signed URL expires — fetch now
    with open("figures/prompts.md", "a") as log:    # Law: log every generation
        log.write(f"\n## {fig_id} — {datetime.date.today()}\n"
                  f"provider: {prov} | model: {model} | size: {size} | "
                  f"refs: {refs or '—'}\n"
                  f"prompt: {prompt}\n")
    return out
```

Ask for 16:9-compatible sizing (2K on ModelArk; `1664*928` on Model
Studio). Full-bleed on a 13.33in slide wants ≥2000px width — 2K clears that;
upscale Model Studio finals with `magick in.png -resize 2560x out.png`.
Upscale drafts only if they're being promoted to finals.

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
