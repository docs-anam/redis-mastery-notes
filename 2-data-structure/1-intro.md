# Redis Data Structures - Introduction & Overview

## Table of Contents
1. [Overview](#overview)
2. [Data Structures at a Glance](#data-structures-at-a-glance)
3. [Choosing the Right Data Structure](#choosing-the-right-data-structure)
4. [Memory Efficiency Comparison](#memory-efficiency-comparison)
5. [Performance Characteristics](#performance-characteristics)
6. [Common Patterns](#common-patterns)
7. [Best Practices](#best-practices)
8. [Common Mistakes](#common-mistakes)
9. [Next Steps](#next-steps)

---

## Overview

Beyond simple key-value pairs (strings), Redis provides powerful data structures that let you model complex real-world scenarios efficiently. Each structure is optimized for specific access patterns and use cases.

### Why Multiple Data Structures?

A single data type (strings) wouldn't be optimal for:
- **Queues**: Need efficient push/pop from both ends
- **Sets**: Need fast membership testing and set operations
- **Leaderboards**: Need ranking with scores
- **Real-time events**: Need time-series data
- **User profiles**: Need structured key-value data

Redis provides specialized structures for each, enabling:
- ✅ Better performance (O(1) vs O(N) operations)
- ✅ Lower memory usage (native types vs JSON strings)
- ✅ Atomic operations on complex data
- ✅ Cleaner, more intuitive code

---

## Data Structures at a Glance

### 1. **Strings** (Review from Basics)
- **What**: Raw bytes or text values
- **Best For**: Session data, counters, serialized objects
- **Memory**: ~45 bytes overhead + data
- **Key Operation**: GET/SET (O(1))

**Example:**
```bash
SET user:1:name "Alice"
GET user:1:name  # "Alice"
```

### 2. **Lists**
- **What**: Ordered collections (linked list)
- **Best For**: Queues, stacks, activity feeds
- **Memory**: ~8 bytes per element
- **Key Operations**: LPUSH/RPUSH/LPOP/RPOP (O(1))

**Example:**
```bash
RPUSH myqueue "task1" "task2" "task3"
LPOP myqueue  # "task1"
```

### 3. **Sets**
- **What**: Unordered, unique elements
- **Best For**: Tags, unique visitors, membership testing
- **Memory**: ~8 bytes per element + hash table overhead
- **Key Operations**: SADD/SREM/SMEMBERS (O(1) to O(N))

**Example:**
```bash
SADD users:online "user1" "user2" "user3"
SISMEMBER users:online "user1"  # 1 (yes)
```

### 4. **Hashes**
- **What**: Maps of field → value (like objects)
- **Best For**: User profiles, config objects, document-like data
- **Memory**: ~16 bytes per field + overhead
- **Key Operations**: HSET/HGET/HGETALL (O(1))

**Example:**
```bash
HSET user:1 name "Alice" age "30" city "NY"
HGET user:1 name  # "Alice"
HGETALL user:1    # {name: "Alice", age: "30", city: "NY"}
```

### 5. **Sorted Sets**
- **What**: Elements with scores, sorted by score
- **Best For**: Leaderboards, rankings, priority queues
- **Memory**: ~16 bytes per element + index overhead
- **Key Operations**: ZADD/ZRANGE/ZRANK (O(log N))

**Example:**
```bash
ZADD leaderboard 100 "alice" 95 "bob" 110 "charlie"
ZRANGE leaderboard 0 -1 WITHSCORES  # sorted by score
```

### 6. **Streams**
- **What**: Log-like data with IDs and timestamps
- **Best For**: Event logs, message queues, audit trails
- **Memory**: ~30 bytes per entry + overhead
- **Key Operations**: XADD/XREAD/XRANGE (O(1) append, O(N) read)

**Example:**
```bash
XADD mystream * user "alice" action "login"
XREAD COUNT 10 STREAMS mystream 0
```

### 7. **HyperLogLog**
- **What**: Probabilistic cardinality estimator
- **Best For**: Unique visitor counts, cardinality estimation
- **Memory**: ~12KB fixed (remarkably small!)
- **Key Operations**: PFADD/PFCOUNT (O(1))

**Example:**
```bash
PFADD daily:visitors:2024 "user1" "user2" "user3"
PFCOUNT daily:visitors:2024  # ~3 unique (with small error margin)
```

### 8. **Bitmaps & Bitfields**
- **What**: Bit-level operations on strings
- **Best For**: Flags, presence tracking, compact storage
- **Memory**: 1 bit per flag (extremely compact)
- **Key Operations**: SETBIT/GETBIT/BITCOUNT (O(1) to O(N))

**Example:**
```bash
SETBIT online:users:2024-01-01 0 1  # User 0 is online
GETBIT online:users:2024-01-01 0    # 1 (yes)
BITCOUNT online:users:2024-01-01    # Count all online
```

### 9. **Geospatial Indexes**
- **What**: Locations with coordinates
- **Best For**: Location-based queries, proximity search
- **Memory**: ~13 bytes per location
- **Key Operations**: GEOADD/GEODIST/GEORADIUS (O(N+log N))

**Example:**
```bash
GEOADD cities 13.361389 38.115556 "Palermo" 15.087269 37.502669 "Catania"
GEODIST cities "Palermo" "Catania" km
```

---

## Choosing the Right Data Structure

### Decision Tree

```
Need to store...
│
├─ Simple values?
│  └─ Use: STRINGS (set/get)
│
├─ Ordered sequence of items?
│  ├─ Need queue/stack? → LISTS (lpush/rpop)
│  └─ Need leaderboard? → SORTED SETS (zadd/zrange)
│
├─ Unique items or membership?
│  ├─ Need set operations? → SETS (sadd/sinter)
│  └─ Need counting? → HYPERLOGLOG (pfadd/pfcount)
│
├─ Object with multiple fields?
│  └─ Use: HASHES (hset/hget)
│
├─ Time-series or events?
│  └─ Use: STREAMS (xadd/xread)
│
├─ Geographic coordinates?
│  └─ Use: GEOSPATIAL (geoadd/georadius)
│
└─ Individual bits or flags?
   └─ Use: BITMAPS (setbit/bitcount)
```

### Quick Reference Table

| Structure | Lookup | Insert | Delete | Use Case |
|-----------|--------|--------|--------|----------|
| String | O(1) | O(1) | O(1) | Simple values |
| List | O(N) | O(1) ends | O(1) | Queues, feeds |
| Set | O(1) | O(1) | O(1) | Tags, unique |
| Hash | O(1) | O(1) | O(1) | Objects, profiles |
| Sorted Set | O(log N) | O(log N) | O(log N) | Leaderboards |
| Stream | O(N) | O(1) | O(N) | Events, logs |
| HyperLogLog | ~O(1) | ~O(1) | N/A | Cardinality |
| Bitmap | O(1) | O(1) | O(1) | Flags, bits |

---

## Memory Efficiency Comparison

Storing 1 million user IDs:

```
Method                  Memory Used    Time Lookup
─────────────────────────────────────────────────
JSON strings            ~20 MB         O(N) parsing
Set (SADD)             ~3 MB          O(1) SISMEMBER
List (LPUSH)           ~2.5 MB        O(N) LPOS
HyperLogLog            ~12 KB         O(1) PFCOUNT*
Bitmap                 ~125 KB        O(1) GETBIT*

* Approximate or with trade-offs
```

**Key Insight**: The right structure can save 100x+ memory!

---

## Performance Characteristics

### Operation Speed Comparison

**1 Million elements:**

```
Operation              String  List   Set    Hash   Sorted Set
─────────────────────────────────────────────────────────────
Add element            1 μs    1 μs   2 μs   3 μs   5 μs
Lookup                 1 μs    100μs  2 μs   3 μs   20 μs
Range query            N/A     5 μs   N/A    N/A    25 μs
Member count           N/A     1 μs   1 μs   1 μs   1 μs
```

**Lessons**:
- Lists are fast at ends but slow in middle (O(N) middle access)
- Sets/Hashes have constant lookup (O(1))
- Sorted Sets add logarithmic overhead for ordering
- Avoid KEYS command on any structure - use SCAN instead

---

## Common Patterns

### Pattern 1: Session Storage
**Structure**: Hash  
**Why**: Multiple fields for one user

```python
import redis
r = redis.Redis()

# Store session
session_data = {
    'user_id': '123',
    'username': 'alice',
    'logged_in_at': '2024-01-01T10:00:00',
    'ip_address': '192.168.1.1'
}
r.hset('session:abc123', mapping=session_data)

# Retrieve field
username = r.hget('session:abc123', 'username')
```

### Pattern 2: Activity Feed
**Structure**: List  
**Why**: Time-ordered sequence, fast push/pop at ends

```python
# Add activity
r.lpush('feed:user:123', 'user:456 liked your post')
r.lpush('feed:user:123', 'user:789 commented')

# Get recent 10 activities
recent = r.lrange('feed:user:123', 0, 9)
```

### Pattern 3: Leaderboard
**Structure**: Sorted Set  
**Why**: Efficient ranking by score

```python
# Add scores
r.zadd('game:leaderboard', {'alice': 100, 'bob': 95, 'charlie': 110})

# Get top 10
top_10 = r.zrevrange('game:leaderboard', 0, 9, withscores=True)

# Get rank
rank = r.zrevrank('game:leaderboard', 'alice')
```

### Pattern 4: Tag System
**Structure**: Set  
**Why**: Fast membership testing and set operations

```python
# Add tags to article
r.sadd('article:1:tags', 'python', 'redis', 'database')

# Check if tagged
is_tagged = r.sismember('article:1:tags', 'python')

# Articles with both "python" AND "redis"
articles = r.sinter('tag:python:articles', 'tag:redis:articles')
```

### Pattern 5: Real-time Message Queue
**Structure**: Stream  
**Why**: Time-ordered, distributed consumer groups

```python
# Publish event
r.xadd('orders', '*', {'user': 'alice', 'amount': '99.99'})

# Consume
messages = r.xread({'orders': '0'}, count=10)
```

---

## Best Practices

### 1. **Choose Structure First**
```python
# ❌ DON'T: Store everything as JSON strings
data = json.dumps({'name': 'Alice', 'age': 30})
r.set('user:1', data)

# ✅ DO: Use appropriate structure
r.hset('user:1', mapping={'name': 'Alice', 'age': 30})
```

### 2. **Set Expiration on Temporary Data**
```python
# Session data expires after 1 hour
r.hset('session:123', mapping=session_data)
r.expire('session:123', 3600)
```

### 3. **Use Pipelining for Bulk Operations**
```python
# ❌ DON'T: Individual commands
for i in range(1000):
    r.hset(f'user:{i}', mapping=data)

# ✅ DO: Pipeline them
pipe = r.pipeline()
for i in range(1000):
    pipe.hset(f'user:{i}', mapping=data)
pipe.execute()
```

### 4. **Monitor Memory Usage**
```bash
# Check memory by data structure
INFO memory
INFO stats
```

### 5. **Use Appropriate Commands**
```python
# ❌ DON'T: Get entire list for one element
data = r.lrange('mylist', 0, -1)
if 'item' in data:
    pass

# ✅ DO: Use LPOS for search
index = r.lpos('mylist', 'item')
if index is not None:
    pass
```

---

## Common Mistakes

### Mistake 1: Wrong Structure for the Job
```python
# ❌ WRONG: Using string for leaderboard
leaderboard_json = json.dumps({'alice': 100, 'bob': 95})
r.set('leaderboard', leaderboard_json)
# Now getting top 10 requires: parse JSON → sort → return

# ✅ RIGHT: Use sorted set
r.zadd('leaderboard', {'alice': 100, 'bob': 95})
r.zrevrange('leaderboard', 0, 9)
```

### Mistake 2: Storing Highly Mutable Complex Data
```python
# ❌ WRONG: Entire user object as one hash value
r.hset('user:123', 'profile', json.dumps(big_profile_object))

# ✅ RIGHT: Break into fields
r.hset('user:123', mapping={
    'name': 'Alice',
    'email': 'alice@example.com',
    'age': '30'
})
```

### Mistake 3: Not Handling Missing Data
```python
# ❌ WRONG: Assumes key exists
value = r.lrange('mylist', 0, -1)
for item in value:
    process(item)

# ✅ RIGHT: Handle missing data
value = r.lrange('mylist', 0, -1)
if value:
    for item in value:
        process(item)
```

### Mistake 4: Using KEYS in Production
```bash
# ❌ WRONG: Blocks Redis on large datasets
KEYS pattern:*

# ✅ RIGHT: Use SCAN
SCAN 0 MATCH pattern:*
```

### Mistake 5: Ignoring TTL
```python
# ❌ WRONG: Cache that never expires
r.hset('cache:user:123', mapping=data)

# ✅ RIGHT: Set appropriate TTL
r.hset('cache:user:123', mapping=data, ex=3600)
```

---

## Next Steps

Now that you understand when to use each structure:

1. **[Lists](2-lists.md)** - Queues and activity feeds
2. **[Sets](3-sets.md)** - Tags and unique collections
3. **[Hashes](4-hashes.md)** - Objects and profiles
4. **[Sorted Sets](5-sorted-set.md)** - Leaderboards and rankings
5. **[Streams](6-stream.md)** - Event logs and queues
6. **[Geospatial](7-geospatial.md)** - Location-based queries
7. **[HyperLogLog](8-hyperloglog.md)** - Cardinality estimation
8. **[Others](9-others.md)** - Bitmaps and bitfields
- Supports push/pop operations at both ends
- Use cases: queues, activity feeds

### 3. **Sets**
- Unordered collections of unique strings
- Efficient membership testing
- Use cases: tags, unique visitors, relationships

### 4. **Sorted Sets (ZSets)**
- Sets with associated scores for ordering
- Use cases: leaderboards, rate limiting, rankings

### 5. **Hashes**
- Field-value pairs, similar to dictionaries
- Efficient for storing objects
- Use cases: user profiles, configuration objects

### 6. **Streams** (Redis 5.0+)
- Log-like data structure for event streaming
- Use cases: message queues, real-time analytics

## Advanced Data Structures

### 1. **Geospatial Indexes**
- Store geographic coordinates and perform location-based queries
- Supports radius searches and distance calculations
- Use cases: location tracking, proximity searches, map features

### 2. **HyperLogLog**
- Probabilistic data structure for cardinality estimation
- Provides approximate count of unique elements with minimal memory
- Use cases: unique visitor counting, analytics, large-scale deduplications

### 3. **Other Specialized Structures**
- Additional data structures for specific use cases
- Bitmap operations for efficient flagging
- Bitfields for compact data storage

## Why Learn Data Structures?
Choosing the right data structure impacts performance, memory usage, and code simplicity.
