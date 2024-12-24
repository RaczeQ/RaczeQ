---
title: Visualizing a building shapes taxonomy - The City Summit 🏙️🗻 project
tags: [City Summit, Overture Maps, Python, Geospatial, Data Visualization]
style: fill
color: success
description: Summary of building shapes exploration from the OvertureMaps dataset.
---

{% include elements/figure.html image="https://raw.githubusercontent.com/RaczeQ/RaczeQ/refs/heads/gh-pages/assets/images/blog/city_summit/20_cities_grid.png" caption="Final visualisation for 20 different cities around the world." %}

## Back story

While working on new features for the [OvertureMaestro](https://github.com/kraina-ai/overturemaestro) library, I started to explore the dataset of buildings that is provided by Overture Maps. Remembering an old project focusing on assembling multiple images of women of different ethnicities ([see here](https://theonlinephotographer.typepad.com/the_online_photographer/2013/09/averaged-faces-of-various-nationalities.html)) the question came to mind - *What would the "average" shape of a building in a given city look like?*

So, as is my habit, I started on a side project instead of finishing the functionality I was currently working on.

The results of this project can be accessed on Streamlit: [https://city-summit.streamlit.app/](https://city-summit.streamlit.app/)

![alt text](https://raw.githubusercontent.com/RaczeQ/RaczeQ/refs/heads/gh-pages/assets/images/projects/city_summit.png "City Summit project screenshot")

## Getting the data

Downloading the Overture Maps data for the buildings is quite easy. I have used the `OvertureMaestro` library, but you can also access it with official `overturemaps-py` library as well as using DuckDB SQL query.

```python
from overturemaestro import convert_geometry_to_geodataframe, geocode_to_geometry

buildings = convert_geometry_to_geodataframe(
    "buildings", "building", geometry_filter=geocode_to_geometry("Paris")
)
```

This code will download the data locally as a GeoParquet and return a GeoDataFrame for the user. You can also get the path to the file instead using `convert_geometry_to_parquet` function.

## First tests

To begin with, I knew I had to solve two problems: how to centre all the buildings so that they were in the same space, and how to rotate them in a similar orientation so that the edges of the buildings go mainly vertical and horizontal.

#### Translating building vertices

For the first part, I have used the [Azimuthal Equidistant](https://en.wikipedia.org/wiki/Azimuthal_equidistant_projection) projection, that maintains the distances and azimuths. Each building was projected based on their centroid.

{% include elements/figure.html image="https://upload.wikimedia.org/wikipedia/commons/e/ec/Azimuthal_equidistant_projection_SW.jpg" caption="Example of the Azimuthal Equidistant projection (from Wikipedia)." %}

```python
# Geometry projection to AEQD
import pyproj as proj
from shapely.ops import transform
from shapely import Polygon

# Example building
g = Polygon(
    [
        [17.0251207, 51.0607852],
        [17.0250685, 51.0607947],
        [17.0250616, 51.0607808],
        [17.025115, 51.060771],
        [17.0251207, 51.0607852],
    ]
)

centroid = g.centroid
crs_wgs = proj.Proj("epsg:4326")
aeqd_proj = proj.Proj(
    f"+proj=aeqd +lat_0={centroid.y} +lon_0={centroid.x} +datum=WGS84 +units=m"
)
project = proj.Transformer.from_proj(crs_wgs, aeqd_proj, always_xy=True).transform

geom_proj = transform(project, g)

geom_proj # notice that values are close to zero - above and below, these are in meters
# POLYGON ((2.0498206612070975 0.2566105251930882, -1.6097078406850456 1.3134800558941384, -2.093439190884303 -0.2328869958116028, 1.6502174436552757 -1.3231316731725968, 2.0498206612070975 0.2566105251930882))
```

#### Finding the best building rotation angle

The second part was trickier. How can I determine when the building is "straight" or "right"?

My first idea was to rotate the building for each angle individually from 0 to 89 and for each angle calculate the azimuths of edges, weighted by their length and create some kind of score to determine the best rotation angle.

After thinking things through, and after reviewing the Shapely library documentation, I found a simpler solution.

Using the `minimum_rotated_rectangle` function, I can get a minimum rectangle surrounding a given building, a de facto rotated bounding box. What if I just rotate this rectangle so that it is equal to the real bounding box? The idea was that this configuration should keep the footprint of a building the smallest and most edges should be in horizontal and vertical orientations.

{% include elements/figure.html image="https://raw.githubusercontent.com/RaczeQ/RaczeQ/refs/heads/gh-pages/assets/images/blog/city_summit/rotated_building.png" caption="The difference between original and rotated building with bounding box and a minimum rotated rectangle." %}

To get the right angle, I iterated all the points of the resulting rectangle in turn, counted the azimuth between two points and rotated the shape by the negative value obtained. But in this way I will obtain 4 angles, which will be two pairs in the horizontal and vertical axis. How do I choose one angle from these 4? I decided that I wanted the rotated building to be longer than taller (width >= height), and for the centroid to be in the middle, or to be above half the height of the rectangle.

```python
import math
from shapely import affinity

# Rotation logic
coords = list(geometry.minimum_rotated_rectangle.exterior.coords)

for i in range(len(coords) - 1):
    p1 = coords[i]
    p2 = coords[i + 1]
    angle = math.degrees(
        math.atan2(p2[1] - p1[1], p2[0] - p1[0])
    )  # https://stackoverflow.com/questions/42258637/how-to-know-the-angle-between-two-points
    rotated_geometry = affinity.rotate(geometry, angle=-angle, origin="centroid")

    minx, miny, maxx, maxy = rotated_geometry.bounds
    width = maxx - minx
    height = maxy - miny

    # Building is higher than it is wider
    if round(height, 2) > round(width, 2):
        continue

    # The centroid is in the lower half of the rectangle
    if abs(round(maxy, 2)) > abs(round(miny, 2)):
        continue

    return rotated_geometry

raise RuntimeError("Rotation not found")
```

Now that I think about it, it would be possible to choose the right angle of rotation without rotating the building, based on the length and angle between the diagonals, but I would rather stay with the current implementation.

## Basic visualizations

To start, I used the `plot()` function from the GeoPandas library displaying only the edges of the buildings.

{% include elements/figure.html image="https://raw.githubusercontent.com/RaczeQ/RaczeQ/refs/heads/gh-pages/assets/images/blog/city_summit/all_buildings_edges_only.png" caption="All building shapes outlines visualized." %}

To make these less cluttered, I divided the set of buildings into small and large based on the total area. I also added a slight background colouring for the buildings. 

{% include elements/figure.html image="https://raw.githubusercontent.com/RaczeQ/RaczeQ/refs/heads/gh-pages/assets/images/blog/city_summit/small_buildings.png" caption="Small buildings visualization." %}

{% include elements/figure.html image="https://raw.githubusercontent.com/RaczeQ/RaczeQ/refs/heads/gh-pages/assets/images/blog/city_summit/big_buildings.png" caption="Bigger buildings visualization." %}

I have also experimented with leaving only vertices, without edges.

{% include elements/figure.html image="https://raw.githubusercontent.com/RaczeQ/RaczeQ/refs/heads/gh-pages/assets/images/blog/city_summit/big_buildings_dots.png" caption="Bigger buildings vertices only visualization." %}

## Switch to the heightmap

For me, those previous results weren't satisfactory, so I started experimenting with the `rasterio` library and the `rasterize` functionality for switching vector data into raster data.

```python
import numpy as np
from affine import Affine
from rasterio.features import MergeAlg, rasterize

# 1 px will equal to 1 meter,
# higher values will increase number of pixels per meter
resolution = 1

minx, miny, maxx, maxy = geoseries.total_bounds

canvas_width = int(np.ceil(maxx - minx)) * resolution
canvas_height = int(np.ceil(maxy - miny)) * resolution

canvas = rasterize(
    shapes=geoseries,
    fill=0,
    # Add padding of 2 pixels on each side
    out_shape=(canvas_height + 4, canvas_width + 4),
    # Add values together from many shapes
    merge_alg=MergeAlg.add,
    transform=(
        # Add 2 pixels to move away from edge
        Affine.translation(xoff=minx - 2, yoff=miny - 2)
        # Scale buildings to fit bigger canvas
        * Affine.scale(1 / resolution)
    ),
)

heightmap = np.flipud(canvas) # flip canvas vertically
```

#### Plotting the heightmap

```python
import matplotlib.pyplot as plt
import numpy as np
from matplotlib.colors import LogNorm

fig, axes = plt.subplots(1, 2, figsize=(20, 10), dpi=300)

heightmap_masked = np.ma.masked_where(heightmap == 0, heightmap)

axes[0].imshow(heightmap_masked)
axes[0].set_axis_off()
axes[0].set_title("Linear colour scale")

axes[1].imshow(heightmap_masked, norm=LogNorm())
axes[1].set_axis_off()
axes[1].set_title("Log colour scale")

plt.show()
```

{% include elements/figure.html image="https://raw.githubusercontent.com/RaczeQ/RaczeQ/refs/heads/gh-pages/assets/images/blog/city_summit/heightmap_plt_2d_plot.png" caption="Generated heightmap in linear and logarithmic colour scales." %}

As you can see, the centre of the heightmap is dominated by extremely high values. This makes sense; after all, most buildings in cities have a fairly small footprint. To discover more detail, it is useful to display the values on a logarithmic scale.

#### 3D visualization

Having a heightmap, I began to wonder what it would look like in 3D. Using the Surface plot function from the Plotly library, I was able to generate an interactive visualisation for the calculated data.

{% include elements/figure.html image="https://raw.githubusercontent.com/RaczeQ/RaczeQ/refs/heads/gh-pages/assets/images/blog/city_summit/plotly_surface_linear_scale.png" caption="Basic 3D surface plot." %}

#### Loading the palette

The next step to improve the look was the ability to change the colour palette. For this I used the PyPalettes library with which you can easily access thousands of palettes from various sources.

```python
from pypalettes import load_cmap
import numpy as np

loaded_cm = load_cmap("ag_Sunset", cmap_type="continuous")
# resample to 256 values
interpolated_colours = loaded_cm(np.linspace(0, 1, 256))

# replace first colour with white or black
white = np.array([1, 1, 1, 1])
interpolated_colours[:1, :] = white

# transform the palette to a plotly-compatible format:
# a list of rgb(a) css values
plotly_map = list(
    map(
        lambda c: f"rgb({int(c[0] * 255)}, {int(c[1] * 255)}, {int(c[2] * 255)})",
        interpolated_colours[:, :3]
    )
)
plotly_map
```

{% include elements/figure.html image="https://raw.githubusercontent.com/RaczeQ/RaczeQ/refs/heads/gh-pages/assets/images/blog/city_summit/pypalletes_docs.png" caption="Screenshot from the <a href='https://python-graph-gallery.com/color-palette-finder/' target='_blank'>Python Color Palette Finder</a> website." %}

#### Polishing the visualization

## Streamlit implementation

## Summary

---

This blog post is Work In Progress.

TODO:

- rotation or not - difference

- first visualizations
- iterations on visualizations
- stacking buildings in rasterio
- switching to Plotly 3D - interactive
- refactor to utm crs - faster
- streamlit deploy
- where posted on social media
