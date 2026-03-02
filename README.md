# ⛏️ Mining Manager

Standalone Streamlit app for managing mining STAC collections and survey items.

## Setup

```bash
cd ~/Projects/mining-manager

# Option A: reuse the stac-ingestor venv (fastest)
source ~/Projects/stac-ingestor/venv/bin/activate

# Option B: create a fresh venv
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
```

## Run

```bash
streamlit run app.py --server.address 0.0.0.0 --server.port 8503
```

Open: **http://10.50.0.170:8503**

## Project structure

```
mining-manager/
├── app.py                  ← entry point
├── config.py               ← LAN IPs, service URLs, paths
├── logger.py
├── requirements.txt
├── .streamlit/config.toml
├── backend/
│   ├── cog.py              ← GeoTIFF → COG conversion & metadata
│   ├── stac_api.py         ← STAC API push / delete helpers
│   ├── titiler.py          ← TiTiler stats & URL builders
│   └── stac_builder.py
└── frontend/
    └── tab_mining.py       ← All mining UI (collections + items)
```

## What it does

- **Mining Areas** — create STAC collections for each mine region (Mancherial, Adilabad…)
- **Create Survey Item** — upload GeoTIFF (auto-converted to COG), paste GeoJSON layers, push to STAC
- **Browse Items** — view items in a collection, inspect assets, delete items + files

## Syncing changes from stac-ingestor

```bash
cp ~/Projects/stac-ingestor/frontend/tab_mining.py ~/Projects/mining-manager/frontend/tab_mining.py
```
