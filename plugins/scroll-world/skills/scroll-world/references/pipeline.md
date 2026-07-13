# Pipeline: copy-paste scripts (bash 3.2 safe)

Set these once. `NAMES` is the ordered section ids; the last is the hero/finale.

```bash
WORK=/tmp/scroll-world           # scratch dir for prompts, sources, frames
ASSETS=./assets                  # where the site reads stills (webp) + clips (mp4)
mkdir -p "$WORK" "$ASSETS/vid"
NAMES="farm kitchen shop delivery plaza finale"   # <-- your section ids, in order

# Chain video model — ONE for every chained clip (SKILL Step 4 roster).
# Must accept --start-image AND --end-image (verify: higgsfield model get <model>):
# seedance_2_0 | kling3_0 | seedance_2_0_mini (draft tier). Reference-only models can't
# hold a seam; models without --mode (e.g. kling3_0_turbo) need their own flag branch below.
VMODEL=seedance_2_0
case "$VMODEL" in                                  # per-model flags + durations (bash 3.2 safe)
  kling3_0)          VOPTS="--mode std --sound off";          DIVE_DUR=10; CONN_DUR=5 ;;  # no --resolution param on Kling
  seedance_2_0_mini) VOPTS="--mode std --resolution 720p";    DIVE_DUR=8;  CONN_DUR=5 ;;  # cheap frame-locked previz
  *)                 VOPTS="--mode std --resolution 1080p";   DIVE_DUR=8;  CONN_DUR=5 ;;  # seedance_2_0 default
esac
```

Higgsfield generations take minutes — every `higgsfield ... --wait` call below is meant
to run inside a **backgrounded** script. Launch the whole script with your tool's
background/detached mode and poll the progress log; never block the foreground.

**Resume / idempotency.** Every `gen_*` function below skips work whose output file
already exists and is non-empty — `$WORK` *is* the run state. A crash, credit stall, or
NSFW re-roll never costs finished assets: just re-run the same loop and only the missing
pieces regenerate. To force a re-roll of one asset, delete its file first
(`rm "$WORK/dive_shop.mp4"; gen_dive shop`). Check where a run stands any time:

```bash
status() { for n in $NAMES; do
  printf '%-10s still:%s dive:%s\n' "$n" \
    "$([ -s "$WORK/still_$n.png" ] && echo ok || echo -- )" \
    "$([ -s "$WORK/dive_$n.mp4" ] && echo ok || echo -- )"
done; ls "$WORK"/conn_*.mp4 2>/dev/null | while read f; do printf 'conn: %s ok\n' "$f"; done; }
```

**Previz first (recommended default).** Run the whole chain once on the draft tier
before spending full-model credits:

```bash
VMODEL=seedance_2_0_mini   # frame-locking intact (~720p) — seams behave like the final's
# … run §2–§5, review the assembled page, fix journey/prompts/seams cheaply …
VMODEL=seedance_2_0        # then clear the draft clips and re-render final
rm -f "$WORK"/dive_*.mp4 "$WORK"/conn_*.mp4 "$WORK"/first_*.png "$WORK"/last_*.png
```

Because the mini tier still frame-locks, everything you validate (journey, camera
grammar, seam continuity, copy pacing) translates directly to the final render. Stills
are reused as-is — only the video passes re-run. Skip previz only for small (≤4-scene)
runs where a full-model re-roll is cheaper than the extra pass.

## 1. Scene stills (Step 2) — anchor first, then batch

Write one prompt file per section to `$WORK/still_<name>.txt` (see prompts.md).

**Do NOT batch all N immediately.** Generate ONE anchor still first (the most
representative scene), get the user's approval on the art direction, then batch the
rest with the approved anchor passed as `--image` to lock the style. A style miss
caught on the anchor costs 1 gen; caught after the batch it costs N.

```bash
STYLE_LOCK=""   # set to the approved anchor after the gate below
gen_still() { # name
  [ -s "$WORK/still_$1.png" ] && { echo "still $1 cached"; return 0; }
  higgsfield generate create gpt_image_2 --prompt "$(cat "$WORK/still_$1.txt")" \
    ${STYLE_LOCK:+--image "$STYLE_LOCK"} \
    --aspect_ratio 3:2 --resolution 2k --quality high --wait --wait-timeout 15m --json \
    > "$WORK/still_$1.json" 2> "$WORK/still_$1.err"
  url=$(jq -r '.[0].result_url // empty' "$WORK/still_$1.json")
  [ -n "$url" ] && curl -fsSL "$url" -o "$WORK/still_$1.png" && echo "still $1 ok" || echo "still $1 FAIL"
}

# 1. Anchor + approval gate (pick the scene that best expresses the world):
gen_still farm                      # ← your anchor section
# → SHOW the user still_farm.png; iterate the style preamble until approved.
#   A rejected anchor: fix the preamble in ALL prompt files, rm the png, re-roll.

# 2. Then batch the rest, style-locked to the approved anchor:
STYLE_LOCK="$WORK/still_farm.png"
for n in $NAMES; do gen_still "$n" & done ; wait   # anchor skips itself (cached)
```

Convert to webp for the site (and optionally run knockout.py first for transparency):

```bash
for n in $NAMES; do cwebp -quiet -q 84 -resize 1800 0 "$WORK/still_$n.png" -o "$ASSETS/$n.webp"; done
```

Review the batch for cohesion before continuing. Re-roll any off-style one
(`rm "$WORK/still_shop.png"; gen_still shop` — the style lock is still in force).

## 2. Dive-in clips (Step 4)

Prompt files at `$WORK/dive_<name>.txt`. Start image = the solid-bg still PNG.

```bash
gen_dive() { # name                       ($VOPTS is unquoted on purpose — word-split flags)
  [ -s "$WORK/dive_$1.mp4" ] && { echo "dive $1 cached"; return 0; }
  higgsfield generate create "$VMODEL" --prompt "$(cat "$WORK/dive_$1.txt")" \
    --start-image "$WORK/still_$1.png" \
    $VOPTS --aspect_ratio 16:9 --duration "$DIVE_DUR" \
    --wait --wait-timeout 20m --json > "$WORK/dive_$1.json" 2> "$WORK/dive_$1.err"
  url=$(jq -r '.[0].result_url // empty' "$WORK/dive_$1.json")
  [ -n "$url" ] && curl -fsSL "$url" -o "$WORK/dive_$1.mp4" && echo "dive $1 ok" || echo "dive $1 FAIL"
}
for n in $NAMES; do gen_dive "$n" & done ; wait
```

Re-roll individual failures (503 / credit race are transient):
`gen_dive shop`  (just that one).

## 3. Extract boundary frames — the seam handoff (Step 5)

For each adjacent pair, the connector's start = dive_i's LAST frame, end = dive_{i+1}'s
FIRST frame — extracted from the **rendered videos**, never the stills.

```bash
set -- $NAMES
prev=""
for n in "$@"; do
  ffmpeg -v error -ss 0 -i "$WORK/dive_$n.mp4" -frames:v 1 -q:v 2 "$WORK/first_$n.png"      # establishing
  ffmpeg -v error -sseof -0.15 -i "$WORK/dive_$n.mp4" -frames:v 1 -q:v 2 "$WORK/last_$n.png" # interior
done
```

**Architecture A: the chain handoff must be the TRUE last frame — not the -0.15s frame
above.** The -0.15s offset is fine for a connector's *start-image* (the model re-renders
anyway), but when the next leg literally continues from the frame, starting ~4 frames
early plays back as a rewind at the seam (measured: exact frame → seam SSIM ≈0.90+;
-0.15s → 0.59). Extract the exact final frame with the `-update` trick:

```bash
ffmpeg -v error -y -sseof -0.3 -i "$WORK/leg_$prev.mp4" -update 1 -q:v 2 "$WORK/last_$prev.png"
```

## 4. Connector clips (Step 5)

Prompt files at `$WORK/conn_<i>.txt` (i = 1..N-1). Iterate adjacent pairs:

```bash
gen_conn() { # i startPng endPng          (end-image required → seedance/kling3_0 only)
  [ -s "$WORK/conn_$1.mp4" ] && { echo "conn $1 cached"; return 0; }
  higgsfield generate create "$VMODEL" --prompt "$(cat "$WORK/conn_$1.txt")" \
    --start-image "$2" --end-image "$3" \
    $VOPTS --aspect_ratio 16:9 --duration "$CONN_DUR" \
    --wait --wait-timeout 20m --json > "$WORK/conn_$1.json" 2> "$WORK/conn_$1.err"
  url=$(jq -r '.[0].result_url // empty' "$WORK/conn_$1.json")
  [ -n "$url" ] && curl -fsSL "$url" -o "$WORK/conn_$1.mp4" && echo "conn $1 ok" || echo "conn $1 FAIL"
}
set -- $NAMES ; i=0 ; prev=""
for n in "$@"; do
  if [ -n "$prev" ]; then i=$((i+1)); gen_conn "$i" "$WORK/last_$prev.png" "$WORK/first_$n.png" & fi
  prev="$n"
done ; wait
```

## 5. Encode everything for scrubbing (Step 6)

Native resolution (1080p from seedance std; kling3_0 std returned **720p** in testing —
never upscale, encode what ffprobe reports), crf 20, GOP 8, light sharpen, no audio,
faststart. Same for dives + connectors.

```bash
enc() { ffmpeg -v error -y -i "$1" -an -vf "unsharp=5:5:0.8:5:5:0.0" \
  -c:v libx264 -preset slow -crf 20 -pix_fmt yuv420p \
  -g 8 -keyint_min 8 -sc_threshold 0 -movflags +faststart "$2"; echo "enc $2 $(du -h "$2"|cut -f1)"; }

for n in $NAMES; do enc "$WORK/dive_$n.mp4" "$ASSETS/vid/$n.mp4"; done
i=0; for f in "$WORK"/conn_*.mp4; do i=$((i+1)); enc "$f" "$ASSETS/vid/conn$i.mp4"; done
```

Now the engine config's `sections[k].clip = assets/vid/<name>.mp4` and
`connectors = [assets/vid/conn1.mp4, …]` (length N-1, in order).

## 5b. Posters — extract from the ENCODED clips (kills the still→video pop)

The generated still is 3:2 and the clip is a 16:9 re-render of it, so if the still is
the loading poster, the moment the video paints there's a visible crop/render jump —
on the very first scene a visitor sees. Same doctrine as the connectors: hand off
actual frames. The poster must be the encoded clip's own first frame:

```bash
for n in $NAMES; do
  ffmpeg -v error -y -ss 0 -i "$ASSETS/vid/$n.mp4" -frames:v 1 -q:v 2 "$WORK/poster_$n.png"
  cwebp -quiet -q 84 "$WORK/poster_$n.png" -o "$ASSETS/$n-poster.webp"
done
```

Wire as `sections[k].poster = 'assets/<name>-poster.webp'`. Keep `still` too — it stays
the reduced-motion artwork and the no-clip fallback (the engine prefers `poster` while a
clip will load, `still` otherwise).

## 5c. Verify the seams — automated, before any eyeballing

Seamlessness is the product; don't ship it on a squint. Every seam in the chain must be
near-identical across its boundary frames. SSIM-check them all from the encoded files
(chain order: dive0, conn1, dive1, conn2, … — for architecture A it's just leg0, leg1, …):

```bash
# last frame of A vs first frame of B, SSIM score on stdout
# (-update 1 = exact final frame; approximate seeks under-score high-motion seams)
seam_ssim() { # fileA fileB
  ffmpeg -v error -y -sseof -0.3 -i "$1" -update 1 "$WORK/_sa.png"
  ffmpeg -v error -y -ss 0      -i "$2" -frames:v 1 "$WORK/_sb.png"
  ffmpeg -v info -i "$WORK/_sa.png" -i "$WORK/_sb.png" -lavfi ssim -f null - 2>&1 \
    | grep -o 'All:[0-9.]*' | cut -d: -f2
}

check() { # fileA fileB label
  s=$(seam_ssim "$1" "$2")
  case $(awk -v s="${s:-0}" 'BEGIN{ if (s>=0.90) print "pass"; else if (s>=0.75) print "warn"; else print "fail" }') in
    pass) echo "PASS  $3  ssim=$s" ;;
    warn) echo "WARN  $3  ssim=$s (crossfade will mostly hide it — eyeball this seam)" ;;
    *)    echo "FAIL  $3  ssim=$s — endpoints are NOT the neighbours' frames; redo this connector (SKILL Step 5)" ;;
  esac
}

# Architecture B (dive/conn interleave):
set -- $NAMES ; i=0 ; prev=""
for n in "$@"; do
  if [ -n "$prev" ]; then i=$((i+1))
    check "$ASSETS/vid/$prev.mp4" "$ASSETS/vid/conn$i.mp4" "$prev>conn$i"
    check "$ASSETS/vid/conn$i.mp4" "$ASSETS/vid/$n.mp4"    "conn$i>$n"
  fi ; prev="$n"
done
# Architecture A (sequential legs): check "$ASSETS/vid/legI.mp4" "$ASSETS/vid/legI+1.mp4" pairs.
```

Thresholds from the frame-handoff physics: a true actual-frame handoff scores ≥0.95
even after encoding; ≥0.90 pass, 0.75–0.90 warn (Seedance's end-image landed close but
not exact — the engine crossfade usually covers it), <0.75 means a still was used as an
endpoint or the wrong frame was extracted — regenerate, don't rationalize. Run this
after every re-roll too: replacing one clip can silently break BOTH of its seams.

## 6. Mobile encodes (Step 6) — only if the user picked a mobile tier

**Skip this section if the user chose the crop-safe tier in the Step 1
interview.** Scrubbing sets `currentTime` every frame, and a phone decoder's **seek cost scales with
how many frames it must decode from the nearest keyframe** — so a 1080p `-g 8` master
that scrubs fine on a laptop stutters on a phone. A **smaller frame + tighter GOP** fixes
that (and halves the bytes on cellular). Produce a `-m.mp4` sibling for every clip:

```bash
# 720p, GOP 4 (twice the keyframes = ~half the seek-decode work), crf 23, same sharpen/faststart.
encm() { ffmpeg -v error -y -i "$1" -an -vf "scale=-2:720,unsharp=5:5:0.6:5:5:0.0" \
  -c:v libx264 -preset slow -crf 23 -pix_fmt yuv420p \
  -g 4 -keyint_min 4 -sc_threshold 0 -movflags +faststart "$2"; echo "encm $2 $(du -h "$2"|cut -f1)"; }

for n in $NAMES; do encm "$WORK/dive_$n.mp4" "$ASSETS/vid/$n-m.mp4"; done
i=0; for f in "$WORK"/conn_*.mp4; do i=$((i+1)); encm "$f" "$ASSETS/vid/conn$i-m.mp4"; done
```

Extract each mobile encode's first frame as its poster (§5b doctrine — the poster must
match the encode the device actually gets):

```bash
for n in $NAMES; do
  ffmpeg -v error -y -ss 0 -i "$ASSETS/vid/$n-m.mp4" -frames:v 1 -q:v 2 "$WORK/poster_${n}_m.png"
  cwebp -quiet -q 84 "$WORK/poster_${n}_m.png" -o "$ASSETS/$n-poster-m.webp"
done
```

Wire the variants in the engine config — the engine serves them on phone-class devices
(screen short side ≤600 CSS px; tablets/iPads get the master), falling back to the
desktop `clip` when a mobile one is absent:

```js
sections[k].clipMobile   = 'assets/vid/<name>-m.mp4';
sections[k].posterMobile = 'assets/<name>-poster-m.webp';
connectorsMobile = ['assets/vid/conn1-m.mp4', …];   // length N-1, in order
```

If phone scrubbing still stutters, tighten the GOP further (`-g 2`, or `-g 1` for all-intra
= instant seeks at the cost of larger files); if cellular weight is the bigger worry, raise
`crf` (24–26) or drop to `scale=-2:600`. If the master is already 720p (e.g. kling3_0 std),
the mobile encode still pays off — the tighter GOP is what makes phone seeks cheap.
Plain mobile-tier encodes stay 16:9 — the engine centre-crops them; for true portrait
assets see §7.

## 7. Portrait tiers (Step 1.6 hero-reframe / full portrait chain) — extra credits

The gold standard for mobile is a **differently-framed render, not a crop** (Apple ships
re-art-directed per-breakpoint assets). Two levels:

**Hero reframe (cheap).** For the 1–2 scenes whose focal subject can't hold a centre
crop (usually hero + finale): regenerate JUST those stills at 9:16 (same prompt + style
anchor, recompose vertically), render a 9:16 dive from each, encode with `encm()`
settings, wire as that scene's `clipMobile`/`posterMobile`. Other scenes keep the
cropped 16:9 mobile encode. Seams: a portrait dive still hands off within its own clip
only, so nothing about the 16:9 chain changes — the phone crossfades between a cropped
connector and the portrait dive; keep the reframed composition centred on the same focal
point so the transition reads.

**Full portrait chain (gold, ≈2× video credits).** A complete parallel 9:16 chain:

- 9:16 stills (or reuse 16:9 stills as `--image` style refs and prompt the vertical
  recomposition), then the full Step 4/5 flow at `--aspect_ratio 9:16` — own frame
  extractions, own connectors, own handoffs. **Aspect ratios cannot mix mid-chain**: a
  9:16 clip can't continue a 16:9 frame, so the portrait chain is generated end-to-end
  as its own world.
- Run the §5c SSIM gate on the portrait chain separately (its own seam list).
- Encode with `encm()` (already 720-class; portrait at `scale=720:-2`), extract portrait
  posters, wire ALL of it as `clipMobile`/`connectorsMobile`/`posterMobile`.
- Same-model rule, same NSFW re-roll budget — everything from the 16:9 chain applies.

State the credit cost to the user before starting either tier (SKILL Step 1.6).

## Notes

- `.[0].result_url` is the field on the `--wait --json` job object. `.min_result_url` is
  a lower-res preview if you ever want it.
- **NSFW fallback across models**: if one clip keeps getting flagged on seedance after
  re-rolls + prompt scrubbing, regenerate just that clip on `kling3_0` with the SAME
  start/end frames: `VMODEL=kling3_0; VOPTS="--mode std --sound off"; gen_conn 3 …` —
  then restore your chain model. See SKILL Gotchas for the trade-off.
- **Previz**: the draft-tier pass is the recommended default — see the setup block at
  the top. Don't reach for reference-only models for it: without `--start/--end-image`
  they can't hold a seam, so their output can't be chained (Step 4 rule).
- If a whole batch stalls, check `higgsfield workspace list` for credits and
  `$WORK/*.err` for the reason.
- Concurrency: launching ~5–6 gens at once is fine; much more can trigger transient
  credit/race errors — stagger or re-roll.
