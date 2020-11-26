---
layout: post
title:  "How to aggregate geometries to higher-order geometries with GeoPandas overlay"
date:   2020-11-26
tag:
  - geopandas
  - spatial
---

GeoPandas offers some very convenient functionalities for users wanting to compare multiple spatial datasets. See their user guide [here](https://geopandas.org/set_operations.html). However, while GeoPandas's illustrated examples are very useful, the instructions would benefit from an example of what one can do beyond applying the overlay operation. 

I ran into this while working on a project for which I needed to map spatial data on a grid, to the shapes of counties. As an additional rule, I only wanted border cells mapped to those counties that have the highest share of area. Let's illustrate the problem:


```python
import matplotlib.pyplot as plt
import geopandas
from shapely.geometry import Polygon

regions_pol = geopandas.GeoSeries(
    [
        Polygon([(0, 0), (2.6, 0), (2.6, 1.9), (0, 1.9)]),
        Polygon([(2, 2), (4, 2), (4, 4), (2, 4)])
    ]
)
cells_pol = geopandas.GeoSeries(
    [
        Polygon([(1, 1), (3, 1), (3, 3), (1, 3)]),
        Polygon([(3, 1), (5, 1), (5, 3), (3, 3)])
    ]
)

regions = geopandas.GeoDataFrame(
  {"geometry": regions_pol, "region":[1,2]}
)
cells = geopandas.GeoDataFrame(
  {"geometry": cells_pol, "cell":[1,2]}
)
intersection = geopandas.overlay(regions, cells, how="intersection")
```


```python
ax = regions.plot(cmap="Paired")
cells.plot(ax=ax, color='green', edgecolor="black", alpha=0.5)
intersection.plot(ax=ax, color="None", edgecolor="yellow", lw=2)
regions.apply(
    lambda x: ax.annotate(
        text=f"region_{x.region}",
        xy=x.geometry.centroid.coords[0],
        ha="center",
        fontsize=14,
    ),
    axis=1,
)
cells.apply(
    lambda x: ax.annotate(
        text=f"cell_{x.cell}",
        xy=x.geometry.centroid.coords[0],
        ha="center",
        fontsize=14,
    ),
    axis=1,
)
plt.show()
```


    
![png](/assets/img/geopandas_overlay_2_0.png)
    


Here, I plot two hypothetical administrative regions, and overlay them with two cells of a grid in green. I highlight the overlapping regions with a yellow edgecolor.

What I want to do here is assign cell `cell_1` to `region_1`, given that `region_1` has a bigger share of the area than `region_2`. And `cell_2` obviously should get assigned to `region_2` since that's the only region there. 

To highlight the overlay, I already sneaked in GeoPandas's overlay function to the code above, overlaying the regions with the cells. This returns a geodataframe for combinations of our regions and cells. Using the geometry column, we can add the area for each of these:


```python
intersection["area"] = intersection.area
intersection
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>region</th>
      <th>cell</th>
      <th>geometry</th>
      <th>area</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>1</td>
      <td>POLYGON ((1.00000 1.90000, 2.60000 1.90000, 2....</td>
      <td>1.44</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2</td>
      <td>1</td>
      <td>POLYGON ((2.00000 2.00000, 2.00000 3.00000, 3....</td>
      <td>1.00</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2</td>
      <td>2</td>
      <td>POLYGON ((4.00000 3.00000, 4.00000 2.00000, 3....</td>
      <td>1.00</td>
    </tr>
  </tbody>
</table>
</div>



We can then use this information to decide which region each cell belongs to. First, index the spatial data to the combinations. Then group by the cell and get the index for the max area:


```python
intersection = intersection.set_index(["region", "cell"])
assigned = intersection.loc[
    intersection.groupby(["cell"])["area"].idxmax()
]
assigned
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th></th>
      <th>geometry</th>
      <th>area</th>
    </tr>
    <tr>
      <th>region</th>
      <th>cell</th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1</th>
      <th>1</th>
      <td>POLYGON ((1.00000 1.90000, 2.60000 1.90000, 2....</td>
      <td>1.44</td>
    </tr>
    <tr>
      <th>2</th>
      <th>2</th>
      <td>POLYGON ((4.00000 3.00000, 4.00000 2.00000, 3....</td>
      <td>1.00</td>
    </tr>
  </tbody>
</table>
</div>



That gives us a multi-index of the assigned cell-regions, which we can use as a lookup table of sorts:


```python
assigned = assigned.reset_index()[["region", "cell"]]
assigned
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>region</th>
      <th>cell</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>1</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2</td>
      <td>2</td>
    </tr>
  </tbody>
</table>
</div>



And that information, finally, we can merge into our regions:


```python
assigned_cells = cells.merge(assigned, on="cell")
assigned_cells
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>geometry</th>
      <th>cell</th>
      <th>region</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>POLYGON ((1.00000 1.00000, 3.00000 1.00000, 3....</td>
      <td>1</td>
      <td>1</td>
    </tr>
    <tr>
      <th>1</th>
      <td>POLYGON ((3.00000 1.00000, 5.00000 1.00000, 5....</td>
      <td>2</td>
      <td>2</td>
    </tr>
  </tbody>
</table>
</div>



Plot that, sorted by region:


```python
ax = regions.plot(cmap="Paired")
assigned_cells.sort_values(by="region").plot(
    ax=ax, cmap="Paired", alpha=0.8, edgecolor="black"
)

assigned_cells.apply(
    lambda x: ax.annotate(
        text=f"cell_{x.cell}", 
        xy=x.geometry.centroid.coords[0], 
        ha='center', 
        fontsize=14
    ), 
    axis=1
)
plt.show()
```


    
![png](/assets/img/geopandas_overlay_12_0.png)
    

