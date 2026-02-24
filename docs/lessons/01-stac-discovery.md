# STAC — Discovery

**Goal:** Use the SpatioTemporal Asset Catalog (STAC) to discover raster assets by location, time, and metadata—without downloading full datasets.

---

## What is STAC?

**STAC** (SpatioTemporal Asset Catalog) is an open specification for describing geospatial assets (imagery, derived rasters, etc.) in a consistent way. A **STAC Catalog** exposes:

- **Collections**: groups of assets (e.g. “Sentinel-2 L2A”, “Landsat C2 L2”).
- **Items**: individual scenes or products, each with:
  - **Spatial extent**: `bbox` and/or `geometry`.
  - **Temporal extent**: `datetime` or interval.
  - **Assets**: links to actual files (e.g. COG URLs) with roles (e.g. `visual`, `data`, `metadata`).

By querying a STAC API, you get **metadata and URLs**; you do not download the imagery until you open it (e.g. with rioxarray).

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
search = catalog.search(
    collections=["sentinel-2-l2a"],
    bbox=[-122.5, 37.2, -122.0, 37.6],  # [min_lon, min_lat, max_lon, max_lat]
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

The `href` for a COG is what you will pass to **rioxarray** or **rasterio** to open and stream the raster.

---

## Signing URLs (Planetary Computer)

Microsoft Planetary Computer uses **signed URLs** so that access is time-limited. Use the `planetary_computer` package to sign asset URLs:

```python
import planetary_computer

for item in items:
    item = planetary_computer.sign(item)
# Now item.assets["..."].href are signed and can be used by rioxarray
```

---

## Best practices

- **Filter early**: Use `bbox`, `datetime`, and `query` (e.g. cloud cover) to keep the result set small.
- **Respect limits**: Use `max_items` or pagination to avoid pulling thousands of items.
- **Reuse signed items**: Sign once, then open multiple assets from the same item.
- **Check licenses**: Catalogs and collections document license and attribution requirements.

---

## Summary

| Concept | Detail |
|--------|--------|
| STAC | Catalog of geospatial assets with bbox, time, and asset links |
| STAC API | HTTP API to search collections/items |
| pystac-client | Python client to open catalog and run `.search()` |
| Item → assets | Each item has `assets`; each asset has `href` (e.g. COG URL) |
| Signing | For PC, use `planetary_computer.sign(item)` before using `href` |

In next lesson we use those asset URLs to open rasters as **Cloud-Optimized GeoTIFFs** and stream only the data we need.

---

Download the notebook for this lesson: `notebooks/01-stac-discovery.ipynb`.
