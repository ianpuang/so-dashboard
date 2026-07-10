# Visit Counter for SO Dashboard — Design Spec

**Date:** 2026-07-10
**Status:** Approved by Ian (in-session)
**Scope:** One feature — a simple visible visit counter on the SO contribution dashboard.

## Problem

The SO dashboard (static GitHub Pages site, single `index.html`, no backend) has no way to
show how many times it has been opened. Ian wants a simple counter, visible on the page
itself, to monitor traffic.

## Chosen approach

Google Apps Script web app backed by a dedicated Google Sheet. Chosen over hosted
analytics (GoatCounter) and public hit-counter APIs because it stays inside the Google
stack already used by this dashboard, adds no third-party dependency, is free, and the
data is owned by us (a cell in a sheet).

## Components

### 1. Counter sheet (new, manual setup)

- A **new, dedicated Google Sheet** named `so-dashboard-counter`.
- Deliberately separate from the SO stats sheet so the counter can never interfere with
  the staff data or its published CSV (that sheet has a history of column-shift breakage
  — see incident log 2026-07-09).
- Cell `A1` holds the lifetime visit count (a plain integer).

### 2. Apps Script web app (new, manual setup guided in-session)

- Script bound to the counter sheet.
- `doGet()`: increment `A1` by 1, return the new count.
- Response format: JSON `{"count": N}` via `ContentService`.
- Deployment: web app, **execute as me**, **anyone with the link** can access.
  The deployment URL is unguessable but public — acceptable for a hit counter.
- **Known uncertainty:** Apps Script + browser CORS has quirks. The standard
  `ContentService` JSON pattern usually works for simple cross-origin GETs; if it does
  not, fall back to returning plain text (`ContentService.MimeType.TEXT`), which is a
  known-good workaround. Verify during implementation; do not assume.

### 3. Frontend change (`index.html` — the only code change in this repo)

- New constant `COUNTER_URL` next to `PUB_KEY` / `DEFAULT_GID` at the top of the script.
- On page load, a single `fetch(COUNTER_URL)`; on success, render the count.
- The 60-second data-refresh loop does **not** touch the counter. A kiosk left open all
  day therefore counts as exactly 1 visit — intended behaviour.
- Display: a small telemetry-style line (JetBrains Mono, muted colour, consistent with
  the page's existing style) at the bottom of the page (footer position), e.g.
  `1,234 visits` (thousands-separated). If the page has no footer today, add a minimal
  one for this line only.

## Error handling

- If the counter fetch fails for any reason (script deleted, offline, Apps Script quota,
  CORS), the counter element stays hidden. No error text, no console spam beyond a
  single caught rejection, and zero effect on the dashboard's real job of showing SO
  stats.

## Explicitly out of scope (YAGNI)

- Unique-visitor dedup, per-day/per-month breakdowns, timestamps, referrers, admin view.
- It is a single lifetime number. The dedicated sheet leaves room to add a timestamp log
  later if trends are ever wanted.

## Success criteria

1. Opening the dashboard increments the count by 1 and displays the new total on the page.
2. Reload → count goes up by 1 again.
3. With `COUNTER_URL` unreachable, the dashboard looks and works exactly as it does today
   (counter simply absent).
