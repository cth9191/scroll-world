# scroll-world

> **Fork notice** — this is [cth9191's](https://github.com/cth9191/scroll-world) hardened fork of
> [oso95/scroll-world](https://github.com/oso95/scroll-world): budget tiers + spend gates
> (a 6-scene run is ~17 paid generations upstream — this fork asks first and offers 8-gen
> lean runs), a previz draft pass, automated SSIM seam verification, extracted-frame
> posters, an iOS Low Power Mode fallback, device-class clip tiering (iPads get the 1080p
> master), data-saver handling, true 9:16 portrait tiers, and a crawlable SEO copy block.
> Engine configs stay backward compatible. Full diff: [FORK-CHANGES.md](FORK-CHANGES.md).


https://github.com/user-attachments/assets/b08e641e-985b-4bd4-83ff-6750272d0c37


A Claude Code skill that builds an immersive, **scroll-scrubbed "fly through the world"
landing page** for any industry or brand — the kind where, as you scroll, a camera flies
from *outside* each scene *into* its interior, then flows on to the next scene with **no
cuts**. One continuous connected flight through a little generated world (think the Emons
logistics site, applied to whatever you want).

## Install

### As a plugin (recommended)

```
/plugin marketplace add cth9191/scroll-world
/plugin install scroll-world@scroll-world
```

Then just ask for a scroll-through world landing page, or invoke `/scroll-world`.
(For the original, unhardened version: `/plugin marketplace add oso95/scroll-world`.)

### Manually (drop-in skill)

Copy the skill folder into your Claude Code skills directory:

```bash
git clone https://github.com/cth9191/scroll-world
cp -R scroll-world/plugins/scroll-world/skills/scroll-world ~/.claude/skills/
```

## Requirements

- The [Higgsfield CLI](https://higgsfield.ai), authenticated (`higgsfield auth login`),
  with credits.
- `ffmpeg` / `ffprobe` for frame extraction and encoding.
- Python 3 with Pillow (optional — only for the transparent-scene knockout).

## What it does

It leans on [Higgsfield](https://higgsfield.ai) for the art: cohesive isometric diorama
scenes (GPT Image 2) and the camera flights themselves (Seedance image-to-video), scrubbed
by scroll position — the same technique behind Apple's scroll-through product pages. The
camera genuinely moves; scroll only drives time. It's **framework-agnostic**: you get the
Higgsfield pipeline, the prompt templates, and a portable vanilla-JS scrub engine that
drops into plain HTML, Next.js, Vue, or a Python-served page — nothing assumes a stack.

When invoked, the skill:

1. **Interviews you** — the subject/industry + pitch, a brand kit (import from a URL, hand
   it over, or have it proposed), art direction, **a budget tier** (lean ~8 generations /
   standard ~11–14 / showcase 17+ — scene count and camera architecture follow from it),
   the ordered scenes the camera visits, and a **mobile tier** (crop-safe → full 9:16
   portrait chain). It closes with a spend estimate you approve before anything generates.
2. **Generates the assets** with Higgsfield, gated so credits aren't wasted — ONE anchor
   still first (you approve the art direction, then the rest batch style-locked), a cheap
   **previz pass** of the whole chain on the draft model for bigger runs, then the final
   dive/leg clips and the **connector** clips that join consecutive scenes.
3. **Verifies and wires it up** — posters extracted from the encoded clips, an automated
   **SSIM seam check** on every join, then a config-driven scroll engine that plays the
   whole chain as one flight.

### The part that makes it good

The scenes connect **seamlessly** because each connector clip is generated with the
*actual rendered frames* of its neighbours as its start/end images (not the original
stills — those re-render slightly differently and would pop at the seam). Both sides of
every seam end up frame-identical, so the camera never cuts. This is baked into the skill
as the central rule — and machine-enforced: an SSIM gate scores every seam from the
encoded files before the page ever opens in a browser.

It also captures the non-obvious production gotchas: blob-URL loading so scrubbing works on
hosts that don't serve HTTP byte-range requests, GOP/encoding settings that stay sharp
without bloating, an iOS Low Power Mode stills fallback, device-class clip tiering
(phones get 720p tight-GOP encodes, iPads and desktops the 1080p master), data-saver
handling, a crawlable SEO copy block, and Higgsfield's quirks.

## What's in the skill

```
skills/scroll-world/
├── SKILL.md                    the procedure + budget/mobile tiers + the seam rule
└── references/
    ├── prompts.md              intake checklist + every Higgsfield prompt template
    ├── pipeline.md             idempotent batch scripts (anchor-gated stills → previz →
    │                           dives → connectors → encode → posters → SSIM seam gate)
    ├── scrub-engine.js         portable, config-driven scrub engine (blob-seek, lazy load,
    │                           seam crossfade, device-class tiering, LPM/data-saver fallbacks)
    ├── index-template.html     a minimal standalone page that mounts the engine (+ SEO block)
    ├── gotchas.md              full symptom → cause → fix list + frame-sequence upgrade path
    └── knockout.py             background knockout for floating scenes
```

## Notes

- Asset generation costs Higgsfield credits — the budget tier sets the bill: lean ≈ 8
  generations (4 scenes, N videos), standard ≈ 11–14, showcase ≈ 17+ (N stills + 2N-1
  videos), plus a re-roll buffer. Generation takes a while — the skill runs them in the
  background and polls, and every step is resumable (finished assets are never re-paid).
- The generated `.mp4`/`.webp` assets are produced per project; they're not shipped here.

## License

MIT — see [LICENSE](LICENSE).
