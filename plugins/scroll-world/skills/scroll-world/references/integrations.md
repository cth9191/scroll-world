# Conversion integrations — analytics, lead capture, audio

The base skill ships a beautiful page that is a **dead end**: the finale CTA is `href:'#'`,
nothing is measured, nothing is captured. This file wires the three things a shipped
landing page actually needs — **GA4 flight analytics**, **GHL lead capture**, and optional
**ElevenLabs scroll audio**. Everything here is **opt-in**: omit a config block and the page
behaves exactly as before. The engine support already lives in `scrub-engine.js`; this doc
is the operator's recipe.

Ask for these three in the interview (SKILL Step 1.7) so you have the values before you wire
the page (SKILL Step 9).

---

## 1 — GA4 flight analytics

A scroll-scrubbed page is analytically **blind by default**: there are no page navigations
and no clicks until the very end, so a stock GA4 install records one pageview and nothing
about whether anyone actually *flew the world*. The engine fixes that by emitting flight
events you can turn into **validation gates**.

### Wire it

Two ways — pick one:

- **Let the engine load GA4.** Pass your measurement id and it injects `gtag` for you:
  ```js
  analytics: { ga4: 'G-XXXXXXX' }
  ```
- **Reuse an existing GA4 / GTM install.** If you already put the GA4 or GTM snippet in
  `<head>` (or the page is inside a site that has one), pass `true` — the engine will use the
  existing `gtag()` / `window.dataLayer` and inject nothing:
  ```js
  analytics: { ga4: true }
  ```

Options: `prefix` (event-name prefix, default `sw_`), `params` (merged into every event —
handy for a campaign tag), `debug:true` (console-logs each event as you scroll).

### Events emitted

| Event | When | Params |
|---|---|---|
| `sw_scene_view` | a section becomes the active scene | `scene_index, scene_id, scene_label` |
| `sw_scroll_depth` | whole-flight depth crosses 25/50/75/100% (once each) | `percent` |
| `sw_flight_complete` | the visitor reaches the end (once) | — |
| `sw_cta_click` | any CTA button (finale or top bar) is clicked | `scene_index, label, href` |
| `sw_lead_submit` | the inline capture form posts successfully | `source` |
| `sw_audio_toggle` | the visitor toggles sound | `on` |

If `gtag` is absent the engine pushes `{ event:'sw_…', …params }` to `window.dataLayer`
instead, so **GTM** picks the same events up as Custom Event triggers.

### Map events → validation gates

This is the point: the flight events feed the same **gate methodology** used elsewhere —
ship, watch two numbers, decide.

- **G1 — completion / attention.** `sw_flight_complete` count ÷ sessions, and the
  `sw_scroll_depth{percent:75}` rate. If most visitors bail before 50%, the journey is too
  long or the early beats don't earn the scroll — cut a scene or raise its payoff.
- **G2 — engagement / intent.** (`sw_cta_click` + `sw_lead_submit`) ÷ sessions. This is the
  real conversion signal; `sw_lead_submit` is the money event — mark it as a **Key Event**
  (Conversion) in GA4 Admin so it shows in reports and can be imported to Ads.

In GA4: **Admin → Events** to see them appear (give it ~24h, or use **DebugView** now with
`debug:true`). Mark `sw_lead_submit` (and optionally `sw_flight_complete`) as Key Events.

---

## 2 — GHL lead capture (the finale becomes real)

`capture:{…}` turns the last section from a `#` link into an actual capture surface. Two
modes; both are additive and only render on the **finale** section.

### Mode A — embed (fastest, most robust): `mode:'embed'`

Drop a GHL **Form** or **Booking/Calendar** widget straight in as an iframe. GHL handles the
submission, validation, and pipeline routing — you wire nothing.

```js
capture: {
  heading: 'Book your discovery call',
  mode: 'embed',
  embedUrl: 'https://api.leadconnectorhq.com/widget/booking/XXXXXXXX', // or /widget/form/XXXX
  height: 680,
}
```

Get the URL in GHL: **Sites → Forms/Calendars → your widget → Share → the iframe `src`**.
Because GHL owns the form, source attribution and pipeline placement are set **inside GHL**
(the form/calendar settings or the workflow it triggers). This is the recommended default —
nothing to break, and the booking flow is native.

> Mobile: an embedded widget is tall. On phones the finale copy sits at the bottom, so keep
> `height` modest (≤620) or prefer the inline form (below) for phone-heavy traffic.

### Mode B — inline form → GHL inbound webhook: `mode:'inline'`

A native, on-brand form (no iframe) that POSTs JSON to a GHL **Inbound Webhook**, so you own
the markup and the exact payload — with **source tagging** and **UTM passthrough** baked in.

```js
capture: {
  heading: 'Start here',
  note: 'Leave your details and we’ll be in touch.',
  mode: 'inline',
  webhookUrl: 'https://services.leadconnectorhq.com/hooks/LOCATION/webhook-trigger/UUID',
  source: 'scroll_world_<subject>',   // becomes the lead's source → your pipeline tag
  fields: [                            // optional; this is the default set
    { name:'name',  label:'Name',  type:'text',  required:true },
    { name:'email', label:'Email', type:'email', required:true },
    { name:'phone', label:'Phone', type:'tel',   required:false },
  ],
  submitLabel: 'Get started',
  thankYou: 'Got it — we’ll reach out shortly.',
  // redirect: 'https://…/thank-you',  // optional post-submit redirect
}
```

**Set up the GHL side:**
1. **Automation → Workflows → Create → Start with: Inbound Webhook.** Copy the webhook URL
   into `webhookUrl`.
2. Trigger a test submit (fill the form on the page) so GHL captures a sample payload, then
   **map fields**: `name` → Contact Name, `email` → Email, `phone` → Phone. The engine also
   sends `source`, `brand`, `page`, `referrer`, and any `utm_*` / `gclid` / `fbclid` present
   in the URL — map `source` to a tag or the Contact's Source, and stash UTMs on custom
   fields for attribution.
3. Add **Create/Update Contact**, apply a **tag** (e.g. `scroll-world`), and **Add to
   Pipeline / Opportunity** at the stage you want new leads to land — the standard
   contact→tag→pipeline shape; if an existing funnel already uses it, reuse that pipeline.

**What the engine sends** (JSON body):
```json
{ "name":"…", "email":"…", "phone":"…",
  "source":"scroll_world_<subject>", "brand":"<brand name>",
  "page":"<full url>", "referrer":"<document.referrer>",
  "utm_source":"…", "utm_medium":"…", "utm_campaign":"…", "gclid":"…" }
```

**Behavior & guardrails (already in the engine):**
- A hidden **honeypot** field (`company_url`) silently drops bots that fill it.
- Required-field check before send; on success the form swaps to the `thankYou` message and
  fires `sw_lead_submit`.
- The POST is `mode:'no-cors'` — GHL inbound webhooks are public POST endpoints and the
  response is CORS-opaque, so the engine **fires and trusts** and you confirm receipt in the
  GHL workflow (watch the first live test land as a contact). Don't expect a status code in
  the browser; that's normal for this endpoint type.

**Security (hard rule):** the **webhook URL is the only credential and it is meant to be
public** — an inbound endpoint that only accepts writes. **Never** put a GHL **API key**,
location token, or private key in client JS; if a task ever needs the GHL *API* (not the
inbound webhook), that call belongs on a server/edge function, never in `scrub-engine.js`.

---

## 3 — ElevenLabs scroll audio (optional)

Per-scene narration or ambient that **fades in and out with the flight** — each scene's
volume tracks its on-screen opacity, so as the camera leaves one scene and enters the next,
its voice recedes and the next one rises. This is the audio analogue of the seamless visual
chain.

### Generate the narration (ElevenLabs)

- One short file per scene you want voiced. Keep them **tight** (a sentence or two — this is
  a landing page, not a chapter) and write for the beat, not the whole pitch.
- Use a **consistent voice** across scenes for cohesion (or a deliberate two-voice pattern if
  the concept calls for it). Multilingual v2 or Turbo v2 are both fine; Turbo is cheaper for
  short lines.
- Loudness-match the files so one scene isn't louder than the next (the engine only scales
  volume by opacity; it doesn't normalize).

### Encode small

Landing-page audio must not become the heaviest asset on the page. Mp3 at a modest bitrate
is plenty for voice:

```bash
ffmpeg -i scene1.wav -ac 1 -c:a libmp3lame -b:a 96k scene1.mp3   # mono, 96 kbps voice
# ambient beds can go lower: -b:a 64k
```

Put them under `assets/aud/` and wire per section + enable globally:

```js
audio: { unmuteLabel: 'Play narration', onLabel: 'Mute', gain: 1 },  // global
// …and on each voiced section:
{ id:'farm', /* … */ audio: 'assets/aud/farm.mp3' }
```

### The autoplay reality (why there's an unmute button)

Browsers block audio until a user gesture, and iOS blocks it harder. So the engine is
**muted until the visitor taps the "Play sound" toggle** (bottom-left). After that, each
scene's audio plays and its volume follows the scene opacity as you scroll. This is honest:
you cannot autoplay sound, and you shouldn't — a landing page that blares on load bounces.
The toggle is the affordance.

**Off automatically** under `prefers-reduced-motion` and Chromium **data-saver** — the engine
never creates the audio elements there (`preload:'none'` means nothing is fetched until a
tap, so muted visitors pay zero bytes). Loop is on by default so a short bed doesn't fall
silent if a visitor lingers.

**Costs to state to the visitor's device:** audio adds weight on mobile data; keep files
small and consider skipping audio on the mobile tier if the world is data-sensitive. Voice
narration also raises production cost per run (ElevenLabs credits) — treat it as a showcase
add-on, not a default.

---

## Deploy (Vercel) & framework drop-in

- **Vercel (static).** The generated page is plain HTML + `scrub-engine.js` + `assets/`. From
  the page directory: `vercel --prod` (or drag the folder into the Vercel dashboard). Set
  the GA4 id and the GHL webhook/embed URL in the config **before** deploying; they're not
  secrets (see the security note above), so they can live in the committed HTML. Confirm the
  `data-sw-seo` block is present in the served HTML (View Source) so crawlers and link
  previews get real copy.
- **Drop into an existing app (Lovable / Next / Vue).** Copy `scrub-engine.js` into
  `public/`, put clips/stills/audio under `public/assets/`, and call `mountScrollWorld(ref,
  config)` from a mount effect (React `useEffect`, Vue `onMounted`) on the target route.
  **Server-render the `data-sw-seo` block** into that route's markup so the SEO copy exists
  without JS. The engine injects its own scoped CSS (a `@layer`, so your app's tokens win),
  so it won't fight the host styles.
