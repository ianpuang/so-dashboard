# SO Contribution Dashboard

## What this is
Live dashboard for WeMAL technician SO contribution stats, pulled straight from a Google Sheet and hosted as a static site on GitHub Pages. Two sections — individual contribution (top) and group contribution (bottom) — each sorted highest to lowest, each toggleable between bar chart and ranked table.

Built July 2026, replacing a manual screenshot/export-from-Sheets workflow.

## Files
- `index.html` — the entire app. Single file, no build step, no backend, no dependency except Chart.js (loaded from cdnjs at runtime).

## Data source
- Google Sheet, published to web. Publish key is baked into `index.html` as `PUB_KEY`:
  `2PACX-1vTxTi4jJQmXHlV7tG71ROZ_eXwnSUKU5oxOQ0YMQdRcXnTYrTk--TvbW9JkeMDZR_nAQECsSsqShTtH`
- One tab per month, named like `so july`, `so august`. Tab in use when this was built: `so july`, gid `754850172` — stored as `DEFAULT_GID`, the fallback if tab auto-detection ever fails.
- Columns (sheet letters vs. 0-indexed `COL` constant in code):

  | Sheet col | Code index | Contains |
  |---|---|---|
  | M | 12 | Technician name |
  | N | 13 | SO count — group contribution |
  | O | 14 | SO count — individual contribution |

  The sheet also has an "SO value by individual contribution" column — not used here; this dashboard is counts only.

  **2026-07-09:** shifted one column right (was L/M/N = 11/12/13) after someone inserted a column ahead of the technician block on the `so july` tab. If the dashboard shows "couldn't load data" again but the sheet is confirmed still published and the CSV endpoint returns data, check for this first — fetch the raw CSV and diff actual column contents against `COL` before assuming a publish/connectivity issue.
- Visit counter: separate Google Sheet `so-dashboard-counter` (count lives in A1, owned
  by ianpuang48@gmail.com) with a bound Apps Script web app. Its `/exec` URL is baked into `index.html` as `COUNTER_URL`
  ("" disables the counter). One increment per page load; on any fetch failure the
  footer line stays hidden and the dashboard is unaffected.

## How the live part works
1. On load and every `REFRESH_SECONDS` (60s), fetches `.../pub?gid=<gid>&output=csv`.
2. Tab auto-detection: scrapes the `.../pubhtml` sheet-switcher menu for `{gid, name}` pairs, matches the current month's name/abbreviation against tab names, and switches automatically. Re-scans every 6 hours so a wall/kiosk display rolls into next month's tab on its own — no manual edit needed at month-end.
3. If scraping fails (CORS, Google changes markup), falls back to `DEFAULT_GID`. Update that constant manually as an interim fix if auto-detection ever misbehaves.
4. A month dropdown appears whenever more than one tab is detected, for browsing past months.

## Locked-in design decisions
(from a mockup shown inline and approved before building — don't flip these without a reason)
- Individual contribution chart sits **above** group contribution, not the reverse.
- Both lists sorted descending by value, always.
- Chart ↔ table toggle is per-section, not one global toggle.
- Top 3 non-zero entries are visually emphasized (darker bar, bold table row).
- Individual table shows zero as "–" (matches the sheet's own convention); group table shows "0".

## Known constraints
- Sheet must stay **published to web** — unpublishing silently breaks the fetch (page then shows an amber "connection issue" pill and keeps displaying last-good data rather than going blank).
- ~5 minute lag between a sheet edit and it showing here — Google's republish cadence for published sheets, not adjustable from this side.
- Published sheet + public GitHub Pages repo means anyone with the URL can read it. It's staff performance data — keep the link internal.

## Deployment (GitHub Pages)
```bash
mkdir so-dashboard && cd so-dashboard
mv ~/Downloads/index.html .
git init && git add . && git commit -m "SO dashboard"
gh repo create so-dashboard --public --source=. --push
gh api repos/{owner}/so-dashboard/pages -f 'source[branch]=main' -f 'source[path]=/'
```
Live at `https://<user>.github.io/so-dashboard/` a minute or two after Pages is enabled.

## Quick edits for future sessions
- Refresh cadence → `REFRESH_SECONDS` near the top of the `<script>`.
- Sheet's column layout changes → update the `COL` constant.
- New month → add a sheet tab named `so <month>`; no code change needed as long as it's under the same publish key. If it doesn't auto-pick-up, check/update `DEFAULT_GID`.
- Counter broken or sheet deleted → set `COUNTER_URL` to `""` to hide it, or redeploy the
  Apps Script and paste the new `/exec` URL.
