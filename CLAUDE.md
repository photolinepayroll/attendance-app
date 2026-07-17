# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository overview

Static GitHub Pages site — no build step, no package manager, no tests. All code is plain HTML/CSS/JS in two files:

- `index.html` — employee-facing attendance form (selfie + GPS log in/out)
- `admin.html` — password-protected admin dashboard (view, filter, export, print all records)
- `Code.gs` — local mirror of the Google Apps Script backend source, kept for reference/version-tracking. **Not deployed automatically** — the live backend only updates when this is manually pasted into the Apps Script editor and redeployed. `OldAppscrip.txt` is a backup of the pre-optimization version, kept for comparison.

## Deploying changes

Push directly to `main`. GitHub Pages serves from `main` automatically — changes go live within ~1 minute.

```bash
git add <file>
git commit -m "description"
git push origin main
```

## Architecture

### Backend: Google Apps Script
Both pages talk to the same Google Apps Script web app deployment (`SUBMIT_URL` in `index.html`, `SCRIPT_URL` in `admin.html` — kept in sync manually, there's no shared config, so re-check both if the deployment URL ever changes). The script reads/writes a Google Sheet and handles:
- `doPost` — append a new attendance row + upload the selfie to Drive (called from `index.html`'s `syncRecord()`)
- `action=geocode&lat=&lng=&callback=` (via `doGet`) — reverse geocode via JSONP (called from `admin.html` to avoid CORS)

Backend perf notes (see `Code.gs` vs `OldAppscrip.txt` for the diff): `doPost` no longer does a synchronous Nominatim reverse-geocode call on the submit path (that's now left to `admin.html`'s lazy per-row geocoding), and the Drive folder id is cached via `PropertiesService` with sharing set once at the folder level instead of per uploaded file.

### Data source: Google Sheets CSV
`admin.html` reads all records by fetching a public CSV export URL (`CSV_URL`) from Google Sheets directly in the browser. Records are parsed client-side with a custom CSV parser (`parseCSV()`).

### index.html — key flows
- **GPS**: `startGPS()` uses `navigator.geolocation.watchPosition`. Fake GPS is detected via speed checks (`isMockLocation()`), known spoofed coordinates, and `gps.mocked` flag. GPS state is stored in the `gps` global.
- **Clock integrity**: A manually-set phone clock would otherwise poison the displayed time, the selfie's burned-in watermark, and the record's `ts`. `checkNetworkTime()` fetches the page's own URL (same-origin `HEAD`, no backend involved) and reads the HTTP `Date` response header as a trusted time source, storing the drift in `clockOffsetMs`. `trustedNow()` (used everywhere a record/display timestamp is generated, in place of `new Date()`) applies that offset so the clock self-corrects even before the phone setting is fixed. If the drift exceeds `CLOCK_DRIFT_LIMIT_MS` (2 minutes), `submitAttendance()` blocks submission via `isClockBad()` until the user re-enables automatic (network) date & time — mirroring how fake-GPS checks already block submission. Fails open (no block) if the check can't complete, e.g. offline.
- **Camera**: `openCamera()` calls `getUserMedia`, explicitly calls `video.play()` for iOS/in-app browser compatibility, and detects in-app browsers (`isInAppBrowser()`) to warn before attempting. `takePhoto()` caps the captured frame to 720px on the long edge and encodes at JPEG quality 0.78 (was full camera resolution @ 0.88) to keep the upload small.
- **Submission (optimistic)**: `submitAttendance()` shows the success screen immediately — it does not wait on the network. The record is saved to `localStorage` as `synced: false`, then `syncRecord()` POSTs it to `SUBMIT_URL` in the background. On success the record flips to `synced: true`; on failure it retries with backoff (5s/15s/45s) and again on the browser `online` event, and `syncPendingRecords()` re-attempts any still-unsynced records on every page load. The success screen's `#syncStatus` line reflects live state ("Please don't close..." → "✅ Logs Recorded", or a tap-to-retry prompt if all retries fail) so a failed save is never silent.
- **Local cache**: Recent records stored in `localStorage` key `attendRecords2`. Records cached before this sync-tracking existed are migrated to `synced: true` on load (they were only ever cached after the old blocking flow completed).

### admin.html — key flows
- **Auth**: Plain password check against hardcoded `ADMIN_PASS`. Session persisted in `sessionStorage`.
- **Rendering**: `renderTable()` renders 50 rows at a time (pagination via `currentPage` / `PAGE_SIZE`). Geocode requests for visible rows are staggered at 1100ms intervals to respect Nominatim rate limits.
- **Geocoding**: Uses JSONP to call the Apps Script `action=geocode` endpoint (avoids CORS from GitHub Pages). Results cached in `_geoCache`.
- **Export/Print**: `exportCSV()` and `printPreview()` operate on `getSelectedRecords()` — either checked rows or all filtered rows if none checked.
