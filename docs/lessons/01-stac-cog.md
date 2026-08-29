# STAC & Cloud-Optimized GeoTIFF

**Goal:** Discover raster assets with STAC, then stream only the required pixels from a Cloud-Optimized GeoTIFF — without downloading full scenes.

---

## What is STAC?

**STAC** (SpatioTemporal Asset Catalog) is an open specification for describing geospatial assets (imagery, derived rasters, etc.) in a consistent way. A **STAC Catalog** exposes:

- **Collections**: groups of assets (e.g. “Sentinel-2 L2A”, “Landsat C2 L2”).
- **Items**: individual scenes or products, each with:
  - **Spatial extent**: `bbox` and/or `geometry`.
  - **Temporal extent**: `datetime` or interval.
  - **Assets**: links to actual files (e.g. COG URLs) with roles (e.g. `visual`, `data`, `metadata`).

A STAC API query returns **metadata and URLs**; imagery is not downloaded until it is opened (e.g. with rasterio or rioxarray).

---

## The STAC Specification

The STAC Specification consists of **4 semi-independent specifications**. Each can be used alone, but they work best in concert with one another.

| Specification | Role |
|---------------|------|
| **STAC Item** | The core atomic unit: a single spatiotemporal asset represented as a GeoJSON feature plus `datetime` and `links`. |
| **STAC Catalog** | A simple, flexible JSON file of links that provides a structure to organize and browse STAC Items. A series of best practices helps make recommendations for creating real-world STAC Catalogs. |
| **STAC Collection** | An extension of the STAC Catalog with additional information—extents, license, keywords, providers, etc.—that describe the STAC Items that fall within the Collection. |
| **STAC API** | A RESTful endpoint that enables search of STAC Items, specified in OpenAPI and following OGC's WFS 3. |

In this guide we interact mainly with a **STAC API**: we search for **Items** (often scoped by **Collection**), then use each Item’s asset links to stream rasters. The **Catalog** is the entry point we open with `pystac_client.Client.open(...)`.

Read More: [STAC Specification](https://stacspec.org/en/tutorials/intro-to-stac/).

---

## STAC API and clients

Many providers expose a **STAC API** (REST endpoint). Examples:

- [Microsoft Planetary Computer](https://planetarycomputer.microsoft.com/)
- [Earth Search (Element 84)](https://earth-search.aws.element84.com/)
- [Google Earth Engine STAC (where applicable)](https://earthengine-stac.storage.googleapis.com/)

In Python, **pystac-client** is the standard way to search a STAC API: open the catalog, then query by bbox, datetime, and other filters.

---

## Core operations

### 1. Open a catalog

```python
import pystac_client

# Planetary Computer (no API key required for read)
catalog = pystac_client.Client.open(
    "https://planetarycomputer.microsoft.com/api/stac/v1"
)
```

### 2. List collections

```python
for col in catalog.get_collections():
    print(col.id)
```

### 3. Search by bbox and time

```python
# Northern Karnataka, India
BBOX = [75.62, 16.20, 77.29, 17.47]  # [min_lon, min_lat, max_lon, max_lat]

search = catalog.search(
    collections=["sentinel-2-l2a"],
    bbox=BBOX,
    datetime="2023-06-01/2023-06-30",
    max_items=10,
)
items = list(search.items())
```

### 4. Inspect an item and its assets

Each **item** has an `id`, `geometry`, `bbox`, `datetime`, and `assets` (a dict of asset keys → asset objects). Each asset has `href` (URL), optional `title`, and `roles`.

```python
item = items[0]
print(item.id, item.datetime)
for key, asset in item.assets.items():
    print(key, asset.href)
```

The `href` for a COG is passed to **rasterio** or **rioxarray** to open and stream the raster.

---

## Signing URLs (Planetary Computer)

Microsoft Planetary Computer uses **signed URLs** so that access is time-limited. Use the `planetary_computer` package to sign asset URLs:

```python
import planetary_computer

signed_item = planetary_computer.sign(item)
url = signed_item.assets["B04"].href  # red band
```

---

## What is a Cloud-Optimized GeoTIFF?

A **GeoTIFF** is a TIFF file with georeferencing (CRS, extent, resolution) in the same container. A **Cloud-Optimized GeoTIFF (COG)** is a GeoTIFF organized so that:

1. **Tiling**: The image is stored in small tiles (e.g. 512×512).
2. **Overview pyramids**: Lower-resolution overviews are embedded.
3. **Metadata and layout**: Key metadata and the tile index are near the start of the file.

When the file is served over HTTP (e.g. from S3 or Azure Blob), a client can issue **HTTP range requests** to read only specific byte ranges. A 2 GB scene can then be reduced to a single tile or a small window—**streaming** only the required bytes.

---

## Why this matters for analytics

- **Band selection**: Read only the required bands.
- **Spatial windowing**: Read a bounding box (e.g. a study area) instead of the full scene.
- **Overview levels**: For quick previews or coarse analysis, read from an overview.
- **Lazy loading**: Libraries like **rasterio** and **rioxarray** can open the URL and fetch data only when a window is read or computation is triggered.

---

## How reading a COG works (conceptually)

```
┌─────────────────────────────────────────────────────────┐
│  COG on HTTP (e.g. S3 / Azure)                          │
│  ┌─────┬─────┬─────┐                                    │
│  │Tile │Tile │Tile │  ...                               │
│  ├─────┼─────┼─────┤                                    │
│  │Tile │Tile │Tile │  Client requests range(bytes)     │
│  └─────┴─────┴─────┘       → only those tiles transfer  │
└─────────────────────────────────────────────────────────┘
```

The client (rasterio/rioxarray) uses the file’s internal structure to map a **window** (in pixel or geographic coordinates) to byte ranges, then requests those ranges. No special “COG protocol” is needed—just HTTP Range.

---

## Opening a COG from a URL

With **rasterio**, the signed STAC `href` is opened as if it were a path. The driver uses HTTP range requests. Metadata is read first; pixels are fetched only on `.read()`.

**Open the COG and read metadata** (no full download):

```python
import rasterio
from rasterio.windows import from_bounds

with rasterio.open(url) as src:
    print("CRS:", src.crs)
    print("Bounds:", src.bounds)
    print("Shape (height, width):", src.shape)
    print("Block size:", src.block_shapes)
```

**Read a window by geographic bounds** (only those tiles are streamed):

```python
from rasterio.warp import transform_bounds

# Small window at the center of this scene (WGS84)
minx, miny, maxx, maxy = signed_item.bbox
cx, cy = (minx + maxx) / 2, (miny + maxy) / 2
delta = 0.08
left, bottom, right, top = cx - delta, cy - delta, cx + delta, cy + delta

with rasterio.open(url) as src:
    left_t, bottom_t, right_t, top_t = transform_bounds(
        "EPSG:4326", src.crs, left, bottom, right, top
    )
    window = from_bounds(left_t, bottom_t, right_t, top_t, src.transform)
    data = src.read(1, window=window)
    print("Window shape:", data.shape)
```

**Optional: plot the window**

Sentinel-2 L2A is `uint16` (reflectance × 10000). Stretch with percentiles so the window is visible.

```python
import numpy as np
import matplotlib.pyplot as plt

valid = data[data > 0]
vmin, vmax = np.percentile(valid, (2, 98))

fig, ax = plt.subplots(1, 1, figsize=(6, 5))
ax.imshow(data, cmap="gray", vmin=vmin, vmax=vmax)
ax.set_title("COG window (B04) — streamed only")
ax.axis("off")
plt.tight_layout()
plt.show()
```

The same idea applies with **rioxarray**: `rioxarray.open_rasterio(url)`, then slice or `.sel()` so that only the required window is read.

**RGB window and local download** — the `visual` asset is a 3-band RGB COG. The same geographic window can be plotted and written to a small tiled GeoTIFF:

```python
visual_url = signed_item.assets["visual"].href

with rasterio.open(visual_url) as src:
    left_t, bottom_t, right_t, top_t = transform_bounds(
        "EPSG:4326", src.crs, left, bottom, right, top
    )
    window = from_bounds(left_t, bottom_t, right_t, top_t, src.transform)
    rgb = src.read(window=window)
    rgb_transform = src.window_transform(window)
    rgb_profile = src.profile.copy()

img = np.moveaxis(rgb, 0, -1)
fig, ax = plt.subplots(1, 1, figsize=(6, 5))
ax.imshow(img)
ax.set_title("COG window (RGB visual)")
ax.axis("off")
plt.show()

rgb_profile.update(
    height=rgb.shape[1],
    width=rgb.shape[2],
    count=rgb.shape[0],
    transform=rgb_transform,
    compress="deflate",
    tiled=True,
)
with rasterio.open("s2_rgb_aoi.tif", "w", **rgb_profile) as dst:
    dst.write(rgb)
```

---

## Requirements for a “true” COG

For efficient streaming, the file should:

- Be **tiled** (not strip-based).
- Have **overview** levels for fast coarse reads.
- Be served from a store that supports **HTTP Range** (S3, Azure Blob, and most HTTP servers do).

Layout can be checked with `gdalinfo` or rasterio’s `src.profile` (e.g. `blockxsize`, `blockysize`).

---

## Best practices

- **Filter early**: Use `bbox`, `datetime`, and `query` (e.g. cloud cover) to keep the result set small.
- **Respect limits**: Use `max_items` or pagination to avoid pulling thousands of items.
- **Sign once**: Then open multiple assets from the same item.
- **Window reads**: Transform lon/lat into the raster CRS before `from_bounds`.
- **Check licenses**: Catalogs and collections document license and attribution requirements.

---

## Summary

| Concept | Detail |
|--------|--------|
| STAC | Catalog of geospatial assets with bbox, time, and asset links |
| STAC API | HTTP API to search collections/items |
| pystac-client | Python client to open catalog and run `.search()` |
| Item → assets | Each item has `assets`; each asset has `href` (e.g. COG URL) |
| Signing | For Planetary Computer, `planetary_computer.sign(item)` before using `href` |
| COG | GeoTIFF with tiling + overviews, optimized for HTTP range requests |
| Streaming | Client requests only the byte ranges for the window/bands needed |

In the next lesson we model rasters as labeled **xarray** data cubes.

---

Download the notebook for this lesson: `notebooks/01-stac-cog.ipynb`.
