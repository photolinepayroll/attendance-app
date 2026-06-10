# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository overview

Static GitHub Pages site — no build step, no package manager, no tests. All code is plain HTML/CSS/JS in two files:

- `index.html` — employee-facing attendance form (selfie + GPS log in/out)
- `admin.html` — password-protected admin dashboard (view, filter, export, print all records)

## Deploying changes

Push directly to `main`. GitHub Pages serves from `main` automatically — changes go live within ~1 minute.

```bash
git add <file>
git commit -m "description"
git push origin main
```

## Architecture

### Backend: Google Apps Script
Both pages talk to a single Google Apps Script web app (`SCRIPT_URL` in each file). The script reads/writes a Google Sheet and handles:
- `action=submit` — append a new attendance row (called from `index.html`)
- `action=geocode&lat=&lng=&callback=` — reverse geocode via JSONP (called from `admin.html` to avoid CORS)

The script URL in `index.html` and `admin.html` may differ — `index.html` uses an older deployment, `admin.html` uses a newer one.

### Data source: Google Sheets CSV
`admin.html` reads all records by fetching a public CSV export URL (`CSV_URL`) from Google Sheets directly in the browser. Records are parsed client-side with a custom CSV parser (`parseCSV()`).

### index.html — key flows
- **GPS**: `startGPS()` uses `navigator.geolocation.watchPosition`. Fake GPS is detected via speed checks (`isMockLocation()`), known spoofed coordinates, and `gps.mocked` flag. GPS state is stored in the `gps` global.
- **Camera**: `openCamera()` calls `getUserMedia`, explicitly calls `video.play()` for iOS/in-app browser compatibility, and detects in-app browsers (`isInAppBrowser()`) to warn before attempting.
- **Submission**: `submitAttendance()` POSTs form data (name, destination, type, GPS, base64 photo) to the Apps Script URL as JSON.
- **Local cache**: Recent records stored in `localStorage` key `attendRecords2`.

### admin.html — key flows
- **Auth**: Plain password check against hardcoded `ADMIN_PASS`. Session persisted in `sessionStorage`.
- **Rendering**: `renderTable()` renders 50 rows at a time (pagination via `currentPage` / `PAGE_SIZE`). Geocode requests for visible rows are staggered at 1100ms intervals to respect Nominatim rate limits.
- **Geocoding**: Uses JSONP to call the Apps Script `action=geocode` endpoint (avoids CORS from GitHub Pages). Results cached in `_geoCache`.
- **Export/Print**: `exportCSV()` and `printPreview()` operate on `getSelectedRecords()` — either checked rows or all filtered rows if none checked.
