# Redis Geospatial - Location-Based Queries

## Overview

Redis Geospatial indexes store locations (longitude, latitude) and enable location-based queries. Perfect for:
- **Location Tracking**: Stores, restaurants, users
- **Proximity Search**: Find nearby locations
- **Delivery Routes**: Closest warehouse, drivers
- **Social Features**: Friend locations
- **Geo-fencing**: Alert on location changes

### Why Geospatial?

- **Efficient Proximity**: Find nearby items in O(log N)
- **Atomic Operations**: Add location and query together
- **Accurate Distances**: Haversine formula built-in
- **Multiple Units**: Meters, kilometers, miles
- **Built on Sorted Sets**: Reuse sorted set operations

---

## Core Commands

### Add Locations

```bash
# Add single location
GEOADD cities 13.361389 38.115556 "Palermo"
# (longitude, latitude, member)

# Add multiple
GEOADD cities 13.361389 38.115556 "Palermo" 15.087269 37.502669 "Catania"

# Get coordinates
GEOPOS cities "Palermo"              # Returns: [[lon, lat]]
```

### Distance Queries

```bash
# Get distance between locations
GEODIST cities "Palermo" "Catania"   # Default: meters
GEODIST cities "Palermo" "Catania" km  # Kilometers
GEODIST cities "Palermo" "Catania" mi  # Miles
```

### Nearby Locations

```bash
# Find nearby locations
GEORADIUS cities 15 37 200 km        # Within 200km of 15,37

# From a member
GEORADIUSBYMEMBER cities "Palermo" 200 km

# With distance & coordinates
GEORADIUS cities 15 37 200 km WITHCOORD WITHDIST
```

### Search & Count

```bash
# Get members in area
GEOHASH cities "Palermo"             # Returns: geohash string

# Search and limit
GEORADIUS cities 15 37 100 km COUNT 10
```

---

## Practical Examples

### Example 1: Find Nearby Stores

```python
import redis
r = redis.Redis()

# Add store locations
stores = {
    'store:1': (40.7128, -74.0060),  # NYC
    'store:2': (34.0522, -118.2437),  # LA
    'store:3': (41.8781, -87.6298)    # Chicago
}

for store_id, (lat, lon) in stores.items():
    r.geoadd('stores', lon, lat, store_id)

# Find stores near user (40.7505, -73.9972)
nearby = r.georadius('stores', -73.9972, 40.7505, 50, 'km')
print(f"Nearby stores: {nearby}")

# Get distances
for store in nearby:
    dist = r.geodist('stores', store, 'store:1', 'km')
    print(f"{store}: {dist}km away")
```

### Example 2: Driver Location Tracking

```python
# Update driver locations (real-time)
drivers = {
    'driver:1': (40.7128, -74.0060),
    'driver:2': (40.7505, -73.9972),
    'driver:3': (40.7614, -73.9776)
}

# Add/update drivers
for driver_id, (lat, lon) in drivers.items():
    r.geoadd('active_drivers', lon, lat, driver_id)

# Find closest driver for delivery
delivery_loc = (40.7489, -73.9680)  # Pickup location
closest = r.georadius(
    'active_drivers',
    delivery_loc[1],
    delivery_loc[0],
    5,  # Within 5km
    'km',
    count=1
)

if closest:
    driver = closest[0]
    distance = r.geodist('active_drivers', driver, 'delivery_point', 'km')
    print(f"Closest: {driver} ({distance}km)")
```

### Example 3: Geo-fencing Alerts

```python
def check_geofence(user_id, lat, lon, geofence_lat, geofence_lon, radius_km=1):
    """Check if user entered geofence"""
    distance = r.geodist(
        f'user:{user_id}:location',
        'current',
        'geofence',
        'km'
    )
    
    if distance and float(distance) <= radius_km:
        return True  # User in geofence
    return False

# Usage
if check_geofence(123, 40.7128, -74.0060, 40.7140, -74.0050, radius_km=0.5):
    send_notification("You entered the store!")
```

---

## Performance

```
Operation              Time         Memory
──────────────────────────────────────────
GEOADD                 O(log N)     13 bytes per location
GEODIST                O(log N)     Constant
GEORADIUS              O(N+log N)   For N results
GEOPOS                 O(N)         For N locations
```

**Note**: Behind the scenes, geospatial uses sorted sets with geohash encoding.

---

## Best Practices

### 1. Use Geospatial for Location Queries

```python
# ✅ DO: Use GEORADIUS for proximity
nearby = r.georadius('stores', lon, lat, 10, 'km')

# ❌ DON'T: Calculate distances manually
all_stores = r.hgetall('stores')
distances = [calculate_distance(...) for store in all_stores]
# Inefficient! O(N) instead of O(log N)
```

### 2. Update Locations Periodically

```python
# ✅ DO: Periodic location updates
def update_location(user_id, lat, lon):
    r.geoadd('user_locations', lon, lat, user_id)
    r.zadd('location_timestamps', {user_id: time.time()})

# Cleanup stale locations
def cleanup_old_locations(max_age_minutes=30):
    cutoff = time.time() - (max_age_minutes * 60)
    r.zremrangebyscore('location_timestamps', 0, cutoff)
```

### 3. Cache Proximity Results

```python
# ✅ DO: Cache nearby results
def get_nearby_stores(user_id, lat, lon, radius=10):
    cache_key = f'nearby:{user_id}'
    
    # Check cache
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)
    
    # Compute
    nearby = r.georadius('stores', lon, lat, radius, 'km')
    r.setex(cache_key, 300, json.dumps(nearby))  # Cache 5 min
    return nearby
```

---

## Common Mistakes

### Mistake 1: Longitude/Latitude Order

```python
# ❌ WRONG: Swapped order
r.geoadd('locations', lat, lon, name)  # Wrong!

# ✅ RIGHT: Longitude first, then latitude
r.geoadd('locations', lon, lat, name)  # Correct
```

### Mistake 2: Wrong Units

```python
# ❌ WRONG: Assuming meters
distance = r.geodist('locations', 'place1', 'place2')
# Returns distance in meters!

# ✅ RIGHT: Specify units
distance = r.geodist('locations', 'place1', 'place2', 'km')
```

### Mistake 3: Not Handling Null Results

```python
# ❌ WRONG: Assumes result exists
dist = float(r.geodist('locations', 'place1', 'place2'))

# ✅ RIGHT: Handle None
dist = r.geodist('locations', 'place1', 'place2')
if dist:
    distance_km = float(dist) / 1000
```

### Mistake 4: Stale Location Data

```python
# ❌ WRONG: Never update locations
r.geoadd('drivers', lon, lat, driver_id)
# Driver moves, data is stale!

# ✅ RIGHT: Regular updates
schedule.every(10).seconds.do(update_driver_locations)
```

---

## Advanced: Geohashing

Internally, Redis uses geohashing:

```python
# Get geohash
r.geohash('locations', 'place1')  # Returns: geohash string

# Use hash for grouping nearby items
def get_nearby_optimized(lon, lat, radius=10):
    # Geohash buckets nearby items together
    hash = r.geohash('locations', 'reference_point')[0]
    # Fetch items in same hash bucket for efficiency
```

---

## Next Steps

- **[HyperLogLog](8-hyperloglog.md)** - Cardinality estimation
- **[Streams](6-stream.md)** - Location event logs
- **[Bitmaps](9-others.md)** - Geo-fencing with bitmaps

