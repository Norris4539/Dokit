---
name: dokit
description: >
  Working knowledge of the Dokit field-service PWA in this repository. Use when
  reading or changing anything in index.html, sw.js, manifest.json or icons/ —
  UI tweaks, calendar/job-detail/Site Visit Report changes, PWA install or
  service-worker behaviour, demo data, or deploying to GitHub Pages.
---

# Dokit — Field Technician Job Management PWA

## What this app is

A single-file, dependency-free Progressive Web App for field technicians:
a calendar of work orders (day/week/month), a job-detail view, and a
Site Visit Report (SVR) form with time clock, note templates, materials,
signature capture and photos. Hosted on **GitHub Pages** (static — there is
no live backend; API calls exist in the code for a future backend).

## Repository layout

| File | Role |
|---|---|
| `index.html` | The entire app: CSS in one `<style>`, JS in one `<script>` (~1,120 lines). No build step, no frameworks. |
| `sw.js` | Service worker: shell pre-cache, cache-first navigations, push/notification handlers. |
| `manifest.json` | Web app manifest. Light/dark icon variants via `media`; separate `purpose: any` and `purpose: maskable` entries; `orientation: any` (needed for landscape signature capture). |
| `icons/` | `icon-192/512-{light,dark}.png` (full-bleed, `any`) + `icon-512-maskable-{light,dark}.png` (padded so the gold corner brackets survive the OS mask crop). Names must match `manifest.json` and `sw.js` `SHELL` exactly — a 404 here silently breaks installability. |

## Non-negotiable conventions

1. **Bump the service-worker cache on every change to `index.html`**:
   `sw.js` line 1, `const CACHE = 'dokit-vNN'` → `vNN+1`. Without this,
   installed apps keep serving the old cached shell indefinitely.
2. **Deploys are pushes to `main`.** GitHub Pages auto-deploys `main`
   ("pages build and deployment" workflow). Work on other branches never
   reaches the live site.
3. **Validate JS before committing**: extract the inline `<script>` block and
   run `node --check` on it (a syntax error white-screens the whole app).
4. **Match the existing style**: ES6+, no semicolon-free style, single-quote
   strings, compact one-liner helpers, theme via CSS custom properties only —
   never hard-code colours that exist as tokens.

## Architecture (all in index.html)

- **DOM helper `h(tag, attrs, ...children)`** — `document.createElement`
  based. `className`, `style` object, `onClick`-style listeners, everything
  else via `setAttribute`. **It cannot create SVG elements** (no
  `createElementNS`); build SVG via `createElementNS` + `innerHTML` (see
  `clockPicker`) or an HTML string.
- **State**: global `STATE = {token, user, jobs, offline, calView, calDate}`.
  `token === 'demo'` means demo mode → all API calls are skipped/faked.
- **Router**: `route(hash)` + `render()` on `hashchange`. Screens:
  `#login`, `#calendar`, `#profile`, `#job/{id}`, `#job/{id}/svr`.
- **Persistence**: IndexedDB `dokit` v1 with stores `jobs`, `sync_queue`
  (offline status updates, replayed by `replaySyncQueue`), `svr_drafts`
  (autosaved SVR form state, keyed by `job_id`). `localStorage`: `dk_dark`,
  `dk_cal_view`.
- **API layer**: `apiFetch`/`apiJSON` → `/api/*` with Bearer token and a
  401→`/api/auth/refresh` retry. On GitHub Pages these endpoints 405 —
  harmless; `boot()` catches it and falls through to the login screen.

### Demo mode

- Credentials in `LoginScreen` / constants: `DEMO_EMAIL = 'admin@dokit.co.za'`,
  `DEMO_PASSWORD = 'Admin1234!'` (exact, case-sensitive match, client-side only).
- `DEMO_JOBS` (3 jobs with title/location/contact/description/required_stock),
  `DEMO_STOCK` (materials dropdown items), `DEMO_USER`.

### Key components (search by function name; line numbers drift)

- `CalendarScreen` + `DayView`/`WeekView`/`MonthView` — time-grid with
  priority-coloured events; week header is sticky inside the scroll container.
- `JobScreen` → `renderJobDetail` → `jobCard` (the `.biz6` report-style
  grid card) and `buildActions` (status dropdown + conditional
  "Submit Site Visit Report" shown only when status is `completed`).
- `updateStatus` — optimistic update with `.ws-skeleton` shimmer while saving;
  queues to `sync_queue` when offline.
- `SVRScreen`/`renderSVR` — the SVR form: Note Template dropdown
  (`NOTE_TEMPLATES`: `onsite` "On Site Callout", `remote` "Remote Work";
  templates swap the free-text Visit Notes for structured fields — contact +
  Work Completed + Work Remaining — and relabel the clock), `clockPicker`
  (draggable analog SVG time picker; persistent SVG node, touch listeners on
  the SVG itself — do not re-render it per pointer-move or mobile drag breaks),
  materials rows, `openSignatureCapture` (fullscreen canvas modal, exports an
  upright PNG), photos (max 5, ≤10 MB each). `composeNotes(st)` builds the
  final Work Description. Drafts autosave to `svr_drafts` (2 s debounce).
- `reportPreview` — display-only A4-styled overlay shown after submit.
- `styledSelect(options, value, onChange, opt)` — themed custom dropdown
  (`.ssel*`) used instead of native `<select>` (which opens the OS picker on
  phones). `options` is `[[value, label], …]`; `opt.triggerStyle` inline-styles
  the trigger (used for the status-coloured dropdown), `opt.wide` for the
  materials variant.
- `statusInfo`/`priorityInfo` — canonical status/priority colours and labels.
- Install flow: `beforeinstallprompt` → custom banner (`showInstallBanner`);
  iOS gets manual instructions. Splash: `#splash` spinning icon, shown only in
  standalone/PWA display mode, hidden by `hideSplash()` on load.

### Theme

CSS custom properties on `.ws-root` (light) overridden by `.ws-root.dark`:
`--sumi` (ink), `--washi` (paper), `--shiro`, `--tsuchi`, `--nori`, `--aka`
(red/outstanding), `--matcha` (green/completed), `--indigo` (in-progress),
`--ki` (gold accent), plus `--bg/--card-bg/--card-border/--text-pri/--text-sec/
--text-dim/--input-bg/--grid`. Fonts: Playfair Display (brand serif),
Noto Serif JP, Noto Sans JP (body). Section labels use `.sl`; inputs `.ws-input`.

## Verifying changes

1. `node --check` the extracted inline script.
2. Serve locally: `python3 -m http.server 8123` in the repo root.
3. Drive it headlessly (Playwright for Python + the pre-installed Chromium at
   `/opt/pw-browsers/chromium-*/chrome-linux/chrome`): log in with the demo
   credentials, `location.hash='#job/2/svr'` (or any route), interact, and
   screenshot. Demo mode exercises the full UI without a backend.
4. For visual samples, render standalone HTML mockups reusing the theme tokens
   above with headless Chromium `--screenshot`.

## Deploying

Commit `index.html` + the `sw.js` cache bump together, push to `main`, then
confirm the "pages build and deployment" workflow run for the new commit
succeeds. Live URL: https://norris4539.github.io/Dokit/. Installed clients
pick the change up on next launch via the cache-version bump.
