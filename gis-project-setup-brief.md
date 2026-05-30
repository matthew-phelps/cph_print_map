# GIS Project — Environment & Directory Setup Brief

Hand this to Claude Code. It defines a reproducible **data-prep** environment on **Windows 11** and a project directory. Cartography (styling, labels, layout) happens separately in the QGIS GUI and is out of scope for this setup.

## Context & decisions (already made)

- **Goal:** Prepare geospatial data with reproducible Python scripts. Styling and print layout are done in QGIS by hand and are intentionally *not* scripted.
- **Reproducibility scope:** Only the data-prep stage. Scripts read from `data/raw/`, write to `data/processed/`. QGIS points at `data/processed/`.
- **Environment manager:** **uv** (not pixi/conda). QGIS's own Python is not used, so there's no need for the conda-forge stack.
- **No `qgis_process`, no PyQGIS.** The prep env and QGIS share nothing but files on disk.
- **Output format:** GeoPackage (`.gpkg`). Do not use Shapefile.
- **Default CRS:** EPSG:25832 (ETRS89 / UTM zone 32N) — standard for Danish data. Confirm per dataset.

## Environment setup steps

1. Install uv if not present (PowerShell):
   ```powershell
   powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
   ```
   (or `winget install --id=astral-sh.uv`)

2. Initialise the project with a pinned Python:
   ```powershell
   uv init gis-project
   cd gis-project
   uv python pin 3.12
   ```

3. Add the data-prep stack:
   ```powershell
   uv add geopandas pyogrio shapely pyproj pandas matplotlib
   ```
   - `geopandas` (uses `pyogrio` for I/O — ships its own GDAL in the wheel, so no separate GDAL install is needed)
   - `matplotlib` for quick visual sanity checks of prep output, not for final maps
   - Add `rasterio` later only if raster prep becomes necessary
   - GDAL CLI tools (`ogr2ogr`, `gdalwarp`) already exist via the QGIS/OSGeo4W install — do **not** try to add GDAL to the uv env

4. Initialise git:
   ```powershell
   git init
   ```

## Project directory structure to create

```
gis-project/
  pyproject.toml          # created by uv
  uv.lock                 # commit this
  .gitignore
  README.md
  data/
    raw/                  # immutable inputs; sources documented in README
      .gitkeep
    processed/            # script outputs; regenerable; gitignored
      .gitkeep
  scripts/                # numbered prep scripts, e.g. 01_clean.py, 02_join.py
  notebooks/              # exploration only, never part of the pipeline
  qgis/                   # the .qgs project + exported map outputs (managed in GUI)
```

## .gitignore contents

```
# Python / uv
.venv/
__pycache__/
*.pyc

# Data — keep folders, ignore contents
data/raw/*
data/processed/*
!data/raw/.gitkeep
!data/processed/.gitkeep

# QGIS cruft
*.qgs~
*.qgz~
*.gpkg-shm
*.gpkg-wal
```

## README.md should contain

- One-line project description
- **Data sources table:** dataset name, source URL, download date, CRS, licence. (Likely Danish sources: Dataforsyningen, Datafordeler, GeoDanmark, DAGI.)
- How to reproduce: `uv sync`, then run prep scripts in numeric order.
- Note that QGIS reads `data/processed/*.gpkg`; rerunning prep regenerates those files.

## Script conventions

- Read only from `data/raw/`, write only to `data/processed/`.
- Reproject everything to EPSG:25832 unless a dataset specifically requires otherwise.
- Write outputs as GeoPackage: `gdf.to_file("data/processed/layers.gpkg", layer="<name>", driver="GPKG")`.
- Deterministic: same inputs produce identical outputs. No GUI steps, no manual edits to processed files.

## Windows 11 gotchas to flag

- **Do not overwrite a `.gpkg` while it is open in QGIS** — Windows file locking will fail the write or leave QGIS showing stale data. Close/reload the layer in QGIS before rerunning a script, or write to a fresh file.
- Use forward slashes or raw strings in Python paths; prefer `pathlib.Path`.
- The `.gpkg-shm` / `.gpkg-wal` sidecar files are normal (SQLite WAL mode); they're gitignored above.
