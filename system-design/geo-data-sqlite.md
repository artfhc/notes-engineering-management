# Storing Geographic Data in SQLite

## Overview

To store geographic (geo) data in SQLite, you have a few options depending on your use case.

---

## Option 1: Store Latitude and Longitude as REAL Columns

This is the simplest and most common approach.

```sql
CREATE TABLE locations (
    id INTEGER PRIMARY KEY,
    name TEXT,
    latitude REAL,
    longitude REAL
);
```

**Pros:**
- Simple, portable, works with most SQLite clients

**Cons:**
- No native spatial queries like distance or area

---

## Option 2: Use GeoJSON in a TEXT Column

Store full geo shapes (like polygons or paths) as GeoJSON strings.

```sql
CREATE TABLE geo_shapes (
    id INTEGER PRIMARY KEY,
    name TEXT,
    geojson TEXT
);
```

**Pros:**
- Can store complex shapes (e.g., polygons, multipoints)

**Cons:**
- Requires custom logic or a library to parse/query GeoJSON

---

## Option 3: Use SpatiaLite (Spatial Extension for SQLite)

SpatiaLite extends SQLite to support spatial SQL and indexing.

```sql
-- Example (requires SpatiaLite):
SELECT AddGeometryColumn('locations', 'geom', 4326, 'POINT', 'XY');
```

**Pros:**
- Full spatial query support (e.g., `ST_Distance`, `ST_Within`)

**Cons:**
- Adds complexity, requires SpatiaLite installation

---

## Summary

| Use Case | Recommended Approach |
|----------|---------------------|
| Simple coordinates | Latitude/Longitude columns |
| Storing full geometry (GeoJSON) | GeoJSON in TEXT |
| Advanced spatial queries | SpatiaLite |

---

## What is SpatiaLite?

SpatiaLite is a spatial extension to SQLite that enables it to handle geographic data (points, lines, polygons, etc.) and perform spatial queries, similar to what PostGIS does for PostgreSQL.

### 1. What SpatiaLite Adds to SQLite

- **Geometry types:** POINT, LINESTRING, POLYGON, MULTIPOLYGON, etc.
- **Spatial indexing:** Efficient querying using R-Tree indexes
- **Spatial SQL functions:** `ST_Distance`, `ST_Within`, `ST_Intersects`, etc.
- **Geo standards support:** Follows OGC (Open Geospatial Consortium) standards

### 2. How It Works

SpatiaLite is a shared library (`mod_spatialite.so` or `.dll`) that you load into SQLite. It adds:

- Custom SQL functions (like `ST_Distance`)
- Geometry types stored as binary blobs
- Metadata tables (like `geometry_columns`, `spatial_ref_sys`)

---

## Basic Usage Steps

### A. Set Up SpatiaLite

Install SpatiaLite command-line tool or integrate the `.so`/`.dll` with SQLite in your code.

### B. Create a SpatiaLite-Enabled Database

```sql
-- Load the SpatiaLite extension (in CLI or your SQLite code)
SELECT load_extension('mod_spatialite');

-- Initialize spatial metadata
SELECT InitSpatialMetaData();
```

### C. Create a Table with a Geometry Column

```sql
CREATE TABLE places (id INTEGER PRIMARY KEY, name TEXT);
SELECT AddGeometryColumn('places', 'geom', 4326, 'POINT', 'XY');
```

- `4326` is the SRID for WGS84 (used by GPS)
- `'POINT'` defines the geometry type

### D. Insert Geo Data

```sql
INSERT INTO places (name, geom)
VALUES ('Park', ST_GeomFromText('POINT(122.123 37.456)', 4326));
```

### E. Query with Spatial Functions

```sql
-- Find places within 1000 meters of a point
SELECT name
FROM places
WHERE ST_Distance(geom, ST_GeomFromText('POINT(122.100 37.400)', 4326)) < 1000;
```

### F. Create Spatial Index (Optional but Boosts Performance)

```sql
SELECT CreateSpatialIndex('places', 'geom');
```

---

## Tooling

You can work with SpatiaLite using:

- **SpatiaLite GUI:** A GUI database browser
- **QGIS:** Visualize and query spatial data
- **Python:** (with `sqlite3` + `mod_spatialite`)
- **C/C++/Java:** If you can load the extension

---

## Geo Indexes

A geo index (geospatial index) is a specialized data structure that enables fast spatial queries—like "find all locations near this point" or "which objects are within this area"—by indexing location data (like latitude and longitude or geometry).

### How Geo Indexes Work

There are a few common types of geo indexes depending on the database and use case:

### 1. R-Tree Index (used in SQLite/SpatiaLite)

- R-Tree indexes rectangles (bounding boxes) rather than individual points
- Each geometry is wrapped in a bounding box (minX, minY, maxX, maxY)
- These boxes are organized hierarchically, where parent boxes contain child boxes

**Example:**

If you have locations:
- A at (1, 1)
- B at (10, 10)
- C at (5, 5)

The R-Tree groups nearby points into bounding boxes to efficiently rule out regions when querying.

**Query Example:**

```sql
-- Find geometries intersecting a bounding box (minX, minY, maxX, maxY)
SELECT * FROM places
WHERE geom && BuildMbr(0, 0, 6, 6);
```

Only A and C will be returned—B is outside the box.

### 2. QuadTree / Geohash (used in NoSQL systems or search engines)

- The earth is recursively divided into quadrants (QuadTree) or encoded into hash strings (Geohash)
- Nearby locations share similar prefixes (like "9q8yy" for San Francisco)
- Useful for text-based key-value stores (e.g., Redis, Elasticsearch)

### 3. B-Tree Index on Latitude/Longitude (Basic Filtering)

- You can create separate indexes on lat/lon
- Helps filter by rectangular areas (e.g., bounding boxes), but not great for circular radius queries

```sql
CREATE INDEX idx_lat ON places(latitude);
CREATE INDEX idx_lon ON places(longitude);
```

---

## Spatial Query Flow with Index

1. Filter by bounding box using the geo index (fast)
2. Refine with exact geometry comparison (like `ST_Distance`, `ST_Contains`) for precision

This two-step strategy avoids scanning the entire table.

### Index Comparison

| Index Type | Used In | Supports Radius Queries? | Best For |
|-----------|---------|-------------------------|----------|
| R-Tree | SQLite, SpatiaLite | Yes (with refinement) | Bounding box & shape queries |
| QuadTree | Redis, MongoDB | Yes | Grid-based proximity queries |
| Geohash | Elasticsearch | Yes | Search/autocomplete + geo |
| B-Tree | SQL | Only for rectangular area | Simple lat/lon filtering |

---

## Indoor Maps Database Schema

To store indoor maps with pins in SQLite, your database should capture:

- Buildings/floors
- Indoor geometries (e.g., rooms, hallways)
- Pins (e.g., user-added markers, POIs)
- Optional: paths for navigation

Below is a normalized schema example:

### 1. Tables and Schema

#### buildings

```sql
CREATE TABLE buildings (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL
);
```

#### floors

```sql
CREATE TABLE floors (
    id INTEGER PRIMARY KEY,
    building_id INTEGER NOT NULL,
    name TEXT,           -- e.g., "1F", "B2"
    floor_number INTEGER, -- optional for ordering
    map_image_url TEXT,  -- floor plan image if any
    FOREIGN KEY(building_id) REFERENCES buildings(id)
);
```

#### features (optional - rooms, zones, etc.)

```sql
CREATE TABLE features (
    id INTEGER PRIMARY KEY,
    floor_id INTEGER NOT NULL,
    name TEXT,                    -- e.g., "Room 101"
    geometry TEXT,                -- e.g., GeoJSON or WKT polygon
    type TEXT,                    -- e.g., "room", "restroom"
    FOREIGN KEY(floor_id) REFERENCES floors(id)
);
```

#### pins (user-added markers or POIs)

```sql
CREATE TABLE pins (
    id INTEGER PRIMARY KEY,
    floor_id INTEGER NOT NULL,
    x_coord REAL,                -- X on map image or relative to layout
    y_coord REAL,                -- Y on map image
    lat REAL,                    -- optional for GPS-based overlay
    lon REAL,                    -- optional for GPS-based overlay
    label TEXT,
    type TEXT,                   -- e.g., "bathroom", "shelf", "exit"
    metadata TEXT,               -- JSON blob for flexibility
    FOREIGN KEY(floor_id) REFERENCES floors(id)
);
```

#### paths (optional - for navigation)

```sql
CREATE TABLE paths (
    id INTEGER PRIMARY KEY,
    floor_id INTEGER NOT NULL,
    from_pin_id INTEGER,
    to_pin_id INTEGER,
    geometry TEXT,              -- linestring for route
    cost REAL,                  -- time, distance, etc.
    FOREIGN KEY(floor_id) REFERENCES floors(id),
    FOREIGN KEY(from_pin_id) REFERENCES pins(id),
    FOREIGN KEY(to_pin_id) REFERENCES pins(id)
);
```

---

### 2. Implementation Notes

- **Use SpatiaLite** if you want spatial indexing or run geometric queries
- **Use GeoJSON or WKT** (Well-Known Text) for geometries in `features` and `paths`
- **For basic use,** store pixel-relative coordinates (`x_coord`, `y_coord`) if working with floorplan images
- **Add a `last_updated` timestamp field** if you want change tracking

---

### 3. Query Examples

**All pins on Floor 1:**

```sql
SELECT * FROM pins WHERE floor_id = 1;
```

**Rooms intersecting a point (SpatiaLite):**

```sql
SELECT name FROM features
WHERE ST_Intersects(geometry, ST_GeomFromText('POINT(10 15)', 4326));
```

---

## Summary: SpatiaLite Benefits

SpatiaLite gives SQLite GIS capabilities by:

- Adding spatial data types
- Enabling spatial functions and indexing
- Supporting spatial standards (OGC compliance)
