# Kenya Health Partners Map — Technical Specifications

## 1. Data Source

### Google Sheet
| Property | Value |
|----------|-------|
| Sheet URL | `https://docs.google.com/spreadsheets/d/17uBPv58cH3hRDRk94vzTGjZmZFrCy5S5tBbhLB2TKMY/edit?usp=sharing` |
| Sheet name | `Partners Directory` |
| Header rows | 1–4 (skip entirely) |
| Data rows | Row 5 onwards |

### Column Mapping
| Column | Field | Notes |
|--------|-------|-------|
| A | `#` | Row number — ignore |
| B | `County Name` | Must match GeoJSON county names exactly (case-sensitive) |
| C | `Partner Name` | Display in tooltip |
| D | `Latitude` | Decimal degrees — validate as numeric, range −5 to 5 |
| E | `Longitude` | Decimal degrees — validate as numeric, range 33 to 42 |
| F | `Health Facilities Supported` | Display in tooltip |

### Fetch Endpoint (Google Sheets JSON API — no API key required)

```
https://docs.google.com/spreadsheets/d/{SHEET_ID}/gviz/tq?tqx=out:json&sheet=Partners%20Directory
```

**Full URL for this project:**
```
https://docs.google.com/spreadsheets/d/17uBPv58cH3hRDRk94vzTGjZmZFrCy5S5tBbhLB2TKMY/gviz/tq?tqx=out:json&sheet=Partners%20Directory
```

**Response format:** JSON-P. The response text is wrapped in `google.visualization.Query.setResponse({...});`. Strip this wrapper before parsing:

```javascript
const jsonText = responseText
  .replace(/^.*?google\.visualization\.Query\.setResponse\(/, '')
  .replace(/\);?\s*$/, '');
const data = JSON.parse(jsonText);
const rows = data.table.rows; // Array of row objects
```

Each row object has a `c` (cells) array. Each cell has a `v` (value) property:
- `row.c[1].v` → County Name (column B)
- `row.c[2].v` → Partner Name (column C)
- `row.c[3].v` → Latitude (column D)
- `row.c[4].v` → Longitude (column E)
- `row.c[5].v` → Health Facilities Supported (column F)

**Prerequisite:** The sheet must be shared as **"Anyone with the link → Viewer"** in Google Sheets. No publishing, no API key, and no OAuth flow required.

---

## 2. GeoJSON County Boundaries

| Property | Value |
|----------|-------|
| Source URL | `https://raw.githubusercontent.com/iamckn/kenya-geojson/master/kenyan-counties.geojson` |
| Format | GeoJSON (FeatureCollection) |
| Features | 47 county polygons |
| Name property | `properties.COUNTY_NAM` or `properties.name` — verify at runtime |

### Styling
| Property | Value |
|----------|-------|
| Fill color | `#F0F0F0` |
| Fill opacity | `0.3` |
| Border color | `#666666` |
| Border weight | `1px` |
| Border opacity | `1.0` |

### Hover State
| Property | Value |
|----------|-------|
| Fill color | `#BDD7EE` |
| Fill opacity | `0.6` |
| Border weight | `2px` |
| Label | County name displayed in a fixed map label (top-right corner) |

---

## 3. Mapping Library

| Property | Value |
|----------|-------|
| Library | Leaflet.js v1.9.4 |
| CSS CDN | `https://unpkg.com/leaflet@1.9.4/dist/leaflet.css` |
| JS CDN | `https://unpkg.com/leaflet@1.9.4/dist/leaflet.js` |
| Clustering (optional) | `leaflet.markercluster` via CDN — use if marker count > 50 |

### Map Initialization
```javascript
const map = L.map('map').setView([0.5, 38], 6);
```

### Base Tile Layer
```javascript
L.tileLayer('https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png', {
  attribution: '© <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors © <a href="https://carto.com/">CARTO</a>',
  subdomains: 'abcd',
  maxZoom: 19
}).addTo(map);
```

> CartoDB Positron is preferred over OSM Standard — it provides a clean, low-contrast base that keeps county boundaries and markers visually prominent.

---

## 4. Partner Markers

### Marker Style
```javascript
L.circleMarker([lat, lng], {
  radius: 7,
  fillColor: '#e63946',
  color: '#ffffff',
  weight: 1.5,
  opacity: 1,
  fillOpacity: 0.85
})
```

### Tooltip Content
Display on `mouseover`, hide on `mouseout`. Use `sticky: true` so the tooltip follows the cursor.

```javascript
marker.bindTooltip(`
  <strong>${partnerName}</strong><br/>
  County: ${countyName}<br/>
  Facilities: ${facilities}
`, { sticky: true });
```

### Data Validation Rules
A row is considered **invalid** and must be skipped (with a console warning) if any of the following are true:

| Condition | Action |
|-----------|--------|
| `Latitude` is empty or non-numeric | Skip, log: `Row N: missing or invalid latitude` |
| `Longitude` is empty or non-numeric | Skip, log: `Row N: missing or invalid longitude` |
| `Latitude` outside range −5 to 5 | Skip, log: `Row N: latitude out of Kenya bounds` |
| `Longitude` outside range 33 to 42 | Skip, log: `Row N: longitude out of Kenya bounds` |
| `Partner Name` is empty | Skip, log: `Row N: missing partner name` |

---

## 5. Data Loading

### Fetch Strategy
Load GeoJSON and Sheet data **in parallel** using `Promise.all()`. Render the county layer first, then markers.

```javascript
Promise.all([
  fetch(GEOJSON_URL).then(r => r.json()),
  fetch(SHEET_URL).then(r => r.text())
])
.then(([geojsonData, sheetText]) => {
  renderCounties(geojsonData);
  renderMarkers(parseSheetData(sheetText));
  hideLoading();
})
.catch(err => {
  showError(`Failed to load map data: ${err.message}`);
  hideLoading();
});
```

### Sheet Data Parsing

```javascript
function parseSheetData(responseText) {
  // Strip the JSON-P wrapper
  const jsonText = responseText
    .replace(/^.*?google\.visualization\.Query\.setResponse\(/, '')
    .replace(/\);?\s*$/, '');
  const data = JSON.parse(jsonText);

  // Skip header rows (indices 0–3 correspond to sheet rows 1–4)
  return data.table.rows
    .slice(4)
    .map((row, i) => ({
      rowIndex: i + 5,
      countyName:  row.c[1]?.v ?? '',
      partnerName: row.c[2]?.v ?? '',
      lat:         row.c[3]?.v,
      lng:         row.c[4]?.v,
      facilities:  row.c[5]?.v ?? ''
    }));
}
```

### Loading Sequence
1. Page loads → `#loading` overlay appears immediately
2. `Promise.all()` fires both fetches simultaneously
3. On success: render counties → render markers → hide `#loading`
4. On failure: show `#error` banner → hide `#loading`

---

## 6. UI Components

### Loading Overlay
```html
<div id="loading">Loading Kenya counties and partner data...</div>
```
- Position: fixed, full-screen, centered text
- Background: `rgba(255,255,255,0.85)`
- Z-index: above the map (`z-index: 1000`)

### Error Banner
```html
<div id="error" style="display:none;">
  <span id="error-message"></span>
  <button onclick="loadData()">Retry</button>
</div>
```
- Position: fixed top, full-width
- Background: `#f8d7da` (Bootstrap danger light) or equivalent red
- Includes a **Retry** button that re-invokes the load function

### Page Title
- Render a `<h1>` title above or overlaid on the map: `Kenya Health Partners Map`
- Include a brief subtitle: `Showing health partner locations across all 47 counties`

---

## 7. Output Requirements

| Requirement | Detail |
|-------------|--------|
| File format | Single `index.html` file |
| External dependencies | CDN links only (Leaflet, optionally MarkerCluster) |
| Local assets | None required |
| Build tools | None — no npm, webpack, or bundler |
| Browser compatibility | Chrome 90+, Firefox 88+, Edge 90+, Safari 14+ |
| File size | `index.html` should not exceed 200 KB excluding CDN resources |

---

## 8. Performance Guidelines

| Concern | Approach |
|---------|----------|
| Many markers | Use `L.markerClusterGroup()` if > 50 markers |
| Parallel loading | `Promise.all()` for GeoJSON + Sheet data |
| Layer order | Add GeoJSON layer before markers (correct z-order) |
| Tooltip performance | Use `L.tooltip` (not `L.popup`) — popups trigger reflow on open |
| GeoJSON simplification | If county boundaries load slowly, use a simplified GeoJSON (< 500 KB) |

---

## 9. Error Handling Matrix

| Error Scenario | User-Facing Behaviour | Developer Log |
|----------------|----------------------|---------------|
| GeoJSON fetch fails (404/network) | Red error banner: "Could not load county boundaries." + Retry button | `console.error` with full error |
| Sheet fetch fails (not shared / network error) | Red error banner: "Could not load partner data. Ensure the sheet is shared as 'Anyone with the link'." + Retry button | `console.error` with full error |
| Sheet JSON-P parse fails (unexpected format) | Red error banner: "Could not parse partner data." + Retry button | `console.error` with full error |
| Row with invalid coordinates | Row silently skipped; map continues loading | `console.warn("Row N: reason")` |
| Empty sheet (no data rows after row 4) | Markers layer is empty; counties still render; info message shown | `console.warn("No valid partner rows found")` |
| GeoJSON missing county name property | County renders without hover label | `console.warn("GeoJSON feature missing name property")` |

---

## 10. Implementation Sequence

Follow this sequence exactly. Do not skip steps or reorder them — each step validates the previous one before adding complexity.

### Phase 1 — HTML Scaffold and Map Base

**Step 1.1 — Create index.html boilerplate**
- HTML5 doctype, `<head>` with `charset=UTF-8` and `viewport` meta tag
- Link Leaflet CSS from CDN in `<head>`
- Add `<script>` for Leaflet JS at the bottom of `<body>`
- Set `body` and `html` to `margin: 0; padding: 0; height: 100%`

**Step 1.2 — Add UI structure**
- Add `<div id="map" style="height:100vh"></div>`
- Add `<div id="loading">Loading Kenya counties and partner data...</div>` — style as full-screen overlay
- Add `<div id="error" style="display:none"></div>` — style as top-fixed red banner
- Add `<h1>` title element (position: absolute, top-left, above map)

**Step 1.3 — Initialize Leaflet map**
- Instantiate `L.map('map')` with center `[0.5, 38]`, zoom `6`
- Add CartoDB Positron tile layer
- Open browser — confirm base map renders with no console errors

---

### Phase 2 — County Boundaries

**Step 2.1 — Fetch GeoJSON**
- Define `const GEOJSON_URL = 'https://raw.githubusercontent.com/iamckn/kenya-geojson/master/kenyan-counties.geojson'`
- `fetch(GEOJSON_URL)` on page load, parse as JSON
- Log the response to console — inspect the feature property that contains the county name (e.g., `COUNTY_NAM`, `name`, or `NAME`)

**Step 2.2 — Render county polygons**
- Create `L.geoJSON(data, { style: countyStyle })` with the specified fill/border values
- Add to map with `.addTo(map)`
- Confirm all 47 counties are visible at zoom 6

**Step 2.3 — Add county hover interaction**
- In `onEachFeature`, add `mouseover` and `mouseout` event listeners
- On `mouseover`: call `layer.setStyle(hoverStyle)` and update a fixed info control with the county name
- On `mouseout`: call `layer.setStyle(defaultStyle)` and clear the info control

---

### Phase 3 — Sheet Data Fetch and Parsing

**Step 3.1 — Construct the gviz URL**
- Define `const SHEET_URL` using the `gviz/tq?tqx=out:json` pattern with the Sheet ID and sheet name
- `fetch(SHEET_URL)`, parse as text, log the raw response to console to inspect the JSON-P wrapper structure

**Step 3.2 — Strip JSON-P wrapper and parse**
- Remove the `google.visualization.Query.setResponse(` prefix and `);` suffix from the response text
- `JSON.parse()` the inner text
- Navigate to `data.table.rows` and log the row count to confirm

**Step 3.3 — Map rows to partner objects**
- Slice rows from index 4 onwards (skipping the 4-row header block)
- Map each row to `{ rowIndex, countyName, partnerName, lat, lng, facilities }` using `row.c[n].v`
- Handle null cells with optional chaining (`row.c[n]?.v ?? ''`)

**Step 3.4 — Validate rows**
- Filter rows using the validation rules in Section 4
- Log each skipped row to `console.warn` with row number and reason
- Return only valid rows for marker rendering

---

### Phase 4 — Partner Markers

**Step 4.1 — Render circle markers**
- Iterate over valid partner rows
- Create `L.circleMarker([lat, lng], markerStyle)` for each
- Add each marker to the map

**Step 4.2 — Bind tooltips**
- Bind a tooltip to each marker using the content template from Section 4
- Verify hover behaviour in the browser — tooltip should appear on hover and disappear on mouseout

**Step 4.3 — Optional: Add marker clustering**
- Install Leaflet MarkerCluster via CDN
- Create a `L.markerClusterGroup()` layer
- Add all markers to the cluster group instead of directly to the map
- Add the cluster group to the map

---

### Phase 5 — Parallel Loading and Error Handling

**Step 5.1 — Combine fetches with Promise.all**
- Refactor GeoJSON and Sheet fetches into named async functions: `fetchGeoJSON()` and `fetchSheetData()`
- Wrap in `Promise.all([fetchGeoJSON(), fetchSheetData()])`
- In `.then()`: call render functions in order (counties first, then markers)
- In `.catch()`: display error banner with message

**Step 5.2 — Implement loading/error UI logic**
- `showLoading()`: make `#loading` visible
- `hideLoading()`: hide `#loading`
- `showError(message)`: populate `#error-message` and make `#error` visible
- Call `showLoading()` before `Promise.all()`, `hideLoading()` in both `.then()` and `.catch()`

**Step 5.3 — Add retry functionality**
- Wrap the entire load sequence in a named function `loadData()`
- The Retry button in the error banner calls `loadData()` on click
- `loadData()` hides the error banner, shows loading, then re-runs `Promise.all()`

---

### Phase 6 — Testing and QA

**Step 6.1 — Happy path testing**
- Open `index.html` in Chrome
- Confirm loading overlay appears, then disappears
- Confirm all 47 counties render with visible borders
- Hover over 5 different counties — confirm name label updates
- Hover over 5 different markers — confirm tooltip shows correct data

**Step 6.2 — Error path testing**
- Temporarily set `GEOJSON_URL` to an invalid URL — confirm error banner appears
- Temporarily set `SHEET_URL` to an invalid URL — confirm error banner appears
- Click Retry — confirm the map attempts to reload

**Step 6.3 — Data validation testing**
- Add a test row to the sheet with an empty latitude — confirm it is skipped (check console)
- Add a row with coordinates outside Kenya bounds — confirm it is skipped

**Step 6.4 — Cross-browser and responsive testing**
- Test in Chrome, Firefox, and Edge
- Resize to 375px width — confirm map fills screen and tooltips are readable

**Step 6.5 — Final validation**
- Run HTML through W3C Markup Validator
- Confirm no console errors on clean load
- Confirm `index.html` is under 200 KB

---

## 11. User Flow

```
User opens index.html
        │
        ▼
Loading overlay appears immediately
        │
        ▼
Promise.all fires: fetch GeoJSON + fetch Sheet data (parallel)
        │
   ┌────┴────┐
Success      Failure
   │              │
   ▼              ▼
Render counties   Show error banner
Render markers    with Retry button
Hide loading      Hide loading
   │
   ▼
User sees map with county boundaries + partner markers
   │
   ▼
User hovers county → county highlights, name label updates
   │
   ▼
User hovers marker → tooltip shows partner details
   │
   ▼
User zooms and pans freely
```
