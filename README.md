# Scalable Raster Analytics with STAC, xarray & Dask

A **course** on cloud-native raster workflows: **STAC** (discovery), **Cloud-Optimized GeoTIFF** (streaming), **xarray** (data cube), **rioxarray** (geospatial I/O), and **Dask** (scalable compute). 



## Quick start
### Install Dependencies
```bash
python3 -m venv .venv

# MacOS/Linux
source .venv/bin/activate   

# Windows: 
.venv\Scripts\activate

pip install -r requirements.txt

```

### Serve Documentation Locally

```bash
mkdocs serve
mkdocs serve --livereload

```

The documentation will be available at `http://localhost:8000`


### Build Documentation

```bash
mkdocs build
```

This creates a `site/` directory with static HTML files.

### Deploy Documentation
This deploys to GitHub Pages

```bash
mkdocs gh-deploy
```





## License

Course content and code: use and adapt with attribution. Data and third-party libraries follow their respective licenses (e.g. Sentinel-2, Planetary Computer terms).
