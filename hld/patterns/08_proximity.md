# Pattern 8 — Proximity-Based Services

> **Problem:** "Show me restaurants within 2 km" over a table of 10M restaurants. A naive `WHERE distance(lat,lng, ?, ?) < 2` is a full table scan — 10M haversine calculations per query. Geospatial indexing turns this into a small bounded lookup.

## Why Regular Indexes Fail

- Latitude and longitude are two dimensions. A B-tree only orders on one.
- `WHERE lat BETWEEN a AND b AND lng BETWEEN c AND d` uses at most one column; the other is a scan.
- Distance is a function of both → indexes on either column alone are useless for proximity.

Geospatial indexes are structures that project 2D → 1D while preserving locality (nearby points get nearby keys).

## The Main Techniques

| Technique | How | Best for |
|-----------|-----|----------|
| **Geohash** | Interleave lat/lng bits → base-32 string prefix = area | Simple, works in any k/v store |
| **Quadtree** | Recursively split space into 4 quadrants | Non-uniform density (city vs desert) |
| **R-tree** | Bounding rectangles, tree of overlaps | Polygonal regions, not just points |
| **S2 (Google)** | Cells on a sphere via Hilbert curve | Global scale, accurate on the sphere |
| **H3 (Uber)** | Hexagonal grid | Uniform distances between neighbors |
| **PostGIS GiST** | R-tree in Postgres | Full SQL + geo, fast enough for many apps |
| **Redis GEO** | Sorted set + geohash score | Simple in-memory geo |
| **Elasticsearch geo_point** | BKD tree | Search + geo combined |

## Geohash — The Interview Favorite

**Idea:** interleave bits of latitude and longitude, then base-32 encode.

```
lat  37.7749, lng -122.4194 (San Francisco)
→ geohash "9q8yy" (5 chars, ~5 km precision)
→ geohash "9q8yywe" (7 chars, ~150 m precision)
```

Key property: **strings with common prefix are geographically close** (mostly — edge cases at grid boundaries).

**Query "restaurants within 2 km":**
```
1. Compute geohash of query point at precision P (P chosen so cell ~ 2 km).
2. Look up all restaurants with geohash prefix = that hash, plus 8 neighbor cells (handle boundaries).
3. For each candidate, compute exact distance, filter to radius.
```

**Storage:**
```
restaurant_id | geohash    | lat  | lng  | ...
101           | 9q8yywe    | ...
```
Index on `geohash` → prefix scan is fast.

## Redis GEO Commands

```
GEOADD restaurants -122.4194 37.7749 "shake-shack"
GEOADD restaurants -122.4183 37.7758 "chipotle"

GEOSEARCH restaurants FROMLONLAT -122.4194 37.7749 BYRADIUS 2 km ASC
```
- Stores as sorted set with geohash-based score. Great for < 10M points.
- Not sharded across keys — a single Redis instance limit.

## PostGIS

```sql
CREATE TABLE restaurant (id int, name text, location geography(point));
CREATE INDEX idx_restaurant_geo ON restaurant USING GIST (location);

SELECT id, name
FROM restaurant
WHERE ST_DWithin(location, ST_MakePoint(-122.4194, 37.7749)::geography, 2000)
ORDER BY location <-> ST_MakePoint(-122.4194, 37.7749)::geography
LIMIT 20;
```
GiST index is an R-tree over bounding boxes. Great to millions of points on a single Postgres.

## H3 (Uber)

Hex grid over the globe with 16 resolution levels. Each cell has a 64-bit ID.
- `h3.geoToH3(lat, lng, res=9)` → cell ID.
- `h3.kRing(cell, k=2)` → all cells within 2 rings (used for "nearby").

Why hexagons? Equal distance from center to all 6 neighbors — no diagonal-vs-axial ambiguity of squares.

Use case: Uber's surge pricing (aggregate demand per hex), delivery zones.

## The Query Pattern

Regardless of index type, the shape is the same:

```
1. Coarse filter: find candidate points in a bounding region (index scan).
2. Fine filter: compute exact distance / geo predicate on candidates.
3. Rank / limit.
```

The magic is step 1 — reducing 10M points to ~100 candidates.

## Reference Architecture — Uber "Nearby Drivers"

**Requirement:** rider opens app → shows drivers within 5 km, updated as drivers move.

```
Driver → Mobile → Kafka (topic: driver-location, keyed by driverId)
                       │
                       ▼
              Location Consumer
                       │
             updates Redis:
               GEOADD drivers <lng> <lat> driverId
               HSET driver:{id} status='available' cellId=<h3>
                       │
                       ▼
             Rider App:
               GET /nearby?lat=X&lng=Y
                       │
                       ▼
             API → GEOSEARCH BYRADIUS 5 km → up to 20 drivers
                → HMGET driver:{id} status → filter available
                → return list with ETA
```

**Scaling considerations:**
- Redis has a single-instance limit. Shard by **geographic region** (SF, NYC, London) using consistent hashing on cellId.
- Driver location updates are high-frequency (every 4s per driver × 1M drivers = 250k QPS). Batch or downsample.
- Rider queries much lower volume — 1 per app open.

## Real-Time Movement — Handling Updates

- **Location updates**: driver sends every 4 seconds.
- **Snap to road**: use a map-matching service so noisy GPS doesn't jitter.
- **Sparse for idle**: parked driver? Reduce ping frequency.
- **Presence dashboard**: WebSocket (pattern 1) pushes driver location to nearby riders.

## Interview Q&A

**Q: Geohash vs quadtree — how do you choose?**
- Geohash: dead simple, sits in any DB, string prefix = locality. Downside: rectangular cells, uneven cell area near poles.
- Quadtree: adapts to density (denser subdivision in cities). Needs custom index; usually implemented in-memory or via a specialized service.
- H3/S2: hex or spherical cells; better for global scale and uniform neighbors.

**Q: How do you handle the "boundary problem" in geohash?**
- Points on opposite sides of a grid line have very different hashes despite being close.
- Solution: query the target cell *and its 8 neighbors*; merge candidates, then distance-filter.

**Q: How do you shard geospatial data?**
- **By region** (country / state / city) — natural boundaries, but hot regions overflow.
- **By geohash prefix** — first N chars determine shard; auto-balances by density with variable-length prefixes.
- **By H3 cell at low resolution** — 122 cells at res 0 → each shard owns a set.

**Q: How does GEOSEARCH scale to global?**
- It doesn't — Redis GEO is single-instance. For global, shard by region → a router picks the right Redis based on query location.
- Query at region boundaries hits two shards.

**Q: Precision vs performance in geohash?**
- Precision 5 (~5 km cell) → cheap prefix scan, more candidates to filter.
- Precision 8 (~40 m cell) → few candidates, but must expand neighbor set for a 5 km search → 100s of cells.
- Pick precision so cell size ≈ query radius / 2.

**Q: How do you rank results by distance?**
- Compute haversine (or geodesic) distance for each candidate, sort ascending.
- Redis GEOSEARCH does this natively with `WITHCOORD WITHDIST ASC`.

**Q: How do you support "along-a-route" (Waze / navigation)?**
- Polyline of route → sample points along it → geohash each → union of cells intersected.
- Query candidates in that union.

**Q: Non-Euclidean distances (driving vs straight-line)?**
- Proximity index gives candidates by great-circle distance.
- Then rank by ETA from a routing engine (OSRM, Google Directions).
- Never assume 2 km straight-line = 2 km driving.

## Real-World Examples
- **Uber**: H3 hex grid; each hex is a "supply/demand cell". Drivers' locations kept in per-region service.
- **DoorDash / Gopuff**: PostGIS for restaurant/store catalog, Redis GEO for driver tracking.
- **Yelp**: Elasticsearch geo_point queries for "restaurants near me".
- **Tinder / Bumble**: geohash prefix + age/gender filter for candidate pool.
- **Snapchat "Snap Map"**: real-time location stream + tile-based clustering.
- **Zomato / Swiggy**: PostGIS + Redis for restaurant listings and delivery ETA.

## Gotchas
- **Coordinate systems**: WGS84 (lat/lng) vs Mercator projection (x/y meters). Mixing them silently → 100 km errors.
- **Antimeridian**: crossing the international date line breaks naive bounding-box queries.
- **Pole singularity**: geohash cells become very tall near the poles.
- **Privacy**: fine-grained location is regulated (GDPR, CCPA). Never log raw lat/lng longer than necessary; snap to zip code or H3 res-6 for analytics.
- **Stale locations**: driver went offline but Redis still shows them. Expire entries with TTL and refresh on heartbeat.
- **Battery drain**: high-frequency GPS on mobile kills battery. Use activity recognition (stationary vs moving) to throttle.
