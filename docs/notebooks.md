# Notebooks

Here are the notebooks for each lesson. Download them and run locally after [Environment Setup](setup.md).

**[Download all notebooks (ZIP)](https://github.com/amanchry/scalable-raster-analytics/archive/refs/heads/main.zip)** — extract the archive and open the `notebooks/` folder.

Browse on GitHub: [notebooks/](https://github.com/amanchry/scalable-raster-analytics/tree/main/notebooks)

| Lesson | Notebook | Description |
|--------|----------|-------------|
| 1. STAC & COG | [01-stac-cog.ipynb](https://github.com/amanchry/scalable-raster-analytics/raw/main/notebooks/01-stac-cog.ipynb) | Search STAC, sign URLs, stream a COG window |
| 2. xarray | [02-xarray-datacube.ipynb](https://github.com/amanchry/scalable-raster-analytics/raw/main/notebooks/02-xarray-datacube.ipynb) | Build and slice a small data cube |
| 3. rioxarray | [03-rioxarray.ipynb](https://github.com/amanchry/scalable-raster-analytics/raw/main/notebooks/03-rioxarray.ipynb) |  Prec + Temp + DEM GeoTIFFs, clip, align, analyse |
| 4. Dask | [04-dask-analytics.ipynb](https://github.com/amanchry/scalable-raster-analytics/raw/main/notebooks/04-dask-analytics.ipynb) | Chunked local precip/temp/DEM, lazy graph, persist, compute |
| 5. Application | [05-end-to-end.ipynb](https://github.com/amanchry/scalable-raster-analytics/raw/main/notebooks/05-end-to-end.ipynb) | Karnataka NDVI app: STAC, COG, xarray, rioxarray, Dask |

After installing dependencies, open the `notebooks/` folder in JupyterLab and start with `01-stac-cog.ipynb`.
