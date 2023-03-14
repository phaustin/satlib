---
jupytext:
  notebook_metadata_filter: all,-language_info,-toc,-latex_envs
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.14.0
kernelspec:
  display_name: Python 3 (ipykernel)
  language: python
  name: python3
---

+++ {"tags": [], "user_expressions": []}

(rasterio_png)=
# Making a png  composite image

In the cells below I read in bands 3, 4 and 5 from the
`vancouver_345_refl.tiff` that was produced by the
{ref}`rasterio_3bands` notebook. and convert it to a png file so
I can look at it with standard image viewers.   I use "histogram equalization"
from the scikit-image library to boost the color contrast in each of the bands.  I
check the histograms using the jointplot function from the seaborn plotting library
that adds a lot of nice features to matplotlib.  The new normailzed bands are put
together to make a "false color composite" that shows vegetation as purple.

```{code-cell} ipython3
import datetime
from pathlib import Path

import numpy as np
import pytz
import rasterio
import seaborn as sns
from IPython.display import Image
import rioxarray
import copy
from affine import Affine
```

```{code-cell} ipython3
from matplotlib import pyplot as plt
from skimage import exposure, img_as_ubyte
```

```{code-cell} ipython3
import a301_lib  # noqa

pacific = pytz.timezone("US/Pacific")
date = datetime.datetime.today().astimezone(pacific)
print(f"written on {date}")
```

```{code-cell} ipython3

```

```{code-cell} ipython3
notebook_dir = Path().resolve().parent / "notebooks"
data_dir = notebook_dir.parent / "data"
van_tif = list(data_dir.glob("*van*345*tif"))[0]

with rasterio.open(van_tif) as van_raster:
    b3_refl = van_raster.read(1)
    chan1_tags = van_raster.tags(1)
    b4_refl = van_raster.read(2)
    chan2_tags = van_raster.tags(2)
    b5_refl = van_raster.read(3)
    chan3_tags = van_raster.tags(3)
    crs = van_raster.profile["crs"]
    transform = van_raster.profile["transform"]
    tags = van_raster.tags()
print(tags, chan1_tags)
```

+++ {"user_expressions": []}

## Note the low reflectivity for band 3

```{code-cell} ipython3
plt.imshow(b3_refl)
```

+++ {"user_expressions": []}

* Below I do some joint histograms of band3 vs. band 4 to get a feeling for the distribution

```{code-cell} ipython3
sns.jointplot(
    x=b3_refl.flat,
    y=b4_refl.flat,
    xlim=(0, 0.2),
    ylim=(0.0, 0.2),
    kind="hex",
    color="#4CB391",
)
```

```{code-cell} ipython3
sns.jointplot(
    x=b4_refl.flat,
    y=b5_refl.flat,
    kind="hex",
    xlim=(0, 0.3),
    ylim=(0.0, 0.5),
    color="#4CB391",
)
```

+++ {"tags": [], "user_expressions": []}

* I'm okay with changing the data values to get a qualitative feeling
  for the image.  To do this, I can use the [scikit-image equalization module](https://scikit-image.org/docs/dev/auto_examples/color_exposure/plot_equalize.html)

```{code-cell} ipython3
channels = np.empty([3, b3_refl.shape[0], b3_refl.shape[1]], dtype=np.uint8)
for index, image in enumerate([b3_refl, b4_refl, b5_refl]):
    stretched = exposure.equalize_hist(image)
    channels[index, :, :] = img_as_ubyte(stretched)
```

```{code-cell} ipython3
plt.imshow(channels[0, :, :])
```

+++ {"user_expressions": []}

https://seaborn.pydata.org/generated/seaborn.jointplot.html

```{code-cell} ipython3
the_data = {"band3": channels[0, ...].flat, "band4": channels[1, ...].flat}
sns.jointplot(
    x="band3",
    y="band4",
    data=the_data,
    xlim=(0, 255),
    ylim=(0.0, 255),
    kind="hex",
    color="#4CB391",
)
```

+++ {"user_expressions": []}

## Write out the png

Now that I have 3 bands scaled from 0-255, I can write them out as
a png file, with new tags

```{code-cell} ipython3
png_filename = notebook_dir / "vancouver.png"
num_chans, height, width = channels.shape
with rasterio.open(
    png_filename,
    "w",
    driver="PNG",
    height=height,
    width=width,
    count=num_chans,
    dtype=channels.dtype,
    crs=crs,
    transform=transform,
    nodata=0.0,
) as dst:
    chan_tags = [
        "LC8_Band3_refl_counts",
        "LC8_Band4_refl_counts",
        "LC8_Band5_refl_counts",
    ]
    dst.update_tags(**tags)
    dst.update_tags(written_on=str(datetime.date.today()))
    dst.update_tags(history="written by scene_image.md")
    dst.write(channels)
    keys = ["3", "4", "5"]
    for index, chan_name in enumerate(keys):
        chan_name = f"band_{chan_name}"
        valid_range = "0,255"
        dst.update_tags(index + 1, name=chan_name)
        dst.update_tags(index + 1, valid_range=valid_range)
```

+++ {"user_expressions": []}

## View the image

Here's the finished image -- since Band 4 is the wavelength that
plants use for photosynthesis, it's reflectivity values are
very low for vegetated pixels.  Only Band 3 (green now mapped to blue) and
Band 5 (near-ir now mapped to red) are reflecting, which makes purple.

```{code-cell} ipython3
Image(filename=png_filename)
```

+++ {"user_expressions": [], "tags": []}

## move to rioxarray

```{code-cell} ipython3
van_array = rioxarray.open_rasterio(van_tif)
van_array
```

```{code-cell} ipython3
van_array.rio.crs
```

```{code-cell} ipython3
van_array.rio.transform()
```

```{code-cell} ipython3
type(van_array)
```

+++ {"tags": [], "user_expressions": []}

### replace data with histogram stretch

```{code-cell} ipython3
png_array = copy.deepcopy(van_array)
png_array.data = channels.data
```

```{code-cell} ipython3
png_array.plot.imshow()
```

```{code-cell} ipython3
png_array.rio.crs
```

```{code-cell} ipython3
outfile = notebook_dir / "rio_van.png"
png_array.rio.to_raster(outfile)
```

+++ {"user_expressions": [], "tags": []}

## Find the clipping window to download original files into a new dataset

```{code-cell} ipython3
orig_transform = Affine(30.0, 0.0, 399960.0,
                         0.0, -30.0, 5500020.0)
```

```{code-cell} ipython3
rasterio.windows.from_bounds(*png_array.rio.bounds(),transform=orig_transform)
```

```{code-cell} ipython3
Image(outfile)
```
