# Environment Setup

Create a **conda** environment and install the Python libraries used in this course. The notebooks run in any Jupyter environment; **JupyterLab** is recommended.

---

## 1. Install Conda

Follow the step-by-step [Conda Installation Guide (Spatial Thoughts)](https://courses.spatialthoughts.com/install-conda.html) to install **Miniconda** for the host operating system.

Official installers: [Miniconda](https://docs.conda.io/en/latest/miniconda.html) — pick Windows, macOS (Intel or Apple Silicon), or Linux.

After installing, **restart the terminal** (Windows: open **Anaconda Prompt** or **Anaconda Powershell Prompt**). Verify:

```bash
conda --version
```

---

## 2. Create an environment

*(Windows)* Search for **Anaconda Powershell Prompt** and launch it.  
*(macOS / Linux)* Open a Terminal window.

Create a dedicated environment so the workshop stack does not conflict with other projects:

```bash
conda create --name geoenv python=3.10 -y
conda activate geoenv
```

The prompt should now start with `(geoenv)`.

---

## 3. Install the workshop packages

Install from `conda-forge`. Copy the block for the host platform.

**Windows** (Anaconda Powershell Prompt)

```powershell
conda install -c conda-forge -y `
  numpy rasterio xarray rioxarray dask `
  pystac-client planetary-computer geopandas pyproj `
  matplotlib jupyterlab ipykernel
```

**macOS / Linux**

```bash
conda install -c conda-forge -y \
  numpy rasterio xarray rioxarray dask \
  pystac-client planetary-computer geopandas pyproj \
  matplotlib jupyterlab ipykernel
```

The local environment is now ready.

### Option: pip only

Pip-only install (activate `geoenv` first):

```bash
pip install numpy rasterio xarray rioxarray dask pystac-client \
  planetary-computer geopandas pyproj matplotlib jupyterlab ipykernel
```

Geospatial wheels can be fragile on Windows; conda-forge is the more reliable path.

---

## 4. Open the notebooks

From the course repository, with `geoenv` active:

```bash
conda activate geoenv
jupyter lab
```

In the file browser, open the `notebooks/` folder and start with `01-stac-cog.ipynb`.

Classic Jupyter Notebook or VS Code / Cursor: **Kernel → Change Kernel → geoenv** (or the Python interpreter from that environment).

To register the kernel explicitly:

```bash
python -m ipykernel install --user --name=geoenv --display-name "Python (geoenv)"
```

---

## 5. Check that it works

In a notebook or `python` prompt:

```python
import pystac_client
import planetary_computer
import rasterio
import rioxarray
import xarray
import dask

print("Ready")
```

A `Ready` print confirms the stack. Continue with [Lesson 1](lessons/01-stac-cog.md).


---

