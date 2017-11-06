

# **Workshop PostGIS raster**

This is a short workshop to explain PostGIS raster.
---------------------------------------------------


----------

## Create and restore the database
 
 In order to start the workshop we will load and existing database. 

Using pgAdmin please create a new, empty database and then restore the existing dump ***postgis_raster.backup*** from this repository into the new recently created database.


----------
## Database structure

The restored database consists in the folowing structure:

 - schema_name *(you should rename it with your name)*
 - public
 - rasters
 - vectors
	 - ferrovia (lines)
	 - lugares (points)
	 - porto_freguesias (polygons)

Schema schema_name and rasters are empty,  you will add tables to them as you advance with the exercises.


Please explore the database before continue.

----------
## Loading rasters

We will start by loading the raster files, *Landsat8_L1TP_RGBN.tif* and *srtm_1arc_v3.tif* into the rasters schema. 
We will use software [raster2pgsql](http://postgis.refractions.net/docs/using_raster.xml.html#RT_Raster_Loader) , please check software documentation for more information.


#### Load the elevation data

We will use two examples, first we will use *raster2pgsql* to create a new .sql file. Later we can load this .sql file using *psql* or *pgAdmin*. Please replace mypath\ by the correct path to your *raster2pgsql* executable; the *srtm_1arc_v3.tif* file and the path were *raster2pgsql* will create the *dem.sql* file.

**Example 1 - Elevation**

```bash
mypath\raster2pgsql.exe -s 3763 -N -32767 -t 100x100 -I -C –M -d mypath\rasters\srtm_1arc_v3.tif rasters.dem > mypath\dem.sql
```

In example 2 we will load the data directly into the database.

**Example 2 - Elevation**
```bash 
mypath\raster2pgsql.exe -s 3763 -N -32767 -t 100x100 -I -C –M -d mypath\rasters\srtm_1arc_v3.tif rasters.dem | psql –d postgis_raster –h localhost –U postgres –p 5432
```

Now lets load landsat 8 data with a 128x128 tile size directly into the database.

**Example 3 - Landsat 8**

```bash
mypath\raster2pgsql.exe -s 3763 -N -32767 -t 128x128 -I -C –M -d mypath\rasters\Landsat8_L1TP_RGBN.TIF rasters.landsat8 | psql –d postgis_raster –h localhost –U postgres –p 5432

```

After loading the data please explore your database carefully, especially the schema *rasters*, also have a look at the **view** *public.raster_columns*.


----------

## Create rasters from other rasters

In the first example we'll see how to extract tiles that overlap a geometry. Optionally you can create a table with the result of the query, lets save this result in the *schema_name* schema so you can see the result in QGIS.

**Example 1 - ST_Intersects**

```sql
CREATE TABLE schema_name.intersects AS 
SELECT a.rast, b.concelho
FROM rasters.dem AS a, vectors.porto_freguesias AS b 
WHERE ST_Intersects(a.rast, b.geom) AND b.concelho ilike 'porto';
```

The flowing 3 steps of code are for when we create new raster tables:

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
C - and for last we can add the raster constraints:
```sql
-- schema::name table_name::name raster_column::name
SELECT AddRasterConstraints('schema_name'::name, 'intersects'::name,'rast'::name);
```
**Example 2 - ST_Clip**

```sql
CREATE TABLE schema_name.clip AS 
SELECT ST_Clip(a.rast, b.geom, true), b.concelho 
FROM rasters.dem AS a, vectors.porto_freguesias AS b 
WHERE ST_Intersects(a.rast, b.geom) AND b.concelho like 'PORTO';
```
**Example 3 - ST_Union**

```sql
CREATE TABLE schema_name.union AS 
SELECT ST_Union(ST_Clip(a.rast, b.geom, true))
FROM rasters.dem AS a, vectors.porto_freguesias AS b 
WHERE b.concelho ilike 'porto' and ST_Intersects(b.geom,a.rast);
```

In addition to the example above, St_Union also allow us operations in overlaping rasters based in a given predicate, namely FIRST LAST SUM COUNT MEAN and RANGE. For example if we have multiple precipitation rasters and we need an average of them, we can use  st_union or map_algebra. For more information about st_union please check the documentation: [https://postgis.net/docs/RT_ST_Union.html](https://postgis.net/docs/RT_ST_Union.html)


----------
## Create rasters from vectors

In the next examples we will rasterize one vector and learn some important PostGIS raster functions.

**Example 1 - ST_AsRaster**

First lets use *ST_AsRaster* to rasterize table freguesias from Porto with the same spatial characteristics: pixel size, extents etc. 

```sql
CREATE TABLE schema_name.porto_freguesias AS
WITH r AS (
	SELECT rast FROM rasters.dem 
	LIMIT 1
)
SELECT ST_AsRaster(a.geom,r.rast,'8BUI',a.id,-32767) AS rast
FROM vectors.porto_freguesias AS a, r
WHERE a.concelho ilike 'porto';
```
In this example we used pixeltype '8BUI' 8-bit unsigned integer. Unsigned integers are capable of representing only non-negative integers, signed integers are capable of representing negative integers as well. For more information about PostGIS raster types check the documentation: [https://postgis.net/docs/RT_ST_BandPixelType.html](https://postgis.net/docs/RT_ST_BandPixelType.html)


**Example 2 - ST_Union**

In this example we will use ST_Union to union all the rasterized polygons into one single raster.

```sql
DROP TABLE schema_name.porto_freguesias;
CREATE TABLE schema_name.porto_freguesias AS
WITH r AS (
	SELECT rast FROM rasters.dem 
	LIMIT 1
)
SELECT st_union(ST_AsRaster(a.geom,r.rast,'8BUI',a.id,-32767)) AS rast
FROM vectors.porto_freguesias AS a, r
WHERE a.concelho ilike 'porto';
```
**Example 3 - ST_Tile**

After having one single raster we can generate tiles with *ST_Tile* function.

```sql
DROP TABLE schema_name.porto_freguesias; --> drop table porto_freguesias first
CREATE TABLE schema_name.porto_freguesias AS
WITH r AS (
	SELECT rast FROM rasters.dem 
	LIMIT 1
)
SELECT st_tile(st_union(ST_AsRaster(a.geom,r.rast,'8BUI',a.id,-32767)),128,128,true,-32767) AS rast
FROM vectors.porto_freguesias AS a, r
WHERE a.concelho ilike 'porto';
```


----------

## Convert rasters into vectors

Now we will use ST_Intersection and ST_DumpAsPolygons to convert from rasters to vectors. PostGIS as more available functions for this, please check [https://postgis.net/docs/RT_reference.html#Raster_Processing_Geometry](https://postgis.net/docs/RT_reference.html#Raster_Processing_Geometry) for more information.

**Example 1 - ST_Intersection**

St_Intersection is similar to ST_Clip, in ST_Clip the function returns a raster while in ST_Intersection the function returns a set of geometry-pixelvalue pairs, this function converts the raster into a vector before the actual "clip".  ST_Intersection is usually slower that ST_Clip, it may be wise to perform a ST_Clip in the raster before the ST_Intersection.

```sql
create table schema_name.intersection as 
SELECT a.rid,(ST_Intersection(b.geom,a.rast)).geom,(ST_Intersection(b.geom,a.rast)).val
FROM rasters.landsat8 AS a, vectors.porto_freguesias AS b 
WHERE b.freguesia ilike 'paranhos' and ST_Intersects(b.geom,a.rast);
```

**Example 2 - ST_DumpAsPolygons**

```sql
CREATE TABLE schema_name.dumppolygons AS
SELECT a.rid,(ST_DumpAsPolygons(ST_Clip(a.rast,b.geom))).geom,(ST_DumpAsPolygons(ST_Clip(a.rast,b.geom))).val
FROM rasters.landsat8 AS a, vectors.porto_freguesias AS b 
WHERE b.freguesia ilike 'paranhos' and ST_Intersects(b.geom,a.rast);
```

Both functions return a set of geomvals, form more information about geomval datatype check the documentation: [https://postgis.net/docs/geomval.html](https://postgis.net/docs/geomval.html)


----------
## Analyzing rasters 

**Example 1 - ST_Band**

In order to extract one band from a raster we can use ST_Band function.

```sql
CREATE TABLE schema_name.landsat_nir AS
SELECT rid, ST_Band(rast,4) AS rast
FROM rasters.landsat8;
```

**Example 2 - ST_Clip**
Lets now clip one "freguesia" from *vectors.porto_freguesias* table. This will help in the next examples.

```sql
CREATE TABLE schema_name.paranhos_dem AS
SELECT a.rid,ST_Clip(a.rast, b.geom,true) as rast
FROM rasters.dem AS a, vectors.porto_freguesias AS b
WHERE b.freguesia ilike 'paranhos' and ST_Intersects(b.geom,a.rast);
```
**Example 3 - ST_Slope**

```sql
CREATE TABLE schema_name.paranhos_slope AS
SELECT a.rid,ST_Slope(a.rast,1,'32BF','PERCENTAGE') as rast
FROM schema_name.paranhos_dem AS a;

```

**Example 4 - ST_Reclass**

```sql
CREATE TABLE schema_name.paranhos_slope_reclass AS
SELECT a.rid,ST_Reclass(a.rast,1,']0-15]:1, ]16-30]:2, ]31-9999:3', '32BF',0)
FROM schema_name.paranhos_slope AS a;
```

**Example 5 - ST_SummaryStats**

```sql
SELECT st_summarystats(a.rast) AS stats
FROM schema_name.paranhos_dem AS a;
```
**Example 6 - ST_SummaryStats with Union**

```sql
SELECT st_summarystats(ST_Union(a.rast))
FROM schema_name.paranhos_dem AS a;
```
As we can see ST_SummaryStats returns a composite datatype. For more information about composite datatypes please check the documentation:
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

In order to have the statistics by polygon "freguesia" we can perform a GROUP BY.
```sql
WITH t AS (
	SELECT b.freguesia AS freguesia, st_summarystats(ST_Union(ST_Clip(a.rast, b.geom,true))) AS stats
	FROM rasters.dem AS a, vectors.porto_freguesias AS b
	WHERE b.concelho ilike 'porto' and ST_Intersects(b.geom,a.rast)
	group by b.freguesia
)
SELECT freguesia,(stats).min,(stats).max,(stats).mean FROM t;
```

**Example 9 - ST_Value**

ST_Value allow us to extract a pixel value from a point or a set of points. In this example we will extract the elevation of the  points from *vectors.lugares* table.

```sql
SELECT b.name,st_value(a.rast,(ST_Dump(b.geom)).geom)
FROM 
rasters.dem a, vectors.lugares AS b
WHERE ST_Intersects(a.rast,b.geom)
ORDER BY b.name;
```


----------

## Map Algebra

There are two ways to use Map algebra in PostGIS. One is to use an expression and the other is to use a callback function. In the following examples we will create a NDVI based on the Landsat8 image, using both techniques.

The NDVI formula:

NDVI=(NIR-Red)/(NIR+Red)


**Example 1 - The Map Algebra Expression**

```sql
CREATE TABLE schema_name.porto_ndvi AS 
WITH r AS (
	SELECT a.rid,ST_Clip(a.rast, b.geom,true) AS rast
	FROM rasters.landsat8 AS a, vectors.porto_freguesias AS b
	WHERE b.concelho ilike 'porto' and ST_Intersects(b.geom,a.rast)
)
SELECT
	r.rid,ST_MapAlgebra(
		r.rast, 1,
		r.rast, 4,
		'([rast2.val] - [rast1.val]) / ([rast2.val] + [rast1.val])::float','32BF'
	) AS rast
FROM r;
```
After the table creation we will create spatial index
```sql
CREATE INDEX idx_porto_ndvi_rast_gist ON schema_name.porto_ndvi
USING gist (ST_ConvexHull(rast));
```
Add constraints
```sql
SELECT AddRasterConstraints('schema_name'::name, 'porto_ndvi'::name,'rast'::name);
```

**Example 2 - The callback Function**

Using a callback function first we create the callback function. In this function is were the magic happens!!

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

Now it comes the map algebra query:
```sql
CREATE TABLE schema_name.porto_ndvi2 AS 
WITH r AS (
	SELECT a.rid,ST_Clip(a.rast, b.geom,true) AS rast
	FROM rasters.landsat8 AS a, vectors.porto_freguesias AS b
	WHERE b.concelho ilike 'porto' and ST_Intersects(b.geom,a.rast)
)
SELECT
	r.rid,ST_MapAlgebra(
		r.rast, ARRAY[1,4],
		'schema_name.ndvi(double precision[], integer[],text[])'::regprocedure, --> This is the function!
		'32BF'::text
	) AS rast
FROM r;
```

Lets add the spatial index ...
```sql
CREATE INDEX idx_porto_ndvi2_rast_gist ON schema_name.porto_ndvi2
USING gist (ST_ConvexHull(rast));
```
... and the raster constrains.
```sql
SELECT AddRasterConstraints('schema_name'::name, 'porto_ndvi2'::name,'rast'::name);
```
For more information about PostGIS map algebra please check the documentation:

MapAlgebra with expression:
[https://postgis.net/docs/RT_ST_MapAlgebra_expr.html](https://postgis.net/docs/RT_ST_MapAlgebra_expr.html)

MapAlgebra with callback function:
[https://postgis.net/docs/RT_ST_MapAlgebra.html](https://postgis.net/docs/RT_ST_MapAlgebra.html)




----------
## Export data

In the next examples we will export our rasters. Postgis can save rasters into diferent file formats. In the following examples we will use ST_AsTiff and ST_AsGDALRaster, also we will use pure Gdal functionality. 

**Example 1 - ST_AsTiff**

```sql
SELECT ST_AsTiff(ST_Union(rast))
FROM schema_name.porto_ndvi;
```

 
**Example 2 - ST_AsGDALRaster**

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

For more information on exporting rasters using PostGIS check the documentation:
[https://postgis.net/docs/RT_reference.html#Raster_Outputs](https://postgis.net/docs/RT_reference.html#Raster_Outputs)

**Example 4 - Using Gdal**

Gdal as support for reading PostGIS rasters. You can use gdal_translate to export the raster into any GDAL supported format. if your raster as tiles you should use *mode=2* option.

```bash
gdal_translate -co COMPRESS=DEFLATE -co PREDICTOR=2 -co ZLEVEL=9 PG:"host=localhost port=5432 dbname=postgis_raster user=postgres password=postgis schema=schema_name table=porto_ndvi mode=2" porto_ndvi.tiff
```

## Publish data using Mapserver

Since GDAL supports PostGIS rasters, it is possible to publish a raster as a WMS. Please note that in this case it might be recommended to generate overviews for better performance.

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

[http://54.37.13.53/mapservices/srtm](http://54.37.13.53/mapservices/srtm)

or using the browser with:

[http://54.37.13.53/mapservices/srtm?layer=srtm&mode=map](http://54.37.13.53/mapservices/srtm?layer=srtm&mode=map)
