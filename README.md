# Cinemanalysis — Project Documentation

## System Overview

Cinemanalysis is a personal movie tracking dashboard with three main components that work together:

- **`index.html`** — the dashboard, hosted on GitHub Pages, reads data from a Google Sheet and renders charts and stats
- **`log.html`** — a custom logging form, also hosted on GitHub Pages, that uses the TMDB API to auto-fill movie metadata and submits entries to the Google Sheet via a Google Apps Script endpoint
- **Google Apps Script** — a serverless endpoint deployed from within the Google Sheet that receives POST requests from `log.html` and appends rows to the sheet

External services used:
- **TMDB (The Movie Database)** — provides movie search autocomplete, and auto-fills title, director, runtime, genres, and cast
- **Google Sheets** — the data store; published as a CSV that `index.html` fetches directly
- **Google Apps Script** — the write endpoint that `log.html` posts to
- **GitHub Pages** — hosts both HTML files

---

## File Inventory

| File | Location | Purpose |
|---|---|---|
| `index.html` | GitHub repo root | Main dashboard |
| `log.html` | GitHub repo root | Movie logging form |
| Apps Script | Google Sheet → Extensions → Apps Script | Receives form submissions and writes rows to the sheet |

---

## Data Flow

### Logging a movie
1. User opens `log.html` and types a film title
2. `log.html` queries the TMDB search API and displays matching results as an autocomplete dropdown
3. User selects a film — `log.html` fetches full details and credits from TMDB, auto-filling title, year, director, runtime, genres, and top 10 billed actors into hidden fields
4. User fills in the four manual fields: date watched, source, new or rewatch, and rating
5. User clicks Submit — `log.html` POSTs a JSON payload to the Apps Script web app URL
6. Apps Script receives the payload, parses it, and appends a new row to the Google Sheet
7. The Google Sheet updates immediately

### Viewing the dashboard
1. User opens `index.html`
2. PapaParse fetches the Google Sheet CSV URL and parses it into row objects
3. `buildDashboard()` runs and populates all stat cards, charts, and tables
4. The dashboard reflects all rows currently in the sheet — there is no caching, so it always shows the latest data on page load

---

## Column Reference

The Google Sheet has the following columns, in order. The `COL` object in `index.html` maps friendly key names to these exact header strings — if a header changes in the sheet, the corresponding value in `COL` must be updated to match.

| Column Header | COL Key | Source | Notes |
|---|---|---|---|
| Timestamp | `timestamp` | Auto (Apps Script) | `new Date()` at time of submission |
| Movie Title | `title` | TMDB auto-fill | |
| Date Watched | `dateWatched` | User input | Date picker field |
| Source | `source` | User input | Dropdown selection |
| New or Rewatch | `newOrRewatch` | User input | Dropdown: "New" or "Rewatch" |
| Release Year | `year` | TMDB auto-fill | 4-digit year extracted from release date |
| Director | `director` | TMDB auto-fill | First crew member with job "Director" |
| Runtime (minutes) | `runtime` | TMDB auto-fill | Integer, in minutes |
| Genres (Letterboxd) | `genres` | TMDB auto-fill | Comma-separated list of genre names |
| Rating | `rating` | User input | Integer 1–10 |
| Actors | `actors` | TMDB auto-fill | Top 10 billed cast members, comma-separated |

---

## Chart and Metric Reference

### The Big Picture

| Stat | Calculation |
|---|---|
| Total Movies Watched | Count of all rows |
| Total Directors Watched | Count of unique values in the Director column |
| Average Movies per Week | Total row count divided by weeks elapsed since January 1, 2026 |
| Total Watch Time | Sum of all Runtime values, formatted as `Xd Xh Xm` |
| Movie Watching Age | Weighted average of `(2046 - release year)` across all rated films, where each film's value is weighted by its rating. Films you rate higher pull the average more. The constant 2046 assumes a "reasonable watching age" of 20 (2026 - 20 = 2006 baseline, applied per film's release year). |
| Longest Streak | Most consecutive calendar days with at least one film watched. Multiple films on the same day count as one day. Tooltip shows the date range of the streak. |

### Genre Data

| Chart | Calculation | Filters |
|---|---|---|
| Most Watched Genres | Count of genre tag appearances across all films. One film can contribute multiple genres. | Top 8 shown |
| Highest Rated Genres | Average rating per genre, weighted equally (not by rating). | Minimum 3 appearances to be eligible. Top 10 shown. |

### Director Data

| Chart | Calculation | Filters |
|---|---|---|
| Most Watched Directors | Count of unique films per director. Rewatches of the same film are deduplicated — only counted once per director. | Minimum 2 films to appear. Top 8 shown. |
| Highest Rated Directors | Average rating per director across unique films. If a film is watched twice, only the highest rating is used. | Minimum 2 films to be eligible. Top 5 shown. |

### Actor Data

| Chart | Calculation | Filters |
|---|---|---|
| Most Watched Actors | Count of unique films per actor. Rewatches of the same film are deduplicated. | Minimum 2 appearances to appear. Top 8 shown. |
| Highest Rated Actors | Average rating per actor across unique films. If a film is watched twice, only the highest rating is used. | Minimum 3 films to be eligible. Top 10 shown. |

### Source Data

| Chart | Calculation |
|---|---|
| Watch Volume by Source | Count of watches per source, displayed as a doughnut chart with percentages |
| Average Rating by Source | Mean rating per source, y-axis 0–10 |

### Watch Status Data

| Chart | Calculation |
|---|---|
| Rewatch Percentage | Pie chart of First Watch vs Rewatch counts with percentages |
| Average Rating by Watch Status | Mean rating for First Watch entries vs Rewatch entries |

### Ratings Data

| Chart | Calculation |
|---|---|
| Rating Distribution | Count of films at each integer rating 1–10. Includes a dashed vertical line showing your average rating. |
| Runtime vs. Rating | Scatter plot of runtime (x) vs rating (y) for all films with both values present. Hover shows film title. |
| Average Rating by Release Decade | Mean rating grouped by the decade of each film's release year, sorted chronologically |

### Watch Volume Data

| Chart | Calculation |
|---|---|
| Watch Volume by Month | Count of watches per calendar month, sorted chronologically |
| Watch Volume by Day of the Week | Count of watches per day name, Sunday through Saturday |
| Watch Volume by Release Decade | Count of watches grouped by the decade of each film's release year |

### Highest Rated New Movies This Year

Table of first-watch films only, deduplicated by title (keeps highest rating if watched multiple times), sorted by rating descending. Shows top 15. Columns: rank, title, director, year, genres, source, rating.

---

## Maintenance Notes

**Updating the year** — the weekly average and streak calculations reference a hardcoded start date of `2026-01-01`. At the start of each new year, update this value in `index.html`:
```javascript
const weeksSinceStart = Math.max(1, (new Date() - new Date('2026-01-01')) / ...
```

**Column header changes** — if you ever rename a column in the Google Sheet (including by editing the corresponding Google Form question), you must update the matching value in the `COL` object in `index.html` to match exactly, including capitalization and punctuation. A mismatch will cause that column's data to silently return empty across all charts.

**Adding new columns** — to add a new data column: (1) add the question to the Google Form or the `log.html` form, (2) add the column header to the sheet, (3) add the key/value pair to `COL` in `index.html`, (4) add the field to the Apps Script `appendRow` call, and (5) redeploy the Apps Script.

**Redeploying Apps Script** — any change to the Apps Script code requires a new deployment. Go to Deploy → Manage deployments → edit the existing deployment → set to New version → Deploy. The URL does not change.

**Google Sheet must be published** — the dashboard reads data via a public CSV URL. If the sheet ever stops being published, the dashboard will show the loading error. To republish: File → Share → Publish to web → select the correct sheet → CSV → Publish.

**TMDB API token** — the TMDB Read Access Token is stored in plaintext in `log.html`. It is a read-only token so exposure is low risk, but be aware it is publicly visible in the GitHub repo. TMDB read tokens do not expire but can be regenerated in your TMDB account settings if needed.

**Apps Script URL** — the web app URL is stored in `log.html`. If you ever need to create a new deployment (rather than updating the existing one), the URL will change and must be updated in `log.html`.

---

## Tech Stack Summary

| Component | Technology |
|---|---|
| Hosting | GitHub Pages |
| CSV parsing | PapaParse 5.4.1 (Cloudflare CDN) |
| Charts | Chart.js 4.4.1 (Cloudflare CDN) |
| Annotations | chartjs-plugin-annotation 3.0.1 (Cloudflare CDN) |
| Movie metadata | TMDB API v3 (Read Access Token auth) |
| Form submission | Google Apps Script Web App (POST endpoint) |
| Data storage | Google Sheets (published as CSV) |
| Fonts | Google Fonts — Inter |
