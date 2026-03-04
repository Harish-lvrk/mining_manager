# ⛏️ Mining Manager — Backend Deep Dive (Lecture Notes)

> **Audience:** Developers / engineers wanting to understand exactly what each backend function does, why it exists, and how data flows through it.

---

## Overview — The Backend Package

```
backend/
├── __init__.py        ← makes it a package (empty)
├── cog.py             ← file processing: convert GeoTIFF → COG, read spatial metadata
├── stac_api.py        ← HTTP client: CRUD operations against the STAC API & file server helpers
├── stac_builder.py    ← data assembly: construct a valid STAC 1.0.0 item dict
└── titiler.py         ← tile service client: fetch band stats, build tile/preview URLs
```

Each file has **one job**. They are designed to be independent — `stac_builder.py` calls `titiler.py`, and `stac_api.py` is standalone. The frontend (`tab_mining.py`) orchestrates them all.

---

## 1. `backend/cog.py` — COG Conversion & Metadata Extraction

### What problem does it solve?

Raw GeoTIFFs uploaded by users are **not streaming-friendly**. TiTiler (the tile server) needs a **Cloud-Optimized GeoTIFF (COG)** — a special GeoTIFF layout where data is organized so you can request individual tiles without downloading the whole file. This file handles:

1. Converting any GeoTIFF → COG
2. Reading spatial metadata (bounding box, CRS/EPSG, bands, GSD)

---

### `convert_to_cog(input_path, output_path) → (bool, str)`

**Purpose:** Shell out to GDAL's `gdal_translate` tool to reformat the input GeoTIFF as a COG.

**Flow:**

```
input_path (raw .tif)
    │
    └── subprocess.run([
            "gdal_translate",
            "-of", "COG",          ← output format = Cloud-Optimized GeoTIFF
            "-co", "COMPRESS=LZW", ← lossless compression to reduce file size
            "-co", "OVERVIEWS=AUTO",← auto-generate zoom levels (pyramid overviews)
            str(input_path),
            str(output_path)
        ])
    │
    ├── returncode == 0 → return (True, "")   ← success
    └── returncode != 0 → return (False, stderr[:400])  ← failure with error msg
```

**Why `gdal_translate`?**  
Python's `rasterio` can read GeoTIFFs but rewriting as COG requires GDAL's native COG driver. `gdal_translate -of COG` is the standard, battle-tested approach. The `LZW` compression + `OVERVIEWS=AUTO` are standard COG best practices.

**Why subprocess?**  
GDAL's Python bindings (`gdal.Translate()`) exist but are harder to install. `subprocess.run` is simpler, portable, and gives full stderr for debugging.

**What `capture_output=True, text=True` does:**  
Captures stdout and stderr as strings (instead of printing to terminal). Lets us read `result.stderr` to show error messages in the UI.

---

### `read_metadata(filepath) → dict`

**Purpose:** Open the COG (or any GeoTIFF) with `rasterio` and extract all spatial metadata needed downstream.

**Flow:**

```
filepath (COG .tif)
    │
    └── rasterio.open(filepath) as src
         │
         ├── src.bounds       → native CRS bounding box (left, bottom, right, top)
         ├── src.crs          → native CRS object (e.g. EPSG:32644)
         ├── src.count        → number of bands (1=grayscale, 3=RGB, 4=RGBNIR)
         ├── src.dtypes       → data type per band (uint8, uint16, float32…)
         └── src.transform.a  → pixel size in native CRS units (ground sampling distance)
         │
         ├── transform_bounds(crs, "EPSG:4326", ...)
         │       → reproject bbox from native CRS → WGS84 lat/lon
         │       → result: [min_lon, min_lat, max_lon, max_lat]
         │
         └── transform_geom(crs.to_wkt(), "EPSG:4326", native_polygon)
                 → reproject the footprint polygon → WGS84
                 → result: GeoJSON geometry dict {"type":"Polygon","coordinates":[...]}
         │
         └── return {
               "bbox":       [min_lon, min_lat, max_lon, max_lat],   ← STAC bbox
               "geometry":   <GeoJSON Polygon>,                       ← STAC geometry
               "epsg":       32644,                                   ← CRS code
               "band_count": 3,                                       ← RGB
               "dtypes":     ["uint8", "uint8", "uint8"],
               "gsd":        0.5                                      ← 0.5 m/pixel
             }
```

**Why WGS84 (EPSG:4326)?**  
STAC specification requires all bounding boxes and geometries in WGS84 (decimal degrees). The native CRS of the image could be anything (UTM, State Plane, local systems), so we always reproject.

**Why `abs(src.transform.a)`?**  
`transform.a` is the pixel width in native CRS units. It can be negative if the raster is flipped. `abs()` ensures we get a positive GSD value.

**Why build `native_poly` first?**  
`transform_geom` works on a geometry dict, not on bounding box coordinates. We construct the bounding box as a Polygon in the native CRS, then reproject it.

---

## 2. `backend/titiler.py` — Tile Service Client

### What problem does it solve?

After a COG is saved and accessible via the file server, we need:

1. **Band statistics** (min, max, percentiles) → to correctly set contrast/brightness when displaying the image
2. **Tile URL** → a `/tilejson.json` endpoint that map libraries (Leaflet, MapLibre) can consume
3. **Preview URL** → a static PNG thumbnail of the whole image

This file talks to the **TiTiler server** (a FastAPI app running on port 8008) to get all of the above.

---

### `fetch_titiler_stats(file_url) → dict`

**Purpose:** Ask TiTiler for per-band pixel statistics of a COG, given its public HTTP URL.

**Flow:**

```
file_url = "http://10.50.0.170:8085/Documents/serverimages/mining/mancherial/survey1/survey1_cog.tif"
    │
    └── GET http://10.50.0.170:8008/cog/statistics?url=<file_url>
         │
         ├── 200 OK → return response JSON, e.g.:
         │     {
         │       "b1": { "min":0, "max":255, "mean":87.2,
         │               "percentile_2":12, "percentile_98":210 },
         │       "b2": { ... },
         │       "b3": { ... }
         │     }
         └── any error → log warning, return {}
```

**Why percentile_2 and percentile_98?**  
Raw min/max of satellite imagery are often extreme outliers (1 dark pixel, 1 hot pixel). Using the 2nd–98th percentile as the display range gives much better visual contrast — this is standard remote sensing practice called **histogram stretching**.

**Why `timeout=30`?**  
TiTiler has to read significant portions of the COG to compute statistics. Large files on network storage can be slow. 30 seconds gives enough headroom.

---

### `compute_rescale(stats, band_indices) → str`

**Purpose:** Turn the per-band stats dict into a TiTiler-compatible `rescale` query string.

**Flow:**

```
stats        = {"b1": {"percentile_2": 12, "percentile_98": 210}, "b2": {...}, "b3": {...}}
band_indices = [1, 2, 3]   ← for RGB image
    │
    └── for each index i in band_indices:
          key = "b{i}"      → "b1", "b2", "b3"
          lo  = stats["b1"]["percentile_2"]   → 12
          hi  = stats["b1"]["percentile_98"]  → 210
          parts.append("12,210")
    │
    └── join with "&rescale=" → "12,210&rescale=45,188&rescale=30,201"
```

**Why join with `&rescale=`?**  
TiTiler accepts multiple `rescale` parameters, one per band. When building a URL, `?rescale=12,210&rescale=45,188&rescale=30,201` means band 1 renders 12→210, band 2 renders 45→188, etc. This is per-band normalization — much better than single global rescale.

**Fallback:** If `stats` is empty (TiTiler unreachable), returns `"0,255"` — a safe default.

---

### `build_tile_url(file_url, bidx_qs, rescale) → str`

**Purpose:** Construct the `/tilejson.json` URL that a web map can load as a raster tile layer.

**Output:**

```
http://10.50.0.170:8008/cog/WebMercatorQuad/tilejson.json
    ?url=http://10.50.0.170:8085/.../survey1_cog.tif
    &bidx=1&bidx=2&bidx=3
    &rescale=12,210&rescale=45,188&rescale=30,201
    &nodata=0
```

- `WebMercatorQuad` = standard web map tile grid (same as Google Maps, OpenStreetMap)
- `bidx=1&bidx=2&bidx=3` = which bands to use for R, G, B display
- `nodata=0` = treat black pixels (value 0) as transparent

**What does TiTiler return at this URL?**  
A JSON document with a `tiles` array containing a template URL like `http://.../cog/{z}/{x}/{y}.png?...`. Leaflet/MapLibre use this to load individual map tiles on demand.

---

### `build_preview_url(file_url, bidx_qs, rescale) → str`

**Purpose:** Construct a URL that TiTiler uses to render the **entire image as a single PNG thumbnail**.

**Output:**

```
http://10.50.0.170:8008/cog/preview.png
    ?url=http://.../.../survey1_cog.tif
    &bidx=1&bidx=2&bidx=3
    &rescale=12,210&rescale=45,188&rescale=30,201
    &nodata=0
```

The app downloads this PNG and saves it locally as `preview.png` in the item folder — so it's available even if TiTiler is offline later.

---

## 3. `backend/stac_api.py` — STAC API HTTP Client

### What problem does it solve?

All metadata (collections, items) is stored in a **STAC API** server (using `pgstac` or similar). This file is a thin HTTP client layer over that REST API. It handles:

- Building correct URLs (internal vs public)
- Logging every request with timing
- Returning `(bool, error_str)` tuples for UI-friendly error handling

---

### Internal Helpers: `_get`, `_post`, `_put`, `_delete`

These are **private functions** (underscore prefix = convention for internal use).

```python
def _get(path: str, **params) -> requests.Response:
    url = f"{STAC_API_INTERNAL}{path}"   # e.g. http://localhost:8082/collections
    t0  = time.monotonic()               # start timer
    r   = requests.get(url, params=params or None, timeout=10)
    ms  = int((time.monotonic() - t0) * 1000)  # elapsed ms
    log.info("GET %s → %d (%dms)", path, r.status_code, ms)
    return r
```

**Why `STAC_API_INTERNAL` (localhost) vs `STAC_API_URL` (LAN IP)?**  
The app server and STAC API run on the same machine. Using `localhost` for actual HTTP calls is faster and avoids network routing. But URLs embedded in STAC item JSON (for external clients to use) must use the LAN IP so that other machines on the network can reach them.

**Why timed logging?**  
Every request logs its path, status code, and milliseconds. This makes it easy to spot slow operations in `logs/stac_manager.log` without a profiler.

**Why `params or None`?**  
`requests.get(url, params={})` appends a `?` to the URL. Passing `None` avoids the empty query string. `**params` captures keyword args like `limit=200` and passes them as query parameters.

---

### `local_to_url(filepath) → str`

**Purpose:** Convert an absolute local file path to a public file-server URL.

**Flow:**

```
filepath = Path("/home/hareesh/Documents/serverimages/mining/mancherial/s1/survey.tif")
FILE_SERVER_ROOT = Path("/home/hareesh")       ← from config
FILE_SERVER_URL  = "http://10.50.0.170:8085"

rel = filepath.relative_to(FILE_SERVER_ROOT)
    → "Documents/serverimages/mining/mancherial/s1/survey.tif"

return "http://10.50.0.170:8085/Documents/serverimages/mining/mancherial/s1/survey.tif"
```

**Why does this work?**  
The file server (a simple HTTP static server on port 8085) is rooted at the user's home directory. So the URL path mirrors the relative filesystem path.

---

### Collections CRUD

#### `fetch_collections() → list[dict]`

```
GET /collections
    │
    ├── response is a list  → use directly
    ├── response is a dict  → extract data["collections"]
    └── any error           → return []
```

The branching on `list` vs `dict` exists because different STAC API implementations return different formats (pgstac returns a wrapper object, others return a raw array).

#### `build_collection_payload(col_id, title, description, license_) → dict`

Constructs a minimal valid **STAC Collection** object:

- `stac_version: "1.0.0"` — required by spec
- `extent.spatial.bbox: [[-180,-90,180,90]]` — global extent (we let the API update this as items are added)
- `extent.temporal.interval: [[None, None]]` — open-ended time range

#### `api_create_collection(payload) → (bool, str)`

```
POST /collections  {payload}
    ├── 200/201 → (True, "")   ← created
    ├── 409     → (False, "409 — collection already exists.")
    └── other   → (False, "HTTP 422: ...")
```

Returns a tuple instead of raising exceptions → the UI shows `st.error(err)` without a stacktrace.

#### `api_update_collection(col_id, payload) → (bool, str)`

`PUT /collections/{col_id}` — replaces the collection metadata. Accepts 200, 201, or 204 (No Content).

#### `api_delete_collection(col_id) → (bool, str)`

`DELETE /collections/{col_id}` — permanently removes the collection and all its items from the STAC API.

---

### Items CRUD

#### `fetch_items(col_id, limit=200) → list[dict]`

```
GET /collections/{col_id}/items?limit=200
    │
    ├── response dict → extract data["features"]   ← GeoJSON FeatureCollection wrapper
    ├── response list → use directly
    └── error         → return []
```

STAC items are **GeoJSON Features**, so the API returns a `FeatureCollection` object. We extract the `features` array.

#### `api_push_item(col_id, item) → (bool, str)`

```
POST /collections/{col_id}/items  {stac_item}
    ├── 200/201 → (True, "")
    ├── 409     → (False, "409 — item already exists. Use a different Item ID.")
    └── other   → (False, "HTTP ...")
```

#### `api_delete_item(col_id, item_id) → (bool, str)`

`DELETE /collections/{col_id}/items/{item_id}` — removes a single item.

---

## 4. `backend/stac_builder.py` — STAC Item Assembler

### What problem does it solve?

A STAC 1.0.0 item is a **GeoJSON Feature** with a very specific structure and required fields. Assembling it manually in the UI code would be messy and error-prone. This file encapsulates all the STAC spec knowledge in one place.

> **Note:** In the mining workflow (`tab_mining.py`), the `_build_mining_item()` function in `frontend/tab_mining.py` is used instead of `stac_builder.py`. `stac_builder.py` is the older/generic builder used for non-mining COG workflows.

---

### `band_list(band_count) → list[dict]`

**Purpose:** Given the number of raster bands, return the `eo:bands` metadata array.

```python
band_count = 3 → [
    {"name": "Red",   "common_name": "red"},
    {"name": "Green", "common_name": "green"},
    {"name": "Blue",  "common_name": "blue"},
]

band_count = 4 → [
    {"name": "Blue",  "common_name": "blue"},
    {"name": "Green", "common_name": "green"},
    {"name": "Red",   "common_name": "red"},
    {"name": "NIR",   "common_name": "nir"},   ← Near-Infrared band
]

band_count = other → [{"name": "Band 1"}, {"name": "Band 2"}, ...]
```

The 4-band order is BGRNIR (not RGBNIR) — this follows the typical convention for multispectral satellite imagery (e.g. Sentinel-2, WorldView).

---

### `build_stac_item(item_id, collection, dt_str, metadata, file_url, stats, ...) → dict`

**Purpose:** Assemble the complete STAC item dictionary, ready to POST to the API.

**Flow:**

```
Step 1: Determine band display order
    band_count = 3 → bidx = [1, 2, 3]      ← RGB
    band_count ≥ 4 → bidx = [3, 2, 1]      ← Red=band3, Green=band2, Blue=band1 (BGRNIR → show as RGB)
    bidx_qs = "bidx=3&bidx=2&bidx=1"       ← query string for TiTiler

Step 2: Compute contrast-stretch rescale
    rescale = compute_rescale(stats, bidx)  → "12,210&rescale=45,188&rescale=30,201"

Step 3: Build URLs
    tile_url    = build_tile_url(file_url, bidx_qs, rescale)
    preview_url = build_preview_url(file_url, bidx_qs, rescale)

Step 4: Build properties dict
    {
      "datetime":  "2024-03-03T10:00:00Z",
      "gsd":       0.5,                       ← from cog.py metadata
      "proj:epsg": 32644,                     ← from cog.py metadata
      "eo:bands":  [...]                      ← from band_list()
      "title":     "...",                     ← optional
      "platform":  "...",                     ← e.g. "Sentinel-2"
      "instruments": ["MSI"],                 ← split from comma string
    }

Step 5: Assemble full STAC Feature
    {
      "type":           "Feature",
      "stac_version":   "1.0.0",
      "stac_extensions": ["eo/v1.1.0", "projection/v1.1.0"],
      "id":             item_id,
      "collection":     collection,
      "bbox":           [min_lon, min_lat, max_lon, max_lat],
      "geometry":       { "type": "Polygon", "coordinates": [...] },
      "links": [
        { "rel": "collection", "href": ".../collections/{collection}" },
        { "rel": "parent",     "href": ".../collections/{collection}" },
        { "rel": "root",       "href": ".../" },
        { "rel": "self",       "href": ".../collections/{collection}/items/{id}" },
      ],
      "assets": {
        "visual":  { "href": file_url,    "type": "image/tiff;...", "roles": ["data","visual"] },
        "tiles":   { "href": tile_url,    "type": "application/json", "roles": ["tiles"] },
        "preview": { "href": preview_url, "type": "image/png", "roles": ["overview"] },
      },
      "properties": { ... }
    }
```

**Why `stac_extensions`?**  
STAC extensions are optional add-ons to the core spec. Adding the URIs declares that this item uses `eo:` (electro-optical) fields like `eo:bands` and `proj:` fields like `proj:epsg`. Validators and downstream tools use these to know which extra fields to expect.

**Why 4 links?**  
The STAC spec requires `self`, `root`, `parent`, and `collection` links for full navigability. A STAC browser can start at any item and walk up to the collection or root.

---

## End-to-End Call Chain (Mining Workflow)

```
[User clicks "Save & Push"]
    │
    ├── cog.py:convert_to_cog()        → convert .tif → COG
    ├── cog.py:read_metadata()         → bbox, epsg, gsd, band count
    │
    ├── [Files copied to server folder]
    │
    ├── titiler.py:fetch_titiler_stats() → per-band percentile stats
    ├── titiler.py:compute_rescale()     → "12,210&rescale=..."
    ├── titiler.py:build_tile_url()      → tilejson.json URL
    ├── titiler.py:build_preview_url()   → preview.png URL
    │
    ├── [Preview PNG downloaded from TiTiler and saved to disk]
    │
    ├── [GeoJSONs + analytics.json written to disk]
    │
    ├── tab_mining._build_mining_item()  → full STAC Feature dict
    │
    └── stac_api.api_push_item()         → POST to STAC API
```

---

## Key Design Decisions (for your lecture)

| Decision                                       | Reason                                                                                                              |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| All functions return `(bool, str)`             | UI can show `st.error(msg)` without catching exceptions at the UI layer                                             |
| Separate `STAC_API_INTERNAL` vs `STAC_API_URL` | App uses localhost (fast), STAC items embed LAN IP (accessible by all clients)                                      |
| `gdal_translate` via subprocess                | More portable than GDAL Python bindings; gives full stderr for debugging                                            |
| COG format required                            | TiTiler can only efficiently serve tiles from COGs — raw TIFFs would require reading the full file per tile request |
| Per-band rescale with percentiles              | Histogram stretching with p2/p98 is standard remote sensing practice for best visual output                         |
| Band stats fetched AFTER file is on server     | TiTiler reads the file via HTTP URL — the file must exist and be accessible before stats can be fetched             |
