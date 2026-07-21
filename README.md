# Trip Planner

A single-file web app for planning trips while studying abroad. It gives you a
semester-at-a-glance timeline plus a detailed view for each trip (itinerary,
transportation, accommodation, packing, budget, and notes with photos). No build
step, no framework, no server — one `.html` file that runs in any browser.

This document is written so you (or Claude Code) can pick the project up, run it,
understand every part, and extend it.

---

## Quick start

- **Just run it:** open `trip-planner.html` in any browser (double-click it).
- **Serve it locally** (nicer, avoids `file://` quirks):
  ```bash
  npx serve .        # or: python3 -m http.server
  ```
  then open the printed URL.

There is no install and no dependencies. Fonts load from Google Fonts over the
network; everything else is inline.

---

## What it does

**Overview screen**
- Editable **semester budget** at the top, with live "spent so far" and "left to
  spend" rolled up across every trip.
- A **timeline ribbon** plotting all trips across the semester, with a "today"
  marker. Tap a trip to open it.
- A **card per trip**: countdown, dates, quick counts (plans / transit / packed),
  and a budget bar.

**Per-trip detail** — six tabs:
- **Itinerary** — day-by-day stops (date, time, title, note), auto-grouped and sorted.
- **Transport** — flights/trains/buses/etc. (mode, route, date/time, confirmation, cost).
- **Stay** — accommodations (name, check-in/out, address/confirmation, cost).
- **Packing** — a checklist with progress.
- **Budget** — directly editable **Budget** and **Spent**; optional itemized
  purchases that add into Spent automatically.
- **Notes** — free text (autosaves) plus **photo uploads** (downscaled and stored
  with the trip).

**Data safety**
- **Back up** — copy all data out as JSON (or download it as a file).
- **Restore** — paste/load a backup to replace current data.
- A **template** copy (`trip-planner-template.html`) starts empty and is meant to
  be shared with someone else — each copy is fully independent.

---

## Design language

Nordic / Scandinavian-inspired: calm, lots of whitespace, one signature element
(the timeline ribbon). Everything is themed via CSS variables at the top of the file.

- **Palette:** paper `#EAEEEB`, card `#FBFBF8`, ink `#1B2A2A`, petrol `#2E5A5C`
  (primary accent), ochre `#C98A2B`, rust `#B85539` (alerts/over-budget).
- **Trip accent colors:** the `PALETTE` array (petrol, rust, ochre, navy, plum,
  moss, teal, bronze) — each trip picks one.
- **Type:** `Bricolage Grotesque` (display/headings), `Instrument Sans` (body/UI),
  `Space Mono` (dates, money, labels).

---

## Architecture

One file, vanilla JS, no framework. Structure inside `<script>`:

- **State** — a single `state` object: `{ view, tripId, tab, trips[], overallBudget }`.
- **Rendering** — pure functions return HTML strings (`overviewHTML`, `detailHTML`,
  `tabBody`, and per-tab `*Body` functions). `render()` swaps `#app` innerHTML;
  `renderTab()` swaps only the active tab body.
- **Events** — **event delegation**, attached **once** in `initEvents()` to `#app`.
  This is important: because tab bodies are re-rendered on every change, per-element
  listeners would go stale. Delegation on the stable `#app` element survives all
  re-renders. (An earlier version bound listeners per-render and buttons went dead
  after a refresh — delegation is the fix.)
- **Mutations** — small helpers update `state`, call `persist()`, then re-render.

### Data model

```js
state = {
  view: "overview" | "detail",
  tripId: string | null,
  tab: "itinerary" | "transport" | "stay" | "packing" | "budget" | "notes",
  overallBudget: number,          // editable semester-wide budget
  trips: Trip[]
}

Trip = {
  id, destination, country, color,
  startDate, endDate,             // "YYYY-MM-DD"
  budget: number,                 // planned, editable
  spent: number,                  // actual, editable; expenses add into this
  notes: string,
  itinerary: [{ id, date, time, title, note }],
  transport: [{ id, mode, label, date, time, detail, cost }],
  stay:      [{ id, name, checkin, checkout, detail, cost }],
  packing:   [{ id, text, done }],
  expenses:  [{ id, label, cat, amount }],
  images:    [{ id, src }]         // src = base64 data URL (downscaled to ~1280px JPEG)
}
```

### Persistence

Saved on every change via `store.save()`, which writes the payload
`{ trips, overallBudget }` to **two** places (best-effort, each wrapped in try/catch):

1. **`window.storage`** — Claude's per-artifact persistent storage. Only works when
   the artifact is **published** on a paid plan, web/desktop. Key: `"planner"`.
2. **`localStorage`** — works for a downloaded/hosted copy in a normal browser.
   Key: `"tripPlanner.v2"`.

`store.load()` reads new keys first, then falls back to older keys for migration.
`normalize()` fills in any missing fields so old backups keep working.

> ⚠️ **No cross-device sync.** Each browser/device keeps its own copy. Moving data
> between devices is manual (Back up → Restore). Removing this limitation is the
> main item on the roadmap below.

---

## Known limitations

- **No automatic sync** between phone and computer (see roadmap).
- **Mobile auto-save is unreliable** — `window.storage` is web/desktop only; a
  downloaded/hosted copy relies on `localStorage`, which mobile browsers keep but
  don't share across devices.
- **Photos are heavy.** They're base64 strings inside the JSON and count against
  storage limits (~20 MB `window.storage`, ~5 MB `localStorage`). Fine for a handful
  per trip; a real image store is the proper fix.
- **`file://` on iOS** is awkward (Safari won't cleanly run/keep a local file).
  Hosting it (roadmap) is the smooth path to "on my phone."

---

## Roadmap — taking it further in Claude Code

Ordered by impact for the "use it everywhere, especially my phone" goal:

1. **Make it a PWA + host it.** Add a `manifest.json` and a service worker, split
   the single file into `index.html` / `styles.css` / `app.js`, and deploy to a
   static host (Netlify, Vercel, or GitHub Pages). Result: a stable URL you can
   "Add to Home Screen" for a real app icon that works offline.
2. **Add cross-device sync.** Introduce a lightweight backend so your trips follow
   you across phone and laptop. Good low-effort options: Supabase or Firebase
   (auth + database in ~an afternoon), or a tiny serverless key-value store keyed to
   a login. Replace `store.save/load` with calls to it; keep `localStorage` as an
   offline cache.
3. **Move photos to real file storage.** Upload images to object storage (e.g.
   Supabase Storage / S3) and keep only URLs in the trip data — removes the size cap.
4. **Optional multi-user.** With accounts in place, your friend just signs in and
   gets her own private data instead of needing a separate file.
5. **Nice-to-haves.** Currency selector, transport/stay costs optionally feeding the
   budget, category spending breakdown, recurring packing templates, calendar export
   (`.ics`).

### Paste-into-Claude-Code starter prompt

> I have a single-file vanilla-JS trip planner (`trip-planner.html`, included) with a
> README describing its features, data model, and persistence. I want to turn it into
> a hosted Progressive Web App I can install on my phone, and add cross-device sync so
> my trips stay in sync between my laptop and phone.
>
> Please: (1) split the single file into `index.html`, `styles.css`, and `app.js`
> without changing behavior; (2) add a PWA manifest and service worker so it installs
> and works offline; (3) add Supabase for email login plus a `trips` table, and swap
> the current `store.save`/`store.load` to read/write there while keeping
> `localStorage` as an offline cache; (4) keep the existing Back up / Restore JSON
> feature. Preserve the current design (CSS variables, Bricolage Grotesque / Instrument
> Sans / Space Mono, the timeline ribbon). Walk me through deploying it.

---

## Files in this bundle

- `trip-planner.html` — the full working app (your personal copy).
- `trip-planner-template.html` — an empty copy meant for sharing (starts blank,
  no location label).
- `README.md` — this document.
