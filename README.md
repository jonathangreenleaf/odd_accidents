Grab an AWS EC2 instance, install PostgreSQL and PostGIS
I used debian-7-amd64-default 1403184978 (ami-00a15868)
Postgres 9.1 and PostGIS 2.0.6
get the TIGER data for at least MA loaded
follow:  http://postgis.net/docs/postgis_installation.html
this involves enabling extensions and just don't forget this:
SELECT install_missing_indexes();
Geocoding was MUCH slower without it :-)


Load the accidents csv into Postgres (accidents table)

####geocode the address data

```
drop table addresses_to_geocode;
 
CREATE TABLE addresses_to_geocode(addid serial PRIMARY KEY, crash_number character varying(40), address text,
        lon numeric, lat numeric, new_address text, rating integer, geom geometry);
 
INSERT INTO addresses_to_geocode(crash_number, address)
 select crash_number, street_num || ' ' || street_name || ' ' || 'Cambridge, MA' as raw
  from accidents
  where 
  street_num is not null order by crash_number;
```
 
####for i in `seq 1000`; do psql -d geocoder -f /tmp/geocode.sql; done; where geocode.sql:

```
BEGIN TRANSACTION;
UPDATE addresses_to_geocode
  SET  (rating, new_address, lon, lat, geom)
  = ( COALESCE((g.geo).rating,-1), pprint_addy((g.geo).addy),
     ST_X((g.geo).geomout)::numeric(8,5), ST_Y((g.geo).geomout)::numeric(8,5), (g.geo).geomout )
FROM (SELECT addid
     FROM addresses_to_geocode
     WHERE rating IS NULL ORDER BY addid LIMIT 3) As a
     LEFT JOIN (SELECT addid, (geocode(address,1)) As geo
    FROM addresses_to_geocode As ag
    WHERE ag.rating IS NULL ORDER BY addid LIMIT 3) As g ON a.addid = g.addid
WHERE a.addid = addresses_to_geocode.addid;
END TRANSACTION;
```

#### repeat approach on intersection data
i.e., select * from accidents where 
      street_num is null and street_name is not null 
       and cross_street is not null; = 3251

```
 CREATE TABLE intersections_to_geocode(
 addid serial PRIMARY KEY, crash_number character varying(40), street_name text,
 cross_street text,
 lon numeric, lat numeric, 
 new_address text, rating integer, geom geometry);

 INSERT INTO intersections_to_geocode(crash_number, street_name, cross_street)
 select crash_number, street_name, cross_street
  from accidents
  where 
 street_num is null and street_name is not null 
 and cross_street is not null
 order by crash_number;
```

####repeat until complete
for i in `seq 2000`; do psql -d geocoder -f /tmp/geocode_intersections.sql; done
where geocode_intersections.sql:

```
BEGIN TRANSACTION;
UPDATE intersections_to_geocode
  SET  (rating, new_address, lon, lat, geom)
  = ( COALESCE((g.geo).rating,-1), pprint_addy((g.geo).addy),
     ST_X((g.geo).geomout)::numeric(8,5), ST_Y((g.geo).geomout)::numeric(8,5), (g.geo).geomout )
FROM (SELECT addid
     FROM intersections_to_geocode
     WHERE rating IS NULL ORDER BY addid LIMIT 3) As a
     LEFT JOIN (SELECT addid, (geocode_intersection_debug(street_name, cross_street, 'MA', 'Cambridge','', 1)) As geo
    FROM intersections_to_geocode As ag
    WHERE ag.rating IS NULL ORDER BY addid LIMIT 3) As g ON a.addid = g.addid
WHERE a.addid = intersections_to_geocode.addid;
END TRANSACTION;
```

as per http://gis.stackexchange.com/questions/11567/spatial-clustering-with-postgis
install kmeans 
comes via wget http://api.pgxn.org/dist/kmeans/1.1.0/kmeans-1.1.0.zip
```
psql -f /usr/share/postgresql/9.1/extension/kmeans.sql -U postgres -d geocoder
```

approach: do the clustering - target 50
remember GeoJSON for leaflet


note: keep geoms (not geography types) as per
http://workshops.boundlessgeo.com/postgis-intro/geography.html
sticking with degrees not meters 
http://gis.stackexchange.com/questions/23820/how-do-i-get-the-distance-meter-value-between-two-geometries-in-postgis

Focus on the good data (geocoded with rating >= 3) (~90% was ok)

```
DROP TABLE IF EXISTS geoms_to_cluster;

CREATE TABLE geoms_to_cluster as
SELECT geom FROM addresses_to_geocode
WHERE 
new_address like '%, Cambridge,%' and geom is not null 
and rating in (0,1,2,3)
union all
SELECT geom FROM intersections_to_geocode
WHERE 
new_address like '%, Cambridge,%' and geom is not null
and rating in (0,1,2,3);

CREATE INDEX geoms_to_cluster_geom_gist ON geoms_to_cluster USING gist (geom);
```

Run the kmeans function - remember the size of each cluster, geom, centroid of the geom
Analysis in QGis shows the expanded cluster geoms overlayed on City of Cambridge centerline data
http://www.cambridgema.gov/gis/gisdatadictionary/trans/trans_centerlines.aspx
![Image of QGis](http://54.173.37.176/accidents/img/qgis_pic.png)

```
DROP TABLE IF EXISTS clusters;

CREATE TABLE clusters as 
SELECT kmeans, count(*) as cnt, ST_Buffer(ST_Centroid(ST_Collect(geom)),(count(*)/100000.0000),20) AS geom,
ST_Centroid(ST_Collect(geom)) AS cgeom
,ST_AsGeoJSON(ST_Buffer(ST_Centroid(ST_Collect(geom)),(count(*)/100000.0000),20)) as geojson
FROM (
  SELECT kmeans(ARRAY[ST_X(geom), ST_Y(geom)], 50) OVER (), geom
FROM geoms_to_cluster) AS ksub
GROUP BY kmeans
ORDER BY kmeans;

CREATE INDEX clusters_cgeom_gist  ON clusters USING gist (cgeom);
CREATE INDEX clusters_geom_gist  ON clusters USING gist (geom);
```

run nearest neighbor approach (cluster centroid vs centerlines with srid 4269)
following http://www.bostongis.com/TutBook.aspx#130
shoutout:  Regina Obe is a rock star

```
DROP table centerlines_srid;
create table centerlines_srid as select *, st_transform(geom, 4269) as new_geom from centerlines;
CREATE INDEX centerlines_srid_geom_gist  ON centerlines_srid USING gist (geom);
```

validate the pgis_fn_nn works as expected
select b.*, pgis_fn_nn(geom, 1000000, 10,1000,'centerlines_srid', 'true', 'gid', 'new_geom')
from clusters b where kmeans = 26;

this approach seems to make sense for finding candidate streets
on which to place signage; should be based on proximity to the cluster area,
not necessarily overlap of the cluster edge with a something; plus
this give multiple centerline geoms back, and the longest and/or most meaningful street
can be selected from this pgis_fn_nn approach after this

check:  select * from clusters - have 0-49 clusters

do the neighbors for each cluster

```
DROP TABLE IF EXISTS cluster_neighbors;

create table cluster_neighbors as
select b.*, pgis_fn_nn(cgeom, 1000000, 10,1000,'centerlines_srid', 'true', 'gid', 'new_geom')
from clusters b where kmeans = 0;

DO
$do$
BEGIN 
FOR i IN 1..49 LOOP

   INSERT INTO cluster_neighbors
   select b.*, pgis_fn_nn(cgeom, 1000000, 10,1000,'centerlines_srid', 'true', 'gid', 'new_geom')
   from clusters b where kmeans = i;
   
END LOOP;
END
$do$
```


identify gids closest to cluster (Cambridge centerlines dataset gid)
this has street name and range of address numbers usually
we have lat lon already, but nice to have meta info

```
drop table if exists cluster_centerlines;

create table cluster_centerlines as
select kmeans, cnt, 
cast(substring(cast(pgis_fn_nn as text) from 2 for position(',' in cast(pgis_fn_nn as text))-2) as integer) as gid
FROM cluster_neighbors;
```

get the logical places for signage
use 'fatten' centerline approach to create buffer against which geom overlaps 
are evaluated.  Take the intersection with the largest shared area
as representative of a 'major' intersection, near a cluster
cnt > 30 removes some outliers (focus on larger clusters)

```
drop table if exists centerline_sign_locations;

create table centerline_sign_locations as
select 
centroid(ST_Intersection(st_buffer(a.new_geom,0.00005), st_buffer(b.new_geom,0.00005))),
astext(centroid(ST_Intersection(st_buffer(a.new_geom,0.00005), st_buffer(b.new_geom,0.00005))))
FROM
centerlines_srid as a ,
centerlines_srid as b 
WHERE 
ST_Overlaps(st_buffer(a.new_geom,0.00005), st_buffer(b.new_geom,0.00005)) 
AND a.gid != b.gid
AND a.gid in (select gid from cluster_centerlines where kmeans = 0 and cnt > 30)
AND b.gid in (select gid from cluster_centerlines where kmeans = 0 and cnt > 30) 
ORDER BY ST_AREA(ST_Intersection(st_buffer(a.new_geom,0.00005), st_buffer(b.new_geom,0.00005))), a.gid DESC
LIMIT 1;

DO
$do$
BEGIN 
FOR i IN 1..49 LOOP

INSERT INTO centerline_sign_locations
select 
centroid(ST_Intersection(st_buffer(a.new_geom,0.00005), st_buffer(b.new_geom,0.00005))),
astext(centroid(ST_Intersection(st_buffer(a.new_geom,0.00005), st_buffer(b.new_geom,0.00005))))
FROM
centerlines_srid as a ,
centerlines_srid as b 
WHERE 
ST_Overlaps(st_buffer(a.new_geom,0.00005), st_buffer(b.new_geom,0.00005)) 
AND a.gid != b.gid
AND a.gid in (select gid from cluster_centerlines where kmeans = i and cnt > 30)
AND b.gid in (select gid from cluster_centerlines where kmeans = i and cnt > 30) 
ORDER BY ST_AREA(ST_Intersection(st_buffer(a.new_geom,0.00005), st_buffer(b.new_geom,0.00005))), a.gid DESC
LIMIT 1;

END LOOP;
END
$do$
```


from here out were just export for Leaflet.js

export accident GeoJSON

```
SELECT 
'{"type": "Feature","geometry":' ||
ST_AsGeoJSON(a.geom) || ',"properties":{
"popup_detail": "' || a.new_address 
|| '",
"crash_number":"' || a.crash_number 
|| '",
"timestamp":"' || b.timestamp 
|| '",
"day_of_week":"' || b.day_of_week
|| '",
"object1":"' || b.object1 
 || '",
"object2":"' || b.object2 
|| '"}},'

FROM addresses_to_geocode a left join accidents b
on a.crash_number = b.crash_number
WHERE 
a.new_address like '%, Cambridge,%' and a.geom is not null 
and a.rating in (0,1,2,3)
union all 
SELECT 
'{"type": "Feature","geometry":' ||
ST_AsGeoJSON(a.geom) || ',"properties":{
"popup_detail": "' || a.new_address 
|| '",
"crash_number":"' || a.crash_number 
|| '",
"timestamp":"' || b.timestamp 
|| '",
"day_of_week":"' || b.day_of_week
|| '",
"object1":"' || b.object1 
 || '",
"object2":"' || b.object2 
|| '"}},'

FROM intersections_to_geocode a left join accidents b
on a.crash_number = b.crash_number
WHERE 
a.new_address like '%, Cambridge,%' and a.geom is not null
and a.rating in (0,1,2,3);
```


export cluster GeoJSON

```
SELECT 
'{"type": "Feature","geometry":' ||
ST_AsGeoJSON(a.cgeom) || ',"properties":{
"count": "' || a.cnt 
|| '",
"popup_detail":cluster_id"' || a.kmeans  
|| '"}},'
FROM clusters a 
WHERE a.cnt > 30;
```

export sign locations along with the actual pointers
to signs.  This is a tad hacky, but alas I'm rushing here near
the deadline :-)

```
drop table if exists signs;

CREATE TABLE signs
(
  gid numeric,
  sign_text character varying(200),
  code character varying(2000)
)
WITH (
  OIDS=FALSE
);
ALTER TABLE signs
  OWNER TO postgres;

insert into signs (gid, sign_text)
    values 
        (2016, '.addTo(map).bindPopup("<img src=''img/3kids.png'' width=300 ><br />That Bicyclist has 3 Kids - Drive Safe'),
        (2437, '.addTo(map).bindPopup("<img src=''img/becareful.png'' width=300 ><br />Don''t Drive Angry - Be [Car]eful'),
        (1628, '.addTo(map).bindPopup("<img src=''img/bikesarepeople.png'' width=300 ><br />Bikes Are People Too'),
        (2261, '.addTo(map).bindPopup("<img src=''img/coexist.png'' width=300 ><br />Coexist With Bikes'),
        (2269, '.addTo(map).bindPopup("<img src=''img/friendly.png'' width=300 ><br />Drive Friendly'),
        (2192, '.addTo(map).bindPopup("<img src=''img/shareyourspace.png'' width=300 ><br />Share This Space'),
        (914, '.addTo(map).bindPopup("<img src=''img/hipsterhelmet.png'' width=300 ><br />That Hipster Bicyclist Needs a Helmet'),
        (2539, '.addTo(map).bindPopup("<img src=''img/bikesarepeople.png'' width=300 ><br />Bikes Are People Too'),
        (990, '.addTo(map).bindPopup("<img src=''img/doesntseeyou.png'' width=300 ><br />That Biker Doesn''t See You'),
        (1326, '.addTo(map).bindPopup("<img src=''img/makenice.png'' width=300 ><br />Make Nice With Bikes'),
        (924, '.addTo(map).bindPopup("<img src=''img/bikeslow.png'' width=300 ><br />Bike Slow & Boring'),
        (571, '.addTo(map).bindPopup("<img src=''img/nohelmet.png'' width=300 ><br />No Helmet, No Brains'),
        (1827, '.addTo(map).bindPopup("<img src=''img/nohelmet.png'' width=300 ><br />No Helmet, No Brains'),
        (483, '.addTo(map).bindPopup("<img src=''img/traffic.png'' width=300 ><br />That Bicyclist isn''t Causing This Traffic'),
        (1501, '.addTo(map).bindPopup("<img src=''img/swerve.png'' width=300 ><br />Anticipate Bike Swerve'),
        (2337, '.addTo(map).bindPopup("<img src=''img/bikeslow.png'' width=300 ><br />Bike Slow & Boring'),
        (2610, '.addTo(map).bindPopup("<img src=''img/swerve.png'' width=300 ><br />Anticipate Bike Swerve'),
        (201, '.addTo(map).bindPopup("<img src=''img/coexist.png'' width=300 ><br />Coexist With Bikes'),
        (1790, '.addTo(map).bindPopup("<img src=''img/nobumpers.png'' width=300 ><br />Bikes Have No Bumpers'),
        (715, '.addTo(map).bindPopup("<img src=''img/friendly.png'' width=300 ><br />Drive Friendly'),
        (698, '.addTo(map).bindPopup("<img src=''img/hipsterhelmet.png'' width=300 ><br />That Hipster Bicyclist Needs a Helmet'),
        (909, '.addTo(map).bindPopup("<img src=''img/traffic.png'' width=300 ><br />That Bicyclist isn''t Causing This Traffic'),
        (2510, '.addTo(map).bindPopup("<img src=''img/bikesarepeople.png'' width=300 ><br />Bikes Are People Too'),
        (604, '.addTo(map).bindPopup("<img src=''img/becareful.png'' width=300 ><br />Don''t Drive Angry - Be [Car]eful'),
        (2291, '.addTo(map).bindPopup("<img src=''img/doesntseeyou.png'' width=300 ><br />That Biker Doesn''t See You'),
        (20, '.addTo(map).bindPopup("<img src=''img/traffic.png'' width=300 ><br />That Bicyclist isn''t Causing This Traffic'),
        (662, '.addTo(map).bindPopup("<img src=''img/nohelmet.png'' width=300 ><br />No Helmet, No Brains'),
        (41, '.addTo(map).bindPopup("<img src=''img/becareful.png'' width=300 ><br />Don''t Drive Angry - Be [Car]eful'),
        (795, '.addTo(map).bindPopup("<img src=''img/coexist.png'' width=300 ><br />Coexist With Bikes'),
        (1345, '.addTo(map).bindPopup("<img src=''img/becareful.png'' width=300 ><br />Don''t Drive Angry - Be [Car]eful'),
        (768, '.addTo(map).bindPopup("<img src=''img/becareful.png'' width=300 ><br />Don''t Drive Angry - Be [Car]eful'),
        (634, '.addTo(map).bindPopup("<img src=''img/nobumpers.png'' width=300 ><br />Bikes Have No Bumpers'),
        (83, '.addTo(map).bindPopup("<img src=''img/shareyourspace.png'' width=300 ><br />Share This Space'),
        (903, '.addTo(map).bindPopup("<img src=''img/traffic.png'' width=300 ><br />That Bicyclist isn''t Causing This Traffic'),
        (1379, '.addTo(map).bindPopup("<img src=''img/swerve.png'' width=300 ><br />Anticipate Bike Swerve'),
        (1370, '.addTo(map).bindPopup("<img src=''img/makenice.png'' width=300 ><br />Make Nice With Bikes'),
        (2008, '.addTo(map).bindPopup("<img src=''img/nobumpers.png'' width=300 ><br />Bikes Have No Bumpers'),
        (311, '.addTo(map).bindPopup("<img src=''img/3kids.png'' width=300 ><br />That Bicyclist has 3 Kids - Drive Safe'),
        (1964, '.addTo(map).bindPopup("<img src=''img/swerve.png'' width=300 ><br />Anticipate Bike Swerve'),
        (2013, '.addTo(map).bindPopup("<img src=''img/becareful.png'' width=300 ><br />Don''t Drive Angry - Be [Car]eful')      
        ;

update signs a
set code = 'L.marker([' ||
trim(both ')' from substring(b.astext from position(' ' in b.astext)+1 for 40)) || ',' || 
trim(both 'POINT(' from substring(b.astext from 0 for position(' ' in b.astext)+1)) || '])' || a.sign_text 
from centerline_sign_locations b
where a.gid = b.gid;

update signs a
set code = code || '<br />' || 
 case 
  when
   b.l_from || '-' || b.l_to || ' ' || b.street || ' Cambridge, MA' || ' ' || b.zip_left like '%--1%'
   and  b.r_from || '-' || b.r_to || ' ' || b.street || ' Cambridge, MA' || ' ' || b.zip_right like '%--1%'
 then b.street || ' Cambridge, MA' || ' ' || b.zip_left
  when 
   b.l_from || '-' || b.l_to || ' ' || b.street || ' Cambridge, MA' || ' ' || b.zip_left like '%--1%'
 then 
   b.r_from || '-' || b.r_to || ' ' || b.street || ' Cambridge, MA' || ' ' || b.zip_right
 else 
   b.l_from || '-' || b.l_to || ' ' || b.street || ' Cambridge, MA' || ' ' || b.zip_left
end || '");'
from centerline_sign_locations c, centerlines_srid b
where a.gid = c.gid
and c.gid = b.gid;
```


export signs.code for U/I (from PgAdmin)
this goes directly in bike_accidents.html

