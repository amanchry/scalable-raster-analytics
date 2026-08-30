# Dask — Scalable Raster Analytics

**Goal:** Use **Dask** with xarray/rioxarray to run **chunked**, **lazy**, and **parallel** raster analytics so that large rasters never need to be fully loaded into memory.

---

## The scalability problem

- A single Sentinel-2 tile might be 10+ GB (all bands, full resolution).
- A time series of 100 dates exceeds RAM on most machines if loaded at once.
- **Solution**: Keep data in **chunks**, compute only the chunks that are needed, and run operations **in parallel** over chunks. **Dask** does exactly that.

---

## Dask 

**Dask** provides array-like types (`dask.array`) that look like NumPy but:

- Are **lazy**: operations build a task graph instead of executing immediately.
- Are **chunked**: the logical array is split into blocks; each block is a task.
- Execute in **parallel** when `.compute()` is called (using threads, processes, or a cluster).

**xarray** integrates Dask: if a DataArray’s data is a `dask.array`, then xarray operations (slicing, reduction, arithmetic) stay lazy until `.compute()` or I/O (e.g. plot, write).

---

## Chunking strategy

- **Chunk size**: Small enough to fit in memory (e.g. 512×512 or 1024×1024 pixels per chunk); large enough to amortize overhead.
- **Alignment**: Chunk boundaries should align with the raster’s internal blocks (e.g. COG tiles) when possible, so one read = one chunk.
- **Dimensions**: For a `(time, band, y, x)` cube, chunk `y` and `x` and keep `time` and `band` unchunked (or chunk time if there are many dates).

Typical pattern:

```python
# When opening with rioxarray
data = rioxarray.open_rasterio(url, chunks={"x": 1024, "y": 1024})
# Or rechunk after open
data = data.chunk({"x": 1024, "y": 1024})
```

---

## Lazy pipeline example

The lesson notebook opens the Lesson 3 GeoTIFFs with `chunks=` (no STAC). ERA5-Land is many days and few pixels; the DEM is one band and many pixels.

```python
import rioxarray

prec = rioxarray.open_rasterio(
    "data/prec_daily_2024_ERA5LAND_native.tif",
    chunks={"band": 30, "x": 200, "y": 200},
)
dem = rioxarray.open_rasterio(
    "data/copernicus_dem_30m.tif",
    chunks={"x": 1024, "y": 1024},
).squeeze("band", drop=True)

# All of this is lazy
prec_ts = prec.mean(dim=("y", "x"))
dem_mean = dem.mean()

# Only this triggers the read
prec_ts.compute()
dem_mean.compute()
```

Dask will:

1. Split each raster into chunks (time slabs on climate; 1024×1024 tiles on the DEM).
2. Run arithmetic and reduction in parallel over chunks.
3. Return a small result (a daily series, a scalar) without holding the full grid in memory.

Lesson 5 uses the same `chunks=` + reduce pattern on Sentinel-2 COGs from STAC.

---

## When compute runs

- **Explicit**: `.compute()` returns a concrete NumPy/xarray result.
- **Implicit**: `.plot()`, `.to_netcdf()`, `.rio.to_raster()` trigger computation for the data they need.
- **Persist**: `.persist()` computes and keeps the result in distributed memory (useful in a cluster) so later steps don’t recompute.

---

## Local vs distributed

- **Default (threaded)**: `dask.array` uses a thread pool; good for I/O-bound COG reads.
- **Processes**: `dask.distributed.Client(processes=True)` for CPU-bound work.
- **Cluster**: Point xarray/Dask to a `distributed.Client("scheduler-address")` to run on multiple machines. The same code (open COG, chunk, reduce) scales up.

---

## Best practices

- **Chunk to align with COG tiles** when possible (e.g. 512 or 1024).
- **Avoid too many tiny chunks** (overhead) and **too few huge chunks** (memory).
- **Use reductions** (`.mean()`, `.sum()`) along dimensions to shrink data before `.compute()`.
- **Profile**: Use `dask.distributed.dashboard` or Dask’s profilers to see where time is spent.

---

## Summary

| Concept | Detail |
|--------|--------|
| Dask | Lazy, chunked, parallel arrays; xarray can wrap Dask arrays |
| Chunking | Split (e.g. y, x) into blocks; align with COG tiles when possible |
| Lazy | Operations build a graph; `.compute()` or I/O triggers execution |
| Scale | Same code can run on one machine or a Dask cluster |


--- 

Download the notebook for this lesson: `notebooks/04-dask-analytics.ipynb`.
