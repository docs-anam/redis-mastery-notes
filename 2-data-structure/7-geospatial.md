# Redis Geospatial

## Overview
Redis Geospatial indexes enable storing and querying geographic coordinates efficiently. Internally implemented using sorted sets with geohashing, allowing location-based queries like radius searches and distance calculations (Redis 3.2+).

## Key Characteristics
- **Coordinate Storage**: Latitude/longitude pairs (WGS84 format)
- **Sorted Set Based**: Uses sorted sets internally for efficient indexing
- **Geohashing**: Encodes coordinates as sortable strings
- **Radius Queries**: Find members within specified distance
- **Distance Calculation**: Calculate distance between locations
- **High Precision**: ~3.57 meter accuracy for typical use cases

## Common Commands

### Adding & Modifying Coordinates
| Command | Complexity | Description |
|---------|-----------|-------------|
| `GEOADD key longitude latitude member [...]` | O(N log M) | Add one or more locations |
| `GEOPOS key member [member ...]` | O(N) | Get latitude/longitude of members |
| `GEODIST key member1 member2 [unit]` | O(log M) | Get distance between two members |
| `GEOHASH key member [member ...]` | O(N) | Get geohash string of members |

### Distance Units
- `m` - meters (default)
- `km` - kilometers  
- `mi` - miles
- `ft` - feet

### Location Queries
| Command | Complexity | Description |
|---------|-----------|-------------|
| `GEORADIUS key lon lat radius unit` | O(N+log M) | Find members within radius |
| `GEOSEARCH key FROMMEMBER m... BYRADIUS r...` | O(N+log M) | Modern radius search |
| `GEOSEARCHSTORE dest src ...` | O(N+log M) | Store search results |

## Use Cases

### 1. Store Locator
Find nearby stores/restaurants for users.

```redis
// Add store locations (longitude, latitude, name)
GEOADD stores:nyc -73.98 40.75 "flagship_store"
GEOADD stores:nyc -73.95 40.73 "downtown_store"
GEOADD stores:nyc -74.00 40.72 "outlet"
GEOADD stores:nyc -73.96 40.74 "mall_location"

// Find stores within 5km
GEOSEARCH stores:nyc FROMLONLAT -73.97 40.74 BYRADIUS 5 km

// Get coordinates of specific store
GEOPOS stores:nyc "flagship_store"

// Distance between two stores
GEODIST stores:nyc "flagship_store" "downtown_store" km  // Returns: ~3.14 km

// Get geohash
GEOHASH stores:nyc "flagship_store"
```

**Real-world scenario**: E-commerce store finder, restaurant locator, point of interest discovery.

### 2. Ride-sharing (Driver Nearby)
Find nearby drivers for ride requests.

```redis
// Add active driver locations
GEOADD drivers:active -73.98 40.75 "driver:101"
GEOADD drivers:active -73.97 40.74 "driver:102"
GEOADD drivers:active -73.96 40.73 "driver:103"
GEOADD drivers:active -73.95 40.72 "driver:104"

// Find drivers within 2km of passenger
GEOSEARCH drivers:active FROMLONLAT -73.97 40.74 BYRADIUS 2 km

// Calculate distance to each driver
GEODIST drivers:active "driver:101" "driver:102" km

// Driver is far away, remove
ZREM drivers:active "driver:104"

// Update driver location
GEOADD drivers:active -73.98 40.76 "driver:101"  // Overwrites with new location
```

**Real-world scenario**: Uber, Lyft, taxi dispatch systems, delivery services.

### 3. Geofencing & Location Alerts
Alert users when they enter/exit defined areas.

```redis
// Define geofences (stores, dangerous areas, etc.)
GEOADD geofences:stores -73.98 40.75 "store:flagship"
GEOADD geofences:stores -73.95 40.73 "store:downtown"

// Add user location
GEOADD users:locations -73.97 40.74 "user:123"
GEOADD users:locations -74.00 40.72 "user:456"

// Check if user is near any store (within 100m)
GEOSEARCH users:locations FROMMEMBER "user:123" BYRADIUS 100 m

// Get all stores near user
GEOSEARCH geofences:stores FROMMEMBER "user:123" BYRADIUS 500 m
```

**Real-world scenario**: Location-based marketing, asset tracking, smart home zones.

### 4. Delivery Route Optimization
Find nearest delivery destinations.

```redis
// Add delivery points
GEOADD deliveries:pending -73.98 40.75 "order:1001"
GEOADD deliveries:pending -73.97 40.74 "order:1002"
GEOADD deliveries:pending -73.96 40.73 "order:1003"
GEOADD deliveries:pending -73.95 40.72 "order:1004"

// Warehouse location
// Find next 3 closest deliveries
GEOSEARCH deliveries:pending FROMLONLAT -73.96 40.74 BYRADIUS 5 km

// Calculate total distance between order stops
GEODIST deliveries:pending "order:1001" "order:1002" km
GEODIST deliveries:pending "order:1002" "order:1003" km
GEODIST deliveries:pending "order:1003" "order:1004" km
```

**Real-world scenario**: Last-mile delivery, logistics planning, route optimization.

### 5. Event Discovery & Recommendations
Find events near user's current location.

```redis
// Add event locations
GEOADD events:2024 -73.98 40.75 "event:concert:1"  // Concert at Madison Square Garden
GEOADD events:2024 -73.95 40.73 "event:sports:2"   // Sports game at Barclays Center
GEOADD events:2024 -74.00 40.72 "event:festival:3" // Street fair in SoHo

// User's current location
// Find events within 10km
GEOSEARCH events:2024 FROMLONLAT -73.97 40.74 BYRADIUS 10 km

// Find and store results
GEOSEARCHSTORE user:123:nearby:events 10 events:2024 FROMLONLAT -73.97 40.74 BYRADIUS 10 km
```

**Real-world scenario**: Event discovery apps, location-based recommendations, local search.

### 6. Check-in & Social Networking
Record and query user check-ins.

```redis
// User check-ins
GEOADD checkins:2024-01-15 -73.98 40.75 "user:123:checkin:1"  // Coffee shop
GEOADD checkins:2024-01-15 -73.95 40.73 "user:123:checkin:2"  // Office
GEOADD checkins:2024-01-15 -74.00 40.72 "user:456:checkin:1"  // Mall

// Find all check-ins in Manhattan
GEOSEARCH checkins:2024-01-15 FROMLONLAT -73.97 40.75 BYRADIUS 20 km

// Check distance between user check-ins
GEODIST checkins:2024-01-15 "user:123:checkin:1" "user:123:checkin:2" km
```

**Real-world scenario**: Foursquare/Yelp check-ins, location-based social networks.

## Advanced Patterns

### Geospatial Grid Indexing
```redis
// Store locations at multiple resolutions
GEOADD places:detailed -73.98 40.75 "place:1"    // Exact location
GEOADD places:city -73.97 40.74 "place:1"        // City-level
GEOADD places:region -73.95 40.75 "place:1"      // Region-level

// Different granularities for different queries
```

### Bounding Box Search
```redis
// Search within lat/lon box
GEOSEARCH locations FROMMEMBER "user:123" BYBOX 10 10 km
```

### Nearby + Exclude
```redis
// Find locations nearby, except blacklisted
GEOSEARCH locations FROMLONLAT -73.97 40.74 BYRADIUS 5 km COUNT 10
// Then filter blacklist in application
```

## Performance Characteristics

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| Add location | O(log M) | M = number of locations |
| Get distance | O(log M) | Efficient lookup |
| Radius search | O(N+log M) | N = results returned |
| Geohash | O(N) | N = member count |

## Limitations

- **Precision**: ~3.57 meters for typical geohashing
- **No indexing**: Not true 2D spatial index, R-tree would be more efficient
- **Single stream**: Cannot query across multiple geospatial keys efficiently
- **WGS84 only**: Fixed to WGS84 coordinate system
- **Latitude bounds**: -85.05112878 to 85.05112878

## Considerations

### Coordinate Format
- Longitude first, then latitude (not standard longitude/latitude)
- Range: longitude -180 to 180, latitude -85.05 to 85.05

### Accuracy
```
Geohash precision levels:
- Level 1: ±2500km
- Level 5: ±4.89km
- Level 8: ±19.2m
- Level 11: ±1.49m
- Default: Level 11 (~1.49m accuracy)
```

## Best Practices

1. **Store with key for reference**: Include location name/ID as member
2. **Update locations regularly**: For moving objects, refresh coordinates
3. **Use appropriate units**: Choose km, mi based on your needs
4. **Limit result count**: Use COUNT to avoid returning huge result sets
5. **Cache distances**: For frequently queried distances
6. **Index by region**: Separate geospatial indexes by region/country
7. **Monitor memory**: Geospatial indexes can consume significant memory

## Common Pitfalls

1. **Longitude/Latitude confusion**: Redis expects longitude first!
2. **Out of bounds coordinates**: Latitude must be -85 to 85
3. **Distance precision**: Approximate distance, not exact geodesic
4. **No transactions**: Multiple geospatial commands aren't atomic
5. **Memory consumption**: Storing millions of locations uses significant memory
