# Visit Counter Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Show a lifetime visit count on the SO dashboard, backed by a Google Apps Script web app that increments a cell in a dedicated Google Sheet.

**Architecture:** A new Google Sheet (`so-dashboard-counter`) holds the count in cell A1. A bound Apps Script `doGet()` increments it under a lock and returns `{"count": N}` as JSON. `index.html` fetches that URL once per page load and shows the number in the existing footer; any failure leaves the footer exactly as it is today.

**Tech Stack:** Google Apps Script (counter backend), vanilla JS in `index.html` (frontend). No new libraries, no build step.

**Spec:** `docs/superpowers/specs/2026-07-10-visit-counter-design.md`

## Global Constraints

- `index.html` stays a single file with no build step; the only allowed external dependency remains Chart.js (already present). Do not add fonts, libraries, or other script tags.
- The counter must never affect the dashboard's real job: any counter failure → the counter line stays hidden, no visible error, dashboard renders normally.
- The counter fetch runs **once per page load** — it must not be called from `refresh()` or any interval.
- Use a **new, dedicated** Google Sheet for the count. Never touch the SO stats sheet (publish key `2PACX-1vTxTi4jJQmXHlV7tG71ROZ_eXwnSUKU5oxOQ0YMQdRcXnTYrTk--TvbW9JkeMDZR_nAQECsSsqShTtH`) or the `COL` / `PUB_KEY` / `DEFAULT_GID` logic.
- Only modify what each task lists. No refactoring, renaming, or reformatting of existing code.
- Pushing to GitHub (Pages deploy) requires the user's explicit in-session confirmation — never push without it.
- This project has no automated test framework (static HTML). Every task ends with a **manual verification step with exact expected output** instead of a unit test. Do not claim a step works without running its verification.

---

### Task 1: Counter backend (Google Sheet + Apps Script web app)

This task happens in the browser (Google account: the user's), not in the repo. It is performed by Ian with the exact content below; the deliverable is a working web-app URL.

**Files:**
- None in the repo. External: new Google Sheet `so-dashboard-counter`, bound Apps Script.

**Interfaces:**
- Produces: a deployment URL of the form `https://script.google.com/macros/s/<DEPLOYMENT_ID>/exec` which, on HTTP GET, increments the count and responds `{"count": <integer>}` with mime type JSON. Task 2 consumes this URL as `COUNTER_URL`.

- [ ] **Step 1: Create the sheet**

At sheets.google.com create a new spreadsheet named `so-dashboard-counter`. Type `0` into cell **A1** of the first tab. Nothing else goes in the sheet.

- [ ] **Step 2: Add the script**

In the sheet: **Extensions → Apps Script**. Delete the placeholder code and paste exactly:

```javascript
function doGet() {
  const lock = LockService.getScriptLock();
  lock.waitLock(5000); // serialize concurrent hits so no increment is lost
  try {
    const cell = SpreadsheetApp.getActiveSpreadsheet().getSheets()[0].getRange("A1");
    const count = (Number(cell.getValue()) || 0) + 1;
    cell.setValue(count);
    SpreadsheetApp.flush();
    return ContentService.createTextOutput(JSON.stringify({ count: count }))
      .setMimeType(ContentService.MimeType.JSON);
  } finally {
    lock.releaseLock();
  }
}
```

Save (Ctrl+S), name the project `so-dashboard-counter` when prompted.

- [ ] **Step 3: Deploy as web app**

**Deploy → New deployment → type: Web app**:
- Description: `visit counter v1`
- Execute as: **Me**
- Who has access: **Anyone**

Click Deploy, authorize when Google asks (it will warn the app is unverified — Advanced → Go to so-dashboard-counter), then copy the **Web app URL** (`https://script.google.com/macros/s/…/exec`).

- [ ] **Step 4: Verify — two GETs increment by exactly 1 each**

From the repo machine (PowerShell):

```powershell
Invoke-RestMethod "<WEB_APP_URL>"
Invoke-RestMethod "<WEB_APP_URL>"
```

Expected: first call prints `count : 1`, second prints `count : 2` (Invoke-RestMethod follows the script.google.com → googleusercontent.com redirect automatically). Also confirm sheet cell A1 now shows `2`.

If the response is an HTML login page instead of JSON: the deployment's "Who has access" is not **Anyone** — redeploy with the correct setting.

- [ ] **Step 5: Record the URL**

Paste the web app URL into the session for Task 2. Nothing to commit — this task changes no repo files.

---

### Task 2: Frontend — fetch and display the count

**Files:**
- Modify: `index.html` (three small additions: one line in the config block ~line 128, one element in the footer ~line 117-119, one IIFE at the end of the script ~line 356)

**Interfaces:**
- Consumes: `COUNTER_URL` — the web app URL produced by Task 1 (`https://script.google.com/macros/s/…/exec`). Response shape: `{"count": <integer>}`.
- Produces: nothing consumed by later tasks.

- [ ] **Step 1: Add the config constant**

In the `/* ============ CONFIG ============ */` block, directly after the `COL` line:

```javascript
const COUNTER_URL = "<WEB_APP_URL from Task 1>";  /* Apps Script visit counter; "" disables */
```

- [ ] **Step 2: Add the footer element**

Change the footer from:

```html
  <footer>
    Data source: published Google Sheet · sheet edits appear here within ~5 minutes (Google republish delay)
  </footer>
```

to:

```html
  <footer>
    Data source: published Google Sheet · sheet edits appear here within ~5 minutes (Google republish delay)
    <div id="visitCount" class="hidden"></div>
  </footer>
```

No CSS changes — it inherits the footer's existing 12px `--faint` centred style, and `.hidden` already exists.

- [ ] **Step 3: Add the counter fetch**

At the end of the script, after the existing `/* ---------- boot ---------- */` IIFE (after line `})();`), add:

```javascript
/* ---------- visit counter (runs once per page load) ---------- */
(async () => {
  if (!COUNTER_URL) return;
  try {
    const res = await fetch(COUNTER_URL);
    if (!res.ok) throw new Error("counter " + res.status);
    const data = await res.json();
    if (Number.isFinite(data.count)){
      $("visitCount").textContent = Number(data.count).toLocaleString("en") + " visits";
      $("visitCount").classList.remove("hidden");
    }
  } catch (e) {
    console.warn("Visit counter unavailable.", e);
  }
})();
```

**Contingency (only if Step 4 shows a CORS error in the console):** change the Apps Script (Task 1 Step 2) to return plain text — replace the `return` statement with:

```javascript
    return ContentService.createTextOutput(String(count))
      .setMimeType(ContentService.MimeType.TEXT);
```

redeploy (**Deploy → Manage deployments → edit → New version**, same URL), and in the frontend replace the two lines `const data = await res.json();` / `if (Number.isFinite(data.count)){` with:

```javascript
    const data = { count: parseInt(await res.text(), 10) };
    if (Number.isFinite(data.count)){
```

- [ ] **Step 4: Verify locally — counter shows and increments**

Note cell A1's current value (call it N). Open `index.html` in a browser (double-click the file). Expected: dashboard loads normally, and the footer shows a second line `N+1 visits` (thousands-separated once ≥1,000). Reload the page → `N+2 visits`. DevTools console shows no counter errors.

- [ ] **Step 5: Verify failure mode — dashboard unaffected**

Temporarily set `const COUNTER_URL = "https://script.google.com/macros/s/invalid/exec";`, reload. Expected: footer shows only the original single line (no counter, no error text on the page); console shows one `Visit counter unavailable.` warning; charts/tables render normally. Then restore the real URL and reload once more to confirm the counter is back.

- [ ] **Step 6: Commit**

```powershell
git add index.html
git commit -m "Add visit counter to footer via Apps Script endpoint"
```

---

### Task 3: Document + deploy to GitHub Pages

**Files:**
- Modify: `CLAUDE.md` (this repo's — `so-dashboard/CLAUDE.md`), sections "Data source" area and "Quick edits for future sessions"

**Interfaces:**
- Consumes: the committed `index.html` from Task 2.
- Produces: nothing — terminal task.

- [ ] **Step 1: Document the counter in CLAUDE.md**

Add to the end of the "## Data source" section:

```markdown
- Visit counter: separate Google Sheet `so-dashboard-counter` (count lives in A1) with a
  bound Apps Script web app. Its `/exec` URL is baked into `index.html` as `COUNTER_URL`
  ("" disables the counter). One increment per page load; on any fetch failure the
  footer line stays hidden and the dashboard is unaffected.
```

Add one bullet to "## Quick edits for future sessions":

```markdown
- Counter broken or sheet deleted → set `COUNTER_URL` to `""` to hide it, or redeploy the
  Apps Script and paste the new `/exec` URL.
```

- [ ] **Step 2: Commit the docs**

```powershell
git add CLAUDE.md
git commit -m "Document visit counter setup"
```

- [ ] **Step 3: Push (REQUIRES explicit user confirmation in the current message)**

Ask Ian for confirmation, then:

```powershell
git push
```

- [ ] **Step 4: Verify live**

Wait ~1–2 minutes for Pages to rebuild, then open the live dashboard URL (`https://<user>.github.io/so-dashboard/`) in a browser. Expected: footer shows the visits line, one higher than before the page was opened; cell A1 in `so-dashboard-counter` matches the shown number.
