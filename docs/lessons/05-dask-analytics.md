# Dask — Scalable Raster Analytics

**Goal:** Use **Dask** with xarray/rioxarray to run **chunked**, **lazy**, and **parallel** raster analytics so that large rasters never need to be fully loaded into memory.

---

## The scalability problem

- A single Sentinel-2 tile might be 10+ GB (all bands, full resolution).
- A time series of 100 dates exceeds RAM on most machines if loaded at once.
- **Solution**: Keep data in **chunks**, compute only the chunks that are needed, and run operations **in parallel** over chunks. **Dask** does exactly that.

---

## Dask in one paragraph

**Dask** provides array-like types (`dask.array`) that look like NumPy but:

- Are **lazy**: operations build a task graph instead of executing immediately.
- Are **chunked**: the logical array is split into blocks; each block is a task.
- Execute in **parallel** when you call `.compute()` (using threads, processes, or a cluster).

**xarray** integrates Dask: if a DataArray’s data is a `dask.array`, then xarray operations (slicing, reduction, arithmetic) stay lazy until you call `.compute()` or trigger I/O (e.g. plot, write).

---

## Chunking strategy

- **Chunk size**: Small enough to fit in memory (e.g. 512×512 or 1024×1024 pixels per chunk); large enough to amortize overhead.
- **Alignment**: Chunk boundaries should align with the raster’s internal blocks (e.g. COG tiles) when possible, so one read = one chunk.
- **Dimensions**: For a `(time, band, y, x)` cube, you might chunk in `y` and `x` and keep `time` and `band` unchunked (or chunk time if you have many dates).

Typical pattern:

```python
# When opening with rioxarray
data = rioxarray.open_rasterio(url, chunks={"x": 1024, "y": 1024})
# Or rechunk after open
data = data.chunk({"x": 1024, "y": 1024})
```

---

## Lazy pipeline example

```python
import rioxarray
import xarray as xr

url = "https://..."
ds = rioxarray.open_rasterio(url, chunks="auto").to_dataset(dim="band")
ds = ds.rename({1: "red", 2: "green", 3: "blue", 4: "nir"})

# All of this is lazy
ndvi = (ds["nir"] - ds["red"]) / (ds["nir"] + ds["red"])
mean_ndvi = ndvi.mean(dim=["x", "y"])

# Only this triggers actual read + compute
result = mean_ndvi.compute()
```

Dask will:

1. Read only the chunks needed for the requested window or reduction.
2. Run arithmetic and reduction in parallel over chunks.
3. Return a small result (e.g. a scalar or 1D array) without ever holding the full raster in memory.

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

Download the notebook for this lesson: `notebooks/05-dask-analytics.ipynb`.
