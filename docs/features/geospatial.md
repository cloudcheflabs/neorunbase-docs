# Geospatial Support

NeorunBase provides PostGIS-compatible geospatial SQL functions, enabling location-based queries and spatial analysis directly within the database.

## Geospatial Data Types

NeorunBase supports the following geospatial data types:

- **POINT**: A single location in 2D space (longitude, latitude)
- **LINESTRING**: A sequence of connected points forming a line
- **POLYGON**: A closed shape defined by a series of points
- **GEOMETRY**: A generic type that can hold any of the above

## Spatial Functions

NeorunBase includes a set of spatial functions compatible with PostGIS conventions:

- **Distance**: Calculate the distance between two geometries
- **Contains / Within**: Check if one geometry contains or is within another
- **Intersects**: Determine if two geometries intersect
- **Area / Length**: Compute the area of polygons or length of lines
- **Buffer**: Create a buffer zone around a geometry
- And more standard spatial operations

## Spatial Indexing

NeorunBase uses spatial indexing to accelerate geospatial queries. Spatial indexes enable efficient range queries and proximity searches without scanning the entire dataset.

## Use Cases

- **Location-based services**: Find points of interest near a given location
- **Geofencing**: Determine if a point falls within a defined geographic boundary
- **Spatial analytics**: Analyze geographic distribution of data
- **Fleet tracking**: Track and query moving objects across regions
