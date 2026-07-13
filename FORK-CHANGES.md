# How this fork differs from oso95/scroll-world

Upstream builds the same thing — a scroll-scrubbed "fly through the world" landing page
with frame-locked seams. This fork keeps all of that (prompts, seam doctrine, engine)
and hardens the parts that cost real money and real visitors: **credit spend, automated
verification, mobile, and SEO.** Engine config stays backward compatible — an upstream
config runs unchanged; every new field is optional.

## 1. Credit / spend controls (the big one)

A 6-scene upstream run fires ~17 generations (6 stills + 11 videos) with no gate after
the interview — a style miss or mid-run crash burns real credits.

- **Budget tier question** asked *before* the journey is designed: lean (~8 gens,
  4 scenes, architecture A) / standard (~11–14) / showcase (17+). Scene count and
  architecture follow from the answer. Fewer scenes get longer per-scene dwell
  (`scroll`/`linger`) so a 4-beat world still reads complete.
- **Spend estimate + explicit go-ahead** closes the interview: gen counts, re-roll
  buffer (~20–30% on interiors), runtime.
- **Anchor-still gate**: ONE still generated first, user approves the art direction,
  then the rest batch with `--image` style lock. Style miss costs 1 gen, not N.
- **Previz by default** (5+ scenes): whole chain rendered on `seedance_2_0_mini`
  (frame-locking intact), page reviewed from drafts, full-model credits spent only
  after approval. Mini can also BE the final model at 720p on lean budgets.
- **Idempotent pipeline**: every `gen_*` skips existing outputs (output file = run
  state) + `status()` helper. A crash or NSFW re-roll never repays finished work.
- **Cheap-seam levers documented**: connectors are individually optional (`null` slot
  = direct crossfade), spend them on the seams around hero scenes.

## 2. Automated verification (upstream QA is eyeball-only)

- **SSIM seam gate** (`pipeline.md §5c`): every seam machine-checked from the encoded
  files. ≥0.90 pass / 0.75–0.90 warn / <0.75 fail-with-cause. Runs before browser QA
  and after every re-roll (replacing one clip touches both its seams).
- **Poster = extracted first frame of the encoded clip** (`§5b`, engine `poster` /
  `posterMobile`): upstream uses the 3:2 source still as the loading poster for a 16:9
  re-rendered clip — visible jump on the first paint a visitor sees. Same frame-handoff
  doctrine as the connectors, applied to seam zero.
- **Exact-frame handoffs for architecture A** (`§3` note, gotchas): the -0.15s "last
  frame" extraction is a connector recipe; used as an arch-A chain handoff it plays as a
  ~4-frame rewind at every seam (measured SSIM 0.59 vs ≈0.90+ with the true last frame
  via `-update 1`). Found by running the chain on a local Wan 2.2 5B pipeline, where the
  SSIM gate flagged it — the gate doing exactly its job.

## 3. Mobile (research-backed: live apple.com asset probes, OPTIKKA/Shopify/Horeca
case studies, caniuse/MDN support tables)

- **Clip tier by device class** (screen short side ≤600 CSS px), not pointer type —
  upstream serves 720p mobile encodes to an iPad Pro (coarse pointer, desktop-class
  screen). Behaviour hardening still keys off touch.
- **iOS Low Power Mode fallback**: LPM rejects even muted `play()` and breaks
  `currentTime` scrubbing — upstream shows frozen/blank scenes. The engine detects the
  rejected prime on first touch and flips to stills-with-crossfades.
- **Data-saver / slow network** (Chromium signals, used as downgrade-only): `saveData`
  → stills mode; 2g/3g → shrunken clip prefetch window. iOS exposes none of these
  APIs, so the baseline stays conservative for everyone.
- **`scrollMobileFactor`** (default 1.2): longer scroll run per scene on phones.
- **Four honest mobile tiers** replacing upstream's single "beta" toggle: crop-safe
  (free) / mobile encodes (720p `-g 4` + matching posters) / hero reframe (9:16
  re-renders of hero + finale only) / full portrait chain (parallel 9:16 chain with its
  own handoffs + SSIM gate, ≈2× video credits — the Apple approach: re-framed render,
  not a crop).

## 4. SEO

Upstream renders all copy client-side — zero crawlable text on a landing page. The
template ships a `data-sw-seo` static-copy block (h1 + per-section h2/p + real CTA
links); the engine hides it on mount.

## 5. Docs restructure

- Full gotchas moved to `references/gotchas.md` (top 5 run-blockers stay inline in
  SKILL.md), with new entries: Low Power Mode, iPad tier, data-saver inversion.
- Canvas frame-sequence renderer (Apple's actual technique — deterministic paint, no
  decoder seeks) documented as the upgrade path when video scrubbing isn't smooth
  enough.

## Engine config — new optional fields

```js
scrollMobileFactor: 1.2,          // mobile scroll-run multiplier
sections: [{ poster, posterMobile, /* + everything upstream */ }],
// data-sw-seo block in the container = crawlable copy, hidden on mount
```

All verified headless (Chromium): desktop / phone / tablet / data-saver / Low Power
Mode scenarios, plus SEO-hide, poster preference, and scroll-factor math.
