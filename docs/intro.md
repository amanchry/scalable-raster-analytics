# Introduction

## Why cloud-native raster analytics?

Traditional raster workflows often assume:

- Data lives on your machine or a mounted drive.
- You download full scenes or tiles before analysis.
- Processing is limited by RAM and single-machine CPU.

**Cloud-native** workflows invert this:

- **Discovery** happens via catalogs (STAC) over the web.
- **Data stays in the cloud**; you stream only the windows or bands you need (e.g. via COGs).
- **Compute** can be chunked and parallelized (Dask) so that large rasters never need to be fully loaded.

The result: you can work with continental or global, multi-temporal rasters from a laptop or a cluster using the same code.

---

## The data cube concept

A **data cube** (or *raster data cube*) is a multi-dimensional array where dimensions typically include:

- **Spatial**: `y`, `x` (or lat/lon)
- **Temporal**: `time`
- **Thematic**: `band`, `variable`, or `layer`

Conceptually:

```
         time →
band  [  [ 2D raster ]  [ 2D raster ]  ...  ]
      [  [ 2D raster ]  [ 2D raster ]  ...  ]
      ...
```

In this course we use **xarray** to represent this model: dimensions and coordinates are **labeled**, so you select by name (e.g. `time`, `band`) instead of axis index. That makes code readable and less error-prone when combining multiple scenes or variables.

---

## Tooling overview

| Layer | Tool | Purpose |
|-------|------|--------|
| Discovery | **STAC** (pystac-client) | Search catalogs by bbox, time, and metadata |
| Storage / transfer | **Cloud-Optimized GeoTIFF** | HTTP range requests; read windows without full download |
| Data model | **xarray** | Labeled N-dimensional arrays (the “cube”) |
| Geospatial | **rioxarray** | CRS, bounds, and GeoTIFF/COG I/O in xarray |
| Compute | **Dask** | Lazy, chunked, parallel execution |

Together they form a pipeline: **STAC (find) → COG (stream) → xarray (model) → Dask (scale)**.

---
