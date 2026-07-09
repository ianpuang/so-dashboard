# Incident log

## 2026-07-09 — "Couldn't load data from the published sheet"

**Symptom:** Live page (`https://ianpuang.github.io/so-dashboard/`) showed the amber
"connection issue" / "Can't reach the sheet" message. Sheet itself looked normal when
opened directly in Google Sheets, still published to web.

**Root cause:** Someone inserted a new column in the `so july` tab ahead of the
technician/contribution block, shifting it one column to the right:

| | Before | After |
|---|---|---|
| Technician name | L (11) | M (12) |
| SO count — group | M (12) | N (13) |
| SO count — individual | N (13) | O (14) |

`index.html`'s hardcoded `COL` constant still pointed at the old L/M/N indices, so it
read the technician-name column as blank on every row, found zero valid rows, and threw
`"no rows found in columns L–N"`. That error surfaces as a generic "couldn't load /
check it's published" message — misleading, since the sheet was published and reachable
the entire time. The publish status was a red herring.

**Diagnosis method:** Don't trust the UI's error text. Fetch the published CSV directly
(`.../pub?gid=<gid>&single=true&output=csv`) and inspect actual column contents against
the `COL` constant in `index.html` before assuming a publish/connectivity problem.

**Fix:** Updated `COL` in `index.html` from `{ tech: 11, group: 12, indiv: 13 }` to
`{ tech: 12, group: 13, indiv: 14 }`. See `CLAUDE.md` for the current column mapping.

**Takeaway for next time:** If this error recurs, check for a column shift first — it's
the most likely cause given the sheet is manually edited by multiple people. Verify via
raw CSV fetch, not just re-checking the publish toggle.
