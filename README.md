# Scalable Raster Analytics with STAC, xarray & Dask

This README is for **documentation developers** (building, previewing, and deploying the MkDocs site).

Course pages: [amanchry.github.io/scalable-raster-analytics](https://amanchry.github.io/scalable-raster-analytics/).

---

## Quick start (docs site)

### Install documentation dependencies
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
