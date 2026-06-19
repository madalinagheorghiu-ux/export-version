# Export · Download · Notifications — prototype

Interactive prototype for the FlowX version **export → notify → download** experience, with the excluded-resources impact panel. Built in isolation from the FlowX codebase but mirroring the real Designer surfaces (header, Branching console, Export Version modal).

## Run it

No build step. You need either Node or any static server.

```bash
node server.js
# open http://localhost:8000
```

> The folder/repo is fine to clone anywhere. If you open `index.html` via `file://` it should also work, but a local server (`node server.js`) is the reliable path.

## What to try

1. Click the **Branching** pill in the header → **Export Version**.
2. Pick a **target platform version**:
   - **5.9.X (Current)** → no exclusions, clean download.
   - **5.5.x** → 5 excluded resources (amber hint appears).
   - **5.1.x** → 9 excluded resources.
3. Click **Export** → "Preparing…" (~6s, simulated from the real up-to-15-min). **Close the modal and navigate** the left nav — when it finishes you get a **toast + bell badge** anywhere in the app.
4. Open the **bell** → notification + a slim **Exports** history. Click it → **Download** modal with the excluded-resources panel.
5. Re-export the same target with the same media toggle → **instant** (cache/dedup hit).

## What's mocked

Compressed delay, in-memory cache/dedup (keyed on `source@version → target | media`), per-version exclusion sets, and simulated notifications/toasts. "Download" produces a JSON manifest standing in for the ZIP.

## Files

- [`index.html`](index.html) — the self-contained app (React 18 + Babel via CDN).
- [`server.js`](server.js) — tiny zero-dependency Node static server.
- [`EXPORT_DOWNLOAD_PLAN.md`](EXPORT_DOWNLOAD_PLAN.md) — the living design/implementation plan, decision log, and open questions.
