# 🚧 Speed Radars and Road Network Grid Analysis

This GIS project automates the creation of 20km x 20km square grids to analyze road networks and speed radars. It reads shapefiles and CSVs, processes them into geospatial grids, and outputs structured GeoJSON files for mapping and visualization.

---

## 1️⃣ Import Required Libraries

```python
import os
import geopandas as gpd
from shapely.geometry import Point,Polygon,box
from pyproj import CRS
from numpy import arange
```

---

## 2️⃣ Load and Reproject Road Data

```python
road = gpd.read_file(r"inputs\road.shp",encoding="utf8")
road.to_crs(epsg=5179,inplace=True)
road.head(3)
```
|    |    LINK_ID |     F_NODE |     T_NODE |   LANES |   ROAD_RANK |   ROAD_TYPE | ROAD_NO   | ROAD_NAME   |   ROAD_USE |   MULTI_LINK |   CONNECT |   MAX_SPD |   REST_VEH |   REST_W |   REST_H |   LENGTH | REMARK   | geometry                                                                              |
|---:|-----------:|-----------:|-----------:|--------:|------------:|------------:|:----------|:------------|-----------:|-------------:|----------:|----------:|-----------:|---------:|---------:|---------:|:---------|:--------------------------------------------------------------------------------------|
|  0 | 3880778900 | 3880289100 | 3880289500 |       1 |         107 |         000 | -         | 금오14길    |          0 |            0 |       000 |        40 |          0 |        0 |        0 | 134.264  |          | LINESTRING (1138704.411147075 1703375.173355783, 1138719.404664226 1703508.519328477) |
|  1 | 3880779000 | 3880289500 | 3880289000 |       1 |         107 |         000 | -         | 금오14길    |          0 |            0 |       000 |        40 |          0 |        0 |        0 |  40.1724 |          | LINESTRING (1138719.394230216 1703508.432230688, 1138724.463125565 1703548.259988362) |
|  2 | 3880779100 | 3880289000 | 3880289500 |       1 |         107 |         000 | -         | 금오14길    |          0 |            0 |       000 |        40 |          0 |        0 |        0 |  40.1723 |          | LINESTRING (1138712.566136057 1703549.774121644, 1138707.497141144 1703509.946463892) |
---

## 3️⃣ Load Camera Data and Convert to GeoDataFrame

```python
cameras = gpd.read_file(r"inputs\camera.csv",driver="csv",encoding="utf8")
cameras["Longitude"]=cameras["Longitude"].astype(float)
cameras["Latitude"]=cameras["Latitude"].astype(float)
cameras["geometry"]=[Point(xy) for xy in zip(cameras["Longitude"],cameras["Latitude"])]
cameras.crs = CRS.from_epsg(4326).to_wkt()
cameras.to_crs(epsg=5179,inplace=True)
cameras.head(3)
```
|    | ID    | City   |   Direction | Angle   |   Latitude |   Longitude |   Speed Limit |   R_code | geometry                                    |
|---:|:------|:-------|------------:|:--------|-----------:|------------:|--------------:|---------:|:--------------------------------------------|
|  0 | F8863 | 수원시 |           3 | W+E     |    37.2659 |     126.957 |            60 |        2 | POINT (951892.9733604998 1918696.861407467) |
|  1 | F7559 | 용인시 |           1 | SE      |    37.3217 |     127.08  |            50 |        2 | POINT (962779.6532611722 1924826.834853379) |
|  2 | F7569 | 부천시 |           1 | N       |    37.5188 |     126.77  |            30 |        2 | POINT (935499.4257011133 1946858.495053298) |
---

## 4️⃣ Create Square Grid to Cover the Road Network

```python
def Sq_Grid(gdf,length,epsg_code):
    cells=list()
    minx,miny,maxx,maxy = gdf.total_bounds
    for x0 in arange(minx,maxx,length):
        for y1 in arange(maxy,miny,-length):
            x1=x0+length
            y0=y1-length
            cells.append(box(x0,y0,x1,y1))
    return gpd.GeoDataFrame(geometry=cells,crs=CRS.from_epsg(epsg_code))

grid = Sq_Grid(road,20000,5179)
grid["grid_id"]=[ID for ID in range(1,len(grid)+1)]
grid.head(3)
```
|    | geometry                                                                                                                                                                                            |
|---:|:----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|  0 | POLYGON ((876040.0799623571 2045187.046464815, 876040.0799623571 2065187.046464815, 856040.0799623571 2065187.046464815, 856040.0799623571 2045187.046464815, 876040.0799623571 2045187.046464815)) |
|  1 | POLYGON ((876040.0799623571 2025187.046464815, 876040.0799623571 2045187.046464815, 856040.0799623571 2045187.046464815, 856040.0799623571 2025187.046464815, 876040.0799623571 2025187.046464815)) |
|  2 | POLYGON ((876040.0799623571 2005187.046464815, 876040.0799623571 2025187.046464815, 856040.0799623571 2025187.046464815, 856040.0799623571 2005187.046464815, 876040.0799623571 2005187.046464815)) |
### 🧭 Grid Reference (Top-left origin)

![Grid Details](images/grid_details.png)

---

## 5️⃣ Spatial Join and Overlay Analysis

### ➕ Intersect Roads with Grid

```python
road_intersection=gpd.overlay(road,grid,how='intersection')
```

### ➕ Assign Grid ID to Cameras

```python
cameras_grid_id=gpd.sjoin(cameras,grid,how="left",op="within",rsuffix='grid',)
```

---

## 6️⃣ Prepare Grid Boundary for Export

```python
grid["geometry"] = grid["geometry"].boundary
```

---

## 7️⃣ Export Grid Data as GeoJSON Files

```python
path = os.path.join(os.getcwd(),"otuputs")
os.mkdir(path)

for ID in grid["grid_id"]:
    os.mkdir(os.path.join("otuputs",str(ID)))
    grid[grid["grid_id"]==ID].to_file(os.path.join("otuputs",str(ID),"boundary.geojson"),driver="GeoJSON")
    
    if (cameras_grid_id[cameras_grid_id["grid_id"]==ID].empty):
        open(os.path.join("otuputs",str(ID),"cameras.geojson"),"x")
    else:
        cameras_grid_id[cameras_grid_id["grid_id"]==ID].to_file(os.path.join("otuputs",str(ID),"cameras.geojson"),driver="GeoJSON",encoding="utf-8")
    
    if (road_intersection[road_intersection["grid_id"]==ID].empty):
        open(os.path.join("otuputs",str(ID),"roads.geojson"),"x")
    else:
        road_intersection[road_intersection["grid_id"]==ID].to_file(os.path.join("otuputs",str(ID),"roads.geojson"),driver="GeoJSON",encoding="utf-8")
```

---

## 8️⃣ Output Example for Grid ID 158

### 📂 Files Created in Folder `158`

![Files in Grid 158](images/files_in_158.png)

### 🗺️ Visualization of Speed Radars and Roads in Grid 158

![Square Grid 158 Map](images/ID_158.png)

---

## 🔗 Interactive Map Viewer

You can explore the results online at:

👉 [https://speedradars.netlify.app](https://speedradars.netlify.app)

---

### 👨‍💻 **Author**
- **Yasser I. Barhoom**

* **Geomatics Engineer**
