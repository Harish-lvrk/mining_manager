# 🛰️ Real STAC Item — `2019_adani` (kgf collection) — Explained Section by Section

## The Raw JSON (Fetched Live from the STAC API)

```json
{
  "id": "2019_adani",
  "bbox": [79.315289, 19.012375, 79.348503, 19.044236],
  "type": "Feature",
  "links": [
    {
      "rel": "collection",
      "type": "application/json",
      "href": "http://localhost:8082/collections/kgf"
    },
    {
      "rel": "parent",
      "type": "application/json",
      "href": "http://localhost:8082/collections/kgf"
    },
    {
      "rel": "root",
      "type": "application/json",
      "href": "http://localhost:8082/"
    },
    {
      "rel": "self",
      "type": "application/geo+json",
      "href": "http://localhost:8082/collections/kgf/items/2019_adani"
    }
  ],
  "assets": {
    "tiles": {
      "href": "http://10.50.0.170:8008/cog/WebMercatorQuad/tilejson.json?url=http://10.50.0.170:8085/Documents/serverimages/2019_Adani_cog.tif&bidx=3&bidx=2&bidx=1&rescale=19,198",
      "type": "application/json",
      "roles": ["tiles"],
      "title": "TiTiler RGB Tile Service"
    },
    "visual": {
      "href": "http://10.50.0.170:8085/Documents/serverimages/2019_Adani_cog.tif",
      "type": "image/tiff; application=geotiff; profile=cloud-optimized",
      "roles": ["data", "visual"],
      "title": "COG Image"
    },
    "preview": {
      "href": "http://10.50.0.170:8008/cog/preview.png?url=http://10.50.0.170:8085/Documents/serverimages/2019_Adani_cog.tif&bidx=3&bidx=2&bidx=1&rescale=19,198",
      "type": "image/png",
      "roles": ["overview", "thumbnail"],
      "title": "RGB Preview Image"
    }
  },
  "geometry": {
    "type": "Polygon",
    "coordinates": [
      [
        [79.315289, 19.012375],
        [79.348503, 19.012375],
        [79.348503, 19.044236],
        [79.315289, 19.044236],
        [79.315289, 19.012375]
      ]
    ]
  },
  "collection": "kgf",
  "properties": {
    "gsd": 2.86,
    "title": "kgf_2019",
    "datetime": "2026-02-27T11:13:18Z",
    "eo:bands": [
      { "name": "Blue", "common_name": "blue" },
      { "name": "Green", "common_name": "green" },
      { "name": "Red", "common_name": "red" },
      { "name": "NIR", "common_name": "nir" }
    ],
    "proj:epsg": 3857
  },
  "stac_version": "1.0.0",
  "stac_extensions": [
    "https://stac-extensions.github.io/eo/v1.1.0/schema.json",
    "https://stac-extensions.github.io/projection/v1.1.0/schema.json"
  ]
}
```

---

## Section-by-Section Breakdown

---

### 1. `"id"` — Item Name

```json
"id": "2019_adani"
```

- The **unique identifier** of this survey snapshot within the collection.
- Convention here: `{year}_{mine_name}` = 2019 survey of the Adani mine.
- Must be unique within its collection (`kgf`). You cannot have two items with the same ID in the same collection.
- Used in the URL: `.../collections/kgf/items/2019_adani`

---

### 2. `"type"` — GeoJSON Type

```json
"type": "Feature"
```

- STAC items are built on top of **GeoJSON** — an open standard for geographic data as JSON.
- `"Feature"` = GeoJSON's word for "one geographic object with attributes."
- This field tells any GeoJSON-aware tool (like Leaflet, QGIS, PostGIS) that this is a valid spatial feature.
- Think of it like the class declaration: "this object is a geographic Feature."

---

### 3. `"bbox"` — Bounding Box

```json
"bbox": [79.315289, 19.012375, 79.348503, 19.044236]
```

Always 4 numbers in this order:

```
[min_longitude, min_latitude, max_longitude, max_latitude]
      ↑               ↑              ↑              ↑
   west edge       south edge     east edge      north edge
  (79.315289°E)  (19.012375°N)  (79.348503°E)  (19.044236°N)
```

**What this means on a map:**

```
North 19.044°  ┌─────────────────────────────────┐
               │                                 │
               │   Adani KGF Mine Area           │
               │   in Chandrapur region,         │
               │   Maharashtra                   │
               │                                 │
South 19.012°  └─────────────────────────────────┘
            West 79.315°                     East 79.348°
```

**Width of area:** 79.348 - 79.315 = 0.033° longitude ≈ **3.3 km**
**Height of area:** 19.044 - 19.012 = 0.032° latitude ≈ **3.5 km**

So the image covers roughly a **3.3 km × 3.5 km** patch of the mining area.

**Why is bbox important?**

- Used for **spatial search**: "give me all items that intersect this bounding box"
- Web maps use it to **zoom/pan** to the item's location automatically
- Faster to compute than checking the full polygon geometry

---

### 4. `"geometry"` — Exact Footprint

```json
"geometry": {
    "type": "Polygon",
    "coordinates": [[
        [79.315289, 19.012375],   ← bottom-left corner
        [79.348503, 19.012375],   ← bottom-right corner
        [79.348503, 19.044236],   ← top-right corner
        [79.315289, 19.044236],   ← top-left corner
        [79.315289, 19.012375]    ← back to start (polygon must close)
    ]]
}
```

- The **exact geographic boundary** of the image, expressed as a polygon in WGS84 (GPS coordinates, decimal degrees).
- For a rectangular raster image, this is always a 5-point closed rectangle (first point = last point to "close" the polygon).
- For real satellite images with rotation or warping, the polygon can be non-rectangular.

**bbox vs geometry:**
| | `bbox` | `geometry` |
|---|---|---|
| Shape | Always a rectangle | Can be any polygon |
| Use | Fast spatial filtering | Exact overlap calculation |
| Size | 4 numbers | Many coordinate pairs |

Both are in **EPSG:4326** (WGS84 lat/lon) — required by STAC spec so any map can use it regardless of the image's native projection.

---

### 5. `"collection"` — Parent Collection

```json
"collection": "kgf"
```

- Tells you which collection this item belongs to: `kgf` (Kothagudem / KGF mining area).
- This is the **foreign key** linking the item to its parent.
- The same image ID (`2019_adani`) exists in `collection_1` too — they're different items, same name.

---

### 6. `"stac_version"` and `"stac_extensions"`

```json
"stac_version": "1.0.0",
"stac_extensions": [
    "https://stac-extensions.github.io/eo/v1.1.0/schema.json",
    "https://stac-extensions.github.io/projection/v1.1.0/schema.json"
]
```

**`stac_version`:** The specification version. Like a contract version number — tools know how to parse this.

**`stac_extensions`:** STAC core only defines basic fields. Extensions add domain-specific fields:

| Extension URL       | What it adds                       | Used fields here |
| ------------------- | ---------------------------------- | ---------------- |
| `eo/v1.1.0`         | Electro-Optical (satellite) fields | `eo:bands`       |
| `projection/v1.1.0` | Coordinate system fields           | `proj:epsg`      |

Without declaring extensions, you cannot use `eo:bands` or `proj:epsg` — validators would reject them.
Think of extensions like **"I am using these add-on features, so extra fields are expected."**

---

### 7. `"properties"` — Descriptive Metadata

```json
"properties": {
    "gsd": 2.86,
    "title": "kgf_2019",
    "datetime": "2026-02-27T11:13:18Z",
    "eo:bands": [...],
    "proj:epsg": 3857
}
```

#### `"gsd"` — Ground Sampling Distance

```
"gsd": 2.86
```

- **2.86 metres per pixel**
- Every pixel in this image represents a 2.86 m × 2.86 m patch on the ground.
- At 2.86 m GSD you can see trucks, equipment, roads, but not individual people.
- For comparison: Google Maps satellite = ~0.5 m GSD, Sentinel-2 = 10 m GSD.

#### `"title"` — Human-readable Name

```
"title": "kgf_2019"
```

- A readable label. Just a string for display in browsers and dashboards.

#### `"datetime"` — Capture Timestamp

```
"datetime": "2026-02-27T11:13:18Z"
```

- ISO 8601 format: `YYYY-MM-DDTHH:MM:SSZ` (the `Z` = UTC timezone)
- **Note:** This is when the item was _registered_ (Feb 27 2026), not when the image was _captured_ (2019).
- The file is named `2019_Adani` but the datetime was set to upload time. Ideally this should be the actual capture date.

#### `"eo:bands"` — Band Description

```json
"eo:bands": [
    {"name": "Blue",  "common_name": "blue"},
    {"name": "Green", "common_name": "green"},
    {"name": "Red",   "common_name": "red"},
    {"name": "NIR",   "common_name": "nir"}
]
```

- Tells you the image has **4 bands**: Blue, Green, Red, NIR
- `common_name` is a standardised vocabulary (blue, green, red, nir, swir, etc.) — tools use this to automatically pick the right bands for NDVI, true-color display, etc.
- Array order = Band 1 is Blue, Band 2 is Green, Band 3 is Red, Band 4 is NIR
- This is a **4-band multispectral image** — richer than what your phone camera captures.

#### `"proj:epsg"` — Native Projection

```
"proj:epsg": 3857
```

- **EPSG:3857 = Web Mercator** — the same projection used by Google Maps, OpenStreetMap, TiTiler.
- This means the COG file is already in Web Mercator.
- No re-projection needed when displaying on web maps → **faster rendering**.
- Note: `bbox` and `geometry` are still in WGS84 (4326) even though the file is in 3857. STAC always stores spatial footprints in WGS84.

---

### 8. `"links"` — Navigation Links

```json
"links": [
    {"rel": "collection", "href": ".../collections/kgf"},
    {"rel": "parent",     "href": ".../collections/kgf"},
    {"rel": "root",       "href": ".../"},
    {"rel": "self",       "href": ".../collections/kgf/items/2019_adani"}
]
```

These make the STAC API **fully navigable** — like HTML hyperlinks for data.

| `rel` value  | What it points to       | Why useful                                   |
| ------------ | ----------------------- | -------------------------------------------- |
| `self`       | This item's own URL     | Bookmarkable, shareable direct link          |
| `collection` | The parent collection   | Click up to see all items in `kgf`           |
| `parent`     | Same as collection here | For nested catalogs, could be different      |
| `root`       | The API root            | Click all the way up to list all collections |

A STAC browser (like the STAC Browser app) uses these links to let you click through the hierarchy: Root → Collection → Item, like a file browser.

---

### 9. `"assets"` — The Actual Data Files

This is the most important section — **where the actual files live**.

```json
"assets": {
    "visual": { ... },
    "tiles":  { ... },
    "preview":{ ... }
}
```

Each asset has:

- `"href"` — the URL to access it
- `"type"` — MIME type (what kind of file it is)
- `"roles"` — what the file is for
- `"title"` — human-readable label

#### Asset 1: `"visual"` — The Raw COG File

```json
"visual": {
    "href": "http://10.50.0.170:8085/Documents/serverimages/2019_Adani_cog.tif",
    "type": "image/tiff; application=geotiff; profile=cloud-optimized",
    "roles": ["data", "visual"],
    "title": "COG Image"
}
```

- **The actual GeoTIFF file** served directly from the file server.
- `roles: ["data", "visual"]` = this is both the primary data and the visual layer.
- A GIS tool (QGIS, ArcGIS) can load this URL directly and open the full image.
- `profile=cloud-optimized` in the MIME type declares it is a COG.

#### Asset 2: `"tiles"` — The Tile Service URL

```json
"tiles": {
    "href": "http://10.50.0.170:8008/cog/WebMercatorQuad/tilejson.json
             ?url=http://10.50.0.170:8085/Documents/serverimages/2019_Adani_cog.tif
             &bidx=3&bidx=2&bidx=1
             &rescale=19,198",
    "type": "application/json",
    "roles": ["tiles"],
    "title": "TiTiler RGB Tile Service"
}
```

Breaking down the URL parameters:

```
Base:    http://10.50.0.170:8008/cog/WebMercatorQuad/tilejson.json
                                      └── tile grid standard (Google Maps compatible)

?url=    http://10.50.0.170:8085/.../2019_Adani_cog.tif
         └── tells TiTiler which COG file to tile

&bidx=3  → use Band 3 (Red)   as the Red   display channel
&bidx=2  → use Band 2 (Green) as the Green display channel
&bidx=1  → use Band 1 (Blue)  as the Blue  display channel
           └── 3,2,1 order = true-color RGB from a BGRN image

&rescale=19,198
         └── stretch pixel values: 19 → black (0), 198 → white (255)
             (single global rescale, not per-band — this was an earlier item)
```

When a web map loads this URL it gets back a JSON like:

```json
{
  "tiles": [
    "http://10.50.0.170:8008/cog/tiles/WebMercatorQuad/{z}/{x}/{y}.png?url=..."
  ],
  "minzoom": 12,
  "maxzoom": 20,
  "bounds": [79.315, 19.012, 79.348, 19.044]
}
```

The `{z}/{x}/{y}` template = web map tile coordinates. Leaflet/MapLibre fill these in automatically as you pan and zoom.

#### Asset 3: `"preview"` — The Thumbnail PNG

```json
"preview": {
    "href": "http://10.50.0.170:8008/cog/preview.png
             ?url=http://10.50.0.170:8085/Documents/serverimages/2019_Adani_cog.tif
             &bidx=3&bidx=2&bidx=1
             &rescale=19,198",
    "type": "image/png",
    "roles": ["overview", "thumbnail"],
    "title": "RGB Preview Image"
}
```

- Same parameters as tiles, but returns **one single PNG** of the entire image (downscaled to thumbnail size).
- Used for: quick preview in browsers, dashboards, search results.
- `roles: ["overview", "thumbnail"]` = the role tells clients "use this for thumbnail display, not for analysis."

---

## Visual Map of the Complete Item

```
2019_adani (STAC Item)
│
├── IDENTITY
│     id:          "2019_adani"
│     type:        "Feature"           ← GeoJSON
│     stac_version: "1.0.0"
│     collection:  "kgf"
│
├── WHERE (on Earth)
│     bbox:     [79.315, 19.012, 79.348, 19.044]   ← rectangle
│     geometry: Polygon with 5 coordinates          ← exact shape
│
├── WHAT (metadata about the image)
│     properties:
│         datetime:   "2026-02-27T..."   ← when registered
│         title:      "kgf_2019"
│         gsd:        2.86               ← 2.86 m/pixel
│         proj:epsg:  3857               ← Web Mercator projection
│         eo:bands:   [Blue, Green, Red, NIR]
│
├── NAVIGATE (links to walk the API)
│     self       → this item's URL
│     collection → kgf collection
│     parent     → kgf collection
│     root       → API root
│
├── EXTENSIONS (extra field declarations)
│     eo/v1.1.0        → I use eo:bands
│     projection/v1.1.0 → I use proj:epsg
│
└── DATA (actual files)
      assets:
          visual  → COG file (raw download via file server)
          tiles   → TiTiler tile service (for web maps)
          preview → PNG thumbnail (for dashboards)
```
