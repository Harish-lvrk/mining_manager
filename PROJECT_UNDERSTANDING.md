# ⛏️ Mining Manager — Project Understanding

## What is this?

A **Streamlit web app** for managing mining survey data. It lets you:

- Create & manage **mining areas** (backed by a STAC API)
- Upload **GeoTIFF satellite images** (with auto COG conversion)
- Attach **vector layers** (GeoJSON) and **analytics JSON** to each survey
- View/delete survey items via a browser UI

The app runs on port **8502** and talks to 3 external services that must be running separately.

---

## Architecture at a Glance

```
Browser (Streamlit UI)
    │
    └── app.py                    ← entry point, header, service status, dark mode
         └── frontend/tab_mining.py  ← all UI logic (collections + items)
              │
              ├── backend/stac_api.py     ← CRUD calls to STAC API (GET/POST/PUT/DELETE)
              ├── backend/stac_builder.py ← builds STAC 1.0.0 item JSON
              ├── backend/cog.py          ← COG conversion (gdal_translate) + rasterio metadata
              └── backend/titiler.py      ← fetches band stats, builds tile/preview URLs
```

---

## External Services (must be running independently)

| Service         | URL (LAN)          | Port | Purpose                              |
| --------------- | ------------------ | ---- | ------------------------------------ |
| **STAC API**    | `10.50.0.170:8082` | 8082 | Stores collections & item metadata   |
| **TiTiler**     | `10.50.0.170:8008` | 8008 | Renders COG tiles + previews         |
| **File Server** | `10.50.0.170:8085` | 8085 | Serves GeoTIFFs / GeoJSONs over HTTP |

> App calls STAC API via `localhost:8082` internally, but embeds LAN IP in generated URLs so other network clients can reach them.

---

## Key Files

| File                      | Role                                                                             |
| ------------------------- | -------------------------------------------------------------------------------- |
| `app.py`                  | Page config, dark mode toggle, service health status dots, mounts the mining tab |
| `config.py`               | All URLs, IPs, and local paths in one place                                      |
| `logger.py`               | Structured file logging (`logs/stac_manager.log`)                                |
| `frontend/tab_mining.py`  | All UI: create collection form, upload item form, browse items                   |
| `backend/stac_api.py`     | HTTP helpers (`_get`, `_post`, `_put`, `_delete`) + collection/item CRUD         |
| `backend/stac_builder.py` | Assembles a valid STAC 1.0.0 `Feature` dict                                      |
| `backend/cog.py`          | Calls `gdal_translate -of COG` + reads bbox/epsg/gsd with `rasterio`             |
| `backend/titiler.py`      | Calls `/cog/statistics`, builds tile & preview URLs                              |

---

## Data Flow — Creating a Survey Item

```
User uploads .tif (or gives local path)
    → cog.py: convert to COG via gdal_translate
    → cog.py: read bbox, EPSG, GSD, band count
    → titiler.py: fetch per-band stats from TiTiler
    → titiler.py: compute rescale values
    → Files saved to: ~/Documents/serverimages/mining/<collection>/<item>/
        ├── <item>.tif          (original)
        ├── <item>_cog.tif      (cloud-optimized)
        ├── preview.png         (downloaded from TiTiler)
        ├── boundary.geojson
        ├── mining_area.geojson
        ├── ... (other GeoJSONs)
        ├── analytics.json
        └── <item>.json         (full STAC item)
    → stac_builder.py: build STAC item JSON
    → stac_api.py: POST to STAC API
```

---

## Mining-Specific Concepts

- A **collection** = one geographic mining area (e.g. `mancherial`).  
  Only collections with a folder under `MINING_ROOT` appear as "mining areas".
- An **item** = one survey snapshot (one date, one GeoTIFF).
- Each item carries **7 optional GeoJSON vector layers** (boundary, active area, new pit, reclamation, stockpile, water pits, haul roads) + an **analytics JSON** blob.
- `region_id` is auto-assigned as the collection's index in the sorted list.

---

## How to Run

```bash
cd ~/Projects/mining-manager
source venv/bin/activate
streamlit run app.py --server.address 0.0.0.0 --server.port 8502
```
