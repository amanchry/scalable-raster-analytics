# Environment Setup

This page configures an environment for the **Scalable Raster Analytics** workshop: STAC discovery, Cloud-Optimized GeoTIFF (COG) streaming, xarray data cubes, rioxarray, and Dask for parallel, chunked processing.

---

### 1. Install Conda (Recommended)

Conda is a cross-platform environment manager that makes it easy to install geospatial libraries and keep the workshop stack isolated.

#### Windows

1. Download the **Miniconda installer**:  
   [Miniconda Windows 64-bit](https://docs.conda.io/en/latest/miniconda.html#windows-installers)
2. Run the installer and choose “Add Miniconda to PATH” during setup.
3. Open **Anaconda Prompt** or **Command Prompt** and verify:

```bash
conda --version
```

---

#### macOS

1. Download the installer for your chip (Intel or Apple Silicon):  
   [Miniconda macOS](https://docs.conda.io/en/latest/miniconda.html#macos-installers)
2. Run the installer.
3. Restart the terminal and verify:

```bash
conda --version
```

---

### 2. Create a Conda Environment

Use a dedicated environment so the workshop stack does not conflict with other projects:

```bash
conda create --name geoenv python=3.10
conda activate geoenv
```

---

### 3. Install the Workshop Stack

#### Option A: Conda (recommended)

Install the core geospatial and scientific stack from `conda-forge`:

```bash
conda install -c conda-forge rasterio xarray rioxarray dask pystac-client \
  numpy geopandas notebook ipykernel
```

Optional: for [Microsoft Planetary Computer](https://planetarycomputer.microsoft.com/) signed URLs in the lessons:

```bash
pip install planetary-computer
```

#### Option B: Pip only

If you prefer not to use Conda for libraries:

```bash
pip install numpy xarray rioxarray dask pystac-client geopandas
```



---

### 4. Jupyter Kernel (Optional)

To use this environment as a Jupyter kernel:

```bash
conda activate geoenv
conda install -c conda-forge notebook ipykernel
python -m ipykernel install --user --name=geoenv --display-name "Python (geoenv)"
```

In Jupyter: **Kernel → Change Kernel → Python (geoenv)**.

To remove the kernel later:

```bash
jupyter kernelspec uninstall geoenv
```

---


