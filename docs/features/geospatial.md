# Geospatial Support

NeorunBase provides a PostGIS-style set of `ST_*` spatial SQL functions, so location data and spatial predicates work directly inside the same PostgreSQL-wire database as the rest of your schema. The geometry engine is the [LocationTech JTS Topology Suite](https://locationtech.github.io/jts/) (`org.locationtech.jts`), so WKT parsing, predicate evaluation, and constructive operations follow the same semantics as most JTS/GEOS-based stacks.

The functions are registered as scalar SQL functions in the engine's `FunctionRegistry` (see `neorunbase-common/.../func/GeoFunctions.java`), which means they compose with `WHERE`, `SELECT` expressions, `JOIN`, and the rest of the planner like any other built-in function.

## Geometry Data Types

Four geometry types are added to the type system (`neorunbase-common/.../DataType.java`):

| Type         | Meaning                                            |
|--------------|----------------------------------------------------|
| `POINT`      | A single location in 2D space `(x, y)`             |
| `LINESTRING` | A sequence of connected points forming a line      |
| `POLYGON`    | A closed ring (with optional holes)                |
| `GEOMETRY`   | Generic geometry that can hold any of the above    |

On the PostgreSQL wire, `POINT` is surfaced as the native `point` type (OID `600`); `LINESTRING`, `POLYGON`, and `GEOMETRY` are carried as `bytea` (WKB). Values are normally produced and consumed through the `ST_*` constructor / accessor functions rather than typed literally.

## Spatial Functions

All of the functions below are registered and callable today. Names are case-insensitive.

### Constructors & I/O

| Function | Signature | Result |
|---|---|---|
| `ST_GeomFromText` | `ST_GeomFromText(wkt [, srid])` | Parse a WKT string into a geometry |
| `ST_GeomFromGeoJSON` | `ST_GeomFromGeoJSON(json)` | Parse a GeoJSON `Point` / `LineString` / `Polygon` |
| `ST_Point` | `ST_Point(x, y)` | Build a `POINT` from coordinates |
| `ST_MakePoint` | `ST_MakePoint(x, y)` | Alias of `ST_Point` |
| `ST_AsText` | `ST_AsText(geom)` | Geometry → WKT string |
| `ST_AsGeoJSON` | `ST_AsGeoJSON(geom)` | Geometry → GeoJSON string |

### Predicates (return `BOOLEAN`)

| Function | Meaning |
|---|---|
| `ST_Contains(a, b)` | `a` contains `b` |
| `ST_Within(a, b)` | `a` is within `b` |
| `ST_Intersects(a, b)` | `a` and `b` intersect |

### Measurements (return `DOUBLE` / numeric)

| Function | Meaning |
|---|---|
| `ST_Distance(a, b)` | Cartesian (Euclidean) distance between two geometries in coordinate units |
| `ST_Area(geom)` | Area of a polygon |
| `ST_Length(geom)` | Length of a linestring |
| `ST_X(point)` | X coordinate of a point |
| `ST_Y(point)` | Y coordinate of a point |

### Constructive operations (return geometry)

| Function | Meaning |
|---|---|
| `ST_Buffer(geom, distance)` | Buffer zone of `distance` around a geometry |
| `ST_Centroid(geom)` | Centroid point of a geometry |
| `ST_Union(a, b)` | Union of two geometries |
| `ST_Intersection(a, b)` | Intersection of two geometries |
| `ST_Envelope(geom)` | Bounding-box polygon of a geometry |

### SRID management

| Function | Meaning |
|---|---|
| `ST_SetSRID(geom, srid)` | Return a copy of the geometry tagged with `srid` |
| `ST_SRID(geom)` | Read back the geometry's SRID |

> **Distance is Cartesian, not geodesic.** `ST_Distance` returns straight-line distance in the geometry's own coordinate units (JTS `Geometry.distance`). `ST_Distance(ST_Point(0,0), ST_Point(3,4))` is `5.0`. There is no geography type and no metres-on-a-sphere mode; SRID is stored and echoed back but is not used to reproject or to switch to great-circle math. If you need metric distances, project your coordinates before storing them.

## Examples

```sql
-- WKT round-trip.
SELECT ST_AsText(ST_Point(1.0, 2.0));       -- POINT (1 2)

-- Cartesian distance — 3-4-5 triangle.
SELECT ST_Distance(ST_Point(0, 0), ST_Point(3, 4));   -- 5

-- Point-in-polygon.
SELECT ST_Contains(
         ST_GeomFromText('POLYGON((0 0, 10 0, 10 10, 0 10, 0 0))'),
         ST_Point(5, 5));                     -- t

SELECT ST_Intersects(
         ST_Point(5, 5),
         ST_GeomFromText('POLYGON((0 0, 10 0, 10 10, 0 10, 0 0))'));  -- t
```

Spatial predicates and measurements work over table columns like any other expression. The convention for `ST_Point(x, y)` is `x = longitude`, `y = latitude`:

```sql
CREATE TABLE geo_locations (
    id   BIGINT PRIMARY KEY,
    name VARCHAR(100),
    lat  FLOAT,
    lon  FLOAT
) SHARD KEY (id) SHARDS 2;

INSERT INTO geo_locations (id, name, lat, lon)
VALUES (1, 'Office', 37.7749, -122.4194);

-- Distance from a stored point to a fixed coordinate.
SELECT ST_Distance(ST_Point(lon, lat), ST_Point(-122.4094, 37.7849))
FROM geo_locations
WHERE id = 1;
```

## Use Cases

- **Location-based services** — find points of interest near a location with `ST_Distance` in a `WHERE` / `ORDER BY`.
- **Geofencing** — `ST_Contains` / `ST_Within` test whether a point falls inside a boundary polygon.
- **Spatial analytics** — `ST_Area`, `ST_Length`, `ST_Centroid`, `ST_Union`, `ST_Intersection` for geographic aggregation.

## Notable Limits

- **No spatial index.** The engine's `CREATE INDEX ... USING` accepts only `btree`, `hnsw`, and `fts` (`DdlParser.validateIndexType`). There is no GiST / R-tree spatial index; spatial predicates are evaluated as scalar functions during the scan, so filter selectivity comes from other indexed columns rather than the geometry itself.
- **Planar math only.** As noted above, distances and areas are Cartesian in coordinate units; there is no geography (geodesic) type.
- **GeoJSON parsing covers `Point`, `LineString`, and `Polygon`.** Other GeoJSON geometry types are not parsed by `ST_GeomFromGeoJSON`.
