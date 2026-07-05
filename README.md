# Encore Office Map

A touch-first web app for the IT service desk: visit a site, lay out the office
map on a blank canvas, assign names, and track a per-seat checklist of
hardware and to-dos.

**One file. No build, no dependencies, no CDN.** `index.html` contains the whole
app — Objective brand fonts, the Encore icon/wordmark, and the jsPDF library
(MIT) are embedded, so it works from any static host (or even opened from a
file share) with no internet access.

---

## Using the app

| Mode | What you do |
|---|---|
| **Layout** | `+ Cubicle` / `+ Office` add the two main seat types; `+ Other…` opens the full space list (Conf. Room, Reception, Huddle, Break Room, Warehouse, Server Room, Storage, Restroom, Copy/Print, Door) plus a **custom label** field for anything else (e.g. "Mother's Room"). Drag to move (snaps to an 8px grid). Tap to select — then drag the blue corner dot to resize, or use Rename / Duplicate / Delete. Undo covers all layout changes. |
| **Checklist** | Tap any space → bottom sheet: assign a person, build the checklist (quick-add chips for common hardware + free-text items), keep notes (port numbers, follow-ups). |

- **Pinch to zoom, one-finger drag to pan** (in Checklist mode, dragging anywhere pans — seats can't be moved accidentally). Double-tap empty canvas to zoom. Mouse: wheel zooms, drag pans. The **percentage readout** under FIT shows the current zoom; tap it to snap back to 100%.
- Space colors: **blue** = not started, **orange** = in progress, **green** = every checklist item done. The header badge shows completed spaces for the site.
- **Duplicate is the fast path**: build one cubicle's checklist, select it, tap Duplicate — the copy keeps the checklist items (unchecked) with a blank name.
- **Site name** in the header opens the site picker — alphabetically sorted, scrolls past ~10 sites. The **gear** opens Settings: new/rename/duplicate/delete site, **site address**, JSON **Export / Import**, and **Export PDF** — a one-page landscape-letter snapshot of the whole map (status colors, names, progress) with the site name + address bottom-left and the Encore wordmark bottom-right.

### iPad / iPhone
Open the URL in Safari → Share → **Add to Home Screen**. It launches full-screen
with the Encore icon and works offline after first load (data is saved on the
device via localStorage; use Export to back it up or hand it to a teammate).

## Local preview

```
npx serve encore-seat-mapper
```

---

## Where should the data live? (Microsoft / Azure options)

Today the app stores data **on the device** (localStorage) with JSON
export/import to move it around. That already keeps data off third-party
clouds. For shared, centralized data inside the company tenant, ranked options:

### 1. Azure Static Web Apps + Azure Functions + Table Storage — recommended
- **Hosting:** Azure Static Web Apps (Free tier) serves `index.html`.
- **Auth:** SWA's built-in Entra ID login (`staticwebapp.config.json`, no MSAL code) — only company accounts can open the app.
- **Data:** a small Functions API (comes free with SWA) reads/writes one JSON document per site to **Azure Table Storage** or **Cosmos DB serverless**.
- **Cost:** effectively $0 (free hosting tier, pennies for storage).
- **Why it fits:** same "static page + tiny REST backend" shape as the old GitHub + Firebase setup, but everything lives in the company subscription behind Entra ID. The app already treats a site as a single JSON document, so the API is ~4 endpoints (list/get/put/delete site).

### 2. SharePoint List via Microsoft Graph — zero Azure resources
- Store each site as an item in a SharePoint/Microsoft List (JSON in a multiline column); the app signs in with MSAL.js and calls Graph.
- Data lives in M365 where IT already manages permissions, versioning, and retention. No Azure subscription needed.
- Trade-offs: an Entra app registration + consent is required; Graph throttling; JSON-in-a-column is crude (no server-side querying of seats).

### 3. Keep it serverless: localStorage + Export to Teams/SharePoint — works today
- Techs export the site JSON when done and drop it in a Teams channel or SharePoint library; anyone can import it.
- Zero infrastructure, data at rest in M365, fully offline-capable in the field (remote sites often have no Wi-Fi for guests anyway).
- Trade-off: no live multi-user sync; last export wins.

### 4. Power Apps + Dataverse — not recommended for this UI
Fully managed and mobile-ready, but rebuilding the pinch-zoom canvas in Power
Apps would fight the platform, and Dataverse needs premium licensing per user.

**Suggested path:** ship with option 3 now (it's already built), stand up
option 1 when live sharing between techs becomes a need. The storage layer is
isolated in `load()` / `save()` / `flushSave()` in `index.html`, so swapping
localStorage for a fetch-based API is a contained change.

---

## Technical notes (for the next maintainer)

- **Gesture engine:** custom pointer-events state machine (`pan / pinch / move / resize / tap` with an 8px tap-slop), `touch-action:none` on the canvas, pinch anchored to the finger midpoint, second-finger landing mid-drag commits the drag and becomes a pinch. This replaces v1's CSS-scale + browser-scroll approach that was janky on iPad.
- **Rendering:** seats are absolutely-positioned divs inside a `translate/scale` world layer; grid dots are drawn on the stage background and kept in sync with the camera. Selection handle and borders are compensated by `--inv` (1/scale) so they stay a constant screen size at any zoom.
- **Persistence:** debounced 250ms writes to `localStorage['encore-seatmap-v1']`, flushed on `pagehide`/`visibilitychange`. A corrupted payload is backed up to `…corrupt-backup`, never overwritten. Per-site camera position is saved and restored.
- **Undo:** snapshot stack (50 deep) per site, layout operations only.
- All inputs are 16px+ (prevents iOS focus-zoom); safe-area insets handled on all four edges; `user-scalable=no` + gesture-event suppression stops Safari page zoom from fighting the canvas.
