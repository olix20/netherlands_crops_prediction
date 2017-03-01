Analyzing and predicting crop data in the Netherlands 





I found QGIS extremely handy in exploring datasets, debugging and inspecting errors. 



### Requirements

Postgres + PostGIS
gdal: brew install gdal --enable-unsupported --with-postgresql



### Data sources 

 Loading basisregistratie-gewaspercelen-brp from an ESRI file geodatabase (.gdb) using R

refs:
https://data.overheid.nl/data/dataset/basisregistratie-gewaspercelen-brp
https://geodata.nationaalgeoregister.nl/brpgewaspercelen/atom/brpgewaspercelen.xml
http://gis.stackexchange.com/questions/151613/how-to-read-feature-class-in-file-geodatabase-using-r

Dataset download links can be found in: http://geodata.nationaalgeoregister.nl/brpgewaspercelen/atom/brpgewaspercelen.xml

### importing data into PostGres 

SRID 28992


// Converting GDB files to Shape files 

	ogr2ogr -f "ESRI Shapefile” destination_data.shp BRP_Gewaspercelen_2009.gdb 


// Importing shape files to postgres


	shp2pgsql -I -s 28992 2012.shp 2012 | psql -p 5433 -U postgres -d crops
	shp2pgsql -I <PATH/TO/SHAPEFILE> <SCHEMA>.<DBTABLE> | psql -U postgres -d <DBNAME>







### Preparing Database 


// fix geometries 
	alter table "2014" add column geom_fixed geometry; 
	update "2014" set geom_fixed = ST_makeValid(geom); 
	CREATE INDEX geom_fixed_2014_gix ON "2014" USING GIST (geom_fixed); 


// add centeroid of geoms 
	alter table "2016" add column center geometry; 
	update "2016" set center= ST_SetSRID(ST_Centroid(geom_fixed),28992);
	CREATE INDEX centroid_2016_gix ON "2016" USING GIST (center); 



//create foreign key indexes on each table 

alter table "2015" add column next_year_parcel_fk integer REFERENCES "2016" (gid);
update "2015" set next_year_parcel_fk="2016".gid from "2016"
where 
ST_CoveredBy("2016".center,"2015".geom_fixed) 
-- and (st_area(st_intersection("2015".geom_fixed,"2016".geom_fixed))/("2016".shape_area)) > 0.6


A more accurate alternative would probably be: 

update "2015" 
set next_year_parcel_fk="2016".gid 

from "2016" where 
ST_Intersects("2016".geom_fixed,"2015".geom_fixed) and 
(st_area(st_intersection("2015".geom_fixed,"2016".geom_fixed))/("2016".shape_area)) >0.8






// creating the fishnet 

	get bounding box of the 2009:
	xmin ymin:
	12856.7 305608.34, 
	xmax, ymax 
	277980.31 613446.06

	— 
	create fish nets 
	CREATE TABLE fishnet (gid serial, cell geometry);
	insert into fishnet (cell)

	(SELECT cell FROM 
	(SELECT (
	ST_Dump(makegrid_2d(ST_GeomFromText('Polygon((12856.7 305608.34, 12856.7 613446.06, 277980.31 613446.06, 277980.31 305608.34, 12856.7 305608.34))',
	 28992), -- WGS84 SRID
	 10000) -- cell step in meters
	)).geom AS cell) AS q_grid
	)


based on bounding box :

	get bounding box of the 2009:
	xmin ymin:
	12856.7 305608.34, 
	xmax, ymax 
	277980.31 613446.06



// assign fishnet id to each parcel in each year 

alter table "2009" add column cell_id integer ; 
update "2009" set cell_id = fishnet.gid from fishnet 
where ST_CoveredBy("2009".center,fishnet.cell) 

// importing soil data 

https://snorfalorpagus.net/blog/2016/03/13/splitting-large-polygons-for-faster-intersections/



//  add soil types to reference table 

	alter table "2016" add column soil character varying ; 
	alter table "2016" add column soil_code smallint ;  

	update "2016" 
	set soil = soil_map_dump.omschrijvi,  soil_code=soil_map_dump.gronds
	from soil_map_dump where ST_CoveredBy("2016".center,soil_map_dump.geom) 



### Import db dump into PostGres

gunzip -c crops_pgdump_march1.gz | psql your_db_name




### Indexing 

	vacuum analyze  "2010"

fix polygons
check if geometries are valid:
select gid  from soil_map where ST_isvalid(soil_map.geom)=false

	update "soil_map" set geom_fixed = ST_MakeValid(geom)

