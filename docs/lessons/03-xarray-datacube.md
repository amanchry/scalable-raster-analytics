# xarray & the Data Cube

**Goal:** Model multi-dimensional rasters as **labeled** arrays with xarray: dimensions (e.g. time, band, y, x), coordinates, and attributes.

---

## From raw arrays to a data cube

A single-band 2D raster is just a 2D array (e.g. NumPy). As soon as you have:

- **Multiple bands** (e.g. red, green, blue, NIR),
- **Multiple times** (e.g. monthly composites),
- **Multiple variables** (e.g. reflectance and a mask),

you have a **multi-dimensional** raster. The **data cube** is the conceptual model: a cube (or hypercube) where each axis is a dimension and each point has a label (e.g. time, band name).

**xarray** implements this with:

- **Dimensions**: named axes (e.g. `time`, `band`, `y`, `x`).
- **Coordinates**: arrays that label each point along a dimension (e.g. dates for `time`, band names for `band`).
- **Data variables**: arrays that depend on those dimensions.
- **Attributes**: metadata (e.g. CRS, units) on the dataset or variables.


![xarray](/assets/images/xarray.png)
*Source: Stephan Hoyer, ECMWF presentation.*  
[Link to original talk](https://docs.google.com/presentation/d/16CMY3g_OYr6fQplUZIDqVtG-SKZqsG8Ckwoj2oOqepU/edit#slide=id.g2b68f9254d_1_27)
---

## Core structures

### DataArray

A **DataArray** is a single array with dimensions and coordinates:

```python
import xarray as xr
import numpy as np

# Minimal example: 2D (y, x)
data = np.random.rand(100, 200)
da = xr.DataArray(
    data,
    dims=["y", "x"],
    coords={
        "y": np.linspace(4000, 3000, 100),  # e.g. northing
        "x": np.linspace(1000, 3000, 200),  # e.g. easting
    },
    attrs={"units": "m", "long_name": "elevation"},
)
da.sel(x=1500, method="nearest")
```

### Dataset

A **Dataset** is a dict-like container of **DataArrays** that share dimensions. Typical for rasters: one variable per band or one variable per product, all sharing `y` and `x` (and optionally `time`, `band`).

```python
ds = xr.Dataset(
    {
        "red": (["y", "x"], red_array),
        "nir": (["y", "x"], nir_array),
    },
    coords={"y": y_coord, "x": x_coord},
)
ds["ndvi"] = (ds["nir"] - ds["red"]) / (ds["nir"] + ds["red"])
```

---

## Why labels matter

With **NumPy**, you do `arr[i, j]` or `arr[0:10, :]` and must remember which axis is time or band. With **xarray** you do:

```python
ds.sel(time="2023-06-15", band="red")
ds.sel(x=slice(1000, 2000), y=slice(3000, 4000))
ds.isel(time=0)
```

That makes code **readable** and **less error-prone** when combining multiple scenes or variables.

---

## Dimensions typical in raster cubes

| Dimension | Meaning | Example coordinates |
|-----------|--------|----------------------|
| `x` | Horizontal (easting / lon) | Projected x or longitude |
| `y` | Vertical (northing / lat) | Projected y or latitude |
| `band` | Spectral/thematic layer | `["red", "green", "blue", "nir"]` |
| `time` | Time step | `datetime64` dates |

So a multi-temporal, multi-band stack might have shape `(time, band, y, x)`.

---

## Lazy vs eager

xarray can wrap **Dask** arrays instead of NumPy. Then the array is **lazy**: operations build a task graph; actual reads and compute happen when you call `.compute()` or when you trigger plotting. We use this in Lesson 5 for scalable analytics; in Lesson 4 we open COGs as xarray (with rioxarray) and can make them lazy by default.

---

## Summary

| Concept | Detail |
|--------|--------|
| Data cube | Multi-dimensional raster with labeled dimensions (time, band, y, x) |
| DataArray | Single array + dims + coords + attrs |
| Dataset | Multiple DataArrays sharing dimensions |
| Selection | `.sel()` by coordinate, `.isel()` by index, `.slice()` |
| Lazy | xarray can hold Dask arrays; compute on demand |

In next lesson we use **rioxarray** to open COGs as xarray objects with CRS and spatial coordinates.

---

Download the notebook for this lesson: `notebooks/03-xarray-datacube.ipynb`.
