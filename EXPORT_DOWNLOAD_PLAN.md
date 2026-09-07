# Export → Notify → Download — Implementation Plan

> **Purpose:** Living design doc for the version-settings export/download + notification mechanism.
> Keep this updated as decisions are made. See the **Decision Log** at the bottom.
>
> **Last updated:** 2026-07-07 (Summary two-card redesign · reactive Readiness filter · life-buoy nav icon) · [Live prototype](https://madalinagheorghiu-ux.github.io/export-version/) · [PR #1](https://github.com/madalinagheorghiu-ux/export-version/pull/1)

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
- **Runtime → Builds (second export entry point):** the header `Config / Runtime` toggle switches to a runtime shell whose nav is grouped into sections — **Deployment** (Builds) · **Runtime Configurations** (Active Policy · Configure Params Overrides · Scheduled Processes · Triggers) · **Monitoring** (Process Instances · UI Flows Sessions · Task Manager · Failed Process Start · Failed Triggers) · **Runtime Control** (Corrective Actions). The **Builds** page lists builds (e.g. `Bizkids 1.6.2`), each with an **Export build** icon. Exporting a build uses the **same** experience — target platform version + media toggle → processing → notification → download + excluded-resources panel. The only difference is the *source*: a build (`Bizkids 1.6.2`) instead of a committed version (`main 1.6.2`); labels, cache key, and the download manifest reflect it.
- **Runtime → Corrective Actions (migration entry point):** the **Corrective Actions** page (under Runtime Control in the nav) is the home for operations that fix or move running process instances. Currently surfaces **Bulk Migration** and **Move Token Bulk** as the two operation types. The operations table shows: Operation Type · Operation ID · Source Build · Target Build · Operation Status · Updated At · kebab `⋮` menu.

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
- **Context on every notification (multi-workspace):** users switch workspace + environment from the **logo menu** (env groups `SANDBOX` / `STAGING` / `PRODUCTION`, each with its workspaces). Every export is **stamped with the context it was launched in** — `{ environment, workspace, project }` — and each notification (and toast) shows an **env badge** (blue/amber/dark) + **workspace** + **project** (e.g. `[PRODUCTION] Silviu prod · bizkids`). Because the bell is global, this tells the user *where* a notification belongs even after they've switched context.
- **Item layout (de-crowded):** two rows — **context header on top** (env pill · workspace · project · time · dismiss ×) and the **notification below** (status icon · title · source · action). Unread = blue left-accent bar.
- **Environment filter:** chips at the top of the center — `All / Sandbox / Staging / Production` with live counts — filter the feed.
- **Dismiss:** per-item × removes a notification from the feed (artifact/cache untouched, so Builds "Download ready" persists).

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
> ⚠️ These resources don't exist on **{targetVersion}** and will be **excluded** from the downloaded ZIP. Download anyway, or change the target version.

- Actions: **Download anyway** (primary) · **Change Target Version** → returns to the Step 1 target-version picker.

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

## 9. Corrective Actions — Bulk Migration

### Overview

**Bulk Migration** moves running process instances from a source build to a target build. Like the export flow it is a long-running async operation (seconds to minutes in production) that notifies the user on completion and offers a result detail view.

### Migration Job states

```
IN_PROGRESS ──► COMPLETED
            └──► FAILED
```

| State | Meaning | User-facing |
|-------|---------|-------------|
| `IN_PROGRESS` | Worker migrating instances | spinner / progress bar |
| `COMPLETED` | All instances processed | ✅ "Bulk migration completed" |
| `FAILED` | Worker error | ❌ "Bulk migration failed" |

Each instance within a completed migration has its own outcome: **Success**, **Failed**, or **Terminated**.

### 9.1 Migration flow (step by step)

**Entry point:** Corrective Actions → `+` button → **Migrate Bulk**

1. **Setup modal (Step 1)**
   - Two build pickers: **Source Build** (where instances currently live) and **Target Build** (where they should move). Dropdowns list available builds (`3.7 – 3.10`); target excludes the selected source.
   - **Continue** advances to the summary; **Cancel** dismisses.

   #### Edge case: source build with no running instances

   Migration can only proceed when the source build has at least one instance in a running (active/incident) state. Two design options are under consideration:

   **Option A — Inline validation after selection**
   The user selects any build freely; if the chosen source has no running instances, an error/warning is shown below the dropdown and **Continue** is disabled.
   - ✅ Simple, uncluttered dropdown — no visual noise before the user has context.
   - ✅ Familiar form-validation pattern (feedback after input).
   - ✅ User can see all builds first, then understand the constraint.
   - ❌ Error-after-action: the user must undo a choice already made.
   - ❌ If many builds have no instances, the user may trial-and-error through several before finding a valid one.

   **Option B — Disabled dropdown items with hover tooltip** *(preferred)*
   Builds with no running instances are shown in the dropdown but grayed out and non-selectable; hovering reveals a tooltip explaining why (e.g. *"No running instances on this build"*).
   - ✅ Error-prevention over error-recovery (Nielsen heuristic #5) — invalid choices are never made.
   - ✅ Full eligibility picture at a glance: valid vs. ineligible builds visible together.
   - ✅ Tooltip educates the user inline — no need to read documentation.
   - ❌ Tooltip discoverability is lower on touch/mobile.
   - ❌ If *all* builds are ineligible, the dropdown alone is not enough — requires an additional callout above the field.
   - **Safeguard:** when every build is ineligible, show a callout above the picker (*"No builds currently have running instances. Migration requires at least one active or incident instance."*) so the user is never left with a fully-disabled dropdown and no explanation.

   > **Status:** ✅ Decided — **switched to Option A (inline validation) as shipped.** Option B (disabled items) was built first, then replaced: all builds are now freely selectable. The default source is the **first build (`3.7`)**, which has running instances. Selecting a build with no running instances (mock: `3.9`) shows an **inline error** — red border on the select, a red error icon **inside the field** (right of the chevron), and helper text *"No running process instances on this build."* **Continue is disabled** while an errored build is selected and re-enables on a valid one. Rationale for the change: all builds visible/selectable is simpler and matches the requested behaviour; error-prevention is preserved by gating Continue. The `(i)` icon next to "Source Build" keeps the styled tooltip *"Only active or incident instances can be migrated."*

2. **Summary modal (Step 2)** — "Migration plan:"
   - Shows the breakdown as **two collapsible grey cards**, each: a caret · **title + build-version pill** (e.g. *Migrate to `3.9.1`*) · right-aligned **`N processes`** count. Rows sit under a **left rail** (vertical accent line):
     - **Migrate to {targetBuild}** (default **expanded**) — the on-target processes; rows show **icon + name only**.
     - **Remain on {sourceBuild}** (default **collapsed**) — the not-migrated processes (left + terminated); rows show **icon + name + a right-aligned fate description** (*"…stay on the source build, untouched."* / *"…move to a TERMINATED state."*). Titled "Remain on" rather than "Leave on" because the group also contains terminated processes.
   - **Process names + counts come from the Configuration page** (`genMigSummary` derived from `MIG_CONFIG_PROCESSES`): on-target = migrate (3), not-found = not migrated (2).
   - **32px breathing room** between the title area (header), the body (cards), and the CTA row.
   - Warning banner: *"Once started, the migration cannot be stopped or canceled."*
   - Actions: **Back to Setup** (returns to Setup) · **Start Migration** (launches the job).

3. **Processing modal**
   - Mirrors the "Preparing your export…" pattern: animated progress bar, spinner, copy *"Migrating from {src} to {tgt}. This may take a few minutes. Close — keep working."*
   - On completion the modal **auto-transitions** to the Migration Detail Page (no manual action needed).
   - **Close — keep working** dismisses the modal early; the job continues; a toast fires on completion (unless the bell panel is already open).

4. **Notification + toast** (mirrors export pattern)
   - Migration appears in the unified bell feed alongside exports, sorted by timestamp.
   - Bell badge increments on completion; feed item shows "Bulk migration completed" + `{src} → {tgt}` + process count.
   - Toast: "Bulk migration completed · View results" — clicking opens the Migration Detail Page.
   - **Toast suppression:** no toast when (a) the bell panel is open, or (b) the processing modal for that migration is still open.
   - Each migration is stamped with `{ environment, workspace, project }` — shown in the feed and toast, same as exports.

5. **Migration Detail Page** (full-page, replaces the content area)
   - Opened by: clicking a completed live row in the Corrective Actions table, clicking "View results" in the toast, or clicking the bell feed item.
   - **Breadcrumb:** `← Operations / Migration {opId}…` — clicking "Operations" returns to the Corrective Actions list.
   - **Summary card:** "Migration ● Completed" heading + badge · Operation ID · Started timestamp · Run time · **SOURCE BUILD → TARGET BUILD** chips (with git-branch icon). No outcome data here — kept focused on identity/context.
   - **Instances card (three-zone layout):**
     - *Header:* "Instances" title · subtitle ("Per-instance result … retried individually.") · search box.
     - *Outcome zone:* sits between the header and the table — OUTCOME label · total count · segmented progress bar (green / red / amber) · inline clickable stats: Success N (X%) · Failed N (X%) · Terminated N (X%).
     - *Table:* column headers + rows + footer.
   - **Status filter:** clicking a stat chip in the Outcome zone filters the table to that status. Active chip gets a coloured background highlight and border. Click the same chip again to clear. No extra badge is shown in the "Instances" heading — the highlighted chip in the Outcome zone is the only filter indicator.
     - Columns: PROCESS INSTANCE UUID · PROCESS NAME · STATUS · DETAILS · MIGRATED AT.
     - Coloured status pills: **Success** (green) · **Failed** (red) · **Terminated** (amber).
     - Table scrolls horizontally at narrow viewports (`min-width: 680px`).
     - Footer: "Showing N of N instances" + prev/next pagination.
   - Mock data: 20 instances — 16 Success, 3 Failed, 1 Terminated — seeded deterministically.

6. **Migration Details modal (kebab `⋮`)** — compact alternative view
   - Clicking the `⋮` button on any **live completed row** in the Corrective Actions table opens `MigrationDetailModal` — a compact modal with the process-group breakdown (Migrated to / Terminated / Left on).
   - This is distinct from the full-page detail — the modal is a quick-glance summary; the full page has the per-instance table.
   - Kebab click stops row-click propagation (so it doesn't also navigate to the detail page).
   - Static / non-live rows (pre-seeded `INITIAL_OPS`) do not open any modal on kebab click in the current prototype.

### 9.2 Notification feed integration

Migrations share the unified bell panel feed with exports. The `notifStatus` helper branches on `item.type`:

| type | state | Feed title | Icon |
|------|-------|-----------|------|
| `migration` | `IN_PROGRESS` | Bulk migration in progress… | spinner |
| `migration` | `COMPLETED` | Bulk migration completed | ✅ |
| `migration` | `FAILED` | Bulk migration failed | ❌ |

Feed action on a completed migration: "View results" → opens Migration Detail Page (navigates to Runtime → Corrective Actions, sets `migDetailId`).

### 9.3 Prototype implementation notes

- `MIGRATION_PROCESS_MS = 8000` (8 s simulated, representing minutes in production).
- `migrations` state array in App; each entry: `{ id, type:'migration', opType, opId, srcBuild, tgtBuild, state, summary, ctx, read, dismissed, ts }`.
- `migDetailId` state in App controls full-page routing — checked **before** the Config/Runtime branch so the detail page renders regardless of current view.
- `migWatchRef` tracks whether the processing modal is still open (mirrors `watchRef` for exports) for toast suppression.
- `MOCK_INSTANCES` (20 rows, deterministically seeded at page load) is the shared fixture used by all Migration Detail Pages in the prototype.

### 9.4 Node mapping vs Move tokens — clarity redesign

**Product challenge.** On the Migration Configuration page the two controls inside a process card — *Node mapping between builds* and *Move tokens* — look like equal siblings, but they answer completely different questions and have opposite defaults. Users can't tell what each is for or which they must act on.

**The two concepts.**

| | **Node mapping** | **Move tokens** |
|---|---|---|
| Perspective | **Process / structural** (design-time) | **Runtime / intent** |
| Premise | The diagram changed between builds | The user *knows* tokens are blocked on a node |
| Unit | Node → node correspondence | **All tokens blocked on a chosen node** → target node |
| Token-aware? | **No** — mapped even if no tokens sit there | **Yes** — the whole point is stuck tokens |
| Default | Automatic → last visited node | None — opt in |
| Required? | **Conditional** — only when source and target diagrams differ; identical processes need no mapping | **No** (optional) |

In one line each:
- **Node mapping = recreate the process on the target so running instances keep a valid path** (structural, token-agnostic, mostly automatic — act only on unmatched nodes).
- **Move tokens = reposition tokens blocked on a node, by intent** (runtime, node-level bulk, always optional).

The recreated-node exception: if a configurator deletes a node by mistake and recreates it — even with the same name — the new node has a different **node ID**, so it will **not** map automatically. This is the main case where the user must intervene.

**Ordering argument — keep mapping first, re-weight instead of reorder.**
- **Why mapping stays first (dependency, not preference):** move tokens' "resume at node" target can only be a node that exists on the target build — that node-space is exactly what mapping establishes. Mapping is upstream. Unresolved mapping is also a *silent risk*: if a recreated node fell back to "last visited node" and the user reroutes tokens before noticing, they move tokens relative to a mislabeled node. **Correctness before intent.**
- **Why it feels like move tokens should win:** mapping is usually automatic (zero effort), while move tokens is the deliberate thing the user came to do. The fix is **prominence, not position** — the checkout pattern: "confirm address" precedes "place order," but the order button is the loud one.
- **Rejected alternative:** conditional ordering (move tokens first only when mapping is clean). A section that changes position by state is disorienting; stable order + prominence gets the same benefit without the instability.

**Shipped design.** The concepts and ordering rationale above hold; the visual model was simplified during implementation (the framing line, the "What's the difference?" expander, and the "Review/Show auto-mapped" toggles were all removed as clutter). Each process card is expandable and shows two sections in dependency order — **Map nodes** first, then **Set tokens destination** — plus the not-found variant.

**Map nodes** (shown for every process that exists on the target):

| Process state | What shows |
|---|---|
| **Ready** (identical or auto-mapped) | Just the line *"The process is identical on both builds — nothing to map."* No table, no toggles. |
| **Needs attention** (≥1 unmatched node) | Amber chip *"N nodes need your attention"*; a **warning hint under the title** — *"These nodes no longer exist on {target}. If a node was recreated, its ID changed so it can't match automatically — pick the new node, or it falls back to the last visited node."*; then `Current node [src] → New node [tgt]` rows. |

- The **New node** column is a **functional dropdown**, **empty by default** (`Select new node`); its **first option is `last visited node`** (the fallback), followed by target nodes.
- Column headers carry build tags (`Current node ⌥ {src}` / `New node ⌥ {tgt}`) in the same small style used by Set tokens destination.
- **Self-resolving status:** once every unmatched node has a New node selected, the chip flips to green *"All nodes mapped"*, the hint disappears, the process's **status pill flips to Ready**, and the **Readiness card counts + bar update live**.
- **Readiness filter reacts to live counts:** a Readiness chip whose count is **0** is **disabled** (dashed, greyed, non-clickable — it would only ever show an empty list). And if the user resolves the **last** process in the currently-filtered bucket (e.g. maps the last "Need node mapping" process), that filter **auto-clears** so the just-resolved card stays visible under the full list instead of vanishing into a "no processes match" empty state.

**Set tokens destination (post-migration)** (shown for every on-target process; no "Optional" chip, no info tooltip):
- Subtitle: *"If tokens are blocked on a node, move all tokens waiting on it to another node on {target} — forward or back — based on your fix."*
- **No row by default.** `+ Move Token` (right-aligned in the section title row) adds a row.
- Rows use **functional node dropdowns**; a **single header row** shows `Blocked on node [src] → Resume at node [tgt]` with build tags (labels are **not** repeated per row); each row has a remove (trash) control.
- Helper under the rows: *"All tokens currently on the selected node will be moved."*

**Not found on target** (process missing on the target build): a **ban icon** on the status pill, the line *"This process doesn't exist on target build {target}, so its running instances can't be migrated…"*, and a radio choice — **Terminate all running instances** (preselected) / **Leave on current build**.

**Layout.** The card title + Readiness filter zone are pinned at the top and the footer CTA (Cancel · Check Operation Summary) is pinned at the bottom; the **process list scrolls inside** its own region (the page itself does not scroll).

**Prototype implementation notes.**
- `MIG_CONFIG_PROCESSES`: each process has `status` (`ready` / `mapping` / `notfound`), `mappingState` (`unchanged` / `auto` / `attention`), `nodeMap: [{ from, fromIcon, matched }]`, and node-level `moveTokens: [{ blockedNode, resumeNode }]`.
- `MAP_TARGET_NODES` = `['last visited node', …]` (fallback first); `MOVE_TOKEN_NODES` = node pool for the token dropdowns.
- Mapping selections are lifted to the page (`mapSel` keyed by process + node index); `effStatus(p)` returns `ready` once all unmatched nodes are selected, driving both the pill and the Readiness counts.

**Out of scope (mocked).** Real diagram-diff detection, actual node IDs / token counts, and backend behaviour are simulated; the prototype demonstrates the *interaction and framing*, not live mapping logic.

---

## 10. UX explorations — where the notification + download live

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

> **Note:** The bell panel now also surfaces **migration** notifications alongside export notifications in the same unified feed — proving the "extensible by type" promise from Exploration A.

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

## Ideas to make the center more useful (future features)

**Shipped so far:** unified feed · per-notification env/workspace/project context · two-row layout · environment filter · dismiss · instant-vs-impact download · context shown in the download modal · cross-view "Download ready" · **Bulk Migration flow** (setup → summary → processing → completion) · **migration notifications in the unified bell feed** · **Migration Detail Page** (full-page: summary card + outcome bar + per-instance table) · **status filter on instances** (click Success / Failed / Terminated chip to filter) · **Migration Details modal via kebab** (compact process-group summary, distinct from full-page).

**Next candidates (rough priority):**
1. **Type filter / tabs** — once there are more kinds (exports, build status, licence expiry, errors): `All / Downloads / System / Alerts`.
2. **Deep-link "Go to"** — jump to the build/version/project a notification refers to (auto-switch workspace/env context).
3. **Expiry indicator + Retry** — show when a download artifact expires (ties to artifact TTL); "Export expired — re-export"; Retry on `FAILED`.
4. **Bulk actions** — "Download all ready", "Clear read", "Clear all", multi-select.
5. **Grouping** — by time (Today / Earlier) or by workspace, once volume grows.
6. **Unread-only toggle** + search.
7. **Priority / severity** — errors and licence-expiry styled distinctly, optionally pinned to top.
8. **Persistence** — survive reload/login (backend-backed); **"See all"** full-page activity view.
9. **Preferences** — mute types, do-not-disturb; optional email/desktop for long-running exports.

## Decision Log

| Date | Decision | Status |
|------|----------|--------|
| 2026-06-19 | Iteration workflow = **publish-on-demand**: work on `prototype/export-notifications` (pushes update PR #1); live `gh-pages` URL updates only on explicit "publish". | ✅ Decided |
| 2026-06-19 | **Same export experience reused in Runtime → Builds.** Config/Runtime toggle switches shells; build rows have an Export build icon that opens the same modal → notify → download flow, parameterized by source (build vs version). | ✅ Decided |
| 2026-06-19 | **Notification center = single unified feed** (merged the old "notifications" + "exports" sections). Each row shows status + read/unread + inline action. **Clean exports download instantly from the row; only excluded exports open the impact modal.** Feed designed to host other notification types later (licence expiry, etc. — not built). | ✅ Decided |
| 2026-06-19 | **Don't double-surface a ready export:** suppress the toast when the center is open; when the "Preparing…" modal is left open it switches straight to the download modal. | ✅ Decided |
| 2026-06-19 | **Cross-view download readiness:** a ready export links Config↔Runtime by version tag — surfaces as "Download ready" on the matching Runtime build row (and the shared artifact is downloadable from either view). | ✅ Decided |
| 2026-06-19 | **Notifications carry workspace context.** Logo menu switches workspace+environment (SANDBOX/STAGING/PRODUCTION); each export is stamped with `{environment, workspace, project}` and every notification/toast shows env badge + workspace + project. | ✅ Decided |
| 2026-06-19 | **Notification item redesign:** context header on top (env pill · workspace · project · time · dismiss), notification below; unread = left accent bar. Added **environment filter** chips with counts and per-item **dismiss**. | ✅ Decided |
| 2026-06-19 | Impact UX = inline grouped breadcrumb list + warning banner in the modal (per provided mockup). Path format `Category \ Subcategory \ **Name**`. | ✅ Decided |
| 2026-06-19 | Row **color coding** of excluded resources — deferred. NB: FlowX already uses green=added / yellow=modified / red=deleted in *Resources changed*; align with this when revisited. | ⏸️ Deferred |
| 2026-06-19 | Prototype is **isolated** from the FlowX codebase but mirrors the real Designer surfaces (header, Branching console, Export Version modal). | ✅ Decided |
| 2026-06-19 | Export launches from the existing **Branching console → Export Version** modal; missing post-export screens to be built. | ✅ Decided |
| 2026-06-19 | Notification + download surface = **Exploration A + slim history**: header notification bell (toast + badge), download modal with excluded-resources panel, and a small Exports list inside the bell panel for cached re-downloads. | ✅ Decided |
| 2026-06-19 | Prototype format = **interactive React app** (single self-contained `index.html`, React 18 + Babel via CDN, no build step), with a mocked backend (compressed delay, in-memory cache/dedup, simulated notifications). | ✅ Decided |
| 2026-06-30 | **Bulk Migration added to Corrective Actions.** 4-step flow: Setup modal (source + target build) → Summary modal (Migrate to / Terminate / Leave on breakdown, collapsible) → Processing modal (auto-transitions to detail page on completion) → Migration Detail Page. Same toast-suppression and bell-feed rules as exports. | ✅ Decided |
| 2026-06-30 | **Migration Detail Page = full-page view** (not a modal). Replaces the content area via `migDetailId` state, checked before the Config/Runtime routing branch. Breadcrumb `← Operations / Migration {id}…` returns to the Corrective Actions list. | ✅ Decided |
| 2026-06-30 | **Dual access pattern for completed migrations:** row click → full Migration Detail Page (summary + per-instance table); kebab `⋮` → compact Migration Details modal (process-group summary only). Both are live-migration-only; static seed rows don't open anything on kebab. | ✅ Decided |
| 2026-06-30 | **Outcome zone lives inside the Instances card**, between the card header and the table — not in the summary card. Summary card is identity-only (title, meta, builds). Instances card has three zones: header · outcome · table. | ✅ Decided |
| 2026-06-30 | **Status filter on instances table.** Clicking Success / Failed / Terminated in the Outcome zone filters the table to that status. Active chip gets coloured background highlight. Click again to clear. No badge shown in the "Instances" heading — the highlighted chip is the only filter indicator. Resets page to 1. | ✅ Decided |
| 2026-06-30 | **Migrations surface in the unified bell feed** alongside exports — proving the "extensible by type" design. Feed item action "View results" → navigates to Runtime → Corrective Actions + opens Migration Detail Page. | ✅ Decided |
| 2026-06-30 | **Source build with no running instances (Setup modal) — Option B chosen.** Builds with no running instances are shown in the Source Build dropdown but grayed out and non-selectable; hovering shows a styled tooltip ("No running instances on this build") + always-visible "No instances" inline tag. The `(i)` icon next to "Source Build" uses the same styled tooltip ("Only active or incident instances can be migrated"). Safeguard: if all builds are empty, an orange callout replaces per-item tooltips as the primary signal. Continue is disabled while an ineligible source is selected. | ⛔ Superseded (see 2026-07-03) |
| 2026-07-03 | **Source build validation switched to Option A.** All builds selectable; default = first build (`3.7`, has instances). Selecting a no-instance build (`3.9`) shows an inline error: red border + error icon **inside the field** + helper "No running process instances on this build". **Continue is disabled** until a valid build is chosen. Replaces the Option B disabled-items approach. | ✅ Decided |
| 2026-07-03 | **Node mapping "Map nodes" is state-driven and self-resolving.** Ready processes show *"nothing to map"*; attention processes show unmatched-node rows with a warning hint under the title + **New-node dropdowns** (first option "last visited node", empty by default). "Review/Show auto-mapped" toggles were removed. **Mapping every unmatched node flips the process to Ready and updates the Readiness card counts/bar live.** | ✅ Decided |
| 2026-07-03 | **Move tokens → "Set tokens destination (post-migration)".** Functional node dropdowns, build tags after each label, one header row (no repeated labels), no default row, right-aligned "+ Move Token". "Not found on target" uses a ban icon. | ✅ Decided |
| 2026-07-03 | **Migration Configuration layout:** card title + Readiness filter + footer CTA are pinned; the process list scrolls inside its own region (page itself doesn't scroll). | ✅ Decided |
| 2026-07-07 | **Readiness filter reacts to live counts.** A chip with count 0 is disabled (dashed/greyed, non-clickable). If the active filter's bucket empties (last process in it just resolved), the filter auto-clears so the resolved card stays visible instead of dropping into an empty state. | ✅ Decided |
| 2026-07-07 | **Bulk Migration Summary redesign → two-card layout (final).** Grey collapsible cards: caret · title + build-version pill · right-aligned `N processes`; rows under a left rail. **Migrate to {tgt}** (expanded, icon+name only) and **Remain on {src}** (collapsed, icon+name+fate description grouping the left + terminated processes — "Remain on" not "Leave on" since it includes terminated processes). Names/counts sourced from the Configuration page (`genMigSummary` ← `MIG_CONFIG_PROCESSES`). 32px between title/body/CTA. Warning → *"Once started, the migration cannot be stopped or canceled."* Supersedes the three-card badge version explored earlier the same day. | ✅ Decided |
| 2026-07-07 | **Life-buoy icon for Runtime Control** nav section — replaces the ambiguous refresh-loop icon (read as retry/sync) with a recover/rescue metaphor. Registered in `SECTION_ICONS`; the old `control` icon stays on the Migrate Bulk dropdown and Start Migration button. | ✅ Decided |

## Open questions

- **Notification infra** — reuse existing FlowX real-time/notification infrastructure vs. build a dedicated export notification path. (Recommend reuse; needs codebase confirmation.)
- **Job cancelability** — can an in-flight export be canceled from the modal/notification, or does it run to completion?
- **Color-bar semantics** — if/when color coding returns: severity (fully excluded vs. partially affected) vs. resource category vs. decorative; add a legend if meaningful.
- **Artifact TTL** — exact retention window (24h / 48h / 72h).
- **Exclusion report export** — include CSV/JSON download in v1 or later?
- ~~**Bulk Migration — source build with no running instances**~~ — resolved. Shipped with **Option A** (all builds selectable + inline error + Continue gated); Option B was built first then replaced. See §9.1 and Decision Log.
