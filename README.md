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

Three ways this dashboard could get its data, and why the CSV-upload path won:

- **Live Notion API integration.** This is a static page with no server — a "live"
  integration would mean either shipping a Notion API token to the browser (worse
  than the current passcode, since now the *data source* is exposed, not just the
  view) or standing up a backend purely to hold that secret. That's real
  infrastructure — hosting, monitoring, a place for the integration to silently
  break — for a database that's reviewed a few times a week, not a live feed.
- **Make.com / Zapier automation.** This moves the sync problem to a third party
  instead of removing it: it's still an external dependency with its own
  failure modes (rate limits, silent auth expiry, a scenario that needs
  maintaining), plus a subscription cost, and it still needs somewhere to land
  the data — which would just be this same CSV-in-a-repo pattern with an extra
  hop in front of it.
- **Manual CSV export + upload (current approach).** Zero always-on
  infrastructure. The only secret involved is a fine-grained GitHub PAT, scoped to
  exactly this repo's contents, that the user supplies and controls — not a
  standing integration credential. The manual step matches how often the data
  actually changes, and the GitHub write-back (via the Contents API) means the
  data still persists across devices and sessions without a database.

In short: every fancier alternative trades a five-second manual export for an
always-on system to operate, for a tool one person checks periodically. The CSV
upload is the version of this dashboard that has no uptime to worry about.

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
