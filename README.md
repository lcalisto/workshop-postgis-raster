

# **Workshop PostGIS Raster**

### This workshop aims to explain and exemplify the usage of PostGIS raster.



----------
## Create and restore the database
 
 In order to start the workshop we will load an existing database. 

Using pgAdmin please create a new, empty database and then restore the existing dump ***postgis_raster.backup*** from this repository into the new recently created database.


----------
## Database structure

The restored database consists of the folowing structure:

 - schema_name *(you should rename it with your name)*
 - public
 - rasters
 - vectors
	 - railroad (lines)
	 - places (points)
	 - porto_parishes (polygons)

Schema schema_name and rasters are empty,  you will add tables to them as you advance with the exercises.

Table ```vectors.porto_parishes``` has all the **parishes that exist in the district of Porto, Portugal**.

Please explore the database before continuing.

----------
## Loading rasters

We will start by loading the raster files, *Landsat8_L1TP_RGBN.tif* and *srtm_1arc_v3.tif* into the rasters schema. These files are located inside the [rasters](https://github.com/lcalisto/workshop-postgis-raster/tree/master/rasters) folder in this repository.
For this operation, we'll use [raster2pgsql](http://postgis.refractions.net/docs/using_raster.xml.html#RT_Raster_Loader), please check software documentation for more information.


#### Load the elevation data

Starting with the first two examples, to begin we'll use *raster2pgsql* to create a new .sql file. Later, we can load this .sql file using *psql* or *pgAdmin*. Please replace mypath\ by the correct path to your *raster2pgsql* executable; the *srtm_1arc_v3.tif* file and the path where *raster2pgsql* will create the *dem.sql* file.

**Example 1 - Elevation**

```bash
mypath\raster2pgsql.exe -s 3763 -N -32767 -t 100x100 -I -C -M -d mypath\rasters\srtm_1arc_v3.tif rasters.dem > mypath\dem.sql
```

In example 2, we will load the data directly into the database.

**Example 2 - Elevation**
```bash 
mypath\raster2pgsql.exe -s 3763 -N -32767 -t 100x100 -I -C -M -d mypath\rasters\srtm_1arc_v3.tif rasters.dem | psql -d postgis_raster -h localhost -U postgres -p 5432
```

Now let's load landsat 8 data with a 128x128 tile size directly into the database.

**Example 3 - Landsat 8**

```bash
mypath\raster2pgsql.exe -s 3763 -N -32767 -t 128x128 -I -C -M -d mypath\rasters\Landsat8_L1TP_RGBN.TIF rasters.landsat8 | psql -d postgis_raster -h localhost -U postgres -p 5432

```

After loading the data please explore your database carefully, especially the schema *rasters*, and also take a look at the **view** *public.raster_columns*.


----------

## Create rasters from existing rasters & interact with vectors

In the first example, we'll see how to extract tiles that overlap a geometry. Optionally, you can create a table with the result of the query, let's save this result in your schema so you can view the result in QGIS. Please change *schema_name* by the schema you want to use, e.g. your name.

**Example 1 - ST_Intersects**

Intersecting a raster with a vector.

```sql
CREATE TABLE schema_name.intersects AS 
SELECT a.rast, b.municipality
FROM rasters.dem AS a, vectors.porto_parishes AS b 
WHERE ST_Intersects(a.rast, b.geom) AND b.municipality ilike 'porto';
```

The flowing three steps of code are recomended when creating new raster tables:

A - add serial primary key:

```sql
alter table schema_name.intersects
add column rid SERIAL PRIMARY KEY;
```
B - create spatial index:

```sql
CREATE INDEX idx_intersects_rast_gist ON schema_name.intersects
USING gist (ST_ConvexHull(rast));
```
C - and at last, we can add the raster constraints:
```sql
-- schema::name table_name::name raster_column::name
SELECT AddRasterConstraints('schema_name'::name, 'intersects'::name,'rast'::name);
```
**Example 2 - ST_Clip**

Clipping a raster based on a vector.

```sql
CREATE TABLE schema_name.clip AS 
SELECT ST_Clip(a.rast, b.geom, true), b.municipality 
FROM rasters.dem AS a, vectors.porto_parishes AS b 
WHERE ST_Intersects(a.rast, b.geom) AND b.municipality like 'PORTO';
```
**Example 3 - ST_Union**

Union of multiple tiles into one single raster.

```sql
CREATE TABLE schema_name.union AS 
SELECT ST_Union(ST_Clip(a.rast, b.geom, true))
FROM rasters.dem AS a, vectors.porto_parishes AS b 
WHERE b.municipality ilike 'porto' and ST_Intersects(b.geom,a.rast);
```

In addition to the example above, st_union also allows us operations on overlapping rasters based on a given aggregate function, namely FIRST LAST SUM COUNT MEAN or RANGE. For example, if we have multiple precipitation rasters and we need an average value, we can use st_union or map_algebra. For more information about st_union please check the documentation: [https://postgis.net/docs/RT_ST_Union.html](https://postgis.net/docs/RT_ST_Union.html)


----------
## Create rasters from vectors (rasterize)

In the next examples, we will rasterize one vector and learn some important PostGIS raster functions.

**Example 1 - ST_AsRaster**

First, let us use *ST_AsRaster* to rasterize table parishes from the city of Porto with the same spatial characteristics: pixel size, extents etc. 

```sql
CREATE TABLE schema_name.porto_parishes AS
WITH r AS (
	SELECT rast FROM rasters.dem 
	LIMIT 1
)
SELECT ST_AsRaster(a.geom,r.rast,'8BUI',a.id,-32767) AS rast
FROM vectors.porto_parishes AS a, r
WHERE a.municipality ilike 'porto';
```
In this example, we use pixeltype '8BUI' makring an 8-bit unsigned integer. Unsigned integers are capable of representing only non-negative integers; signed integers are capable of representing negative integers as well. For more information about PostGIS raster types, check the documentation: [https://postgis.net/docs/RT_ST_BandPixelType.html](https://postgis.net/docs/RT_ST_BandPixelType.html)


**Example 2 - ST_Union**

In the previous example, the resulting raster is one parish per record, per table row. Please visualize the previous result.

Now, in this example, we will use *ST_Union* to union all the records (table rows) of the previous example into one single raster.

```sql
DROP TABLE schema_name.porto_parishes; --> drop table porto_parishes first
CREATE TABLE schema_name.porto_parishes AS
WITH r AS (
	SELECT rast FROM rasters.dem 
	LIMIT 1
)
SELECT st_union(ST_AsRaster(a.geom,r.rast,'8BUI',a.id,-32767)) AS rast
FROM vectors.porto_parishes AS a, r
WHERE a.municipality ilike 'porto';
```
**Example 3 - ST_Tile**

After having obtained a single raster, we can generate tiles with the *ST_Tile* function.

```sql
DROP TABLE schema_name.porto_parishes; --> drop table porto_parishes first
CREATE TABLE schema_name.porto_parishes AS
WITH r AS (
	SELECT rast FROM rasters.dem 
	LIMIT 1
)
SELECT st_tile(st_union(ST_AsRaster(a.geom,r.rast,'8BUI',a.id,-32767)),128,128,true,-32767) AS rast
FROM vectors.porto_parishes AS a, r
WHERE a.municipality ilike 'porto';
```


----------

## Convert rasters into vectors (vectorize)

Now we will use ST_Intersection and ST_DumpAsPolygons to convert from rasters to vectors. PostGIS has more available functions for this, please check [https://postgis.net/docs/RT_reference.html#Raster_Processing_Geometry](https://postgis.net/docs/RT_reference.html#Raster_Processing_Geometry) for more information.

**Example 1 - ST_Intersection**

St_Intersection is similar to ST_Clip. In ST_Clip the function returns a raster while in ST_Intersection the function returns a set of geometry-pixel value pairs, as this function converts the raster into a vector before the actual "clip."  ST_Intersection is usually slower than ST_Clip, it may be wise to perform an ST_Clip in the raster before executing ST_Intersection.

```sql
create table schema_name.intersection as 
SELECT a.rid,(ST_Intersection(b.geom,a.rast)).geom,(ST_Intersection(b.geom,a.rast)).val
FROM rasters.landsat8 AS a, vectors.porto_parishes AS b 
WHERE b.parish ilike 'paranhos' and ST_Intersects(b.geom,a.rast);
```

**Example 2 - ST_DumpAsPolygons**

*ST_DumpAsPolygons* converts a raster into a vector (polygons).

```sql
CREATE TABLE schema_name.dumppolygons AS
SELECT a.rid,(ST_DumpAsPolygons(ST_Clip(a.rast,b.geom))).geom,(ST_DumpAsPolygons(ST_Clip(a.rast,b.geom))).val
FROM rasters.landsat8 AS a, vectors.porto_parishes AS b 
WHERE b.parish ilike 'paranhos' and ST_Intersects(b.geom,a.rast);
```

Both functions return a set of geomvals, for more information about the geomval data type check the documentation: [https://postgis.net/docs/geomval.html](https://postgis.net/docs/geomval.html)


----------
## Analyzing rasters 

**Example 1 - ST_Band**

To extract one band from a raster we can use the *ST_Band* function.

```sql
CREATE TABLE schema_name.landsat_nir AS
SELECT rid, ST_Band(rast,4) AS rast
FROM rasters.landsat8;
```

**Example 2 - ST_Clip**

Let's now clip one "parish" from the *vectors.porto_parishes* table. This will help in the next examples.

```sql
CREATE TABLE schema_name.paranhos_dem AS
SELECT a.rid,ST_Clip(a.rast, b.geom,true) as rast
FROM rasters.dem AS a, vectors.porto_parishes AS b
WHERE b.parish ilike 'paranhos' and ST_Intersects(b.geom,a.rast);
```
**Example 3 - ST_Slope**

Now we will generate slope using the previous generated table (elevation).

```sql
CREATE TABLE schema_name.paranhos_slope AS
SELECT a.rid,ST_Slope(a.rast,1,'32BF','PERCENTAGE') as rast
FROM schema_name.paranhos_dem AS a;

```

**Example 4 - ST_Reclass**

In order to reclass a raster we can use *ST_Reclass* function.

```sql
CREATE TABLE schema_name.paranhos_slope_reclass AS
SELECT a.rid,ST_Reclass(a.rast,1,']0-15]:1, ]16-30]:2, ]31-9999:3', '32BF',0)
FROM schema_name.paranhos_slope AS a;
```

**Example 5 - ST_SummaryStats**

In order to compute statistics from our raster we can use *ST_SummaryStats* function. In this example we will generate **statistics per tile**.

```sql
SELECT st_summarystats(a.rast) AS stats
FROM schema_name.paranhos_dem AS a;
```
**Example 6 - ST_SummaryStats with Union**

Using union we can generate one statistics of the selected raster.

```sql
SELECT st_summarystats(ST_Union(a.rast))
FROM schema_name.paranhos_dem AS a;
```
As we can see, ST_SummaryStats returns a composite data type. For more information about composite data types, please check the documentation:
[https://www.postgresql.org/docs/current/static/rowtypes.html](https://www.postgresql.org/docs/current/static/rowtypes.html)

**Example 7 - ST_SummaryStats with better control over composite type**

```sql
WITH t AS (
	SELECT st_summarystats(ST_Union(a.rast)) AS stats
	FROM schema_name.paranhos_dem AS a
)
SELECT (stats).min,(stats).max,(stats).mean FROM t;
```

**Example 8 - ST_SummaryStats with GROUP BY**

In order to have the statistics by polygon "parish" we can perform a GROUP BY.

```sql
WITH t AS (
	SELECT b.parish AS parish, st_summarystats(ST_Union(ST_Clip(a.rast, b.geom,true))) AS stats
	FROM rasters.dem AS a, vectors.porto_parishes AS b
	WHERE b.municipality ilike 'porto' and ST_Intersects(b.geom,a.rast)
	group by b.parish
)
SELECT parish,(stats).min,(stats).max,(stats).mean FROM t;
```

**Example 9 - ST_Value**

*ST_Value* function allows us to extract a pixel value from a point or a set of points. In this example, we extract the elevation of the points that are in the *vectors.places* table. Because the points geometry is multipoint and *ST_Value* function requires single point geometry, we convert from multi point into single point geometry using *(ST_Dump(b.geom)).geom*.

```sql
SELECT b.name,st_value(a.rast,(ST_Dump(b.geom)).geom)
FROM 
rasters.dem a, vectors.places AS b
WHERE ST_Intersects(a.rast,b.geom)
ORDER BY b.name;
```

### Info about Topographic Position Index (TPI)

TPI compares the elevation of each cell in a DEM to the mean elevation of a specified neighborhood around that cell. Positive values represent locations that are higher than the average of their surroundings, as defined by the neighborhood (ridges). Negative values represent locations that are lower than their surroundings (valleys). TPI values near zero are either flat areas (where the slope is near zero) or areas of constant slope.
More information about TPI can be found [here.](http://www.jennessent.com/downloads/tpi-poster-tnc_18x22.pdf)


**Example 10 - ST_TPI**

*ST_Value* function allows us to create a TPI map from an elevation DEM. Current PostGIS version can calculate TPI of one pixel using a neighborhood around that cell of one cell only. 
In this example, we are going to compute the TPI using *rasters.dem* table as input. Because this table as a resolution of 30 meters, and because this TPI only uses one neighborhood cell for the computations, we call it TPI30. We can create a table with the result of the query in *schema_name* schema so you can view the result in QGIS.

```sql
create table schema_name.tpi30 as
select ST_TPI(a.rast,1) as rast
from rasters.dem a;
```
After the table creation, we will create a spatial index
```sql
CREATE INDEX idx_tpi30_rast_gist ON schema_name.tpi30
USING gist (ST_ConvexHull(rast));
```
Add constraints
```sql
SELECT AddRasterConstraints('schema_name'::name, 'tpi30'::name,'rast'::name);
```


### First challenge

Time to put together what you have learned so far to solve a spatial problem. 

The previous query can take more than one minute to process. As you can imagine some queries can take too long to process. In order to improve processing time sometimes it's possible to restrict our area of interest and compute a smaller region. Adapt the query from *example 10* in order to process only the municipality of Porto. You need to use *ST_Intersects*, check *Example 1 - ST_Intersects* for reference. Compare the diferent processing times.

In the end check the result in QGIS.

----------

## Map Algebra

There are two ways to use Map algebra in PostGIS. One is to use an expression and the other is to use a callback function. In the following examples, we will create NDVI values based on the Landsat8 image, using both techniques.

The NDVI formula is well-known:

NDVI=(NIR-Red)/(NIR+Red)


**Example 1 - The Map Algebra Expression**

```sql
CREATE TABLE schema_name.porto_ndvi AS 
WITH r AS (
	SELECT a.rid,ST_Clip(a.rast, b.geom,true) AS rast
	FROM rasters.landsat8 AS a, vectors.porto_parishes AS b
	WHERE b.municipality ilike 'porto' and ST_Intersects(b.geom,a.rast)
)
SELECT
	r.rid,ST_MapAlgebra(
		r.rast, 1,
		r.rast, 4,
		'([rast2.val] - [rast1.val]) / ([rast2.val] + [rast1.val])::float','32BF'
	) AS rast
FROM r;
```
After the table creation, we will create a spatial index
```sql
CREATE INDEX idx_porto_ndvi_rast_gist ON schema_name.porto_ndvi
USING gist (ST_ConvexHull(rast));
```
Add constraints
```sql
SELECT AddRasterConstraints('schema_name'::name, 'porto_ndvi'::name,'rast'::name);
```

**Example 2 - The callback Function**

Using a callback function first we create the callback function. In this function, is where the magic happens!!

```sql
create or replace function schema_name.ndvi(
	value double precision [] [] [], 
	pos integer [][],
	VARIADIC userargs text []
)
RETURNS double precision AS
$$
BEGIN
	--RAISE NOTICE 'Pixel Value: %', value [1][1][1];-->For debug purposes
	RETURN (value [2][1][1] - value [1][1][1])/(value [2][1][1]+value [1][1][1]); --> NDVI calculation!
END;
$$
LANGUAGE 'plpgsql' IMMUTABLE COST 1000;
```

Now comes the map algebra query:
```sql
CREATE TABLE schema_name.porto_ndvi2 AS 
WITH r AS (
	SELECT a.rid,ST_Clip(a.rast, b.geom,true) AS rast
	FROM rasters.landsat8 AS a, vectors.porto_parishes AS b
	WHERE b.municipality ilike 'porto' and ST_Intersects(b.geom,a.rast)
)
SELECT
	r.rid,ST_MapAlgebra(
		r.rast, ARRAY[1,4],
		'schema_name.ndvi(double precision[], integer[],text[])'::regprocedure, --> This is the function!
		'32BF'::text
	) AS rast
FROM r;
```

Let's add the spatial index also ...
```sql
CREATE INDEX idx_porto_ndvi2_rast_gist ON schema_name.porto_ndvi2
USING gist (ST_ConvexHull(rast));
```
... and the raster constraints.
```sql
SELECT AddRasterConstraints('schema_name'::name, 'porto_ndvi2'::name,'rast'::name);
```


**Example 3 - The TPI functions**

Current implemented TPI function inside PostGIS uses map algebra with a callback function. We can analyse this functions to better understand map algebra.
In *public* schema functions there are two functions for TPI:

1. *public._st_tpi4ma* - The callback function used in map algebra.

2. *public.st_tpi* - The TPI function that calls the previous function. Please note that there are two *st_tpi* functions, they diverge in the number of inputs they allow. In the end both functions perform the same action.

Take some time to analyse these functions and how they perform.


**For more information about PostGIS map algebra, please check the documentation:**

MapAlgebra with expression:
[https://postgis.net/docs/RT_ST_MapAlgebra_expr.html](https://postgis.net/docs/RT_ST_MapAlgebra_expr.html)

MapAlgebra with callback function:
[https://postgis.net/docs/RT_ST_MapAlgebra.html](https://postgis.net/docs/RT_ST_MapAlgebra.html)

**New custom implementation of TPI**

Current PostGIS implementation of TPI using *ST_TPI*, only allows to compute TPI with one neighborhood cell. A new implementation of TPI allowing the user to specify neighborhood cells (inner annnulus and outter annulus), using map algebra, can be found here: [https://github.com/lcalisto/postgis_customTPI](https://github.com/lcalisto/postgis_customTPI) 


----------
## Export data

In the next examples, we will export our rasters. Postgis can save rasters in diferent file formats. In the following examples, we use ST_AsTiff and ST_AsGDALRaster, but also we will use pure Gdal functionality. 

**Example 0 - Using QGIS**

If you load the raster table/view in QGIS then you will be able to save/export the raster layer into any GDAL supported format using QGIS interface.

**Example 1 - ST_AsTiff**

ST_AsTiff function **does not save the output into the disk***, instead outputs is the binary representation of the tiff, this can be very usefull for webplatforms; scripts etc. were the developer can control what to do with the binary, for example save it into the disk or just display it.

```sql
SELECT ST_AsTiff(ST_Union(rast))
FROM schema_name.porto_ndvi;
```

 
**Example 2 - ST_AsGDALRaster**

Like ST_AsTiff function ST_AsGDALRaster **does not save the output into the disk***, instead outputs is the binary representation of the any GDAL format.

```sql
SELECT ST_AsGDALRaster(ST_Union(rast), 'GTiff',  ARRAY['COMPRESS=DEFLATE', 'PREDICTOR=2', 'PZLEVEL=9'])
FROM schema_name.porto_ndvi;
```
**Note:**

ST_AsGDALRaster functions allow us to save the raster into any gdal supported format. You can use:
```sql
 SELECT ST_GDALDrivers();
``` 
 to get a list of formats supported by your library.

**Example 3 - Saving data into disk using *large object* (lo)**

```sql
CREATE TABLE tmp_out AS
SELECT lo_from_bytea(0,
       ST_AsGDALRaster(ST_Union(rast), 'GTiff',  ARRAY['COMPRESS=DEFLATE', 'PREDICTOR=2', 'PZLEVEL=9'])
        ) AS loid
FROM schema_name.porto_ndvi;
----------------------------------------------
SELECT lo_export(loid, 'G:\myraster.tiff') --> Save the file in a place were the user postgres have access. In windows a flash drive usualy works fine.
   FROM tmp_out;
----------------------------------------------
SELECT lo_unlink(loid)
  FROM tmp_out; --> Delete the large object.
```

For more information on exporting rasters using PostGIS, check the documentation:
[https://postgis.net/docs/RT_reference.html#Raster_Outputs](https://postgis.net/docs/RT_reference.html#Raster_Outputs)

**Example 4 - Using Gdal**

Gdal has support for reading PostGIS rasters. You can use gdal_translate to export the raster into any GDAL supported format. **If your raster as tiles you should use *mode=2* option.**

```bash
gdal_translate -co COMPRESS=DEFLATE -co PREDICTOR=2 -co ZLEVEL=9 PG:"host=localhost port=5432 dbname=postgis_raster user=postgres password=postgis schema=schema_name table=porto_ndvi mode=2" porto_ndvi.tiff
```

## Publish data using MapServer

Since GDAL supports PostGIS rasters, it is possible to publish a raster as a WMS. Please note that in this case it is recommended to generate overviews for better performance.

The following example is a mapfile with a raster using standard options and a where clause.

**Example 1 - Mapfile example**
```
MAP
	NAME 'map'
	SIZE 800 650
	STATUS ON
	EXTENT -58968 145487 30916 206234
	UNITS METERS

	WEB
		METADATA
			'wms_title' 'Terrain wms'
			'wms_srs' 'EPSG:3763 EPSG:4326 EPSG:3857'
			'wms_enable_request' '*'
			'wms_onlineresource' 'http://54.37.13.53/mapservices/srtm'
		END
	END

	PROJECTION
		'init=epsg:3763'
	END

	LAYER
		NAME srtm
		TYPE raster
		STATUS OFF
		DATA "PG:host=localhost port=5432 dbname='postgis_raster' user='sasig' password='postgis' schema='rasters' table='dem' mode='2'"
		PROCESSING "SCALE=AUTO"
		PROCESSING "NODATA=-32767"
		OFFSITE 0 0 0
		METADATA
			'wms_title' 'srtm'
		END
	END
END
```

 You can access this WMS using the following URL: 

[https://sigap.calisto.pt/mapservices/srtm](https://sigap.calisto.pt/mapservices/srtm)

or using the browser with:

[https://sigap.calisto.pt/mapservices/srtm?layer=srtm&mode=map](https://sigap.calisto.pt/mapservices/srtm?layer=srtm&mode=map)

## Publish data using GeoServer

Follow our [guide to publish the raster using GeoServer](doc/geoserver.md).

----------

## First challenge solution

```sql
create table schema_name.tpi30_porto as
SELECT ST_TPI(a.rast,1) as rast
FROM rasters.dem AS a, vectors.porto_parishes AS b 
WHERE ST_Intersects(a.rast, b.geom) AND b.municipality ilike 'porto'
```
Let's add the spatial index ...
```sql
CREATE INDEX idx_tpi30_porto_rast_gist ON schema_name.tpi30_porto
USING gist (ST_ConvexHull(rast));
```
... and the raster constraints.
```sql
SELECT AddRasterConstraints('schema_name'::name, 'tpi30_porto'::name,'rast'::name);
```
