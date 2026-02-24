# Cloud-Optimized GeoTIFF (COG)

**Goal:** Understand how Cloud-Optimized GeoTIFFs enable **streaming** and **windowed reads** over HTTP, so you only transfer the bytes you need.

---

## What is a Cloud-Optimized GeoTIFF?

A **GeoTIFF** is a TIFF file with georeferencing (CRS, extent, resolution) in the same container. A **Cloud-Optimized GeoTIFF (COG)** is a GeoTIFF organized so that:

1. **Tiling**: The image is stored in small tiles (e.g. 512×512).
2. **Overview pyramids**: Lower-resolution overviews are embedded.
3. **Metadata and layout**: Key metadata and the tile index are near the start of the file.

When the file is served over HTTP (e.g. from S3 or Azure Blob), a client can issue **HTTP range requests** to read only specific byte ranges. So instead of downloading a 2 GB scene, you can read a single tile or a small window—**streaming** only what you need.

---

## Why this matters for analytics

- **Band selection**: Read only the bands you need.
- **Spatial windowing**: Read a bounding box (e.g. your study area) instead of the full scene.
- **Overview levels**: For quick previews or coarse analysis, read from an overview.
- **Lazy loading**: Libraries like **rasterio** and **rioxarray** can open the URL and only fetch data when you actually read a window or trigger computation.


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

With **rasterio**, you open the URL as if it were a path. The driver uses the appropriate backend (e.g. GDAL’s `/vsicurl/` or cloud-specific handlers) to perform range requests.

**Get a COG URL from STAC** (e.g. signed Planetary Computer URL):

```python
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
```

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
left, bottom, right, top = -122.35, 37.65, -122.30, 37.70

with rasterio.open(url) as src:
    window = from_bounds(left, bottom, right, top, src.transform)
    data = src.read(1, window=window)
    print("Window shape:", data.shape)
```

**Optional: plot the window**

```python
import matplotlib.pyplot as plt

with rasterio.open(url) as src:
    window = from_bounds(left, bottom, right, top, src.transform)
    data = src.read(1, window=window)

fig, ax = plt.subplots(1, 1, figsize=(6, 5))
ax.imshow(data, cmap="gray")
ax.set_title("COG window (B04) — streamed only")
plt.tight_layout()
plt.show()
```

With **rioxarray**, you will do the same conceptually: `rioxarray.open_rasterio(url)` and then slice or use `.sel()` so that only the required window is read.

---

## Requirements for a “true” COG

For efficient streaming, the file should:

- Be **tiled** (not strip-based).
- Have **overview** levels if you want fast coarse reads.
- Be served from a store that supports **HTTP Range** (S3, Azure Blob, and most HTTP servers do).

You can check with `gdalinfo` or rasterio’s `src.profile` (e.g. `blockxsize`, `blockysize`).

---

## Summary

| Concept | Detail |
|--------|--------|
| COG | GeoTIFF with tiling + overviews, optimized for HTTP range requests |
| Streaming | Client requests only the byte ranges for the window/bands needed |
| Benefit | No full-file download; band and spatial subsetting over the network |
| In Python | rasterio / rioxarray open URL; reads trigger range requests |


--- 

Download the notebook for this lesson: `notebooks/02-cog-streaming.ipynb`.
