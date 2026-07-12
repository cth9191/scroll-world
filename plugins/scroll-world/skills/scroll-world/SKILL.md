---
name: scroll-world
description: >
  Build an immersive scroll-scrubbed "fly through the world" landing page for any
  industry or brand using Higgsfield. As the visitor scrolls, a pre-rendered camera
  flies from outside each scene into its interior, then flows on to the next scene
  with NO cuts — one continuous connected flight (Emons-style isometric diorama world,
  or any art direction you pick). The skill interviews the user for the topic, the
  story beats/sections, and brand kit, then generates cohesive scenes + seamless camera
  clips with Higgsfield and wires a portable, framework-agnostic scroll-scrub engine.
  Use when the user wants a "3D world" / "browse-through-the-industry" hero, a scroll
  cinematic, a diorama landing, or to turn a business into a scrollable world. It also
  wires opt-in conversion so the page isn't a dead end: GA4 flight analytics (scene /
  scroll-depth / CTA / lead events for validation gates), GHL lead capture on the finale
  (embedded form/calendar or an inline form to a GHL inbound webhook), and optional
  ElevenLabs per-scene scroll audio.
allowed-tools: Bash, Read, Write, Edit, AskUserQuestion, Skill
---

# scroll-world

Produces a landing page where **scroll drives a camera**: it dives from outside a scene
into its interior, then flies out and into the next scene, continuously, with no visible
cuts. The visuals are AI-generated (Higgsfield); the page just scrubs pre-rendered video
by scroll position. This is the same technique behind Apple's scroll-through product
pages — the camera genuinely moves, scroll only drives time.

**What you generate:** N scene stills (anchor-gated) → N "dive-in" camera clips → N-1
"connector" clips that join consecutive scenes seamlessly → encoded clips + extracted
posters → an automated SSIM seam check → a portable scrub engine that plays the whole
chain as one flight. Chain clips render on the cheap previz tier first by default;
full-model credits are spent only after the user approves the draft.

**The one rule that makes or breaks it:** seams must be *frame-identical*. Read
[The seamless chain](#step-5--the-seamless-chain-the-critical-part) before generating any
connector. Getting this wrong is the single most common failure and produces a visible
"pop" between scenes.

Do not assume a frontend framework. The scrub engine in `references/scrub-engine.js` is
self-contained vanilla JS (it builds its own DOM + injects its own CSS into a container
you give it), so it drops into plain HTML, Next.js, Vue, a Python-served page, anything.
The value of this skill is the Higgsfield pipeline, the prompts, and the seam method —
not the framework.

---

## Step 0 — Bootstrap

1. **Higgsfield CLI.** If `higgsfield` is not on `$PATH`, install per the
   `higgsfield-generate` skill. If `higgsfield workspace list` fails auth, ask the user
   to run `higgsfield auth login` (interactive OAuth — you cannot run it) and, if needed,
   `higgsfield workspace set <id>`. Confirm there are enough credits: a run is `N` image
   gens + `N` video gens (architecture A) or `(2N-1)` video gens (architecture B), plus
   re-rolls — the Step 1 budget tier sets `N`.
2. **ffmpeg / ffprobe** on `$PATH` (frame extraction + encoding).
3. **An image tool** for background knockout if you want floating scenes: PIL
   (`python3 -c "import PIL"`), or `cwebp`/`sips`. Optional — see Step 3.
4. Caveats: macOS ships **bash 3.2** (no `declare -A`); don't use associative arrays in
   scripts. Higgsfield generations take **3–8 min each** — always run them detached
   (background) and poll, never a foreground blocking call. Reference-by-job-UUID is
   rejected by media flags — pass **local file paths** to `--image/--start-image/--end-image`.
   Video models differ in accepted params (e.g. Kling has no `--resolution`) and in whether
   they support start/end-image conditioning at all — before batching, confirm the chosen
   model's schema with `higgsfield model get <job_type>` and see the Step 4 model table.

---

## Step 1 — Interview the user

The **subject is the user's to state — ask it as an open question in plain prose**, never a
fabricated multiple-choice. A made-up list of industries biases them and reads as you
deciding their business for them; let them answer in their own words (their real business,
a client's, or any idea). Reserve `AskUserQuestion` (with options) for the genuinely
enumerable, lower-stakes choices below — art direction and brand-kit approach — and even
there, signal they can go their own way ("Other"). Ask only what you can't sensibly
default. Cover:

1. **Subject** (ask openly, not multiple-choice) — "What should this world be about? Your
   business, a client's, or any idea — a word or a sentence is fine." Capture the
   industry/product + a one-line pitch (e.g. "a bubble tea company, from leaf to last
   sip"), and a brand name if they have one; otherwise you'll propose one below.
2. **Brand kit** — offer three paths, pick one:
   - Import from a URL: `higgsfield marketing-studio brand-kits fetch --url <site> --wait`
     (pulls name, colours, tone). Then read it back with `brand-kits list --json`.
   - The user hands you palette + name + tone directly.
   - You propose a palette + name and let them approve.
   Capture **4–6 named hex values**, a display name, and a tone word or two.
3. **Art direction** — default is "soft matte low-poly **clay diorama**, isometric,
   tilt-shift miniature, warm light." Offer alternatives (flat papercraft, glossy toy,
   claymation, neon night). Whatever is chosen becomes the shared **style preamble**
   reused verbatim in every scene prompt (this is what makes the world cohesive).
4. **Budget tier — ask BEFORE proposing the journey; video generations dominate cost.**
   The bill scales with scene count and architecture: architecture A (forward take) =
   `N` videos, architecture B (dives + connectors) = `2N-1` videos, plus `N` stills and
   a ~20–30% re-roll buffer. A 6-scene arch-B run is ~17 generations — a serious bite
   out of a subscription. Use `AskUserQuestion` with concrete numbers:
   - **Lean (~8 gens)** — 4 scenes, architecture A (4 stills + 4 legs), skip previz
     (small runs re-roll cheaper than a draft pass). Same site, same continuous flight —
     just 4 beats instead of 6.
   - **Standard (~11–13 gens)** — 5 scenes; arch A (10 gens) or arch B (5 + 9 = 14).
   - **Showcase (~17+ gens)** — 6–7 scenes, full arch-B world, previz pass, mobile tiers.
   **Fewer scenes ≠ thinner site.** 3–4 well-chosen beats read as a complete world: give
   each scene more scroll distance (`scroll: 1.6–2`) and `linger`, and let the copy
   carry more per beat. A tight 4-scene world beats a budget-starved 6-scene one.
   Two more levers when the user wants the B aesthetic on a lean budget: connectors are
   **individually optional** (a `null` slot crossfades that seam directly — honest
   tradeoff: that one transition is a dissolve, not a flight; spend connectors on the
   seams around the hero scenes) and `seedance_2_0_mini` can BE the final model if the
   user accepts 720p — it frame-locks, so the site stays seamless.
5. **The journey (sections)** — the ordered scenes the camera flies through, **sized to
   the budget tier**. Propose a set derived from the subject's own value chain and let
   the user edit. Boba example (6-scene showcase): farms → pearl kitchen → flagship shop
   → delivery → community plaza → the hero product; lean 4-scene cut: farms → kitchen →
   shop → hero product. Each section needs: a short subject description (what's IN the
   diorama), an eyebrow, a headline, one line of body, and 0–3 tag pills. The last
   section is usually the hero product + the CTA.
6. **Mobile tier — ALWAYS ask this; never silently generate extra assets.** Phones get
   the full scroll animation in every tier — the engine's hardening (seek-coalescing,
   iOS priming + Low Power Mode stills fallback, safe-area CSS, data-saver downgrade)
   is always on; the tiers only decide what ASSETS exist. Use `AskUserQuestion`,
   options in this order, with the credit cost stated:
   - **Crop-safe (default, no extra cost)** — phones scrub the desktop clips,
     centre-cropped to portrait. Every prompt already composes focal subjects
     centre-safe (prompts.md), so this reads fine for most worlds.
   - **Mobile encodes (recommended, no extra credits — just encode time)** — `-m.mp4`
     siblings (720p, `-g 4`, Step 6) + `clipMobile`/`connectorsMobile`/`posterMobile`
     wiring (Step 7) + mobile QA (Step 8). Phone-class devices (NOT tablets — the
     engine tiers by screen, iPad gets the master) scrub the lighter files.
   - **Hero reframe (small extra credit cost — state it)** — mobile encodes + 9:16
     re-renders of the 1–2 scenes where the focal subject can't hold a centre crop
     (usually hero + finale). Wire per-scene as the mobile clip/poster.
   - **Full portrait chain (gold tier, ≈2× video credits — state it)** — a parallel
     9:16 chain: portrait stills, own frame handoffs, own connectors, own SSIM gate
     (aspect ratios can't mix mid-chain — a 9:16 clip can't continue a 16:9 frame).
     This is what Apple ships: the mobile asset is a differently-framed render, not a
     crop. Offer only when the user signals mobile traffic matters or the budget is
     loose.
7. **Conversion wiring — ask, don't assume; a landing page that captures nothing is a
   waste of the render spend.** These are opt-in and cost no generations. Cover:
   - **Analytics** — a GA4 measurement id (`G-…`) if they want flight metrics. The engine
     emits scene/scroll-depth/CTA/lead events; without an id, the page is analytically
     blind. Default to asking for it.
   - **Lead capture** — where the finale CTA should go. Best options: a GHL **form or
     booking widget** to embed (get its iframe URL), or an **inline form → GHL inbound
     webhook** (get the webhook URL) with a source tag for their pipeline. Without this the
     finale is a dead `#` link.
   - **Audio (optional)** — do they want per-scene narration/ambient (ElevenLabs)? Off by
     default; it adds production cost and weight. Only if they ask.
   Full recipes (GA4 event schema + validation-gate mapping, GHL setup, ElevenLabs
   encoding) are in `references/integrations.md`; wire them in Step 9.

Video model is **not** an interview question — default `seedance_2_0` silently. If the user
names a preference, honor it **only if it can frame-lock seams** (Step 4 roster:
`seedance_2_0`, `kling3_0`, `seedance_2_0_mini`). This skill only ships seamless output, so
a model that can't frame-lock is declined with a one-line why, not substituted in — use a
roster model instead.

**Close the interview with a spend estimate — get explicit go-ahead before any
generation.** State the bill in plain numbers from the chosen budget tier: `N` image
gens + video gens (`N` for architecture A, `2N-1` for B; previz adds a mini-tier chain
on showcase runs) + a re-roll budget (~20–30% extra on interiors, thanks to the NSFW
filter), plus any mobile-tier extras, and roughly how long it runs (gens are 3–8 min
each; architecture A is sequential). The user approves the spend once, here — after
that the only further gates are the anchor-still approval (Step 2) and the previz
review (Step 4), both of which exist to keep the big spend from being wasted, not to
re-ask permission.

Keep the scroll mechanic fixed (continuous fly-through) — that's the point of the skill.
See `references/prompts.md` for the intake checklist and copy structure.

---

## Step 2 — Generate the scene stills

One image per section, **all sharing the same style preamble** for cohesion. Default
model **`gpt_image_2`** (crisp, great at isometric illustration; returns a solid/white
background which is perfect for floating diorama "islands"). Use `nano_banana_2` only if
the brief is character/cartoon-heavy (note: `nano_banana_2` is a CLI alias — it resolves
to `nano_banana_pro`; it won't appear under that name in `higgsfield model list`).

Prompt shape (full templates in `references/prompts.md`):

```
<STYLE PREAMBLE, identical every time>. On a plain solid <bg> background with a soft
contact shadow. <PALETTE hexes>. No text, no letters, no logos, centered, 3:2.
Subject: <what is in THIS diorama>.
```

- **Anchor first — never batch all N cold.** Generate ONE anchor still (the most
  representative scene), show it to the user, and iterate the style preamble until they
  approve. Only then batch the remaining N-1 **with the approved anchor passed as
  `--image`** to lock the style. A style miss on the anchor costs 1 gen; after a cold
  batch it costs N. This is a hard gate — do not proceed past an unapproved anchor.
- Batch the rest concurrently, detached. Command per scene:
  `higgsfield generate create gpt_image_2 --prompt "$(cat scene_i.txt)" --image anchor.png --aspect_ratio 3:2 --resolution 2k --quality high --wait --wait-timeout 15m --json > scene_i.json 2>scene_i.err`
- Result URL is `.[]0.result_url` in the `--wait --json` output. `curl` it down.
- A generation may fail transiently (HTTP 503) — re-roll that one individually; don't
  restart the batch.
- **Review the batch before continuing.** It must read as one cohesive world (same
  angle, palette, light). Re-roll any off-style scene individually — the anchor style
  lock stays in force.

See `references/pipeline.md` for the exact batch script (idempotent — re-runs skip
finished assets, so a crash or re-roll never repays for done work).

---

## Step 3 — (Optional) Float the scenes

If you want the dioramas to float over an atmospheric background instead of sitting in a
solid box, knock out the flat background to transparency with
`references/knockout.py` (border-connected flood fill — preserves interior colour that
matches the bg, e.g. cream walls). Then encode to webp. If you'd rather keep it simple,
just make the page background the same colour as the scene background and skip this.

Keep the stills either way — they're the **reduced-motion artwork and no-clip
fallback**. (Loading posters are NOT the stills: they're extracted from the encoded
clips in Step 6, so the still→video swap can't pop.)

---

## Step 4 — Camera architecture (pick one — this makes or breaks the feel)

How the camera moves *between* scenes is the single biggest quality lever. Two shapes;
pick by aesthetic.

### Video model — pick ONE for the whole chain

**This skill only ships seamless output**, so the only usable models are ones that can
frame-lock a seam: every chained clip must accept `--start-image`, and connectors also
need `--end-image`. That capability — not preference — is the selection rule. Check any
model with `higgsfield model get <job_type>` and **skip anything whose media inputs are
reference-only** (no start/end image): it can only *condition* a generation, not
*continue* a shot, so it physically can't hold a seam. Schemas below were confirmed
against the CLI:

| Model | start/end image | Notes |
|---|---|---|
| `seedance_2_0` (default) | ✓ / ✓ | Full chain (legs + connectors). `--mode std --resolution 1080p`. Its NSFW filter is the touchy one (see Gotchas). |
| `kling3_0` | ✓ / ✓ | Full chain — tested: `--mode std --sound off --duration 5` with start+end images accepted, seams frame-lock cleanly. **No `--resolution` param** (don't pass one; `--mode std` returns **720p native** — encode what ffprobe reports, never upscale). Sound defaults **on** → `--sound off`. `--duration` default 5, try 10 for legs. Different content filter than Seedance — the sanctioned NSFW fallback. |
| `seedance_2_0_mini` | ✓ / ✓ | Cheap draft tier that keeps frame-locking (720p). The previz tier: run the whole chain here first, then re-render final legs on the full model — still seamless, so it translates directly. |

Those three are the roster — all do both architectures. (`kling3_0_turbo` also frame-locks
via `--start-image`, but has no `--end-image`, so it's architecture-A-only and can't make
connectors; it also takes a different flag set — no `--mode`, has `--resolution` — so it
doesn't drop into the pipeline as-is. It's not in the default roster; only reach for it, and
wire it by hand, if architecture A's sequential render time is a proven bottleneck and you've
benchmarked it as actually faster.)

**Previz first (default, not optional-extra).** Unless the run is small (≤4 scenes),
render the whole chain on `seedance_2_0_mini` first. It frame-locks, so everything that
matters — journey order, camera grammar, seam continuity, copy pacing against the scrub —
is validated at draft cost; assemble the page from the previz clips and review it with
the user before a single full-model credit is spent. Then clear the draft clips, flip
`$VMODEL`, and re-render final (stills are reused; the pipeline's idempotency makes the
second pass mechanical — `references/pipeline.md`, setup block).

Rules:
- **One model for all chained clips.** Each renderer has its own motion/color/grain
  character; mixing models mid-chain keeps *position* continuity (frames still hand off)
  but the render-character shift reads as a subtle pop. The one sanctioned exception is
  the NSFW fallback for a single stubborn clip (Gotchas) — a slight character shift on
  one 5s connector beats a missing connector.
- Default to `seedance_2_0`; honor a user's stated preference **only if the model
  qualifies** (frame-locking). If it doesn't, say so and use a supported model — never
  ship a non-seamless build to satisfy a model request.
- The pipeline scripts take the model as `$VMODEL` with per-model flags already cased
  out (`references/pipeline.md`).

### A) Continuous forward take — RECOMMENDED for grounded / realistic / walkthrough
One camera that only ever glides **forward**, first scene through last, as a single take.
Generate the legs **sequentially**: leg 0 from scene-0's still (glide forward into it);
then each leg's `--start-image` = the **previous leg's ACTUAL last frame** (extract with
ffmpeg), prompt *"continue gliding smoothly FORWARD into [scene i], never pulling back"*
(or an expressive mid-leg move under the motion-handoff contract — see **Camera grammar**
below), and **no `--end-image`** — an end-image of a wide establishing shot forces the
camera to pull back, which is the #1 cause of stutter. Extract each leg's last frame to feed the
next. Result: every seam is frame-identical **and** the camera never reverses. There are
**no connectors** (skip Step 5) — the legs ARE the journey. Wire each leg as a section
clip with `connectors: []` and a small `crossfade` (~0.08). Even without an `--end-image`
the legs still arrive at distinct rooms (the prompt steers the content). Cost: strictly
**sequential** (can't parallelize) and slower; interiors trip the NSFW filter, so build in
re-rolls (3 attempts/leg).

### B) Dive-in + aerial connector — only for diorama / miniature / god's-eye worlds
A "dive into each scene" clip + a connector that pulls **up and out** and flies over to the
next scene (Step 5). The pull-out **reverses camera direction at every seam** (forward dive
→ backward pull-out). In a miniature/diorama world that reads as an intentional "zoom out
to the map, fly to the next island"; in a grounded first-person walkthrough it reads as a
jarring **rewind/stutter**. Use B only for the map-like aesthetic. When in doubt, use A.

### Camera grammar — the move should fit the concept (A is NOT "forward only")

"Forward only" is the *seam* rule, not the *leg* rule. The physics of the chain:

- **Position continuity** at a seam comes from the frame handoff (next leg starts from the
  previous leg's actual last frame).
- **Velocity continuity** at a seam means the camera must never *reverse across a seam* —
  that's the rewind stutter.
- **Inside a single leg the camera is free.** One leg is one continuous render — there is
  no seam to break mid-leg, so orbits, crane-ups, lateral tracking, even a push-in that
  eases back out are all safe *within* the clip. Reversals are only fatal *across* seams.

So give each leg an expressive move chosen from the scene's own logic, under a **motion
handoff contract**: every leg **ends by settling into a slow, steady forward drift** toward
the next destination (final ~1 s), and every leg **begins by continuing that same drift**.
Keep both clauses in the prompts verbatim (templates in `references/prompts.md`).

Pick the grammar from the concept:

| Concept / tone | Mid-leg move |
|---|---|
| Product / luxury retail | slow half-orbit around the hero object, then continue past it |
| Real estate / hospitality | steadicam glide through doorways; gentle crane-up in atria |
| Industrial / process / logistics | low lateral track alongside the line, foreground parallax |
| Travel / outdoors / campus | drone-style rise-and-reveal, then a descending swoop |
| Food / craft / detail-driven | push in close to the craft moment, ease back, carry on |
| Playful miniature (arch. B) | dives + aerial hops — the connector IS the grammar |

Honest costs: expressive mid-leg moves raise re-roll odds — the model can end a fancy move
in a state that isn't a clean forward drift. Mitigations: keep the final-second settle
clause verbatim; **eyeball each leg's last frame before chaining the next** (it should look
like a frame from a gentle forward glide — if not, re-roll before wasting the next leg);
budget ~1 extra re-roll per expressive leg. A plain forward glide stays the zero-risk
default — use it for legs where the scene itself is the show.

Two related pacing knobs live in the engine (Step 7): per-section `scroll` (more scroll
distance = longer dwell in that scene) and `linger` (the camera settles mid-scene exactly
while the copy peaks, then picks up speed toward the seam). Prefer expressive motion in the
*clip* and restraint in the *scrub mapping* — they compound.

And remember scroll is a scrubber: visitors can scroll **up**, so every move also plays in
reverse. That's free and expected — no extra work — but it's another reason seam velocity
must be consistent in both directions (a seam that reads fine forward reads as a stutter
backward too if velocity flips).

**For B**, one camera flight per scene: starts high/outside, descends into the interior,
structure opens. Model: the chain model you picked above (default **`seedance_2_0`**),
`--start-image = the scene still`.

- Use the **solid-background still** (not the knocked-out transparent one) as the
  start image, so the video has a full frame.
- Prompt: "Single continuous cinematic camera move, no cuts. Begin high and far looking
  at the whole <scene> from outside … descend and fly inside toward <focal point> … the
  roof/walls gently open to reveal the interior. <style>, smooth graceful slow motion.
  No text." (Template in `references/prompts.md`.)
- Params (seedance): `--mode std --resolution 1080p --aspect_ratio 16:9 --duration 8`.
  For Kling: drop `--resolution` (no such param), add `--sound off`, `--duration 10`.
  Do **not** pass `--generate-audio` (it errors on seedance; audio is wasted anyway —
  you'll mute).
- Run concurrently, detached, then download each `.result_url`. Re-roll individual
  failures. Keep the raw 1080p sources — you need their frames next.

---

## Step 5 — Connectors (architecture B only)

Skip this whole step for architecture **A** — the forward take has no connectors; its legs
already chain seamlessly. This step applies to **B** (diorama/miniature), and note the
reversal caveat from Step 4.

The connector clips are what make the world feel *connected* instead of cut. A connector
flies from the end of scene i out and into the start of scene i+1. **Both of its
endpoints must be the ACTUAL RENDERED FRAMES of the neighbouring clips — never the
original diorama still.**

Why: every Higgsfield generation renders slightly differently. If a connector *ends* on
a fresh render of "the kitchen diorama," but the next dive clip *starts* on its own
different render of that same diorama, the two won't match and you get a pop at the seam.
The fix is to hand off the exact pixels:

```
For each connector between dive_i and dive_{i+1}:
  start-image = the LAST frame extracted from dive_i's rendered video
  end-image   = the FIRST frame extracted from dive_{i+1}'s rendered video
```

Now every seam is frame-identical on *both* sides:
`dive_i.end == connector.start` and `connector.end == dive_{i+1}.start`.

Extract the boundary frames from the rendered dives (not the stills):

```bash
ffmpeg -sseof -0.15 -i dive_i.mp4   -frames:v 1 -q:v 2 dive_i_last.png    # interior of i
ffmpeg -ss 0      -i dive_{i+1}.mp4 -frames:v 1 -q:v 2 dive_next_first.png # establishing of i+1
```

Generate the connector (`--duration 5` is plenty). Connectors need `--end-image`, so the
model must accept it — any roster model does (`seedance_2_0`, `seedance_2_0_mini`,
`kling3_0`):

```bash
higgsfield generate create "$VMODEL" \
  --prompt "$(cat connector_i.txt)" \
  --start-image dive_i_last.png --end-image dive_next_first.png \
  $VOPTS --aspect_ratio 16:9 --duration 5 --wait --json
# seedance: VOPTS="--mode std --resolution 1080p"; kling3_0: VOPTS="--mode std --sound off"
```

Connector prompt: "Single continuous camera move, no cuts. Pull up and back out of
<scene i>, rise into the sky, glide across the connected miniature world, and arrive
above <scene i+1>, beginning to descend toward it. Seamless flowing aerial transition.
<style>. No text." (Template in `references/prompts.md`.)

Insurance: Seedance lands *close* to the end-image but not always pixel-perfect, so the
engine still applies a **short crossfade** (a few frames) at each seam. Frame-matched
endpoints + a small crossfade = no visible cut. Never skip the actual-frame handoff and
rely on the crossfade alone; a big content jump can't be hidden by a crossfade.

Whether a handoff actually held is **machine-checkable** — don't wait for the browser
QA to find out. After encoding, run the SSIM seam check (`references/pipeline.md` §5c):
a true actual-frame handoff scores ≥0.95 across the boundary; <0.75 means an endpoint
was a still or the wrong frame. Run it again after every re-roll — replacing one clip
silently touches BOTH of its seams.

---

## Step 6 — Encode for smooth scrubbing

Scrubbing = setting `video.currentTime` from scroll. Two things matter, and they are
often gotten wrong:

1. **Seekability, not keyframe density, is what makes scrubbing work.** Many static
   hosts (and `python -m http.server`) don't serve HTTP byte-range requests, which pins
   `video.seekable` to `[0,0]` and clamps *every* seek to frame 0 — the video looks
   frozen. The robust fix is to **fetch each clip as a `Blob` and play it from an
   in-memory object URL** (blobs are always fully seekable). The engine does this.
   Because of it, you do **not** need all-intra video.
2. **Don't shrink quality to get smooth seeks.** Encode at the **native resolution**
   (1080p from Seedance — don't downscale), `crf ~20`, a **small GOP** (`-g 8`) rather
   than all-intra (all-intra bloats an 8s clip to ~25 MB; GOP 8 is ~8 MB and scrubs
   fine via blob). Strip audio, add faststart, and a light `unsharp` counters video
   softness:

```bash
ffmpeg -i src.mp4 -an -vf "unsharp=5:5:0.8:5:5:0.0" \
  -c:v libx264 -preset slow -crf 20 -pix_fmt yuv420p \
  -g 8 -keyint_min 8 -sc_threshold 0 -movflags +faststart out.mp4
```

Encode all 2N-1 clips (dives + connectors) with the same settings for uniform quality.

**Extract posters from the ENCODED clips** (`references/pipeline.md` §5b). The still is
a 3:2 image; the clip is a 16:9 *re-render* of it — if the still is the loading poster,
the first video paint visibly jumps (crop + render drift) on the very first scene a
visitor sees. Same doctrine as the connectors, applied to seam zero: the poster must be
the clip's own extracted first frame. Wire it as `sections[k].poster` (Step 7); keep the
still as the reduced-motion artwork.

Then run the **automated seam check** (`references/pipeline.md` §5c) before touching a
browser — every seam SSIM ≥0.90 or you have a redo, not a QA note.

**Mobile encodes (only if the user picked a tier beyond crop-safe at Step 1.6).** Phone
video decoders seek far slower than a laptop's, and seek cost scales with GOP length, so
the 1080p `-g 8` master that scrubs smoothly on desktop can stutter on a phone. Produce a
lighter `-m.mp4` sibling for every clip — **720p, `-g 4`** (more keyframes = cheaper
seeks; keyframe-every-4-to-10 is the industry recipe for scrub-smooth mobile video), crf
23 — and wire them as `clipMobile` / `connectorsMobile` (Step 7). Extract each mobile
encode's first frame as its `posterMobile` (same doctrine as §5b — the poster must match
the encode the device actually gets). The engine serves mobile files on phone-class
devices only (screen short side ≤600 CSS px — tablets and iPads get the master) and
falls back to the desktop clip when absent. Scripts in `references/pipeline.md` §6; for
the hero-reframe / portrait-chain tiers see §7. If the user chose crop-safe, skip this —
the engine still hardens phone scrubbing regardless (seek-coalescing, iOS priming, Low
Power Mode stills fallback), so the page degrades gracefully rather than breaking.

---

## Step 7 — Assemble the page

Copy `references/scrub-engine.js` (and, if you want a fully standalone page, the tiny
`references/index-template.html`) into the user's project — or adapt into their
framework. It's config-driven and self-contained:

```js
mountScrollWorld(document.getElementById('world'), {
  brand: { name: 'Pearl & Co.' },
  diveScroll: 1.3, connScroll: 0.9,          // viewport-heights of scroll per clip
  analytics: { ga4: 'G-XXXXXXX' },           // opt-in flight events → validation gates (Step 9)
  capture: { mode:'embed', embedUrl:'…GHL form/calendar iframe…' },  // finale lead capture (Step 9)
  audio: { unmuteLabel:'Play sound', onLabel:'Mute' },  // opt-in; + per-section `audio:'…mp3'` (Step 9)
  sections: [
    { id:'farm', label:'The Farms', still:'assets/farm.webp',
      poster:'assets/farm-poster.webp',          // encoded clip's extracted first frame (Step 6)
      clip:'assets/vid/farm.mp4', clipMobile:'assets/vid/farm-m.mp4',   // mobile beta only
      audio:'assets/aud/farm.mp3',   // optional per-scene narration/ambient (fades with the scene)
      scroll: 1.6, linger: 0.45,   // optional pacing: longer dwell + camera settles mid-scene
      accent:'#8FB98A', eyebrow:'From leaf to last sip', title:'It starts in the hills.',
      body:'…', tags:['Single-origin','Hand-picked'] },
    // …one per section; last may carry a `cta`
  ],
  connectors:       ['assets/vid/conn1.mp4','assets/vid/conn2.mp4',   /* … length = sections-1 */],
  connectorsMobile: ['assets/vid/conn1-m.mp4','assets/vid/conn2-m.mp4' /* … same length; mobile beta only */],
});
```

The engine handles: the ordered dive/connector chain, scroll→currentTime with rAF
smoothing, blob loading, lazy prefetch of nearby clips, frame-matched crossfades, pinned
per-section copy (first section greets on landing, last holds its CTA), a route rail,
`prefers-reduced-motion`, and mobile. **Pacing per section:** `scroll` overrides
`diveScroll` for that scene (more scroll = longer dwell) and `linger` (0–1, keep ≤ 0.6)
remaps time so the camera settles mid-scene — exactly while the copy peaks — then speeds
up toward the seam; seam frames are untouched (f(0)=0, f(1)=1). Give the hero and finale
scenes a higher `scroll` + some `linger`; keep transit scenes brisk. Theme it with CSS variables (`--accent`,
`--sw-bg`, `--sw-ink`, …) — the visual identity comes from the generated clips, so the
chrome stays quiet. See the header of `scrub-engine.js` for the full config + CSS vars.

**On phones the engine adapts automatically, along two separate axes.** *Clip tier*
(which file) is decided by device class — screen short side ≤600 CSS px = phone →
`clipMobile`/`posterMobile`; tablets (iPad Pro included — coarse pointer but
desktop-class screen and decoder) and desktops get the master. *Behaviour hardening*
(how it acts) keys off coarse pointer / ≤860px viewport: **coalesced seeks** (never
queues a new `currentTime` mid-seek — what stops a fast flick freezing the clip),
poster held until the clip paints, **video priming on first touch** (iOS
blank-until-played fix), a longer scroll run per scene (`scrollMobileFactor`, default
1.2 — small viewports read the same flight as faster), dropped particles, URL-bar-safe
resizes, safe-area insets. **Automatic stills fallback:** `prefers-reduced-motion`,
Chromium data-saver, and iOS **Low Power Mode** (detected at runtime — a rejected muted
`play()` on first touch) all flip the page to stills-with-crossfades instead of frozen
video. On 2g/3g (Chromium signal) the clip prefetch window shrinks. All of this is on
by default — no config. The `clipMobile`/`connectorsMobile`/`posterMobile` encodes are
the opt-in tiers from Step 1.6: wire them only when the user picked one.

**SEO copy is not optional — this is a landing page.** The engine renders all copy
client-side, so on its own the page has zero crawlable text. Always put a plain-markup
mirror of the copy (h1 = hero line, one h2 + p per scene, real CTA links) inside the
container in a `data-sw-seo` block — the engine hides it on mount, crawlers/link
previews/no-JS visitors read it from the served HTML. `references/index-template.html`
ships the block; when adapting into a framework, server-render it.

For non-JS backends (Python/Rails/etc.): serve the assets and drop the engine `<script>`
into the rendered HTML; nothing about it is framework-specific.

---

## Step 8 — QA the seams (don't skip)

**First, the machine check:** the SSIM seam gate (`references/pipeline.md` §5c) must
already be green — every seam ≥0.90 (0.75–0.90 only where you've eyeballed the
crossfade and accepted it). If you skipped it, run it now; browser QA below verifies
the *page*, the SSIM gate verifies the *assets*, and a red asset can't be QA'd into a
green page.

Then drive the page in a headless browser and **verify frame continuity at the seams**
end-to-end:

- Screenshot at scroll positions just before and just after each seam. The two frames
  must be near-identical (the dive's last frame == the connector's first frame). If they
  pop, you used the diorama still instead of the actual rendered frame (redo Step 5), or
  the crossfade band is too short.
- Confirm the first paint is clean: the poster (extracted frame) shows instantly, and
  the poster→video takeover does not shift the image (if it does, `poster` is missing
  or points at the still — Step 6).
- Check the console for errors, confirm `video.seekable.end(0) > 0` (blob working), and
  that `currentTime` tracks scroll across each clip's band.
- **Stills-mode fallbacks (every build, cheap to check):**
  - Data-saver: emulate `navigator.connection.saveData = true` (DevTools override or an
    init script) — page must render as stills-with-crossfades, zero clip fetches in the
    Network panel.
  - Low Power Mode: hardest to emulate — on a real iPhone, enable it and confirm the
    page falls back to stills on first touch instead of frozen video. Emulated proxy:
    stub `HTMLMediaElement.play` to return a rejected promise, tap, confirm stills mode.
  - Tablet tier: iPad viewport (834×1194, touch) must fetch the **desktop** clip
    (Network panel), not the `-m.mp4` — while still getting touch behaviour.
- **Mobile — full checklist only if the user picked a mobile tier (Step 1.6).**
  For a crop-safe build, just sanity-check a phone viewport once: page loads, still
  posters show, nothing overlaps — the engine's hardening covers graceful degradation.
  For the mobile tiers (do this on a real phone or an emulated one, portrait + landscape):
  - Emulate a phone viewport **with CPU throttled 4–6×** and scroll fast — the clip should
    track without freezing (the seek-coalescing + `-m.mp4` encodes are what make this hold).
  - Confirm the first scene shows immediately (its still is the poster) and the video takes
    over the instant you scroll — no blank/black scene (the iOS priming fix). Test iOS Safari
    specifically; it's the one that goes blank if this regresses.
  - Verify the `-m.mp4` variant is actually served on mobile (Network panel), and the
    heavy 1080p master on desktop.
  - Slowly scroll so the URL bar collapses — the page must **not jump** (height-only resizes
    are ignored on touch). Rotate the device — layout should recompose cleanly.
  - Portrait crops a 16:9 clip to its centre; confirm the focal subject still reads. If a
    hero scene's subject sits off-centre and gets cut, recompose it (prompts.md) or generate
    a 9:16 variant for that scene.
- Check reduced-motion (should fall back to the stills, no video, no particles).
- **Conversion (if wired — see Step 9):** with `analytics: { …, debug:true }`, scroll top→
  bottom and confirm the console logs `sw_scene_view`, `sw_scroll_depth` 25→100, and
  `sw_flight_complete`; click the CTA (`sw_cta_click`). Then submit the capture form once
  against the **real** GHL endpoint and confirm both the thank-you message **and** a tagged
  contact landing in the right pipeline — never ship an unverified webhook. Audio (if used):
  tap unmute, confirm the active scene's narration plays and fades as you scroll on.

---

## Step 9 — Conversion & deploy (don't ship a dead end)

The render spend buys a beautiful page; this step makes it *earn*. All three are opt-in and
cost zero generations. Full recipes — GA4 event schema + validation-gate mapping, GHL form/
webhook setup with source tagging, ElevenLabs narration encoding — are in
`references/integrations.md`.

1. **Analytics.** Pass `analytics:{ ga4:'G-…' }` (engine injects gtag) or `{ ga4:true }` (you
   already placed a GA4/GTM snippet). The engine emits `sw_scene_view`, `sw_scroll_depth`,
   `sw_flight_complete`, `sw_cta_click`, `sw_lead_submit`. Mark `sw_lead_submit` a **Key
   Event** in GA4. Read the numbers as gates: completion = `flight_complete` rate,
   engagement/intent = `cta_click` + `lead_submit` rate.
2. **Lead capture** (the finale). `capture:{ mode:'embed', embedUrl:'…' }` drops in a GHL
   form or booking widget (GHL owns submission + routing — simplest, recommended), or
   `capture:{ mode:'inline', webhookUrl:'…', source:'…' }` renders an on-brand form that
   POSTs name/email/phone + UTMs to a GHL **inbound webhook**, honeypot-guarded, with a
   source tag for the pipeline. **Never put a GHL API key in the page** — the webhook URL is
   a public inbound endpoint and is the only credential that belongs client-side.
3. **Audio (optional).** `audio:{ unmuteLabel:'Play sound' }` + per-section `audio:'…mp3'`.
   Muted until the visitor taps unmute (autoplay policy); each scene's volume then follows
   its opacity, so narration cross-fades with the flight. Off automatically under
   reduced-motion / data-saver. Keep files small (mono ~96 kbps).
4. **Deploy.** The page is static HTML + `scrub-engine.js` + `assets/`: `vercel --prod` from
   the page dir, or drop the engine into a Lovable/Next/Vue route and server-render the
   `data-sw-seo` block. Set the GA4 id and GHL URL **before** deploy (they aren't secrets),
   then fire one live test lead and confirm it lands as a tagged contact in the right
   pipeline before calling it done.

---

## Gotchas — top 5 inline; full list in `references/gotchas.md`

Read `references/gotchas.md` the moment any generation fails or any QA check reads
wrong — it maps symptom → cause → fix for every failure seen in production (encode,
theming, phone, iOS, Kling flags, portrait crops, and more). The five that block runs
most often:

- **Seam pop** → connector endpoints were the diorama stills, not the neighbouring
  clips' actual frames. Always extract real frames (Step 5); the SSIM gate (pipeline
  §5c) catches this before a browser ever opens.
- **Seam stutter / camera "jumps backward"** → camera *velocity reverses* across a seam
  (inherent to architecture B's pull-outs). Grounded walkthroughs must use
  architecture A (Step 4).
- **NSFW false-positives (Seedance `status "nsfw"`)** → flags innocuous interiors
  (bedroom/pool/spa; words like "bed", "wine", "swim"). In order: re-roll (often passes
  2nd–3rd try) → strip trigger words + add "empty, unoccupied, no people, architectural"
  → regenerate that one clip on `kling3_0` with the same start/end frames → set the
  connector slot to `null` (engine crossfades that seam directly). Budget re-rolls on
  interiors.
- **Frozen video / stuck at frame 0** → host doesn't serve byte ranges, `seekable=[0,0]`.
  Blob URLs fix it (engine does this).
- **Concurrent gens 503 / "not_enough_credits" race** → transient under parallel launch;
  re-roll the individual failure (verify credits with `higgsfield workspace list`).

## References

- `references/prompts.md` — the intake checklist, style-preamble pattern, and every
  prompt template (scene still, dive, connector) with fill-in slots.
- `references/pipeline.md` — copy-paste batch scripts for the whole run (anchor-gated
  stills → previz → dives → frames → connectors → encode → posters → SSIM seam gate →
  mobile encode), idempotent + bash-3.2-safe.
- `references/scrub-engine.js` — the portable, config-driven scrub engine (builds DOM +
  injects CSS; blob-seek, lazy load, seam crossfade, extracted-frame posters, hidden
  `data-sw-seo` static-copy block, copy, route rail, reduced-motion, and phone hardening:
  mobile encodes, seek-coalescing, iOS priming, safe-area, no-jump resize).
- `references/index-template.html` — a minimal standalone page that mounts the engine,
  including the crawlable `data-sw-seo` copy block and the opt-in conversion config.
- `references/knockout.py` — border-connected background knockout for floating scenes.
- `references/gotchas.md` — the full symptom → cause → fix list, plus the canvas
  frame-sequence alternative for when video scrubbing isn't smooth enough.
- `references/integrations.md` — Step 9 conversion recipes: GA4 flight-event schema +
  validation-gate mapping, GHL lead capture (embedded widget or inline → inbound webhook
  with source tagging), ElevenLabs scroll-audio setup, and Vercel / framework deploy.
