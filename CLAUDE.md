# Kenya Health Partners Map — Project Overview

## Project Summary

An interactive single-page web application that renders Kenya's 47 county boundaries and overlays health partner locations fetched live from a Google Sheet. Built with Leaflet.js and vanilla JavaScript. No backend required.

---

## Objective

Build a self-contained `index.html` that:

1. Fetches partner data directly from a published Google Sheet (CSV)
2. Renders all 47 Kenya county boundaries from a GeoJSON source
3. Plots each health partner as a clickable/hoverable marker
4. Works offline-ready after initial data load, requiring no file uploads or manual input

---

## Data Source

| Item | Value |
|------|-------|
| Google Sheet URL | `https://docs.google.com/spreadsheets/d/17uBPv58cH3hRDRk94vzTGjZmZFrCy5S5tBbhLB2TKMY/edit?usp=sharing` |
| Sheet Name | `Partners Directory` |
| Data starts at | Row 5 |
| Key columns | B: County Name · C: Partner Name · D: Latitude · E: Longitude · F: Health Facilities Supported |

### Publishing the Sheet as CSV (required for browser fetch)

1. Open the Google Sheet
2. Go to **File → Share → Publish to web**
3. Select the `Partners Directory` sheet
4. Choose **Comma-separated values (.csv)**
5. Click **Publish** and copy the URL
6. Replace the `CSV_URL` constant in `index.html` with this URL

> **Note:** The published CSV URL has the form:
> `https://docs.google.com/spreadsheets/d/{SHEET_ID}/gviz/tq?tqx=out:csv&sheet=Partners%20Directory`

---

## Core Features

### 1. Automatic Data Loading
- Fetch CSV from the published Google Sheet on page load — no user interaction required
- Display a `"Loading Kenya counties and partner data..."` message while fetching
- Show a clear, styled error banner if either the GeoJSON or CSV fetch fails
- Parse CSV rows starting at row 5 (skip header rows 1–4)

### 2. County Boundaries Layer (GeoJSON)
- Source: `https://raw.githubusercontent.com/iamckn/kenya-geojson/master/kenyan-counties.geojson`
- Style: light grey fill (`#F0F0F0`, opacity 0.3), grey border (`#666666`, weight 1)
- On hover: highlight county with a darker fill and display county name in a fixed label
- Ensure all 47 counties are visible at the initial zoom level

### 3. Partner Markers
- Plot one marker per valid row using `Latitude` and `Longitude` columns
- Skip rows with missing, null, or non-numeric coordinates — log skipped rows to the console
- Marker style: red circular markers, distinct from county polygons
- On hover (mouseover): show a Leaflet tooltip with:
  - **Partner Name**
  - **County**
  - **Health Facilities Supported**
- Close tooltip on mouseout

### 4. Map Configuration (Leaflet.js)
- Library: Leaflet.js v1.9.x via CDN
- Base tile: CartoDB Positron (`https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png`)
- Initial center: `[0.5, 38]`, zoom: `6`
- Attribution: OpenStreetMap + CartoDB

### 5. Performance
- Use Leaflet `MarkerClusterGroup` for datasets exceeding 50 markers
- Load GeoJSON and CSV in parallel using `Promise.all()`
- Render the county layer before markers to ensure correct z-order

---

## Output

- **Single file:** `index.html` — all JS, CSS, and configuration inline or via CDN
- **No build step** — open directly in any modern browser (Chrome, Firefox, Edge, Safari)
- **No server required** — works from the local filesystem (`file://`) once data URLs are configured

---

## Why County Boundaries Matter

County boundaries transform a simple dot map into an analytical tool by:

- Revealing geographic distribution patterns and coverage density
- Validating that partner-declared counties match their actual coordinates
- Surfacing coverage gaps in underserved counties
- Providing geopolitical context essential for Ministry of Health reporting

---

## Repository Structure

```
kenya-health-partners-map/
├── index.html          # Single deliverable — the complete map application
├── CLAUDE.md           # Project overview and context (this file)
├── SPECIFICATIONS.md   # Detailed technical specifications
└── README.md           # Setup and usage instructions
```

---

## Getting Started

```bash
git clone https://github.com/yourusername/kenya-health-partners-map.git
cd kenya-health-partners-map
# Open index.html in your browser
open index.html
```

> If fetching the CSV fails due to CORS, publish the sheet as described above and update `CSV_URL` in `index.html`.

---

## Implementation Sequence

Follow this order to build the project correctly. Each step depends on the previous.

### Step 1 — Project Scaffold
- Create `index.html` with HTML5 boilerplate
- Add `<title>` and a visible page heading
- Link Leaflet.js CSS and JS via CDN
- Add a `#map` div with `height: 100vh`
- Add a `#loading` overlay div with the loading message
- Add a `#error` div (hidden by default) for error messages

### Step 2 — Initialize the Leaflet Map
- Create the Leaflet map instance targeting `#map`
- Set center `[0.5, 38]` and zoom `6`
- Add the CartoDB Positron tile layer with attribution
- Verify the base map renders before adding data layers

### Step 3 — Fetch GeoJSON County Boundaries
- `fetch()` the Kenya counties GeoJSON from the GitHub raw URL
- On success, add a `L.geoJSON()` layer with the specified fill/border styles
- On hover, highlight the county and show its name
- On error, show the `#error` banner and halt marker loading

### Step 4 — Fetch and Parse Partner CSV
- Construct the published CSV URL from the Sheet ID
- `fetch()` the CSV; split into lines; skip the first 4 rows (header block)
- Parse each remaining row: extract columns B–F
- Validate `Latitude` and `Longitude` — skip and log invalid rows
- Store valid partners as an array of objects: `{ countyName, partnerName, lat, lng, facilities }`

### Step 5 — Plot Partner Markers
- Iterate over valid partner objects
- For each, create `L.circleMarker([lat, lng], { radius: 7, color: '#e63946', ... })`
- Bind a tooltip: `marker.bindTooltip(...)` showing Partner Name, County, Facilities Supported
- Add each marker to the map (or to a `MarkerClusterGroup` if using clustering)

### Step 6 — Parallel Loading with Promise.all
- Wrap Steps 3 and 4 in a `Promise.all([fetchGeoJSON(), fetchCSV()])`
- On resolution, execute Steps 3 and 5 sequentially (county layer first, then markers)
- On any rejection, display the `#error` banner with the specific failure message
- Hide the `#loading` overlay on success or failure

### Step 7 — Error Handling and UX Polish
- Validate that `Promise.all` catches both network errors and parse errors
- Add a retry button inside the `#error` banner that calls the load function again
- Ensure the `#loading` overlay is always removed, even on error
- Add console warnings for every skipped row with its row number and reason

### Step 8 — Testing and Validation
- Test with Chrome DevTools Network tab — verify both fetches succeed (200 OK)
- Simulate fetch failure by using an invalid URL — confirm the error banner appears
- Confirm all 47 counties render with visible borders
- Hover over 5+ markers and confirm tooltip content is accurate
- Test on mobile viewport (375px width) — map and tooltips should be usable
- Validate HTML with the W3C validator

---

## License

MIT License — see `LICENSE` file for details.
