# CropBrain Maharashtra — Development Roadmap

## ✅ v1.0 — SHIPPED (tag: v1.0-cropbrain)
Live: https://gattooze.github.io/biobrain/cropbrain.html

- Map-first split layout, full-height interactive map
- 36 districts, 16 crops (area + production), 2021-22 APY data
- 50 power boiler sites + 112 industrial boiler sites (162 total)
- Location search (Nominatim + instant boiler/district match)
- Radius filter (25/50/100/150 km) with Site Assessment panel
- Crop residue biomass estimates (MT/yr by district within radius)
- Pellet co-firing demand at 5% blend for all boiler sites
- Layer toggles for power + industrial on map

---

## 🔜 v2.0 — NEXT PHASE

### 2A · Production as Primary Metric (quick win)
- Swap choropleth from area → production MT as default view
- Update legend label: "PRODUCTION (MT)" vs "AREA (Ha)"
- Update Site Assessment: show production-based residue, not area × factor
- Update crop detail panel: lead with production, show yield (MT/Ha)
- Residue factor recalibration: straw ratio × production (not area × factor)

### 2B · Road Distance for Boilers (OSRM — free, no API key)
- Replace haversine straight-line distance with actual road km
- API: https://router.project-osrm.org/route/v1/driving/{lng1},{lat1};{lng2},{lat2}?overview=false
- Returns: road_km + drive_time_hours
- Implementation:
  - On site assessment trigger: batch-call OSRM for all boilers within 2× radius
  (haversine pre-filter → OSRM precise distance)
  - Cache results in sessionStorage to avoid repeat calls
  - Show: "X km road | Y hrs" instead of "X km" in assessment panel
  - Rate limit: stagger calls 100ms apart, max 10 parallel
- Display: road distance + travel time in boiler customer rows
- Note: OSRM public server is free but throttled; for production use
  self-host or switch to OpenRouteService (free 2K req/day with API key)

### 2C · Satellite Crop Mapping (Track 2 — Sentinel-2 via GEE)
- Data source: Google Earth Engine (free service account)
  - Sentinel-2 L2A: 10m resolution, monthly composites
  - Crop classification: NDVI + EVI time-series → Random Forest classifier
  - Training labels: ICAR/FASAL crop calendar per district
- Output: per-pixel crop type raster → district-level production estimates
- Integration: compare GEE estimates vs. APY ground truth in crop detail panel
- Show: "Satellite estimate" vs "Official APY" with confidence band
- Implementation steps:
  1. Set up GEE service account → credentials JSON
  2. Write GEE Python script (crop_classify.py) → export district crop maps
  3. Host output as Cloud Optimized GeoTIFF (COG) on GitHub Releases
  4. Load in Leaflet via georaster-layer-for-leaflet
- Status: DEFERRED (requires GEE service account setup)

### 2D · Mahabhumi Land Records Integration (background/silent)
- API: https://api.mahabhumi.gov.in/method/bE8vVXV6dTRGZk1CaUMwbXd0QTJMZz09
- Endpoint type: POST with lat/lon + village/gat number → returns ROR
  (Record of Rights — owner name, survey no., land type, area in guntha)
- ROR fields of interest:
  - Owner name (मालक नाव)
  - Survey / Gat number
  - Land area (hectares)
  - Land type (jirayat/bagayat/padkar)
  - Encumbrances / mortgage status
- Implementation plan:
  - When user pins a location: silently call Mahabhumi API in background
  - Store response in app state (do NOT display yet)
  - Persist to localStorage keyed by lat/lng
  - Future v3 feature: show land availability overlay when evaluating a site
- Auth: needs API key from mahabhumi.gov.in (register at portal)
  URL pattern appears to be base64-encoded endpoint slug

---

## 🔮 v3.0 — FUTURE

### 3A · Land Parcel Intelligence
- Display Mahabhumi data: show owner, area, land type in site assessment
- Highlight "available" agricultural land (no encumbrance, jirayat type)
- Estimate acquisition cost based on district circle rates (stamp duty dept)

### 3B · Logistics Optimizer
- Given a selected site: optimize pellet plant catchment
  - Which crops to source (by road distance to district centroid)
  - Which boilers to target (by road distance, prioritize larger ones)
  - Show supply-demand balance: residue available vs. boiler demand
  - Suggest optimal plant capacity (TPD) based on catchment

### 3C · Real-time Data Feeds
- APMC mandi prices for crop residue (agmarknet.gov.in API)
- Coal import prices (monthly update) for pellet price parity calc
- MNRE co-firing compliance tracker (quarterly POSOCO data)

---

## TECHNICAL NOTES

### OSRM Road Routing (ready to use)
```js
async function getRoadDistance(lat1, lng1, lat2, lng2) {
  const url = `https://router.project-osrm.org/route/v1/driving/${lng1},${lat1};${lng2},${lat2}?overview=false`;
  const r = await fetch(url);
  const d = await r.json();
  if (d.code !== 'Ok') return null;
  return {
    road_km: (d.routes[0].distance / 1000).toFixed(1),
    drive_hrs: (d.routes[0].duration / 3600).toFixed(1)
  };
}
// Tested: Pune(18.52,73.86) → Chandrapur(19.96,79.30) = 753 km road, 9.1 hrs ✓
```

### Mahabhumi API (needs key + POST body investigation)
```
Base URL: https://api.mahabhumi.gov.in/
Endpoint: /method/bE8vVXV6dTRGZk1CaUMwbXd0QTJMZz09
Method: POST (suspected)
Input params: lat, lon, gat_no (village survey number)
Returns: ROR (Record of Rights) — owner, area, land type
Auth: API key required (register at mahabhumi.gov.in)
```

### Sentinel-2 Crop Classification (deferred — needs GEE)
```python
# GEE crop classification sketch (Python API)
import ee
ee.Initialize()
s2 = ee.ImageCollection('COPERNICUS/S2_SR') \
    .filterDate('2023-10-01', '2023-12-31') \
    .filterBounds(maharashtra_geometry) \
    .median()
ndvi = s2.normalizedDifference(['B8','B4']).rename('NDVI')
# → train classifier on ICAR crop calendar points
# → export as COG to Cloud Storage
```
