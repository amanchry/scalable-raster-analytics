# rioxarray — Geospatial xarray

**Goal:** Open  GeoTIFFs as **xarray** objects with correct CRS, bounds, and dimensions using **rioxarray**, then clip, align a DEM, and write outputs.

Lessons 1–2 streamed rasters from STAC. This lesson uses **files on disk** so clip, `reproject_match`, and write stay in the foreground.

---

## What is rioxarray?

**rioxarray** extends xarray so that raster I/O and spatial metadata are first-class:

- **Open** rasters (GeoTIFF, COG, etc.) from a path or URL into a `DataArray` or `Dataset`.
- **Attach** CRS and transform via the `.rio` accessor.
- **Reproject**, **clip**, **align**, and **write** rasters using the same accessor.

Under the hood it uses **rasterio** for reading/writing and GDAL for formats.

---

## Sample Data

The lesson notebook reads GeoTIFFs from `data/`:

| File | Variable |
|------|----------|
| `prec_daily_2024_ERA5LAND_native.tif` | ERA5-Land daily precipitation (m), one band per day in 2024 |
| `temp_daily_2024_ERA5LAND_native.tif` | ERA5-Land daily 2 m temperature (K) |
| `copernicus_dem_30m.tif` | Copernicus DEM, 30 m |
| `NL_provinces.shp` | Province polygons (clip to Utrecht) |

```python
import rioxarray as rxr

prec = rxr.open_rasterio("data/prec_daily_2024_ERA5LAND_native.tif", masked=True)
print(prec.rio.crs, prec.rio.bounds(), prec.rio.resolution())
```

Default behavior:

- **Dimensions**: often `band`, `y`, `x` (daily stacks use `band` for time until you rename it).
- **Coordinates**: `x` and `y` come from the geotransform.
- **CRS / bounds / transform**: `data.rio.crs`, `data.rio.bounds()`, `data.rio.transform()`.

`masked=True` turns NoData into NaN.

---

## The `.rio` accessor

| Method / property | Purpose |
|-------------------|--------|
| `.rio.crs` | Coordinate reference system |
| `.rio.bounds()` | (left, bottom, right, top) |
| `.rio.resolution()` | Pixel size |
| `.rio.transform()` | Affine transform (pixel ↔ geo) |
| `.rio.clip_box()` | Clip to a bounding box |
| `.rio.clip()` | Clip with a geometry (e.g. GeoDataFrame) |
| `.rio.reproject()` | Reproject to another CRS |
| `.rio.reproject_match()` | Warp onto another raster's grid |
| `.rio.to_raster()` | Write to GeoTIFF/COG |

Daily climate bands are mapped to a `time` coordinate. The 30 m DEM does not share that grid — `reproject_match` warps it onto one ERA5 slice:

```python
dem = dem.rio.reproject_match(prec.isel(time=0))
ds = xr.Dataset({"prec": prec, "temp": temp, "dem": dem})
```

Then clip a rectangle or a polygon (Utrecht), convert kelvin → °C, mask wet days (`prec > 0.001` m), and write GeoTIFF / NetCDF.

---

## Chunked (lazy) opening

For large rasters and Dask-based workflows, open with **chunks** so the underlying array is a **Dask** array:

```python
data = rioxarray.open_rasterio(path, chunks="auto")  # or chunks={"x": 1024, "y": 1024}
```

Operations then stay **lazy** until `.compute()` or plotting is triggered. This is the bridge to **Lesson 4** (Dask).

---

## Summary

| Concept | Detail |
|--------|--------|
| rioxarray | Geospatial extension for xarray: open rasters, CRS, bounds, clip, reproject |
| Opening | `rioxarray.open_rasterio(path)` → DataArray with `band`, `y`, `x` |
| Align | `reproject_match` puts the DEM on the ERA5 grid |
| Clip | `clip_box` (rectangle) or `clip` (polygon) |
| .rio | Accessor for CRS, bounds, transform, clip, reproject, to_raster |

Next, **Dask** runs parallel, chunked analytics on these structures.

---

Download the notebook for this lesson: `notebooks/03-rioxarray.ipynb`.
