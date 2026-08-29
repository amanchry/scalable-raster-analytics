# rioxarray — Geospatial xarray

**Goal:** Open Cloud-Optimized GeoTIFFs as **xarray** objects with correct CRS, bounds, and dimensions using **rioxarray**, then clip and reproject.

---

## What is rioxarray?

**rioxarray** extends xarray so that raster I/O and spatial metadata are first-class:

- **Open** rasters (GeoTIFF, COG, etc.) from a path or URL into a `DataArray` or `Dataset`.
- **Attach** CRS and transform via the `.rio` accessor.
- **Reproject**, **clip**, **align**, and **write** rasters using the same accessor.

Under the hood it uses **rasterio** for reading/writing and GDAL for formats; the result is an xarray structure that combines with Dask and the rest of the scientific Python stack.

---

## Opening a COG from STAC

Use a signed STAC asset URL (see Lesson 1 for search + sign). The lesson notebook opens **Copernicus DEM** (`cop-dem-glo-30`) and one **Sentinel-2** red band so two grids can be aligned.

```python
import rioxarray
import pystac_client
import planetary_computer

catalog = pystac_client.Client.open(
    "https://planetarycomputer.microsoft.com/api/stac/v1"
)
CLIP = (76.40, 16.75, 76.55, 16.90)  # northern Karnataka, India

search = catalog.search(
    collections=["cop-dem-glo-30"],
    bbox=list(CLIP),
    max_items=1,
)
item = planetary_computer.sign(list(search.items())[0])
url = item.assets["data"].href

data = rioxarray.open_rasterio(url)
```

Default behavior:

- **Dimensions**: often `band`, `y`, `x` (band as first dimension).
- **Coordinates**: `x` and `y` come from the geotransform.
- **CRS**: stored in `data.rio.crs`.
- **Bounds**: `data.rio.bounds()`.
- **Transform**: `data.rio.transform()`.

The result is a **data cube slice** (one tile, one or more bands) with spatial labels. `open_rasterio` on a COG URL reads the header first; pixels load when a clip, plot, or write needs them.

ERA5-Land daily precipitation and temperature GeoTIFFs are not published as COGs on Planetary Computer. Elevation is available as `cop-dem-glo-30`. Climate reanalysis on that catalog is `era5-pds` (Zarr, not GeoTIFF).

---

## The `.rio` accessor

All spatial operations live under the **`.rio`** accessor:

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

Example: clip to a bounding box in WGS84 (`crs="EPSG:4326"`), then reproject to UTM zone 43N.

```python
data = data.squeeze("band", drop=True)  # single-band: drop the extra dim

clipped = data.rio.clip_box(
    minx=76.40, miny=16.75, maxx=76.55, maxy=16.90, crs="EPSG:4326"
)
utm = clipped.rio.reproject("EPSG:32643")
```

`reproject_match` is the usual way to put two STAC rasters on one grid (for example DEM onto Sentinel-2):

```python
dem_on_s2 = dem_clip.rio.reproject_match(s2_clip)
```

---

## Chunked (lazy) opening

For large rasters and Dask-based workflows, open with **chunks** so that the underlying array is a **Dask** array:

```python
data = rioxarray.open_rasterio(url, chunks="auto")  # or chunks={"x": 1024, "y": 1024}
```

Operations then stay **lazy** until `.compute()` or plotting is triggered. This is the bridge to **Lesson 4** (Dask).

---

## Multi-band and multi-scene

- **Single COG, multiple bands**: `open_rasterio` returns one DataArray with dim `band`; `ds = data.to_dataset(dim="band")` yields a Dataset with one variable per band.
- **Multiple scenes (e.g. from STAC)**: Open each asset with `open_rasterio`, optionally align with `xr.concat(..., dim="time")` and assign `time` coordinates from STAC item dates.

This is combined in **Lesson 5** for a full STAC → COG → xarray → Dask pipeline.

---

## Summary

| Concept | Detail |
|--------|--------|
| rioxarray | Geospatial extension for xarray: open rasters, CRS, bounds, clip, reproject |
| Opening | `rioxarray.open_rasterio(path_or_url)` → DataArray with `band`, `y`, `x` |
| .rio | Accessor for CRS, bounds, transform, clip, reproject, reproject_match, to_raster |
| Chunks | `chunks="auto"` or explicit for Dask-backed lazy reading |

Next, **Dask** runs parallel, chunked analytics on these structures.

---

Download the notebook for this lesson: `notebooks/03-rioxarray.ipynb`.
