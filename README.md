# Copenhagen Print Map

Reproducible data-prep pipeline for a QGIS print map of Copenhagen. Scripts read from `data/raw/`, write GeoPackage outputs to `data/processed/`. Cartography and print layout are done by hand in QGIS.

## Data sources

| Dataset | Source | Downloaded | CRS | Licence |
|---------|--------|------------|-----|---------|
| _add datasets here_ | | | | |

Likely sources: [Dataforsyningen](https://dataforsyningen.dk/), [Datafordeler](https://datafordeler.dk/), [GeoDanmark](https://www.geodanmark.dk/), [DAGI](https://dataforsyningen.dk/data/3784).

## Reproduce

```powershell
uv sync
```

Then run prep scripts in numeric order:

```powershell
uv run scripts/01_clean.py
uv run scripts/02_join.py
# ...
```

QGIS reads `data/processed/*.gpkg`. Rerunning prep regenerates those files.

## Notes

- All outputs in EPSG:25832 (ETRS89 / UTM zone 32N) unless noted otherwise.
- Do not overwrite a `.gpkg` while it is open in QGIS — close/reload the layer first (Windows file locking).
- Use `pathlib.Path` for all paths in scripts.
