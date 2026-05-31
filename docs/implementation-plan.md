# Implementation Plan: CPH Print Map Data Pipeline

## Goal

Reproduce the QGIS data-preparation workflow as numbered Python scripts.
Styling and print layout remain in QGIS by hand — only data prep is scripted.

---

## Decisions

| # | Question | Decision |
|---|----------|----------|
| 1 | Raw data acquisition | **Scripted download** (`urllib`/`requests`), idempotent if files already present |
| 2 | Geofabrik vs Overpass scripts | **Separate scripts** — easier to iterate on independently |
| 3 | Output format | **One `.gpkg` per layer** — matches QGIS project, safe to rerun individually |
| 4 | Overpass parsing | **`osmnx`** — handles relation geometry correctly |
| 5 | Download vs processing | **Separate** — raw files cached in `data/raw/`, processing reruns without re-downloading |
| 6 | Overpass intermediate storage | **GeoJSON to `data/raw/overpass/`** — cache raw results, iterate on processing without re-querying |
| 8 | Inland water | **Merge** Geofabrik water + Overpass harbour water → one `cph_inland_water.gpkg` |
| 9 | Road/landuse filtering | **One file each**, `fclass` + `bridge` columns preserved — QGIS rules handle tier filtering |

---

## Pipeline

```
01_download_geofabrik.py
    └── downloads denmark-latest-free.shp.zip → data/raw/geofabrik/
    └── downloads land-polygons-split-4326.zip → data/raw/land_polygons/

02_fetch_overpass.py
    └── 4 queries → data/raw/overpass/*.geojson

03_process_geofabrik.py
    └── reads data/raw/geofabrik/ + data/raw/land_polygons/
    └── clip to bbox (55.632, 12.488, 55.738, 12.652)
    └── reproject to EPSG:25832
    └── writes data/processed/

04_process_overpass.py
    └── reads data/raw/overpass/*.geojson
    └── reads data/processed/cph_water_geofabrik.gpkg (written by 03)
    └── writes data/processed/
```

---

## Outputs (`data/processed/`)

| File | Source | Notes |
|------|--------|-------|
| `cph_buildings.gpkg` | Geofabrik buildings | clip + reproject only |
| `cph_roads.gpkg` | Geofabrik roads | `fclass` + `bridge` columns preserved |
| `cph_landuse.gpkg` | Geofabrik landuse | `fclass` column preserved |
| `cph_ocean.gpkg` | Land polygons | clip + reproject; land mask over blue background |
| `cph_inland_water.gpkg` | Geofabrik water ∪ Overpass harbour | union + dissolve |
| `cph_pedestrian_areas.gpkg` | Overpass | ways/relations with highway=pedestrian or place=square |
| `cph_sports.gpkg` | Overpass | leisure=pitch/sports_centre/stadium/track |
| `cph_industrial_extra.gpkg` | Overpass | landuse=railway/port/depot + industrial + infrastructure |

---

## QGIS Rule Filters (for reference)

### Roads (`cph_roads.gpkg`)

| Label | Filter |
|-------|--------|
| `motorway-primary` | `fclass IN ('motorway', 'trunk', 'primary')` |
| `secondary-tertiary` | `fclass IN ('secondary', 'tertiary')` |
| `residential-small` | `fclass IN ('residential', 'unclassified', 'pedestrian', 'living_street')` |
| `major-cycle-pedestrian` | `fclass IN ('cycleway', 'footway', 'path', 'pedestrian') AND bridge IN ('T', 'yes')` |

### Landuse (`cph_landuse.gpkg`)

| Label | Filter |
|-------|--------|
| `green space` | `fclass IN ('park', 'recreation_ground', 'forest', 'grass', 'meadow', 'village_green', 'cemetery', 'nature_reserve', 'orchard', 'allotments')` |
| `industrial` | `fclass IN ('industrial', 'railway', 'port', 'quarry', 'landfill', 'military', 'brownfield', 'construction')` |
| `commercial` | `fclass IN ('commercial', 'retail', 'institutional')` |

---

## Overpass Queries

### Harbour water
```
[out:xml][timeout:120];
(
  way["natural"="water"](55.632,12.488,55.738,12.652);
  relation["natural"="water"](55.632,12.488,55.738,12.652);
  way["waterway"="dock"](55.632,12.488,55.738,12.652);
  relation["waterway"="dock"](55.632,12.488,55.738,12.652);
  way["waterway"="moat"](55.632,12.488,55.738,12.652);
  way["water"="moat"](55.632,12.488,55.738,12.652);
  way["water"="basin"](55.632,12.488,55.738,12.652);
  way["leisure"="marina"](55.632,12.488,55.738,12.652);
);
(._;>;);
out body;
```

### Pedestrian areas
```
[out:xml][timeout:60];
(
  way["highway"="pedestrian"]["area"="yes"](55.632,12.488,55.738,12.652);
  way["place"="square"](55.632,12.488,55.738,12.652);
  relation["highway"="pedestrian"](55.632,12.488,55.738,12.652);
  relation["place"="square"](55.632,12.488,55.738,12.652);
);
(._;>;);
out body;
```

### Sports
```
[out:xml][timeout:60];
(
  way["leisure"="pitch"](55.632,12.488,55.738,12.652);
  way["leisure"="sports_centre"](55.632,12.488,55.738,12.652);
  way["leisure"="stadium"](55.632,12.488,55.738,12.652);
  way["leisure"="track"](55.632,12.488,55.738,12.652);
  relation["leisure"="pitch"](55.632,12.488,55.738,12.652);
  relation["leisure"="sports_centre"](55.632,12.488,55.738,12.652);
  relation["leisure"="stadium"](55.632,12.488,55.738,12.652);
);
(._;>;);
out body;
```

### Industrial extra
```
[out:xml][timeout:60];
(
  way["landuse"="railway"](55.632,12.488,55.738,12.652);
  relation["landuse"="railway"](55.632,12.488,55.738,12.652);
  way["landuse"="port"](55.632,12.488,55.738,12.652);
  way["landuse"="depot"](55.632,12.488,55.738,12.652);
  way["industrial"](55.632,12.488,55.738,12.652);
  way["man_made"="wastewater_plant"](55.632,12.488,55.738,12.652);
  way["man_made"="works"](55.632,12.488,55.738,12.652);
  way["power"="plant"](55.632,12.488,55.738,12.652);
  way["power"="substation"](55.632,12.488,55.738,12.652);
  way["aeroway"="aerodrome"](55.632,12.488,55.738,12.652);
  way["aeroway"="apron"](55.632,12.488,55.738,12.652);
);
(._;>;);
out body;
```

---

## Key Constants

```python
BBOX_WGS84 = (12.488, 55.632, 12.652, 55.738)  # (minx, miny, maxx, maxy)
TARGET_CRS = "EPSG:25832"
```
