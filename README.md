# svrwx-aap
Severe Weather At a Point: A lightweight mapping tool for visualizing [NWS Damage Assessment Toolkit](https://apps.dat.noaa.gov/StormDamage/DamageViewer/) damage survey records near any fixed point of interest.
---

## How it works

| File | Role |
|------|------|
| `config.json` | Site coordinates, radius, date range, display preferences, custom markers. |
| `fetch_dat.py` | Reads `config.json`, queries the DAT ArcGIS FeatureServer, filters locally, and maintains `data/dat_cache.json`. |
| `index.html` | GitHub Pages viewer. Reads `data/dat_cache.json`, filters client-side by date and layer type, renders an interactive Leaflet map. |

---

## Quick start

### 1. Fork the Repository

Click **Fork** on the [svrwx-aap GitHub page](https://github.com/Metspy/svrwx-aap) to create your own copy. This allows you to publish your own GitHub Pages map under your account. Forking gives you an independent repo you can push to and enable Pages on.

### 2. Clone your fork locally

```bash
git clone https://github.com/YOUR-USERNAME/svrwx-aap.git
cd svrwx-aap
mkdir data
```

### 3. Configure your site

Copy the example config and edit it:

```bash
cp config.json.example config.json
```

```json
{
  "site": {
    "name":        "My Weather Station",
    "short_name":  "MWS",
    "description": "Backyard station, Huntsville AL",
    "lat":         34.730,
    "lon":        -86.586
  },
  "filter": {
    "radius_km":      100,
    "start_date":     "2024-01-01",
    "end_date":       null,
    "retention_days": 400
  },
  "display": {
    "default_window_days": 30,
    "map_zoom":            7,
    "ring_distances_km":  [25, 50, 100]
  }
}
```

See [Configuration reference](#configuration-reference) below for all options.

### 4. Run the initial fetch to populate record

```bash
python3 fetch_dat.py --backfill
```

No virtual environment or `pip install` needed — `fetch_dat.py` uses only the Python standard library. The backfill fetches all DAT records from `start_date` to today within your configured radius and writes `data/dat_cache.json`.

### 5. Commit and push

```bash
git add data/dat_cache.json
git commit -m "Initial DAT backfill"
git push
```

### 6. Enable GitHub Pages

In your repo: **Settings → Pages → Source: Deploy from branch → `main` / `(root)`**

Your map will be live at `https://YOUR-USERNAME.github.io/svrwx-aap/` within a minute or two.

---

## Daily cron updates

Add to your crontab (`crontab -e`) to enable daily updates:

```cron
# Fetch new DAT records at 06:00 daily, then commit and push
0 6 * * * cd /path/to/svrwx-aap && \
    python3 fetch_dat.py && \
    git add data/dat_cache.json && \
    git commit -m "DAT update $(date +\%Y-\%m-\%d)" && \
    git push >> /tmp/dat_fetch.log 2>&1
```

### What each run does

1. Reads `data/dat_cache.json` to find the last successful fetch timestamp.
2. Queries DAT layers **0** (damage points), **1** (damage tracks/lines), **2** (damage polygons) for records since `last_fetch − 3 days`. The 3-day overlap catches late-submitted NWS surveys.
3. Applies a local **Haversine distance filter** — only features whose centroid falls within `radius_km` of your site are retained. The bounding box sent to the API is slightly larger than the radius; exact filtering is done in Python.
4. **Merges** new records into the cache, de-duplicating by `globalid`.
5. **Flushes** records older than `retention_days` (or before `start_date`, whichever is more recent).
6. Writes an updated `data/dat_cache.json` atomically.

---

## Viewer features

- **Quick window buttons** — 7 / 14 / 30 / 90-day rolling windows
- **Custom date range** — any span from `start_date` to today
- **Layer toggles** — Points (spot damage), Tracks (tornado paths), Polygons (damage areas)
- **EF-scale colour coding** — EF0 green → EF1 yellow → EF2 orange → EF3 red → EF4/EF5 purple; TSTM/Wind cyan
- **Configurable range rings** — drawn from `ring_distances_km` in your config
- **Sidebar event list** — sorted by date, showing EF rating, NWS office, and distance from site
- **Click-to-zoom** — clicking a sidebar event pans the map to that feature
- **Popup details** — EF scale, storm date, NWS office, wind speed, damage description

---

## Configuration reference

All settings live in `config.json`. The fetcher writes selected values into the cache metadata so the viewer never needs a separate copy.

### `site`

| Key | Required | Description |
|-----|----------|-------------|
| `lat` | ✓ | Reference point latitude (decimal degrees) |
| `lon` | ✓ | Reference point longitude (decimal degrees, negative = west) |
| `name` | ✓ | Full name shown in the viewer header and site popup |
| `short_name` | | Abbreviated name shown on the map marker (≤4 chars works best) |
| `description` | | Free-text description (used in site popup) |

### `filter`

| Key | Required | Default | Description |
|-----|----------|---------|-------------|
| `radius_km` | ✓ | — | Haversine filter radius in km. Events beyond this distance from the site are discarded. |
| `start_date` | ✓ | — | Earliest date to fetch/retain, `YYYY-MM-DD`. Used as the `--backfill` start and as a hard floor for the cache flush. |
| `end_date` | | `null` | Optional campaign end date. The fetcher will skip runs after this date. Set to `null` for ongoing deployments. |
| `retention_days` | | `400` | Records older than this many days are flushed on each run. Should be ≥ the span from `start_date` to `end_date` (or today) to avoid losing historical data. The script will warn if this value is too short. |

### `display`

| Key | Default | Description |
|-----|---------|-------------|
| `default_window_days` | `30` | Which quick-window button is active when the viewer first loads. |
| `map_zoom` | `7` | Initial Leaflet zoom level. |
| `ring_distances_km` | `[]` | List of range rings to draw (km). E.g. `[25, 50, 100]`. Empty list = no rings. |

### `markers` (optional)

An array of custom map markers — instruments, reference locations, nearby towns, road crossings, etc. Each appears on the map with a clickable popup and is rendered independently of the DAT damage data.

```json
"markers": [
  {
    "name":        "Ka-SACR",
    "lat":         34.342481,
    "lon":        -87.338177,
    "color":       "#ff9900",
    "symbol":      "circle",
    "description": "Ka-band Scanning ARM Cloud Radar"
  }
]
```

| Key | Required | Description |
|-----|----------|-------------|
| `name` | ✓ | Label shown in the popup title |
| `lat` | ✓ | Latitude (decimal degrees) |
| `lon` | ✓ | Longitude (decimal degrees) |
| `color` | | Hex color for the marker fill. Defaults to `#ffffff`. |
| `symbol` | | Shape: `circle`, `square`, `diamond`, `triangle`, `cross`. Defaults to `circle`. |
| `description` | | Optional note shown in the popup. |

Markers are rendered on every page load from the cache metadata.

---

## Data notes

- DAT records come from **NWS post-event damage surveys**; they are not real-time. Records typically appear hours to a few days after an event.
- Data is considered **preliminary**; official statistics are published in the [NCEI Storm Data](https://www.ncdc.noaa.gov/IPS/sd/sd.html) publication.
- The DAT includes tornado tracks (EF0–EF5), straight-line wind/TSTM damage, hail, and occasionally tropical events.
- A 403 response from the DAT server may occur if the server is temporarily unavailable or under maintenance. The fetcher logs errors and exits gracefully — the existing cache is left untouched.

---

## API reference

NWS DAT ArcGIS FeatureServer:
```
https://services.dat.noaa.gov/arcgis/rest/services/nws_damageassessmenttoolkit/DamageViewer/FeatureServer
  Layer 0 — Damage Points SDE   (esriGeometryPoint)
  Layer 1 — Damage Lines SDE    (esriGeometryPolyline — tornado tracks, damage paths)
  Layer 2 — Damage Polygons SDE (esriGeometryPolygon)
```

Key fields: `stormdate`, `efscale`, `windspeed`, `injuries`, `deaths`, `office`, `damage_txt`, `comments`, `globalid`, `lat`, `lon`

---

## Using multiple sites

You can maintain caches for several sites in the same repo by passing `--config`:

```bash
python fetch_dat.py --config configs/site1.json
python fetch_dat.py --config configs/site2.json
```

Each config should point to a different `data/` subdirectory if you want to keep the caches separate.
