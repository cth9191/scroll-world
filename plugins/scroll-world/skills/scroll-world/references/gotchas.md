# Gotchas (hard-won)

The five most run-blocking of these are summarized inline in SKILL.md; this is the full
list. Symptom → cause → fix.

- **Seam pop** → connector endpoints were the diorama stills, not the neighbouring
  clips' actual frames. Always extract real frames (SKILL Step 5). The SSIM gate
  (pipeline.md §5c) catches this before a browser ever opens.
- **Seam stutter / camera "jumps backward"** → even with frame-matched seams, if the
  camera *velocity reverses* (forward dive, then a connector that pulls back out) it
  reads as a rewind. This is inherent to architecture B. For any grounded walkthrough use
  architecture A (one continuous forward take — legs chained from actual last frames, no
  pull-back, no `--end-image`); see SKILL Step 4.
- **Still→video pop on first paint** → the poster is the 3:2 source still, not the
  encoded clip's extracted first frame. Wire `sections[k].poster` (pipeline.md §5b).
- **Frozen video / stuck at frame 0** → `seekable=[0,0]`; the host isn't serving byte
  ranges. Use blob URLs (engine does).
- **Huge files** → you used all-intra. Use `-g 8` + blob instead.
- **Soft / low quality** → you downscaled or over-compressed. Encode native 1080p,
  crf ≤ 20, add `unsharp`. Video is inherently softer than the stills — keep the stills
  as the lite fallback for max fidelity.
- **Concurrent gens 503 / "not_enough_credits" race** → transient when many launch at
  once; re-roll the individual failure, it's not really out of credits (verify with
  `higgsfield workspace list`).
- **NSFW false-positives (Seedance `status "nsfw"`)** → the video content filter flags
  perfectly innocuous clips, especially **bedroom, pool, spa/wellness** contexts and
  trigger words like "bed", "pool", "waterfall", "wine", "swim". It's partly the prompt
  wording and partly the reference frames. Fixes, in order: (1) re-roll — it's often
  non-deterministic and passes on the 2nd–3rd try; (2) strip trigger words and add
  "empty, unoccupied, no people, no figures, architectural, tasteful"; (3) regenerate
  just that clip on **`kling3_0`** with the same start/end frames — a different
  provider's filter often passes what Seedance blocks. Expect a slight render-character
  shift on that one clip (each model has its own grain/motion feel); for a 5s connector
  behind a crossfade that usually beats option (4): set the connector slot to `null` —
  the engine crossfades that seam directly (optional connectors), so the page still
  completes. Budget extra credits/time for these re-rolls on interiors/real-estate content.
- **Dark / custom theme** → the engine wraps its default tokens in `@layer sw`, so a
  page-level `:root` / `.sw-root { --sw-bg; --sw-ink; --sw-accent; --sw-font-* }` block
  wins cleanly (no specificity hacks). `--sw-ink` is your primary **text/heading** colour;
  the **accent** fills the primary button and active nav. For a dark theme, set `--sw-bg`
  dark and `--sw-ink` light — the copy scrim and title shadow follow `--sw-bg` automatically.
- **Phone scrub stutters / freezes on a fast flick** → the 1080p master is too heavy for a
  phone decoder and seeks pile up. Ship the `-m.mp4` mobile encodes (720p, `-g 4`) and wire
  `clipMobile`/`connectorsMobile` (SKILL Step 6/7). The engine already coalesces seeks; the
  lighter encode is the other half. Still choppy on a low-end device? Tighten GOP
  (`-g 2` / all-intra).
- **Blank / black scene on iOS (desktop was fine)** → an iOS Safari quirk: a muted video that
  was never played won't paint a seeked frame. The engine fixes this by keeping the poster up
  until the clip paints and priming each video on first touch — so **don't** hide the
  poster on `loadedmetadata` or strip the `playsinline`/`muted` attributes if you adapt the
  engine into a framework.
- **Frozen video on iOS with battery saver on** → iOS **Low Power Mode** rejects even a
  muted, playsinline `play()`, and `currentTime` scrubbing doesn't work either — no video
  technique survives it. The engine detects the rejected prime on first touch and flips
  the whole page to stills-with-crossfades. If you adapt the engine, keep that fallback:
  a `.catch()` on the priming `play()` that enters stills mode.
- **iPad gets blurry 720p video** → clip tier was keyed to pointer type. iPadOS reports a
  coarse pointer AND a desktop Mac UA — both useless signals. The engine tiers by screen
  short side (≤600 CSS px = phone); if you port it, keep that rule (or decide by
  `clientWidth × devicePixelRatio`), never by pointer/UA.
- **Data-saver / 2g users pull the full blob payload** → Chromium exposes
  `navigator.connection.saveData` / `effectiveType`; iOS exposes nothing. The engine's
  answer: conservative default for everyone (posters first, lazy blob fetch near the
  viewport), stills mode on `saveData`, shrunken prefetch window on 2g/3g. Don't invert
  it (heavy default + downgrade-on-signal) — the devices most on cellular are iPhones,
  which can't signal.
- **Page jumps while scrolling on mobile** → something is re-running layout on the URL-bar
  show/hide `resize`. The engine ignores height-only resizes on touch; if you ported it, gate
  your resize handler on a width change (keep the `orientationchange` path for rotation).
- **Copy hidden behind the URL bar / notch on mobile** → use the engine's safe-area-aware
  bottom offset (`env(safe-area-inset-bottom)` + `dvh`); make sure the page's
  `<meta viewport>` includes `viewport-fit=cover` (the template does).
- **Portrait crops the scene** → a 16:9 clip on a tall phone shows only its centre. Keep each
  scene's focal subject centred with a little headroom (prompts.md), or generate a 9:16 hero
  for the scenes that matter most. The engine centre-crops (`object-fit:cover`); it can't
  un-crop a widescreen composition.
- **`--generate-audio` errors on seedance** → omit it; mute in HTML and `-an` on encode.
- **Kling rejects your flags** → `kling3_0` has **no `--resolution` param** (don't pass
  one; encode at whatever native res ffprobe reports) and **sound defaults on** — pass
  `--sound off`. Duration default is 5; legs/dives want 10.
- **Seam pop only where you "saved credits"** → you swapped models mid-chain, or used a
  start-image-only model where a connector needs an `--end-image`. One model for the whole
  chain; the only cheap tier is `seedance_2_0_mini`, which keeps frame-locking so it stays
  seamless. (Any model with reference-only inputs can't hold a seam at all — SKILL Step 4.)
- **Arch-A seam scores SSIM 0.6–0.8 even though the legs were chained** → the next leg's
  start image was extracted with the *connector* recipe (`-sseof -0.15 … -frames:v 1`),
  which lands ~4 frames before the end — at playback the seam rewinds those frames. For
  architecture-A chaining the handoff must be the clip's TRUE last frame:
  `ffmpeg -sseof -0.3 -i leg.mp4 -update 1 -q:v 2 last.png` (overwrites until EOF).
  Measured on a local Wan 2.2 5B chain: exact frame → seam ≈0.90+; -0.15s offset → 0.59.
  Use the same exact-frame extraction on the A-side of the §5c SSIM check.
- **White-box scenes** → `gpt_image_2` returns a solid bg; either match the page bg to it
  or knock it out (SKILL Step 3).
- **bash 3.2** on macOS → no associative arrays in scripts.

## Beyond video scrubbing — the frame-sequence alternative

Scrubbing `video.currentTime` is the pragmatic middle tier. The technique Apple actually
ships on its scroll pages is a **pre-extracted image sequence drawn to a `<canvas>`**
(and, on modern browsers, WebCodecs decoding frames ahead into canvas): frame paint is
deterministic — no decoder seek latency, no blob download of full clips, frames stream
progressively. Trade-offs: more build tooling (extract N frames/clip, pack as
webp/avif), more requests, and total bytes comparable-to-larger at these clip lengths.
If a client demands butter on low-end devices or the blob payload (~8 MB × 2N-1 clips)
is unacceptable, port the engine's scrub mapping to a canvas frame-sequence renderer —
the seam doctrine, chain math, and pacing knobs all carry over unchanged; only the
"paint frame at time t" primitive swaps.
