# Job Application Dashboard

A single-page dashboard for tracking a job search — upload a CSV export from a Notion
database and it renders trends, an aging distribution, and interview/application lists.
No backend, no database, no build step: it's one static `index.html` file plus the
CSV it reads (`data/data.csv`).

## Using the dashboard

1. **Open the page.** If a CSV has already been saved to the repo, the dashboard
   loads it automatically. Otherwise you'll land on the upload screen.
2. **Export from Notion.** In the source database (`Resumes and JDs`): `···` → `Export`
   → Format: **CSV** → Include: **Everything**.
3. **Drop the CSV** onto the upload screen (or click it to browse). The dashboard
   parses it entirely in your browser and renders:
   - **Applications by Month** — volume trend, Sep 2025 onwards
   - **Aging Distribution** — how long open applications have been sitting (JDs from
     Jan 2026 onwards, capped at 120 days so a handful of very old stragglers don't
     flatten the curve), with a fitted normal curve and a live datapoint per
     application — hover a dot for its role and company
   - **Recent Applications** — new JDs added in the last 15 days
   - **Recent Interviews** — anything `In Progress - Interviewing` or
     `Done - Interviewed` in the last 6 months
   - **Role Theme Trends** — which functional area (architecture, product, data/AI,
     engineering, ops) applications are clustering around, by quarter
4. **It saves itself.** After a successful upload, the dashboard commits the CSV
   back to this repo (`data/data.csv`) via the GitHub API, so the next visit — from
   any device — auto-loads the latest data without re-uploading.
5. **Column mapping is tolerant, not silent.** Notion export columns get matched
   fuzzily (`Application Status` → `status`, etc.), and the dashboard always shows
   what it matched in a debug strip, so a renamed column shows up as `NOT FOUND`
   instead of quietly breaking a chart.

## Architecture decisions

This is a personal, single-user tool, and the design leans hard into "no
infrastructure to maintain" over "technically more correct." A few decisions that
follow from that:

### Why a passcode, not real auth

The dashboard sits behind a simple view passcode (and a separate one gating uploads),
hashed with SHA-256 and stored in `localStorage` per device — no accounts, no
session server, no third-party auth provider.

This is a deliberate trade-off, not an oversight: the app is a static page in a
**public** repo, so the check itself is fully visible and bypassable via
view-source. Real access control (OAuth, server-side sessions) would require
standing up and maintaining a backend for a tool that has exactly one user. The
passcode is a *casual deterrent* against a stranger stumbling onto the page and
seeing salary figures and company names — not a security boundary. If that
threat model ever changes (e.g. this becomes multi-user, or the data becomes more
sensitive than "a job search"), this is the first thing to replace, not extend.

### Why CSV upload, not a live Notion sync or a Make.com/Zapier workflow

Both of the "proper" alternatives were actually tried first, and both ran into the
same core problem: Notion's API doesn't return the values this dashboard actually
needs, only the raw values it stores.

- **Live Notion API integration — abandoned.** Two separate blockers, not just
  "extra infrastructure":
  - **Token cost.** Notion's API has no incremental/delta query — pulling only
    what changed requires their search API's filtering, which isn't available on
    the free plan. Every sync meant re-pulling the *entire* JD database, burning
    through API calls for data that mostly hadn't changed.
  - **Derived values don't come back derived.** `Aging (Days)` is a Notion
    rollup/formula field — the API doesn't return the computed number, so it
    would have to be recalculated client-side in an ETL step from raw dates
    anyway. `Company` is a relation field — the API returns a Notion *record ID*,
    not the company name, which means a second full download of the entire
    companies database just to resolve IDs to names. What looked like "call an
    API" turned into "reimplement two of Notion's own computed columns," which
    was too elaborate for a personal tracker.
- **Make.com / Zapier automation — also abandoned.** This solved the first
  problem (it could pull the full database in manageable chunks without hitting
  the free-plan search-API wall) but not the second: it still received the same
  raw relation IDs and formula-less values Notion's API returns, so `Aging` and
  `Company Name` still needed the exact same resolution/recalculation step,
  just relocated into a Zap/scenario instead of removed.
- **Manual CSV export + upload (current approach) — what actually works.**
  Notion's own **export** (as opposed to its API) already resolves both problem
  fields — `Aging (Days)` comes out as a plain number, `Company` comes out as the
  actual name — because the export runs through Notion's UI rendering layer, not
  its raw API. That sidesteps both blockers at once, for the cost of one manual
  click every so often. The only secret involved from there is a fine-grained
  GitHub PAT, scoped to exactly this repo's contents, that the user supplies and
  controls — not a standing integration credential. The GitHub write-back (via
  the Contents API) then means the data persists across devices and sessions
  without a database.

In short: this isn't "manual because simpler is nicer" — the API-based paths were
tried and didn't produce usable data without rebuilding logic Notion's UI export
already does for free.

### Why GitHub-as-database

Rather than a real database, uploaded CSVs are committed straight to
`data/data.csv` in this repo. That gives free version history (every upload is a
commit), free persistence, and free "hosting" (the same static page serves the
latest data on load) — at the cost of the data being visible in the repo's commit
history to anyone with read access, which is the same trade-off the passcode
already makes.

## Local development

There's no build step or package manager. To run it locally:

```
python -m http.server
```

Then open `http://localhost:8000/`. A local server is required (rather than
opening `index.html` directly) because the dashboard `fetch()`es
`data/data.csv` on load, which browsers block under the `file://` protocol.
