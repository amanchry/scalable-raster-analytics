# rioxarray — Geospatial xarray

**Goal:** Open Cloud-Optimized GeoTIFFs (and other rasters) as **xarray** objects with correct CRS, bounds, and dimensions using **rioxarray**.

---

## What is rioxarray?

**rioxarray** extends xarray so that raster I/O and spatial metadata are first-class:

- **Open** rasters (GeoTIFF, COG, etc.) from path or URL into a `DataArray` or `Dataset`.
- **Attach** CRS and transform via the `.rio` accessor.
- **Reproject**, **clip**, **merge**, and **write** rasters using the same accessor.

Under the hood it uses **rasterio** for reading/writing and GDAL for formats; the result is an xarray structure you can combine with Dask and the rest of the scientific Python stack.



---

## Opening a COG

Use a signed STAC asset URL (see Lesson 1 for search + sign). Example:

```python
import rioxarray
import pystac_client
import planetary_computer

catalog = pystac_client.Client.open(
    "https://planetarycomputer.microsoft.com/api/stac/v1"
)
search = catalog.search(
    collections=["sentinel-2-l2a"],
    bbox=[-122.4, 37.6, -122.2, 37.8],
    datetime="2023-06-15",
    max_items=1,
)
item = planetary_computer.sign(list(search.items())[0])
url = item.assets["B04"].href

data = rioxarray.open_rasterio(url)
```

Default behavior:

- **Dimensions**: often `band`, `y`, `x` (band as first dimension).
- **Coordinates**: `x` and `y` come from the geotransform (centers or edges, depending on driver).
- **CRS**: stored in `data.rio.crs`.
- **Bounds**: `data.rio.bounds`.
- **Transform**: `data.rio.transform()`.

So you get a **data cube slice** (one scene, one or more bands) with spatial labels.

---

## The `.rio` accessor

All spatial operations live under the **`.rio`** accessor:

| Method / property | Purpose |
|-------------------|--------|
| `.rio.crs` | Coordinate reference system |
| `.rio.bounds` | (left, bottom, right, top) |
| `.rio.transform()` | Affine transform (pixel ↔ geo) |
| `.rio.reproject()` | Reproject to another CRS |
| `.rio.clip()` | Clip with a geometry (e.g. GeoDataFrame) |
| `.rio.write_crs()` | Set CRS before writing |
| `.rio.to_raster()` | Write to GeoTIFF/COG |

Example: clip to a bounding box (same CRS as data), or clip with a vector.

```python
# Clip to a box
clipped = data.rio.clip_box(minx=-122.35, miny=37.65, maxx=-122.28, maxy=37.72)
# Or clip with a GeoDataFrame
# clipped = data.rio.clip(geodf.geometry, geodf.crs)
```

Single-band rasters: drop the band dimension for simpler indexing: `data = data.squeeze("band", drop=True)`.

---

## Chunked (lazy) opening

For large rasters and Dask-based workflows, open with **chunks** so that the underlying array is a **Dask** array:

```python
data = rioxarray.open_rasterio(url, chunks="auto")  # or chunks={"x": 1024, "y": 1024}
```

Then operations are **lazy** until you call `.compute()` or trigger plotting. This is the bridge to **Lesson 5** (Dask).

---

## Multi-band and multi-scene

- **Single COG, multiple bands**: `open_rasterio` returns one DataArray with dim `band`; you can `ds = data.to_dataset(dim="band")` to get a Dataset with one variable per band.
- **Multiple scenes (e.g. from STAC)**: Open each asset with `open_rasterio`, optionally align with `xr.concat(..., dim="time")` and assign `time` coordinates from STAC item dates.

We combine this in **Lesson 6** for a full STAC → COG → xarray → Dask pipeline.

---

## Summary

| Concept | Detail |
|--------|--------|
| rioxarray | Geospatial extension for xarray: open rasters, CRS, bounds, clip, reproject |
| Opening | `rioxarray.open_rasterio(path_or_url)` → DataArray with `band`, `y`, `x` |
| .rio | Accessor for CRS, bounds, transform, clip, reproject, to_raster |
| Chunks | `chunks="auto"` or explicit for Dask-backed lazy reading |

Next we use **Dask** to run parallel, chunked analytics on these structures.

---

Download the notebook for this lesson: `notebooks/04-rioxarray.ipynb`.
