---
title: Chronographic Isochrones ⏱️🧭
tags: [OSMnx, Routing, Python, Geospatial, Data Visualization]
style: fill
color: light
description: Exploration of automatic generation of chronographic isochrones with Python 🐍 and OSMnx 🛣️.
---

{% assign video_urls = "/assets/images/blog/chrono_isochrone/wroclaw_animation_10m.mp4,/assets/images/blog/chrono_isochrone/wroclaw_animation_10m.webm" | split: ',' %}
{% include posts/native_video.html id="chrono-video" poster="/assets/images/blog/chrono_isochrone/1500_padded.png" urls=video_urls caption="Final animation showing isochrone change up to 1500 meters." %}

## Inspiration

Last year I saw an article by [John Nelson](https://www.esri.com/arcgis-blog/author/j_nelson) from ESRI about [Time-warped walkability mapping in ArcGIS Pro](https://www.esri.com/arcgis-blog/products/arcgis-pro/mapping/time-warp-walkability-mapping-in-arcgis-pro) ([LinkedIn post](https://www.linkedin.com/posts/johnmnelson_heres-how-to-make-a-time-warped-walkability-activity-7358137015327985664-7lTN?utm_source=share&utm_medium=member_desktop&rcm=ACoAAB4C-1oBjSoQnpHjRs32Y4HNXkKZcoPnGyQ)).

{% include elements/figure.html image="/assets/images/blog/chrono_isochrone/johnnelson_timewarp.jpg" caption="Time-warped walkability map. Credit: John Nelson." %}

Later I have also stumbled upon [Katie Walker](https://www.linkedin.com/in/kqwalker/)'s LinkedIn [post](https://www.linkedin.com/posts/kqwalker_30daymapchallenge-gis-arcgispro-activity-7392576575428378624-MG9z?utm_source=social_share_send&utm_medium=member_desktop_web&rcm=ACoAAB4C-1oBjSoQnpHjRs32Y4HNXkKZcoPnGyQ) during the #30DayMapChallenge and decided to give it a try myself.

There is one problem though - I don't use ArcGIS. Or even QGIS for that matter. I code everything geo 🌍 related in Python (or SQL) and visualize it in Jupyter notebooks, or Kepler.

---

If you want to skip reading about the creation process and see the results, click [here](#results).

You can also run the code locally (or on Google Colab) using [this Jupyter notebook](https://github.com/RaczeQ/isochrone-transform/blob/main/generate_isochrones.ipynb).

## Thinking process

---

{% include elements/highlight.html text="Disclaimer" %}

> During testing I have wrongly named my isochrones "**_Chronographic_**", because they were not time based, but distance based. I noticed it while writing this blog post and there is just too much effort for me to change every visualistion title now.

> In the final results and notebook, I have applied automatic conversion from minutes to meters based on a constant walking speed so that the results are truly chronographic.

---

From the beginning I knew that using [OSMnx](https://github.com/gboeing/osmnx) by [Geoff Boeing](https://geoffboeing.com/) is a no-brainer (I know the that there is a [city2graph](https://github.com/c2g-dev/city2graph) library created by [Yuta Sato](https://ysato.blog/) which can build street network from Overture Maps data, but I didn't have time to explore it yet).

The only thing that I wasn't sure how to tackle, was the whole isochrone boundary generation from the graph.

I have seen some articles or tutorials on this matter:

- [OSMnx isolines example notebook](https://github.com/gboeing/osmnx-examples/blob/main/notebooks/13-isolines-isochrones.ipynb) - showing simple convex hull based isochrones and also some buffered geometries.
{% assign osmnx_images_urls = "/assets/images/blog/chrono_isochrone/isolines_osmnx_convex_hull.png,/assets/images/blog/chrono_isochrone/isolines_osmnx_buffer.png" | split: ',' %}
{% include posts/figure_multiple_images.html urls=osmnx_images_urls caption="Example isochrones (convex hull and buffer) generated with OSMnx notebook." %}

- [Isochrones in Python](https://towardsdatascience.com/isochrones-in-python-fe21814e5cb1/) by [Milan Janosov](https://www.janosov.com/) - also focusing on simple convex hull isochrones.
- [Relative time map](https://computinggeographically.org/chapters/chap6/fig6-14-sb-drive-time.html) by [David O'Sullivan](https://geospatialstuff.com/) - showing some time-space manipulation inside convex hull-based isochrones.
- [Street level isochrones](https://geospatialstuff.com/posts/2025-08-20-street-level-isochrones/#isochrones-like-streets-matter) also by David O'Sullivan - this is the first example where I saw concave hull operation being used.

<div class="small_img">
{% include elements/figure.html image="/assets/images/blog/chrono_isochrone/davidosullivan_street_isochrone.png" caption="Example of the combined street isochrone calculated using concave hull operation. Credit: David O'Sullivan." %}
</div>

Based on those examples I knew that I would like to avoid using convex hull operation if possible. In my mind, I wanted to achieve isochrones similar to the typical outputs from routing engines like [Valhalla](https://github.com/valhalla/valhalla) (developed by Mapzen).

<div class="small_img">
{% include elements/figure.html image="/assets/images/blog/chrono_isochrone/mapbox_playground_example.png" caption="Mapbox Isochrones API Playground result (it uses Valhalla under the hood)." %}
</div>

### Getting the streets network

To create any isochrones, I needed to have a street network in the form of a graph. Naturally, I have used the [OSMnx](https://github.com/gboeing/osmnx) library, which was created exactly for this reason.

I started by defining the center point for the isochrones and the max distance they should cover.

```python
# Old Market in Wrocław, Poland
center_point = Point(17.03291465713426, 51.10909801275284)
# I started with pure meters, later switched to minutes
distance_meters = 500
```

Using those values, it's very easy to download the graph for the selected area of interest using the [`osmnx.graph.graph_from_point`](https://osmnx.readthedocs.io/en/stable/user-reference.html#osmnx.graph.graph_from_point) function. But I also wanted to download some building geometries for the visualizations, so I created my own buffer around the point and used the [`osmnx.graph.graph_from_polygon`](https://osmnx.readthedocs.io/en/stable/user-reference.html#osmnx.graph.graph_from_polygon) function.

```python
import osmnx as ox

# transform to UTM CRS in meters
projected_point, projected_crs = ox.projection.project_geometry(center_point)
# buffer it and reproject back to WGS84 (lon/lat coordinates)
query_buffer, _ = ox.projection.project_geometry(
    projected_point.buffer(distance_meters),
    crs=projected_crs,
    to_latlong=True,
)

# download the graph for the given polygon
G = ox.graph_from_polygon(
    query_buffer, # inside, the polygon is buffered by additional 500 meters
    network_type="walk",
    truncate_by_edge=True,
)
```

To download buildings, I used my library called [QuackOSM](https://github.com/kraina-ai/quackosm) for downloading OSM data at scale. Of course you could also use OSMnx to do that, especially at small scale like this.

```python
import quackosm as qosm

buildings = qosm.convert_geometry_to_geodataframe(
    geometry_filter=query_buffer, tags_filter={"building": True}
)
# OSMnx code alternative
# buildings = ox.features.features_from_polygon(
#     polygon=query_buffer, tags={"building": True}
# )
```

<div class="small_img">
{% include elements/figure.html image="/assets/images/blog/chrono_isochrone/wro_500_graph.png" caption="Downloaded street graph (orange) and buildings (black outlines) within the 500 meters buffer." %}
</div>

After acquiring the graph, we can find the closest node to our center point and save it for future calculations.

```python
from shapely import Point

# ID of the node in the graph
center_node_id = ox.nearest_nodes(G, X=center_point.x, Y=center_point.y)
# Point geometry of this node
center_node_point = Point(G.nodes[center_node_id]["x"], G.nodes[center_node_id]["y"])
```

### Clipping the network

It should be obvious that there is a difference between reachable walking distance and distance in a straight line without any obstructions. If you could fly, then the circle buffer around the center point would represent the area that you can cover. Howerever we usually walk on predefined paths and we have to clip the graph by the reachable distance.

Fortunately, OSMnx has also a function exactly for this use-case: [`osmnx.truncate.truncate_graph_dist`](https://osmnx.readthedocs.io/en/stable/user-reference.html#osmnx.truncate.truncate_graph_dist). By default, OSMnx calculates the length of each edge in meters during creation and it allows us to find exact paths lengths.

```python
subgraph = ox.truncate.truncate_graph_dist(
    G, center_node_id, distance_meters, weight="length"
)
```

{% include elements/figure.html image="/assets/images/blog/chrono_isochrone/graph_clip_buffer_distance_comparison.png" caption="Difference between graph clipped by buffer and by the reachable distance." %}

There is a slight problem however. OSMnx clips graph at nodes, so the distance to most of the end nodes is below required distance (example: the edge start is 496 meters from the center and the edge end is 507 meters from the center - this edge won't be included). I would like to calculate the isochrone with the _exact_ distance, so I wrote some additional code that checks the edges reaching beyond the expected distance and clips them based on the remaining distance.

<details>
    <summary>See the code for extending the edges</summary>
    {%- gist 0b08ef50c3a47e433a3874d831c113fa %}
</details>

{% include elements/figure.html image="/assets/images/blog/chrono_isochrone/graph_clip_edges_extensions.png" caption="Edges extensions after clipping by the reachable distance." %}

### Finding the boundary

After preparing the edges with their geometries, it is now time to prepare the boundary that will define the isochrone.

The simplest option is to use the convex hull operation.

<div class="small_img">
{% include elements/figure.html image="/assets/images/blog/chrono_isochrone/wro_convex_hull.png" caption="Convex hull boundary." %}
</div>

The obvious problem with this approach is that it captures too big of an area and the boundary if farther than expected distance.

I tried to follow the Davis O'Sullivan's approach with the _concave_ hull operation. It tries to wrap all the extending points with a boundary, ideally without leaving such big distances between the boundary and wrapped geometry.

After seeing the gaps and the boundary wrapping back inside the geometry, I had an idea to project the rays from the center point and locate the farthest intersection point and then combine those into the boundary.

{% assign concave_hull_images_urls = "/assets/images/blog/chrono_isochrone/concave_hull_test.png,/assets/images/blog/chrono_isochrone/rays_test_1.png,/assets/images/blog/chrono_isochrone/rays_test_2.png" | split: ',' %}
{% include posts/figure_multiple_images.html urls=concave_hull_images_urls caption="Concave hull tests with rays." %}

This approach has a one problem though - concave hull function has a parameter that should probably be tuned for a given example. I wanted to implement a solution that will be working anywhere automatically, and I experimented with automatic increasing of the `ratio` parameter until the ray intersects only once for each angle. Unfortunately, the sudden jumps between closer and farther points created unsatisfying final results, so I was looking for another approach.

---

My next idea was to select all the end points of the farthest edges from the clipped graph and combine them into a polygon. I assumed that none of the nodes were located at a perfect clipping distance, so I took only the edges extensions into consideration.

I selected all the end points of the clipped edges, sorted them by the angle and combined into a boundary.

{% assign end_points_1_images_urls = "/assets/images/blog/chrono_isochrone/end_points_test_1.png,/assets/images/blog/chrono_isochrone/edge_points_test_1_zoom.png" | split: ',' %}
{% include posts/figure_multiple_images.html urls=end_points_1_images_urls caption="First test with the edges end points." %}

As you can see there are clearly parts of the graph that are outside the boundary. To fix it, I used the `polygonize` operation on the union of graph edges and the previous boundary.

{% assign end_points_2_images_urls = "/assets/images/blog/chrono_isochrone/end_points_test_2.png,/assets/images/blog/chrono_isochrone/edge_points_test_2_zoom.png" | split: ',' %}
{% include posts/figure_multiple_images.html urls=end_points_2_images_urls caption="Second test with the edges end points. Blue regions are the result of the polygonize operation, white edge is the old boundary and the edge orange is the new boundary." %}

<details>
    <summary>See the code for finding the boundary</summary>
    {%- gist 936dce120eca3bc4490ad1a1bcb494f8 %}
</details>

---

I am aware that this approach is also not perfect. Here are the issues that I have encountered:

- Sometimes the final boundary becomes a multipolygon - to fix it I just keep the biggest component only.
- The boundary for the bigger distance sometimes is intersecting with the boundary of the smaller distance - to fix it I union it with the smaller boundary.

These are quite signifact pitfalls in this approach, but I didn't have more time to try and fix them. If it bothers you and you have some free time to spare, you are welcome to try and fix it 😉

### Clipping geometries

To plot the edges and building geometries, I just clip it with the boundaries geometries from the previous step. It is a quite trivial operation. Edges are already clipped to the required reachable distance.

```python
# clip the buildings with the boundary geometry and explode MultiPolygons into Polygons
clipped_buildings = buildings.clip(isochrone_boundary).explode()
# keep only Polygons for the visualisation
clipped_buildings = clipped_buildings[clipped_buildings.geom_type == "Polygon"]
```

<div class="small_img">
{% include elements/figure.html image="/assets/images/blog/chrono_isochrone/clipped_buildings.png" caption="Buildings clipped by the boundary." %}
</div>

### Transforming geometries

### Multiple isochones at once

{% include elements/figure.html image="/assets/images/blog/chrono_isochrone/isochrones_100_500.png" caption="Geographic and Chronographic isochrones in five bands from 100 to 500 meters." %}

## Results {#results}

You can generate the same plots for the region of your choice using [this notebook](https://github.com/RaczeQ/isochrone-transform/blob/main/generate_isochrones.ipynb). <br/>
You can also run it on [Google Colab](https://colab.research.google.com/github/RaczeQ/isochrone-transform/blob/main/generate_isochrones.ipynb).

### London - Big Ben

{% include elements/figure.html image="/assets/images/blog/chrono_isochrone/london_default_chrono.png" caption="Geographic and Chronographic Isochrones for London.<br/>Center point: Big Ben." %}

{% include elements/figure.html image="/assets/images/blog/chrono_isochrone/london_colour_chrono.png" caption="Rainbow coloured Geographic and Chronographic Isochrones for London.<br/>Center point: Big Ben." %}

---

And a more "dense" example with isochrones every 1 minute. Above 10 isochrones, the plotting style is automatically altered and no labels are visible.

{% include elements/figure.html image="/assets/images/blog/chrono_isochrone/london_dense_default_chrono.png" caption="Geographic and Chronographic Isochrones for London.<br/>Center point: Big Ben." %}

{% include elements/figure.html image="/assets/images/blog/chrono_isochrone/london_dense_colour_chrono.png" caption="Rainbow coloured Geographic and Chronographic Isochrones for London.<br/>Center point: Big Ben." %}

### New York - Times Square

Notice how the isochrones are more diamond-shaped vs London. That's what you get from the rectangular city blocks.

{% include elements/figure.html image="/assets/images/blog/chrono_isochrone/nyc_default_chrono.png" caption="Geographic and Chronographic Isochrones for New York City.<br/>Center point: Times Square." %}

{% include elements/figure.html image="/assets/images/blog/chrono_isochrone/nyc_colour_chrono.png" caption="Rainbow coloured Geographic and Chronographic Isochrones for New York City.<br/>Center point: Times Square." %}

### Paris - Arc de Triomphe

I was interested in how the roads spread out from this iconic roundabout. Unfortunately, there is no street edge / node exactly in the center of the Charles de Gaulle Square available in the street network, so the center is slightly shifted.

You can also see how circular the isochrones are.

{% include elements/figure.html image="/assets/images/blog/chrono_isochrone/paris_default_chrono.png" caption="Geographic and Chronographic Isochrones for Paris.<br/>Center point: Arc de Triomphe." %}

{% include elements/figure.html image="/assets/images/blog/chrono_isochrone/paris_colour_chrono.png" caption="Rainbow coloured Geographic and Chronographic Isochrones for Paris.<br/>Center point: Arc de Triomphe." %}

### Tokyo - Shibuya crossing

I also wanted to include most popular crossing in the world - the Shibuya Scramble Crossing, located in Tokyo, near Shibuya Station.

{% include elements/figure.html image="/assets/images/blog/chrono_isochrone/shibuya_default_chrono.png" caption="Geographic and Chronographic Isochrones for Shibuya, Tokyo.<br/>Center point: Shibuya crossing." %}

{% include elements/figure.html image="/assets/images/blog/chrono_isochrone/shibuya_colour_chrono.png" caption="Rainbow coloured Geographic and Chronographic Isochrones for Shibuya, Tokyo.<br/>Center point: Shibuya crossing." %}

## Summary

#### What I learned

<!-- The main thing I learned during this project was how to write apps with Streamlit.

The data wrangling part (getting all the building shapes into a common space and finding the proper rotation) was a nice mind-puzzle to figure out, but wasn't as hard to achieve with currently available tools in Python.

For the visualization part, I initially tried to use Matplotlib 3D surface plot functions, but the results weren't satisfactory, so I switched to Plotly.

As an additional feature, I was thinking about transforming the generated heightmap into an STL file for 3D printing purposes if anyone wishes to do that. I've checked some sources on how to transform heightmaps into 3D objects, but decided not to pursue this path for now.

In the end, I am satisfied with the visualisations generated, and with the interactive application I deployed on Streamlit. -->

#### Used libraries

<!-- - [OvertureMaestro](https://github.com/kraina-ai/overturemaestro) - for downloading Overture Maps data for a given city and for geocoding the string to a geometry.
- [GeoPandas](https://github.com/geopandas/geopandas) - for geospatial operations on a dataset of downloaded buildings and I/O.
- [Shapely](https://github.com/shapely/shapely) - for rotations and translations of a building geometry.
- [Rasterio](https://github.com/rasterio/rasterio) - for rastering the buildings dataset into a heightmap.
- [Plotly](https://github.com/plotly/plotly.py) - for displaying 3D surface plot using heightmap data.
- [NumPy](https://github.com/numpy/numpy) - for 2D array (heightmap) manipulation.
- [PyArrow](https://github.com/apache/arrow) - for parquet file batch processing.
- [PyPalettes](https://github.com/JosephBARBIERDARNAL/pypalettes) - for easy access to many palletes.
- [Matplotlib](https://github.com/matplotlib/matplotlib) - for transforming palettes.
- [streamlit-folium](https://github.com/randyzwitch/streamlit-folium) - a third party Streamlit component for displaying a Folium map, used for preview geocoded geometry to the user.
- [Lonboard](https://github.com/developmentseed/lonboard) - for displaying geocoded geometry in the Jupyter Notebook. -->

#### Social media mentions

<!-- Streamlit:

- [Deployed app](https://city-summit.streamlit.app/)
- [Show the Community! discuss post](https://discuss.streamlit.io/t/city-summit-generate-building-data-visualization-for-your-city/88248) -->

<!-- LinkedIn:
- [First results](https://www.linkedin.com/posts/raczyckikamil_geospatial-geo-overturemaps-activity-7273033425983328256-0U5x?utm_source=share&utm_medium=member_android)
- [First 3D matplotlib visualization](https://www.linkedin.com/posts/raczyckikamil_opensource-geo-geospatial-activity-7273508171153887232-EtOb?utm_source=share&utm_medium=member_android)
- [Rotating 3D Plotly animation](https://www.linkedin.com/posts/raczyckikamil_geo-geospatial-urban-activity-7273853844806074368-rfth?utm_source=share&utm_medium=member_android)
- [Streamlit deploy mention](https://www.linkedin.com/posts/raczyckikamil_overturemaps-geo-geospatial-activity-7275679432143446016-0d57?utm_source=share&utm_medium=member_android)
- [New features update](https://www.linkedin.com/posts/raczyckikamil_city-summit-generator-activity-7276026160747020288-HmeL?utm_source=share&utm_medium=member_android) -->

<!-- Bluesky:
- [First results](https://bsky.app/profile/raczeq.bsky.social/post/3ld2of3xrnc2v)
- [Rotating animation](https://bsky.app/profile/raczeq.bsky.social/post/3ldfwqdrdlk2h)
- [Streamlit deploy mention](https://bsky.app/profile/raczeq.bsky.social/post/3ldrnj4uk6k2p) -->
