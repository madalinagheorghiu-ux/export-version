# Export → Notify → Download — Implementation Plan

> **Purpose:** Living design doc for the version-settings export/download + notification mechanism.
> Keep this updated as decisions are made. See the **Decision Log** at the bottom.
>
> **Last updated:** 2026-06-19

---

## Feature summary

In version settings a user can export a build or a WIP version to import into a **target version**. The download is a two-step flow:

1. **Request export** — choose the target version, trigger the export (may take up to 15 min). If the target platform version is lower than the source's, resources not present in the target are excluded from the ZIP.
2. **Download** — once the ZIP is built, download it; the user sees the impact (excluded resources, if any) before downloading.

The user can close the modal and keep working while the export runs; a notification reaches them **wherever they are** when it's ready. A previously-built identical export is available instantly.

**Goals:** (a) clarity about the process, (b) notifications + actionable follow-up, (c) clarity about the impact of importing a ZIP with excluded resources.

---

## 0. Integration into FlowX Designer (observed UI)

Grounded in the real Designer screens (prototype is isolated, but mirrors these surfaces):

- **Persistent header (dark top bar):** logo · `SANDBOX` · `workspace / project` breadcrumb · branching pill (`main` / `draft` / commit msg) · **Commit Changes to Version** · `Config / Runtime` toggle. This is the only surface visible on *every* page → **the global notification indicator must live here.**
- **Branching console** — opened by the **Branching** (git-branch) icon in the header. Full overlay with: left **Version Details** panel, center **All Branches** graph, right commit list. The export is launched from here.
- **Export entry points:** **Export Version** button in the left Version Details panel, and an export icon at the top of *All Branches*.
- **Export Version modal (exists today):**
  - Info box: *"Choose the target FlowX.AI version and whether to include media files in the export package. The exported ZIP can only be imported into environments running the selected version."*
  - **Target platform version** dropdown — defaults to current (e.g. `5.9.X — Current Version`); lower versions selectable (`5.5.x`, `5.1.x`, …). Choosing lower → exclusions.
  - **Include media file content** toggle.
  - **Cancel** / **Export** actions.
- **Existing color language:** in *Resources changed*, bars are **green = added, yellow = modified, red = deleted**. The excluded-resources panel should align with this vocabulary when color coding is revisited.
- **Runtime → Builds (second export entry point):** the header `Config / Runtime` toggle switches to a runtime shell (nav: Builds · Runtime Settings · Processes · UI Flows Sessions · Task Manager · Configure Params Overrides). The **Builds** page lists builds (e.g. `Bizkids 1.6.2`), each with an **Export build** icon. Exporting a build uses the **same** experience — target platform version + media toggle → processing → notification → download + excluded-resources panel. The only difference is the *source*: a build (`Bizkids 1.6.2`) instead of a committed version (`main 1.6.2`); labels, cache key, and the download manifest reflect it.

This prototype rebuilds these surfaces in isolation and **adds** the missing post-export experience (processing → notification → download + impact), shared by both the **Config** (version) and **Runtime** (build) export entry points.

---

## 1. Core model: an asynchronous, cacheable Export Job

The export is a **server-side job with a lifecycle**. The notification is the messenger that tells the user the job changed state. The download is a separate action against a ready artifact.

### Export Job states

```
REQUESTED ──► PROCESSING ──► READY                     (zip ready, no exclusions)
                   │     └──► READY_WITH_EXCLUSIONS     (zip ready, some resources dropped)
                   └──────►  FAILED
READY / READY_WITH_EXCLUSIONS ──► DOWNLOADED ──► EXPIRED
```

| State | Meaning | User-facing |
|-------|---------|-------------|
| `REQUESTED` | Job accepted, queued | "Preparing your export… (up to 15 min)" |
| `PROCESSING` | Worker building the zip | spinner / progress |
| `READY` | Zip built, all resources included | ✅ "Download ready" |
| `READY_WITH_EXCLUSIONS` | Zip built, some resources excluded | ⚠️ "Ready — N resources excluded" |
| `FAILED` | Build error | ❌ "Export failed — retry" |
| `DOWNLOADED` | User pulled the zip | (no notification) |
| `EXPIRED` | Artifact TTL passed | re-export required |

## 2. Caching / deduplication (the "instant download" path)

Compute a deterministic **cache key** at request time:

```
cacheKey = hash(
  sourceVersionId + sourceContentRevision,   // exact version/build WIP snapshot
  targetPlatformVersion,                      // exclusions depend on this
  includeMediaFiles + mediaFilesRevision      // media set hash
)
```

On a new request:
1. Look up a **non-expired** artifact by `cacheKey`.
2. **Hit** → return the existing job in `READY` / `READY_WITH_EXCLUSIONS` immediately — no worker, no 15-min wait. Download available at once.
3. **Miss** → create a new job, enqueue the worker.

Key on the *content revision* (not just version ID) and on `targetPlatformVersion`, so a changed WIP or a different target produces a distinct artifact. Store artifacts in object storage (S3/MinIO) with a TTL (24–72h). Persist the exclusion report alongside the artifact so a cache hit shows impact without recomputation.

## 3. Exclusion computation (the source of impact)

When the target platform version is **lower** than the source's, some resource types/schemas won't exist in the target. During the build the worker:

1. Enumerates resources in the source snapshot.
2. Checks each against the **target platform version's capability/schema manifest**.
3. Records incompatible resources in an `ExclusionReport`, each entry carrying its full location breadcrumb:

```json
{
  "path": ["Integrations", "Data Sources"],
  "name": "MailQueue",
  "resourceType": "DATA_SOURCE",
  "reason": "Not supported on 5.8.x"
}
```

This report is the single source of truth for impact shown in the notification (count) and the download panel (full list).

## 4. Notification mechanism

"A notification should be shown wherever the user is" → notifications are **persisted + pushed in real time + survive navigation/refresh**, decoupled from the modal.

**a. Notification store (backend)** — a `Notification` per user:
```
id, userId, type, status(UNREAD/READ),
payload(jobId, exportName, targetVersion, exclusionCount, ...), createdAt
```
Types: `EXPORT_READY`, `EXPORT_READY_WITH_EXCLUSIONS`, `EXPORT_FAILED`.

**b. Real-time delivery** — on worker completion (or cache hit) the backend writes the `Notification` row and pushes it over the existing real-time channel (WebSocket/STOMP/SSE), per-user topic.

**c. Fallback delivery** — if no live socket, the row is already persisted; the frontend fetches unread notifications on next load. *Push if connected, pull on reconnect — never lost.*

**d. Frontend notification center (unified feed)** — a global **bell** in the app shell (always mounted, independent of the export modal) with an unread badge; a **toast** for live arrivals.
- **Single list** (the export status and the notification are the *same* row — not two sections). Each item shows: read/unread indicator · status icon · title · source label · time · an inline **action**.
- **Statuses:** `Preparing…` (spinner) → `Export ready to download` / `Export ready — N excluded` / `Export failed`.
- **Action behaviour (key):**
  - **Clean export (`READY`)** → **Download** button downloads the ZIP **instantly** — no modal.
  - **With exclusions (`READY_WITH_EXCLUSIONS`)** → **Review & download** opens the impact modal (excluded-resources panel) first.
- **Extensible by type:** today every item is an export, but the feed is built to host other notification kinds later (e.g. licence-expiration) — not built yet.
- Toast action mirrors the same rule (Download vs. Review & download) and points at the same `jobId`.
- **Toast suppression (don't double-surface):** the right-side toast is suppressed when the user is already looking at the result —
  - the **notification center is open** (the feed updates live), or
  - the **"Preparing…" modal for that export is still open** — in which case that modal **switches straight to the download modal** (clean → "Download ZIP"; excluded → impact panel) instead of toasting.
- **Cross-view download readiness:** exports are linked by version `tag`. A ready export made in **Config** (version `1.6.2`) surfaces as a green **"Download ready"** affordance on the matching **Runtime → Builds** row (`Bizkids 1.6.2`), and vice-versa — the artifact is shared, so it's downloadable from either view.

Because the notification lives in the app shell, the user can close the modal, navigate, refresh, or return later and still be informed.

## 5. Frontend UX — the two steps

### Step 1 — Request export (existing Export Version modal, extended)
- User picks the **target platform version** (dropdown, current or lower) and the **Include media file content** toggle.
- If target platform version < current → inline hint under the dropdown: *"Some resources may be excluded. You'll see the full list before downloading."*
- Click **Export** → backend responds with either a **cache hit** (jump to Step 2, download available now) or a **new job**. On a new job the modal switches to a **"Preparing export… (up to 15 min)"** state with copy: *"You can close this and keep working — we'll notify you when it's ready."*
- Modal is **non-blocking and closeable**; the job runs server-side regardless.

### Step 2 — Download (from notification, or by reopening the modal)
- `READY` → "All resources included. Ready to download."
- `READY_WITH_EXCLUSIONS` → the **excluded-resources panel** (below), then download.
- On download → mark `DOWNLOADED`.

#### Excluded-resources panel

**Warning banner:**
> ⚠️ These resources don't exist on **{targetVersion}** and will be **excluded** from the downloaded ZIP. Download anyway, or cancel and choose a higher target version.

- Actions: **Download anyway** (primary) · **Cancel** → returns to the Step 1 target-version picker.

**List header:** `Excluded resources (N)`

**Each row:** breadcrumb path, muted, `\`-separated, with the **resource name bold** at the end, divider between rows:

`Integrations \ Data Sources \ **MailQueue**`

UI builds the breadcrumb from `path` + `name` in each `ExclusionReport` entry. (Row color coding intentionally **out of scope for now** — see Decision Log.) An optional **Download exclusion report** (CSV/JSON) can export the full list.

## 6. Impact clarity — surfaced in three escalating places

1. **Pre-export hint** (Step 1) — "may exclude resources" when target < source.
2. **Notification copy** — "Ready — N resources excluded."
3. **Pre-download panel** (Step 2) — full breadcrumb list + reasons + explicit "Download anyway" confirmation, with "choose a higher target version" as the remedy.

## 7. Edge cases

- **Concurrent identical requests** — cache key + job lock; the second request attaches to the in-flight job.
- **Failure / timeout** — `FAILED` notification with Retry; bound worker time.
- **Artifact expiry** — clicking a stale notification shows "Export expired — re-export," not a broken download.
- **Source changed after request** — content-revision in the cache key prevents reuse of a stale artifact.
- **Permissions** — re-check export rights at download time, not just request time.
- **Multiple queued exports** — notifications carry `jobId` + a human label (export name + target version) to stay distinguishable.

## 8. Delivery phases

1. **Async job backbone** — Export Job entity + state machine, worker, object storage + TTL, status endpoint. (Download works via reopening the modal.)
2. **Caching/dedup** — cache key + lookup → instant repeat downloads.
3. **Exclusion engine** — capability-manifest comparison + `ExclusionReport` + the Step 2 panel.
4. **Notifications** — store, real-time push, fallback pull, bell + toast, deep-link to download.
5. **Polish** — exclusion report export, retry-on-fail, expiry UX, analytics.

---

## 9. UX explorations — where the notification + download live

The open design question: **where do export status, the global notification, and the download action live?** Three explorations.

### Exploration A — Global notification bell (header) + download modal
- Add a **bell** to the dark header (near Config/Runtime or the avatar). Export runs in background; toast on completion + bell badge.
- Bell dropdown lists notifications (*"Export of main 1.6.2 → 5.5.x ready · 5 excluded"*). Click → **Download modal** with the excluded-resources panel.
- **Pros:** satisfies "notify wherever you are" directly; familiar, reusable for builds/commits/collab later; minimal change to the Branching console.
- **Cons:** brand-new global surface to build (+ backend notification store); download is one click "behind" the notification; no persistent export *history* unless the bell keeps it.

### Exploration B — Export Center (downloads tray/drawer)
- A dedicated **Exports** drawer (icon in header), like a browser downloads tray: every job listed with status/progress, target version, exclusion count, **Download** + re-download for cached artifacts. Impact panel expands inline per row. Toast + badge on completion.
- **Pros:** persistent, scannable **history** — strong fit since exports are cacheable & re-downloaded; one obvious home for downloading; centralizes the whole feature.
- **Cons:** a second panel concept beside the Branching console ("export where? download where?"); still needs the toast for global reach; heavy surface if only exports use it.

### Exploration C — In-console status + global toast (minimal)
- Status lives **inside the Branching console**: the exporting version shows a chip (Preparing → Ready/Excluded); Export button becomes **Download**. Global reach = a **toast** only; clicking reopens the console focused on that version's download.
- **Pros:** smallest footprint; reuses existing console; conceptually tight (export + download live with versions).
- **Cons:** toasts are ephemeral — miss it and there's no persistent "ready" indicator; weak for multiple concurrent exports or other notification types; must reopen console to act.

### Recommendation — **A + a slim history (blend of A & B)**
Lead with the **header bell** (best satisfies "wherever you are" + reusable), and give its dropdown/panel a small **"Exports" list** so cached re-downloads and past exports are discoverable. Defers B's full tray but keeps history. Build target for the prototype.

---

## Running the prototype

- Files: `index.html` (self-contained app), `server.js` (tiny Node static server, no deps).
- Start: `node server.js` from the project folder → open **http://localhost:8000**.
- Note: the folder name contains a `:`, which breaks raw `file://` URLs and Python's `http.server` in some sandboxes — serve over HTTP via `server.js` instead.
- Gotcha (fixed): Babel standalone's `react` preset defaults to the **automatic** JSX runtime (emits `import` → fails in a classic `<script>`). The page registers a `react-classic` preset (`runtime: 'classic'`) so JSX compiles to `React.createElement`.

## Hosting & iteration workflow

- **Repo:** `madalinagheorghiu-ux/export-version` (public).
- **Live prototype:** https://madalinagheorghiu-ux.github.io/export-version/ — served from the `gh-pages` branch.
- **Review PR:** https://github.com/madalinagheorghiu-ux/export-version/pull/1 (`prototype/export-notifications` → `main`).
- **Hosted spec:** https://madalinagheorghiu-ux.github.io/RandomDocs/export-download-notifications-spec.html
- **Publish model = publish-on-demand:** iterate on `prototype/export-notifications` (each push updates the PR for review); the live `gh-pages` URL only moves on an explicit **"publish"** (fast-forward `gh-pages` → latest prototype commit). Colleagues see a stable demo between publishes.

## Decision Log

| Date | Decision | Status |
|------|----------|--------|
| 2026-06-19 | Iteration workflow = **publish-on-demand**: work on `prototype/export-notifications` (pushes update PR #1); live `gh-pages` URL updates only on explicit "publish". | ✅ Decided |
| 2026-06-19 | **Same export experience reused in Runtime → Builds.** Config/Runtime toggle switches shells; build rows have an Export build icon that opens the same modal → notify → download flow, parameterized by source (build vs version). | ✅ Decided |
| 2026-06-19 | **Notification center = single unified feed** (merged the old "notifications" + "exports" sections). Each row shows status + read/unread + inline action. **Clean exports download instantly from the row; only excluded exports open the impact modal.** Feed designed to host other notification types later (licence expiry, etc. — not built). | ✅ Decided |
| 2026-06-19 | **Don't double-surface a ready export:** suppress the toast when the center is open; when the "Preparing…" modal is left open it switches straight to the download modal. | ✅ Decided |
| 2026-06-19 | **Cross-view download readiness:** a ready export links Config↔Runtime by version tag — surfaces as "Download ready" on the matching Runtime build row (and the shared artifact is downloadable from either view). | ✅ Decided |
| 2026-06-19 | Impact UX = inline grouped breadcrumb list + warning banner in the modal (per provided mockup). Path format `Category \ Subcategory \ **Name**`. | ✅ Decided |
| 2026-06-19 | Row **color coding** of excluded resources — deferred. NB: FlowX already uses green=added / yellow=modified / red=deleted in *Resources changed*; align with this when revisited. | ⏸️ Deferred |
| 2026-06-19 | Prototype is **isolated** from the FlowX codebase but mirrors the real Designer surfaces (header, Branching console, Export Version modal). | ✅ Decided |
| 2026-06-19 | Export launches from the existing **Branching console → Export Version** modal; missing post-export screens to be built. | ✅ Decided |
| 2026-06-19 | Notification + download surface = **Exploration A + slim history**: header notification bell (toast + badge), download modal with excluded-resources panel, and a small Exports list inside the bell panel for cached re-downloads. | ✅ Decided |
| 2026-06-19 | Prototype format = **interactive React app** (single self-contained `index.html`, React 18 + Babel via CDN, no build step), with a mocked backend (compressed delay, in-memory cache/dedup, simulated notifications). | ✅ Decided |

## Open questions

- **Notification infra** — reuse existing FlowX real-time/notification infrastructure vs. build a dedicated export notification path. (Recommend reuse; needs codebase confirmation.)
- **Job cancelability** — can an in-flight export be canceled from the modal/notification, or does it run to completion?
- **Color-bar semantics** — if/when color coding returns: severity (fully excluded vs. partially affected) vs. resource category vs. decorative; add a legend if meaningful.
- **Artifact TTL** — exact retention window (24h / 48h / 72h).
- **Exclusion report export** — include CSV/JSON download in v1 or later?
