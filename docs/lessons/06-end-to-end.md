# End-to-End Workflow

**Goal:** Combine **STAC** (discovery), **COG** (streaming), **xarray** (data cube), **rioxarray** (geospatial I/O), and **Dask** (scalable compute) into one reproducible pipeline.

---

## Pipeline overview

```
STAC API (search)  →  Items + asset URLs  →  Sign (if needed)
       →  open_rasterio(url, chunks)  →  xarray Dataset
       →  Derive index (e.g. NDVI)  →  Reduce (mean, std, etc.) or write
       →  .compute() / .to_raster()
```

All steps stay in Python; data stays in the cloud until you explicitly read or write.

---

## Step 1: Discover assets (STAC)

```python
import pystac_client
import planetary_computer

catalog = pystac_client.Client.open(
    "https://planetarycomputer.microsoft.com/api/stac/v1"
)
search = catalog.search(
    collections=["sentinel-2-l2a"],
    bbox=[-122.4, 37.6, -122.2, 37.8],
    datetime="2023-06-01/2023-06-30",
    max_items=5,
)
items = list(search.items())
items = [planetary_computer.sign(item) for item in items]
```

---

## Step 2: Open as chunked xarray (COG + rioxarray)

Pick one item and open one or more bands (e.g. red and NIR for NDVI). Use **chunks** so the array is Dask-backed.

```python
import rioxarray
import xarray as xr

def open_bands(item, band_names, chunks=None):
    bands = {}
    for name in band_names:
        asset = item.assets[name]
        da = rioxarray.open_rasterio(asset.href, chunks=chunks or "auto")
        bands[name] = da.squeeze("band", drop=True)
    ds = xr.Dataset(bands)
    ds["time"] = item.datetime
    return ds

# One scene
item = items[0]
ds = open_bands(item, ["B04", "B08"], chunks={"x": 1024, "y": 1024})
```

---

## Step 3: Derive an index (xarray)

```python
ds["ndvi"] = (ds["B08"] - ds["B04"]) / (ds["B08"] + ds["B04"])
```

---

## Step 4: Reduce or subset (Dask)

- **Spatial mean** (one value per scene):

```python
mean_ndvi = ds["ndvi"].mean(dim=["x", "y"])
mean_ndvi.compute()
```

- **Time series**: Concatenate multiple scenes along `time`, then reduce:

```python
scenes = [open_bands(it, ["B04", "B08"], chunks={"x": 1024, "y": 1024}) for it in items]
combined = xr.concat(scenes, dim="time")
combined["ndvi"] = (combined["B08"] - combined["B04"]) / (combined["B08"] + combined["B04"])
ts = combined["ndvi"].mean(dim=["x", "y"])
ts.compute()
```

---

## Step 5: Optional — write outputs

- **To raster**: `combined["ndvi"].rio.to_raster("ndvi.tif")` (triggers compute for that variable).
- **To NetCDF**: `combined.to_netcdf("cube.nc")` (with engine and encoding as needed).

---

## Putting it together

A minimal end-to-end script:

1. Search STAC (bbox, time, collection).
2. Sign items if required.
3. Open selected bands with `rioxarray.open_rasterio(..., chunks=...)`.
4. Build a Dataset (and optionally concatenate over time).
5. Compute derived variables (NDVI, EVI, etc.).
6. Reduce (e.g. mean over space) or write (GeoTIFF/NetCDF).
7. Call `.compute()` or use plotting/writing to run the graph.

This is the **scalable raster analytics** workflow: discovery (STAC), streaming (COG), modeling (xarray/rioxarray), and scalable compute (Dask).

---

## Summary

| Step | Tool | Action |
|------|------|--------|
| 1 | STAC (pystac-client) | Search by bbox, time; get asset URLs |
| 2 | planetary_computer (if PC) | Sign asset URLs |
| 3 | rioxarray | open_rasterio(url, chunks) → xarray |
| 4 | xarray | Derive indices, concat over time |
| 5 | Dask (via xarray) | Lazy reductions; .compute() or write to run |

---

Download the notebook for this lesson: `notebooks/06-end-to-end.ipynb`.
