# Application — vegetation greenness

**Goal:** Run one application that uses **every** workshop tool: STAC, COG, xarray, rioxarray, and Dask.

**Question:** How does Sentinel-2 NDVI change from January to June 2023 over a small AOI in northern Karnataka, and is greenness different on higher ground?

---

## Pipeline

```
STAC search (S2 + DEM)  →  sign URLs
       →  open_rasterio(url, chunks)  →  clip_box
       →  reproject_match (DEM onto S2)
       →  concat time  →  NDVI
       →  persist / mean / mask  →  plot + GeoTIFF / NetCDF
```

Clip first. Data stays in the cloud until a plot or write.

---

## What each lesson does here

| Application step | Tool | Lesson |
|------------------|------|--------|
| Search S2 + DEM by bbox, time, cloud cover | STAC + `planetary_computer.sign` | 1 |
| Open a COG; read CRS / shape only | COG + rioxarray | 1 |
| Clip the window; warp DEM onto S2 | `.rio.clip_box`, `.rio.reproject_match` | 3 |
| Stack dates; NDVI; `.sel` / `.isel` | xarray Dataset | 2 |
| `chunks=`, persist, spatial mean, elevation mask | Dask | 4 |
| RGB + NDVI maps, time series, write | matplotlib, `.rio.to_raster` | 1–4 |

---

## Search (STAC)

```python
CLIP = (76.40, 16.75, 76.55, 16.90)

catalog = pystac_client.Client.open(
    "https://planetarycomputer.microsoft.com/api/stac/v1"
)
search = catalog.search(
    collections=["sentinel-2-l2a"],
    bbox=list(CLIP),
    datetime="2023-01-01/2023-06-30",
    max_items=5,
    query={"eo:cloud_cover": {"lt": 15}},
)
items = [planetary_computer.sign(it) for it in search.items()]
```

A Copernicus DEM tile for the same box is signed the same way (`cop-dem-glo-30`).

---

## Cube + NDVI (rioxarray, xarray, Dask)

Open B04 and B08 with `chunks={"x": 1024, "y": 1024}`, `clip_box` to `CLIP`, `concat` along `time`, then:

```python
cube["ndvi"] = (cube["B08"] - cube["B04"]) / (cube["B08"] + cube["B04"])
ts = cube["ndvi"].mean(dim=("y", "x")).compute()
ndvi_high = cube["ndvi"].where(cube["dem"] > 500).mean(dim=("y", "x")).compute()
```

The DEM is aligned with `reproject_match` onto the first Sentinel-2 grid (Lesson 3).

---

## Outputs

- RGB preview from the `visual` asset
- NDVI map for the first date
- Time series: AOI mean vs DEM > 500 m
- `output/ndvi_YYYY-MM-DD.tif`, `dem_on_s2.tif`, `karnataka_ndvi_dem.nc`

---

Download the notebook for this lesson: `notebooks/05-end-to-end.ipynb`.
