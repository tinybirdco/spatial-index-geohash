# Point in polygon

This is a project to show how to create a spatial index on top of Tinybird using geohash.

It allows to get the polygons a given point is inside.

In order to test it use

```bash
curl https://storage.googleapis.com/geonames-index/geonames.csv --output datasources/fixtures/geonames.csv
tb push --fixtures
```

And then check the endpoints `https://api.tinybird.co/v0/pipes/api_which_polygons_raw.json?lon=0&lat=0` and `https://api.tinybird.co/v0/pipes/api_which_polygons_index.json?lon=0&lat=0` to see the differences.

See next section to recreate it step by step.

## How does it work

### Get the sample data

It was too heavy for GitHub :)

`curl https://storage.googleapis.com/geonames-index/geonames.csv --output datasources/fixtures/geonames.csv`

### Push the Data source and the csv with some rows

`tb push --fixtures datasources/geonames.datasource`

### Push the raw endpoint

`tb push endpoints/api_which_polygons_raw.pipe`

Endpoint performance is OK for the amount of rows we have, but if our geonames grow it will become too slow.

### Generate a new Data Source —geo_index_mv— with a mapping geohash -> Array(geoname_id)

`tb push pipes/geo_index_mv.pipe --populate --wait`

In order to calculate all the geohash covered by a polygon, the polygon bound box is calculated (longitude is wrapped to [-180, 180]) and then used with `geohashesInBox` which returns a list of them.

> Geohash level is decided based on the polygon area. In this example, polygons are pretty large, so a low number is used. For small polygons, a higher level of detail should be used. Level 3 is hardcoded in this case.

## Push the new pipe using the index to fiter

`tb push endpoints/api_which_polygons_index.pipe`

Then, when querying data, geohash(point) is calculated and, with the `geo_index` table, the list of `geoname_id` is retrieved and used to get the actual polygons.

That list is not the final one, the point can have the same geohash but be outside of the polygon, so we need to check if the point is inside the polygon with `pointInPolygon`.

This function is pretty expensive but with the index we just query a small percentaje of the data.

---
Repo is based on this [blogpost](https://www.tinybird.co/blog-posts/spatial-indexing-aids-finding-which-polygons-contain-a-point) but adapted to work without Join tables.
