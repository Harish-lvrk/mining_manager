# ⛏️ Mining Manager — Full-Stack Deep Dive (Frontend + Backend Together)

This document explains **how the user's every action in the UI maps to backend logic**, showing exactly what runs, in what order, and why.

---

## The Big Picture

```
BROWSER (user)
    │  click / type / upload
    ▼
app.py              ← Page shell: config, dark mode, service health dots
    │  imports render_mining_tab()
    ▼
frontend/tab_mining.py   ← All UI logic (3 tabs)
    │  calls backend functions
    ├──► backend/stac_api.py   ← HTTP to STAC REST API
    ├──► backend/cog.py        ← File processing (GDAL + rasterio)
    ├──► backend/titiler.py    ← HTTP to TiTiler tile server
    └──► backend/stac_builder.py ← Assembles STAC JSON
```

Everything starts from `app.py → render_mining_tab()` which renders 3 tabs:

- **Tab 1 — Mining Areas**: browse & create collections
- **Tab 2 — Create Survey Item**: upload GeoTIFF, paste GeoJSONs, push to STAC
- **Tab 3 — Browse Items**: view saved survey items

---

## How Streamlit Works (Critical to Understand Everything Below)

> Streamlit **re-runs the entire Python script from top to bottom** on every user interaction (click, type, file upload). There is NO persistent server state between runs — only `st.session_state` survives between reruns.

This is why:

- COG conversion results are cached in `st.session_state` (so conversion doesn't re-run on every keystroke)
- Confirmation dialogs use `st.session_state[f"confirm_delete_{id}"] = True`
- Tab selection state is stored in `st.session_state["mining_active_tab"]`

---

## STEP 0 — App Startup (`app.py`)

**What the user sees:** The page loads with a header, dark mode toggle, and 3 status dots.

**What runs:**

```
app.py starts
    │
    ├── st.set_page_config(...)         → title, icon, wide layout
    │
    ├── Dark mode setup
    │     st.session_state["dark_mode"] = False (default)
    │     CSS variables (_BG, _CARD, _HEAD, _MUTED, _BORD, _ACC) set accordingly
    │     st.markdown(f"<style>...</style>")  → injects CSS into page
    │
    ├── Service status check (for each of 3 services)
    │     _ping(STAC_API_URL)    → requests.get(url, timeout=3).status_code < 500
    │     _ping(TITILER_URL)     → same
    │     _ping(FILE_SERVER_URL) → same
    │     Renders: 🟢 Operational  or  🔴 Unreachable
    │
    └── from frontend.tab_mining import render_mining_tab
          render_mining_tab()   → hands off to the tab UI
```

**Backend involved:** None for the tab render itself. Only `requests.get()` for the 3 health pings (done inline in `app.py`).

---

## TAB 1 — Mining Areas (Collections)

### User Action: Page loads / Tab 1 is active

**Function:** `_render_collections_section()`

```
_render_collections_section()
    │
    ├── _mining_collections()
    │     │
    │     ├── _mining_collection_ids()
    │     │     MINING_ROOT = ~/Documents/serverimages/mining/
    │     │     → list all subdirectories in MINING_ROOT
    │     │       e.g. ["mancherial", "bellary", "kothagudem"]
    │     │
    │     └── fetch_collections()          [backend/stac_api.py]
    │           GET http://localhost:8082/collections
    │           → returns all STAC collections
    │           → filters: only keep ones whose "id" is in the local folders list
    │
    └── For each collection: render a card showing
          - title, id, description
          - count of items (calls fetch_items(cid) for each!)
          - "Browse Items →" button
          - 🗑️ delete button
```

**Why filter by local folders?**
The STAC API may contain non-mining collections (COG items, test data, etc.).
Only collections that have a folder under `MINING_ROOT` are "mining" collections.
This folder is created when you create a mining area (see below).

---

### User Action: Fill form → Click "Create Mining Area"

**Function:** `_render_create_collection_form()`

```
User fills: Area ID = "mancherial", Area Name = "Mancherial Coal Mines"
User clicks ✅ Create Mining Area
    │
    ├── Validation: col_id and col_title must not be empty
    ├── col_id = "mancherial".strip().lower().replace(" ", "-")
    │
    ├── build_collection_payload(col_id, col_title, col_desc, col_lic)
    │     [backend/stac_api.py]
    │     Constructs minimal STAC Collection JSON:
    │     {
    │       "type": "Collection",
    │       "id": "mancherial",
    │       "stac_version": "1.0.0",
    │       "title": "Mancherial Coal Mines",
    │       "description": "...",
    │       "license": "proprietary",
    │       "links": [],
    │       "extent": { "spatial": {"bbox": [[-180,-90,180,90]]},
    │                   "temporal": {"interval": [[null,null]]} }
    │     }
    │
    ├── api_create_collection(payload)
    │     [backend/stac_api.py]
    │     POST http://localhost:8082/collections  ← sends the JSON
    │     201 Created → (True, "")
    │     409 Conflict → (False, "409 — collection already exists.")
    │
    ├── If success:
    │     MINING_ROOT / "mancherial"  → mkdir (creates local folder)
    │     st.success("✅ Mining area mancherial created!")
    │     st.rerun()   ← reruns the whole script → card appears in grid
    │
    └── If fail: st.error(err)
```

---

### User Action: Click 🗑️ → Confirm → Delete Collection

```
Click 🗑️
    → st.session_state["mine_confirm_del_col_mancherial"] = True
    → st.rerun() shows warning + Yes/No buttons

Click ✅ Yes
    → api_delete_collection("mancherial")
          DELETE http://localhost:8082/collections/mancherial
          200/204 → (True, "")
    → st.success → st.rerun()
```

> **Note:** This only deletes from STAC API. The local folder (`MINING_ROOT/mancherial/`) is **NOT** deleted automatically — this is intentional to avoid accidentally destroying files.

---

## TAB 2 — Create Survey Item

This is the most complex tab. It has 4 numbered steps.

### Full Flow Diagram

```
Step 1: Select Mining Area (dropdown)
    └── auto_region_id = index of selected area + 1

Step 2: Upload / path GeoTIFF
    ├── Path mode  → read from disk directly
    └── Upload mode → stream to temp file

    Either way:
    └── cog.py:convert_to_cog()    → gdal_translate → COG temp file
    └── cog.py:read_metadata()     → bbox, epsg, gsd, band_count
    └── Cache result in st.session_state

Step 4: Analytics JSON (text area)
    └── Validate: json.loads(text)

Step 3: GeoJSON Layers (7 text areas)
    └── Validate each: _validate_geojson_text()

[Click Preview] → Build STAC item JSON → st.json(stac_item)
[Click Save]    → Full save pipeline (see below)
```

---

### Step 2a — Path Mode (large files)

**User enters:** `/home/hareesh/Documents/survey/mine_jan.tif`

```
p = Path("/home/hareesh/Documents/survey/mine_jan.tif")
    │
    ├── p.exists() check → error if not found
    ├── p.suffix in {".tif", ".tiff"} → error if wrong type
    │
    ├── cache_key = "mining_cog_path_/home/hareesh/.../mine_jan.tif"
    ├── Check st.session_state[cache_key] → skip conversion if already done
    │
    └── If not cached:
          cog_path = tempfile.mkdtemp() / "mine_jan_cog.tif"
          convert_to_cog(p, cog_path)   [backend/cog.py]
              subprocess: gdal_translate -of COG -co COMPRESS=LZW -co OVERVIEWS=AUTO
                          /home/.../mine_jan.tif   /tmp/xyz/mine_jan_cog.tif
          meta = read_metadata(cog_path)  [backend/cog.py]
              rasterio.open() → bounds, crs, count, gsd
              → reproject bbox + polygon to WGS84
          st.session_state[cache_key] = {"cog_path": ..., "meta": meta}
```

**Why cache in session_state?**  
Streamlit reruns the script on EVERY interaction (keypress, dropdown change, etc.). Without caching, the 2–10 minute COG conversion would re-run every time the user types anything on the page. The cache key includes the full file path — so a different file triggers a fresh conversion.

---

### Step 2b — Upload Mode (< 1 GB)

```
User selects file via file uploader widget
    │
    ├── file_size_mb > 1024 → error: "Too large, use path mode"
    │
    └── Stream to temp file in 64 MB chunks (prevents RAM overload):
          CHUNK = 64 * 1024 * 1024   ← 64 MB
          while True:
              chunk = tif_file.read(CHUNK)
              raw_tmp.write(chunk)
              pb.progress(...)    ← progress bar updates

          convert_to_cog(raw_path, cog_path)   → same as path mode
          raw_path.unlink()                    → delete original temp file
          read_metadata(cog_path)              → same as path mode
```

---

### When User Clicks "Save & Push to STAC API"

This is the full save pipeline — 6 sequential operations:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
OPERATION 1 — Save original TIF to server folder
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
folder = ~/Documents/serverimages/mining/mancherial/mine_jan/

Path mode:   shutil.copy2(raw_src_path, folder/mine_jan.tif)
Upload mode: stream tif_file → folder/mine_jan.tif  (64MB chunks)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
OPERATION 2 — Copy COG TIF to server folder
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
shutil.copy2(cog_tmp_path, folder/mine_jan_cog.tif)
cog_tmp_path.unlink()   ← clean up temp file

Now the COG is accessible at:
http://10.50.0.170:8085/Documents/serverimages/mining/mancherial/mine_jan/mine_jan_cog.tif

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
OPERATION 3 — Fetch band stats from TiTiler
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
stats = fetch_titiler_stats(cog_url)    [backend/titiler.py]
    GET http://10.50.0.170:8008/cog/statistics?url=http://10.50.0.170:8085/.../mine_jan_cog.tif
    → {"b1": {"percentile_2":12, "percentile_98":210, ...}, "b2":{...}, "b3":{...}}

rescale = compute_rescale(stats, bidx=[1,2,3])  [backend/titiler.py]
    → "12,210&rescale=45,188&rescale=30,201"

⚠️ This MUST happen after the COG is on the server (operation 2),
   because TiTiler fetches it via HTTP URL.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
OPERATION 4 — Download preview PNG from TiTiler
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
preview_url = build_preview_url(cog_url, bidx_qs, rescale)
    → http://10.50.0.170:8008/cog/preview.png?url=...&bidx=1&bidx=2&bidx=3&rescale=...

requests.get(preview_url, timeout=30) → PNG bytes
folder/preview.png.write_bytes(response.content)

asset_urls["preview"] = local_to_url(folder/preview.png)
    → http://10.50.0.170:8085/Documents/.../mine_jan/preview.png

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
OPERATION 5 — Save GeoJSONs + Analytics JSON
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
For each of the 7 GeoJSON text areas (if not empty):
    folder/boundary.geojson.write_text(geojson_texts["boundary"])
    folder/mining_area.geojson.write_text(...)
    ... etc.

If analytics JSON was pasted:
    folder/analytics.json.write_text(analytics_text)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
OPERATION 6 — Build STAC item & push
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
tile_url = build_tile_url(cog_url, bidx_qs, rescale)
    → http://10.50.0.170:8008/cog/WebMercatorQuad/tilejson.json?url=...

stac_item = _build_mining_item(
    item_id       = "mine_jan",
    collection    = "mancherial",
    datetime_str  = "2026-03-03T05:00:00Z",
    region_id     = 1,
    location_name = "mine_jan",
    bbox          = [81.12, 18.85, 81.19, 18.91],
    geometry      = {"type":"Polygon","coordinates":[...]},
    asset_urls    = {
        "visual":   "http://.../mine_jan_cog.tif",
        "tiles":    "http://titiler.../tilejson.json?...",
        "preview":  "http://.../preview.png",
        "boundary": "http://.../boundary.geojson",
        "analytics":"http://.../analytics.json",
        ...
    },
    tif_meta = {"band_count":3, "gsd":0.5, "epsg":32644}
)

folder/mine_jan.json.write_text(json.dumps(stac_item))   ← local backup

api_push_item("mancherial", stac_item)   [backend/stac_api.py]
    POST http://localhost:8082/collections/mancherial/items  {stac_item}
    201 → ✅ "Survey item saved and pushed!"   + 🎈 balloons
    409 → ❌ "Item already exists"
```

---

## TAB 3 — Browse Items

### User Action: Select a mining area from dropdown

```
_render_browse_items_section()
    │
    ├── _mining_collection_ids()   → list of folder names from disk
    ├── st.selectbox(...)
    │     → user picks "mancherial"
    │
    └── fetch_items("mancherial", limit=200)   [backend/stac_api.py]
          GET http://localhost:8082/collections/mancherial/items?limit=200
          → FeatureCollection → extract "features" list
          → list of item dicts
```

### Each item renders as an expander:

```
for item in items:
    iid   = item["id"]               → "mine_jan"
    props = item["properties"]
    loc   = props["mining_location_name"]
    dt    = props["datetime"][:10]    → "2026-03-03"

    expander("🪨 mine_jan · mine_jan · 2026-03-03")
        │
        ├── Left col: Location, Region ID, Survey Date
        └── Right col: Asset links (clickable URLs for each asset)
              - [COG Image](http://.../mine_jan_cog.tif)
              - [TiTiler RGB Tile Service](http://titiler/tilejson.json?...)
              - [RGB Preview Image](http://.../preview.png)
              - [Mining Boundary](http://.../boundary.geojson)
              - [Analytics Data](http://.../analytics.json)
              ...
```

### User Action: Click 🗑️ Delete Item

```
Click 🗑️ Delete `mine_jan`
    → st.session_state["mine_del_confirm_mine_jan"] = True
    → Page reruns → shows "Yes / Cancel" buttons

Click ✅ Yes
    │
    ├── api_delete_item("mancherial", "mine_jan")   [backend/stac_api.py]
    │     DELETE http://localhost:8082/collections/mancherial/items/mine_jan
    │     200/204 → (True, "")
    │
    └── If success:
          item_folder = MINING_ROOT/mancherial/mine_jan/
          shutil.rmtree(item_folder)   ← deletes ALL files from disk
          st.success("Deleted item mine_jan and its files.")
          st.rerun()
```

> Unlike collection delete, **item delete also removes the disk folder**.

---

## Dark Mode Toggle

**User clicks 🌙 / ☀️ button** (top-right of header in `app.py`):

```
st.session_state["dark_mode"] = not current_value
st.rerun()
    │
    → Entire script re-runs
    → _DARK = True/False
    → CSS variables (_BG, _CARD, _HEAD...) recalculate
    → st.markdown(f"<style>:root {{ --accent: {_ACC}; ... }}</style>")
    → All components pick up new CSS variables automatically
```

---

## State Machine — Tab 1 Card "Browse Items →" Button

When you click "Browse Items →" on a collection card in Tab 1:

```
st.session_state["mining_selected_col"] = "mancherial"
st.session_state["mining_active_tab"]   = 2
st.rerun()
    │
    → render_mining_tab() runs again
    → Tab 3 (Browse Items) is selected
    → st.selectbox pre-selects "mancherial" via session_state
```

This is how Streamlit simulates "navigation" — there are no routes, just session state flags.

---

## Complete File-to-Function Responsibility Map

| User Action       | Frontend Function                | Backend Called                                                                                                                                                         | What Backend Does           |
| ----------------- | -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------- |
| Page load         | `app.py`                         | `requests.get` (inline)                                                                                                                                                | Pings 3 services            |
| View mining areas | `_render_collections_section`    | `stac_api.fetch_collections`                                                                                                                                           | GET /collections            |
| Create collection | `_render_create_collection_form` | `stac_api.build_collection_payload` + `api_create_collection`                                                                                                          | POST /collections           |
| Delete collection | (button in collections section)  | `stac_api.api_delete_collection`                                                                                                                                       | DELETE /collections/{id}    |
| Enter file path   | `_render_create_item_section`    | `cog.convert_to_cog` + `cog.read_metadata`                                                                                                                             | gdal_translate + rasterio   |
| Upload file       | `_render_create_item_section`    | `cog.convert_to_cog` + `cog.read_metadata`                                                                                                                             | same as above               |
| Preview STAC JSON | (preview button)                 | `titiler.build_tile_url` + `build_preview_url`                                                                                                                         | builds URLs (no HTTP call)  |
| Save & Push       | (save button)                    | `stac_api.local_to_url` → `titiler.fetch_titiler_stats` → `compute_rescale` → `build_tile_url` → `build_preview_url` → `_build_mining_item` → `stac_api.api_push_item` | Full 6-step pipeline        |
| Browse items      | `_render_browse_items_section`   | `stac_api.fetch_items`                                                                                                                                                 | GET /collections/{id}/items |
| Delete item       | (delete button in browse)        | `stac_api.api_delete_item`                                                                                                                                             | DELETE + rmtree             |

---

## Key Concepts Summary (Lecture Takeaways)

| Concept                       | How it's applied                                                                   |
| ----------------------------- | ---------------------------------------------------------------------------------- |
| **Separation of Concerns**    | UI code never does HTTP directly. Backend never touches Streamlit.                 |
| **Streamlit rerun model**     | Every click reruns the whole script. Session state is the only memory.             |
| **COG caching**               | Expensive conversion cached in `session_state` with file path as key.              |
| **Two-URL design**            | `localhost` for internal API calls, LAN IP in generated STAC URLs.                 |
| **Streaming for large files** | 64 MB chunks prevent RAM overflow for 1–2 GB GeoTIFFs.                             |
| **Order of operations**       | Files must be on server BEFORE TiTiler stats are fetched.                          |
| **Error propagation**         | Backend returns `(bool, str)`. Frontend shows `st.error(msg)` — no raw exceptions. |
| **Disk + API sync**           | Every collection has both a local folder AND an API entry. They must stay in sync. |
