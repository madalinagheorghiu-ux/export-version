# Notification Center & Long-Running-Process Notifications — Spec

> **Scope:** The notification *mechanism* and *notification center* for long-running,
> asynchronous operations in the FlowX Designer — the surface that tells a user
> "your thing is ready / done / failed" **wherever they are** in the app.
>
> This is the reusable pattern extracted from the **Export → Notify → Download** work.
> Two operations already ride on it — **Export version/build** and **Bulk Migration** —
> and it is deliberately built to host more (licence expiry, build status, errors, …).
>
> **Companion doc:** `EXPORT_DOWNLOAD_PLAN.md` (the full export + migration design; this
> file is the notification slice of it, generalized). · **Last updated:** 2026-08-03

---

## 1. Problem & goals

Some operations are **long-running** (an export may take up to ~15 min; a bulk migration
runs seconds-to-minutes). The user must be free to **close the launching modal, navigate,
refresh, or leave and come back** — and still be told, reliably, when the operation changes
state. A toast alone is not enough: it's ephemeral and easily missed.

**Goals**

1. **Reach the user wherever they are** — the indicator lives in the always-mounted app shell, not in the operation's modal.
2. **Never lose a notification** — persisted server-side, pushed in real time, pulled on reconnect.
3. **Actionable** — each notification carries the next step (download / view results / retry), not just a status.
4. **Extensible by type** — one feed, many operation kinds, without a redesign per kind.
5. **Unambiguous context** — in a multi-workspace / multi-environment product, every notification says *where* it belongs.

---

## 2. Core model — an async job whose state changes are the notifications

Any operation that uses this mechanism is modelled as a **server-side job with a lifecycle**.
The **notification is the messenger** that announces a state transition; the follow-up
**action** (download, view, retry) is a separate interaction against the job's result.

Generic lifecycle:

```
REQUESTED ──► IN_PROGRESS ──► DONE
                    │      └──► DONE_WITH_CAVEAT   (succeeded, but the user must see something first)
                    └────────► FAILED
DONE / DONE_WITH_CAVEAT ──► ACTED_ON ──► EXPIRED
```

Mapped to the two shipped operations:

| Generic state | Export | Bulk Migration |
|---|---|---|
| `IN_PROGRESS` | `PROCESSING` — "Preparing your export…" | `IN_PROGRESS` — "Migrating…" |
| `DONE` | `READY` — download, all resources included | `COMPLETED` — all instances processed |
| `DONE_WITH_CAVEAT` | `READY_WITH_EXCLUSIONS` — N resources excluded, review before download | *(n/a today; per-instance outcomes shown on the detail page)* |
| `FAILED` | `FAILED` — export failed, retry | `FAILED` — migration failed, retry |
| `ACTED_ON` | `DOWNLOADED` | results viewed |
| `EXPIRED` | artifact TTL passed → re-export | — |

**Key idea:** the notification fires on every user-relevant transition. A `DONE_WITH_CAVEAT`
state exists so the feed can route the user through a review step (e.g. the excluded-resources
panel) instead of a one-click finish.

---

## 3. Backend — notification store

A persisted `Notification` row **per user**, decoupled from the launching modal:

```
Notification {
  id,
  userId,
  type,               // EXPORT_READY | EXPORT_READY_WITH_EXCLUSIONS | EXPORT_FAILED
                      // | MIGRATION_COMPLETED | MIGRATION_FAILED | … (extensible)
  status,             // UNREAD | READ
  payload,            // { jobId, label, context{environment, workspace, project}, … type-specific fields }
  createdAt
}
```

- `type` is an open enum — new operation kinds add new types without schema change.
- `payload.jobId` links the notification to its job/artifact so the action can be re-derived at click time (permissions and expiry re-checked then, not at creation).
- `payload.context` stamps the launching `{ environment, workspace, project }` (see §6).

---

## 4. Delivery — push if connected, pull on reconnect (never lost)

1. **Real-time push.** On job completion (or an instant cache hit), the backend writes the
   `Notification` row **and** pushes it over the existing real-time channel
   (WebSocket / STOMP / SSE), on a **per-user topic**.
2. **Fallback pull.** If there's no live socket, the row is already persisted; the frontend
   **fetches unread notifications on next load / reconnect**.

> The row is written **before** (or atomically with) the push, so a dropped socket never
> loses a notification — the client will pull it. *Push if connected, pull on reconnect.*

**Reuse question (open):** reuse FlowX's existing real-time / notification infrastructure vs.
a dedicated path. Recommendation: **reuse** — pending codebase confirmation.

---

## 5. Frontend — the notification center

### 5.1 Where it lives
A global **bell** in the dark app-shell header (near `Config / Runtime` / the avatar) —
**always mounted, independent of any operation modal** — with an **unread badge**. Live
arrivals also raise a **toast**. Because the bell is global, closing a modal / navigating /
refreshing never hides the status.

**Panel width — 420px** (min 380 / max 440; scrolls internally, capped ~480px tall). Benchmarked
against contextual notification panels (Jira ~400, Notion ~420–460, Slack ~400, GitHub ~370 + full
page). Below ~360 the two-row item truncates; above ~480 a bell-anchored dropdown reads like it
wants to be a page. Since FlowX has **no dedicated notifications page** the panel is the primary
surface (lean comfortable), but per-item detail lives in the operation **modals** (so it needn't grow
into a page) — 420 is the balance. Toasts stay ~360 (transient, single-action).

### 5.2 Unified feed (one list, many types)
The center is a **single list**, not one section per operation kind. The operation's *status*
and its *notification* are the **same row**. A `notifStatus`-style helper branches on
`item.type` to render the right title/icon/action:

| type | state | Feed title | Icon |
|---|---|---|---|
| `export` | preparing | Preparing export… | spinner |
| `export` | ready | Export ready to download | ✅ |
| `export` | ready w/ exclusions | Export ready — N excluded | ⚠️ |
| `export` | failed | Export failed | ❌ |
| `migration` | in progress | Bulk migration in progress… | spinner |
| `migration` | completed | Bulk migration completed | ✅ |
| `migration` | failed | Bulk migration failed | ❌ |
| `org` | bulk import done | Bulk Import End-Users | ✅ |
| `org` | licence | Licence expires in N days | ⚠️ |

> Every terminal item's status icon resolves to exactly one of **success / warning / failed** (in-progress
> shows a spinner). Org kinds are no exception — no bespoke per-kind icons.

New kinds (licence expiry, build status, errors) slot in as new `type` branches — **the
"extensible by type" promise**. Bulk Migration was the first proof that a non-export kind
shares the feed cleanly; **org-level** notifications (see §5.9) are the second — they proved the
feed can hold items that belong to no environment at all.

### 5.3 Item layout (de-crowded, two rows)
Each feed item is **two rows**:
- **Context header (top):** env pill (or org area — §5.9) · workspace · project · time. A **mark-as-read
  ✓** appears on hover in a fixed slot **before the time**, so timestamps stay column-aligned across
  read/unread rows alike.
- **Notification (below):** status icon · title · source label · inline **action**
- **Status icon** is deliberately **discrete** — a small **colour-coded glyph** (success / warning /
  failed), *not* a filled tile — so a column of them stays calm rather than reading as a traffic light.
- **Unread indicator:** a blue **left-accent bar**.

### 5.4 Inline action (the payoff)
Every item ends in an action that points at the same `jobId`:
- **Clean result** → a direct action, **no modal** (export `READY` → **Download** the ZIP instantly).
- **Result with a caveat** → opens a **review step first** (export `READY_WITH_EXCLUSIONS` → **Review & download** → excluded-resources panel).
- **Completed migration** → **View results** → the Migration Detail Page.
- **Failed** → **Retry**.

**Action styling.** Feed actions are **secondary buttons with blue text** — consistent, low-weight.
An **icon appears only when the click downloads immediately** (`Download` → ⬇). Anything that opens
a step first (`Review`, `View results`, `Retry`) is text-only, so the icon reliably signals
"one click = file in hand." The caveat on `READY_WITH_EXCLUSIONS` is carried by the item's amber
warning icon + "N excluded" title, not by the button.

**Failed state.** A `FAILED` item shows **no status chip** — the red error icon + title already say
it, and a "Failed" pill next to a red icon is redundant. Its action slot stays empty until Retry
(§8, a future candidate) lands there.

**Navigating from a notification never discards work silently.** Acting on a notification that
routes to a page (a migration's *View results*, an export's *Review & download*) can fire while a
modal is open (a toast sits above the scrim) or from a config page (the bell is reachable there).
If the user is **mid-setup** — the migration setup/config, or an export setup — a **"Discard
unsaved changes?"** guard appears first: **Keep working** stays put and leaves the notification
unread; **Discard & leave** tears the flow down and routes. A *running* operation's progress modal
is **not** treated as dirty (it keeps going in the background, so leaving loses nothing), and a
plain download doesn't navigate at all — so neither prompts.

The guard **never stacks a modal on a modal.** When the dirty flow is itself a modal (the migration
setup, reachable via a toast that floats above the scrim), the confirm doesn't open a *second*
scrim on top — the underlying modal is hidden while the confirm is shown (its state is preserved),
so exactly one surface and one focus target exist at a time. *Keep working* restores the modal;
*Discard & leave* tears it down. Over a config *page* it's a single scrim over the page anyway.
(Stacked modals are avoided on purpose: double-dimming + competing focus traps / Esc handling.)

### 5.5 Toast (live arrival)
A live toast is **the same object as a feed item, just shorter-lived** — it reuses the exact
two-row structure (context row + status row), the discrete status glyph, the same colours, and the
same action rule (⬇ **Download** for a clean result, text-only **Review** / **View** otherwise). It
carries only what's needed (env context, status, title, one-line subtitle, one action) — a toast
showing fewer fields than the feed is expected. A status-matched left accent (green / amber / red)
is the only toast-specific chrome. It points at the same `jobId`.

**Toast suppression — don't double-surface.** The toast is suppressed when the user is
already looking at the result:
- the **notification center is open** (the feed updates live), **or**
- the **launching "Preparing…/In progress" modal for that job is still open** — in which case
  that modal **switches straight to the result** (clean → download; caveat → review panel;
  migration → detail page) instead of toasting.

### 5.6 Filter — read-state + scope (one control)
Filtering is an **occasional** control, so it collapses to a **filter icon next to search** rather
than a persistent row. The icon opens a single-select popover that folds *read-state* and *scope*
into one list:

```
All notifications
Unread (N)                    ← read-state
─────────────
Organization                 ← org-level items (no environment)
─────────────
ENVIRONMENT TYPE
  Production / Staging / Sandbox
```

**One active filter at a time** — *Unread* shows unread across every scope; a scope shows that
scope's items (read + unread). *(Trade-off: the two can't currently be intersected, e.g. "unread
Production" — chosen for simplicity; it replaced the earlier standalone All/Unread toggle, now
removed.)* Env types carry a colour-coded dot; *All / Unread / Organization* use a neutral dot.
Because it's **single-select**, there's **no active-filter chip** — the highlighted **filter icon**
alone signals a filter is on, and re-opening the popover to pick *All notifications* clears it. (A
chip would just duplicate what the one selected row in the popover already shows.) (Evolved: four
counted chips → grouped scope dropdown → icon + chip → icon only, with read-state folded in.)

### 5.7 Find & focus — search, all/unread, time grouping, read controls
As the feed grows, these affordances keep it navigable; the filters **compose** (search ∩ scope ∩
all/unread all narrow the same list):
- **Search** — a text box matching each item's title, source label (`Bizkids 1.6.1 → 5.9.X`),
  and stamped context (scope / workspace / project / org).
- **Unread view** — now a **filter option** (§5.6), not a standalone toggle; its count lives on that
  option and is global. (Per-env chip counts and a running "N shown" total were removed as noise.)
- **Mark one as read** — a ✓ affordance revealed **on item hover** for any unread row, so a single
  item can be cleared without acting on it or touching the others.
- **Mark all as read** — a text link in the **panel header** (top-right). Driven by *global* unread
  (it clears everything), and **disabled (greyed), not hidden,** when nothing is unread — so its
  position is stable and its state reads as "nothing to do" rather than vanishing.
- **Time grouping** — the (time-sorted) feed is bucketed under **Today / This week / Older**
  day headers, so recent arrivals stay at the top and older results fall away visually.
- **Retention** — the center keeps the **last 30 days**; a persistent footer states this so the
  absence of older items reads as policy, not loss.

### 5.8 Dismiss / archive (deferred)
The per-item **×** was **removed for now** — dismiss/archive is deferred until there's a real
**archive** destination (a dismissed item shouldn't just vanish with nowhere to go). When it lands,
it will live as a second **on-hover** action alongside mark-as-read (§5.7), and must leave the
**artifact/cache/result untouched** — e.g. an archived export still shows "Download ready" on its
Builds row. Until then the feed's only reduction is read/unread + the 30-day window.

### 5.9 Notification scope — organization vs environment type
An organization spans **three environment types** — `PRODUCTION` / `STAGING` / `SANDBOX` — plus the
org itself. Every notification therefore has one of two **scopes**:

- **Environment-type scope** — the work-in-an-environment kinds (export, migration). Stamped with
  `ctx = { env, workspace, project }`; shown with the env badge. These appear under *All* and under
  their env-type filter, never under a *different* env type.
- **Organization scope** — org-wide events that belong to no single environment (e.g. **Bulk Import
  End-Users**, **licence expiry**, billing/plan). Stamped with `ctx = { area, target }` and shown as
  a plain **`{area} · {target}`** context line — e.g. `Access Management · End-Users`, `Billing ·
  Subscription` — parallel to the env items' `{workspace} · {project}` but with **no leading marker
  and no folder icon** (there's no dedicated org colour, and the *absence* of an env badge is itself
  the "not environment-specific" signal). The status is carried entirely by the discrete
  success/warning/failed glyph (§5.3), same as every other kind. These appear under *All* and under
  *Organization* only.

`itemScope(item)` returns `ORG` for org items and the `ctx.env` otherwise; the scope filter (§5.6)
matches on it. This is the model behind "filter by env type, org, all."

### 5.10 Notification titles — naming spec (for dev)

**Convention.** Title = `{subject} {outcome}` — the *subject stays identical across all statuses of a
type*; only the outcome changes. Counts / versions / names go in the **subtitle**, never the title —
except the **warning count** (that's the point of the warning). Two success verbs by category:
file-producing → *"ready to download"*; change-applying → past participle (*"imported" / "migrated"*).

| # | Type | In progress | Success | Warning | Failed |
|---|---|---|---|---|---|
| 1 | Export Build | Preparing build export… | Build export ready to download | Build export ready — {N} excluded | Build export failed |
| 2 | Export Project Version | Preparing version export… | Version export ready to download | Version export ready — {N} excluded | Version export failed |
| 3 | Import Build | Importing build… | Build imported | Build import needs review — {N} conflicts | Build import failed |
| 4 | Import Project Version | Importing version… | Version imported | Version import needs review — {N} conflicts | Version import failed |
| 5 | Bulk Import End-Users | Importing end-users… | End-users imported | End-users imported — {N} skipped | End-user import failed |
| 6 | Migrate Instances | Migrating instances… | Instances migrated | Instances migrated — {N} failed | Instance migration failed |
| 7 | Download KB Data Source | Preparing KB data source… | KB data source ready to download | KB data source ready — {N} skipped | KB data source download failed |

> **`Bulk Import End-Users` is the operation *type*, not the title.** Its success title is
> **"End-users imported"** (title = subject + outcome, like every other type).

**Warning has two flavours** (same amber, different CTA):
- **W-a · needs review / accept impact** (not finished; proceeds only if the user accepts) → types 1, 2, 3, 4, 7 · **CTA = Review**.
- **W-b · completed with warnings** (finished, with caveats) → types 5, 6 · **CTA = View**.

**Subtitle + CTA per status:**

| Status | Subtitle | CTA |
|---|---|---|
| In progress | what → where | — |
| Success (file) | `{source} → {target}` | **Download** (⬇) |
| Success (applied) | `{source} → {target} · {N} items` | **View** |
| Warning W-a | same as success | **Review** |
| Warning W-b | `{source} → {target} · {N} affected` | **View** |
| Failed | short cause; "nothing was applied" if atomic (§8) | **Retry** |

*(Open: do imports surface a pre-apply review (W-a) or report after auto-apply (W-b)? Flips types 3–4 — see §12.)*

---

## 6. Context on every notification (multi-workspace / multi-environment)

Users switch **workspace + environment** from the logo menu (env groups
`SANDBOX` / `STAGING` / `PRODUCTION`, each with its workspaces). Because the bell is global, a
notification could otherwise be ambiguous about *where* it belongs.

So **every job is stamped at launch with `{ environment, workspace, project }`**, and that
context is shown on:
- the **feed item** (context header),
- the **toast**,
- and the **result modal / detail page** it opens.

The **env badge is colour-coded**: blue `SANDBOX` · amber `STAGING` · dark `PRODUCTION`
(e.g. `[PRODUCTION] Silviu prod · bizkids`).

---

## 7. Cross-surface readiness

Results are linked across views by a stable key (the version `tag`). A result produced in one
surface surfaces in the other:
- An export made in **Config** (version `1.6.2`) shows a green **"Download ready"** affordance
  on the matching **Runtime → Builds** row (`Bizkids 1.6.2`), and vice-versa.
- The artifact is **shared**, so it's downloadable from either surface.

The notification is the push; the cross-surface affordance is the persistent, in-context pull.

---

## 8. Failure handling

A `FAILED` transition raises a notification like any other. For a **system-outage** failure the
copy should: name the failure, **reassure that nothing was applied** (the main anxiety with a
mid-run failure), and offer **Retry**. Keep the wording **identical across toast, table row,
and detail page** so it reads as one error, not three.

Example (bulk migration, service unavailable):

> **Bulk migration failed** — the migration service is temporarily unavailable, so the
> operation couldn't run. Your process instances weren't changed. **Retry** once it's back.

> ⚠️ Only claim "nothing was changed" if the operation is transactional/atomic. If a partial
> effect is possible, soften to *"Some items may not have been processed — open details to
> review before retrying."*

---

## 9. Edge cases (notification-relevant)

- **Concurrent identical requests** — cache key + job lock; the second request attaches to the in-flight job (one notification, not two).
- **Failure / timeout** — `FAILED` notification with **Retry**; bound worker time.
- **Result expiry** — clicking a stale notification shows "expired — re-run," not a broken action.
- **Source changed after request** — content-revision in the cache key prevents reuse of a stale artifact (a fresh run → a fresh notification).
- **Permissions** — re-checked at **action time** (download/view), not just at request time.
- **Multiple queued operations** — notifications carry `jobId` + a human label (e.g. export name + target version) to stay distinguishable.

---

## 10. Why a header bell + slim history (the chosen surface)

Three surfaces were explored for "where do status, the global notification, and the follow-up
action live":

- **A — Header notification bell + result modal.** Satisfies "notify wherever you are"; familiar; reusable for many kinds. *Chosen.*
- **B — dedicated Export/Activity Center (downloads tray).** Great persistent history, but a second panel concept and heavy if under-used.
- **C — in-console status + global toast only.** Smallest footprint, but toasts are ephemeral and weak for multiple/other kinds.

**Decision:** **A + a slim history** — lead with the header bell (best "wherever you are" +
reusable), and give its panel a small list so past/cached results are discoverable. This is
the unified feed above; it now hosts both exports and migrations.

---

## 11. Future candidates

1. **Type filter / tabs** — once there are more kinds: `All / Downloads / System / Alerts`. (Partly realized: the scope filter §5.6 already separates org-wide from env-type items.)
2. **Deep-link "Go to"** — jump to the build/version/project a notification refers to (auto-switch workspace/env context).
3. **Expiry indicator + Retry** — show artifact TTL; "expired — re-run"; Retry on `FAILED`. (Distinct from the 30-day *notification* retention in §5.7 — this is the *artifact* TTL.)
4. **Bulk actions** — ✅ *shipped: "Mark all read" + per-item mark-as-read on hover (§5.7).* Still open: "Clear read", "Clear all", multi-select.
5. **Grouping** — ✅ *shipped: by time (Today / This week / Older) — see §5.7.* Still open: grouping by workspace.
6. ~~**Unread-only toggle** + search.~~ ✅ *shipped — All/Unread segment + search, see §5.7.*
7. **Priority / severity** — errors and licence-expiry styled distinctly, optionally pinned. (Licence-expiry now exists as an org-level `warn` item — see §5.9.)
8. **Persistence** — survive reload/login (backend-backed); a **"See all"** full-page activity view.
9. **Preferences** — mute types, do-not-disturb; optional email/desktop for long-running jobs.

---

## 12. Open questions

- **Notification infra** — reuse FlowX's real-time/notification infrastructure vs. a dedicated path. (Recommend reuse; needs codebase confirmation.)
- **Job cancelability** — can an in-flight job be canceled from the modal/notification, or does it run to completion?
- **Artifact / result TTL** — exact retention window (24h / 48h / 72h) and how expiry is surfaced in the feed.
- **Per-user vs. shared** — are notifications strictly per-user, or should a shared operation notify a team?
